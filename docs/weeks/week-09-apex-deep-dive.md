# Week 9 — Apex Deep Dive

**Week of:** August 3, 2026
**Estimated study time:** ~2 hours
**Tags:** `salesforce` `apex` `triggers`

---

## Overview

Apex is Salesforce's proprietary, strongly-typed, Java-like language that runs entirely on the Salesforce platform. Unlike Python or Java running on infrastructure you own, every Apex transaction executes inside a governor-enforced sandbox: the platform tracks CPU time, heap, SOQL queries, DML rows, and a dozen other resources per transaction, and kills your code the moment any limit is breached. This isn't just a quirk — it's the single most important architectural constraint you must design around. Every pattern covered this week exists to stay within those limits.

For a senior engineer who writes both Apex and Python/FastAPI, the contrast is instructive. In Python you think about I/O and memory at the infrastructure level; in Apex you think about it at the platform limit level. A bulk trigger that fires on 200 records must perform the same number of SOQL queries as one that fires on 1 record — or you'll hit the 100-SOQL-per-transaction limit and crash your batch inserts in production. This forces a programming discipline — collect, query, process in bulk, DML once — that makes Apex code structurally different from equivalent Python.

The AESF codebase you work in every day is a living example of these patterns and their edge cases. AccountTriggerHandler, ContactTriggerHandler, and OpportunityTriggerHandler handle bidirectional sync between Salesforce objects and Epic EHR via HTTP callouts to the FastAPI middleware. Each of these handlers must bulkify Epic API calls, manage async execution (callouts can't happen in the same synchronous DML transaction that fires the trigger), and handle partial failures without rolling back unrelated records. Understanding why those handlers are structured the way they are is the goal of this week.

By the end of this session you'll be able to: explain every major governor limit and why it exists, implement a one-trigger-per-object framework from scratch, choose the right async Apex mechanism (@future vs. Queueable vs. Batch vs. Scheduled), write bulkified SOQL and avoid N+1 query patterns, build proper exception handling that doesn't swallow failures silently, write meaningful unit tests with genuine coverage, and read the AESF trigger handlers with full comprehension of every architectural decision.

---

## 1. Execution Context and Governor Limits

Every Apex execution runs inside a **transaction**, and every transaction has a fixed resource budget. The platform resets all governor counters at transaction boundaries. Limits are per-transaction — not per-method, not per-object, not per-user.

### The Limits That Matter Most

| Governor Limit | Synchronous | Asynchronous |
|---|---|---|
| SOQL queries | 100 | 200 |
| SOQL rows returned | 50,000 | 50,000 |
| DML statements | 150 | 150 |
| DML rows | 10,000 | 10,000 |
| CPU time | 10,000 ms | 60,000 ms |
| Heap size | 6 MB | 12 MB |
| Callouts (HTTP/web service) | 100 | 100 |
| Future method calls | 50 | — |
| Queueable jobs enqueued | 50 | 1 (per execute) |
| Batch Apex records | up to 50M | — |

Notice the callout limit: 100 callouts per transaction. In AESF, a trigger on Opportunity might fire for 200 records. If you naively made one HTTP call to the middleware per record, you'd hit the callout limit at record 101 and fail the entire transaction. This is why `OpportunityTriggerHandler` collects all records first, batches them, and sends a single (or small number of) callout(s).

### Checking Limits Programmatically

```apex
// Defensive guard before an expensive operation
if (Limits.getQueries() + 1 > Limits.getLimitQueries()) {
    // log and bail gracefully instead of crashing
    System.debug(LoggingLevel.ERROR, 'SOQL limit would be exceeded');
    return;
}

Integer remaining = Limits.getLimitCpuTime() - Limits.getCpuTime();
System.debug('CPU ms remaining: ' + remaining);
```

The `Limits` class exposes both current consumption (`Limits.getQueries()`) and the ceiling (`Limits.getLimitQueries()`). In production AESF code you rarely see these guards explicitly — the architecture is designed to stay well inside limits by construction — but during debugging they're invaluable.

**Common mistake:** Assuming limits reset between method calls within the same transaction. They do not. If MethodA uses 60 SOQL queries and then calls MethodB which uses 50 more, the transaction total is 110 — over the 100 limit. The reset only happens when the entire transaction completes (commit or rollback).

---

## 2. Trigger Framework Patterns — One Trigger Per Object

Salesforce allows multiple triggers per object, but running multiple triggers on the same object is an antipattern: execution order is non-deterministic, you can't control which fires first, and logic gets scattered across files. The standard is **one trigger per object**, which dispatches to a handler class.

### The Minimal Framework

```apex
// AccountTrigger.trigger
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    AccountTriggerHandler handler = new AccountTriggerHandler();
    handler.run();
}
```

```apex
// TriggerHandler.cls — base class all handlers extend
public virtual class TriggerHandler {

    // Trigger context properties
    @TestVisible
    private static Boolean bypassAll = false;

    public static void bypass(String handlerName) {
        // store in a Set<String> for selective bypass in tests
    }

    public void run() {
        if (bypassAll) return;

        if (Trigger.isBefore) {
            if (Trigger.isInsert)  this.beforeInsert();
            if (Trigger.isUpdate)  this.beforeUpdate();
            if (Trigger.isDelete)  this.beforeDelete();
        } else {
            if (Trigger.isInsert)  this.afterInsert();
            if (Trigger.isUpdate)  this.afterUpdate();
            if (Trigger.isDelete)  this.afterDelete();
            if (Trigger.isUndelete) this.afterUndelete();
        }
    }

    // Virtual methods — override only what you need
    protected virtual void beforeInsert()  {}
    protected virtual void beforeUpdate()  {}
    protected virtual void beforeDelete()  {}
    protected virtual void afterInsert()   {}
    protected virtual void afterUpdate()   {}
    protected virtual void afterDelete()   {}
    protected virtual void afterUndelete() {}
}
```

```apex
// AccountTriggerHandler.cls
public class AccountTriggerHandler extends TriggerHandler {

    private List<Account> newList;
    private Map<Id, Account> oldMap;

    public AccountTriggerHandler() {
        this.newList = (List<Account>) Trigger.new;
        this.oldMap  = (Map<Id, Account>) Trigger.oldMap;
    }

    protected override void afterInsert() {
        // Callout can't run in before context; must be after
        AccountEpicSync.syncToEpic(this.newList);
    }

    protected override void afterUpdate() {
        // Only sync accounts where relevant fields changed
        List<Account> changed = new List<Account>();
        for (Account acc : this.newList) {
            Account old = this.oldMap.get(acc.Id);
            if (acc.Name != old.Name
                || acc.BillingStreet != old.BillingStreet) {
                changed.add(acc);
            }
        }
        if (!changed.isEmpty()) {
            AccountEpicSync.syncToEpic(changed);
        }
    }
}
```

### Why Before vs. After Matters

- **Before triggers** run before the record is saved to the database. Use them for field defaulting, validation, and computed field population. `Trigger.new` is writable here — you can set fields directly without a DML statement.
- **After triggers** run after the record is committed and has an Id. Use them for cross-object operations, callouts (via async), and anything that requires the record Id. `Trigger.new` is read-only.

In AESF, all Epic sync happens in `afterInsert` and `afterUpdate` because: (a) you need the Salesforce Id to build the Epic payload, and (b) the HTTP callout to the middleware is dispatched asynchronously anyway.

**Common mistake:** Performing SOQL inside a for loop over `Trigger.new`. Every iteration fires a new query against the query limit. Always query outside the loop, then use a Map for O(1) lookup inside the loop.

---

## 3. Async Apex — Choosing the Right Mechanism

Because triggers run synchronously and callouts to external systems can't happen in the same DML transaction that saves the record, AESF uses async Apex to hand off Epic sync work. There are four async mechanisms and choosing the wrong one has real consequences.

### @future — Fire-and-Forget

```apex
public class AccountEpicSync {

    @future(callout=true)
    public static void syncToEpic(Set<Id> accountIds) {
        // Re-query inside @future — you can't pass sObjects across async boundaries
        List<Account> accounts = [
            SELECT Id, Name, BillingStreet, BillingCity,
                   AESF__Epic_Client_Id__c
            FROM   Account
            WHERE  Id IN :accountIds
        ];
        // Build payload and callout
        String payload = buildEpicPayload(accounts);
        EpicMiddlewareClient.post('/clients', payload);
    }
}
```

Constraints you must know:
- Cannot pass sObject lists as parameters — pass Ids instead, re-query inside.
- Cannot call another `@future` from within a `@future`.
- Max 50 `@future` calls per transaction.
- No way to chain or monitor completion.

Use `@future` for simple, one-shot callouts where you don't need result handling.

### Queueable — Chainable, Monitorable

```apex
public class AccountSyncQueueable implements Queueable, Database.AllowsCallouts {

    private List<Id> accountIds;

    public AccountSyncQueueable(List<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext ctx) {
        List<Account> accounts = [
            SELECT Id, Name, BillingStreet, AESF__Epic_Client_Id__c
            FROM   Account
            WHERE  Id IN :this.accountIds
        ];
        HttpResponse resp = EpicMiddlewareClient.post(
            '/clients',
            JSON.serialize(buildPayload(accounts))
        );
        if (resp.getStatusCode() != 200) {
            // Log failure, optionally enqueue retry
            ErrorLogger.log('AccountSync failed', resp.getBody());
        }
    }

    private List<Map<String, Object>> buildPayload(List<Account> accounts) {
        List<Map<String, Object>> result = new List<Map<String, Object>>();
        for (Account a : accounts) {
            result.add(new Map<String, Object>{
                'sfId'    => a.Id,
                'epicId'  => a.AESF__Epic_Client_Id__c,
                'name'    => a.Name,
                'address' => a.BillingStreet
            });
        }
        return result;
    }
}
```

Enqueue it from the trigger handler:
```apex
System.enqueueJob(new AccountSyncQueueable(
    new List<Id>(Trigger.newMap.keySet())
));
```

Queueable advantages over @future: can implement interfaces, has a job Id you can monitor in Setup > Apex Jobs, can be chained (enqueue another Queueable from within `execute`), and can accept complex objects.

### Batch Apex — Large Data Volumes

```apex
public class AccountEpicBatchSync
    implements Database.Batchable<SObject>, Database.AllowsCallouts {

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, Name, BillingStreet, AESF__Epic_Client_Id__c,
                   AESF__Sync_Status__c
            FROM   Account
            WHERE  AESF__Sync_Status__c = 'Pending'
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        // scope size is 1–200 records, configurable
        // Each execute() call is its own transaction with fresh governor limits
        EpicMiddlewareClient.post('/clients/bulk', JSON.serialize(scope));

        for (Account a : scope) {
            a.AESF__Sync_Status__c = 'Synced';
        }
        update scope;  // DML inside execute is fine
    }

    public void finish(Database.BatchableContext bc) {
        // Send notification, trigger downstream process
        AsyncApexJob job = [
            SELECT Id, Status, NumberOfErrors
            FROM   AsyncApexJob
            WHERE  Id = :bc.getJobId()
        ];
        System.debug('Batch finished: ' + job.Status
                     + ' errors: ' + job.NumberOfErrors);
    }
}

// Launch:
Database.executeBatch(new AccountEpicBatchSync(), 50);
// 50 = scope size per execute() call
```

Key: each `execute()` call gets a **fresh set of governor limits**. This is why Batch Apex can process millions of records — it never exceeds limits per chunk. The tradeoff is overhead: each chunk is a separate async transaction with startup cost.

### Scheduled Apex

```apex
public class DailyEpicReconcile implements Schedulable {
    public void execute(SchedulableContext sc) {
        Database.executeBatch(new AccountEpicBatchSync(), 50);
    }
}

// Schedule via Apex (can also use Setup > Scheduled Jobs UI):
String cron = '0 0 2 * * ?';  // 2 AM daily
System.schedule('Daily Epic Reconcile', cron, new DailyEpicReconcile());
```

**Common mistake:** Implementing `Database.AllowsCallouts` on a Batch class but using a scope size larger than 1 when each record requires its own HTTP callout. HTTP callout limits (100 per transaction) apply per `execute()` call. With scope=50 records and one callout per record, you'll hit the limit halfway through.

---

## 4. SOQL and SOSL in Depth

SOQL (Salesforce Object Query Language) is to Apex what SQL is to Python — the primary way to read data. It looks like SQL but has important differences.

### SOQL — Core Patterns

```apex
// Basic query with relationship traversal (parent → child)
List<Opportunity> opps = [
    SELECT Id, Name, Amount, StageName,
           Account.Name, Account.AESF__Epic_Client_Id__c,
           (SELECT Id, Subject FROM Tasks WHERE Status = 'Open')
    FROM   Opportunity
    WHERE  StageName NOT IN ('Closed Won', 'Closed Lost')
         AND CloseDate >= :Date.today()
    ORDER BY CloseDate ASC
    LIMIT  200
];

// Map-based query — critical for bulkification
Map<Id, Account> accountMap = new Map<Id, Account>([
    SELECT Id, Name, AESF__Epic_Client_Id__c
    FROM   Account
    WHERE  Id IN :accountIds  // :variable bind — safe, no SOQL injection
]);

// Aggregate query
AggregateResult[] results = [
    SELECT OwnerId, COUNT(Id) cnt, SUM(Amount) total
    FROM   Opportunity
    WHERE  StageName = 'Closed Won'
    GROUP BY OwnerId
    HAVING COUNT(Id) > 5
];
for (AggregateResult ar : results) {
    Id owner = (Id) ar.get('OwnerId');
    Integer count = (Integer) ar.get('cnt');
}
```

### Dynamic SOQL

When field names or conditions come from configuration (as in AESF's metadata-driven sync field mapping), you need dynamic SOQL:

```apex
String objectApiName = 'Account';
String fields = 'Id, Name, BillingStreet';
String filter = 'AESF__Sync_Status__c = \'Pending\'';

String query = 'SELECT ' + fields
             + ' FROM ' + objectApiName
             + ' WHERE ' + filter
             + ' LIMIT 200';

List<SObject> records = Database.query(query);
```

Dynamic SOQL is vulnerable to SOQL injection if you concatenate user input directly. Always use `String.escapeSingleQuotes()` on any externally supplied value.

### SOSL — When to Use It

SOSL (Salesforce Object Search Language) searches across multiple objects simultaneously using a full-text index:

```apex
// Find 'Acme' across Accounts, Contacts, and Opportunities in one call
List<List<SObject>> results = [
    FIND 'Acme*'
    IN ALL FIELDS
    RETURNING
        Account(Id, Name, BillingCity),
        Contact(Id, FirstName, LastName, Email),
        Opportunity(Id, Name, StageName)
    LIMIT 20
];

List<Account> accounts   = (List<Account>) results[0];
List<Contact> contacts   = (List<Contact>) results[1];
List<Opportunity> opps   = (List<Opportunity>) results[2];
```

SOSL is expensive (one SOSL = one query toward your 20 SOSL limit per transaction, separate from the 100 SOQL limit). Use it only when searching across multiple objects by keyword. For all other cases, use SOQL with indexed fields.

**Common mistake:** Using SOQL `LIKE '%keyword%'` instead of SOSL for text search. The leading wildcard forces a full table scan (non-selective query), which Salesforce can reject on large objects with a `System.QueryException: Non-selective query against large object type` error.

---

## 5. Bulkification Patterns

Bulkification is the practice of writing code that handles 1 record and 200 records identically from a governor-limit perspective. Every trigger fires with `Trigger.new` potentially containing up to 200 records (the DML batch size). Code that isn't bulkified will work in unit tests (1 record) and fail in production batch loads (200 records).

### The N+1 Anti-Pattern and Its Fix

```apex
// BAD — SOQL inside loop: 1 query per record
for (Contact con : Trigger.new) {
    Account acc = [SELECT Id, Name FROM Account WHERE Id = :con.AccountId];
    // process acc
}

// GOOD — collect IDs, one query, Map lookup inside loop
Set<Id> accountIds = new Set<Id>();
for (Contact con : Trigger.new) {
    if (con.AccountId != null) accountIds.add(con.AccountId);
}

Map<Id, Account> accountMap = new Map<Id, Account>([
    SELECT Id, Name, AESF__Epic_Client_Id__c
    FROM   Account
    WHERE  Id IN :accountIds
]);

for (Contact con : Trigger.new) {
    Account acc = accountMap.get(con.AccountId);
    if (acc != null) {
        // process
    }
}
```

### Bulkifying DML

```apex
// BAD — DML inside loop
for (Contact con : Trigger.new) {
    AESF__Contact_Number__c num = new AESF__Contact_Number__c(
        AESF__Contact__c = con.Id,
        AESF__Type__c    = 'Primary'
    );
    insert num;  // 1 DML per record → 200 records = 200 DML statements
}

// GOOD — collect and DML once
List<AESF__Contact_Number__c> toInsert = new List<AESF__Contact_Number__c>();
for (Contact con : Trigger.new) {
    toInsert.add(new AESF__Contact_Number__c(
        AESF__Contact__c = con.Id,
        AESF__Type__c    = 'Primary'
    ));
}
if (!toInsert.isEmpty()) {
    insert toInsert;
}
```

### Bulkifying Callouts in AESF

The middleware `/clients` endpoint accepts a list in the POST body. The AESF sync classes collect all changed accounts from the trigger batch and send one HTTP request:

```apex
public class AccountEpicSync {

    @future(callout=true)
    public static void syncToEpic(Set<Id> accountIds) {
        List<Account> accounts = [
            SELECT Id, Name, BillingStreet, BillingCity, BillingState,
                   BillingPostalCode, BillingCountry,
                   AESF__Epic_Client_Id__c, AESF__Sync_Status__c
            FROM   Account
            WHERE  Id IN :accountIds
        ];

        List<Map<String, Object>> payload = new List<Map<String, Object>>();
        for (Account a : accounts) {
            payload.add(new Map<String, Object>{
                'salesforce_id' => a.Id,
                'epic_id'       => a.AESF__Epic_Client_Id__c,
                'name'          => a.Name,
                'address'       => new Map<String, Object>{
                    'street'  => a.BillingStreet,
                    'city'    => a.BillingCity,
                    'state'   => a.BillingState,
                    'zip'     => a.BillingPostalCode,
                    'country' => a.BillingCountry
                }
            });
        }

        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:AESF_Middleware/api/v2/clients/bulk');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(JSON.serialize(payload));
        req.setTimeout(30000);

        Http http = new Http();
        HttpResponse resp = http.send(req);

        if (resp.getStatusCode() != 200) {
            // Log, update sync status fields to 'Failed'
            ErrorLogger.logCalloutFailure('AccountEpicSync', resp);
        }
    }
}
```

One HTTP callout for up to 200 records — well within the 100-callout limit.

**Common mistake:** Forgetting to add the external endpoint to Setup > Remote Site Settings (or Named Credentials). A missing remote site setting will throw a `CalloutException: Unauthorized endpoint` at runtime, not at compile time.

---

## 6. Exception Handling

Apex exceptions fall into two categories: **caught** (you can write a try/catch) and **uncaught** (DML governor violations, which kill the transaction immediately). Good exception handling strategy depends on which one you're dealing with.

### try/catch/finally

```apex
public class EpicMiddlewareClient {

    public static HttpResponse post(String path, String body) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:AESF_Middleware' + path);
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(body);
        req.setTimeout(20000);

        try {
            Http http = new Http();
            HttpResponse resp = http.send(req);
            return resp;
        } catch (System.CalloutException e) {
            // Network-level failure (timeout, DNS, TLS)
            ErrorLogger.log('Callout failed: ' + path, e.getMessage());
            throw e;  // re-throw so caller knows it failed
        } catch (Exception e) {
            ErrorLogger.log('Unexpected error: ' + path, e.getMessage());
            throw e;
        }
    }
}
```

### Database.SaveResult and Partial Success

When you use `Database.insert(records, false)` (allOrNone = false), Salesforce attempts each record independently and returns results — some may succeed while others fail. This is critical for AESF sync where one bad record shouldn't block 199 good ones:

```apex
List<Database.SaveResult> results = Database.insert(recordsToInsert, false);
List<String> errors = new List<String>();

for (Integer i = 0; i < results.size(); i++) {
    Database.SaveResult sr = results[i];
    if (!sr.isSuccess()) {
        for (Database.Error err : sr.getErrors()) {
            errors.add('Record ' + i + ': ' + err.getMessage()
                       + ' [' + err.getStatusCode() + ']');
        }
    }
}

if (!errors.isEmpty()) {
    // Log to custom error object, send notification, etc.
    ErrorLogger.bulkLog(errors);
}
```

### Custom Exception Classes

```apex
public class EpicSyncException extends Exception {
    private Integer httpStatusCode;

    public EpicSyncException(String message, Integer statusCode) {
        this(message);
        this.httpStatusCode = statusCode;
    }

    public Integer getHttpStatusCode() {
        return this.httpStatusCode;
    }
}

// Usage
if (resp.getStatusCode() >= 500) {
    throw new EpicSyncException(
        'Middleware returned ' + resp.getStatusCode(),
        resp.getStatusCode()
    );
}
```

**Common mistake:** Catching `Exception` at the top of every method and silently swallowing it with just a `System.debug`. This makes failures invisible. In AESF, the pattern is to log to a custom `AESF__Sync_Error__c` object and update the originating record's sync status field so that failures are visible in the UI and queryable in reports.

---

## 7. Unit Testing and @isTest

Salesforce requires 75% code coverage across all Apex to deploy to production. But coverage is a floor, not a goal — a test that touches lines without asserting anything is worse than useless because it gives false confidence. The AESF standard is tests that assert both the happy path and error paths.

### Test Structure

```apex
@isTest
private class AccountTriggerHandlerTest {

    @TestSetup
    static void makeData() {
        // Runs once before all test methods in the class
        // Records created here are visible to all test methods
        Account testAcc = new Account(
            Name          = 'Test Corp',
            BillingStreet = '100 Main St',
            BillingCity   = 'Boston'
        );
        insert testAcc;
    }

    @isTest
    static void testAfterInsert_syncQueued() {
        // Query the record created in @TestSetup
        Account acc = [SELECT Id FROM Account WHERE Name = 'Test Corp' LIMIT 1];

        // Set up mock HTTP response BEFORE calling code that makes callouts
        Test.setMock(HttpCalloutMock.class, new EpicMiddlewareMock(200, '{"status":"ok"}'));

        Test.startTest();
        // Trigger the after insert logic
        Account newAcc = new Account(
            Name          = 'New Epic Client',
            BillingStreet = '200 Elm St'
        );
        insert newAcc;
        Test.stopTest();  // Flushes all async (@future, Queueable) jobs

        // Assert the sync was attempted — check a side effect (e.g., status field)
        Account result = [
            SELECT AESF__Sync_Status__c
            FROM   Account
            WHERE  Id = :newAcc.Id
        ];
        System.assertEquals('Synced', result.AESF__Sync_Status__c,
            'Account should be marked Synced after successful middleware callout');
    }

    @isTest
    static void testAfterInsert_calloutFailure() {
        Test.setMock(HttpCalloutMock.class, new EpicMiddlewareMock(500, 'Internal Server Error'));

        Test.startTest();
        Account failAcc = new Account(Name = 'Fail Corp', BillingStreet = '1 Error Rd');
        insert failAcc;
        Test.stopTest();

        Account result = [SELECT AESF__Sync_Status__c FROM Account WHERE Id = :failAcc.Id];
        System.assertEquals('Failed', result.AESF__Sync_Status__c,
            'Account should be marked Failed when middleware returns 500');
    }
}
```

### HTTP Callout Mock

```apex
@isTest
global class EpicMiddlewareMock implements HttpCalloutMock {
    private Integer statusCode;
    private String  body;

    public EpicMiddlewareMock(Integer statusCode, String body) {
        this.statusCode = statusCode;
        this.body       = body;
    }

    global HttpResponse respond(HttpRequest req) {
        HttpResponse resp = new HttpResponse();
        resp.setStatusCode(this.statusCode);
        resp.setBody(this.body);
        resp.setHeader('Content-Type', 'application/json');
        return resp;
    }
}
```

### Test.startTest() / Test.stopTest()

These two calls are essential for async testing. `Test.startTest()` resets governor limit counters (giving the code inside a fresh budget) and `Test.stopTest()` synchronously executes all queued async work (`@future`, `Queueable`, scheduled jobs) before returning. Without `Test.stopTest()`, async work runs after your assertions and you can't verify its effects.

**Common mistake:** Writing `System.assert(true)` or no assertions at all just to get coverage. This is gaming the metric. The actual value of a test is in its assertions. When the trigger handler is broken, a test with real assertions will fail; one without assertions will pass and leave you debugging production instead of CI.

---

## 8. AESF Trigger Architecture Walkthrough

The AESF trigger architecture for bidirectional sync follows a consistent pattern across all synced objects. Understanding it abstractly lets you navigate any of the 454 Apex classes quickly.

```
Trigger (1 per object)
    └── TriggerHandler (extends base TriggerHandler)
            └── Sync Class (static @future or Queueable)
                    └── EpicMiddlewareClient (HTTP utility)
                            └── Named Credential: AESF_Middleware
                                    └── FastAPI endpoint (aesf-py-middleware)
                                            └── Epic BDE backend
```

### The Callout-in-Trigger Problem

Apex does not allow HTTP callouts in the same synchronous transaction as a DML operation. If a trigger fires on `insert Account`, the transaction is already in a DML context — you cannot call `http.send(req)` directly. The workaround is `@future(callout=true)` or Queueable with `Database.AllowsCallouts`.

```apex
// THIS FAILS — CalloutException: You have uncommitted work pending
trigger AccountTrigger on Account (after insert) {
    Http http = new Http();
    HttpRequest req = new HttpRequest();
    req.setEndpoint('https://middleware.example.com/clients');
    http.send(req);  // runtime error
}

// THIS WORKS — defer the callout to async context
trigger AccountTrigger on Account (after insert) {
    AccountEpicSync.syncToEpic(Trigger.newMap.keySet());
    // syncToEpic is annotated @future(callout=true)
}
```

### Recursion Prevention

When the middleware writes back to Salesforce (e.g., updating `AESF__Epic_Client_Id__c` after creating an Epic client), that DML fires the trigger again. Without a recursion guard, you get an infinite loop that terminates only when Apex stack depth limit is hit.

```apex
public class TriggerRecursionGuard {
    private static Set<Id> processedIds = new Set<Id>();

    public static Boolean hasProcessed(Id recordId) {
        return processedIds.contains(recordId);
    }

    public static void markProcessed(Id recordId) {
        processedIds.add(recordId);
    }

    public static void clearAll() {
        processedIds.clear();
    }
}

// In the handler:
protected override void afterUpdate() {
    List<Account> toSync = new List<Account>();
    for (Account acc : this.newList) {
        if (!TriggerRecursionGuard.hasProcessed(acc.Id)) {
            TriggerRecursionGuard.markProcessed(acc.Id);
            toSync.add(acc);
        }
    }
    if (!toSync.isEmpty()) {
        AccountEpicSync.syncToEpic(
            new Map<Id, Account>(toSync).keySet()
        );
    }
}
```

**Common mistake:** Using a simple `Boolean` static flag (`static Boolean isRunning = false`) for recursion prevention. This blocks recursion for the entire object across all records in the transaction. The `Set<Id>` approach is more surgical: it prevents the same record from being re-processed, but allows a legitimately different record's trigger logic to run.

---

## 9. Key Concepts Summary

```
Apex Deep Dive
├── Execution Model
│   ├── Transaction boundary = governor limit reset
│   ├── Sync limits (100 SOQL, 150 DML, 10K DML rows, 10s CPU)
│   └── Async limits (200 SOQL, 60s CPU, 12MB heap)
│
├── Trigger Framework
│   ├── One trigger per object (deterministic order)
│   ├── Base TriggerHandler class (run() dispatches by context)
│   ├── Before = field default/validation (writable Trigger.new)
│   └── After = cross-object ops, async callout dispatch
│
├── Async Apex
│   ├── @future(callout=true) — simple, no chaining, no sObject params
│   ├── Queueable — chainable, monitorable, complex params
│   ├── Batch — large data volume, fresh limits per chunk
│   └── Scheduled — cron-based launch of Batch or Queueable
│
├── SOQL / SOSL
│   ├── Map<Id, SObject> pattern for O(1) lookup
│   ├── Bind variables (:var) prevent SOQL injection
│   └── SOSL for cross-object text search (20/tx limit)
│
├── Bulkification
│   ├── No SOQL/DML inside loops
│   ├── Collect → Query → Process → DML once
│   └── Callouts: batch records, one HTTP call to middleware
│
├── Exception Handling
│   ├── try/catch/re-throw for callouts
│   ├── Database.insert(list, false) for partial-success DML
│   └── Custom Exception classes for typed error handling
│
├── Unit Testing
│   ├── @TestSetup for shared test data
│   ├── HttpCalloutMock for callout simulation
│   ├── Test.startTest()/stopTest() flushes async
│   └── Assert outcomes, not just coverage lines
│
└── AESF Architecture
    ├── Trigger → Handler → Sync Class → EpicMiddlewareClient
    ├── @future defers callout past DML transaction boundary
    └── Set<Id> recursion guard prevents write-back loops
```

---

## Quiz — 20 Questions

### Questions

**1.** What is a Salesforce governor limit, and why does the platform enforce them?

**2.** You have a trigger on Contact that fires for 200 records. Inside the trigger you call `[SELECT Id FROM Account WHERE Id = :con.AccountId]` inside a for loop. What happens and how do you fix it?

**3.** What is the difference between `before insert` and `after insert` trigger contexts? Give one reason to use each.

**4.** In the one-trigger-per-object pattern, what is the role of the base `TriggerHandler` class?

**5.** Why can't you make an HTTP callout directly in a trigger that fires in response to a DML operation?

**6.** You need to call the AESF middleware from a trigger, then chain a second call that depends on the first call's response. Which async mechanism should you use and why?

**7.** What is the difference between `@future` and `Queueable` Apex? Name two limitations of `@future` that Queueable overcomes.

**8.** In Batch Apex, why does each `execute()` call get fresh governor limits?

**9.** What is the `allOrNone` parameter in `Database.insert(records, allOrNone)` and when would you set it to `false` in AESF?

**10.** Explain SOQL injection. How do bind variables (`:variable`) prevent it?

**11.** When should you use SOSL instead of SOQL?

**12.** What does `Test.startTest()` / `Test.stopTest()` do, and why is it required when testing `@future` methods?

**13.** What is an `HttpCalloutMock` and why must you register one before any test that invokes Apex code that makes HTTP callouts?

**14.** Describe the recursion problem that can occur when the AESF middleware writes `AESF__Epic_Client_Id__c` back to a Salesforce Account after creating the Epic client record.

**15.** You have an `Account` trigger that runs `AccountEpicSync.syncToEpic()` in `afterUpdate`. A user edits only the Account's Description field (not synced to Epic). What pattern prevents an unnecessary callout to the middleware?

**16.** What is `@TestSetup` and how does it differ from data created directly inside a test method?

**17.** A Batch Apex job processes 50,000 Accounts with a scope size of 200. How many `execute()` calls will there be, and does each one have access to the full 100 SOQL query limit?

**18.** You want to run a full Epic re-sync every night at 2 AM. Describe the two Apex classes you'd write and how you'd schedule the job.

**19.** What is a Named Credential in Salesforce and why does AESF use one instead of hardcoding the middleware URL?

**20.** What is the minimum code coverage percentage required to deploy Apex to a Salesforce production org, and why is meeting this minimum not sufficient for reliable production code?

---

### Answers

??? note "Reveal Answers"

    **1.** Governor limits are per-transaction resource caps enforced by the Salesforce platform to ensure that no single customer's code can monopolize shared multi-tenant infrastructure. The platform runs thousands of customers' Apex on shared compute; without limits, one poorly written transaction could consume all database connections or CPU for everyone else. Limits apply per transaction and reset when the transaction completes. Common limits include 100 SOQL queries, 150 DML statements, 10,000 DML rows, 10 seconds of CPU time, and 6 MB of heap in synchronous context.

    **2.** The for loop issues one SOQL query per Contact record. With 200 records you'd hit the 100 SOQL limit on the 101st iteration and get a `System.LimitException: Too many SOQL queries: 101`. The fix: collect all `AccountId` values into a `Set<Id>` before the loop, run one query (`WHERE Id IN :accountIds`), build a `Map<Id, Account>` from the results, and do a `map.get(con.AccountId)` lookup inside the loop. Zero SOQL queries inside the loop.

    **3.** Before triggers run before the record is committed to the database; `Trigger.new` is writable so you can modify field values without a DML statement — useful for field defaulting, computed fields, and validation that should block the save. After triggers run after the record is committed and has a permanent Id; `Trigger.new` is read-only but you now have the Id to use in related-record operations, cross-object updates, and dispatching async callouts that need the record Id. In AESF, Epic sync is always in `afterInsert`/`afterUpdate` because you need the record Id to build the middleware payload.

    **4.** The base `TriggerHandler` class provides a single `run()` method that inspects `Trigger.isBefore`, `Trigger.isAfter`, `Trigger.isInsert`, `Trigger.isUpdate`, etc., and routes execution to the appropriate virtual method (`beforeInsert`, `afterUpdate`, etc.). Subclasses override only the methods they need. This keeps the trigger itself to a single line (`handler.run()`), ensures consistent routing logic across all handlers, and provides a single place to add cross-cutting features like bypass flags for testing and recursion guards.

    **5.** Salesforce prohibits callouts in a context that has pending (uncommitted) DML operations. When a trigger fires during `insert Account`, the database transaction is open — the Account rows are locked but not committed. Issuing an HTTP callout at this point would mean holding an open database transaction while waiting for an external system to respond, which could block database rows indefinitely. The platform detects this and throws `System.CalloutException: You have uncommitted work pending`. The solution is to defer the callout to an async context (`@future(callout=true)` or `Queueable`) that executes in a new transaction after the triggering DML has committed.

    **6.** Use Queueable Apex (`implements Queueable, Database.AllowsCallouts`). `@future` methods cannot call other `@future` methods, so chaining is impossible. Queueable allows you to enqueue a second Queueable job from within the first `execute()` method after processing the first callout's response, creating a chain: `System.enqueueJob(new SecondSyncJob(firstResponse))`. Queueable also gives you a job Id for monitoring and accepts complex typed objects as constructor parameters, unlike `@future` which only accepts primitives and collections of primitives.

    **7.** `@future` is a static method annotation that runs the annotated method asynchronously; it cannot accept sObject parameters (only primitives/collections), cannot be called from within another `@future`, and provides no job monitoring. Queueable is an interface (`Queueable`) implemented on a class; it can accept complex objects in its constructor, can be chained (enqueue a new Queueable from `execute()`), returns a job Id that appears in Setup > Apex Jobs, and works cleanly with callouts when also implementing `Database.AllowsCallouts`.

    **8.** Batch Apex was explicitly designed for large-data-volume processing. Each `execute()` call is dispatched as a separate asynchronous Apex transaction with its own governor limit budget. This means whether you process 1,000 records or 10 million, no single `execute()` call ever exceeds limits — it processes its chunk (e.g., 50 records), commits, and the next `execute()` starts fresh. The tradeoff is per-transaction overhead: each chunk has startup cost, so very small scope sizes on very large datasets can be slow.

    **9.** When `allOrNone` is `true` (the default), any single record failure rolls back the entire list. When it is `false`, the platform attempts each record independently and returns a `List<Database.SaveResult>` with per-record success/failure information. In AESF, setting `allOrNone = false` on bulk sync operations means one invalid record (e.g., a Contact missing a required Epic field) doesn't block the other 199 valid records from syncing. The caller checks `SaveResult.isSuccess()` for each record and logs failures to a custom error object without abandoning the successful records.

    **10.** SOQL injection occurs when user-controlled input is concatenated directly into a dynamic SOQL string. For example: `Database.query('SELECT Id FROM Account WHERE Name = \'' + userInput + '\'')`— a malicious input like `' OR Name != '` could alter the query logic and return records the user shouldn't see. Bind variables prevent this because the Salesforce query engine treats bound values as data literals, not query syntax, regardless of their content. `WHERE Name = :userInput` is always safe because the platform never parses `userInput` as SOQL syntax.

    **11.** Use SOSL when you need to search for a keyword across multiple object types in a single operation, or when you need full-text search semantics (partial words, relevance ranking). It is more efficient than running separate SOQL queries against each object type. However, SOSL has a separate limit of 20 calls per transaction, results are limited to 2,000 records per object, and it requires data to be indexed. Use SOQL for all structured queries against known objects with indexed fields — it's faster, more predictable, and has a higher per-transaction limit (100).

    **12.** `Test.startTest()` resets all governor limit counters inside its scope, giving the code under test a fresh budget independent of any test setup code. `Test.stopTest()` signals the end of the test scope and synchronously executes all queued asynchronous work — `@future` calls, Queueable jobs, Scheduled jobs — before returning. Without `Test.stopTest()`, async jobs are queued but never run during the test; your assertions execute before the async code does, meaning side effects (like updated fields or callout results) haven't happened yet. You must call `Test.stopTest()` before any assertions that depend on async behavior.

    **13.** An `HttpCalloutMock` is an interface your test class implements to intercept HTTP requests made during a test and return a controlled `HttpResponse` instead of hitting a real endpoint. Salesforce blocks all real HTTP callouts during tests (to prevent test code from altering external systems or depending on network availability). You register the mock with `Test.setMock(HttpCalloutMock.class, new YourMock())` before the code under test runs. Without a registered mock, any test that triggers Apex code making a callout will throw `System.CalloutException: You cannot make callouts from tests`.

    **14.** When an Account is inserted in Salesforce, `AccountTriggerHandler.afterInsert` fires and calls the AESF middleware to create the client in Epic. The middleware responds with an Epic client ID, and a subsequent process writes that ID to `AESF__Epic_Client_Id__c` on the Account (via a DML update). That DML update fires `AccountTrigger` again, triggering `afterUpdate`, which calls the middleware again — an infinite loop. Each iteration fires a new callout and a new DML update until Apex hits the maximum stack depth or another limit. The fix is a `Set<Id>` recursion guard: before processing a record in `afterUpdate`, check if its Id is in the set; if yes, skip it; if no, add it and proceed.

    **15.** Field-change detection: inside `afterUpdate`, compare `Trigger.new` values against `Trigger.oldMap` values for only the fields that map to Epic. If none of the synced fields changed, skip that record. Only records where at least one synced field differs are added to the `toSync` list that gets passed to `AccountEpicSync.syncToEpic()`. If `toSync` is empty after the check, the `@future` method is never enqueued and no callout is made. This is a standard pattern in all AESF trigger handlers.

    **16.** `@TestSetup` is a static method annotated with `@TestSetup` that runs once before all test methods in the class, and its DML operations are rolled back between test methods (each test method gets a fresh copy). This means every test method in the class can query and use the records created in `@TestSetup` without recreating them. Data created directly inside a test method is only visible to that method. `@TestSetup` is preferable for shared reference data (test accounts, contacts, configurations) because it avoids duplicating setup code across test methods and reduces overall test execution time.

    **17.** With 50,000 records and a scope size of 200, there will be 250 `execute()` calls (50,000 ÷ 200). Yes, each `execute()` call runs in its own transaction with fresh governor limits, so each one has the full 100 SOQL query, 150 DML statement, and 60-second CPU budget (async limit). This is the fundamental advantage of Batch Apex for large datasets: you effectively multiply your per-transaction limit budget by the number of chunks.

    **18.** Write a `Schedulable` class that calls `Database.executeBatch()` in its `execute()` method, and a `Database.Batchable` class that performs the actual sync. For example: `DailyEpicReconcileScheduler implements Schedulable` calls `Database.executeBatch(new AccountEpicBatchSync(), 50)`. Schedule it with `System.schedule('Daily Epic Reconcile', '0 0 2 * * ?', new DailyEpicReconcileScheduler())` — the cron expression `0 0 2 * * ?` means second 0, minute 0, hour 2 (2 AM), every day. The Scheduled class is a thin launcher; all logic lives in the Batch class.

    **19.** A Named Credential is a Salesforce configuration object that stores an external service's endpoint URL, authentication method, and credentials (such as an OAuth token or basic auth password) in a secure, admin-controlled store. Apex code references it by label (`callout:AESF_Middleware`) rather than a hardcoded URL. AESF uses Named Credentials because: the middleware URL differs per environment (QA, Staging, Production), changing it requires no code deployment (just update the Named Credential in Setup), credentials are stored securely and not visible in Apex source code, and Named Credentials automatically manage authentication headers.

    **20.** The minimum is 75% code coverage across all Apex classes and triggers (and 0% on no individual class — each class is tested but the aggregate must be 75%). Meeting 75% is not sufficient because coverage measures which lines were executed, not whether the behavior was verified. A test that inserts a record and makes no assertions can cover 100% of a trigger's lines while verifying nothing. In AESF's context this is especially dangerous: a trigger handler that silently swallows exceptions and skips callouts would achieve full coverage with zero assertions, while leaving Epic perpetually out of sync with Salesforce. Meaningful tests assert specific outcomes: field values, error records created, callout payloads sent.

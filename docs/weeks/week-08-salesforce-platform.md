# Week 8 — Salesforce Platform for Developers

**Week of:** July 27, 2026
**Estimated study time:** ~2 hours
**Tags:** `salesforce` `apex` `platform`

---

## Overview

The Salesforce platform is a multi-tenant cloud environment where every customer org shares the same infrastructure, runtime, and database engine. That fundamental architectural decision — multi-tenancy — is the reason governor limits exist, the reason managed packages use namespaces, and the reason deployment tooling has evolved the way it has. Understanding the platform from first principles, rather than just learning its surface APIs, is what separates a developer who fights the platform from one who works with it.

For engineers coming from a backend web-service background (Python, FastAPI, PostgreSQL), Salesforce's metadata-driven architecture feels foreign at first. In a conventional app you own the schema, the runtime, and the deployment pipeline end-to-end. On Salesforce, Apex code runs inside a managed JVM-like container, objects and fields are metadata artifacts versioned alongside code, and "deployment" means pushing a diff of metadata to an org rather than shipping a Docker image. Once you internalize this model, SFDX, change sets, and packaging all make logical sense.

This week covers the foundational platform concepts every senior Salesforce developer must have at their fingertips: org architecture and metadata layers, tooling choices, deployment strategies, security model (profiles, permission sets, sharing), governor limits and their practical implications, and namespaces with managed packages. Each topic is directly relevant to the AESF project — where 454 Apex classes in the `AESF` namespace and 1,556 in `CanaryAMS` must be deployed reliably across QA, Staging, and Production orgs while respecting platform limits on every trigger invocation.

By the end of this week you should be able to explain *why* each constraint or architectural choice exists, not just *what* it is — that depth is what staff-level engineers demonstrate in design reviews and architecture discussions.

---

## 1. Org Architecture and the Metadata Layer

A Salesforce org is best understood as two distinct layers stacked on top of a shared infrastructure:

1. **Data layer** — the rows in sObjects (Accounts, Contacts, custom objects). Stored in a shared Oracle/Postgres-like engine with per-tenant row-level segregation.
2. **Metadata layer** — everything that *describes* the data layer and the behavior: object definitions, field definitions, page layouts, Apex classes, Flows, permission sets, profiles, etc.

Every change you make in Setup UI or via tooling is a metadata change. When you create a custom field `AESF__Policy_Number__c` on `AESF__Policy__c`, Salesforce generates an XML metadata file internally that describes that field. SFDX exposes this as a real file you can version-control:

```xml
<!-- force-app/main/default/objects/AESF__Policy__c/fields/AESF__Policy_Number__c.field-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>AESF__Policy_Number__c</fullName>
    <externalId>false</externalId>
    <label>Policy Number</label>
    <length>80</length>
    <required>false</required>
    <trackTrending>false</trackTrending>
    <type>Text</type>
    <unique>false</unique>
</CustomField>
```

**Metadata API vs. Tooling API:** The Metadata API is the older, file-based deploy/retrieve interface (`.zip` packages). The Tooling API is a REST interface optimised for IDE integration — it can compile and save a single Apex class without a full metadata deploy. VS Code + SFDX uses Tooling API for "Save to Org" (`Ctrl+Shift+S`) and Metadata API for full deployments (`sf project deploy start`).

**sObject relationships** come in three kinds:

| Relationship | Cascade delete | Roll-up summary | SOQL relationship query |
|---|---|---|---|
| Lookup | No (configurable) | No | `SELECT Id, Parent__r.Name FROM Child__c` |
| Master-Detail | Yes (master deleted → children deleted) | Yes (on master) | Same syntax, but required field |
| Many-to-Many (junction) | Two master-details on junction object | Indirectly via roll-up | Two levels of dot-notation |

**Common mistake:** Treating a Lookup like a foreign key with guaranteed referential integrity. If the parent record is deleted and the lookup is optional, child records become orphaned with a null value — not an error. In AESF, `AESF__Line__c` has a required Master-Detail to `AESF__Policy__c`, which is intentional: deleting a policy cascades to lines, mirroring the Epic BDE behavior.

---

## 2. Developer Console vs. VS Code + SFDX

The Developer Console is a browser-based IDE built into every org. It is useful for quick ad-hoc queries and reading debug logs, but it has significant limitations for production development: no source control integration, no local compilation, no test result caching, and it struggles with large orgs.

**VS Code + Salesforce Extension Pack** is the current standard. The key commands you use daily:

```bash
# Authenticate to an org (opens browser OAuth flow)
sf org login web --alias aesf-qa --instance-url https://test.salesforce.com

# List authenticated orgs
sf org list

# Pull metadata from org to local project
sf project retrieve start --source-dir force-app

# Push local changes to scratch org (or deploy to sandbox)
sf project deploy start --source-dir force-app --target-org aesf-qa

# Run Apex tests
sf apex run test --class-names AccountTriggerHandlerTest --target-org aesf-qa --result-format human

# Execute anonymous Apex
sf apex run --file scripts/apex/seed-data.apex --target-org aesf-qa

# Query SOQL
sf data query --query "SELECT Id, Name FROM AESF__Policy__c LIMIT 10" --target-org aesf-qa
```

**SFDX project structure** for the AESF package looks like:

```
sfdx-appliedcrm/
├── sfdx-project.json
├── force-app/
│   └── main/
│       └── default/
│           ├── classes/
│           │   ├── AccountTriggerHandler.cls
│           │   └── AccountTriggerHandler.cls-meta.xml
│           ├── triggers/
│           │   ├── AccountTrigger.trigger
│           │   └── AccountTrigger.trigger-meta.xml
│           └── objects/
│               └── AESF__Policy__c/
│                   ├── AESF__Policy__c.object-meta.xml
│                   └── fields/
```

**When to use Developer Console:** Reading debug logs in production or Staging when you cannot use a local debug session. The log viewer with filter levels (DEBUG, FINEST) is still the fastest way to see what happened in a synchronous transaction without instrumenting a test.

**Common mistake:** Using "Save to Org" in VS Code (Tooling API save) for production deployments. Tooling API saves bypass validation rules and some metadata checks. Always use `sf project deploy start` with `--test-level RunLocalTests` for sandbox-to-sandbox or sandbox-to-prod deploys.

---

## 3. Deployment Strategies: Change Sets vs. Packages vs. SFDX

There are three deployment mechanisms and each has a distinct use case:

### Change Sets (legacy, declarative)

Change sets are point-and-click deployments configured in the Setup UI. You add metadata components to an outbound change set in the source org, upload it, then deploy it in the target org. They require a Deployment Connection between orgs.

**Limitations:** No version control integration, no rollback, components must be added manually, no dependency resolution. Appropriate for small one-off config changes by admins who do not use Git.

### Unlocked Packages (recommended for new work)

An Unlocked Package is a versioned, installable container of metadata. It is the modern equivalent of a managed package for first-party code.

```bash
# Create package
sf package create --name "AESF Core" --package-type Unlocked --path force-app --target-dev-hub devhub

# Create a new version
sf package version create --package "AESF Core" --installation-key-bypass --wait 20

# Install a specific version in an org
sf package install --package 04t... --target-org aesf-staging --wait 10
```

Unlocked packages support dependencies between packages, making them suitable for the AESF architecture where `sfdx-appliedcrm` could declare a dependency on `sfdx-canaryams`.

### SFDX Source Deploy (current AESF workflow)

For orgs where you are not yet using packages, `sf project deploy start` deploys the full source directory or a subset. This is what the AESF CI/CD pipeline currently uses:

```bash
# Deploy with test run, fail fast
sf project deploy start \
  --source-dir force-app \
  --target-org aesf-staging \
  --test-level RunLocalTests \
  --wait 30

# Check deploy status
sf project deploy report --job-id 0Af...

# Validate only (no commit), useful for PR checks
sf project deploy validate \
  --source-dir force-app \
  --target-org aesf-staging \
  --test-level RunLocalTests
```

**Comparison table:**

| Dimension | Change Sets | Unlocked Packages | SFDX Deploy |
|---|---|---|---|
| Version control | No | Yes (package versions) | Yes (Git) |
| Rollback | Manual | Install prior version | Manual |
| CI/CD integration | No | Yes | Yes |
| Namespace required | No | Optional | No |
| Dependency management | No | Yes | No |
| Best for | Admin config | New greenfield packages | Existing orgs mid-migration |

**Common mistake:** Running `sf project deploy start` without `--test-level RunLocalTests` in a sandbox. Salesforce defaults to `NoTestRun` for sandboxes, which means you can deploy broken code that will fail in production (where 75% code coverage is enforced). Always specify the test level explicitly in your pipeline.

---

## 4. Profiles, Permission Sets, and Sharing Rules

The Salesforce security model has two orthogonal axes:

- **Object/Field access** — who can see which objects and fields (controlled by profiles and permission sets)
- **Record access** — which specific records a user can see (controlled by OWD, roles, sharing rules, manual sharing, Apex sharing)

### Profiles and Permission Sets

A **Profile** is a baseline set of permissions assigned one-per-user. Every user has exactly one profile. Profiles control login hours, IP restrictions, app visibility, object CRUD, field-level security, and system permissions.

A **Permission Set** grants *additional* permissions on top of the profile. A user can have many permission sets. Best practice is to use minimal-permission profiles and layer permissions via permission sets — this makes it easier to grant specific capabilities without creating a proliferation of profiles.

```apex
// Query a user's permission set assignments
List<PermissionSetAssignment> assignments = [
    SELECT PermissionSet.Name, PermissionSet.Label
    FROM PermissionSetAssignment
    WHERE AssigneeId = :UserInfo.getUserId()
];
```

**Permission Set Groups** (introduced in API v51) bundle multiple permission sets into a single assignable unit — useful for AESF where the "Integration User" needs a specific combination of object permissions.

### Object Sharing Model

**Organization-Wide Defaults (OWD)** set the baseline visibility for each object:

| Setting | Meaning |
|---|---|
| Private | Users see only their own records (+ hierarchy) |
| Public Read Only | All users can read, owner can edit |
| Public Read/Write | All users can read and edit |
| Controlled by Parent | Inherits from master object (detail records) |

On top of OWD, the **Role Hierarchy** grants access upward: a manager sees records owned by their subordinates. **Sharing Rules** extend access horizontally (e.g., "share all Accounts owned by users in the 'Sales' role with users in the 'Service' role").

In AESF, the Integration User that calls the middleware API typically runs with a System Administrator profile (or a custom profile with equivalent object access) because it needs to read and write all AESF objects. This is a deliberate security trade-off documented in the architecture.

**Common mistake:** Enabling Apex sharing (`with sharing` / `without sharing`) inconsistently. If `AccountTriggerHandler` is declared `with sharing` but calls a helper class declared `without sharing`, the helper *ignores the caller's sharing context* and sees all records. In triggers this is usually the correct behavior (triggers run in system context by default), but it must be explicit and intentional.

---

## 5. Governor Limits — Why They Exist and How to Work With Them

Multi-tenancy means Salesforce cannot allow one tenant to monopolize the shared database or CPU. Governor limits are the enforcement mechanism — hard caps on resource consumption per transaction.

**Key synchronous transaction limits (API v62 / Spring '26):**

| Resource | Limit |
|---|---|
| SOQL queries | 100 |
| SOQL rows returned | 50,000 |
| DML statements | 150 |
| DML rows | 10,000 |
| CPU time | 10,000 ms |
| Heap size | 6 MB |
| Callouts (HTTP/web service) | 100 |
| Future method calls | 50 |

### The Bulkification Imperative

The most common limit violation is caused by putting SOQL inside a loop — the "SOQL in a loop" anti-pattern:

```apex
// BAD — hits limit after 100 accounts
trigger AccountTrigger on Account (before insert, before update) {
    for (Account acc : Trigger.new) {
        // 1 SOQL per record = limit violation at 101st record
        List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
    }
}

// GOOD — 1 SOQL for all records
trigger AccountTrigger on Account (before insert, before update) {
    Set<Id> accountIds = new Map<Id, Account>(Trigger.new).keySet();
    Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();

    for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
        if (!contactsByAccount.containsKey(c.AccountId)) {
            contactsByAccount.put(c.AccountId, new List<Contact>());
        }
        contactsByAccount.get(c.AccountId).add(c);
    }

    for (Account acc : Trigger.new) {
        List<Contact> contacts = contactsByAccount.get(acc.Id) ?? new List<Contact>();
        // process contacts...
    }
}
```

### AESF Trigger Design and Governor Limits

In AESF, `AccountTriggerHandler.syncToEpic` makes an HTTP callout to the middleware API. HTTP callouts are prohibited in synchronous triggers — they must be deferred to `@future` methods or Queueable Apex:

```apex
// Correct pattern used in AESF — defer callout to @future
public class AccountTriggerHandler {
    public static void syncToEpic(List<Account> newAccounts, Map<Id, Account> oldMap) {
        Set<Id> accountIdsToSync = new Set<Id>();
        for (Account acc : newAccounts) {
            if (hasRelevantChange(acc, oldMap?.get(acc.Id))) {
                accountIdsToSync.add(acc.Id);
            }
        }
        if (!accountIdsToSync.isEmpty()) {
            EpicSyncFuture.syncAccounts(new List<Id>(accountIdsToSync));
        }
    }
}

public class EpicSyncFuture {
    @future(callout=true)
    public static void syncAccounts(List<Id> accountIds) {
        List<Account> accounts = [
            SELECT Id, Name, BillingStreet, AESF__Epic_Client_Id__c
            FROM Account
            WHERE Id IN :accountIds
        ];
        // HTTP callout to middleware
        EpicMiddlewareClient.syncAccounts(accounts);
    }
}
```

**Limit: 50 `@future` calls per transaction.** If a batch operation updates 200 accounts, and each calls `@future`, you hit the limit at record 51. Solution: collect all IDs first, then make a single `@future` call with the full list.

### Checking Limits Programmatically

```apex
// Log remaining limits — useful in debug scripts
System.debug('SOQL remaining: ' + Limits.getLimitQueries() - Limits.getQueries());
System.debug('DML remaining: ' + Limits.getLimitDmlStatements() - Limits.getDmlStatements());
System.debug('CPU remaining: ' + Limits.getLimitCpuTime() - Limits.getCpuTime());
```

**Common mistake:** Assuming `@future` avoids all limits. `@future` runs in a new transaction with fresh limits, but that transaction still has the same caps. A future method that queries 50,001 rows will throw a `System.LimitException` just like a synchronous context.

---

## 6. Namespaces and Managed Packages

A **namespace** is a prefix that uniquely identifies an org's metadata in the global Salesforce ecosystem. Once set on a Developer Edition org and tied to an AppExchange listing, it cannot be changed.

In the AESF ecosystem:
- `AESF` — the Applied Epic for Salesforce package namespace
- `CanaryAMS` — the CanaryAMS Agency Management System namespace

Every API name in a managed package is prefixed: `AESF__Policy__c`, `AESF__Line__c`, `CanaryAMS__Agency__c`. This prevents naming conflicts when two packages are installed in the same org.

### Managed vs. Unmanaged vs. Unlocked Packages

| Type | Namespace required | Code visible | Upgradeable | AppExchange distributable |
|---|---|---|---|---|
| Unmanaged | No | Yes (full source) | No (destructive only) | No |
| Managed (Released) | Yes | No (obfuscated) | Yes | Yes |
| Unlocked | Optional | Yes | Yes (within limits) | No |

### Referencing Namespaced Components

In Apex inside a managed package, you reference your own components *without* the namespace prefix (the compiler adds it). In subscriber org Apex that calls into a managed package, you must use the full prefix:

```apex
// Inside AESF package (no prefix needed for own components)
AESF__Policy__c policy = new AESF__Policy__c();

// In subscriber org Apex calling into installed AESF package
AESF__Policy__c policy = new AESF__Policy__c(
    AESF__Policy_Number__c = 'POL-001',
    AESF__Epic_Policy_Id__c = '12345'
);
```

In SOQL, the namespace prefix is always required when querying from outside the package:

```sql
SELECT Id, AESF__Policy_Number__c, AESF__Epic_Policy_Id__c,
       AESF__Account__r.Name
FROM AESF__Policy__c
WHERE AESF__Epic_Policy_Id__c != null
LIMIT 200
```

### Managed Package Constraints

Managed packages impose restrictions that protect installed subscribers:
- You cannot **delete** a global Apex class, public custom object, or public field once the package has a released version with subscribers.
- You can **add** new components freely.
- You can **deprecate** (mark `@deprecated`) but not remove public Apex methods.
- Test classes in a managed package run with `SeeAllData=false` by default and cannot be opted out.

This is directly relevant to AESF maintenance: adding a new field to `AESF__Policy__c` is safe and fully backward-compatible. Renaming it or changing its type requires a new field plus a migration — the old field cannot be removed while subscribers exist.

**Common mistake:** Trying to change a field's type in a managed package (e.g., Text → Picklist). Salesforce prohibits this once the package has been released. The workaround is to create a new field with the new type, migrate data via a one-time script, update all references, and deprecate (but not delete) the old field.

---

## 7. SOQL and SOSL — Platform Query Languages

SOQL (Salesforce Object Query Language) is a SELECT-only SQL dialect for sObjects. It does not support JOINs in the traditional sense — instead it uses relationship queries:

```sql
-- Parent-to-child (subquery)
SELECT Id, Name,
    (SELECT Id, AESF__Line_Number__c, AESF__Premium__c
     FROM AESF__Lines__r)
FROM AESF__Policy__c
WHERE AESF__Account__c = :accountId

-- Child-to-parent (dot notation)
SELECT Id, AESF__Policy_Number__c,
       AESF__Account__r.Name,
       AESF__Account__r.BillingCity
FROM AESF__Policy__c
WHERE AESF__Status__c = 'Active'

-- Aggregate
SELECT AESF__Account__c, COUNT(Id) policyCount, SUM(AESF__Total_Premium__c) totalPremium
FROM AESF__Policy__c
WHERE AESF__Status__c = 'Active'
GROUP BY AESF__Account__c
HAVING COUNT(Id) > 5
```

**SOQL in Apex with bind variables** (always prefer bind variables over string concatenation to prevent SOQL injection):

```apex
String status = 'Active';
Decimal minPremium = 10000;
List<AESF__Policy__c> policies = [
    SELECT Id, AESF__Policy_Number__c
    FROM AESF__Policy__c
    WHERE AESF__Status__c = :status
    AND AESF__Total_Premium__c >= :minPremium
    WITH SECURITY_ENFORCED
    LIMIT 200
];
```

**`WITH SECURITY_ENFORCED`** applies FLS and object-level security to the query, throwing an exception if the running user lacks access to any queried field. Use this in non-trigger, user-context Apex (Aura/LWC controllers) but *not* in trigger handlers (which run in system context).

**SOSL** (Salesforce Object Search Language) performs full-text search across multiple objects simultaneously:

```apex
List<List<SObject>> results = [
    FIND :searchTerm IN ALL FIELDS
    RETURNING Account(Id, Name), Contact(Id, Name, Email),
              AESF__Policy__c(Id, AESF__Policy_Number__c)
    LIMIT 20
];
```

**Common mistake:** Using SOSL when SOQL with a LIKE clause would work. SOSL is for cross-object full-text search; SOQL LIKE is for field-level pattern matching on a known object. SOSL counts against a separate limit (20 queries/transaction) but is also non-deterministic in ordering.

---

## 8. Apex Testing and Code Coverage

Salesforce enforces a minimum of **75% aggregate code coverage** across all Apex classes and triggers before a production deployment. Individual classes have no minimum, but aggregate must be ≥75%.

**Test class structure:**

```apex
@IsTest
private class AccountTriggerHandlerTest {

    @TestSetup
    static void makeData() {
        Account acc = new Account(
            Name = 'Test Client',
            BillingStreet = '123 Main St',
            BillingCity = 'Boston',
            BillingState = 'MA',
            BillingCountry = 'US'
        );
        insert acc;
    }

    @IsTest
    static void testSyncToEpic_UpdateTriggersCallout() {
        Account acc = [SELECT Id FROM Account LIMIT 1];

        // Mock the HTTP callout
        Test.setMock(HttpCalloutMock.class, new EpicMiddlewareMock());

        Test.startTest();
        acc.BillingStreet = '456 New St';
        update acc;
        Test.stopTest();

        // Assert future job enqueued — check side effects, not the future method itself
        // (future method runs synchronously within Test.startTest/stopTest)
        Account updated = [SELECT AESF__Sync_Status__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals('Synced', updated.AESF__Sync_Status__c,
            'Expected sync status to be updated after Epic callout');
    }
}
```

**`Test.startTest()` / `Test.stopTest()`** serve two purposes: they reset governor limit counters (so setup data queries don't count against test limits) and they force asynchronous jobs (`@future`, `Queueable`, `Batch`) to execute synchronously within the block.

**Common mistake:** Writing tests that assert on mock data you inserted in the same test method, not on the actual behavior being tested. A test that inserts an Account, queries it back, and asserts `Name == 'Test'` tests nothing. Tests should assert on *state changes caused by the code under test* — field values set by triggers, records created by handlers, callouts made by future methods.

---

## 9. Asynchronous Apex Patterns

The AESF architecture relies heavily on async Apex because synchronous triggers cannot make HTTP callouts. The four async options and when to use each:

| Pattern | Use when | Limit | Stateful? |
|---|---|---|---|
| `@future` | Simple fire-and-forget callout, no chaining | 50/transaction | No |
| `Queueable` | Need job ID, chaining, more complex state | 50 enqueued/transaction | Yes |
| `Batch Apex` | Process >50k records, needs chunking | 5 concurrent by default | Yes (per chunk) |
| `Schedulable` | Run on a cron schedule | 100 scheduled jobs | No |

**Queueable example** — useful for AESF's Epic sync because it provides a job ID for monitoring:

```apex
public class EpicAccountSyncJob implements Queueable, Database.AllowsCallouts {
    private List<Id> accountIds;

    public EpicAccountSyncJob(List<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext ctx) {
        List<Account> accounts = [
            SELECT Id, Name, BillingStreet, AESF__Epic_Client_Id__c
            FROM Account
            WHERE Id IN :accountIds
        ];
        EpicMiddlewareClient.syncAccounts(accounts);
    }
}

// In trigger handler:
System.enqueueJob(new EpicAccountSyncJob(new List<Id>(accountIdsToSync)));
```

**Batch Apex** for the nightly reconciliation job that compares AESF records against Epic:

```apex
public class EpicReconciliationBatch implements Database.Batchable<SObject>, Database.AllowsCallouts {
    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, AESF__Epic_Client_Id__c, AESF__Last_Sync__c
            FROM Account
            WHERE AESF__Epic_Client_Id__c != null
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        // Process each chunk of 200 (default) accounts
        EpicMiddlewareClient.reconcileAccounts(scope);
    }

    public void finish(Database.BatchableContext bc) {
        AsyncApexJob job = [SELECT Status, NumberOfErrors FROM AsyncApexJob WHERE Id = :bc.getJobId()];
        System.debug('Reconciliation complete. Errors: ' + job.NumberOfErrors);
    }
}

// Execute from anonymous Apex:
Database.executeBatch(new EpicReconciliationBatch(), 50); // 50 records per chunk
```

**Common mistake:** Setting batch size to 1 to avoid governor limits. Each chunk of a batch job still runs in its own transaction with full limits. Setting chunk size to 1 means 200x more transactions for 200 records, which exhausts the daily async Apex limit (250,000 executions) much faster and increases total execution time. Keep batch sizes at 50–200 unless callout limits force smaller chunks.

---

## 10. Key Concepts Summary

```
Salesforce Platform Architecture
│
├── Org
│   ├── Metadata Layer
│   │   ├── Objects & Fields (XML, SFDX source format)
│   │   ├── Apex Classes & Triggers
│   │   ├── Flows & Process Builder (declarative)
│   │   ├── Profiles & Permission Sets
│   │   └── Sharing Rules & OWD
│   │
│   └── Data Layer
│       ├── Standard Objects (Account, Contact, Opportunity)
│       └── Custom Objects (AESF__Policy__c, AESF__Line__c...)
│
├── Tooling
│   ├── Developer Console (browser, quick logs/queries)
│   └── VS Code + SFDX (recommended, CI/CD ready)
│       ├── Tooling API → Save to Org (single file)
│       └── Metadata API → sf project deploy start (full deploy)
│
├── Deployment
│   ├── Change Sets (admin, no VCS, legacy)
│   ├── SFDX Source Deploy (current AESF workflow)
│   └── Unlocked Packages (versioned, dependency-aware)
│
├── Security Model
│   ├── Object/Field Access → Profiles + Permission Sets
│   └── Record Access → OWD + Role Hierarchy + Sharing Rules
│
├── Governor Limits (multi-tenancy enforcement)
│   ├── SOQL: 100 queries, 50k rows
│   ├── DML: 150 statements, 10k rows
│   ├── Callouts: 100 (async only in triggers)
│   └── CPU: 10,000ms
│
├── Namespaces & Packages
│   ├── AESF namespace → AESF__ prefix
│   ├── CanaryAMS namespace → CanaryAMS__ prefix
│   └── Managed: obfuscated, upgradeable, AppExchange
│
└── Async Patterns
    ├── @future — fire-and-forget callout
    ├── Queueable — chained, stateful
    ├── Batch — bulk data processing
    └── Schedulable — cron-triggered
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the primary reason Salesforce enforces governor limits on Apex execution?

**2.** You have an `after insert` trigger on `AESF__Policy__c` that needs to call the middleware API. Why can't you make the HTTP callout directly inside the trigger, and what is the correct pattern?

**3.** Explain the difference between `sf project deploy start` using Tooling API vs. Metadata API. Which does VS Code use for "Save to Org"?

**4.** A colleague deploys code to the QA sandbox without specifying `--test-level`. What test level does Salesforce default to for sandbox deployments, and why is this dangerous?

**5.** What is the difference between a Lookup relationship and a Master-Detail relationship in terms of cascade behavior and roll-up summaries?

**6.** In the AESF security model, the Integration User runs with broad object-level access. What Apex keyword would you use in a trigger handler to ensure it runs in system context and bypasses sharing rules, and what keyword would you use in an Aura controller to respect the current user's record-level access?

**7.** You need to query `AESF__Policy__c` records and include related `AESF__Line__c` records for each policy. Write a SOQL query that retrieves policy number, total premium, and the line number and premium for each related line.

**8.** What is the maximum number of `@future` method calls allowed in a single transaction, and what pattern do you use to stay within this limit when an update to 200 Account records all need to sync to Epic?

**9.** Describe the difference between Unlocked Packages and Managed Packages. Which one is the AESF namespace package type, and what constraint does that impose on removing public API components?

**10.** What does `WITH SECURITY_ENFORCED` do in a SOQL query, and when should you use it in AESF context vs. when should you avoid it?

**11.** Explain why `Test.startTest()` / `Test.stopTest()` are important for testing async Apex like `@future` methods or Queueable jobs.

**12.** What is the OWD (Organization-Wide Default) setting, and how does the Role Hierarchy interact with it?

**13.** You notice that a batch job processing Account reconciliation is taking too long and hitting CPU limits in some chunks. The batch size is set to 200. What adjustments would you consider, and what trade-off exists with smaller batch sizes?

**14.** In a managed package, you realize that a custom field `AESF__Old_Status__c` (Text type) needs to be a Picklist. You have active subscribers. What is the correct migration path?

**15.** What is SOSL and when would you use it instead of SOQL? Give a use case relevant to AESF.

**16.** Explain the `@TestSetup` annotation. What is its performance benefit, and what is the key caveat about data state within individual test methods?

**17.** What is the difference between a Permission Set and a Permission Set Group? When would you use each?

**18.** Why does the AESF integration user need `without sharing` semantics in its data access classes, even if those classes are called from within a trigger?

**19.** In the SFDX project for `sfdx-appliedcrm`, what file type accompanies every Apex class file, and what does it contain?

**20.** You run `sf apex run test --class-names AccountTriggerHandlerTest --target-org aesf-staging` and get 98% coverage on `AccountTriggerHandler.cls` but the production deployment still fails the coverage check. Why might this happen?

---

### Answers

??? note "Reveal Answers"

    **1.** Salesforce is a multi-tenant platform where thousands of customer orgs share the same database infrastructure, application servers, and runtime environment. If any single tenant could run unbounded SOQL queries or DML operations, it could starve other tenants of database connections and CPU. Governor limits are hard per-transaction caps that enforce "fair use" — they guarantee that no single Apex execution can monopolize shared resources. This is fundamentally different from a dedicated server environment where you own all the resources and can tune them as needed.

    **2.** Salesforce prohibits HTTP callouts in synchronous trigger execution contexts because triggers participate in the enclosing DML transaction and callouts cannot be part of an open transaction (they risk leaving the transaction in an indeterminate state if the callout times out). The correct pattern is to collect the affected record IDs in the trigger handler, then pass them to a `@future(callout=true)` method or `Queueable` job that implements `Database.AllowsCallouts`. The async method runs after the triggering transaction commits, in a separate transaction with callout permissions. In AESF, `AccountTriggerHandler.syncToEpic` uses exactly this pattern.

    **3.** The Tooling API is a REST interface optimized for IDE use — it can compile and save a single Apex class or trigger without requiring a full metadata package. VS Code's "Save to Org" (`Ctrl+Shift+S`) uses the Tooling API for speed during development. The Metadata API is the deployment-grade interface that processes a `.zip` package of metadata components, runs tests, validates dependencies, and commits atomically. `sf project deploy start` uses the Metadata API and should always be used for CI/CD pipelines and any cross-org deployments.

    **4.** For sandbox deployments, Salesforce defaults to `NoTestRun` when no `--test-level` is specified. This means Apex tests are not executed and code coverage is not checked. The danger is that you can successfully deploy code with compilation errors caught at test time, or code that reduces aggregate coverage below 75%, which will then cause the next production deployment to fail. Always specify `--test-level RunLocalTests` (or `RunAllTestsInOrg` for critical paths) in any automated pipeline, even for sandbox targets.

    **5.** A Lookup relationship is a loosely coupled reference — the child has an optional or required field pointing to the parent, but deleting the parent does not automatically delete child records (the field is simply nulled out if the lookup is optional, or blocked if required). Roll-up summaries are not available on Lookup relationships. A Master-Detail relationship is tightly coupled — the child record cannot exist without the parent, deleting the parent cascades and deletes all children, and you can define roll-up summary fields on the master. In AESF, `AESF__Line__c` uses Master-Detail to `AESF__Policy__c` because lines are meaningless without their parent policy.

    **6.** Declare trigger handlers and system-context service classes with `without sharing` to run in system context, bypassing record-level sharing rules — appropriate when the integration user needs to access all records regardless of ownership. Declare Aura/LWC controller classes with `with sharing` to enforce the running user's record visibility, preventing data leakage to users who shouldn't see certain records. A class with no explicit keyword inherits the sharing context of its caller, which is ambiguous and should be avoided — always be explicit.

    **7.**
    ```sql
    SELECT Id, AESF__Policy_Number__c, AESF__Total_Premium__c,
           (SELECT Id, AESF__Line_Number__c, AESF__Premium__c
            FROM AESF__Lines__r)
    FROM AESF__Policy__c
    WHERE AESF__Status__c = 'Active'
    LIMIT 200
    ```
    The child relationship name `AESF__Lines__r` is the plural form of the child object's relationship name as defined on the Master-Detail field, suffixed with `__r` for custom relationships.

    **8.** The limit is 50 `@future` calls per synchronous transaction. If 200 Accounts are updated in one DML and each record independently enqueues a `@future`, you hit the limit at record 51 and the entire transaction rolls back. The correct pattern is to collect all record IDs into a single `Set<Id>` in the trigger handler, then make exactly one `@future` call (or one `System.enqueueJob`) passing the full collection. The async method then processes all records in one callout or batches them internally.

    **9.** Unlocked Packages are versioned but their source code is fully visible to the installing org — they are intended for first-party development and internal distribution. Managed Packages (released) have obfuscated Apex source, are distributable on AppExchange, and support upgrades across subscriber orgs. The AESF namespace package is a Managed Package. The critical constraint is that once a managed package version has been released and installed by subscribers, you cannot remove any public Apex class, public field, or public custom object. You can add new components and deprecate old ones, but deletion requires all subscribers to uninstall the package first.

    **10.** `WITH SECURITY_ENFORCED` causes the SOQL query to throw a `System.QueryException` if the running user lacks field-level security access or object-level read access to any field or object referenced in the query. Use it in Aura and LWC Apex controllers where the code runs in user context and you want to respect FLS automatically. Do not use it in trigger handlers or batch jobs — those run in system context and the integration user or batch user may legitimately access fields that are hidden from end-users via FLS. Using it in triggers could cause unexpected failures when integration processes run.

    **11.** `Test.startTest()` resets the governor limit counters, so any SOQL queries or DML executed in `@TestSetup` or before `startTest()` do not count against the limits your code under test consumes. `Test.stopTest()` is the critical piece for async Apex: it forces all enqueued `@future` methods, `Queueable` jobs, and `Batch` jobs that were enqueued during the test to execute synchronously before the test continues. Without `Test.stopTest()`, async jobs would remain in the queue and never execute during the test, making it impossible to assert on their side effects.

    **12.** The OWD is a per-object setting that defines the minimum record visibility for users who are not the record owner and have no other access grant. `Private` means users see only records they own. `Public Read Only` means all users can read all records but only owners can edit. `Public Read/Write` grants everyone full access. The Role Hierarchy sits on top of OWD: if the OWD is `Private`, a user still sees all records owned by users in roles below them in the hierarchy — access flows upward. Sharing rules then extend access horizontally between peer roles or groups without a hierarchy relationship.

    **13.** Reduce the batch size when passed to `Database.executeBatch()` — for example, from 200 to 50. This gives each chunk fewer records to process, reducing CPU and SOQL usage per transaction. The trade-off is that each chunk is a separate transaction with its own overhead (query cost, transaction bookkeeping), so processing 2,000 records in 40 chunks of 50 takes more total elapsed time and more async Apex executions than 10 chunks of 200. You should also profile which specific operation is consuming CPU — often it is N+1 SOQL patterns inside the `execute()` method that can be rewritten as bulk queries.

    **14.** You cannot change a field's type in a released managed package. The migration path is: (1) create a new field `AESF__Status__c` of type Picklist with the desired values; (2) write a one-time data migration script (anonymous Apex or batch) that copies values from `AESF__Old_Status__c` to `AESF__Status__c`; (3) update all Apex code, Flows, reports, and page layouts in the next package version to reference the new field; (4) mark `AESF__Old_Status__c` as `@deprecated` in any Apex references; (5) communicate the deprecation in release notes. The old field cannot be deleted while any subscriber org is still on a package version that references it.

    **15.** SOSL (Salesforce Object Search Language) performs full-text search across multiple object types in a single operation, using Salesforce's search index rather than the database query engine. Use it when the user provides a free-text search term and you don't know which object type holds the matching record. In AESF, a support lookup feature might let a user search for "John Smith" and find matching Accounts, Contacts, and `AESF__Policy__c` records simultaneously — a single SOSL query handles this more efficiently than three separate SOQL LIKE queries.

    **16.** `@TestSetup` annotates a static void method that runs once before any test method in the class. The data it inserts is available to all test methods, but each test method gets its own independent copy — changes made by one test method do not persist to other test methods (Salesforce resets data between test method executions). The performance benefit is significant: instead of each of 20 test methods inserting their own setup data (20 DML transactions), `@TestSetup` runs once. This can reduce test suite runtime by 50% or more in large test classes.

    **17.** A Permission Set is a single collection of permissions (object access, field access, system permissions, app access) that can be assigned to users. A Permission Set Group bundles multiple permission sets into one assignable unit, making it easier to manage complex permission combinations. Use Permission Sets when a single discrete capability needs to be granted (e.g., "can access the Epic Admin panel"). Use Permission Set Groups when a role or persona requires a consistent bundle of permissions across multiple domains (e.g., "AESF Integration User" needs object access from three different permission sets).

    **18.** The AESF integration user that runs the trigger-driven sync may own or not own the records being processed, depending on Salesforce record assignment. If a trigger handler is declared `with sharing`, it will only see records the running user owns or has been explicitly shared. In a bulk sync scenario where records are owned by various sales reps, a `with sharing` trigger handler would silently skip records the integration user cannot see, causing incomplete syncs to Epic with no error. The `without sharing` declaration ensures the handler processes all records in the trigger batch regardless of ownership, which is the correct behavior for a system integration process.

    **19.** Every Apex class file (`ClassName.cls`) is accompanied by a metadata XML file named `ClassName.cls-meta.xml`. This file contains the Apex API version the class targets (e.g., `<apiVersion>62.0</apiVersion>`) and its status (`Active` or `Inactive`). The API version determines which platform features and governor limits apply to that class at runtime — a class compiled at API v45 uses older limit values and may not have access to newer platform features. In AESF, `sfdx-canaryams` classes at API v45 and `sfdx-appliedcrm` classes at API v51 reflect their respective package histories.

    **20.** Running tests against a single class in a sandbox checks coverage for that class only. Salesforce's production deployment check requires **aggregate** code coverage across **all** Apex classes and triggers in the org to be ≥75%. If other classes in the AESF or CanaryAMS namespace have low coverage (e.g., classes that have never had tests written), the aggregate can fall below the threshold even if `AccountTriggerHandler` itself has 98%. The fix is to either write tests for the low-coverage classes or run `--test-level RunAllTestsInOrg` in staging to see the actual aggregate coverage before attempting the production deploy.

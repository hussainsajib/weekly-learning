# Week 11 — Salesforce Data Model & SOQL Mastery

**Week of:** August 17, 2026
**Estimated study time:** ~2 hours
**Tags:** `salesforce` `soql` `data-model`

---

## Overview

Salesforce's data model is the foundation beneath every ETL pipeline, Apex trigger, and reporting snapshot you touch in the CRM-EHR Integration Platform ecosystem. At its core the platform organizes data into standard objects (Account, Contact, Opportunity, etc.) and custom objects (APP__Policy__c, APP__Line__c, the AMS package objects, and so on), each described by metadata that controls field types, validation, sharing, and relationship cardinality. Understanding how those relationships are physically stored — lookup fields, master-detail relationships, external ID fields, and the hidden junction tables that power many-to-many associations — lets you predict both query performance and data-integrity behavior before you write a single line of code or ETL transform.

SOQL (Salesforce Object Query Language) is the primary way code interacts with platform data, and its performance characteristics differ substantially from PostgreSQL or BigQuery SQL. Salesforce enforces governor limits — a maximum of 50,000 rows per query, 100 SOQL queries per synchronous transaction — so query design is never purely academic. Selective queries, proper use of indexed fields, semi-joins, and aggregate functions are not micro-optimizations; they determine whether an Apex trigger completes within governor limits or fails an entire save operation for your users, and whether an ETL batch job runs in minutes or hours.

For the integration platform specifically, SOQL mastery has direct ETL implications. The `etl-bde-pipeline` and `etl-bq-pipeline` pipelines fetch Salesforce data through REST API queries whose underlying SOQL shapes the payload size, the number of API calls consumed, and the fidelity of relationship traversal. A poorly selective query on APP__Policy__c (which may have hundreds of thousands of records in a production org) can time out or exceed row limits, causing incomplete ETL runs and data drift between Salesforce and BigQuery. The AMS package, with 134 custom objects, compounds this: complex relationship trees between policy, coverage, and client objects mean that naive query patterns produce either N+1 anti-patterns or bloated result sets.

This week covers the full arc from data model fundamentals through advanced SOQL techniques, with a deep dive into the integration platform and AMS package object models and their ETL implications. By the end you should be able to write selective, relationship-aware SOQL queries, understand when to use SOSL versus SOQL, and design data model patterns (junction objects, polymorphic lookups, rollup summaries) that remain performant at scale.

---

## 1. Standard and Custom Objects — Structure and Metadata

Every Salesforce object is a combination of a schema definition (fields, relationships, page layouts, validation rules) and a set of records stored in a multi-tenant database that Salesforce abstracts away from direct access. Standard objects like Account, Contact, Opportunity, and Lead ship with the platform and carry built-in behaviors (sharing rules, activity tracking, duplicate management). Custom objects are suffixed with `__c` and live in a package namespace when deployed via a managed package — hence `APP__Policy__c` rather than `Policy__c` in subscriber orgs.

Each field on an object maps to one of Salesforce's field types: Text, Number, Currency, Date/DateTime, Picklist, Formula, Rollup Summary, Lookup Relationship, Master-Detail Relationship, External ID, and several others. The field type determines indexing behavior. Salesforce automatically creates indexes on Id, Name, OwnerId, CreatedDate, LastModifiedDate, SystemModstamp, and any field marked as an External ID or Unique. Standard index fields are the key levers for building selective SOQL queries.

For the integration platform package, custom fields like `APP__Policy__c.APP__EHR_Policy_Id__c` are typically marked as External ID fields so that upsert operations in ETL pipelines can use them as idempotent keys. In `etl-bde-pipeline`, the Pentaho jobs rely on this field to avoid duplicate Policy records when EHR system data is pushed into Salesforce. If that External ID field were removed or its uniqueness constraint dropped, the ETL's upsert logic would silently create duplicates.

**Common mistake:** Treating all custom fields as queryable indexes. Only External ID, Unique, and certain formula fields are indexed. Filtering on a non-indexed Text field on a large object (e.g., `WHERE APP__Policy_Name__c = 'XYZ'` on a table with 500k rows) bypasses indexes and forces a full table scan, which Salesforce will either slow-process or reject as non-selective.

---

## 2. Relationship Types — Lookup, Master-Detail, and Polymorphic

Salesforce relationships come in three primary forms. A **lookup relationship** is a soft foreign key: the child record can exist independently of the parent, and deleting the parent does not cascade-delete children (the lookup field is nullified instead). A **master-detail relationship** is a hard dependency: the child inherits the parent's sharing and OWD settings, rollup summary fields on the parent can aggregate child data, and deleting the parent cascade-deletes all children.

**Polymorphic lookups** are lookups that can point to more than one object type. The standard `Task.WhoId` field is the canonical example — it can reference either a Contact or a Lead. In SOQL, you query polymorphic relationships using the `TYPEOF` expression:

```soql
SELECT Id, Subject,
  TYPEOF Who
    WHEN Contact THEN FirstName, LastName, Email
    WHEN Lead THEN FirstName, LastName, Company
  END
FROM Task
WHERE OwnerId = :currentUserId
```

In the integration platform data model, `APP__Relationship__c` and `APP__Relationships_Junction__c` implement a many-to-many structure between Account (client) records and related parties. Understanding whether these use polymorphic lookups or discrete lookup fields affects how you traverse the relationship tree in SOQL when extracting relationship data for the BigQuery ETL.

**Junction objects** implement many-to-many relationships by having two master-detail fields pointing to the two parent objects. Deleting either parent cascades to the junction record. `APP__Category_Junction__c` connects an Account with an `APP__Agency_Defined_Category__c` — the ETL must handle the possibility that junction records disappear when either parent is deleted, and must propagate those deletes to BigQuery via the `SystemModstamp` or a deletion log strategy.

**Common mistake:** Using a lookup instead of a master-detail when rollup summaries or cascade deletes are needed. Lookups cannot have rollup summary fields on the parent. Choosing the wrong relationship type at design time is expensive to fix in a managed package deployed to hundreds of orgs.

---

## 3. SOQL Fundamentals — Syntax, Limits, and Governors

SOQL is syntactically similar to SQL SELECT but with important restrictions: no `JOIN` keyword (relationships are traversed using dot notation and nested queries), no `UNION`, no subqueries in the `FROM` clause, and a 50,000-row result limit per query. The governor limits most relevant to ETL and Apex are:

| Context | SOQL query limit | Row limit per query |
|---------|-----------------|---------------------|
| Synchronous Apex | 100 queries | 50,000 rows |
| Asynchronous (Batch/Future/Queueable) | 200 queries | 50,000 rows |
| REST API (single request) | N/A (one query per call) | 2,000 rows (paginated via `nextRecordsUrl`) |

For ETL workloads, the REST API pagination model matters most. When `etl-bq-pipeline` queries `APP__Policy__c` for all modified records since the last run, it must iterate `nextRecordsUrl` pages until `done: true`. Failing to handle pagination causes silent data loss — the ETL loads only the first 2,000 records and marks the run complete.

A basic parameterized SOQL query on Policy records modified in the last 24 hours:

```soql
SELECT Id, Name, APP__EHR_Policy_Id__c, APP__Account__c,
       APP__Policy_Type__c, APP__Effective_Date__c,
       APP__Expiration_Date__c, SystemModstamp, LastModifiedDate
FROM APP__Policy__c
WHERE LastModifiedDate >= LAST_N_DAYS:1
ORDER BY LastModifiedDate ASC
```

`SystemModstamp` is updated by any write including system-level changes; `LastModifiedDate` is updated only by user-visible changes. ETL incremental loads typically use `SystemModstamp` to avoid missing formula-field recalculations and sharing-rule-triggered updates.

**Common mistake:** Omitting `ORDER BY` on paginated queries. Without a deterministic sort, Salesforce may return the same record on multiple pages or skip records if data changes during pagination. Always sort by `SystemModstamp ASC, Id ASC` for stable pagination.

---

## 4. Relationship Queries — Parent-to-Child and Child-to-Parent

SOQL traverses relationships in two directions. **Child-to-parent** uses dot notation to access parent fields:

```soql
SELECT Id, Name,
       APP__Account__r.Name,
       APP__Account__r.BillingState,
       APP__Policy_Type__r.Name
FROM APP__Policy__c
WHERE APP__Account__r.BillingState = 'CA'
  AND LastModifiedDate >= LAST_N_DAYS:7
```

The `__r` suffix replaces `__c` on the field to navigate the relationship. For standard object lookups, the relationship name is the field name without the `Id` suffix (e.g., `Account.OwnerId` → `Account.Owner.Name`).

**Parent-to-child** uses a nested SELECT (subquery). The child relationship name is defined in the object's metadata — for custom objects it is typically the plural API name:

```soql
SELECT Id, Name,
  (SELECT Id, APP__Line_Type__c, APP__Premium__c,
          APP__Effective_Date__c
   FROM APP__Lines__r
   WHERE APP__Line_Status__c = 'Active')
FROM APP__Policy__c
WHERE APP__Expiration_Date__c >= TODAY
  AND APP__Account__r.Type = 'Customer'
```

Nested subqueries return up to 200 child records per parent. If a Policy can have more than 200 Lines, the subquery result is truncated and you must issue a separate, filtered query on `APP__Line__c` for those specific Policy Ids. The ETL pipelines in `etl-bq-pipeline` typically query child objects separately and join in BigQuery rather than relying on nested SOQL, which avoids the 200-row subquery limit.

**Common mistake:** Assuming the child relationship name is simply the object's API name. The relationship name is a separate metadata attribute. For `APP__Line__c` with a master-detail to `APP__Policy__c`, the relationship name might be `APP__Lines__r` but it could also be something custom — always verify in Schema Builder or the object's metadata XML.

---

## 5. Semi-Joins and Anti-Joins

Semi-joins (`IN` with a subquery) and anti-joins (`NOT IN` with a subquery) allow filtering based on the existence or absence of related records without fetching those records into the main result:

```soql
-- Policies that have at least one active Line (semi-join)
SELECT Id, Name, APP__EHR_Policy_Id__c
FROM APP__Policy__c
WHERE Id IN (
  SELECT APP__Policy__c
  FROM APP__Line__c
  WHERE APP__Line_Status__c = 'Active'
)
AND LastModifiedDate >= LAST_N_DAYS:30

-- Accounts with no associated Opportunity in the last year (anti-join)
SELECT Id, Name, Type
FROM Account
WHERE Id NOT IN (
  SELECT AccountId
  FROM Opportunity
  WHERE CloseDate >= LAST_N_DAYS:365
)
AND Type = 'Customer'
```

Semi-joins are valuable in ETL scenarios where you need to identify records in one object that have or lack corresponding records in another — for example, finding APP__Policy__c records that have no corresponding `APP__Line__c` records, which would indicate a data integrity issue that the reconciliation job should flag.

The inner SELECT in a semi-join must return an Id field or a lookup field that references the outer object. You cannot use aggregate functions, relationship traversals, or LIMIT in the inner SELECT. The inner query is also subject to the 50,000-row limit independently.

**Common mistake:** Using semi-joins on non-selective inner queries. If the inner SELECT returns close to 50,000 records, Salesforce may downgrade performance or throw a `QUERY_TIMEOUT`. Pre-filter the inner query on indexed fields.

---

## 6. Aggregate Functions and GROUP BY

SOQL supports `COUNT()`, `COUNT(fieldName)`, `SUM()`, `AVG()`, `MIN()`, `MAX()`, and `COUNT_DISTINCT()`. Aggregate queries are useful for ETL reconciliation — verifying that the count of records in BigQuery matches Salesforce after a load — and for building summary datasets:

```soql
SELECT APP__Policy_Type__r.Name, APP__Account__r.BillingState,
       COUNT(Id) policyCount,
       SUM(APP__Total_Premium__c) totalPremium,
       MIN(APP__Effective_Date__c) earliestEffective,
       MAX(APP__Expiration_Date__c) latestExpiration
FROM APP__Policy__c
WHERE APP__Account__r.Type = 'Customer'
  AND APP__Policy_Status__c = 'Active'
GROUP BY APP__Policy_Type__r.Name, APP__Account__r.BillingState
HAVING SUM(APP__Total_Premium__c) > 10000
ORDER BY SUM(APP__Total_Premium__c) DESC
```

Aggregate queries return `AggregateResult` objects in Apex, accessed via `get('aliasName')`. In the REST API they return as JSON objects with aliased keys. Note that `COUNT()` (no argument) counts all rows including null fields, while `COUNT(fieldName)` counts only non-null values — a distinction that matters when counting sparse fields.

`GROUP BY ROLLUP` and `GROUP BY CUBE` are available for multi-dimensional aggregations. `GROUP BY ROLLUP(field1, field2)` adds subtotal rows automatically, which is useful for generating hierarchical summary reports without post-processing.

**Common mistake:** Mixing aggregate and non-aggregate fields without including all non-aggregate fields in the `GROUP BY` clause. SOQL enforces this strictly — any field in the SELECT that is not inside an aggregate function must appear in `GROUP BY`.

---

## 7. Query Optimization and Selective Queries

Salesforce's query optimizer evaluates whether a query is "selective" before executing it. A query is selective if the filter conditions use indexed fields and are estimated to return fewer than roughly 10% of the object's total records (thresholds vary by object size). Non-selective queries on large objects (>100k records) are queued and may time out, generating a `QUERY_TIMEOUT` error.

Key strategies for selectivity:

| Strategy | Example |
|----------|---------|
| Filter on indexed fields | `WHERE Id IN :idSet`, `WHERE SystemModstamp >= :lastRun`, `WHERE APP__EHR_Policy_Id__c = :externalId` |
| Combine indexed + non-indexed | Lead with indexed field first: `WHERE LastModifiedDate >= LAST_N_DAYS:1 AND APP__Policy_Status__c = 'Active'` |
| Avoid leading wildcards | `LIKE 'POL-%'` is better than `LIKE '%POL%'` |
| Use `LIMIT` when possible | Cap exploratory queries in Apex to avoid row-limit exceptions |
| Leverage External IDs for upsert | Indexed field, avoids full scan in upsert operations |

The `EXPLAIN` command is available in the Developer Console's Query Plan tool. It returns a cost estimate — values below 1 indicate a selective query; values above 1 indicate a likely full scan. Running EXPLAIN against any new ETL query before deploying to production is a critical discipline.

For the BigQuery ETL (`etl-bq-pipeline`), incremental load queries should always include `WHERE SystemModstamp >= :watermark` where `watermark` is stored in a state table. This ensures that even if a run is restarted, the query remains selective on the `SystemModstamp` index rather than scanning all records.

**Common mistake:** Not accounting for soft-deleted records. By default SOQL excludes records in the Recycle Bin. To include them, append `ALL ROWS` to the query. If the ETL needs to propagate deletes to BigQuery, it must use `ALL ROWS` with `IsDeleted = true` — but note that Recycle Bin records are permanently purged after 15 days, so the ETL must run at least every 15 days or use Salesforce's streaming/Change Data Capture API instead.

---

## 8. SOSL — Salesforce Object Search Language

SOSL performs a text search across multiple objects and fields in a single call, using Salesforce's full-text search index rather than the database query path. It is faster than multiple SOQL queries for cross-object keyword searches and supports relevance ranking:

```
FIND {Acme Insurance*} IN ALL FIELDS
RETURNING
  Account(Id, Name, BillingState WHERE Type = 'Customer'),
  Contact(Id, FirstName, LastName, Email WHERE AccountId != null),
  APP__Policy__c(Id, Name, APP__EHR_Policy_Id__c)
LIMIT 200
```

SOSL is appropriate when the search term comes from a user (type-ahead search, global search) or when you need to find a value across fields and objects without knowing which field it is in. It is inappropriate for ETL incremental loads because it does not support date-range filters or guarantees about completeness — the full-text index can lag real-time data by minutes.

SOSL returns a `List<List<SObject>>` in Apex, one inner list per RETURNING clause object. The total result limit is 2,000 records across all objects.

**Common mistake:** Using SOSL for ETL data extraction. SOSL is a search tool, not a data extraction tool. Its index may not include all records (new records take time to index), and it does not support `SystemModstamp` filtering. Always use SOQL for ETL workloads.

---

## 9. Reporting Snapshots

A Reporting Snapshot is a scheduled process that runs a summary report and saves the results as records on a custom object, creating a point-in-time historical dataset. This pattern solves a fundamental Salesforce limitation: standard reports always reflect current data, so trend analysis over time requires explicitly persisting snapshots.

To implement a Reporting Snapshot: (1) create a custom object to hold snapshot rows, (2) build a summary/matrix report with the metrics you want to capture, (3) configure the Reporting Snapshot to map report columns to custom object fields, (4) schedule it (daily, weekly, etc.). The snapshot records accumulate over time and can be queried with SOQL for trend analysis.

In the integration platform context, a Reporting Snapshot on active policy counts by `APP__Policy_Type__c` and `BillingState` provides historical data that would otherwise require ETL pipelines querying Salesforce at regular intervals and storing the results in BigQuery. For metrics that the business only needs weekly, Reporting Snapshots are a lower-complexity alternative to a full ETL pipeline.

```soql
-- Query accumulated snapshot data
SELECT SnapshotDate, PolicyType__c, BillingState__c, PolicyCount__c, TotalPremium__c
FROM PolicySnapshot__c
WHERE SnapshotDate >= LAST_N_DAYS:90
ORDER BY SnapshotDate ASC, PolicyType__c ASC
```

**Common mistake:** Scheduling Reporting Snapshots too frequently on large datasets. Each snapshot run creates records proportional to the report's row count. A daily snapshot on a 10,000-row report creates 3.65 million records per year, consuming data storage limits.

---

## 10. Integration Platform and AMS Package Object Model Deep Dive

The integration platform package centers on a **Policy hierarchy**: `Account` (client) → `APP__Policy__c` → `APP__Line__c` → `APP__Plan__c` → `APP__Rate__c`. Each level has a master-detail relationship to its parent, meaning cascade deletes propagate downward. The ETL must account for this: a delete on `APP__Policy__c` silently removes all descendant Lines, Plans, and Rates from Salesforce's Recycle Bin, which means the BigQuery side must apply the same cascade logic or it will retain orphaned child records.

The `APP__Servicing__c` object bridges a Policy to its servicing employees (`APP__Employee__c`), implementing a many-to-many through junction logic. `APP__Marketing_Submission__c` and `APP__Marketing_Line__c` form a parallel hierarchy tracking submissions to carriers, separate from the bound policy hierarchy.

Key cross-object query pattern used by ETL to extract a policy with its lines in one pass:

```soql
SELECT Id, Name, APP__EHR_Policy_Id__c,
       APP__Account__r.Id, APP__Account__r.Name,
       APP__Account__r.APP__EHR_Client_Id__c,
       APP__Policy_Type__r.Name, APP__Effective_Date__c,
       APP__Expiration_Date__c, APP__Policy_Status__c,
       SystemModstamp,
  (SELECT Id, APP__EHR_Line_Id__c, APP__Line_Type__c,
          APP__Premium__c, APP__Line_Status__c, SystemModstamp
   FROM APP__Lines__r
   WHERE APP__Line_Status__c != 'Cancelled')
FROM APP__Policy__c
WHERE SystemModstamp >= :watermarkTs
  AND APP__Policy_Status__c != 'Cancelled'
LIMIT 2000
```

The AMS package, with 134 custom objects, follows a similar hierarchy but covers the full agency management surface: coverage types, endorsements, billing transactions, claims, and producer commissions. The key ETL risk in the AMS package is **object cardinality** — objects like billing transactions can have millions of records, meaning any ETL query must be watermarked on `SystemModstamp` with extremely tight incremental windows to avoid scanning the full table.

The BigQuery schema mirrors the Salesforce object hierarchy as a set of denormalized fact tables and dimension tables. `app_policy` is the central fact table, joined to `app_account`, `app_line`, `app_plan`, and `app_rate` dimension tables via foreign keys matching the Salesforce Id fields. When ETL upserts a Line record, it must also verify that the parent Policy record exists in BigQuery — referential integrity that Salesforce enforces via master-detail but BigQuery does not.

**Common mistake:** Querying AMS package high-volume objects without a `LastModifiedDate` or `SystemModstamp` filter. Objects like billing transactions are write-heavy; even a 1-hour incremental window may return tens of thousands of records. Always verify the expected row count in a sandbox before scheduling ETL incremental jobs against production.

---

## 11. Key Concepts Summary

```
Salesforce Data Model
├── Object Types
│   ├── Standard (Account, Contact, Opportunity, Lead, Task)
│   └── Custom (*__c, namespace-prefixed in managed packages)
│       ├── APP__Policy__c
│       ├── APP__Line__c
│       ├── APP__Plan__c
│       ├── APP__Rate__c
│       ├── APP__Servicing__c
│       └── AMS package (134 objects)
│
├── Relationship Types
│   ├── Lookup (soft FK, nullable, no cascade)
│   ├── Master-Detail (hard FK, cascade delete, rollup summary)
│   ├── Many-to-Many (junction object with 2 master-detail fields)
│   └── Polymorphic Lookup (WhoId, WhatId, TYPEOF in SOQL)
│
├── Indexing
│   ├── Auto-indexed: Id, Name, OwnerId, CreatedDate,
│   │   LastModifiedDate, SystemModstamp
│   └── Custom-indexed: External ID, Unique fields
│
├── SOQL
│   ├── Child-to-parent: dot notation (__r)
│   ├── Parent-to-child: nested SELECT (subquery, max 200 rows)
│   ├── Semi-join / Anti-join: IN / NOT IN (subquery)
│   ├── Aggregate: COUNT, SUM, AVG, MIN, MAX, GROUP BY, HAVING
│   └── Optimization: EXPLAIN, selective filters, watermark pattern
│
├── SOSL
│   └── Full-text cross-object search (not for ETL)
│
└── Reporting Snapshots
    └── Scheduled report → custom object records (historical trend)
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the difference between a lookup relationship and a master-detail relationship in Salesforce, and what feature is only available on master-detail?

**2.** In SOQL, what suffix replaces `__c` when navigating a custom relationship in a query?

**3.** You query `APP__Policy__c` with no WHERE clause in an org with 600,000 policy records. What is likely to happen and why?

**4.** What does `SystemModstamp` capture that `LastModifiedDate` does not?

**5.** Explain the 200-row limit on nested SOQL subqueries and describe how the integration platform ETL should handle policies that have more than 200 lines.

**6.** Write a SOQL query using a semi-join to find all Accounts that have at least one active APP__Policy__c.

**7.** What is the purpose of the `EXPLAIN` command in the Developer Console's Query Plan tool, and what cost value indicates a non-selective query?

**8.** How does SOSL differ from SOQL, and why should you never use SOSL for ETL incremental loads?

**9.** What is a polymorphic lookup? Give a standard Salesforce example and explain how `TYPEOF` is used in SOQL.

**10.** In a master-detail hierarchy like Policy → Line → Plan, what happens in Salesforce when a Policy record is deleted? What must the BigQuery ETL do to stay consistent?

**11.** What are External ID fields and why are they important for ETL upsert operations in the integration platform?

**12.** Explain the `ALL ROWS` modifier in SOQL. When would an ETL pipeline need to use it?

**13.** You need to count the total premium grouped by policy type and billing state, but only for groups where total premium exceeds $50,000. Write the SOQL query.

**14.** What is a Reporting Snapshot, and what Salesforce limitation does it address?

**15.** Describe the "watermark pattern" for ETL incremental loads and explain why `SystemModstamp ASC, Id ASC` ordering is recommended.

**16.** What is the maximum number of rows a nested subquery (parent-to-child) can return in SOQL?

**17.** What is the difference between `COUNT()` and `COUNT(fieldName)` in SOQL aggregate queries?

**18.** In the integration platform object hierarchy, which object acts as a junction between a Policy and its servicing employees, and what relationship type does it likely use?

**19.** Why is it risky to query high-volume AMS package objects (like billing transactions) without a `SystemModstamp` filter? What could go wrong?

**20.** A developer adds a new non-indexed Text field `APP__Custom_Reference__c` on `APP__Policy__c` and uses it as the sole WHERE clause filter in an ETL query. Explain the performance implications and suggest a fix.

---

### Answers

??? note "Reveal Answers"

    **1.** A lookup relationship is a soft foreign key: the child record can exist independently, the lookup field is nullified (not deleted) when the parent is deleted, and the parent cannot have rollup summary fields aggregating child data. A master-detail relationship is a hard dependency: deleting the parent cascade-deletes all children, the child inherits the parent's sharing model, and the parent can use rollup summary fields (COUNT, SUM, MIN, MAX) over child records. Rollup summary fields are the key feature exclusive to master-detail.

    **2.** The `__c` suffix is replaced with `__r` when navigating a relationship in SOQL. For example, `APP__Policy__c.APP__Account__c` (the lookup field) becomes `APP__Policy__c.APP__Account__r.Name` to access the parent Account's Name field. For standard objects, the pattern is the relationship name without `Id` — `Opportunity.AccountId` becomes `Opportunity.Account.Name`.

    **3.** Salesforce will likely treat the query as non-selective because there is no WHERE clause filtering on indexed fields, meaning it must scan all 600,000 records. On very large objects, Salesforce may return a `QUERY_TIMEOUT` error, queue the query for offline processing, or reject it entirely if it exceeds governor limits. Even if it succeeds, the 50,000-row query limit would truncate the results to 50,000 records, silently omitting 550,000 records. Always include a selective WHERE clause, especially on high-volume objects.

    **4.** `SystemModstamp` is updated by any write to the record, including system-triggered changes such as sharing rule recalculations, workflow field updates, formula field recalculations triggered by related changes, and platform-level housekeeping updates. `LastModifiedDate` is only updated when a user (human or API call acting as a user) explicitly saves the record. ETL incremental loads should use `SystemModstamp` to avoid missing records that were modified by system processes, which would not update `LastModifiedDate`.

    **5.** Nested SOQL subqueries (parent-to-child) return a maximum of 200 child records per parent record. If a Policy has more than 200 Lines, the subquery result is silently truncated — Salesforce does not raise an error. The integration platform ETL should handle this by querying `APP__Line__c` separately, filtered by the Policy Ids retrieved in the parent query, rather than relying on nested subqueries. This is also more performant at scale because it allows the Line query to use its own `SystemModstamp` watermark independently.

    **6.**
    ```soql
    SELECT Id, Name, BillingState, Type
    FROM Account
    WHERE Id IN (
      SELECT APP__Account__c
      FROM APP__Policy__c
      WHERE APP__Policy_Status__c = 'Active'
    )
    ```
    This semi-join returns only Accounts that have at least one related Policy record in Active status. The inner SELECT must return a lookup field (here `APP__Account__c`) that references the outer object (Account). No aggregate functions, LIMIT, or relationship traversal are allowed in the inner SELECT.

    **7.** The `EXPLAIN` command in the Developer Console's Query Plan tool returns a cost estimate for a SOQL query without executing it. The cost value represents how many rows Salesforce estimates will be scanned relative to using a full-table scan. A cost value below 1.0 indicates a selective query that will use an index efficiently. A cost value above 1.0 — especially values significantly greater than 1 — indicates a non-selective query that will scan a large portion of the table, leading to slow performance, timeouts, or governor limit failures in production.

    **8.** SOQL is a database query language that retrieves records based on field-level filters using the relational database path; it supports date filters, relationship traversal, aggregation, and is suitable for ETL. SOSL is a full-text search language that searches across multiple objects and fields simultaneously using Salesforce's text search index; it supports relevance ranking but not date-range filters. SOSL should never be used for ETL incremental loads because its index can lag real-time data by minutes, it does not guarantee completeness (newly created records may not be indexed yet), and it does not support `SystemModstamp` filtering, which is essential for reliable watermark-based incremental extraction.

    **9.** A polymorphic lookup is a relationship field that can reference records from more than one object type. The canonical Salesforce example is `Task.WhoId`, which can point to either a Contact or a Lead record. In SOQL, you use the `TYPEOF` expression in the SELECT clause to conditionally retrieve different fields depending on the referenced object's type: `TYPEOF Who WHEN Contact THEN FirstName, Email WHEN Lead THEN FirstName, Company END`. Without `TYPEOF`, you can only access fields that exist on all possible referenced object types (typically just Id and Name).

    **10.** When a Policy record is deleted in a master-detail hierarchy, Salesforce cascade-deletes all child Line records, which in turn cascade-deletes all grandchild Plan records, and so on down the hierarchy. These deleted records move to the Recycle Bin and are permanently purged after 15 days. The BigQuery ETL must detect these deletions and propagate them to BigQuery. The recommended approach is to query with `ALL ROWS` and `IsDeleted = true` filtered by `SystemModstamp` within the current watermark window. If the ETL does not run within 15 days, those delete records will be permanently gone from Salesforce, requiring a full reconciliation query to detect orphaned records in BigQuery.

    **11.** External ID fields are custom fields marked as "External ID" in Salesforce metadata, which causes Salesforce to create a database index on that field. They are used in ETL upsert operations to match incoming records by a business key (like `APP__EHR_Policy_Id__c`) rather than by Salesforce's internal Id. This allows the ETL to insert a new record if no match is found, or update the existing record if a match is found, all in a single API call. Without an External ID field, the ETL would need to first query for the Salesforce Id, then decide whether to insert or update, doubling the API call count and introducing race conditions.

    **12.** The `ALL ROWS` modifier in SOQL includes soft-deleted records (records currently in the Recycle Bin) and archived records in the query results, which are normally excluded from standard queries. An ETL pipeline needs `ALL ROWS` when it must propagate hard deletes from Salesforce to a downstream system like BigQuery. The query `SELECT Id, IsDeleted FROM APP__Policy__c WHERE IsDeleted = true AND SystemModstamp >= :watermark ALL ROWS` returns recently deleted Policy records that need to be removed from BigQuery. Without `ALL ROWS`, deleted records are invisible to SOQL, and BigQuery would silently retain stale data indefinitely.

    **13.**
    ```soql
    SELECT APP__Policy_Type__r.Name, APP__Account__r.BillingState,
           COUNT(Id) policyCount,
           SUM(APP__Total_Premium__c) totalPremium
    FROM APP__Policy__c
    WHERE APP__Policy_Status__c = 'Active'
    GROUP BY APP__Policy_Type__r.Name, APP__Account__r.BillingState
    HAVING SUM(APP__Total_Premium__c) > 50000
    ORDER BY SUM(APP__Total_Premium__c) DESC
    ```
    The `HAVING` clause filters groups after aggregation (analogous to SQL's HAVING). Both the grouped-by relationship fields must appear in the GROUP BY clause. Non-aggregate fields in SELECT must all appear in GROUP BY.

    **14.** A Reporting Snapshot is a Salesforce platform feature that runs a summary or tabular report on a schedule and saves each row of the report results as a record on a designated custom object, creating a historical log of point-in-time data. It addresses the limitation that Salesforce standard reports always show current data — there is no built-in historical trend for report metrics. By accumulating snapshot records over time, you can query the custom object with SOQL to analyze trends, build dashboards showing changes over weeks or months, and maintain an audit trail of key metrics without building a custom ETL pipeline.

    **15.** The watermark pattern stores the timestamp of the last successful ETL run in a persistent state store (a database table, a file, or a Salesforce custom setting). Each ETL run queries `WHERE SystemModstamp >= :storedWatermark` to retrieve only records modified since the last run, then updates the watermark to the current timestamp upon success. `ORDER BY SystemModstamp ASC, Id ASC` is recommended because it produces a deterministic, stable sort — if pagination is required across multiple API calls, the sort ensures no records are skipped or duplicated even if new records are written during the ETL run. Without `Id ASC` as a tiebreaker, records with identical `SystemModstamp` values may appear in different orders across pages.

    **16.** A nested SOQL subquery (parent-to-child relationship query) returns a maximum of 200 child records per parent record. This is a hard platform limit that cannot be raised. If more than 200 children exist for a parent, the subquery result is silently truncated — no error is raised and no indicator is provided that records were omitted. For ETL pipelines that need all child records regardless of count, the recommended pattern is to query child objects separately using the parent Id as a filter rather than relying on nested subqueries.

    **17.** `COUNT()` with no argument counts all rows in the result set including rows where fields have null values — it is equivalent to `COUNT(*)` in SQL. `COUNT(fieldName)` counts only rows where the specified field is non-null, ignoring records where that field is empty. The difference matters when counting sparse optional fields: `COUNT(APP__Total_Premium__c)` would exclude Policies where premium has not been set, while `COUNT(Id)` always counts every Policy record in the group. Always choose the form that reflects your actual metric to avoid misleading counts in reports and reconciliation checks.

    **18.** `APP__Servicing__c` acts as the junction between a Policy (or client Account) and its servicing employees (`APP__Employee__c`). It likely uses either two master-detail fields (to Policy and Employee) or a master-detail to Policy and a lookup to Employee, depending on whether the Employee is managed within the integration platform package or sourced from an ETL-only sync. If both relationships are master-detail, the Servicing record is deleted when either parent is deleted, which the ETL must account for by detecting those deletions via `ALL ROWS` queries and propagating them to BigQuery's servicing dimension table.

    **19.** High-volume AMS package objects like billing transactions can contain millions of records accumulated over years of agency operations. Without a `SystemModstamp` filter, the SOQL query has no selective WHERE clause on an indexed field, causing Salesforce to scan the entire table. This will likely trigger a `QUERY_TIMEOUT` error in production, exceed the 50,000-row query limit (silently truncating results), and consume enormous API call quotas during pagination. In the worst case, a misconfigured ETL job could hit daily API limits for the entire org, blocking all other integrations. Always validate the expected row count in a sandbox using `SELECT COUNT() FROM Object WHERE SystemModstamp >= :window` before deploying any ETL query against production.

    **20.** A non-indexed Text field as the sole WHERE clause filter forces a full table scan on `APP__Policy__c`. With 600,000+ records in a production org, this query will be flagged as non-selective by Salesforce's query optimizer, leading to a `QUERY_TIMEOUT` or severely degraded performance. The fix has two parts: (1) mark `APP__Custom_Reference__c` as an External ID field (if values are unique) or a Unique field in the object's metadata, which causes Salesforce to create a database index on it; (2) if the field cannot be made an External ID (e.g., values are not unique), combine the filter with an already-indexed field like `WHERE LastModifiedDate >= LAST_N_DAYS:7 AND APP__Custom_Reference__c = :value` so the optimizer can use the `LastModifiedDate` index to limit the scan before applying the non-indexed filter.

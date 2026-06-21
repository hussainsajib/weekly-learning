# Week 23 — BigQuery & Data Warehousing

**Week of:** November 9, 2026
**Estimated study time:** ~2 hours
**Tags:** `bigquery` `gcp` `data-warehouse` `sql`

---

## Overview

BigQuery is Google Cloud's fully managed, serverless data warehouse, but understanding *why* it performs the way it does requires looking beneath the abstractions. At its core, BigQuery separates storage from compute: data lives in Google's distributed file system (Colossus) in a proprietary columnar format, while queries are executed by Dremel — a massively parallel query execution engine that can fan out work across thousands of workers in seconds. This architecture means BigQuery scales to petabytes without any infrastructure tuning on your part, but it also means that naive SQL patterns that work fine in PostgreSQL or MySQL can become enormously expensive or slow.

For an engineer working on the `etl-appliedcrm-bq` pipeline, this matters in concrete ways. Every Salesforce object sync you write touches BigQuery tables that either pay by bytes scanned or compete for reservation slots. The difference between a query that scans 10 MB and one that scans 10 GB is often a single missing partition filter. Choosing between streaming inserts (low-latency, higher cost) and batch loads (high-throughput, lower cost) directly determines how fresh your analytics data is and what it costs at month-end. Designing your schema as a normalized star schema versus a denormalized OBT (One Big Table) affects not just performance but whether downstream analysts can actually use the data without joining nightmares.

This week covers the internals that explain BigQuery's behavior, the SQL techniques (window functions, analytical queries) that make it powerful, and the design patterns (SCD Type 2, partitioning, clustering) that separate a working ETL from a production-grade one. You already own BQ schema design for the AESF platform — this guide is meant to sharpen the "why" behind decisions you likely already make by intuition, and to surface a few traps that bite even experienced BQ practitioners.

By the end of this session you should be able to reason about slot utilization, write correct SCD Type 2 merge logic in BigQuery SQL, explain when to use streaming inserts versus batch loads in the context of Salesforce sync events, and choose a partition and cluster strategy for any Salesforce object table with confidence.

---

## 1. BigQuery Internals: Columnar Storage and Dremel

### Columnar Storage

Traditional row-oriented databases store each record contiguously. A query asking for one column out of fifty still reads the entire row. BigQuery stores data in Capacitor, a columnar format built on top of Colossus (Google's distributed file system). Each column is stored separately, compressed independently, and read independently. A query that touches three columns out of fifty reads roughly 6% of the physical data.

This has two major implications. First, compression ratios are dramatically better — columns of the same type with similar values (like a Salesforce `RecordType` field with ten possible values) compress extremely efficiently with dictionary or run-length encoding. Second, BigQuery's billing model (on-demand: $6.25/TiB scanned) is directly tied to how much data your queries physically read — so schema design and query writing are financial decisions, not just performance ones.

### Dremel Execution Engine

When you submit a query, BigQuery compiles it into a Dremel execution tree with three layers:

1. **Root server** — receives the query, dispatches to mixers
2. **Mixer nodes** — aggregate and re-dispatch to leaf nodes
3. **Leaf nodes (slots)** — read storage shards and execute local computation

Shuffles between stages move data between nodes and are the primary bottleneck in complex queries. Wide joins, large GROUP BY aggregations, and CROSS JOINs all generate expensive shuffles. The Query Execution Graph in the BigQuery console visualizes this tree and shows per-stage input/output bytes — invaluable for diagnosing slow queries.

```sql
-- Example: force a partitioned scan (Dremel reads only matching file shards)
SELECT
  account_id,
  COUNT(*) AS sync_count
FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
WHERE DATE(created_at) BETWEEN '2026-10-01' AND '2026-10-31'
GROUP BY account_id;
```

> **Common mistake:** Running `WHERE EXTRACT(MONTH FROM created_at) = 10` instead of a range filter. Applying a function to a partition column prevents partition pruning — BigQuery must scan all partitions. Always filter with range comparisons on the raw column.

---

## 2. Partitioning and Clustering

### Partitioning

Partitioning divides a table into segments based on a column value. BigQuery supports three partition types:

| Type | Column | Best for |
|------|--------|----------|
| Ingestion time | `_PARTITIONTIME` | Append-only event streams |
| Date/Timestamp | Any DATE or TIMESTAMP column | Tables with a natural time axis |
| Integer range | Any INT64 column | IDs with known ranges |

For your Salesforce object tables in `etl-appliedcrm-bq`, the natural partition column is almost always the record's `LastModifiedDate` or your ETL's `load_timestamp`. Date partitioning on `LastModifiedDate` means incremental sync queries that filter `WHERE LastModifiedDate >= @watermark` scan only the relevant day partitions instead of the full table.

```sql
-- Create a partitioned table for SF Opportunity syncs
CREATE TABLE `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
(
  opportunity_id        STRING NOT NULL,
  account_id            STRING,
  name                  STRING,
  stage_name            STRING,
  close_date            DATE,
  amount                NUMERIC,
  last_modified_date    TIMESTAMP,
  load_timestamp        TIMESTAMP,
  _is_deleted           BOOL DEFAULT FALSE
)
PARTITION BY DATE(load_timestamp)
OPTIONS (
  partition_expiration_days = 730,
  require_partition_filter = TRUE
);
```

Setting `require_partition_filter = TRUE` enforces that every query supplies a partition predicate — it prevents accidental full-table scans from analysts who forget to filter.

### Clustering

Clustering sorts data within each partition by up to four columns. Queries that filter or group by cluster columns benefit from block-level pruning: BigQuery skips entire storage blocks that cannot contain matching rows.

```sql
-- Cluster by columns that analysts filter/join on most
CREATE TABLE `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
(
  -- ... same columns ...
)
PARTITION BY DATE(load_timestamp)
CLUSTER BY account_id, stage_name, close_date;
```

For Salesforce objects, a good clustering strategy is: primary lookup key first (e.g., `account_id`), then the most common filter dimension (e.g., `stage_name`, `record_type_id`). Clustering is free and maintained automatically on writes.

> **Common mistake:** Over-clustering with high-cardinality columns like `opportunity_id` as the first cluster key. Clustering is only effective when queries filter on cluster columns with *low-to-medium cardinality*. A UUID primary key as cluster key provides almost no pruning benefit because each block contains only unique values.

---

## 3. Slot-Based vs. On-Demand Billing

### On-Demand Billing

On-demand pricing charges $6.25 per TiB of data processed. The first 1 TiB per month is free. Queries get access to a shared pool of up to 2,000 concurrent slots. This model is ideal for development, ad-hoc analysis, and workloads with unpredictable query patterns.

For `etl-appliedcrm-bq` on-demand projects (QA: `aedl-ssa-9999999996`, Data: `aedl-ssa-9999990033`), the cost pressure is primarily about minimizing bytes scanned — hence the importance of partition filters and column selection.

### Slot-Based / Capacity Billing

Capacity pricing purchases dedicated slots (BigQuery Editions: Standard, Enterprise, Enterprise Plus). Slots are virtual CPUs that execute query work. 100 slots at Standard edition cost ~$2,000/month flat. Queries are no longer billed by bytes scanned — you pay for reserved compute capacity regardless of utilization.

This model makes sense when: (a) monthly on-demand costs consistently exceed reservation costs, (b) you need query latency guarantees (shared pool can throttle), or (c) you run long-running ETL jobs that would be expensive on-demand.

```python
# Python BigQuery client: check bytes processed before running expensive queries
from google.cloud import bigquery

client = bigquery.Client(project="aedl-ssa-1668620072")

job_config = bigquery.QueryJobConfig(
    dry_run=True,
    use_query_cache=False
)

query = """
SELECT * FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
WHERE DATE(load_timestamp) >= '2026-01-01'
"""

job = client.query(query, job_config=job_config)
print(f"Estimated bytes: {job.total_bytes_processed / 1e9:.2f} GB")
print(f"Estimated cost:  ${job.total_bytes_processed / 1e12 * 6.25:.4f}")
```

> **Common mistake:** Running capacity reservations in a project where most queries are exploratory one-offs. Reservations require consistent high utilization to be cost-effective. If slot utilization is below ~60% during business hours, on-demand is almost always cheaper.

---

## 4. Window Functions and Analytical SQL

Window functions execute a calculation across a set of rows related to the current row *without collapsing rows into groups*. They are the single most powerful feature for analytics SQL and essential for SCD logic, ranking, and running totals.

### Core Syntax

```sql
function_name(expression) OVER (
  [PARTITION BY partition_expression]
  [ORDER BY sort_expression]
  [frame_clause]
)
```

### Common Patterns in BQ ETL Context

```sql
-- 1. Row number: identify the latest sync record per opportunity
SELECT
  opportunity_id,
  stage_name,
  load_timestamp,
  ROW_NUMBER() OVER (
    PARTITION BY opportunity_id
    ORDER BY load_timestamp DESC
  ) AS rn
FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
WHERE DATE(load_timestamp) >= '2026-01-01';

-- 2. LAG: detect field changes between syncs (for SCD Type 2)
SELECT
  opportunity_id,
  stage_name,
  LAG(stage_name) OVER (
    PARTITION BY opportunity_id
    ORDER BY load_timestamp
  ) AS prev_stage_name,
  load_timestamp
FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`;

-- 3. Running total of opportunity amounts per account per month
SELECT
  account_id,
  close_date,
  amount,
  SUM(amount) OVER (
    PARTITION BY account_id, DATE_TRUNC(close_date, MONTH)
    ORDER BY close_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_monthly_total
FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
WHERE stage_name = 'Closed Won';

-- 4. NTILE: bucket policies by premium size for analysis
SELECT
  policy_id,
  total_premium,
  NTILE(4) OVER (ORDER BY total_premium) AS premium_quartile
FROM `aedl-ssa-1668620072.salesforce_sync.sf_policy`;
```

### Frame Clauses

| Frame | Meaning |
|-------|---------|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Running total from start |
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | Rolling 7-row window |
| `RANGE BETWEEN INTERVAL 30 DAY PRECEDING AND CURRENT ROW` | Rolling 30-day window by value |

> **Common mistake:** Mixing `RANGE` and `ROWS` frames without understanding the difference. `ROWS` is positional (counts rows). `RANGE` is value-based (includes all rows with the same ORDER BY value). For time-based rolling windows, `RANGE BETWEEN INTERVAL N DAY PRECEDING` is usually what you want, not `ROWS BETWEEN N PRECEDING`.

---

## 5. Slowly Changing Dimensions (SCD)

A slowly changing dimension tracks how a dimension attribute changes over time. For your pipeline, this is critical for objects like `AESF__Policy__c` (policy status changes over time) and `Account` (address, assigned rep changes).

### SCD Type 1 — Overwrite
Just update in place. No history preserved. Simplest, but you lose the audit trail. Appropriate for fields where history genuinely does not matter (e.g., a data-quality correction).

### SCD Type 2 — Full History (most important)

SCD Type 2 creates a new row for each change and tracks validity periods. Every row gets `valid_from`, `valid_to`, and `is_current` columns.

```sql
-- SCD Type 2 merge for AESF__Policy__c
MERGE `aedl-ssa-1668620072.salesforce_sync.sf_policy_dim` AS target
USING (
  -- Incoming batch from Salesforce sync
  SELECT
    policy_id,
    status,
    effective_date,
    expiration_date,
    total_premium,
    CURRENT_TIMESTAMP() AS load_ts
  FROM `aedl-ssa-1668620072.salesforce_sync.sf_policy_staging`
) AS source
ON target.policy_id = source.policy_id
   AND target.is_current = TRUE

-- Close the old row when any tracked field changes
WHEN MATCHED AND (
  target.status            != source.status            OR
  target.total_premium     != source.total_premium     OR
  target.expiration_date   != source.expiration_date
) THEN UPDATE SET
  target.valid_to    = source.load_ts,
  target.is_current  = FALSE

-- Insert new current row (also handles new records via NOT MATCHED)
WHEN NOT MATCHED BY TARGET THEN INSERT (
  policy_id, status, effective_date, expiration_date, total_premium,
  valid_from, valid_to, is_current
) VALUES (
  source.policy_id, source.status, source.effective_date,
  source.expiration_date, source.total_premium,
  source.load_ts, TIMESTAMP('9999-12-31'), TRUE
);

-- Insert the new version for MATCHED-but-changed rows
-- (BigQuery MERGE cannot insert and update in the same MATCHED clause,
--  so use a follow-up INSERT ... SELECT for the new current rows)
INSERT INTO `aedl-ssa-1668620072.salesforce_sync.sf_policy_dim`
SELECT
  s.policy_id, s.status, s.effective_date, s.expiration_date, s.total_premium,
  s.load_ts AS valid_from,
  TIMESTAMP('9999-12-31') AS valid_to,
  TRUE AS is_current
FROM `aedl-ssa-1668620072.salesforce_sync.sf_policy_staging` s
JOIN `aedl-ssa-1668620072.salesforce_sync.sf_policy_dim` d
  ON s.policy_id = d.policy_id
  AND d.valid_to != TIMESTAMP('9999-12-31')   -- row we just closed
  AND d.is_current = FALSE
  AND d.valid_to = s.load_ts;                 -- only the rows closed in this batch
```

> **Common mistake:** Forgetting that BigQuery MERGE cannot simultaneously UPDATE (close old row) and INSERT (open new row) in a single MATCHED clause for the same source record. The pattern requires either a two-step merge+insert or using the `WHEN NOT MATCHED BY SOURCE` clause combined with a staging join. Always test SCD logic with a small synthetic dataset before running on production data.

---

## 6. Star Schema vs. One Big Table (OBT)

### Star Schema

The classic data warehouse pattern: a central fact table surrounded by normalized dimension tables joined at query time.

```
sf_policy_fact ──── sf_account_dim
       │
       ├────────── sf_opportunity_dim
       │
       └────────── sf_employee_dim (servicing rep)
```

**Pros:** Reduced storage (dimensions stored once), easier to update dimensions independently, cleaner semantics.  
**Cons:** Every query requires JOINs; in BigQuery, large shuffles on JOINs are expensive.

### One Big Table (OBT)

Pre-join everything into a single wide, denormalized table. Analysts query one table with no joins.

**Pros:** Faster queries (no shuffle from joins), simpler for analysts, BigQuery's columnar storage means unused columns don't cost anything at query time.  
**Cons:** Storage duplication, dimension updates require rewriting fact rows, harder to maintain.

### Practical Recommendation for AESF BQ

Use a **hybrid approach**: denormalize the most frequently joined dimensions (account name, stage name, record type label) into the fact table as redundant columns, while keeping full dimension tables for complex lookups. This gives you join-free performance on the 90% case while preserving correctness.

```sql
-- Hybrid OBT: policy fact with denormalized account name
CREATE TABLE `aedl-ssa-1668620072.salesforce_sync.policy_fact` AS
SELECT
  p.policy_id,
  p.account_id,
  a.name                    AS account_name,      -- denormalized
  a.billing_state           AS account_state,     -- denormalized
  p.status,
  p.effective_date,
  p.total_premium,
  p.load_timestamp
FROM `aedl-ssa-1668620072.salesforce_sync.sf_policy` p
LEFT JOIN `aedl-ssa-1668620072.salesforce_sync.sf_account` a
  ON p.account_id = a.account_id
  AND a.is_current = TRUE;
```

> **Common mistake:** Building a pure star schema in BigQuery and then wondering why dashboard queries are slow. BigQuery's shuffle cost for large joins can be significant. The 1990s wisdom of "always normalize" was designed for row-store databases with index lookups — columnar BQ responds very differently to schema design choices.

---

## 7. Streaming Inserts vs. Batch Load

### Streaming Inserts

The BigQuery Storage Write API (or legacy streaming API) allows row-by-row or micro-batch insertion with data available for queries within seconds.

**Use when:** You need near-real-time data availability. For example, `AESF__Marketing_Submission__c` changes that should be visible in dashboards within minutes of the Salesforce trigger firing.

**Cost:** ~$0.01 per 200 MB inserted (Storage Write API committed mode is cheaper than legacy streaming).

**Limitations:** Streaming rows go into a streaming buffer that is not immediately accessible to `TABLE_DATE_RANGE` queries or table exports. Best-effort deduplication only (use `insertId` for idempotency).

```python
from google.cloud import bigquery_storage_v1
from google.protobuf import descriptor_pool
import json

# Modern approach: Storage Write API with committed mode
client = bigquery_storage_v1.BigQueryWriteClient()

# For simpler use cases, the standard BQ client streaming insert:
from google.cloud import bigquery

bq = bigquery.Client(project="aedl-ssa-1668620072")
table_id = "aedl-ssa-1668620072.salesforce_sync.sf_marketing_submission"

rows = [
    {
        "submission_id": "a0B123",
        "account_id": "0015000001AbCdE",
        "status": "Submitted",
        "load_timestamp": "2026-11-09T14:32:00Z",
        "insert_id": "a0B123-2026-11-09T14:32:00Z"  # for dedup
    }
]

errors = bq.insert_rows_json(table_id, rows)
if errors:
    raise RuntimeError(f"Streaming insert errors: {errors}")
```

### Batch Load

Load jobs read from Cloud Storage (CSV, JSON, Avro, Parquet, ORC) and write directly to managed storage. No per-row cost — loads are free.

**Use when:** You run scheduled ETL windows (hourly, daily). For `etl-appliedcrm-bq`'s Pentaho jobs that run on a schedule, batch load from a GCS staging bucket is almost always the right choice.

```python
from google.cloud import bigquery, storage
import pandas as pd

def batch_load_to_bq(df: pd.DataFrame, table_id: str, project: str):
    """
    Write a Pandas DataFrame to BigQuery via Parquet load job.
    Parquet preserves types (no string-date ambiguity) and compresses well.
    """
    client = bigquery.Client(project=project)
    gcs_client = storage.Client(project=project)

    # Write to GCS staging bucket as Parquet
    bucket_name = f"{project}-etl-staging"
    blob_name = f"sf_sync/{table_id.split('.')[-1]}/load_{pd.Timestamp.now().strftime('%Y%m%dT%H%M%S')}.parquet"

    bucket = gcs_client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    blob.upload_from_string(df.to_parquet(index=False), content_type="application/octet-stream")

    gcs_uri = f"gs://{bucket_name}/{blob_name}"

    job_config = bigquery.LoadJobConfig(
        source_format=bigquery.SourceFormat.PARQUET,
        write_disposition=bigquery.WriteDisposition.WRITE_APPEND,
        autodetect=False,          # never autodetect in production
    )

    load_job = client.load_table_from_uri(gcs_uri, table_id, job_config=job_config)
    load_job.result()  # blocks until complete

    table = client.get_table(table_id)
    print(f"Loaded {table.num_rows} total rows into {table_id}")
```

### Decision Matrix

| Criteria | Streaming Insert | Batch Load |
|----------|-----------------|------------|
| Data freshness needed | < 5 minutes | > 15 minutes |
| Volume per sync | Low (< 10K rows) | High (> 100K rows) |
| Cost sensitivity | Higher | Lower (free) |
| Schema flexibility | Needs stable schema | Same |
| AESF use case | Real-time trigger events | Scheduled Pentaho ETL |

> **Common mistake:** Using streaming inserts for bulk historical backfills. A one-time load of 50M historical Opportunity records via streaming inserts costs ~$2,500 in insert fees. The same load via Parquet files in GCS costs $0 for the load job itself (you pay only for GCS storage and BigQuery storage after load).

---

## 8. Partition and Cluster Strategy for Salesforce Object Tables

### Recommended Strategy by Object Type

| Object | Partition Column | Cluster Columns | Rationale |
|--------|-----------------|-----------------|-----------|
| `sf_opportunity` | `DATE(load_timestamp)` | `account_id, stage_name` | Incremental syncs filter by load date; analysts filter by account and stage |
| `sf_policy` | `DATE(effective_date)` | `account_id, status` | Policy queries are date-range heavy; most dashboards filter by status |
| `sf_account` | `DATE(load_timestamp)` | `billing_state, record_type_id` | Geography and type are primary slice dimensions |
| `sf_contact` | `DATE(load_timestamp)` | `account_id` | Always joined to account |
| `sf_activity` | `DATE(activity_date)` | `account_id, activity_type` | Time-series analytics; high volume |
| `sf_marketing_submission` | `DATE(created_date)` | `account_id, status` | Status funnels, account lookups |

### Partition Expiration

Configure partition expiration to automatically drop old partitions. For staging/QA projects this controls cost; for production set a longer window or no expiration.

```sql
ALTER TABLE `aedl-ssa-9999999996.salesforce_sync.sf_opportunity`
SET OPTIONS (
  partition_expiration_days = 365
);
```

### Checking Partition Metadata

```sql
-- Inspect partition sizes and row counts
SELECT
  partition_id,
  total_rows,
  total_logical_bytes / pow(1024, 3) AS size_gb,
  last_modified_time
FROM `aedl-ssa-1668620072.salesforce_sync.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'sf_opportunity'
ORDER BY partition_id DESC
LIMIT 30;
```

> **Common mistake:** Partitioning a low-volume table (< 1 GB total) by day. Partitioning adds metadata overhead and a minimum partition size cost (~10 MB per partition). For small tables, a single unpartitioned table scanned in full is cheaper and faster than a table with hundreds of tiny partitions.

---

## 9. Query Optimization Patterns

### Avoid SELECT *

Always name columns explicitly. In columnar storage, `SELECT *` reads every column's data, negating the primary benefit of the format.

```sql
-- Bad: reads all 80 columns
SELECT * FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`;

-- Good: reads only 4 columns
SELECT opportunity_id, account_id, stage_name, amount
FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
WHERE DATE(load_timestamp) = CURRENT_DATE();
```

### Use APPROX functions for large aggregations

```sql
-- APPROX_COUNT_DISTINCT uses HyperLogLog — 1% error, 10x cheaper for large tables
SELECT
  stage_name,
  APPROX_COUNT_DISTINCT(account_id) AS approx_unique_accounts
FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
WHERE DATE(load_timestamp) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY stage_name;
```

### Filter before joining

```sql
-- Materialize filtered subsets before joining (reduces shuffle size)
WITH recent_opportunities AS (
  SELECT opportunity_id, account_id, amount
  FROM `aedl-ssa-1668620072.salesforce_sync.sf_opportunity`
  WHERE DATE(load_timestamp) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND stage_name = 'Closed Won'
),
active_accounts AS (
  SELECT account_id, name, billing_state
  FROM `aedl-ssa-1668620072.salesforce_sync.sf_account`
  WHERE is_current = TRUE
    AND billing_state IN ('CA', 'NY', 'TX')
)
SELECT
  a.name,
  a.billing_state,
  SUM(o.amount) AS total_won
FROM recent_opportunities o
JOIN active_accounts a USING (account_id)
GROUP BY a.name, a.billing_state
ORDER BY total_won DESC;
```

> **Common mistake:** Using `NOT IN` with a subquery. If the subquery returns any NULL values, `NOT IN` returns NULL (i.e., no rows match). Use `NOT EXISTS` or `LEFT JOIN ... WHERE key IS NULL` instead. This is a subtle SQL standard behavior that causes silent data loss in ETL deduplication logic.

---

## 10. Key Concepts Summary

```
BigQuery & Data Warehousing
│
├── Storage Layer
│   ├── Capacitor (columnar format on Colossus)
│   ├── Separate from compute (storage-compute decoupling)
│   └── Compression: dictionary, RLE, delta encoding per column
│
├── Execution: Dremel
│   ├── Root → Mixers → Leaf nodes (slots)
│   ├── Shuffle = primary bottleneck
│   └── Query Execution Graph → diagnose stages
│
├── Schema Design
│   ├── Partitioning (date, timestamp, integer range)
│   │   └── require_partition_filter = TRUE (guard clause)
│   ├── Clustering (up to 4 columns, low-medium cardinality)
│   └── Star Schema vs OBT → hybrid for BQ
│
├── Billing Models
│   ├── On-demand: $6.25/TiB scanned → minimize bytes read
│   └── Slot reservation: flat capacity → maximize utilization
│
├── SQL Patterns
│   ├── Window functions (ROW_NUMBER, LAG, LEAD, SUM OVER)
│   ├── MERGE for SCD Type 2
│   └── CTE + filter-before-join for large joins
│
├── Ingestion
│   ├── Streaming → low latency, higher cost, dedup via insertId
│   └── Batch (GCS → Load Job) → free, high throughput
│
└── AESF BQ ETL Application
    ├── Partition: load_timestamp (incremental watermark queries)
    ├── Cluster: account_id first (primary join/filter key)
    ├── SCD Type 2: sf_policy_dim for policy history
    └── Batch load: Pentaho → GCS → BQ Load Job (scheduled ETL)
```

---

## Quiz — 20 Questions

### Questions

**1.** Why does BigQuery's columnar storage format result in lower costs for analytical queries compared to row-oriented databases?

**2.** What is Dremel, and what are its three layers?

**3.** You write `WHERE YEAR(load_timestamp) = 2026` on a date-partitioned table. Will BigQuery prune partitions? Why or why not?

**4.** What is the maximum number of cluster columns you can define on a BigQuery table, and what cardinality works best?

**5.** Explain the difference between on-demand billing and slot-based (capacity) billing in BigQuery. When would you choose each?

**6.** What does `ROW_NUMBER() OVER (PARTITION BY opportunity_id ORDER BY load_timestamp DESC)` return, and how would you use it to get the latest record per opportunity?

**7.** What is the difference between `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` and `RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW`?

**8.** Define SCD Type 2. What columns are typically added to a dimension table to support it?

**9.** Why can a single BigQuery MERGE statement not simultaneously update an existing row (close it) and insert a new version of that row?

**10.** What is the core trade-off between a star schema and a One Big Table (OBT) in BigQuery?

**11.** When should you use streaming inserts instead of batch load jobs in the context of the AESF ETL pipeline?

**12.** What is the `insertId` field in BigQuery streaming inserts, and what guarantee does it provide?

**13.** A Pentaho job exports 5 million Opportunity records to BigQuery. Which load format (CSV, JSON, Parquet) is generally preferred and why?

**14.** You have a `sf_policy` table partitioned by `DATE(effective_date)`. A query filters `WHERE account_id = '0015000001AbCdE'`. Will BigQuery use partition pruning?

**15.** What does `require_partition_filter = TRUE` do, and why is it useful for production Salesforce sync tables?

**16.** Why is `NOT IN (SELECT ...)` dangerous in SQL, and what is the safe alternative?

**17.** You want to find accounts where the assigned service rep changed in the last 30 days. Write the SQL approach using a window function.

**18.** What is the difference between `APPROX_COUNT_DISTINCT` and `COUNT(DISTINCT ...)` in BigQuery?

**19.** Describe the recommended partition and cluster strategy for the `sf_opportunity` table in `etl-appliedcrm-bq` and justify each choice.

**20.** A load job that appended records to `sf_policy` ran twice due to a Pentaho retry. How would you detect and remove duplicate rows?

---

### Answers

??? note "Reveal Answers"

    **1.** BigQuery's columnar format stores each column separately, so a query touching 5 out of 80 columns physically reads only those 5 columns' data — roughly 6% of total storage. Row-oriented databases read the full row width even if only one column is needed. Additionally, columnar data with similar values in the same column compresses much more efficiently than interleaved row data, reducing both storage cost and I/O at query time. In BigQuery's on-demand billing model, physical bytes scanned directly determines cost, making columnar selection the primary cost-control lever available to query authors.

    **2.** Dremel is Google's massively parallel query execution engine powering BigQuery. It has three layers: (1) the root server, which receives and parses the query and fans work out to mixers; (2) mixer nodes, which aggregate partial results from leaf nodes and can further decompose subtasks; and (3) leaf nodes (slots), which physically read storage shards and execute computation. The tree structure allows BigQuery to parallelize a query across thousands of workers simultaneously. Shuffles between tree levels — moving intermediate data between nodes — are the dominant bottleneck in complex multi-stage queries.

    **3.** No, BigQuery will not prune partitions. Partition pruning requires a direct comparison on the raw partition column (e.g., `WHERE DATE(load_timestamp) = '2026-01-01'` or a range filter). Wrapping the column in a function like `YEAR()` or `EXTRACT()` prevents the query optimizer from determining which partitions are relevant at planning time, forcing a full table scan across all partitions. The fix is to rewrite as `WHERE load_timestamp >= '2026-01-01' AND load_timestamp < '2027-01-01'` or `WHERE DATE(load_timestamp) BETWEEN '2026-01-01' AND '2026-12-31'`.

    **4.** BigQuery supports up to 4 cluster columns per table. Low-to-medium cardinality works best — columns with a manageable number of distinct values (e.g., `stage_name` with 10 values, `billing_state` with 50) allow BigQuery to skip entire storage blocks when filtering. High-cardinality columns like UUID primary keys provide almost no pruning benefit because every storage block contains a unique spread of values, so no blocks can be safely skipped.

    **5.** On-demand billing charges $6.25 per TiB of data scanned — you pay per query based on bytes read, with no upfront commitment. Slot-based (capacity) billing charges a flat monthly rate for a reserved number of virtual CPUs (slots), regardless of bytes scanned. Choose on-demand for unpredictable workloads, development environments, and situations where minimizing per-query cost with good schema design is feasible. Choose slot reservations when monthly on-demand costs consistently exceed reservation cost, when you need latency guarantees (shared pool can throttle under contention), or when running long ETL jobs with high concurrency.

    **6.** `ROW_NUMBER()` assigns a sequential integer to each row within the partition (`opportunity_id`), ordered by `load_timestamp DESC` — so the most recent record for each opportunity gets `rn = 1`. To retrieve only the latest record per opportunity, wrap the window function in a CTE or subquery and filter `WHERE rn = 1`. This is the standard deduplication pattern for "latest snapshot" queries on append-only sync tables where the same `opportunity_id` appears multiple times across load batches.

    **7.** `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` is positional: it includes exactly the 6 rows immediately before the current row in ORDER BY sequence, regardless of their actual values — giving a 7-row rolling count. `RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW` is value-based: it includes all rows whose ORDER BY value falls within 6 days before the current row's date, which could be 0 rows or 100 rows depending on data density. For sparse time-series data, RANGE is usually correct; ROWS can silently compute rolling windows over the wrong number of calendar days if data has gaps.

    **8.** SCD Type 2 (Slowly Changing Dimension Type 2) tracks the full history of a dimension record by inserting a new row each time any tracked attribute changes, rather than overwriting. Typical additional columns are: `valid_from` (TIMESTAMP — when this version became effective), `valid_to` (TIMESTAMP — when this version was superseded; often `9999-12-31` for the current row), and `is_current` (BOOL — TRUE for the active record). Queries that need current state filter `WHERE is_current = TRUE`. Historical point-in-time queries join with `WHERE valid_from <= @as_of_date AND valid_to > @as_of_date`.

    **9.** A BigQuery MERGE statement processes each source row against target rows via its WHEN clauses sequentially. A single source record can match at most one WHEN MATCHED clause — it cannot trigger both an UPDATE (to close the old row) and an INSERT (to open a new row) in the same operation, because INSERT is a WHEN NOT MATCHED action. The standard workaround is a two-step process: first run a MERGE that only updates (closes) the changed rows, then run a separate INSERT...SELECT that reads the newly-closed rows and inserts fresh current rows. Alternatively, use a staging/swap pattern with CREATE TABLE AS SELECT.

    **10.** A star schema normalizes dimensions into separate tables, reducing storage redundancy and making dimension updates simple. However, every query against a star schema requires JOINs, which in BigQuery generate expensive shuffle operations between Dremel nodes. A One Big Table pre-joins everything, eliminating shuffle at query time but duplicating dimension data across every fact row. In BigQuery, because columnar storage means unused columns cost nothing at query time, the OBT's storage duplication is less harmful than in row-store databases, making a fully or partially denormalized OBT often faster and simpler for analysts — at the cost of more complex ETL to maintain consistency.

    **11.** Use streaming inserts when the downstream use case requires data to be queryable within seconds to minutes of the Salesforce event — for example, if a marketing submission status change needs to appear in a live dashboard within 5 minutes of the trigger. For all other cases in the AESF ETL pipeline — scheduled Pentaho jobs running hourly or daily syncs of Opportunity, Policy, Account, or Contact data — batch load from a GCS staging file is preferable. Batch loads are free (no per-row insert cost), support higher throughput, and are more straightforward to retry and audit. Streaming inserts cost approximately $0.01 per 200 MB and have subtler deduplication semantics.

    **12.** The `insertId` is an optional string field you provide when streaming rows to BigQuery. BigQuery uses it for best-effort deduplication: if the same `insertId` is received within a short deduplication window (typically a few minutes), BigQuery attempts to deduplicate the rows. It is *best-effort*, not guaranteed — BigQuery explicitly does not guarantee exactly-once delivery for streaming inserts. For true idempotency, use the Storage Write API in committed mode with `offset`-based exactly-once semantics, or design your downstream queries to use `ROW_NUMBER()` deduplication rather than relying on insert-time dedup.

    **13.** Parquet is generally preferred for BigQuery batch loads over CSV or JSON. Parquet is a columnar binary format that preserves native data types (TIMESTAMP stays TIMESTAMP, NUMERIC stays NUMERIC) without ambiguity — CSV requires type inference or schema specification, and JSON has verbose overhead and type coercion issues. Parquet files are also significantly smaller than equivalent CSV/JSON due to columnar compression, meaning cheaper GCS storage and faster load times. The BigQuery load job for Parquet is schema-aware and type-safe, reducing the risk of silent data corruption from string-to-date coercions common in CSV loads.

    **14.** No. Partition pruning is triggered only when the query filter references the *partition column* directly. The `sf_policy` table is partitioned by `DATE(effective_date)`, so only queries filtering on `effective_date` (or an expression involving it) can prune partitions. A filter on `account_id` alone does not help BigQuery determine which date partitions to skip. The query will scan all partitions. This is exactly where *clustering* on `account_id` helps — within each partition, BigQuery can skip blocks that don't contain the target `account_id` value, providing sub-partition pruning.

    **15.** `require_partition_filter = TRUE` is a table option that forces every query against the table to include a filter on the partition column. If a query omits the partition filter, BigQuery rejects it with an error before scanning any data. This is valuable for production Salesforce sync tables for two reasons: it prevents runaway cost from accidental full-table scans (e.g., an analyst who writes `SELECT * FROM sf_policy` without a date filter), and it enforces a query discipline that keeps the table usable as it grows. It is especially important in environments like `aedl-ssa-1668620072` (staging) where on-demand billing applies.

    **16.** `NOT IN (SELECT ...)` returns NULL (meaning no rows) if the subquery returns any NULL values, because `x NOT IN (NULL, 'a', 'b')` evaluates to `UNKNOWN` for any value of `x`. This is a SQL standard behavior that causes silent data loss — rows that should be excluded based on the logic are instead simply dropped from the result set. The safe alternative is `NOT EXISTS (SELECT 1 FROM ... WHERE key = outer.key)`, which handles NULLs correctly, or a `LEFT JOIN ... WHERE inner_key IS NULL` anti-join pattern. In ETL deduplication logic, this bug can cause records to silently disappear from incremental loads.

    **17.** Use the `LAG()` window function to compare the current service rep against the previous one for each account, then filter for rows where the value changed: `SELECT account_id, assigned_rep, LAG(assigned_rep) OVER (PARTITION BY account_id ORDER BY load_timestamp) AS prev_rep, load_timestamp FROM sf_account WHERE DATE(load_timestamp) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)` wrapped in a CTE, then `WHERE assigned_rep != prev_rep OR (prev_rep IS NULL AND assigned_rep IS NOT NULL)` in the outer query. This identifies exactly which accounts had a rep change within the window and preserves the before/after values for audit purposes.

    **18.** `COUNT(DISTINCT column)` computes an exact distinct count, which in BigQuery requires a global sort/deduplication pass across all values — expensive on large tables because it cannot be parallelized without shuffling all distinct values to a single worker. `APPROX_COUNT_DISTINCT(column)` uses the HyperLogLog++ algorithm to produce an estimate with approximately 1% error, but runs in a single distributed pass with no global shuffle. For large tables (hundreds of millions of rows), `APPROX_COUNT_DISTINCT` can be 10-100x faster and significantly cheaper. Use the exact version only when the 1% error is unacceptable (e.g., billing reconciliation); use the approximate version for dashboards and exploratory analysis.

    **19.** Partition by `DATE(load_timestamp)` — the ETL's ingestion timestamp — rather than `effective_date` or `close_date`, because incremental sync queries always filter with `WHERE load_timestamp >= @last_watermark`, making load_timestamp the natural partition key for the ETL workflow. Cluster by `account_id` as the first cluster column because the vast majority of downstream joins and dashboard filters start from an account. Add `stage_name` as the second cluster column because stage-based funnels (e.g., "all Closed Won this month") are the most common analytical query pattern after account-level aggregations. This combination means most production queries hit only one or two date partitions and skip a large fraction of blocks within them.

    **20.** Use a `ROW_NUMBER()` deduplication query with a WRITE_TRUNCATE or MERGE to remove duplicates. The canonical pattern: `CREATE OR REPLACE TABLE sf_policy AS SELECT * EXCEPT(rn) FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY policy_id, load_timestamp ORDER BY _PARTITIONTIME DESC) AS rn FROM sf_policy) WHERE rn = 1`. The `PARTITION BY policy_id, load_timestamp` groups rows that are true duplicates (same record, same load batch), and `rn = 1` keeps only one. For large tables, prefer a MERGE-based deduplication into a separate staging table to avoid rewriting the entire table. Going forward, prevent duplicates at the Pentaho level by using idempotent load keys or switching to a MERGE-based upsert pattern instead of raw WRITE_APPEND.

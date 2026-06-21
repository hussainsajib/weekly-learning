# Week 22 — Data Engineering & ETL Patterns

**Week of:** November 2, 2026
**Estimated study time:** ~2 hours
**Tags:** `data-engineering` `etl` `pipelines`

---

## Overview

Data engineering is the discipline of designing, building, and operating the pipelines that move raw data from operational systems into analytical or downstream systems. At its core, every pipeline answers the same question: how do we reliably get data from A to B, with the right shape, at the right time, without losing anything or duplicating it? The answer requires thinking about extraction strategy (full vs. incremental), transformation placement (ETL vs. ELT), change detection (watermarks, CDC), and fault tolerance (idempotency, dead-letter queues).

Your two production pipelines — `etl-appliedcrm-bde` and `etl-appliedcrm-bq` — are real-world examples of both paradigms. The BDE pipeline is closer to classic ETL: Pentaho PDI jobs extract from Salesforce, transform records, and load them into the Epic BDE backend via the `aesf-py-middleware` REST API. The BQ pipeline is ELT-leaning: Salesforce data lands in BigQuery, and heavy transformations happen there in SQL. Both share the same operational challenges — schema drift, partial failures, idempotency, and orchestration.

Understanding patterns like watermarking, change data capture (CDC), and schema evolution moves you from "it works most of the time" to "it is provably correct and restartable." That shift is what distinguishes senior ETL work from staff-level data engineering. Staff engineers think about the failure modes before writing a single line of transform code, and they design pipelines that degrade gracefully rather than silently corrupting state.

This guide covers the complete vocabulary and pattern set for data engineering, grounded in the tools you already use: Python, PDI, PostgreSQL, BigQuery, and GKE. Every section connects abstract patterns to concrete decisions you can make in the BDE and BQ pipelines today.

---

## 1. ETL vs. ELT — Where Transformation Lives

**ETL (Extract → Transform → Load)** runs transformations in the pipeline layer before data reaches the destination. PDI `.ktr` files in your BDE pipeline are a textbook example: field mappings, lookups, type coercions, and business-rule filters all happen inside Pentaho before the `aesf-py-middleware` API receives a payload.

**ELT (Extract → Load → Transform)** loads raw data into the destination first and uses the destination's compute for transformation. Your BQ pipeline does this: Python upserts raw Salesforce rows into staging tables, then BigQuery SQL functions perform the joins, deduplication, and aggregations that produce the analytics-ready tables.

```
ETL (BDE pipeline)
─────────────────────────────────────────────────────────
Salesforce SOQL → PDI .ktr (field map, type coerce) → POST /api/v2/ → Epic BDE

ELT (BQ pipeline)
─────────────────────────────────────────────────────────
Salesforce SOQL → Python upsert → BQ staging table → BQ SQL transform → BQ final table
```

The ELT model wins when the destination engine (BigQuery, Snowflake, Redshift) is orders of magnitude faster at set-based transforms than your pipeline layer. The ETL model wins when the destination API has strict input contracts (like the Epic SDK endpoints) and rejects mal-formed payloads — you must validate and transform before delivery.

**Common mistake:** Mixing the models ad hoc — doing partial transformation in PDI and finishing in BigQuery SQL with no clear contract between layers. If a field is renamed in the PDI step but not updated in the downstream SQL, you get silent `NULL`s in final tables. Document the transformation boundary explicitly and enforce it in code review.

---

## 2. Incremental vs. Full-Load Strategies

A **full load** truncates the destination and reloads everything from the source on every run. It is simple, always correct, and completely impractical at scale. A **full load** of your entire Salesforce Account table into BigQuery on every hourly run would consume quota, blow past API governor limits, and introduce hours of latency.

An **incremental load** transfers only records that changed since the last successful run. The pipeline maintains a **high-water mark** — the maximum `LastModifiedDate` (or equivalent) seen in the previous run — and filters the source query to `WHERE LastModifiedDate > :last_run`.

```python
# Python incremental loader pattern used in etl-appliedcrm-bq
import datetime
from google.cloud import bigquery

def get_last_watermark(bq_client: bigquery.Client, table: str) -> datetime.datetime:
    query = f"""
        SELECT COALESCE(MAX(sf_last_modified_date), TIMESTAMP('1970-01-01'))
        FROM `{table}`
    """
    result = list(bq_client.query(query).result())
    return result[0][0]

def extract_incremental(sf_client, object_name: str, since: datetime.datetime) -> list[dict]:
    soql = f"""
        SELECT Id, Name, LastModifiedDate
        FROM {object_name}
        WHERE LastModifiedDate > {since.strftime('%Y-%m-%dT%H:%M:%SZ')}
        ORDER BY LastModifiedDate ASC
    """
    return sf_client.query(soql)["records"]
```

**Soft deletes are the trap.** If a Salesforce record is deleted, its `LastModifiedDate` does not change — it disappears from SOQL results entirely. Incremental loads based on `LastModifiedDate` alone will leave stale rows in BigQuery indefinitely. The fix is a periodic **full reconcile** run that compares destination IDs against source IDs and tombstones or hard-deletes rows missing from the source. This is exactly what the `reconcile` job in your BDE pipeline does.

```
Incremental window
──────────────────────────────────────────────────────────────────
Timeline:  |── run 1 ──|── run 2 ──|── run 3 ──|── run 4 ──|
Watermark: T0          T1          T2          T3
Extracted:            [T0..T1]    [T1..T2]    [T2..T3]

Deleted records: invisible to all runs → handled by reconcile job
```

**Common mistake:** Using `SystemModstamp` instead of `LastModifiedDate` for Salesforce incremental queries. `SystemModstamp` updates on any system event (formula recalculate, sharing recalculate) and will over-extract records that have no meaningful data change, increasing load and masking real signal.

---

## 3. Idempotency and Restartable Pipelines

A pipeline is **idempotent** if running it multiple times with the same inputs produces the same final state as running it once. This property is non-negotiable for production pipelines — network failures, Kubernetes pod evictions, and rate-limit retries mean every pipeline run must be safe to replay.

The three building blocks of idempotency are:

1. **Upsert semantics at the destination.** Use `MERGE` in BigQuery or `INSERT ... ON CONFLICT DO UPDATE` in PostgreSQL instead of `INSERT`. A rerun of the same records updates existing rows rather than creating duplicates.
2. **Idempotent API calls.** The `aesf-py-middleware` endpoints are called with the Salesforce `Id` as the natural key. A `POST /clients` that already exists should behave the same as `PUT /clients/{id}` — the middleware achieves this via its queue table's upsert pattern.
3. **Checkpointing.** Track which logical batch (e.g., `offset` or `page`) was last successfully committed so a restart skips already-processed batches.

```python
# Idempotent BQ upsert — safe to run multiple times with the same source batch
def upsert_to_bigquery(
    bq_client: bigquery.Client,
    staging_table: str,
    target_table: str,
    key_column: str = "sf_id",
) -> None:
    merge_sql = f"""
        MERGE `{target_table}` T
        USING `{staging_table}` S
        ON T.{key_column} = S.{key_column}
        WHEN MATCHED THEN
            UPDATE SET
                T.name = S.name,
                T.sf_last_modified_date = S.sf_last_modified_date,
                T.updated_at = CURRENT_TIMESTAMP()
        WHEN NOT MATCHED THEN
            INSERT (sf_id, name, sf_last_modified_date, inserted_at)
            VALUES (S.sf_id, S.name, S.sf_last_modified_date, CURRENT_TIMESTAMP())
    """
    bq_client.query(merge_sql).result()
```

In Pentaho, idempotency is achieved by using **"Insert/Update"** or **"Update"** steps rather than plain **"Table Output"** steps. A plain `Table Output` step with "Truncate table" disabled will insert duplicates on rerun. Confirm every `.ktr` that writes to the middleware API queue tables uses Insert/Update with the Salesforce `Id` as the lookup key.

**Common mistake:** Marking a pipeline "idempotent" because individual steps are idempotent but forgetting that the watermark update itself is not atomic with the data write. If the pipeline writes data successfully then crashes before persisting the new watermark, the next run re-processes the same window — which is fine if writes are idempotent. But if the watermark is updated first and then the write crashes, you lose that window silently. Always update the watermark *after* confirming a successful write.

---

## 4. Watermarking and Change Data Capture (CDC)

**Watermarking** is the technique of recording the "furthest point reached" in a time-ordered stream so incremental runs know where to resume. For Salesforce-sourced pipelines, `LastModifiedDate` is the natural watermark column.

**CDC (Change Data Capture)** is a lower-level technique that captures every row-level change (INSERT, UPDATE, DELETE) from the source system's transaction log rather than polling with timestamps. True CDC produces an ordered, append-only stream of change events — far more reliable than timestamp polling because it captures deletes and handles clock skew.

Salesforce does not expose a transaction log, but it offers two CDC-adjacent mechanisms:
- **Streaming API / Change Data Capture events** — near-real-time CDC events for subscribed objects, delivered via the Streaming API. These capture creates, updates, deletes, and undeletes.
- **`getUpdated()` / `getDeleted()` REST calls** — batch CDC for a time window; returns IDs of changed/deleted records.

For your BQ pipeline, a watermarking table in BigQuery tracks the high-water mark per Salesforce object:

```sql
-- BQ watermark table schema
CREATE TABLE IF NOT EXISTS `aedl-ssa-1668620072.etl_control.watermarks` (
    object_name     STRING NOT NULL,
    last_run_ts     TIMESTAMP NOT NULL,
    last_success_ts TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Read watermark before extraction
SELECT last_success_ts
FROM `aedl-ssa-1668620072.etl_control.watermarks`
WHERE object_name = 'Account';

-- Advance watermark after successful load
UPDATE `aedl-ssa-1668620072.etl_control.watermarks`
SET last_success_ts = CURRENT_TIMESTAMP(),
    updated_at = CURRENT_TIMESTAMP()
WHERE object_name = 'Account';
```

**Common mistake:** Storing the watermark inside the destination data table (e.g., as `MAX(sf_last_modified_date)`) instead of in a dedicated control table. Deriving the watermark from the data conflates "what we loaded" with "what the source had" — a partial load that committed half its records will produce a watermark that skips the other half on the next run.

---

## 5. Schema Evolution Handling

Production pipelines break when source schemas change. A Salesforce admin adds a custom field, renames a picklist value, or changes a field type — and suddenly your PDI `.ktr` mapping throws a NullPointerException or your BigQuery `INSERT` fails with a type mismatch.

Schema evolution strategies range from brittle to robust:

| Strategy | Description | Risk |
|---|---|---|
| **Hard-coded column list** | `SELECT Id, Name, Phone FROM Account` | Misses new fields; breaks on removed fields |
| **SELECT *** | Fetches all fields dynamically | BigQuery schema must match exactly or `INSERT` fails |
| **Schema registry** | External store defines canonical schema; pipeline validates against it | Most robust; requires setup overhead |
| **Additive-only policy** | New columns are always appended; existing columns never removed | Safe for most analytics use cases |

For BigQuery, the additive-only policy is easiest to implement — BigQuery supports `ALTER TABLE ADD COLUMN` without rewriting data, and `INSERT` with a subset of columns leaves new columns `NULL`. The Python upsert script in your BQ pipeline should detect new columns in the Salesforce `describe()` response and auto-apply `ALTER TABLE ADD COLUMN` before running the load.

```python
def sync_schema(bq_client: bigquery.Client, table_ref: str, sf_fields: list[dict]) -> None:
    """Add columns present in Salesforce describe but missing from BigQuery table."""
    SF_TO_BQ_TYPE = {
        "string": "STRING", "boolean": "BOOL", "double": "FLOAT64",
        "int": "INT64", "datetime": "TIMESTAMP", "date": "DATE",
        "textarea": "STRING", "id": "STRING", "reference": "STRING",
        "currency": "NUMERIC", "percent": "FLOAT64",
    }
    table = bq_client.get_table(table_ref)
    existing_cols = {f.name.lower() for f in table.schema}

    for field in sf_fields:
        col_name = field["name"].lower()
        if col_name not in existing_cols:
            bq_type = SF_TO_BQ_TYPE.get(field["type"], "STRING")
            ddl = f"ALTER TABLE `{table_ref}` ADD COLUMN IF NOT EXISTS `{col_name}` {bq_type}"
            bq_client.query(ddl).result()
            print(f"Added column: {col_name} ({bq_type})")
```

In Pentaho PDI, schema evolution is harder because `.ktr` files contain hard-coded field lists. A breaking schema change requires updating the transformation file, rebuilding the container image, and redeploying. Mitigate this by keeping PDI transforms minimal (pass-through where possible) and doing schema-sensitive logic in Python or SQL where it is easier to make dynamic.

**Common mistake:** Silently truncating string values that exceed a declared column length. A Salesforce `Name` field can be 255 chars, but if your BigQuery column is `VARCHAR(200)` (in a PostgreSQL staging area), the upsert silently truncates. Always use unbounded `STRING` / `TEXT` types for Salesforce string fields and apply length constraints only at the presentation layer.

---

## 6. Pentaho/PDI Architecture in Your Pipelines

Pentaho Data Integration (PDI) uses two file types:

- **`.ktr` (Kettle Transformation):** A directed acyclic graph of steps processing rows in memory. Steps are connected by hop pipes. Rows flow one at a time through the graph. Used for row-level transforms: field mapping, type conversion, lookups, filtering, splitting.
- **`.kjb` (Kettle Job):** An orchestration file that runs transformations and sub-jobs in sequence or parallel with conditional branching. A job is the unit of orchestration; a transformation is the unit of data processing.

Your BDE pipeline structure maps cleanly onto this:

```
etl-appliedcrm-bde/Jobs/
├── sync.kjb          ← Master job: runs incremental sync transformations
├── migration.kjb     ← One-time or periodic full-load job
├── reconcile.kjb     ← Diff + correct: detects orphans and corrects drift
└── reset.kjb         ← Truncate + reload; used for schema migrations or recovery

Each .kjb calls multiple .ktr files, e.g.:
sync.kjb
  └─ extract_accounts.ktr      (SOQL → rows)
  └─ transform_accounts.ktr    (field map, lookups)
  └─ load_accounts.ktr         (HTTP POST to middleware API)
```

**PDI step execution model:**

```
Input Step → [row buffer] → Transform Step → [row buffer] → Output Step
                                ↑
                          runs in parallel
                          threads (configurable)
```

Each step runs in its own thread. The row buffer between steps is bounded (default 50,000 rows). If an output step is slower than an input step, backpressure causes the input step to block — this is how PDI handles flow control without out-of-memory errors.

Key PDI steps relevant to your pipelines:

| Step | Use |
|---|---|
| **REST Client** | HTTP calls to `aesf-py-middleware` API |
| **Insert/Update** | Idempotent writes to PostgreSQL middleware queue tables |
| **Modified Java Script Value** | Inline JS transforms (avoid for complex logic; prefer Python step or dedicated `.ktr`) |
| **Filter Rows** | Branch pipeline based on row conditions |
| **Merge Rows (diff)** | Compare old vs. new rows; used in `reconcile.kjb` to detect changed records |
| **Set Variables / Get Variables** | Pass values (e.g., watermark timestamp) between transformation steps and across jobs |

**Common mistake:** Running a `.kjb` directly in the container without error-exit propagation. By default, Pentaho's `pan.sh` (transformation runner) and `kitchen.sh` (job runner) return exit code `0` even on failure unless you set `-level=Error` and check PDI's exit code explicitly. In Kubernetes, a pod that exits `0` is considered successful — a failed ETL job that returns `0` will not trigger a restart or alert.

```bash
# Correct invocation in your Kubernetes Job manifest entrypoint
kitchen.sh -file=/jobs/sync.kjb -level=Basic
EXIT_CODE=$?
if [ $EXIT_CODE -ne 0 ]; then
  echo "PDI job failed with exit code $EXIT_CODE"
  exit $EXIT_CODE
fi
```

---

## 7. Error Handling and Dead-Letter Patterns

No pipeline runs perfectly forever. Records fail to load because the source payload is malformed, the destination API is temporarily unavailable, or a business rule rejects a specific record. The question is not *whether* failures happen — it is *what the pipeline does with them*.

**Fail-fast:** On any error, abort the entire batch and retry. Simple but wasteful — one bad record blocks 10,000 good ones.

**Skip and log:** Write bad records to an error log, continue processing. Fast but silent — errors accumulate unnoticed.

**Dead-letter queue (DLQ):** Route failed records to a separate store (a "dead-letter" table or queue) for later inspection, repair, and replay. This is the production-grade pattern.

```python
# Dead-letter pattern in Python ETL for BQ pipeline
import json
from google.cloud import bigquery
from datetime import datetime, timezone

def load_with_dlq(
    bq_client: bigquery.Client,
    records: list[dict],
    target_table: str,
    dlq_table: str,
) -> dict:
    success_count = 0
    dlq_count = 0

    for record in records:
        try:
            errors = bq_client.insert_rows_json(target_table, [record])
            if errors:
                raise ValueError(f"BQ insert errors: {errors}")
            success_count += 1
        except Exception as exc:
            dlq_record = {
                "sf_id": record.get("Id"),
                "payload": json.dumps(record),
                "error_message": str(exc),
                "failed_at": datetime.now(timezone.utc).isoformat(),
                "source_table": target_table,
            }
            bq_client.insert_rows_json(dlq_table, [dlq_record])
            dlq_count += 1

    return {"success": success_count, "dlq": dlq_count}
```

For the BDE pipeline, the `aesf-py-middleware` queue tables (`sync_queue`, `error_queue`) serve this role. Records that fail to sync to Epic BDE are moved to an error state in the queue table with the error message, and the `reconcile.kjb` job periodically retries them.

**DLQ replay** is as important as DLQ write. A dead-letter queue that fills up and is never drained is just a slow data loss. Build a replay mechanism — a script or Pentaho job that reads from the DLQ, re-attempts the load, and removes successfully replayed records.

**Common mistake:** Using the DLQ as a permanent archive rather than an operational queue. If your DLQ table has millions of rows spanning years, nobody is reading it. Set a retention policy (e.g., keep DLQ records for 30 days), alert when DLQ depth exceeds a threshold, and treat DLQ drain rate as an SLO.

---

## 8. Orchestration: Airflow vs. Prefect vs. Cron

Orchestration is the layer that schedules pipeline runs, tracks dependencies between jobs, handles retries, and provides observability. Your current pipelines use Kubernetes CronJobs — which is fine for simple schedules but lacks dependency management and visibility.

| Feature | Kubernetes CronJob | Apache Airflow | Prefect |
|---|---|---|---|
| Schedule trigger | Cron expression | Cron + sensors + external triggers | Schedules + event-driven flows |
| DAG/dependency model | None | DAG of tasks | Flow of tasks |
| Retry logic | Pod restart policy | Per-task retry count + delay | Per-task, configurable |
| Backfill | Manual | Built-in `backfill` command | Manual with flow parameters |
| Observability | Pod logs only | Web UI with run history | Cloud UI + local UI |
| GKE integration | Native | KubernetesPodOperator | Kubernetes worker pool |
| Learning curve | Low | High (DAG authoring, scheduler tuning) | Medium (Pythonic, less config) |

For your stack, **Prefect** is the pragmatic upgrade from Kubernetes CronJobs. It is Python-native, integrates with GKE, and requires far less operational overhead than a self-managed Airflow cluster.

```python
# Prefect flow wrapping your BQ ETL pipeline
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta

@task(retries=3, retry_delay_seconds=60, cache_key_fn=task_input_hash, cache_expiration=timedelta(hours=1))
def extract_from_salesforce(object_name: str, since: str) -> list[dict]:
    # ... SOQL extraction logic
    return records

@task(retries=2)
def upsert_to_bigquery(records: list[dict], table: str) -> int:
    # ... BQ upsert logic
    return len(records)

@flow(name="sf-to-bq-incremental", log_prints=True)
def sf_to_bq_pipeline(object_name: str = "Account") -> None:
    watermark = get_watermark(object_name)       # task
    records = extract_from_salesforce(object_name, since=watermark)
    count = upsert_to_bigquery(records, table=f"staging.{object_name.lower()}")
    advance_watermark(object_name)               # task
    print(f"Loaded {count} records for {object_name}")
```

**Airflow** is appropriate if your org has many cross-pipeline dependencies (e.g., the BQ pipeline must complete before a downstream dbt model runs) or if there is existing Airflow infrastructure to leverage. For standalone pipelines on GKE, it introduces more operational cost than value.

**Common mistake:** Treating orchestration as purely a scheduling concern. The real value of Airflow/Prefect is in dependency management and observability — knowing exactly which task failed, what its inputs were, and how to rerun just that task. Using cron without these features means debugging failures by grepping Kubernetes logs and manually determining which records need reprocessing.

---

## 9. Connecting the Patterns: BDE and BQ Pipeline Design Review

Applying all the above patterns to your two production pipelines:

**`etl-appliedcrm-bde` (Salesforce → Epic BDE via middleware)**

```
Design analysis
───────────────────────────────────────────────────────────
Extraction:    Incremental (LastModifiedDate watermark)  ✓
Idempotency:   Insert/Update in PDI steps + middleware upsert  ✓
Delete sync:   reconcile.kjb periodic full diff  ✓
Schema evol:   Hard-coded in .ktr files — brittle  ⚠
Error handling: middleware error_queue table (DLQ)  ✓
Observability:  FastAPI monitoring sidecar + Datadog  ✓
Orchestration:  Kubernetes CronJob — no dependency DAG  ⚠
Exit codes:     Verify kitchen.sh exit propagation  ⚠
```

**`etl-appliedcrm-bq` (Salesforce → BigQuery)**

```
Design analysis
───────────────────────────────────────────────────────────
Extraction:    Incremental (LastModifiedDate)  ✓
Idempotency:   MERGE / upsert to BQ  ✓
Delete sync:   Needs explicit reconcile pass  ⚠
Schema evol:   Dynamic ALTER TABLE ADD COLUMN (if implemented)  ✓/⚠
Watermark:     Should be in etl_control.watermarks table  ✓
Error handling: DLQ table for failed records  ✓/⚠
Orchestration:  Kubernetes CronJob  ⚠
```

The highest-leverage improvements for staff-level reliability are: (1) add a reconcile/delete-detection pass to the BQ pipeline, (2) instrument Pentaho job exit codes in Kubernetes, and (3) introduce a lightweight orchestration layer (Prefect) for dependency tracking and retry visibility.

---

## 10. Key Concepts Summary

```
Data Engineering & ETL Patterns
│
├── Extraction Strategy
│   ├── Full load (simple, expensive)
│   └── Incremental (watermark / CDC-based)
│       ├── Timestamp polling (LastModifiedDate)
│       └── CDC (Salesforce Streaming API / getDeleted())
│
├── Transformation Placement
│   ├── ETL — transform before load (BDE pipeline / PDI .ktr)
│   └── ELT — load raw, transform in destination (BQ pipeline / SQL)
│
├── Correctness Guarantees
│   ├── Idempotency (upsert semantics, atomic watermark)
│   ├── Restartability (checkpointing, idempotent batches)
│   └── Schema evolution (additive-only, dynamic DDL)
│
├── Failure Handling
│   ├── Dead-letter queue (route, inspect, replay)
│   └── Retry policies (exponential backoff, max attempts)
│
├── Pentaho PDI
│   ├── .ktr Transformation (row-level processing)
│   ├── .kjb Job (orchestration of .ktr files)
│   └── kitchen.sh / pan.sh (CLI invocation, exit code propagation)
│
└── Orchestration
    ├── Kubernetes CronJob (simple, no DAG)
    ├── Apache Airflow (DAG-based, heavy)
    └── Prefect (Python-native, lightweight, GKE-friendly)
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the core difference between ETL and ELT, and which pattern does each of your production pipelines follow?

**2.** Why is `LastModifiedDate` polling insufficient to detect deleted records in Salesforce, and what job in `etl-appliedcrm-bde` compensates for this?

**3.** Define idempotency in the context of a data pipeline. What SQL construct achieves idempotency in BigQuery?

**4.** What is a watermark in a streaming/incremental pipeline, and why should it be stored in a dedicated control table rather than derived from the data table?

**5.** Explain the difference between a Pentaho `.ktr` file and a `.kjb` file. Which one is the unit of orchestration?

**6.** What is the default exit-code behavior of `kitchen.sh` on PDI job failure, and why does this matter in a Kubernetes CronJob context?

**7.** Describe the dead-letter queue pattern. What are the two most important operational properties a DLQ must have beyond simply accepting failed records?

**8.** What is the "additive-only schema evolution" policy, and how can you implement it dynamically in the Python BQ upsert script?

**9.** A Salesforce admin changes the `Phone` field type from `phone` to `string` on the `Account` object. What breaks in a PDI-based ETL pipeline that hard-codes field types in `.ktr` files, and how do you fix it?

**10.** Compare Kubernetes CronJob, Apache Airflow, and Prefect on three dimensions: dependency management, observability, and GKE integration.

**11.** What is CDC (Change Data Capture), and how does Salesforce's Streaming API approximate it?

**12.** In the BDE pipeline, the `sync.kjb` job processes 10,000 Account records and crashes after 7,000. On the next run, how does idempotency prevent duplicate records in Epic BDE?

**13.** What is backpressure in the context of PDI step execution, and how does PDI handle it?

**14.** Why should you use `INSERT ... ON CONFLICT DO UPDATE` (upsert) rather than `INSERT ... ON CONFLICT DO NOTHING` for ETL loads to a PostgreSQL staging table?

**15.** Explain the reconcile pattern. When should `reconcile.kjb` be run — every hour, daily, or weekly — and why?

**16.** What is the difference between a soft delete and a hard delete in Salesforce, and how does each affect incremental ETL extraction?

**17.** A record appears in the BigQuery DLQ table. What information should the DLQ row contain to make the record replayable and debuggable?

**18.** In Prefect, what does the `cache_key_fn=task_input_hash` parameter on a `@task` decorator accomplish, and when would you NOT want it?

**19.** Your BQ pipeline's watermark shows `2026-11-01 08:00:00`. The pipeline runs at 09:00, extracts records up to `08:59:59`, and updates the watermark to `08:59:59`. The next run at `10:00` extracts from `08:59:59`. What subtle data loss risk exists here, and how do you fix it?

**20.** What distinguishes staff-level data engineering from senior-level when designing a new ETL pipeline for a net-new Salesforce object?

---

### Answers

??? note "Reveal Answers"

    **1.** ETL transforms data in the pipeline layer before it reaches the destination, while ELT loads raw data into the destination and transforms it there using the destination's compute engine. The `etl-appliedcrm-bde` pipeline follows ETL: PDI `.ktr` files transform Salesforce records into the shape expected by the Epic middleware API before any API call is made. The `etl-appliedcrm-bq` pipeline follows ELT: Python upserts raw Salesforce data into BigQuery staging tables, and BigQuery SQL functions perform the joins, deduplication, and aggregations to produce analytics-ready tables. The choice reflects the destination's constraints — Epic has a strict API contract requiring pre-validated payloads, while BigQuery's massive parallel SQL engine makes in-destination transformation faster and cheaper.

    **2.** `LastModifiedDate` only changes when a record is updated — a deleted record disappears from SOQL results entirely and never appears in a `WHERE LastModifiedDate > :watermark` query. This means an incremental load based solely on `LastModifiedDate` will leave orphan rows in the destination indefinitely, creating data drift between Salesforce and Epic. The `reconcile.kjb` job in `etl-appliedcrm-bde` compensates by performing a full comparison: it fetches all IDs from the source, compares them against the destination, and issues delete or correction operations for records present in the destination but absent from the source. This job is typically run on a slower schedule (daily or weekly) since it is more expensive than the incremental sync.

    **3.** An idempotent pipeline produces the same final destination state regardless of how many times it is run with the same input data. In BigQuery, the `MERGE` statement (sometimes called an upsert) achieves this: it matches incoming rows against the destination table on a key column, updates existing rows if the key matches, and inserts new rows if the key does not exist. Running the same `MERGE` twice with identical source data results in the same destination state as running it once — no duplicates are created and no data is lost. This is the foundation of making pipelines safe to retry after partial failures.

    **4.** A watermark is the highest-value marker (typically a timestamp) seen in a successfully processed batch, used by the next run to determine where to start extraction. Storing it in the data table (as `MAX(sf_last_modified_date)`) conflates "what we successfully processed" with "what the source contains" — a partial load that committed half its records will compute a watermark that skips the unloaded half on the next run. A dedicated control table (e.g., `etl_control.watermarks`) decouples the watermark from the data, allows atomic updates (update watermark only after confirming a successful write), and supports per-object watermarks in a single row-per-object store.

    **5.** A `.ktr` (Kettle Transformation) file is a directed graph of data-processing steps that transforms rows — it handles field mappings, type conversions, lookups, filters, and HTTP calls. It is the unit of data processing. A `.kjb` (Kettle Job) file is an orchestration file that runs transformations and sub-jobs in sequence or with conditional branching — it is the unit of orchestration. In your BDE pipeline, `sync.kjb` is the job that calls multiple `.ktr` transformations in the correct order. Running `kitchen.sh` executes a `.kjb`; running `pan.sh` executes a `.ktr` directly.

    **6.** By default, `kitchen.sh` may return exit code `0` even when a PDI job encounters errors, depending on the log level and how error handling is configured in the job. In a Kubernetes CronJob, a pod that exits with code `0` is marked as `Succeeded`, so Kubernetes will not trigger a restart or alert — a silently failing ETL job looks healthy to the cluster. The fix is to explicitly check the `kitchen.sh` exit code in the container entrypoint script and propagate non-zero codes, and to run with `-level=Basic` or higher to ensure errors are captured in the log output that feeds your Datadog monitoring sidecar.

    **7.** A dead-letter queue routes records that fail to process to a separate store for later inspection and replay, rather than aborting the entire batch or silently discarding the record. Beyond accepting failed records, the two most important operational properties are: (1) **alerting** — the pipeline or a monitoring process must emit an alert when DLQ depth exceeds a threshold, so engineers know failures are occurring rather than discovering them during an audit; and (2) **replay capability** — there must be a mechanism (script, job, or UI) to correct the underlying issue and re-submit DLQ records to the main pipeline, with the DLQ row deleted upon successful replay. A DLQ that is never drained is functionally equivalent to data loss.

    **8.** The additive-only schema evolution policy means that new columns can be added to destination tables at any time, but existing columns are never removed or renamed — only new columns appear. In the Python BQ upsert script, this is implemented by calling Salesforce's `describe()` API to get the current field list, comparing it against the BigQuery table schema retrieved via `bq_client.get_table()`, and issuing `ALTER TABLE ADD COLUMN IF NOT EXISTS` DDL for each field present in Salesforce but absent in BigQuery. Because BigQuery `INSERT` statements with a column subset leave new columns as `NULL` in existing rows, this approach is backward-compatible and requires no backfill for historical data.

    **9.** In a PDI `.ktr` file with hard-coded field types, the Phone field's type metadata is baked into the transformation's field mapping step. If the Salesforce API now returns the field as `string` type but the `.ktr` step expects `phone` (which PDI may treat differently for validation or conversion), the step can throw a type mismatch error or silently truncate/mangle values. The fix involves: (1) updating the `.ktr` field mapping to accept the new type, (2) rebuilding and redeploying the container image, and (3) considering a more resilient approach where PDI passes string fields through without type coercion, deferring type enforcement to the destination. This is a strong argument for keeping PDI transforms thin and doing type-sensitive logic in downstream SQL.

    **10.** Kubernetes CronJob has no dependency management (jobs are independent and unaware of each other), minimal observability (pod logs only, no run history UI), and native GKE integration (zero setup). Apache Airflow has full DAG-based dependency management, a rich web UI with historical run tracking and task-level logs, and GKE integration via KubernetesPodOperator (but requires running and maintaining an Airflow cluster). Prefect has flow-based task dependencies, a cloud UI with run history (or a self-hosted server), and clean GKE integration via Kubernetes worker pools with significantly less operational overhead than Airflow — making it the best fit for your current pipeline complexity.

    **11.** Change Data Capture (CDC) is a pattern that captures every row-level change — INSERT, UPDATE, DELETE — from a source system, typically by reading the database transaction log, and produces an ordered stream of change events. Salesforce does not expose its transaction log directly, but its Streaming API with CDC subscriptions approximates it: you subscribe to change events for specific objects, and Salesforce pushes real-time events for creates, updates, deletes, and undeletes with the changed field values. Unlike timestamp polling, CDC events capture delete operations and provide field-level change details (which fields changed in an update), enabling more precise incremental loads and audit trails.

    **12.** Idempotency prevents duplicates because the PDI `.ktr` files use `Insert/Update` steps that match on the Salesforce `Id` field. When the next run starts from the last committed watermark and re-processes the 7,000 successfully loaded records, the `Insert/Update` step finds matching `Id` values in the middleware queue table and issues `UPDATE` instead of `INSERT` — the records are refreshed in place but no new rows are created. The 3,000 unprocessed records are then newly inserted. The final state in Epic BDE reflects all 10,000 records correctly, regardless of the mid-run crash. This only holds if the watermark was not advanced past the crash point — if it was, those 7,000 records would be skipped on the next run.

    **13.** Backpressure in PDI occurs when an upstream step produces rows faster than a downstream step can consume them. PDI connects steps via bounded in-memory row buffers (default 50,000 rows). When a buffer is full, the upstream step blocks (pauses) rather than allocating unbounded memory or dropping rows. This means a slow HTTP output step (posting to the middleware API) will naturally throttle the extraction step — the pipeline self-regulates its throughput to the slowest step without out-of-memory errors. You can observe this in PDI's step metrics, where blocked steps show paused row counts. Tuning the buffer size or parallelizing slow steps (increasing step copies) are the levers to improve throughput.

    **14.** `INSERT ... ON CONFLICT DO NOTHING` silently discards incoming rows when a key conflict exists — meaning updates to existing records in Salesforce are never reflected in the destination. This is correct behavior only if the destination is purely append-only and you never expect the source record to change. For Salesforce objects where records are regularly updated (Accounts, Contacts, Opportunities), you need `INSERT ... ON CONFLICT DO UPDATE SET ...` (upsert), which overwrites the existing row with the new values. Using `DO NOTHING` in an ETL destination effectively means your destination drifts from the source over time as records are updated in Salesforce but the destination retains the original values.

    **15.** The reconcile pattern performs a full comparison between source and destination to detect and correct drift that incremental loads miss — primarily deleted records and records that may have been corrupted or missed due to pipeline failures. `reconcile.kjb` should be run on a schedule that balances freshness against cost: daily is appropriate for most production Salesforce→Epic sync use cases, as delete-driven drift accumulating over 24 hours is generally acceptable. Hourly reconcile runs are usually wasteful and may hit Salesforce API governor limits. Weekly is too infrequent for production data if any downstream process depends on delete accuracy. The reconcile job should also run on-demand after any incident or manual data correction.

    **16.** A soft delete in Salesforce moves a record to the recycle bin — the record is gone from SOQL queries but recoverable for 15 days. Incremental ETL based on `LastModifiedDate` will not see soft-deleted records at all, leaving orphan rows in the destination. A hard delete (after the recycle bin is emptied) is permanent and also invisible to `LastModifiedDate` polling. Both types are detectable via Salesforce's `getDeleted()` REST API, which returns IDs of records deleted within a specified time window (up to 30 days). Your reconcile job can use `getDeleted()` to obtain the IDs of deleted records and issue corresponding deletes or tombstone updates in the destination, handling both soft and hard deletes correctly.

    **17.** A replayable, debuggable DLQ row should contain: (1) the natural key of the failed record (e.g., `sf_id`) so you can correlate it with the source; (2) the full raw payload as JSON so the record can be resubmitted without re-extracting from Salesforce; (3) the error message and stack trace from the failure; (4) the timestamp of the failure; (5) the target table or endpoint the record was being loaded into; and (6) a `retry_count` field so the replay mechanism can skip records that have failed repeatedly (likely indicating a structural data quality issue rather than a transient error). Without the raw payload, replay requires a fresh extraction from Salesforce, which may no longer return the same data if the record was subsequently updated.

    **18.** `cache_key_fn=task_input_hash` causes Prefect to compute a hash of the task's input arguments and cache the task result for the `cache_expiration` duration. If the same task is called again within that window with identical inputs, Prefect returns the cached result without re-executing the task. This is valuable for expensive extraction tasks that may be retried due to a downstream failure — if the Salesforce extraction already succeeded, Prefect won't re-query Salesforce on retry. You would NOT want it on tasks with side effects that must always execute (e.g., watermark updates), on tasks whose output depends on time (the same inputs at different times may yield different results), or on tasks where the cache could mask required re-execution during a backfill.

    **19.** The risk is clock skew or records written to Salesforce between `08:59:59` and the moment the next run's extraction query executes at `10:00`. The watermark is exclusive on the lower bound (`>` not `>=`) of the next run's window — records with `LastModifiedDate = 08:59:59` may be processed or missed depending on sub-second precision and query execution timing. A safer pattern is to use a small **lookback overlap**: the next run starts from `watermark - 5 minutes` rather than `watermark` exactly, and relies on idempotent upserts to handle any duplicates in the overlap window. This trades a small amount of re-processing for guaranteed coverage of any records near the boundary.

    **20.** A senior engineer designs the pipeline to work correctly for the happy path and handles the obvious error cases with try/except and retries. A staff engineer, before writing any code, asks and answers: What is the source's delete story — how will we detect deletes? What is the idempotency contract — what happens if this runs twice? What does schema drift look like for this object, and how will the pipeline behave when a new field appears? What is the SLO for lag, and how will we alert when the pipeline is behind? What is the blast radius if this pipeline writes corrupt data — which downstream systems depend on it? How will we replay a failed run without data loss or duplication? Staff-level work means the pipeline is designed around its failure modes from the start, not hardened against them after the first production incident.

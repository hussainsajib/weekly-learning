# Week 15 — Database Indexing & Query Optimization

**Week of:** September 14, 2026
**Estimated study time:** ~2 hours
**Tags:** `postgresql` `databases` `performance` `indexing`

---

## Overview

Database indexing is the single highest-leverage performance skill for backend engineers working with relational databases. In the crm-middleware, queue tables (`sync_queue`, `sync_status`) are the hottest tables in the system — every Salesforce-triggered sync event writes a row, and every EHR system API callback reads and updates one. When those tables grow past a few hundred thousand rows without the right indexes, query latency climbs from milliseconds into seconds, and EHR sync latency follows directly. Understanding how PostgreSQL physically stores and retrieves data is the foundation for fixing that.

PostgreSQL ships with several index types, each suited to a different access pattern. B-tree handles equality and range queries on scalar values and covers the vast majority of production workloads. GIN and GiST serve full-text search, JSONB, and geometric data. BRIN offers a compact option for naturally ordered, write-heavy append tables like time-series logs. Choosing the wrong index type — or adding no index at all — is the most common database performance mistake at the senior engineer level.

Beyond index choice, the query planner is the intermediary between your SQL and your storage. PostgreSQL's planner uses column statistics, gathered by `ANALYZE`, to choose join strategies (hash join, merge join, nested loop), decide whether a sequential scan beats an index scan, and estimate intermediate row counts. When planner estimates are wildly wrong — which happens when statistics are stale or a column has high correlation — query plans can degrade catastrophically. Reading `EXPLAIN ANALYZE` output fluently is a non-negotiable skill.

At the ORM layer, SQLAlchemy's convenience abstractions can silently generate expensive SQL. The canonical trap is the N+1 query: loading a list of parent objects and then triggering a separate SELECT per row to load a related child. In the integration platform sync status flow, this manifests as one query per `SyncQueue` record to load its associated `Account` or `Policy` object — turning a 200-row result set into 201 round-trips. This guide covers every layer of the stack, from B-tree internals through `EXPLAIN ANALYZE` to SQLAlchemy `joinedload` patterns that eliminate N+1 entirely.

---

## 1. Index Types — Choosing the Right Tool

PostgreSQL supports multiple index access methods. Choosing the wrong one wastes storage and provides no speedup; choosing the right one can reduce a full sequential scan to a single page read.

### B-tree

The default. B-tree (balanced tree) stores values in sorted order across leaf pages linked in a doubly-linked list. It supports `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `LIKE 'prefix%'` (left-anchored only), and `ORDER BY` without a separate sort step.

```sql
-- Queue table: most common access pattern is filtering by status + created_at
CREATE INDEX idx_sync_queue_status_created
    ON sync_queue (status, created_at DESC);

-- EXPLAIN ANALYZE output after index creation:
EXPLAIN ANALYZE
SELECT id, object_type, object_id, status
FROM sync_queue
WHERE status = 'pending'
ORDER BY created_at DESC
LIMIT 100;

-- Index Only Scan using idx_sync_queue_status_created (cost=0.56..12.31 rows=100 width=48)
--   Index Cond: (status = 'pending')
-- Planning Time: 0.3 ms
-- Execution Time: 0.4 ms
```

**Common mistake:** Creating single-column indexes on a frequently-filtered multi-column WHERE clause. A composite index on `(status, created_at)` is far more selective than two separate single-column indexes — the planner can only use one index per table scan (without a bitmap scan merge), and composite indexes allow the planner to filter on the leading column and range-scan the second column in one pass.

### Hash

Hash indexes store a hash of the indexed value, supporting only `=` lookups. Pre-PostgreSQL 10 they were not WAL-logged and broke on crash recovery — avoid them on older versions. On PG 14+ they can outperform B-tree for equality-only access on long string columns (UUIDs, Salesforce IDs).

```sql
-- Salesforce 18-char IDs: equality-only, long string, high cardinality
CREATE INDEX idx_sync_queue_sf_id_hash
    ON sync_queue USING hash (salesforce_id);
```

Hash indexes are roughly 40% smaller than B-tree for high-cardinality string columns. Use them only when the workload is exclusively equality predicates — any range or ordering requirement reverts to B-tree.

### GIN (Generalized Inverted Index)

GIN indexes invert the mapping: instead of `value → row`, they store `element → [rows containing element]`. This makes them efficient for containment queries on composite values: JSONB, arrays, full-text `tsvector`.

```sql
-- sync_queue.payload is JSONB storing the Salesforce change delta
CREATE INDEX idx_sync_queue_payload_gin
    ON sync_queue USING gin (payload);

-- Query: find all queued syncs that changed the BillingState field
SELECT id FROM sync_queue
WHERE payload @> '{"changed_fields": ["BillingState"]}';
```

GIN indexes are expensive to build and update (write amplification is high). Use them on JSONB or array columns with frequent containment queries, not on columns that change on every write.

### GiST (Generalized Search Tree)

GiST is an extensible framework supporting lossy index structures for geometric types (`point`, `box`, `polygon`), range types (`tsrange`, `daterange`), and full-text search (as an alternative to GIN). GiST is lossy — it requires a recheck step — but supports more operator classes than GIN.

```sql
-- Example: index on a daterange column for policy effective period overlap queries
CREATE INDEX idx_policy_effective_range_gist
    ON app_policy USING gist (effective_range);

SELECT * FROM app_policy
WHERE effective_range && '[2026-01-01, 2026-12-31]'::daterange;
```

### BRIN (Block Range Index)

BRIN stores per-block-range min/max values. It is tiny (kilobytes vs. megabytes for B-tree) but only useful when the physical storage order of rows is correlated with the indexed column — meaning rows inserted later have monotonically higher values. This is exactly true for append-only audit logs and time-series tables.

```sql
-- sync_audit_log: append-only, created_at is monotonically increasing
CREATE INDEX idx_audit_log_created_brin
    ON sync_audit_log USING brin (created_at)
    WITH (pages_per_range = 128);

-- BRIN is useless for queue tables where rows are updated in-place (status changes)
-- and physical order has no correlation with status or created_at after vacuuming.
```

**Common mistake:** Applying BRIN to a table with `UPDATE`-heavy workload. BRIN relies on physical correlation; heap updates via HOT can scatter row versions, breaking the correlation assumption.

---

## 2. Partial, Expression, and Covering Indexes

### Partial Indexes

A partial index includes only rows matching a `WHERE` predicate. For queue tables with a small fraction of "active" rows, a partial index on active rows is dramatically smaller and faster than a full-table index.

```sql
-- Only index rows that still need processing — completed/failed rows are cold
CREATE INDEX idx_sync_queue_pending_partial
    ON sync_queue (created_at DESC, object_type)
    WHERE status IN ('pending', 'retrying');

-- This index might cover 5% of the table instead of 100%
-- Query must include the partial index predicate for the planner to use it:
SELECT id, object_type, object_id
FROM sync_queue
WHERE status IN ('pending', 'retrying')
  AND object_type = 'Account'
ORDER BY created_at DESC
LIMIT 50;
```

**Common mistake:** Writing a query predicate that doesn't exactly match the partial index `WHERE` clause. `WHERE status = 'pending'` will NOT use an index defined with `WHERE status IN ('pending', 'retrying')` — the predicate must imply the index condition, not merely overlap it.

### Expression Indexes

An expression index indexes the output of a function or expression rather than a raw column value. Required when queries always filter on a derived value.

```sql
-- Salesforce IDs are case-insensitive; normalize in the index
CREATE INDEX idx_sync_queue_sf_id_lower
    ON sync_queue (lower(salesforce_id));

-- This query now uses the expression index:
SELECT * FROM sync_queue WHERE lower(salesforce_id) = lower($1);

-- Without the expression index, the above is a sequential scan even with
-- a plain B-tree index on salesforce_id.
```

Expression indexes increase write overhead because PostgreSQL must evaluate the expression on every insert and update. Keep expressions cheap (built-in functions, not user-defined PL/pgSQL functions).

### Covering Indexes (INCLUDE)

An `INCLUDE` clause adds non-key columns to the leaf pages of the index without making them part of the B-tree key. Queries that need only those columns can be satisfied entirely from the index (Index Only Scan), avoiding a heap fetch.

```sql
-- The worker fetches id + object_id + payload — add them as non-key includes
CREATE INDEX idx_sync_queue_status_covering
    ON sync_queue (status, created_at DESC)
    INCLUDE (id, object_id, object_type, payload);

EXPLAIN ANALYZE
SELECT id, object_id, object_type, payload
FROM sync_queue
WHERE status = 'pending'
ORDER BY created_at DESC
LIMIT 100;

-- Index Only Scan using idx_sync_queue_status_covering
-- Heap Fetches: 0   ← zero heap I/O
```

**Common mistake:** Overloading `INCLUDE` with too many wide columns. `INCLUDE` bloats index leaf pages, increasing I/O and cache pressure. Only include columns that are selected frequently alongside the key predicates.

---

## 3. Query Planner Statistics and ANALYZE

PostgreSQL's planner chooses a query plan based on statistics stored in `pg_statistic` (read via `pg_stats`). These statistics are populated by `ANALYZE` and include: row count, distinct value count, most-common values (MCV), histogram buckets, and correlation (physical ordering vs. logical ordering).

```sql
-- Check when a table was last analyzed and its row count estimate
SELECT relname, last_analyze, last_autoanalyze, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'sync_queue';

-- Inspect statistics for the status column
SELECT attname, n_distinct, most_common_vals, most_common_freqs, correlation
FROM pg_stats
WHERE tablename = 'sync_queue' AND attname = 'status';
```

When `n_distinct` is `-1`, PostgreSQL treats the column as having as many distinct values as rows (fully unique). When it is a small positive number like `4` (four status values), the planner knows to use MCV lists rather than histograms for selectivity estimates.

**Forcing a re-analyze after bulk loads:**

```sql
-- After bulk-inserting 500k rows from a migration, autovacuum may not have run yet
ANALYZE sync_queue;
-- Or more granularly:
ANALYZE sync_queue (status, created_at);
```

**Statistics target:** The default `statistics_target` is 100 (controls histogram bucket count). For highly skewed columns like `status`, raising it improves planner accuracy.

```sql
ALTER TABLE sync_queue ALTER COLUMN status SET STATISTICS 500;
ANALYZE sync_queue;
```

**Common mistake:** Running `EXPLAIN` (without `ANALYZE`) and treating the cost estimates as ground truth. `EXPLAIN` shows planner estimates; `EXPLAIN ANALYZE` shows actual row counts and timing. Discrepancies between "rows=X" (estimate) and "actual rows=Y" indicate stale or insufficient statistics.

---

## 4. Reading EXPLAIN ANALYZE Output

`EXPLAIN ANALYZE` is the single most important diagnostic tool for query performance. The output is a tree of plan nodes; each node shows estimated vs. actual rows, cost, loops, and timing.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT sq.id, sq.object_type, sq.object_id, ss.last_synced_at
FROM sync_queue sq
JOIN sync_status ss ON ss.object_id = sq.object_id
                    AND ss.object_type = sq.object_type
WHERE sq.status = 'retrying'
  AND sq.created_at > now() - interval '24 hours';
```

Sample output to parse:

```
Hash Join  (cost=142.50..8921.33 rows=4200 width=64)
           (actual time=12.4..45.2 rows=387 loops=1)
  Hash Cond: ((ss.object_id = sq.object_id) AND (ss.object_type = sq.object_type))
  Buffers: shared hit=812 read=203
  ->  Seq Scan on sync_status ss  (cost=0.00..7200.00 rows=180000 width=40)
        (actual time=0.1..28.3 rows=180000 loops=1)
        Buffers: shared hit=601 read=202
  ->  Hash  (cost=130.00..130.00 rows=1000 width=48)
             (actual time=10.2..10.2 rows=387 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 64kB
        ->  Index Scan using idx_sync_queue_status_created on sync_queue sq
              (cost=0.56..130.00 rows=1000 width=48)
              (actual time=0.05..9.8 rows=387 loops=1)
              Index Cond: ((status = 'retrying') AND (created_at > ...))
Planning Time: 1.2 ms
Execution Time: 45.5 ms
```

Key things to read:

| Field | What it means |
|---|---|
| `cost=0.56..130.00` | Planner's startup..total cost estimate (arbitrary units) |
| `rows=1000` | Planner's row count estimate |
| `actual rows=387` | True row count at runtime |
| `loops=1` | How many times this node executed (multiply actual times by loops) |
| `Buffers: shared hit=812 read=203` | Pages served from cache vs. disk |
| `Batches: 1` | Hash join fit in memory; Batches > 1 means spilled to disk |

**Common mistake:** Ignoring the `loops` field. A nested loop inner node with `loops=5000` and `actual time=0.1` per loop is actually 500 ms of total work, not 0.1 ms. Always multiply `actual time` by `loops`.

---

## 5. Join Strategies

PostgreSQL chooses among three join algorithms based on input sizes, available indexes, and `work_mem`.

### Nested Loop Join

Best for small outer relation + indexed inner relation. For each outer row, it probes the inner table's index.

```sql
-- Efficient when sync_queue result is small (e.g., 10 retrying rows)
-- and sync_status has an index on (object_id, object_type)
SET enable_hashjoin = off;  -- force nested loop for demonstration
EXPLAIN ANALYZE
SELECT sq.id, ss.last_synced_at
FROM sync_queue sq
JOIN sync_status ss USING (object_id, object_type)
WHERE sq.status = 'retrying' LIMIT 10;
-- Nested Loop with Index Scan inner: fast when outer is small
```

Nested loop degrades to O(N×M) when neither side has a useful index and the outer result set is large.

### Hash Join

Best for large unsorted inputs without a useful index on the join key. PostgreSQL hashes the smaller relation into an in-memory hash table, then probes it with each row from the larger relation.

```sql
-- Ensure work_mem is sufficient to avoid disk batches for large joins
SET work_mem = '64MB';
EXPLAIN ANALYZE
SELECT sq.id, ss.last_synced_at
FROM sync_queue sq
JOIN sync_status ss USING (object_id, object_type)
WHERE sq.created_at > now() - interval '7 days';
-- Hash Join: hashes sync_status, streams sync_queue rows against it
```

If `work_mem` is too low, hash join spills to disk (`Batches > 1`) and performance degrades significantly. Raise `work_mem` per session for heavy analytical joins, not globally.

### Merge Join

Best when both inputs are pre-sorted on the join key (e.g., both have a B-tree index on the join column and the sort is free).

```sql
CREATE INDEX idx_sync_status_obj ON sync_status (object_id, object_type);
CREATE INDEX idx_sync_queue_obj ON sync_queue (object_id, object_type);
-- With both indexes, planner may choose merge join
```

**Common mistake:** Disabling join types globally (`SET enable_hashjoin = off`) to "fix" a slow query. This forces the planner into potentially worse strategies on other queries. Instead, fix the root cause: add indexes, update statistics, or raise `work_mem`.

---

## 6. SQLAlchemy Patterns and Their SQL Output

Understanding what SQLAlchemy generates is critical for optimizing ORM-based applications.

### ORM Query → SQL

```python
from sqlalchemy.orm import Session
from app.models import SyncQueue, SyncStatus

# Basic filter — generates parameterized SELECT
with Session(engine) as session:
    rows = session.query(SyncQueue).filter(
        SyncQueue.status == "pending"
    ).order_by(SyncQueue.created_at.desc()).limit(100).all()

# Generated SQL:
# SELECT sync_queue.id, sync_queue.object_type, ...
# FROM sync_queue
# WHERE sync_queue.status = 'pending'
# ORDER BY sync_queue.created_at DESC
# LIMIT 100
```

### Select() Core Style (preferred for performance-sensitive code)

```python
from sqlalchemy import select, and_
from app.models import SyncQueue

stmt = (
    select(SyncQueue.id, SyncQueue.object_type, SyncQueue.object_id)
    .where(and_(
        SyncQueue.status == "pending",
        SyncQueue.created_at > func.now() - text("interval '1 hour'")
    ))
    .order_by(SyncQueue.created_at.desc())
    .limit(100)
)
rows = session.execute(stmt).all()
```

Core style is more explicit about which columns are selected, making it easier to design covering indexes that match the projection.

### Lazy Loading Trap

```python
# This loads all SyncQueue rows, then fires one SELECT per row to load .account
queues = session.query(SyncQueue).filter(SyncQueue.status == "pending").all()
for q in queues:
    print(q.account.billing_state)  # ← N+1: one query per row
```

The `account` relationship uses lazy loading by default. With 200 rows, this produces 201 queries.

### Fixing with joinedload

```python
from sqlalchemy.orm import joinedload

queues = (
    session.query(SyncQueue)
    .options(joinedload(SyncQueue.account))
    .filter(SyncQueue.status == "pending")
    .all()
)
# Generated SQL (one query):
# SELECT sync_queue.*, account.*
# FROM sync_queue
# LEFT OUTER JOIN account ON account.id = sync_queue.account_id
# WHERE sync_queue.status = 'pending'
```

### selectinload for Large Collections

`joinedload` uses a LEFT JOIN and can produce row multiplication when the relationship is one-to-many. `selectinload` fires a single `IN` query instead.

```python
from sqlalchemy.orm import selectinload

queues = (
    session.query(SyncQueue)
    .options(selectinload(SyncQueue.sync_events))  # one-to-many
    .filter(SyncQueue.status == "retrying")
    .all()
)
# Generated SQL (two queries — no row multiplication):
# SELECT * FROM sync_queue WHERE status = 'retrying'
# SELECT * FROM sync_event WHERE sync_event.queue_id IN (1, 2, 3, ...)
```

**Common mistake:** Using `joinedload` on a one-to-many collection when the parent result set is large. A parent with 1,000 rows and an average of 10 children per row produces 10,000 rows from the LEFT JOIN before deduplication. `selectinload` avoids this entirely.

---

## 7. N+1 Detection and Fixes

### Detecting N+1

**Method 1: SQLAlchemy event logging**

```python
import logging
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
# Watch for repeating SELECT patterns in the log
```

**Method 2: Query count assertion in tests**

```python
from sqlalchemy import event
from contextlib import contextmanager

@contextmanager
def assert_query_count(session, expected: int):
    count = 0
    def count_queries(conn, cursor, statement, *args):
        nonlocal count
        count += 1
    event.listen(session.bind, "before_cursor_execute", count_queries)
    yield
    event.remove(session.bind, "before_cursor_execute", count_queries)
    assert count <= expected, f"Expected {expected} queries, got {count}"

# In test:
with assert_query_count(session, 2):
    queues = session.query(SyncQueue).options(
        selectinload(SyncQueue.account)
    ).filter(SyncQueue.status == "pending").all()
    _ = [q.account.billing_state for q in queues]
```

**Method 3: Datadog APM / pg_stat_statements**

In the crm-middleware, Datadog APM traces show per-endpoint query counts. A spike to 200+ queries on a single `/sync/process` call is a N+1 indicator. Alternatively:

```sql
-- pg_stat_statements: find queries called many times per second
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
WHERE query LIKE '%sync_queue%'
ORDER BY calls DESC
LIMIT 20;
```

### Integration Platform-Specific Fix: Sync Status Bulk Load

The integration platform worker that processes `sync_queue` rows and checks `sync_status` for each had a classic N+1:

```python
# Before (N+1):
pending = session.query(SyncQueue).filter(SyncQueue.status == "pending").all()
for item in pending:
    status = session.query(SyncStatus).filter_by(
        object_id=item.object_id,
        object_type=item.object_type
    ).first()  # 1 query per row
    process(item, status)

# After (2 queries total):
from sqlalchemy import tuple_

pending = session.query(SyncQueue).filter(SyncQueue.status == "pending").all()
keys = [(item.object_id, item.object_type) for item in pending]

statuses = session.query(SyncStatus).filter(
    tuple_(SyncStatus.object_id, SyncStatus.object_type).in_(keys)
).all()

status_map = {(s.object_id, s.object_type): s for s in statuses}
for item in pending:
    process(item, status_map.get((item.object_id, item.object_type)))
```

**Common mistake:** Fixing N+1 with `joinedload` when the relationship is accessed conditionally. If only 10% of iterations access the related object, `selectinload` (lazy IN-query) or explicit bulk fetch is more efficient than eagerly joining all rows.

---

## 8. Index Maintenance and Bloat

Indexes are not free after creation. PostgreSQL MVCC means that `UPDATE` and `DELETE` leave dead versions of rows in heap pages — and corresponding dead entries in index pages. If `autovacuum` cannot keep up, index bloat degrades performance.

```sql
-- Check index bloat (pgstattuple extension required)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT * FROM pgstattuple('idx_sync_queue_status_created');
-- dead_tuple_count, dead_tuple_len, free_space indicate bloat

-- Force a manual VACUUM on a hot table without locking writes:
VACUUM (VERBOSE, ANALYZE) sync_queue;

-- Rebuild an index without locking writes (PG 12+):
REINDEX INDEX CONCURRENTLY idx_sync_queue_status_created;
```

**Monitoring autovacuum:**

```sql
SELECT relname, n_dead_tup, last_vacuum, last_autovacuum,
       autovacuum_count, n_live_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

For the integration platform queue table, rows transition `pending → processing → completed` rapidly. Dead tuple accumulation is high. Consider:

1. Tuning `autovacuum_vacuum_scale_factor` down from 0.2 to 0.01 for `sync_queue`.
2. Archiving completed rows older than 7 days to a `sync_queue_archive` table.

```sql
ALTER TABLE sync_queue SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_analyze_scale_factor = 0.005
);
```

**Common mistake:** Creating indexes on a table and then running a large `DELETE` sweep without vacuuming. The dead index entries remain, bloating the index structure and slowing all scans until the next autovacuum cycle.

---

## 9. Practical Index Design for Integration Platform Queue Tables

Putting it all together with concrete index recommendations for the crm-middleware schema.

```sql
-- sync_queue: primary access patterns
-- 1. Worker polls: status + age, ordered by created_at
CREATE INDEX idx_sq_worker_poll
    ON sync_queue (status, created_at ASC)
    INCLUDE (id, object_id, object_type)
    WHERE status IN ('pending', 'retrying');

-- 2. Deduplication check before insert: object_id + object_type + status
CREATE INDEX idx_sq_dedup
    ON sync_queue (object_id, object_type, status)
    WHERE status != 'completed';

-- 3. Salesforce callback lookup by external ID
CREATE INDEX idx_sq_sf_id
    ON sync_queue USING hash (salesforce_id)
    WHERE status NOT IN ('completed', 'failed');

-- sync_status: accessed via object_id + object_type lookup
CREATE UNIQUE INDEX idx_ss_object
    ON sync_status (object_id, object_type);
CREATE INDEX idx_ss_last_synced
    ON sync_status (last_synced_at DESC)
    WHERE last_synced_at > now() - interval '7 days';
    -- Note: this partial index is for monitoring dashboards only;
    -- the WHERE clause uses now(), which means the index is rebuilt
    -- at plan time. Use a static cutoff column instead for production.
```

**Correct pattern for time-windowed partial indexes:**

```sql
-- Wrong: now() in partial index WHERE is evaluated at index creation time,
-- not at query time. All rows match at creation, none match a week later.
-- Use a separate boolean flag instead:
ALTER TABLE sync_status ADD COLUMN is_recent boolean
    GENERATED ALWAYS AS (last_synced_at > now() - interval '7 days') STORED;
-- Stored generated columns CAN be indexed:
CREATE INDEX idx_ss_recent ON sync_status (last_synced_at) WHERE is_recent;
-- But note: this column will become stale until rows are updated/vacuumed.
-- For truly dynamic time windows, use a regular index + query predicate.
```

---

## Key Concepts Summary

```
PostgreSQL Query Execution
│
├── Parse & Rewrite
│   └── SQL text → parse tree → rewrite rules
│
├── Planner (uses pg_statistic)
│   ├── Index selection
│   │   ├── B-tree  ── equality, range, sort, prefix LIKE
│   │   ├── Hash    ── equality only, smaller for long strings
│   │   ├── GIN     ── JSONB, arrays, full-text
│   │   ├── GiST    ── geometric, range types, full-text
│   │   └── BRIN    ── append-only, physically ordered columns
│   │
│   ├── Index variants
│   │   ├── Partial  ── WHERE clause reduces index size
│   │   ├── Expression── indexed derived value
│   │   └── Covering ── INCLUDE for index-only scans
│   │
│   └── Join strategy
│       ├── Nested Loop ── small outer + indexed inner
│       ├── Hash Join   ── large unsorted inputs
│       └── Merge Join  ── pre-sorted inputs
│
├── Executor
│   ├── Sequential Scan, Index Scan, Index Only Scan, Bitmap Scan
│   └── EXPLAIN ANALYZE shows actual rows, loops, buffers
│
└── Maintenance
    ├── ANALYZE  ── refresh planner statistics
    ├── VACUUM   ── reclaim dead tuple space
    └── REINDEX CONCURRENTLY ── rebuild bloated indexes live
```

```
SQLAlchemy ORM Loading Strategies
│
├── lazy="select"  (default) ── N+1 trap
├── joinedload()   ── LEFT JOIN; good for many-to-one
├── selectinload() ── IN query; good for one-to-many
└── Bulk fetch     ── manual IN() for cross-table patterns
```

---

## Quiz — 20 Questions

### Questions

**1.** You add a B-tree index on `sync_queue(status)` but queries with `WHERE status = 'pending' ORDER BY created_at DESC` still perform a sequential scan. What is the most likely cause and fix?

**2.** Explain the difference between `EXPLAIN` and `EXPLAIN ANALYZE`. When would you use each?

**3.** A GIN index on `sync_queue.payload` (JSONB) makes reads fast but slows down inserts significantly. Why, and what options do you have?

**4.** Your `EXPLAIN ANALYZE` output shows `Batches: 4` on a hash join node. What does this mean and how do you resolve it?

**5.** Describe a scenario in the crm-middleware where a BRIN index would be appropriate and one where it would not.

**6.** What is the `loops` field in `EXPLAIN ANALYZE` output, and why is ignoring it a common mistake?

**7.** You have a partial index `WHERE status IN ('pending', 'retrying')`. Write a query that will use this index and one that will not.

**8.** What is index bloat, what causes it in PostgreSQL, and how do you detect and fix it without downtime?

**9.** Compare `joinedload` and `selectinload` in SQLAlchemy. Give a concrete example where choosing the wrong one causes a performance problem.

**10.** What does `correlation` in `pg_stats` tell you, and how does a low correlation value affect index scan performance?

**11.** Write the SQL to create a covering index for the query: `SELECT id, object_id, object_type FROM sync_queue WHERE status = 'pending' ORDER BY created_at DESC LIMIT 100`.

**12.** A hash join between `sync_queue` (small) and `sync_status` (large) is slow. You suspect the planner chose to hash the wrong table. How do you verify this in `EXPLAIN ANALYZE` output?

**13.** What is a "bitmap scan" in PostgreSQL and when does the planner prefer it over a plain index scan?

**14.** Explain why `WHERE lower(salesforce_id) = lower($1)` will not use a plain B-tree index on `salesforce_id` and how you fix it.

**15.** You run `ANALYZE sync_queue` but the planner is still making poor row count estimates on the `status` column. What else can you do?

**16.** The integration platform worker processes `sync_queue` rows in batches of 500. Describe the N+1 problem that occurs when loading associated `Account` objects and provide the corrected SQLAlchemy code.

**17.** What is the `INCLUDE` clause in a `CREATE INDEX` statement and how does it enable an "Index Only Scan"?

**18.** A merge join requires both inputs to be sorted on the join key. What must be true for PostgreSQL to consider a merge join "free" (i.e., without an explicit sort step)?

**19.** Explain the difference between `autovacuum_vacuum_scale_factor` and `autovacuum_vacuum_threshold` and how you would tune them for a high-churn queue table.

**20.** You are profiling a FastAPI endpoint in the crm-middleware that calls `session.query(SyncQueue).filter(...).all()` followed by per-row attribute access. `pg_stat_statements` shows the SELECT is called 350 times per request. What is happening and how do you fix it?

---

### Answers

??? note "Reveal Answers"

    **1.** With a single-column index on `status`, the planner uses it to filter rows but must then sort the entire result by `created_at`, which may be more expensive than a sequential scan for low-selectivity status values like 'pending'. The fix is a composite index `ON sync_queue (status, created_at DESC)`. This allows the planner to perform an index scan that delivers rows in the correct order with no separate sort step. Alternatively, if 'pending' rows are a small fraction of the table, a partial index `WHERE status = 'pending'` on `(created_at DESC)` further reduces index size.

    **2.** `EXPLAIN` shows the planner's estimated query plan including estimated costs and row counts — it does not execute the query, so it is safe to run on production. `EXPLAIN ANALYZE` actually executes the query and reports actual runtime, actual row counts, and buffer hit/miss statistics. Use `EXPLAIN` when you want to quickly inspect the plan without side effects (e.g., for a DELETE or INSERT). Use `EXPLAIN ANALYZE` on SELECT queries in development or staging to diagnose performance, since it reveals discrepancies between planner estimates and reality.

    **3.** GIN indexes use an inverted structure that must be updated for every changed element in the JSONB value, causing significant write amplification. For every insert or update, PostgreSQL must decompose the JSONB into individual key-value entries and update the inverted postings lists. Options include: (a) making GIN updates asynchronous by setting `gin_pending_list_limit` to batch small updates into a pending list before merging, which reduces per-write overhead at the cost of slightly stale index entries; (b) using `fastupdate = on` (the default) which buffers GIN updates and merges them periodically; (c) accepting the overhead if reads are far more frequent than writes; (d) switching to a partial GIN index if only a subset of rows are queried.

    **4.** `Batches: 4` on a hash join means the hash table did not fit in memory and was spilled to disk in four batches. PostgreSQL wrote temporary files for three overflow partitions and re-read them during the probe phase. This dramatically increases I/O. The fix is to increase `work_mem` for the session (`SET work_mem = '256MB'`) so the hash table fits in memory, resulting in `Batches: 1`. For recurring queries, set `work_mem` at the role or function level rather than globally to avoid inflating total memory usage across all concurrent sessions.

    **5.** BRIN is appropriate for the `sync_audit_log` table (or any append-only event log), where rows are inserted in chronological order and queries filter on `created_at` time ranges — the physical storage order is highly correlated with the column value. BRIN is inappropriate for `sync_queue`, which has rows constantly updated (status transitions), causing physical row versions to scatter across heap pages via HOT updates, breaking the correlation assumption that BRIN relies on.

    **6.** The `loops` field indicates how many times a plan node was executed. In a nested loop join, the inner node executes once per outer row, so `actual time=0.1 ms` with `loops=5000` represents 500 ms of total work for that node. The common mistake is reading the per-loop time as the total time and concluding the inner node is fast, when it is actually the dominant cost in the query. Always multiply `actual time` (both startup and total) by `loops` to get the true contribution of a node to overall query time.

    **7.** A query that will use the partial index `WHERE status IN ('pending', 'retrying')`: `SELECT id FROM sync_queue WHERE status = 'pending' ORDER BY created_at ASC`. The query predicate implies the index condition because 'pending' is a subset of the indexed set. A query that will NOT use the index: `SELECT id FROM sync_queue WHERE status = 'processing'` — 'processing' is not in the partial index predicate, so the planner cannot use this index and must fall back to a sequential scan or another index.

    **8.** Index bloat occurs when `UPDATE` and `DELETE` operations leave dead tuple references in index pages. PostgreSQL's MVCC model creates new row versions for updates rather than modifying in place, and index entries for dead versions cannot be removed until `VACUUM` runs. Bloat is detected via the `pgstattuple` extension (`SELECT * FROM pgstattuple('index_name')`) or by comparing `pg_relation_size` to a freshly rebuilt equivalent. Fix it without downtime using `REINDEX INDEX CONCURRENTLY index_name` (PG 12+), which builds a new index while the old one remains available for reads and writes, then atomically swaps them.

    **9.** `joinedload` adds a `LEFT OUTER JOIN` to the parent query, returning parent-plus-child columns in each row; it works well for many-to-one relationships (e.g., `SyncQueue → Account`) where there is at most one related row. `selectinload` fires a separate `SELECT ... WHERE id IN (...)` for all related objects after loading the parent set; it is better for one-to-many (e.g., `SyncQueue → SyncEvents`). Using `joinedload` on a one-to-many collection with 1,000 parents and 10 children each returns 10,000 rows from the database, requiring deduplication in Python. `selectinload` returns 1,000 parent rows and one 10,000-row children query, avoiding row multiplication entirely.

    **10.** `correlation` in `pg_stats` measures the linear correlation between the physical storage order of rows and the logical sort order of the column values, ranging from -1.0 to 1.0. A correlation near 1.0 means rows are physically stored in approximately the same order as the column values, making index scans very efficient (sequential page reads). A correlation near 0 means rows are scattered randomly relative to the index order, causing each index entry to point to a different heap page — this is called a "random heap access" pattern and can make an index scan slower than a sequential scan on large result sets. The planner uses correlation to estimate whether an index scan will generate random I/O and may choose a sequential scan instead.

    **11.** The covering index for that query is: `CREATE INDEX idx_sq_covering ON sync_queue (status, created_at DESC) INCLUDE (id, object_id, object_type) WHERE status = 'pending';`. The `(status, created_at DESC)` key columns satisfy the WHERE and ORDER BY clauses. The `INCLUDE (id, object_id, object_type)` columns allow the query to be answered entirely from the index leaf pages without touching the heap. The partial index clause `WHERE status = 'pending'` further reduces index size if 'pending' is a minority of rows.

    **12.** In the `EXPLAIN ANALYZE` output for a hash join, the inner `->` node is the one being hashed (the smaller side). If the planner hashed the large `sync_status` table instead of the small `sync_queue` result, the "Hash" node's `actual rows` will be large and `Memory Usage` will be high (or Batches > 1 indicating spill). You can verify by looking at which subplan appears as the second (Hash) child of the Hash Join node and checking its actual row count. If the planner got it backwards, the fix is to update statistics on both tables (`ANALYZE sync_queue; ANALYZE sync_status`) so the planner has accurate row count estimates.

    **13.** A bitmap scan is a two-phase access method: PostgreSQL first scans an index to build an in-memory bitmap of matching heap page addresses (not individual row pointers), then fetches heap pages in physical order matching the bitmap. The planner prefers a bitmap scan over a plain index scan when a predicate matches many rows (medium to high selectivity) because fetching heap pages in physical order minimizes random I/O. It is also used to combine multiple indexes via `BitmapAnd` / `BitmapOr` when no single index covers a compound predicate.

    **14.** A B-tree index on `salesforce_id` stores the raw column values. The query `WHERE lower(salesforce_id) = lower($1)` requires evaluating `lower()` on every row before comparing, which the index on raw values cannot satisfy. The fix is to create an expression index: `CREATE INDEX idx_sq_sf_id_lower ON sync_queue (lower(salesforce_id))`. PostgreSQL will then use this index for queries that filter on `lower(salesforce_id)`. An alternative is to enforce lower-case storage at insert time via a constraint or application layer, then use a plain index on the raw column.

    **15.** After `ANALYZE`, if planner estimates are still poor on a low-cardinality column like `status`, the default statistics target of 100 may not capture enough MCV (most-common-values) entries for a skewed distribution. Increase the per-column statistics target: `ALTER TABLE sync_queue ALTER COLUMN status SET STATISTICS 500; ANALYZE sync_queue;`. This instructs PostgreSQL to track more distinct value frequencies and histogram buckets for that column. Also check that the `status` column has only a few distinct values — if `n_distinct` is small, the planner should use the MCV list rather than histograms, and raising statistics helps it do so more accurately.

    **16.** The N+1 occurs because the worker loads 500 `SyncQueue` rows and then accesses `queue_item.account` inside the loop, triggering one lazy `SELECT * FROM account WHERE id = $1` per iteration — 501 queries total. The fix using `selectinload`:
    ```python
    from sqlalchemy.orm import selectinload
    batch = (
        session.query(SyncQueue)
        .options(selectinload(SyncQueue.account))
        .filter(SyncQueue.status == "pending")
        .limit(500).all()
    )
    for item in batch:
        process(item, item.account)  # no additional query
    ```
    This generates exactly two queries: one for `sync_queue` and one `SELECT * FROM account WHERE id IN (...)` covering all 500 account IDs at once.

    **17.** The `INCLUDE` clause adds non-key columns to the leaf pages of a B-tree index without incorporating them into the tree structure (they are not sorted and cannot be used in index conditions). When a query's SELECT list consists entirely of the index key columns plus included columns, PostgreSQL can answer the query solely from the index pages without accessing the heap — this is an Index Only Scan. Index Only Scans eliminate the heap I/O entirely (shown as `Heap Fetches: 0` in EXPLAIN ANALYZE), which is significant for hot, frequently-queried tables where heap pages may not fit in `shared_buffers`.

    **18.** For a merge join to be "free" (requiring no explicit sort), PostgreSQL needs both inputs to be delivered in the correct sort order by the child plan nodes. This happens when: (a) both tables have B-tree indexes on the join key columns in matching sort order, and the planner chooses Index Scans on both; (b) one or both inputs are the result of a subquery that already includes a sort matching the join key. If either input lacks a suitable index, the planner must add an explicit Sort node above it, which adds O(N log N) cost and memory pressure.

    **19.** `autovacuum_vacuum_threshold` is a fixed minimum number of dead tuples required before autovacuum triggers (default: 50). `autovacuum_vacuum_scale_factor` is a fraction of total live rows (default: 0.20, i.e., 20%). Autovacuum triggers when dead tuples exceed `threshold + scale_factor × n_live_tup`. For a `sync_queue` table with 500,000 live rows, the default threshold is `50 + 0.20 × 500,000 = 100,050` dead tuples before vacuuming. For a high-churn queue, this is too lenient. Set `autovacuum_vacuum_scale_factor = 0.01` and `autovacuum_vacuum_threshold = 100` to trigger vacuum when just 1% (5,000 rows) have been churned, keeping dead tuples under control and indexes lean.

    **20.** The 350 queries per request is the N+1 pattern. The endpoint loads approximately 350 `SyncQueue` objects, and then accesses a lazy-loaded relationship on each one (e.g., `.account`, `.sync_status`), triggering one `SELECT` per row. The fix is to use `selectinload` or `joinedload` on the relationship in the initial query, depending on the cardinality. Additionally, if only specific columns are needed from the related object, projecting them directly with a join in Core style avoids loading full ORM instances: `select(SyncQueue.id, Account.billing_state).join(Account)`. Use `pg_stat_statements` or SQLAlchemy's query logging to confirm the count drops to 1-2 queries after the fix.

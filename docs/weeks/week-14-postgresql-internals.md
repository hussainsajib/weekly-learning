# Week 14 — PostgreSQL Internals

**Week of:** September 7, 2026
**Estimated study time:** ~2 hours
**Tags:** `postgresql` `databases` `internals`

---

## Overview

PostgreSQL is not just a database — it is a carefully engineered system where every decision about durability, concurrency, and storage has cascading consequences on application behavior. Most engineers interact with Postgres at the SQL surface, but understanding the engine beneath that surface is what separates a developer who can write a query from one who can diagnose a production incident at 2 AM. This week you will learn how the engine actually works: how transactions maintain isolation without locks, how dead rows accumulate and get reclaimed, how the WAL protects your data through crashes, and how storage is physically arranged on disk.

For the AESF middleware, PostgreSQL is not just a data store — it is the queue backbone for Epic sync jobs. Rows are inserted, claimed, processed, and deleted at high frequency. This pattern is one of the most demanding workloads for MVCC: every deleted queue row leaves a dead tuple that VACUUM must eventually reclaim. Without a firm understanding of autovacuum tuning, table bloat will silently erode query performance until an index scan degrades to a sequential scan and sync latency spikes. These are not hypothetical concerns — they are the exact failure modes that have caused incidents in queue-heavy systems built on Postgres.

The operational side of Postgres is equally important. Connection pooling with PgBouncer is non-negotiable for any Kubernetes-hosted application where pods scale horizontally, because each PostgreSQL connection is a forked OS process consuming 5–10 MB of RAM. Without a pool, a burst of 50 pods each opening 5 connections will saturate a Cloud SQL instance far faster than the application logic ever could. Understanding how PgBouncer's three pooling modes interact with transaction-scoped features like prepared statements and advisory locks will save you from subtle bugs.

Finally, schema evolution on a live system is a discipline in itself. Alembic is a powerful migration tool, but Postgres's locking model means that innocent-looking DDL statements — adding a non-nullable column, creating an index, altering a column type — can take an `AccessExclusiveLock` that blocks all reads and writes for minutes on a large table. This week you will learn the safe patterns: `CREATE INDEX CONCURRENTLY`, multi-step nullable/backfill/not-null migrations, and how to write Alembic operations that are safe to run against a live AESF production instance.

---

## 1. MVCC — How Transactions Actually Work

Multi-Version Concurrency Control (MVCC) is the mechanism that lets Postgres serve concurrent readers and writers without readers blocking writers or writers blocking readers. The key insight is that Postgres never overwrites a row in place. Instead, every `UPDATE` writes a new physical row version (called a **heap tuple**) and marks the old version as expired. Every `DELETE` marks a row as expired without removing it immediately. The heap file on disk therefore accumulates multiple versions of the same logical row over time.

Each row version carries two hidden system columns: `xmin` (the transaction ID that created this version) and `xmax` (the transaction ID that deleted or updated it, or 0 if the row is still live). When a transaction reads a row, the visibility rules compare the row's `xmin`/`xmax` against the transaction's **snapshot** — a point-in-time view of which transaction IDs were committed when the read began. A row is visible to a snapshot if its `xmin` committed before the snapshot was taken and its `xmax` had not yet committed at snapshot time.

```sql
-- Inspect the hidden system columns directly
SELECT ctid, xmin, xmax, id, status
FROM aesf_queue
WHERE status = 'pending'
LIMIT 5;
```

```
 ctid    | xmin    | xmax | id   | status
---------+---------+------+------+---------
 (0,1)   | 7823401 | 0    | 1001 | pending
 (0,2)   | 7823402 | 0    | 1002 | pending
```

The `ctid` column is the physical location: `(page, tuple_offset)`. After an update, the original tuple gets a non-zero `xmax` and a new `ctid` tuple appears. Both versions exist on disk until VACUUM removes the dead one.

**Isolation levels** control snapshot timing. `READ COMMITTED` (the default) takes a fresh snapshot at the start of each statement. `REPEATABLE READ` and `SERIALIZABLE` take one snapshot at the start of the entire transaction, making them immune to non-repeatable reads but more sensitive to serialization conflicts.

**AESF connection:** Every queue row that transitions from `pending` → `processing` → `done` → deleted leaves dead tuple debris. On a table receiving 10,000 inserts/deletes per hour, you will accumulate millions of dead tuples per day without proper autovacuum configuration.

> **Common mistake:** Assuming that `DELETE FROM queue WHERE status = 'done'` immediately frees disk space. It does not. It only marks rows as dead. Disk space is only reclaimed after VACUUM runs and, in some cases, only after `VACUUM FULL` (which rewrites the entire table). Use `pg_stat_user_tables.n_dead_tup` to monitor this.

---

## 2. VACUUM and Autovacuum

VACUUM has two jobs: reclaiming dead tuples so the space can be reused (but not returned to the OS, unless `VACUUM FULL`), and advancing the **oldest transaction ID** to prevent transaction ID wraparound — one of Postgres's most dangerous failure modes.

Autovacuum is a background daemon that triggers VACUUM automatically based on configurable thresholds. The default trigger is:

```
n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live_tup
```

With defaults of `threshold=50` and `scale_factor=0.2`, a 100,000-row table triggers autovacuum when dead tuples exceed 20,050. For a small, high-churn queue table with 500 live rows, autovacuum triggers at just 150 dead tuples — which is good. For a large 10M-row history table, the threshold is 2,000,050 dead tuples before autovacuum even starts.

```sql
-- Check autovacuum health for AESF tables
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze,
    autovacuum_count
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_dead_tup DESC;
```

For the AESF queue pattern, override the defaults at the table level to trigger more aggressively:

```sql
ALTER TABLE aesf_queue SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- trigger at 1% dead tuples
    autovacuum_vacuum_threshold = 100,        -- or 100 dead tuples minimum
    autovacuum_analyze_scale_factor = 0.005
);
```

**Transaction ID Wraparound** is a catastrophic scenario where Postgres has issued 2^31 transaction IDs since the last aggressive VACUUM, causing it to refuse new writes (or worse, misidentify old live tuples as invisible). Monitor via:

```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2147483648 - age(datfrozenxid) AS xids_remaining
FROM pg_database
ORDER BY xid_age DESC;
```

If `xid_age` exceeds ~1.5 billion, Postgres will start warning. At 2 billion, it enters "safe" read-only mode to prevent corruption.

> **Common mistake:** Running `VACUUM FULL` in production to reclaim disk space. `VACUUM FULL` holds an `AccessExclusiveLock` for the entire duration, blocking all reads and writes. Use it only in a maintenance window. For routine reclamation, rely on regular VACUUM and ensure autovacuum is not being blocked by long-running transactions.

---

## 3. WAL — Write-Ahead Logging

The Write-Ahead Log is the mechanism that makes Postgres crash-safe. The rule is simple: **no data page is written to disk before the WAL record describing that change has been flushed to disk.** On crash recovery, Postgres replays the WAL from the last checkpoint to reconstruct any writes that made it to WAL but not to the heap files.

WAL records are written to sequentially numbered segment files in `pg_wal/`. Each record has a Log Sequence Number (LSN), a monotonically increasing byte offset into the WAL stream. LSNs are used by replication, backup tools (pg_basebackup), and point-in-time recovery (PITR).

```sql
-- Current WAL position
SELECT pg_current_wal_lsn();

-- How far behind is a replica?
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;
```

**Checkpoints** periodically flush dirty data pages from shared_buffers to disk, creating a "safe point" from which recovery can start. The WAL before a checkpoint can be archived or discarded. The `checkpoint_completion_target` setting (default 0.9) spreads the checkpoint I/O over 90% of the `checkpoint_timeout` interval to avoid I/O spikes.

**WAL and Cloud SQL:** On GCP Cloud SQL, WAL is handled internally for high availability and read replicas. You do not directly access `pg_wal/`, but understanding LSN lag helps you diagnose replica staleness when AESF reads from a read replica for reporting queries.

**`wal_level` settings:**
- `minimal` — enough for crash recovery only; replication not possible
- `replica` — default; supports streaming replication and base backups
- `logical` — enables logical decoding (needed for CDC tools like Debezium)

> **Common mistake:** Setting `synchronous_commit = off` to improve write throughput without understanding the trade-off. With async commit, up to `wal_writer_delay` (default 200ms) of committed transactions can be lost on crash. This is acceptable for low-value queue status updates but catastrophic for financial records. In AESF, consider per-transaction `SET LOCAL synchronous_commit = off` only for idempotent queue heartbeat updates.

---

## 4. Heap and TOAST Storage

Postgres stores table data in **heap files** — fixed-size pages (8 KB by default) containing variable-length tuples. Each page has a header, a line pointer array (offsets to each tuple), free space in the middle, and tuples growing from the bottom. `ctid` is the direct address: page number and slot index.

When a row's data exceeds approximately 2 KB (one-quarter of a page), Postgres transparently compresses and/or chunks the oversized attribute into a separate **TOAST table** (`pg_toast.pg_toast_<oid>`). Each oversized value is split into 2 KB chunks and stored as multiple rows in the TOAST table. The main table row contains only a pointer.

```sql
-- Find TOAST tables for AESF tables
SELECT
    c.relname AS main_table,
    t.relname AS toast_table,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_relation_size(t.oid)) AS toast_size
FROM pg_class c
JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public')
ORDER BY pg_total_relation_size(c.oid) DESC;
```

**TOAST strategies** per column (set via `ALTER TABLE ... ALTER COLUMN ... SET STORAGE`):
| Strategy | Meaning |
|----------|---------|
| `PLAIN` | No compression, no TOAST (for small fixed-width types) |
| `EXTENDED` | Compress first, TOAST if still too big (default for `text`, `jsonb`) |
| `EXTERNAL` | TOAST without compression (faster access, larger storage) |
| `MAIN` | Compress in-line, TOAST only as last resort |

**AESF connection:** If the `payload` column on the queue table is `jsonb` and Epic sync payloads exceed 2 KB, every row access requires an extra TOAST fetch. Benchmark with `EXPLAIN (ANALYZE, BUFFERS)` to see if TOAST access (`... on toast table`) is appearing in query plans.

> **Common mistake:** Running `SELECT *` on tables with large TOAST columns when only a subset of columns is needed. Fetching unneeded TOASTed columns adds I/O for each row. Always select only the columns you need.

---

## 5. EXPLAIN ANALYZE in Depth

`EXPLAIN` shows the query plan. `EXPLAIN ANALYZE` actually executes the query and shows real vs. estimated row counts and timing. `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` adds buffer hit/miss counts, which reveal whether the working set fits in `shared_buffers`.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT id, payload, created_at
FROM aesf_queue
WHERE status = 'pending'
  AND created_at < NOW() - INTERVAL '5 minutes'
ORDER BY created_at
LIMIT 100;
```

**Reading the output — key nodes:**

| Node | What it means |
|------|--------------|
| `Seq Scan` | Full table scan; acceptable only for small tables or when fetching >20% of rows |
| `Index Scan` | Uses index to find rows, then fetches heap pages |
| `Index Only Scan` | All needed columns in index; no heap fetch (fastest) |
| `Bitmap Heap Scan` | Collects matching page pointers, then fetches in physical order (good for moderate selectivity) |
| `Hash Join` / `Merge Join` | Join algorithms; Hash Join works well for large tables |
| `Nested Loop` | Efficient when inner side is small and indexed |

**Rows estimate vs. actual mismatch** is the most important signal. If the planner estimates 10 rows but scans 50,000, statistics are stale. Run `ANALYZE table_name` to refresh.

**Buffer counters explained:**
```
Buffers: shared hit=1234 read=56 dirtied=0 written=0
```
- `hit` — page served from `shared_buffers` (RAM); free
- `read` — page fetched from OS page cache or disk; expensive
- `dirtied` — page modified in this query (unusual for SELECT)

```sql
-- Identify slow queries using pg_stat_statements
SELECT
    query,
    calls,
    total_exec_time / calls AS avg_ms,
    rows / calls AS avg_rows,
    stddev_exec_time AS stddev_ms
FROM pg_stat_statements
WHERE query ILIKE '%aesf_queue%'
ORDER BY avg_ms DESC
LIMIT 10;
```

> **Common mistake:** Running `EXPLAIN ANALYZE` on a data-modifying statement (`INSERT`/`UPDATE`/`DELETE`) without wrapping it in a transaction that you roll back. The statement actually executes and modifies data. Always use: `BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;`

---

## 6. pg_stat_* Views for Diagnostics

The `pg_stat_*` family of views is Postgres's built-in observability layer. Knowing which view to query for which symptom is a core DBA skill.

```sql
-- 1. Table-level health: bloat, vacuum, analyze
SELECT relname, seq_scan, idx_scan,
       n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum,
       last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 2. Index usage: find unused indexes (wasted write overhead)
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- 3. Active connections and lock waits
SELECT
    pid,
    usename,
    application_name,
    state,
    wait_event_type,
    wait_event,
    query_start,
    LEFT(query, 80) AS query_snippet
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- 4. Lock contention
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocker.pid AS blocker_pid,
    blocker.query AS blocker_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocker
    ON blocker.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- 5. Cache hit ratio (should be >99% for OLTP)
SELECT
    sum(heap_blks_hit) AS heap_hits,
    sum(heap_blks_read) AS heap_reads,
    round(
        sum(heap_blks_hit)::numeric /
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100,
        2
    ) AS cache_hit_pct
FROM pg_statio_user_tables;
```

**AESF diagnostic runbook:**
1. Queue latency spike → check `pg_stat_user_tables` for `n_dead_tup` on `aesf_queue`; check `last_autovacuum`
2. Connection exhaustion → `pg_stat_activity` grouped by `state`; look for idle-in-transaction sessions
3. Slow query → `pg_stat_statements` ordered by `avg_ms`; then `EXPLAIN ANALYZE` the culprit
4. Replication lag → `pg_stat_replication` for `replay_lsn` delta

> **Common mistake:** Forgetting to enable `pg_stat_statements` extension. On Cloud SQL it is available but must be activated: `CREATE EXTENSION IF NOT EXISTS pg_stat_statements;` and `shared_preload_libraries = 'pg_stat_statements'` in the instance flags (requires restart). Without it, you have no query-level performance history.

---

## 7. Connection Pooling with PgBouncer

Every PostgreSQL backend connection is a forked OS process consuming ~5–10 MB of RAM and file descriptors. A Cloud SQL instance with 8 GB RAM can support roughly 200–400 connections before connection overhead itself becomes the bottleneck. In a GKE deployment where AESF middleware pods scale from 3 to 30 replicas, each with a SQLAlchemy pool of 5, you need 150 database connections at peak — before autovacuum workers, Cloud SQL internal processes, and monitoring agents consume their share.

PgBouncer sits between your application and Postgres, multiplexing many application connections onto a smaller number of actual database connections.

**Three pooling modes:**

| Mode | Connection handoff | SQLAlchemy advisory locks / prepared statements |
|------|--------------------|------------------------------------------------|
| `session` | One server connection per client session lifetime | Fully supported |
| `transaction` | Server connection returned to pool after each transaction | Most features work; no `SET` persistence |
| `statement` | Server connection returned after each statement | Breaks multi-statement transactions entirely |

For AESF, `transaction` mode is the sweet spot: it multiplexes aggressively while remaining compatible with SQLAlchemy's standard usage patterns.

**PgBouncer config for GKE (`pgbouncer.ini`):**

```ini
[databases]
aesf = host=10.0.0.5 port=5432 dbname=aesf_db

[pgbouncer]
listen_port = 5432
listen_addr = 0.0.0.0
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 500
default_pool_size = 20        ; server connections per database+user pair
reserve_pool_size = 5         ; extra connections for burst
reserve_pool_timeout = 5      ; seconds before using reserve pool
server_idle_timeout = 600     ; close idle server connections after 10 min
client_idle_timeout = 0       ; keep client connections open (managed by app)
log_connections = 0           ; reduce noise in production
log_disconnections = 0
server_tls_sslmode = require
```

**SQLAlchemy pool config when using PgBouncer in transaction mode:**

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg2://user:pass@pgbouncer:5432/aesf_db",
    pool_size=5,           # per-process pool; PgBouncer multiplexes further
    max_overflow=10,
    pool_pre_ping=True,    # detect stale connections
    pool_recycle=1800,     # recycle connections after 30 min
    connect_args={
        "options": "-c statement_timeout=30000"  # 30s statement timeout
    }
)
```

> **Common mistake:** Using SQLAlchemy's `server_side_cursors` or `LISTEN`/`NOTIFY` with PgBouncer in transaction mode. These require a persistent server connection across statement boundaries, which transaction mode does not provide. Switch to session mode or bypass PgBouncer for these specific use cases.

---

## 8. Alembic Migration Best Practices for Live Systems

Alembic is the standard migration tool for SQLAlchemy projects, but its default generated operations can cause production outages on large tables. The danger is Postgres's locking model: most DDL statements take an `AccessExclusiveLock` that blocks all concurrent reads and writes on the affected table.

**Lock-safe migration patterns:**

**Adding a nullable column** — safe, near-instant:
```python
# migrations/versions/0042_add_retry_count.py
def upgrade():
    op.add_column(
        'aesf_queue',
        sa.Column('retry_count', sa.Integer(), nullable=True)
    )
```

**Adding a NOT NULL column with a default** — unsafe if done naively on large tables. The safe three-step pattern:

```python
# Step 1: Add nullable (fast)
def upgrade():
    op.add_column('aesf_queue',
        sa.Column('priority', sa.Integer(), nullable=True))

# Step 2 (separate migration, after backfill): Set default + NOT NULL
# BUT first backfill in batches from application code or a separate script:
#   UPDATE aesf_queue SET priority = 0 WHERE priority IS NULL;
# Use batches of 10,000 rows to avoid long-running transactions.

# Step 3 (separate migration): Set NOT NULL constraint
def upgrade():
    # Use NOT VALID to skip scanning existing rows, then VALIDATE separately
    op.execute("""
        ALTER TABLE aesf_queue
        ALTER COLUMN priority SET DEFAULT 0
    """)
    op.execute("""
        ALTER TABLE aesf_queue
        ADD CONSTRAINT aesf_queue_priority_not_null
        CHECK (priority IS NOT NULL) NOT VALID
    """)

# Step 4 (separate migration, off-peak): Validate
def upgrade():
    op.execute("""
        ALTER TABLE aesf_queue
        VALIDATE CONSTRAINT aesf_queue_priority_not_null
    """)
```

**Creating indexes without blocking writes — always use CONCURRENTLY:**

```python
from alembic import op

def upgrade():
    # Standard op.create_index acquires a ShareLock (blocks writes)
    # Use raw SQL with CONCURRENTLY instead
    op.execute("""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS
        idx_aesf_queue_status_created
        ON aesf_queue (status, created_at)
        WHERE status IN ('pending', 'processing')
    """)

def downgrade():
    op.execute("""
        DROP INDEX CONCURRENTLY IF EXISTS idx_aesf_queue_status_created
    """)
```

**Important:** `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. Alembic wraps migrations in transactions by default. Disable this for concurrent index migrations:

```python
# At module level in the migration file
from alembic.operations import MigrateRevisions

def upgrade():
    ...

# Disable transaction wrapping for this migration
migration_context = op.get_context()
# OR use the connection directly:

# In env.py, handle per-migration transaction control:
# context.run_migrations() should check migration.transactional flag
```

The cleanest pattern is a migration that explicitly calls `op.execute` with `CONCURRENTLY` and sets `transactional = False` in the migration metadata.

**Zero-downtime column rename:**
```python
# Never rename directly — it breaks running app instances
# Step 1: Add new column, write to both, read from old
# Step 2: Backfill new from old
# Step 3: Switch reads to new column
# Step 4: Stop writing to old
# Step 5: Drop old column
```

> **Common mistake:** Running `op.alter_column(..., nullable=False)` on a live table with millions of rows. This rewrites every row and holds `AccessExclusiveLock` for the duration. Use the `NOT VALID` constraint + `VALIDATE CONSTRAINT` in a separate off-peak migration to split the lock into a short exclusive lock + a weaker share-update-exclusive lock.

---

## 9. Queue Table Patterns and Bloat Management

The AESF queue pattern — high-insert, high-delete, relatively small live set — is one of the most demanding for MVCC. Here is a consolidated set of recommendations for managing it in production.

**Partitioning by status or date** eliminates historical bloat:

```sql
-- Partition queue by processing date (new rows always in current partition)
CREATE TABLE aesf_queue (
    id          BIGSERIAL,
    status      TEXT NOT NULL DEFAULT 'pending',
    payload     JSONB NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMPTZ
) PARTITION BY RANGE (created_at);

CREATE TABLE aesf_queue_2026_09
    PARTITION OF aesf_queue
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');

-- Old partitions can be dropped instantly (no VACUUM needed, just DROP TABLE)
DROP TABLE aesf_queue_2026_07;  -- instant, no lock on parent
```

**Monitoring bloat ratio:**

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1)
        AS dead_pct,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_stat_user_tables
JOIN pg_class ON relname = pg_class.relname
WHERE schemaname = 'public'
  AND n_live_tup + n_dead_tup > 1000
ORDER BY dead_pct DESC;
```

Alert when `dead_pct > 20%` — that indicates autovacuum is falling behind.

**`pg_repack`** is the production-safe alternative to `VACUUM FULL` for reclaiming bloated space without a full table lock. It builds a new version of the table online, then swaps with a brief lock. Available as an extension on Cloud SQL via the `pg_repack` flag.

> **Common mistake:** Treating the queue table as an append-only log and never purging completed records. Even with autovacuum running, a table that grows unboundedly will have an ever-larger free-space map and index pages, degrading all operations proportionally.

---

## 10. Key Concepts Summary

```
PostgreSQL Internals
├── MVCC
│   ├── xmin / xmax per tuple
│   ├── Snapshots determine visibility
│   ├── Dead tuples accumulate on UPDATE/DELETE
│   └── Isolation levels: READ COMMITTED / REPEATABLE READ / SERIALIZABLE
│
├── Storage
│   ├── Heap files (8 KB pages, ctid addressing)
│   └── TOAST (chunked overflow for large columns)
│
├── Durability
│   ├── WAL (Write-Ahead Log)
│   │   ├── LSN: monotonic byte offset
│   │   ├── Checkpoints flush dirty pages
│   │   └── Replication via WAL streaming
│   └── synchronous_commit trade-off
│
├── Maintenance
│   ├── VACUUM: reclaim dead tuples, advance frozen XID
│   ├── Autovacuum: threshold + scale_factor triggers
│   └── VACUUM FULL: rewrites table (needs maintenance window)
│
├── Performance
│   ├── EXPLAIN ANALYZE
│   │   ├── Node types: Seq/Index/Bitmap/Index-Only Scan
│   │   └── Buffers: hit vs. read
│   ├── pg_stat_statements: query-level metrics
│   └── pg_stat_user_tables: bloat, vacuum, cache
│
├── Connections
│   └── PgBouncer
│       ├── session / transaction / statement modes
│       ├── transaction mode for GKE scale-out
│       └── Avoid with LISTEN/NOTIFY, server-side cursors
│
└── Migrations (Alembic)
    ├── CREATE INDEX CONCURRENTLY (no transaction wrap)
    ├── NOT VALID constraint → VALIDATE separately
    ├── Three-step NOT NULL column addition
    └── Never rename columns in-place on live tables
```

---

## Quiz — 20 Questions

### Questions

**1.** What are `xmin` and `xmax` in a PostgreSQL heap tuple, and what do they represent?

**2.** Why does a `DELETE` statement in PostgreSQL not immediately free disk space?

**3.** What is the difference between `VACUUM` and `VACUUM FULL` in terms of locking and behavior?

**4.** What happens if transaction ID wraparound is not prevented, and how do you monitor distance to wraparound?

**5.** Explain the relationship between checkpoints and WAL. Why is `checkpoint_completion_target` set to 0.9 by default?

**6.** What is TOAST and when does PostgreSQL use it automatically?

**7.** In `EXPLAIN ANALYZE` output, what is the significance of a large discrepancy between estimated rows and actual rows?

**8.** What does `shared hit` vs `read` in the `EXPLAIN (ANALYZE, BUFFERS)` output tell you about query performance?

**9.** What is the difference between PgBouncer's `transaction` mode and `session` mode, and which features break in `transaction` mode?

**10.** Why can `CREATE INDEX` block production traffic, and what is the safe alternative?

**11.** What `pg_stat_*` view would you query to find the table with the highest number of dead tuples in the AESF database?

**12.** What does the autovacuum trigger formula `n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live_tup` mean in practice for a 500-row queue table with default settings?

**13.** Why is `op.alter_column(..., nullable=False)` dangerous on a large live table, and what is the safe multi-step alternative?

**14.** What is `synchronous_commit = off` and when is it appropriate to use it in AESF?

**15.** How does `pg_stat_statements` help diagnose performance regressions, and what must be done to enable it on Cloud SQL?

**16.** What is the `ctid` column in PostgreSQL, and how does it change after an UPDATE?

**17.** Describe the Bitmap Heap Scan node in `EXPLAIN`. When does the planner choose it over a plain Index Scan?

**18.** What is the WAL `wal_level` setting `logical` used for, and how does it differ from `replica`?

**19.** How does table partitioning help manage bloat in the AESF queue table pattern?

**20.** What is `pg_repack` and how does it differ from `VACUUM FULL` for reclaiming bloated space?

---

### Answers

??? note "Reveal Answers"

    **1.** `xmin` is the transaction ID of the transaction that inserted this tuple version (i.e., created it). `xmax` is the transaction ID that deleted or superseded this tuple (via UPDATE or DELETE), or 0 if the row is still live. Together they define the tuple's visibility window: a reading transaction checks whether `xmin` committed before its snapshot and whether `xmax` had not committed at snapshot time. This is the core of MVCC — multiple versions of the same row can coexist on disk, each visible only to transactions whose snapshots fall within the appropriate window.

    **2.** PostgreSQL's MVCC model never overwrites rows in place. A `DELETE` simply sets the `xmax` field of the tuple to the deleting transaction's ID, marking it as expired. The tuple remains physically on disk until VACUUM processes the page and determines that no active transaction can ever see the old version. Only then is the space marked as reusable. This design avoids locking readers but means disk space reclamation is deferred and depends on VACUUM running regularly.

    **3.** Regular `VACUUM` reclaims dead tuple space for reuse within the same table file but does not shrink the file itself or return space to the OS. It acquires a `ShareUpdateExclusiveLock`, which does not block normal reads and writes. `VACUUM FULL` rewrites the entire table into a new file, reclaiming all bloat and returning space to the OS, but it holds an `AccessExclusiveLock` for the entire duration, blocking all reads and writes. Never run `VACUUM FULL` on production tables during business hours without a maintenance window.

    **4.** PostgreSQL uses 32-bit transaction IDs, meaning after approximately 2.1 billion transactions, the XID counter would wrap around and old "future" XIDs would be misidentified as invisible, potentially corrupting data. Before this happens, Postgres aggressively autovacuums to "freeze" old tuples (replacing their `xmin` with a special frozen marker), preventing them from being affected by wraparound. Monitor distance to wraparound with `SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age DESC`. If `age` exceeds 1.5 billion, investigate why aggressive autovacuum is not keeping up; at 2 billion, Postgres enters a read-only safety mode.

    **5.** A checkpoint flushes all dirty shared_buffer pages to disk, creating a known-good recovery point. WAL records the changes to these pages; after a checkpoint, older WAL segments can be archived or recycled because recovery can start from the checkpoint rather than replaying the entire WAL. Setting `checkpoint_completion_target = 0.9` tells Postgres to spread the checkpoint I/O across 90% of the `checkpoint_timeout` interval (default 5 minutes), smoothing disk writes to ~4.5 minutes of steady I/O rather than a single burst. This prevents the I/O spike that would otherwise cause latency spikes in concurrent queries.

    **6.** TOAST (The Oversized-Attribute Storage Technique) is Postgres's mechanism for storing column values that are too large to fit in a standard 8 KB page. When a column value exceeds approximately 2 KB (one-quarter of a page), Postgres compresses it and/or chunks it into 2 KB pieces stored in a separate TOAST table. The main table row contains only a pointer to the TOAST chunks. This happens automatically for variable-length types like `text`, `bytea`, and `jsonb`. The storage strategy per column controls whether Postgres prefers compression, external chunking, or inline storage.

    **7.** A large discrepancy between estimated rows and actual rows indicates that the query planner's statistics are stale or inaccurate. The planner uses statistics collected by `ANALYZE` (stored in `pg_statistic`) to estimate selectivity. If estimates are far off, the planner may choose a suboptimal plan — for example, using a Nested Loop join expecting 10 rows from the inner side, but actually getting 50,000. This causes catastrophic performance. Fix by running `ANALYZE table_name` to refresh statistics, or by increasing `default_statistics_target` for columns with skewed distributions.

    **8.** `shared hit` means the data page was found in `shared_buffers` (PostgreSQL's in-memory buffer cache) — effectively free from a latency perspective. `read` means the page had to be fetched from the OS page cache or physical disk. A high `read` count on a frequently-queried table means the working set exceeds `shared_buffers` size. For AESF, if the queue table's active pages consistently appear as `read` rather than `hit`, increasing `shared_buffers` (Cloud SQL memory configuration) or ensuring the table is compact (via VACUUM) will improve performance.

    **9.** In `session` mode, a client gets one dedicated server connection for its entire session lifetime — equivalent to no pooling from Postgres's perspective. In `transaction` mode, the server connection is returned to the pool after each transaction commits or rolls back, allowing many more clients to share fewer server connections. Features that rely on persistent per-connection server state break in transaction mode: `SET` commands (except those in a transaction), `PREPARE` (prepared statements), `LISTEN`/`NOTIFY`, advisory locks held across transactions, and `DECLARE CURSOR` outside a transaction. SQLAlchemy's standard ORM usage is compatible with transaction mode.

    **10.** `CREATE INDEX` (without `CONCURRENTLY`) acquires a `ShareLock` on the table, which blocks all writes (INSERT, UPDATE, DELETE) for the entire duration of the index build. On a large table, this can take minutes. The safe alternative is `CREATE INDEX CONCURRENTLY`, which builds the index in multiple passes using weaker locks, allowing reads and writes to continue throughout. The trade-off is that it takes longer and uses more CPU. It also cannot run inside a transaction, so Alembic migrations creating concurrent indexes must disable transaction wrapping for that specific migration file.

    **11.** `pg_stat_user_tables` is the view to query. It has columns `n_live_tup` and `n_dead_tup` showing live and dead tuple counts per table, along with `last_autovacuum` and `last_vacuum` timestamps. Sort by `n_dead_tup DESC` to find the most bloated tables. Combine with `pg_class` to get table sizes via `pg_total_relation_size(oid)`. A high `n_dead_tup` combined with a stale `last_autovacuum` timestamp indicates autovacuum is not keeping up, which may require tuning `autovacuum_vacuum_scale_factor` at the table level.

    **12.** With default `autovacuum_vacuum_threshold = 50` and `autovacuum_vacuum_scale_factor = 0.2`, the trigger for a 500-row table is `50 + 0.2 * 500 = 150` dead tuples. This means autovacuum fires relatively quickly for this small table — after just 150 deletions or updates without VACUUM. This is actually appropriate for the AESF queue pattern. However, if the queue grows temporarily to 100,000 rows, the trigger jumps to 20,050 dead tuples, which may allow significant bloat to accumulate. Setting `autovacuum_vacuum_scale_factor = 0.01` at the table level keeps the threshold proportionally tight as the table grows.

    **13.** `ALTER TABLE ... ALTER COLUMN ... SET NOT NULL` scans the entire table to verify no existing row has a NULL value, and holds an `AccessExclusiveLock` throughout — blocking all reads and writes. On a million-row table, this could take minutes. The safe pattern is: (1) add the column as nullable, (2) backfill existing rows in small batches from the application to avoid long transactions, (3) add a `CHECK (col IS NOT NULL) NOT VALID` constraint (which only locks briefly to record the constraint, skipping existing rows), then (4) `VALIDATE CONSTRAINT` in a separate migration during off-peak hours. `VALIDATE CONSTRAINT` only requires a `ShareUpdateExclusiveLock`, which does not block reads or writes.

    **14.** `synchronous_commit = off` tells Postgres not to wait for the WAL record to be flushed to disk before returning success to the client. This improves write throughput and reduces latency but means up to `wal_writer_delay` (default 200ms) of recently committed transactions could be lost if the database server crashes — even though the client received a success acknowledgment. This is acceptable for low-value, idempotent operations like queue heartbeat updates (where a re-processing of the same job is harmless) but unacceptable for financial records or any data that must not be lost. Use `SET LOCAL synchronous_commit = off` within a specific transaction to apply it selectively without changing the server default.

    **15.** `pg_stat_statements` records query execution statistics — total calls, total time, average time, standard deviation, and rows affected — grouped by query pattern (with literal values replaced by `$1`, `$2` etc.). Sorting by `avg_ms` or `total_exec_time` quickly reveals which queries are the top contributors to database load, even across many short executions. To enable it on Cloud SQL, you must (1) add `pg_stat_statements` to the `shared_preload_libraries` database flag (requires instance restart), and (2) run `CREATE EXTENSION pg_stat_statements` in each database where you want statistics collected.

    **16.** `ctid` is the physical tuple identifier in PostgreSQL, in the format `(page_number, tuple_offset_within_page)`. It directly addresses where the row lives on disk. After an `UPDATE`, the original row's `ctid` is unchanged but its `xmax` is set; a new row version is written to a different physical location with a new `ctid`. The new version's `ctid` will differ from the original. Because `ctid` is a physical address, it changes after any row movement (VACUUM, CLUSTER, `pg_repack`). Never use `ctid` as a stable row identifier in application logic.

    **17.** A Bitmap Heap Scan is a two-phase operation: first, a Bitmap Index Scan collects all matching tuple locations (TIDs) from the index into an in-memory bitmap, sorted by physical page order; then, the Bitmap Heap Scan fetches the actual heap pages in physical order, reducing random I/O. The planner chooses it when a query matches a moderate fraction of rows — too many for a plain Index Scan (which would make random heap fetches for each row) but too few for a Seq Scan. It is common for range queries on indexed columns with moderate selectivity, such as `WHERE created_at BETWEEN x AND y` on the queue table.

    **18.** `wal_level = logical` enables logical decoding, which allows external tools to consume the WAL as a stream of logical changes (INSERT, UPDATE, DELETE with before/after row values) rather than the physical byte-level changes. This is the foundation for Change Data Capture (CDC) tools like Debezium, pglogical, and logical replication slots. `replica` (the default) only supports streaming replication with physical WAL, which requires the replica to be on the same Postgres major version and architecture. Logical replication allows replicating to different Postgres versions, different schemas, or non-Postgres destinations like Kafka or BigQuery.

    **19.** Partitioning the queue table by date (or by status+date) means that old, fully-processed partitions can be dropped with a single `DROP TABLE` on the child partition — an instantaneous operation that acquires only a brief lock on the partition metadata, not the parent table. This completely bypasses the VACUUM problem for historical data: instead of MVCC accumulating dead tuples that VACUUM must reclaim, you simply drop the entire old partition. The parent table's live partitions remain small and focused on current data, keeping index sizes manageable and autovacuum effective.

    **20.** `pg_repack` is a PostgreSQL extension that reorganizes tables and indexes online — rebuilding them into a compact form without bloat — while allowing concurrent reads and writes throughout the process. It works by creating a new copy of the table, using triggers to replay concurrent changes, and then doing a brief lock swap at the end. This contrasts with `VACUUM FULL`, which holds an `AccessExclusiveLock` for the entire rewrite duration, blocking all access. `pg_repack` is the production-safe choice for reclaiming significant bloat on tables that cannot afford a maintenance window. On Cloud SQL, it is available as an extension that must be enabled via the Cloud Console or Terraform.

# Week 17 — System Design — Scalability Patterns

**Week of:** September 28, 2026
**Estimated study time:** ~2 hours
**Tags:** `system-design` `scalability` `architecture`

---

## Overview

Scalability is the ability of a system to handle increasing load by adding resources, either by upgrading existing machines or by adding more of them. For engineers moving from senior to staff level, the distinction is not just knowing *what* to scale but being able to reason through *where* the bottleneck actually lives — network, compute, database, or application state — and choosing the right lever. Getting this wrong is expensive: over-engineering leads to unnecessary operational complexity, while under-engineering leads to outages under load.

The AESF middleware is a real-world case study for this week. It is a FastAPI application running on GKE, synchronizing data between Salesforce and the Epic EHR backend. Every design decision explored here — horizontal scaling, stateless services, idempotency, CQRS, queue table design, and rate limiting — maps directly to problems already present or latent in that system. Reading this guide with your own codebase open will anchor the concepts.

Modern distributed systems rarely fail because of a single overwhelming event. They fail because of accumulating small design compromises: a sync handler that assumes it is the only writer, a database query that was fast at 1,000 rows but crawls at 10 million, or an upstream API that starts throttling at exactly the moment your load is highest. Rate limiting algorithms, read replicas, and idempotency keys are the mundane engineering that prevents these slow-motion failures.

This guide covers eight core scalability patterns with concrete code, diagrams, and direct AESF connections. Work through each section end to end, then attempt the quiz before revealing answers. The goal is not memorization but the ability to walk into a design review and defend trade-offs with evidence.

---

## 1. Horizontal vs. Vertical Scaling

**Vertical scaling** (scale-up) means upgrading a single machine: more CPU cores, more RAM, faster disk. It is simple to implement — no code changes, no coordination — but has a hard ceiling and a single point of failure.

**Horizontal scaling** (scale-out) means adding more instances of the same service behind a load balancer. It has no theoretical ceiling and provides redundancy, but demands that each instance be stateless — no local memory, no local session, no local file state that another instance cannot see.

```
Vertical (scale-up)          Horizontal (scale-out)
┌──────────────────┐          ┌────────┐ ┌────────┐ ┌────────┐
│  Big Machine     │          │ Small  │ │ Small  │ │ Small  │
│  32 CPU / 128 GB │    vs    │  Pod   │ │  Pod   │ │  Pod   │
│  Single Pod      │          └────────┘ └────────┘ └────────┘
└──────────────────┘               ↑           ↑           ↑
                                   └──── Load Balancer ────┘
```

**AESF connection:** The middleware `deployment-manifests` Helm chart configures a `Deployment` with a `replicas` field. Bumping replicas from 1 to 3 is horizontal scaling — but it only works correctly if the FastAPI app holds no in-process state between requests. If any handler caches a Salesforce session token in a module-level variable, replica 2 will not have it, and half your traffic will fail.

```yaml
# kubernetes/values.staging.yaml (simplified)
middleware:
  replicaCount: 3
  resources:
    requests:
      cpu: "250m"
      memory: "512Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
```

```python
# BAD: module-level mutable state breaks horizontal scaling
_sf_session = None  # only replica 0 will have this after warmup

# GOOD: re-acquire or inject per-request
async def get_sf_client(settings: Settings = Depends(get_settings)) -> SalesforceClient:
    return SalesforceClient(
        username=settings.sf_username,
        password=settings.sf_password,
        security_token=settings.sf_token,
    )
```

**Common mistake:** Teams add replicas but forget to move session/token caching to Redis or the database. The system appears to work — requests succeed — but a fraction fail mysteriously because they land on a replica without a warm session. The fix is to treat every in-process cache as a liability and externalize it.

---

## 2. Load Balancing Strategies

A load balancer sits in front of your replicas and distributes incoming requests. The strategy determines how that distribution happens.

| Strategy | How it works | Best for |
|---|---|---|
| Round-robin | Requests cycle through replicas in order | Stateless services with uniform request cost |
| Least connections | Route to the replica with fewest active connections | Long-lived connections, WebSockets |
| IP hash | Hash client IP to a consistent replica | Sticky sessions (avoid if possible) |
| Weighted round-robin | Replicas get traffic proportional to weight | Mixed instance sizes |
| Random | Random selection | Simple, surprisingly effective |

GKE's default `Service` of type `ClusterIP` uses `iptables` rules that approximate random selection. For more control, a Gateway API or Ingress with an annotation selects the algorithm explicitly.

```yaml
# Nginx Ingress with least-conn upstream
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: middleware-ingress
  annotations:
    nginx.ingress.kubernetes.io/upstream-hash-by: ""
    nginx.ingress.kubernetes.io/load-balance: "least_conn"
spec:
  rules:
    - host: middleware.internal.aesf.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: middleware-svc
                port:
                  number: 8000
```

**AESF connection:** The middleware receives Salesforce webhook callbacks (triggered by Apex). These are short, uniform HTTP calls — round-robin or random is appropriate. There is no need for sticky sessions because the app should be stateless (see Section 1). If sticky sessions were used as a workaround for stateful code, that is a red flag, not a feature.

**Common mistake:** Choosing IP-hash load balancing to work around stateful application design. This creates hot spots when clients cluster at NAT gateways (corporate networks, Salesforce outbound IPs), overloading one replica while others sit idle.

---

## 3. Stateless Service Design

A service is stateless when a request can be handled by *any* replica with the same result. This means no local files, no local caches, no local queues, and no in-process sessions that are not shared.

State must live somewhere — the rule is that it moves out of the application tier and into shared infrastructure: a database, a distributed cache (Redis, Memcached), or an object store (GCS).

```
Stateful (bad for scaling)        Stateless (good for scaling)
┌─────────────────────┐           ┌─────────┐   ┌─────────┐
│   FastAPI Pod       │           │   Pod 1 │   │   Pod 2 │
│   + in-memory queue │           └────┬────┘   └────┬────┘
│   + session cache   │                │              │
│   + temp files      │                └──────┬───────┘
└─────────────────────┘                       │
                                    ┌─────────▼─────────┐
                                    │  PostgreSQL        │
                                    │  + Redis           │
                                    │  + GCS bucket      │
                                    └───────────────────-┘
```

For FastAPI specifically, statelessness means:

1. No module-level mutable objects shared across requests.
2. Background tasks that need coordination use the database, not in-process state.
3. File uploads stream directly to GCS, not local `/tmp`.

```python
# app/api/deps.py
from functools import lru_cache
from app.core.config import Settings

@lru_cache()
def get_settings() -> Settings:
    # lru_cache is acceptable here: Settings is immutable config, not mutable state
    return Settings()

# app/db/session.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(settings.database_url, pool_size=10, max_overflow=20)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
        # session is scoped to this request — no shared mutable state
```

**AESF connection:** The AESF middleware's queue tables in PostgreSQL are the canonical example of correctly externalizing state. The sync queue (`aesf_sync_queue` or equivalent) persists between replicas because it lives in Postgres, not in process memory. Each worker replica can poll and claim jobs independently.

**Common mistake:** Using FastAPI's `startup` event to populate a module-level dict with reference data (e.g., Epic lookup codes), then mutating it during runtime. The dict diverges across replicas. Replace with a Redis cache keyed by version hash, or re-fetch from the database per request for small tables.

---

## 4. Database Read Replicas and Write Routing

When the database becomes the bottleneck (not the application tier), the first lever is separating reads from writes. PostgreSQL supports streaming replication out of the box: a primary receives all writes and streams WAL (Write Ahead Log) records to one or more read replicas.

```
                ┌──────────────────────────────────┐
                │            Application           │
                └──────┬───────────────────┬───────┘
                  Writes (INSERT/UPDATE)  Reads (SELECT)
                       ↓                       ↓
               ┌───────────────┐    ┌──────────────────────┐
               │   Primary DB  │───▶│  Read Replica 1      │
               │  (read/write) │    │  Read Replica 2      │
               └───────────────┘    └──────────────────────┘
                  WAL streaming →
```

In SQLAlchemy, this is implemented by maintaining two engine instances and routing at the session level:

```python
# app/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from app.core.config import get_settings

settings = get_settings()

write_engine = create_async_engine(settings.db_primary_url, pool_size=5)
read_engine  = create_async_engine(settings.db_replica_url, pool_size=10)

WriteSession = sessionmaker(write_engine, class_=AsyncSession, expire_on_commit=False)
ReadSession  = sessionmaker(read_engine,  class_=AsyncSession, expire_on_commit=False)

# In your route handlers:
# - Use WriteSession for INSERT/UPDATE/DELETE
# - Use ReadSession for SELECT-only queries
```

**Replication lag** is the key hazard. After a write completes on the primary, the replica may be milliseconds to seconds behind. If you write a record and immediately read it back from the replica, you may see the old value ("read-your-own-writes" violation). The fix is to route reads that immediately follow a write back to the primary, or to add a small intentional delay before redirecting to the replica.

**AESF connection:** The sync queue is write-heavy (Salesforce events land continuously) but the admin panel and reporting queries are read-heavy. Splitting these onto separate sessions reduces primary load during high-sync periods. Any "last synced at" display in the admin panel is a good candidate for replica reads — slight staleness is acceptable.

**Common mistake:** Pointing the read replica connection string at the primary because "it was easier during setup." The replica is never used, the primary remains the bottleneck, and the team wonders why adding replicas did not help.

---

## 5. CQRS — Command Query Responsibility Segregation

CQRS formalizes the read/write split at the *application model* level, not just the database level. Commands (writes) go through one model with validation and business logic; queries (reads) go through a separate, optimized read model — often a denormalized view or a separate table.

```
                    ┌────────────────────────────────┐
  Salesforce        │         Command Side            │
  Webhook ─────────▶│  POST /sync/account             │
                    │  Validates → writes queue table │
                    └────────────┬───────────────────-┘
                                 │ async worker
                    ┌────────────▼───────────────────-┐
                    │         Epic BDE Backend         │
                    │   PUT /clients (Epic API)        │
                    └────────────┬───────────────────-┘
                                 │ result written back
                    ┌────────────▼───────────────────-┐
  Admin Panel ─────▶│         Query Side               │
                    │  GET /sync/status?account_id=X  │
                    │  Reads from read-optimized view  │
                    └────────────────────────────────-┘
```

In the AESF middleware, a natural CQRS boundary already exists: the Apex trigger fires a POST (command) that enqueues a sync job, and the admin panel displays sync status (query). If these share the same model, changes to the write model for performance (adding JSON blobs, splitting tables) break the read model. Separating them explicitly gives each side freedom to evolve independently.

```python
# Command model — validates and enqueues
class SyncAccountCommand(BaseModel):
    salesforce_id: str
    epic_client_id: Optional[str]
    payload: dict
    triggered_at: datetime

# Query model — optimized for display
class SyncStatusView(BaseModel):
    salesforce_id: str
    last_sync_at: Optional[datetime]
    status: Literal["pending", "in_progress", "success", "failed"]
    failure_reason: Optional[str]
    retry_count: int

# In the router:
@router.post("/sync/account")
async def enqueue_account_sync(cmd: SyncAccountCommand, db: AsyncSession = Depends(get_db)):
    job = SyncQueueItem(sf_id=cmd.salesforce_id, payload=cmd.payload, status="pending")
    db.add(job)
    await db.commit()
    return {"queued": True}

@router.get("/sync/status/{sf_id}", response_model=SyncStatusView)
async def get_sync_status(sf_id: str, db: AsyncSession = Depends(get_read_db)):
    row = await db.execute(
        text("SELECT * FROM sync_status_view WHERE salesforce_id = :id"),
        {"id": sf_id}
    )
    return row.mappings().one_or_none()
```

**Common mistake:** Applying CQRS everywhere "because it is a best practice." For a simple CRUD service, CQRS adds two models, two data paths, and eventual consistency complexity with no benefit. Apply it where the write path and read path have genuinely divergent performance or complexity requirements.

---

## 6. Database Sharding

Sharding is horizontal partitioning of a database: rows are split across multiple database servers (shards) by a shard key. Each shard holds a subset of the data and can run on independent hardware, removing the single-machine limit on database capacity.

```
Shard Key: salesforce_org_id % 4

  org_id ends in 0,4,8 → Shard 0 (PostgreSQL instance A)
  org_id ends in 1,5,9 → Shard 1 (PostgreSQL instance B)
  org_id ends in 2,6   → Shard 2 (PostgreSQL instance C)
  org_id ends in 3,7   → Shard 3 (PostgreSQL instance D)
```

Sharding is complex to operate: cross-shard joins require application-level logic, schema migrations must run on every shard in coordination, and rebalancing shards when adding new instances is expensive. It is the option of last resort after connection pooling, read replicas, caching, and query optimization are exhausted.

For AESF's scale today, PostgreSQL table partitioning (a single-server feature) is far more appropriate than true multi-server sharding. Partition the sync queue by `created_at` month — Postgres handles the routing, and old partitions can be detached and archived cleanly.

```sql
-- Partitioned sync queue (by month)
CREATE TABLE sync_queue (
    id          BIGSERIAL,
    sf_id       TEXT        NOT NULL,
    status      TEXT        NOT NULL DEFAULT 'pending',
    payload     JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE sync_queue_2026_09
    PARTITION OF sync_queue
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');

CREATE TABLE sync_queue_2026_10
    PARTITION OF sync_queue
    FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
```

**Common mistake:** Choosing the wrong shard key. Sharding on a low-cardinality field (e.g., `status` with three values) creates hotspots where one shard handles almost all traffic. The shard key should have high cardinality and distribute writes evenly.

---

## 7. Idempotency Design

An operation is idempotent if executing it multiple times produces the same result as executing it once. This is non-negotiable in distributed systems: networks fail, timeouts occur, and retries are mandatory. Without idempotency, a retry doubles data or causes conflicting state.

The pattern is: the caller assigns a stable `idempotency_key` (usually a UUID derived from the triggering event), and the server records it. If the same key arrives again, the server returns the cached result without re-executing.

```sql
CREATE TABLE idempotency_keys (
    key         TEXT PRIMARY KEY,
    status      TEXT NOT NULL,           -- 'in_flight', 'complete', 'failed'
    response    JSONB,
    created_at  TIMESTAMPTZ DEFAULT now(),
    expires_at  TIMESTAMPTZ DEFAULT now() + INTERVAL '24 hours'
);
CREATE INDEX idx_idempotency_expires ON idempotency_keys (expires_at);
```

```python
# app/api/middleware/idempotency.py
from fastapi import Request, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

async def check_idempotency(
    request: Request,
    db: AsyncSession,
) -> Optional[dict]:
    key = request.headers.get("Idempotency-Key")
    if not key:
        return None  # no key provided — proceed normally

    result = await db.execute(
        text("SELECT status, response FROM idempotency_keys WHERE key = :k"),
        {"k": key}
    )
    row = result.mappings().one_or_none()

    if row is None:
        # First time — record as in-flight
        await db.execute(
            text("INSERT INTO idempotency_keys (key, status) VALUES (:k, 'in_flight')"),
            {"k": key}
        )
        await db.commit()
        return None  # caller proceeds with execution

    if row["status"] == "in_flight":
        raise HTTPException(status_code=409, detail="Request in progress")

    if row["status"] == "complete":
        return row["response"]  # return cached result, skip re-execution

    raise HTTPException(status_code=500, detail="Previous attempt failed; use a new key")
```

**AESF connection:** Salesforce Apex triggers can fire multiple times for the same record change (e.g., during bulk DML or re-runs). The Epic BDE API is not idempotent on its own — sending `PUT /clients` twice with the same payload can cause duplicate records in some versions. The middleware must deduplicate using the Salesforce record ID + last-modified timestamp as the idempotency key before forwarding to Epic.

```python
def make_sync_key(sf_id: str, last_modified: datetime) -> str:
    """Stable idempotency key for a given SF record state."""
    return f"sync:{sf_id}:{last_modified.isoformat()}"
```

**Common mistake:** Using a timestamp alone as the idempotency key. If two updates arrive within the same millisecond (common during bulk Salesforce operations), the key collides, and the second update is silently dropped.

---

## 8. Rate Limiting Algorithms

Rate limiting protects both your own service (from abuse or accidental hammering) and upstream APIs (Epic imposes rate limits; exceeding them causes 429 errors and potential account suspension).

### Token Bucket

A bucket holds up to `capacity` tokens. Tokens refill at a steady rate. Each request consumes one token. If the bucket is empty, the request is rejected or queued.

- Allows controlled bursting (up to `capacity` at once).
- Good for client-side rate limiting against an upstream API.

```python
import time
import asyncio
from dataclasses import dataclass, field

@dataclass
class TokenBucket:
    capacity: int       # max tokens (burst size)
    refill_rate: float  # tokens per second
    _tokens: float = field(init=False)
    _last_refill: float = field(init=False)

    def __post_init__(self):
        self._tokens = float(self.capacity)
        self._last_refill = time.monotonic()

    def consume(self, tokens: int = 1) -> bool:
        now = time.monotonic()
        elapsed = now - self._last_refill
        self._tokens = min(self.capacity, self._tokens + elapsed * self.refill_rate)
        self._last_refill = now
        if self._tokens >= tokens:
            self._tokens -= tokens
            return True
        return False

# Usage: limit Epic API calls to 10/s with burst of 20
epic_limiter = TokenBucket(capacity=20, refill_rate=10)

async def call_epic_api(endpoint: str, payload: dict):
    while not epic_limiter.consume():
        await asyncio.sleep(0.05)  # back off briefly
    return await http_client.put(endpoint, json=payload)
```

### Leaky Bucket

Requests enter a queue (the "bucket") and exit at a fixed rate. Excess requests that overflow the bucket are rejected. Unlike token bucket, there is no burst — output is perfectly smooth.

- Good for server-side rate limiting of incoming requests.
- Simpler to reason about for compliance guarantees.

```
Incoming requests (variable rate)
       ↓ ↓ ↓ ↓↓↓ ↓
   ┌──────────────┐
   │  Queue       │  ← overflow → 429 Too Many Requests
   │  (leaky      │
   │   bucket)    │
   └──────┬───────┘
          ↓ (fixed rate, e.g. 10 req/s)
    Processed requests
```

### Sliding Window Counter

A hybrid that tracks request counts in a rolling time window. More accurate than fixed window (which can allow 2x the limit at window boundaries) and less complex than true sliding window log.

| Algorithm | Burst | Accuracy | Complexity |
|---|---|---|---|
| Fixed window counter | Yes (2x at boundary) | Low | Very low |
| Token bucket | Controlled | High | Low |
| Leaky bucket | No | High | Low |
| Sliding window | Controlled | High | Medium |

**AESF connection:** The Epic BDE API has documented rate limits per client. The middleware worker that pulls jobs from the sync queue and calls Epic should use a token bucket per Epic endpoint family. If the bucket is empty, the worker should re-enqueue the job with a short delay rather than blocking, keeping other workers free.

```python
# Distributed rate limiter using Redis (required for multi-replica)
import redis.asyncio as redis

async def redis_token_bucket_consume(
    r: redis.Redis,
    key: str,
    capacity: int,
    refill_rate: float,
) -> bool:
    """Atomic token bucket check via Lua script — safe across replicas."""
    lua_script = """
    local tokens = tonumber(redis.call('GET', KEYS[1]) or ARGV[1])
    local now = tonumber(ARGV[2])
    local last = tonumber(redis.call('GET', KEYS[2]) or now)
    local elapsed = now - last
    tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[3]))
    redis.call('SET', KEYS[2], now)
    if tokens >= 1 then
        redis.call('SET', KEYS[1], tokens - 1)
        return 1
    else
        redis.call('SET', KEYS[1], tokens)
        return 0
    end
    """
    result = await r.eval(lua_script, 2, f"{key}:tokens", f"{key}:last",
                          capacity, time.time(), refill_rate)
    return bool(result)
```

**Common mistake:** Implementing rate limiting in-process (module-level) when running multiple replicas. Each replica maintains its own counter, so the effective limit is `per_replica_limit × replica_count`. The limiter must be backed by Redis (or a similar shared store) to be correct across replicas.

---

## 9. Analyzing the AESF Middleware Architecture

Applying the patterns above to the actual middleware:

```
Salesforce Apex Trigger
        │
        │  POST /sync/account  (webhook)
        ▼
┌───────────────────────────────────────────────┐
│  GKE Ingress (Nginx, round-robin)             │
└──────────┬────────────────────────────────────┘
           │  distributes across replicas
     ┌─────▼──────┐  ┌────────────┐  ┌────────────┐
     │ Middleware │  │ Middleware │  │ Middleware │
     │  Pod 0     │  │  Pod 1     │  │  Pod 2     │
     └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
           │               │               │
           └───────────────┼───────────────┘
                           │  all write to shared queue
                  ┌────────▼────────┐
                  │  PostgreSQL     │
                  │  Primary        │
                  │  sync_queue     │
                  └────────┬────────┘
                           │  WAL stream
                  ┌────────▼────────┐
                  │  Read Replica   │
                  │  (admin panel,  │
                  │   status views) │
                  └─────────────────┘
                           │
              Workers poll sync_queue
                           │
              ┌────────────▼──────────────┐
              │  Token Bucket             │
              │  (Epic API rate limiter)  │
              └────────────┬──────────────┘
                           │
              ┌────────────▼──────────────┐
              │  Epic BDE Backend         │
              │  PUT /clients             │
              └───────────────────────────┘
```

**Current risks to evaluate:**
1. Are any Pydantic models holding mutable state at module level?
2. Is the sync queue polling loop distributed correctly? If only one replica polls, it is a single point of failure.
3. Are idempotency keys enforced before forwarding to Epic?
4. Is the Redis-backed rate limiter in place, or is each replica independently throttling?

A useful exercise: open `app/core/config.py` and `app/api/` and identify every place a module-level mutable variable exists. Each one is a horizontal scaling hazard.

**Common mistake:** Scaling the web tier (adding replicas) while the actual bottleneck is the sync worker. If only one pod runs the background worker loop, replicas help only with incoming webhook throughput — the queue still drains at the same rate. Workers should be a separate `Deployment` resource, scaled independently.

---

## 10. Key Concepts Summary

```
Scalability Patterns
├── Compute tier
│   ├── Vertical scaling (bigger machine) — simple, limited
│   └── Horizontal scaling (more replicas) — requires stateless design
│       └── Load balancing strategies
│           ├── Round-robin (default, uniform cost)
│           ├── Least connections (variable cost)
│           └── IP hash (avoid — creates hotspots)
├── Application design
│   ├── Stateless services (no in-process mutable state)
│   ├── CQRS (separate command/query models)
│   └── Idempotency (safe retries, deduplicate on key)
├── Database tier
│   ├── Read replicas (write → primary, read → replica)
│   ├── Table partitioning (single-server, low complexity)
│   └── Sharding (multi-server, high complexity, last resort)
└── Flow control
    ├── Token bucket (allows burst, good for client-side)
    ├── Leaky bucket (smooth output, good for server-side)
    └── Sliding window counter (accurate, medium complexity)
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the fundamental requirement that a service must satisfy before it can be safely scaled horizontally?

**2.** A FastAPI application caches a Salesforce OAuth token in a module-level variable on startup. The deployment is scaled from 1 to 4 replicas. What failure mode will occur?

**3.** Your GKE `Service` uses the default `iptables` round-robin load balancing, but one replica consistently receives 60% of traffic. What is the most likely cause?

**4.** Explain replication lag in the context of PostgreSQL read replicas and describe a concrete scenario where it causes a correctness bug in the AESF middleware.

**5.** What is the difference between CQRS and simply having separate read/write database connections?

**6.** A sync worker calls `PUT /clients` on the Epic API. The HTTP call times out after 30 seconds with no response. The worker retries. What must be true of the PUT handler for this retry to be safe?

**7.** Describe the token bucket algorithm. What distinguishes it from the leaky bucket algorithm?

**8.** You implement a token bucket rate limiter in Python at the module level in a FastAPI app. The app has 3 replicas. The Epic API allows 30 requests/second. What effective limit does your module-level limiter enforce?

**9.** What is a shard key, and what property must it have to avoid hotspots?

**10.** When is database table partitioning a better choice than multi-server sharding for the AESF middleware?

**11.** You have a sync queue table with 50 million rows. `SELECT * FROM sync_queue WHERE status = 'pending'` is slow. What is the most direct fix, and what risk comes with it on a high-write table?

**12.** A Salesforce bulk DML operation triggers 500 account update webhooks in 200 milliseconds. Your idempotency key is `f"{sf_id}:{datetime.now().isoformat()}"`. Why might this fail to deduplicate?

**13.** What HTTP status code should a server return when it receives a request with an `Idempotency-Key` that is currently being processed by another worker?

**14.** Describe the CQRS pattern and give one concrete example of where it applies in the AESF middleware.

**15.** You add a read replica to PostgreSQL and route all `SELECT` queries to it. You notice that the admin panel sometimes shows a sync job as "pending" even after it has been completed. What is happening?

**16.** What is the purpose of the `pool_size` and `max_overflow` parameters in SQLAlchemy's `create_async_engine`? What happens if `max_overflow` is set too high when running 10 replicas?

**17.** Explain the "read-your-own-writes" consistency problem and describe one strategy to solve it in a read replica setup.

**18.** A new engineer suggests sharding the sync queue by `status` column to improve performance. Evaluate this suggestion.

**19.** The middleware sync worker runs as a Deployment with 3 replicas. Each replica polls `sync_queue WHERE status = 'pending' LIMIT 10`. What race condition exists, and how do you fix it?

**20.** You are asked to design the rate limiting for the AESF middleware's outbound Epic API calls in a way that is correct across all replicas and survives replica restarts. Describe the architecture.

---

### Answers

??? note "Reveal Answers"

    **1.** The service must be stateless — it must hold no mutable state in process memory that cannot be reproduced from a shared external store. Any local cache, session token, in-process queue, or temporary file that is specific to one replica will be invisible to other replicas, causing incorrect behavior when load is distributed. Stateless services can be started, stopped, and scaled arbitrarily because every instance is functionally identical.

    **2.** Only the replica that ran the startup event will have a valid Salesforce session token in its module-level variable. The other three replicas will have `None` (or a stale value). Requests routed to those replicas will fail when they try to use the token, producing 401 or 500 errors for approximately 75% of traffic. The fix is to store the token in Redis or re-acquire it per request using a dependency injection pattern.

    **3.** The most likely cause is connection keep-alive. If some clients maintain long-lived HTTP connections to specific replicas, those connections are not redistributed by round-robin — new connections are distributed evenly, but established ones stick. This is especially common with Salesforce outbound webhooks if they reuse connections. The fix is to enable connection draining and rotation, or to switch to least-connections balancing.

    **4.** Replication lag is the delay between a write being committed on the primary and that write becoming visible on the read replica. In the AESF middleware, a concrete bug: a sync job is marked `status = 'complete'` on the primary, and then an admin panel request immediately reads from the replica. If the replica has not yet received the WAL record for that update, it returns `status = 'pending'`. This is stale data, not an error, but it misleads operators. The fix is to route the read to the primary for the short window after a write, or to accept that the admin panel is eventually consistent.

    **5.** Separate read/write connections is an infrastructure concern — two connection pools pointing to different database endpoints. CQRS is an application architecture concern — two entirely separate domain models, one optimized for write validation and business logic (commands), one optimized for read presentation and query performance (queries). CQRS can be implemented with a single database; it does not require read replicas. The value is that the read model can be denormalized or pre-aggregated independently of the write model without either constraining the other.

    **6.** The PUT handler on the Epic side must be idempotent: calling it twice with the same payload must produce the same result as calling it once, with no duplicate records created. In practice this means the handler must use the client's stable identifier (e.g., Salesforce ID or Epic client ID) as a natural key, and an upsert or conditional create must be used rather than an unconditional insert. The middleware must also send a stable idempotency key header if the Epic API supports it.

    **7.** The token bucket maintains a pool of tokens that refills at a constant rate up to a maximum capacity. Each request consumes one token; if none remain, the request is rejected or queued. The key property is that it allows bursting — if the bucket is full, a spike of requests up to the capacity can be served immediately before the bucket is drained. The leaky bucket, by contrast, queues incoming requests and processes them at a fixed output rate with no burst. The token bucket is better for upstream API calls where you want to use available quota fully; the leaky bucket is better for downstream request acceptance where you need a smooth, predictable output rate.

    **8.** Each replica maintains its own independent token bucket with capacity for 30 req/s. Since there are 3 replicas and the load balancer distributes traffic across them, each replica may send up to 30 req/s to Epic, for a combined total of up to 90 req/s — three times the allowed limit. Epic will respond with 429 errors. The rate limiter must be backed by a shared atomic store (Redis) and use a distributed algorithm (e.g., a Lua script for atomic check-and-decrement) so that the total across all replicas never exceeds 30 req/s.

    **9.** A shard key is the field used to determine which shard a given row belongs to, typically by hashing or range-partitioning the value. To avoid hotspots, the shard key must have high cardinality (many distinct values) and its distribution across rows must be approximately uniform. Fields like `status` (three values) or `created_at` (clustered in time during business hours) make poor shard keys. Fields like `account_id` (UUID or high-cardinality integer) are good choices because rows distribute evenly.

    **10.** Table partitioning is almost always better for AESF's scale because it operates on a single PostgreSQL instance, requires no application-level routing, supports standard SQL joins across partitions, and can be added to an existing table with minimal disruption. Multi-server sharding is only warranted when a single PostgreSQL instance — even a very large one — cannot handle the write throughput or storage volume, typically at hundreds of millions of rows per day or terabytes of data. The AESF sync queue is high frequency but not at a scale that a well-tuned single Postgres instance with partitioning and connection pooling (PgBouncer) cannot handle.

    **11.** The most direct fix is to add an index: `CREATE INDEX CONCURRENTLY idx_sync_queue_pending ON sync_queue (id) WHERE status = 'pending'`. This partial index covers only pending rows, keeping it small and fast even as the table grows. The risk on a high-write table is index bloat and write amplification — every INSERT and UPDATE to `sync_queue` must also update the index, which adds latency under high write load. Monitor index bloat (`pg_stat_user_indexes`) and consider `REINDEX CONCURRENTLY` periodically.

    **12.** `datetime.now()` returns a timestamp with microsecond precision. If two Salesforce webhook calls for the same `sf_id` arrive within the same microsecond — entirely possible during a bulk operation that fires hundreds of webhooks in a tight loop — the timestamp component is identical and the key collides, causing the second event to be treated as a duplicate and silently dropped. The fix is to derive the key from the Salesforce record's `LastModifiedDate` field plus the `sf_id`, which is stable and event-sourced: the same record state always produces the same key regardless of when the webhook arrives.

    **13.** The server should return HTTP `409 Conflict`. This signals to the caller that the request is valid but cannot be processed at this moment due to a conflict with the current state — specifically, an in-flight operation with the same idempotency key. The caller should wait and retry rather than assuming the operation failed. `429 Too Many Requests` would be incorrect here because this is not a rate limit violation; it is a concurrency control signal.

    **14.** CQRS separates the model that handles writes (commands) from the model that handles reads (queries), allowing each to be optimized independently. In the AESF middleware, a concrete example: the command side accepts POST webhooks from Salesforce, validates the payload, enforces business rules (e.g., "do not sync if Epic client ID is missing"), and writes to the `sync_queue` table. The query side serves the admin panel's sync status dashboard, reading from a pre-aggregated view (`sync_status_view`) that joins queue entries with outcome records and formats them for display. Changes to the write model (e.g., splitting the payload into normalized columns) do not require changes to the read model, and vice versa.

    **15.** This is the replication lag problem. The worker updated the sync job status to `complete` on the primary, but the WAL record for that change has not yet been applied to the read replica when the admin panel query runs. The replica returns the old value. This is expected behavior for an asynchronous replica; it is eventually consistent, not strongly consistent. A practical fix is to route admin panel reads that display recently-modified records (within the last 5 seconds, for example) to the primary, and route historical and aggregated reads to the replica.

    **16.** `pool_size` is the number of persistent database connections maintained in the pool per engine instance. `max_overflow` is how many additional connections can be created beyond `pool_size` when the pool is exhausted, before requests start blocking. The risk with a high `max_overflow` at scale: 10 replicas × (`pool_size=10` + `max_overflow=20`) = 300 possible concurrent connections to PostgreSQL. PostgreSQL's default `max_connections` is 100. Exceeding it causes connection refusal. The fix is to use PgBouncer in transaction-pooling mode in front of PostgreSQL, which multiplexes thousands of client connections onto a small number of true PostgreSQL connections.

    **17.** The "read-your-own-writes" problem occurs when a client writes data to the primary and then immediately reads it back from a replica that has not yet received the replication update, so the read returns stale data — the client's own write is invisible. This is confusing in user-facing flows. One strategy is session stickiness: after a write, route reads for the same user/session to the primary for a short TTL (e.g., 5 seconds), then switch back to the replica. Another strategy is to track the primary's WAL position at the time of the write and wait for the replica to advance past it before serving the read from the replica (supported by `pg_last_wal_replay_lsn()`).

    **18.** This is a poor suggestion. `status` has low cardinality — typically three to five values (`pending`, `in_progress`, `complete`, `failed`). Sharding on a low-cardinality field creates severe hotspots: the `pending` shard receives nearly all inserts (every new job starts as pending) and the `complete` shard receives most updates (most jobs eventually succeed). Two of the four shards would be severely overloaded. The correct solution for a large sync queue table is range partitioning by `created_at` (time-series data ages naturally, old partitions can be archived) combined with a partial index on pending rows.

    **19.** The race condition is a classic queue worker double-take: two replicas execute `SELECT ... WHERE status = 'pending' LIMIT 10` simultaneously and both read overlapping sets of rows. Both then attempt to process the same jobs, potentially sending duplicate requests to the Epic API. The fix is pessimistic locking with `SELECT ... FOR UPDATE SKIP LOCKED`. `SKIP LOCKED` causes a worker to skip rows that another worker has already locked, ensuring each job is claimed by exactly one worker: `SELECT id FROM sync_queue WHERE status = 'pending' ORDER BY created_at LIMIT 10 FOR UPDATE SKIP LOCKED`.

    **20.** The correct architecture uses Redis as the shared atomic counter. Each replica, when it is about to call an Epic endpoint, checks a Redis-backed token bucket using a Lua script that atomically reads the current token count, computes the refill based on elapsed time, and either decrements the count and returns "allowed" or returns "denied" — all in one atomic operation, safe across all replicas. Token state persists in Redis, so a replica restart does not reset the bucket. The TTL on the Redis key should be set to slightly longer than the full-refill time (e.g., `capacity / refill_rate + 1` seconds) so that an idle period naturally expires the key and it reinitializes on next use without manual cleanup.

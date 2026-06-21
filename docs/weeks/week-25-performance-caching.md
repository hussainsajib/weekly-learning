# Week 25 — Performance Engineering & Caching

**Week of:** November 23, 2026
**Estimated study time:** ~2 hours
**Tags:** `performance` `caching` `redis` `profiling`

---

## Overview

Performance engineering is the discipline of making systems measurably faster, more resource-efficient, and more predictable under load — not merely *fast enough* by intuition. For a senior-to-staff engineer, the distinction is critical: you are expected to reason about performance from first principles, instrument real code, and propose architectural decisions (e.g., which cache pattern to apply, how to bound queue growth) rather than just applying folk wisdom. This week consolidates two tightly related areas — profiling and caching — because the correct sequence in practice is always: measure first, cache or optimize second.

Latency and throughput are often confused or conflated. Latency is the elapsed time for a single operation (one API call, one database round-trip). Throughput is the number of operations completed per unit of time. They interact but are not inverses: you can have high throughput with high per-request latency (large batches), or low latency with low throughput (serial single-item processing). In the AESF middleware, the Epic API calls for policy type and structure combination lookups add latency to every inbound Salesforce webhook, while the sync queue throughput determines how quickly the backlog drains. Fixing latency in the hot path (via caching) and fixing throughput in the queue (via concurrency tuning and batching) are separate levers.

Python profiling has a layered ecosystem. `cProfile` is the standard-library deterministic profiler — low overhead, call-count accurate, good for identifying hot functions in unit or integration tests. `line_profiler` narrows to line-level granularity inside a specific function, which is essential once `cProfile` identifies a suspect. `py-spy` is a sampling profiler that attaches to a running process without modification, making it the right tool for production diagnosis or for profiling a long-running Gunicorn/Uvicorn worker. Each tool has a distinct role, and knowing when to reach for each is a staff-level skill.

Caching patterns (cache-aside, write-through, write-behind) differ in who owns consistency and when writes propagate. Redis data structures go far beyond simple key/value: sorted sets, streams, and Lua scripts underpin many production patterns. Cache stampede — the thundering-herd event where a popular key expires and hundreds of concurrent requests slam the origin simultaneously — is especially dangerous during reconnect windows, exactly the scenario AESF faces when the Epic server becomes unavailable and then comes back online with a backlog of waiters.

---

## 1. Latency vs. Throughput — Getting the Mental Model Right

A useful mental model: imagine a highway. **Latency** is the time one car takes to travel from on-ramp to off-ramp. **Throughput** is the number of cars that pass the off-ramp per hour. Widening the highway (more lanes / more workers) increases throughput but does not reduce the distance a single car must travel. Reducing speed limits (e.g., synchronous I/O serializing requests) harms both. Caching shortens the route for frequently travelled trips.

For API services, the key latency decomposition is:

| Component | Typical share in AESF middleware | Optimization lever |
|---|---|---|
| Network RTT (client → service) | 1–5 ms (GKE internal) | Topology, keep-alive |
| Application logic (Python) | 2–20 ms | Profiling, async |
| Database query (PostgreSQL) | 5–50 ms | Indexes, query rewrite |
| Upstream API call (Epic) | 80–400 ms | **Caching** |
| Serialization / validation | 1–10 ms | Pydantic v2, orjson |

Epic API calls dominate. This is why caching policy type lookups and structure combinations is the highest-ROI optimization available.

**Little's Law** formalizes the throughput/latency/concurrency relationship:

```
L = λ × W
```

Where `L` is the average number of requests in the system, `λ` is the arrival rate (throughput), and `W` is the average time in system (latency). If the Epic reconnect event causes `W` to spike from 200 ms to 8 s, and arrival rate `λ` stays constant, the queue depth `L` grows 40×. This is why cache-warming on reconnect is an architectural concern, not just a performance nicety.

> **Common mistake:** Optimizing average latency while ignoring p99. A cache hit rate of 95% looks great, but the 5% misses that go to Epic at 400 ms will dominate your p99 and SLO breach events. Always measure tail latency.

---

## 2. Profiling Python: cProfile

`cProfile` is a deterministic profiler built into the standard library. Every function call is instrumented, so it has measurable overhead (~10–30% CPU), but it gives exact call counts and cumulative times.

```python
# profile_sync.py — wrap the sync worker's main processing function
import cProfile
import pstats
import io
from app.workers.sync_worker import process_sync_batch

def run_profile():
    pr = cProfile.Profile()
    pr.enable()

    # Run one batch of 50 sync items (representative workload)
    process_sync_batch(batch_size=50)

    pr.disable()

    stream = io.StringIO()
    stats = pstats.Stats(pr, stream=stream)
    stats.sort_stats(pstats.SortKey.CUMULATIVE)
    stats.print_stats(30)  # top 30 functions
    print(stream.getvalue())

if __name__ == "__main__":
    run_profile()
```

Read the output columns carefully:

| Column | Meaning |
|---|---|
| `ncalls` | Number of calls |
| `tottime` | Time spent in this function *excluding* callees |
| `cumtime` | Total time including callees — use this to find the expensive path |
| `percall` | cumtime / ncalls |

In AESF sync profiling, you will typically find that `httpx.Client.send` or `requests.Session.send` accounts for 70%+ of cumtime. That confirms the Epic API call is the bottleneck and caching is the right solution. If instead you see SQLAlchemy `execute` at the top, the bottleneck is the PostgreSQL queue table, which calls for index tuning or batch upserts.

```python
# Save profile data for later inspection with snakeviz
pr.dump_stats("sync_worker.prof")
# Then: pip install snakeviz && snakeviz sync_worker.prof
```

> **Common mistake:** Running cProfile against a test with mocked HTTP calls. The mock hides the actual bottleneck. Always profile against a real (or realistic stub) upstream.

---

## 3. Profiling Python: line_profiler

Once `cProfile` identifies a hot function, `line_profiler` reveals *which lines* inside it are expensive. Install with `pip install line-profiler`.

```python
# Decorate the suspect function
from line_profiler import LineProfiler
from app.services.epic_client import EpicClient

def get_policy_types(client: EpicClient, account_id: str):
    # cProfile showed this function at 120ms average — diagnose it
    response = client.get(f"/clients/{account_id}/policies")      # line A
    policy_types = [p["policyType"] for p in response.json()]      # line B
    validated = [validate_policy_type(pt) for pt in policy_types]  # line C
    return validated

# Run the profiler
lp = LineProfiler()
lp.add_function(get_policy_types)
lp_wrapper = lp(get_policy_types)
lp_wrapper(client, "ACC-001")
lp.print_stats()
```

The output shows time per line. If line A (the HTTP call) accounts for 115 ms and lines B–C account for 5 ms, you have confirmed the bottleneck is network I/O, not Python computation. If line C (Pydantic validation) accounts for 80 ms on a large response, that points to a different fix (Pydantic v2 model, `model_validate` with `from_attributes=True`).

> **Common mistake:** Using `line_profiler` on a function that calls many sub-functions without profiling those sub-functions too. The line showing a function call will show its total time; you still need to drill into that callee separately.

---

## 4. Profiling Python: py-spy in Production

`py-spy` is a sampling profiler written in Rust. It attaches to an already-running Python process via PID and samples the call stack at configurable Hz without requiring code changes. This makes it production-safe.

```bash
# Install
pip install py-spy

# Attach to a running Uvicorn worker (PID found via: ps aux | grep uvicorn)
py-spy top --pid 12345

# Generate a flame graph (SVG) — most useful for async FastAPI workers
py-spy record -o sync_worker_flamegraph.svg --pid 12345 --duration 30

# On GKE: exec into the pod first
kubectl exec -it middleware-pod-xyz -n aesf -- bash
# Then run py-spy inside the container (requires --cap-add SYS_PTRACE or privileged mode)
py-spy top --pid $(pgrep -f uvicorn)
```

The flame graph SVG shows call stacks as horizontal bars. Wide bars are where time is spent. In an AESF middleware worker, a healthy flame graph under cache-hit conditions will show a wide bar for Redis `GET` (~0.5 ms) and a narrow bar for Epic HTTP calls. During a cache miss storm (post-reconnect), the Epic HTTP bars will dominate.

For GKE deployments, add `SYS_PTRACE` capability to the pod security context:

```yaml
# deployment-manifests/kubernetes/middleware/deployment.yaml (excerpt)
securityContext:
  capabilities:
    add:
      - SYS_PTRACE
```

> **Common mistake:** Running py-spy at 1000 Hz on a production pod. High sampling rates add measurable overhead. Use 100 Hz for production; 1000 Hz only in load-test environments.

---

## 5. Cache Patterns: Cache-Aside, Write-Through, Write-Behind

These three patterns define *who* is responsible for keeping the cache and the source of truth in sync.

### Cache-Aside (Lazy Loading)

The application checks the cache first. On a miss, it fetches from the origin, writes to the cache, and returns the result. The cache is populated only on demand.

```python
import redis.asyncio as aioredis
import json
from app.clients.epic_client import EpicClient

redis_client = aioredis.from_url("redis://memorystore:6379", decode_responses=True)
epic_client = EpicClient()

POLICY_TYPES_KEY = "aesf:ref:policy_types"
POLICY_TYPES_TTL = 3600  # 1 hour — reference data changes rarely

async def get_policy_types() -> list[dict]:
    """Cache-aside: check Redis, fall back to Epic API."""
    cached = await redis_client.get(POLICY_TYPES_KEY)
    if cached:
        return json.loads(cached)

    # Cache miss — fetch from Epic
    data = await epic_client.get_policy_types()
    await redis_client.setex(POLICY_TYPES_KEY, POLICY_TYPES_TTL, json.dumps(data))
    return data
```

**Best for:** Read-heavy reference data that rarely changes (AESF policy types, structure combinations, activity codes). Tolerates stale reads within the TTL window.

### Write-Through

Every write to the database also writes to the cache synchronously. Reads always hit the cache.

```python
async def update_structure_combination(combo_id: str, payload: dict) -> dict:
    """Write-through: update DB and cache atomically."""
    # Write to PostgreSQL
    updated = await db.update_structure_combination(combo_id, payload)

    # Immediately update cache
    cache_key = f"aesf:ref:structure_combo:{combo_id}"
    await redis_client.setex(cache_key, 3600, json.dumps(updated))
    return updated
```

**Best for:** Data that is both read-heavy and write-frequent enough that cache-aside would produce too many misses. Adds write latency (two writes per mutation).

### Write-Behind (Write-Back)

The application writes to the cache immediately and acknowledges the client. The cache asynchronously flushes to the database. Highest write throughput, but risk of data loss if the cache crashes before flush.

```python
import asyncio

async def enqueue_sync_status_update(item_id: str, status: str):
    """Write-behind: ack immediately, flush to DB asynchronously."""
    cache_key = f"aesf:sync:status:{item_id}"
    await redis_client.setex(cache_key, 300, status)
    # Push to a Redis list that a background worker drains to PostgreSQL
    await redis_client.lpush("aesf:sync:status_flush_queue", f"{item_id}:{status}")
```

**Best for:** High-frequency status updates or metrics where losing a few seconds of data is acceptable (e.g., sync item processing state that can be reconstructed from the queue if needed).

| Pattern | Read latency | Write latency | Consistency | Data loss risk |
|---|---|---|---|---|
| Cache-aside | Low (hit) / High (miss) | Origin only | Eventual (TTL) | None |
| Write-through | Low | 2× origin | Strong | None |
| Write-behind | Low | Cache only | Eventual (flush lag) | Yes (crash window) |

> **Common mistake:** Using write-behind for data that must survive a Redis restart without replication/AOF enabled. Always enable Redis persistence (AOF or RDB snapshot) or use GCP Memorystore with replicas before adopting write-behind.

---

## 6. Redis Data Structures and AESF Use Cases

Redis is not a key/value store — it is a data structure server. Choosing the right structure eliminates application-side complexity and reduces round-trips.

### Strings
Simple cache values. Use for serialized JSON blobs (policy type lists, structure combinations).

```python
await redis_client.setex("aesf:ref:policy_types", 3600, json.dumps(data))
```

### Hashes
Field-level access without deserializing the entire blob. Use for per-account configuration or sync state.

```python
# Store sync state fields individually — update one field without touching others
await redis_client.hset("aesf:sync:state:ACC-001", mapping={
    "last_synced_at": "2026-11-23T10:00:00Z",
    "status": "completed",
    "retry_count": "0",
})
status = await redis_client.hget("aesf:sync:state:ACC-001", "status")
```

### Sorted Sets
Ordered collections scored by a float. Use for priority queues, leaderboards, or rate limiting.

```python
import time

# Priority sync queue: lower score = higher priority; use timestamp as score
async def enqueue_sync_item(account_id: str, priority: int = 0):
    score = time.time() - (priority * 1000)  # higher priority = lower score
    await redis_client.zadd("aesf:sync:queue", {account_id: score})

async def dequeue_sync_item() -> str | None:
    # Atomically pop the lowest-score (highest-priority) item
    result = await redis_client.zpopmin("aesf:sync:queue", count=1)
    return result[0][0] if result else None
```

### Lists
Simple FIFO/LIFO queues. Use for the write-behind flush queue.

### Sets
Membership testing. Use for deduplication: "which accounts are already in the sync queue?"

```python
async def is_already_queued(account_id: str) -> bool:
    return await redis_client.sismember("aesf:sync:queued_accounts", account_id)
```

### Pub/Sub and Streams
Redis Streams (`XADD`/`XREAD`) provide durable, consumer-group-aware event delivery. Relevant if AESF moves toward an event-driven sync architecture (each Epic change event published to a stream, multiple worker groups consuming).

> **Common mistake:** Storing large Salesforce account blobs (10–50 KB) in Redis Strings and caching thousands of them. Redis is an in-memory store — every cached object costs RAM. Cache only reference data and hot lookup keys, not full account records.

---

## 7. Distributed vs. Local (In-Process) Cache

Local caches (Python `dict`, `functools.lru_cache`, `cachetools.TTLCache`) live inside one process. They are zero-latency but invisible to other pods.

```python
from cachetools import TTLCache
from functools import wraps

# In-process LRU+TTL cache — valid only for data that is identical across all pods
_policy_types_cache: TTLCache = TTLCache(maxsize=10, ttl=300)

def local_cached(cache: TTLCache):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            key = str(args) + str(kwargs)
            if key not in cache:
                cache[key] = await func(*args, **kwargs)
            return cache[key]
        return wrapper
    return decorator

@local_cached(_policy_types_cache)
async def get_policy_types_local() -> list[dict]:
    return await epic_client.get_policy_types()
```

| Dimension | Local cache | Distributed (Redis) |
|---|---|---|
| Latency | ~0 μs | ~0.5–2 ms |
| Consistency across pods | No — each pod has its own copy | Yes — single shared store |
| Survives pod restart | No | Yes (with persistence) |
| Cache invalidation | Per-pod; must invalidate all | Single invalidation |
| Memory limit | JVM/process heap | Dedicated Redis instance |

**AESF decision rule:** Use local cache only for immutable or near-immutable config (e.g., Epic server URL, feature flags fetched at startup). Use Redis for any data shared across middleware pods, especially sync state and reference data, where a cache invalidation from one pod must be visible to all others.

> **Common mistake:** Using a local TTLCache for session tokens or sync state in a multi-replica GKE deployment. Pod A's cache becomes stale while Pod B updates the state in PostgreSQL, leading to phantom "already synced" decisions.

---

## 8. Cache Stampede and How to Prevent It

Cache stampede (thundering herd) occurs when a popular cached key expires simultaneously for many concurrent requests. All of them find a cache miss and fire requests to the origin at the same instant, overloading it.

**AESF risk scenario:** The Epic server goes down for 5 minutes during a maintenance window. All policy type and structure combination cache entries (TTL = 1 hour) happen to expire while Epic is down — or were never populated because the outage started before the first request. When Epic comes back online, every waiting middleware pod fires a GET to Epic concurrently for the same reference data. Epic's rate limiter throttles them; some requests time out; the sync backlog explodes.

### Prevention Strategy 1: Probabilistic Early Expiration (PER / XFetch)

Randomly re-cache before the key actually expires. Keys with higher latency-to-recompute should expire earlier stochastically.

```python
import math
import random
import time

async def get_with_per(
    key: str,
    fetch_fn,
    ttl: int = 3600,
    beta: float = 1.0,
) -> dict:
    """
    XFetch / Probabilistic Early Revalidation.
    beta > 1.0 means more eager early revalidation (less stampede risk).
    """
    raw = await redis_client.get(f"{key}:data")
    expiry_raw = await redis_client.get(f"{key}:expiry")

    if raw and expiry_raw:
        expiry = float(expiry_raw)
        delta = time.time() - (expiry - ttl)  # time since value was computed
        remaining_ttl = expiry - time.time()

        # XFetch formula: recompute if -delta * beta * log(random()) > remaining_ttl
        if -delta * beta * math.log(random.random()) < remaining_ttl:
            return json.loads(raw)

    # Fetch fresh value
    data = await fetch_fn()
    now = time.time()
    expiry = now + ttl

    pipe = redis_client.pipeline()
    pipe.setex(f"{key}:data", ttl + 60, json.dumps(data))  # small grace buffer
    pipe.setex(f"{key}:expiry", ttl + 60, str(expiry))
    await pipe.execute()
    return data
```

### Prevention Strategy 2: Mutex / Distributed Lock

Only one requester recomputes the value; others wait or return stale data.

```python
from redis.asyncio.lock import Lock

async def get_policy_types_with_lock() -> list[dict]:
    cached = await redis_client.get(POLICY_TYPES_KEY)
    if cached:
        return json.loads(cached)

    # Acquire a distributed lock — only one pod recomputes
    lock = redis_client.lock("aesf:lock:policy_types_refresh", timeout=10)
    async with lock:
        # Double-check after acquiring lock (another pod may have populated it)
        cached = await redis_client.get(POLICY_TYPES_KEY)
        if cached:
            return json.loads(cached)

        data = await epic_client.get_policy_types()
        await redis_client.setex(POLICY_TYPES_KEY, POLICY_TYPES_TTL, json.dumps(data))
        return data
```

### Prevention Strategy 3: Cache Warming on Reconnect

Proactively warm the cache when Epic becomes available again, before the backlog of requests hits.

```python
from app.events import epic_reconnected_event

async def on_epic_reconnect():
    """Warm reference data cache immediately after Epic reconnect."""
    keys_to_warm = [
        ("aesf:ref:policy_types", epic_client.get_policy_types),
        ("aesf:ref:activity_codes", epic_client.get_activity_codes),
        ("aesf:ref:structure_combinations", epic_client.get_structure_combinations),
    ]
    for cache_key, fetch_fn in keys_to_warm:
        data = await fetch_fn()
        await redis_client.setex(cache_key, POLICY_TYPES_TTL, json.dumps(data))

epic_reconnected_event.subscribe(on_epic_reconnect)
```

> **Common mistake:** Setting all reference data keys to the same TTL. They will all expire at roughly the same time (especially if the service restarts and populates them all in a burst). Add jitter: `ttl = base_ttl + random.randint(0, 300)`.

---

## 9. HTTP Load Testing with Locust

Locust is a Python-native load testing framework. Tests are written as Python classes, making them composable, version-controlled, and easy to integrate into CI.

```python
# locustfile.py — load test AESF middleware endpoints
from locust import HttpUser, task, between, events
import random

ACCOUNT_IDS = [f"ACC-{i:04d}" for i in range(1, 201)]

class AESFMiddlewareUser(HttpUser):
    """Simulates Salesforce webhook traffic hitting middleware."""
    wait_time = between(0.1, 0.5)  # think time between requests

    @task(5)
    def get_policy_types(self):
        """High-frequency read — should be cache hit."""
        with self.client.get(
            "/api/v2/reference/policy-types",
            headers={"Authorization": "Bearer test-token"},
            catch_response=True,
            name="GET /reference/policy-types",
        ) as response:
            if response.status_code == 200:
                response.success()
            elif response.status_code == 429:
                response.failure("Rate limited")

    @task(3)
    def get_structure_combinations(self):
        """Reference data — should hit cache after first request."""
        self.client.get(
            "/api/v2/reference/structure-combinations",
            name="GET /reference/structure-combinations",
        )

    @task(2)
    def trigger_account_sync(self):
        """Write path — enqueues sync item."""
        account_id = random.choice(ACCOUNT_IDS)
        self.client.post(
            f"/api/v2/accounts/{account_id}/sync",
            json={"trigger": "salesforce_webhook", "fields_changed": ["Name", "BillingCity"]},
            name="POST /accounts/:id/sync",
        )

    @task(1)
    def get_sync_status(self):
        """Poll sync status for a random account."""
        account_id = random.choice(ACCOUNT_IDS)
        self.client.get(
            f"/api/v2/accounts/{account_id}/sync/status",
            name="GET /accounts/:id/sync/status",
        )


@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    print("Load test started — AESF middleware")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    print("Load test complete")
```

Run the load test:

```bash
# Headless mode — 50 concurrent users, ramp up over 10 seconds, run for 60 seconds
locust -f locustfile.py \
  --host=https://middleware-staging.aesf.internal \
  --headless \
  --users 50 \
  --spawn-rate 5 \
  --run-time 60s \
  --html load_test_report.html

# Interactive web UI (useful for exploratory testing)
locust -f locustfile.py --host=https://middleware-staging.aesf.internal
# Open http://localhost:8089
```

**Interpreting results for AESF:**

| Metric | Target | Red flag |
|---|---|---|
| `GET /reference/policy-types` p50 | < 5 ms (cache hit) | > 50 ms = cache miss rate too high |
| `GET /reference/policy-types` p99 | < 20 ms | > 200 ms = stampede occurring |
| `POST /accounts/:id/sync` p50 | < 50 ms | > 500 ms = queue table contention |
| Error rate | < 0.1% | > 1% = circuit breaker or Epic throttle |

> **Common mistake:** Running locust against production endpoints without rate limiting the test itself. In AESF, the sync POST creates real queue items. Always use a dedicated load-test environment or add a `X-Load-Test: true` header that the middleware uses to skip actual Epic calls.

---

## 10. Putting It Together: AESF Performance Playbook

A practical checklist for diagnosing and fixing a slow AESF middleware deployment:

**Step 1 — Measure tail latency first**
```bash
# Check Datadog p99 for the last hour
# Or use locust to baseline before any changes
locust -f locustfile.py --headless --users 10 --spawn-rate 2 --run-time 30s
```

**Step 2 — Identify hot functions with cProfile**
```bash
python -m cProfile -s cumtime -o baseline.prof app/workers/sync_worker.py
python -c "import pstats; p=pstats.Stats('baseline.prof'); p.sort_stats('cumtime'); p.print_stats(20)"
```

**Step 3 — Attach py-spy to a production pod if cProfile is inconclusive**
```bash
kubectl exec -it $(kubectl get pod -l app=aesf-middleware -o name | head -1) -- \
  py-spy record -o /tmp/flamegraph.svg --pid $(pgrep -f uvicorn) --duration 30
kubectl cp <pod>:/tmp/flamegraph.svg ./flamegraph.svg
```

**Step 4 — Cache Epic reference data in Redis with jitter**
```python
TTL_BASE = 3600
TTL_JITTER = random.randint(0, 300)
await redis_client.setex(key, TTL_BASE + TTL_JITTER, json.dumps(data))
```

**Step 5 — Add distributed lock on cache population**
Use the mutex pattern from Section 8 for any key that multiple pods might populate simultaneously.

**Step 6 — Warm cache on Epic reconnect**
Implement the `on_epic_reconnect` hook from Section 8. Register it with your circuit breaker's `on_close` callback.

**Step 7 — Validate with locust at target concurrency**
Re-run the load test and confirm p99 for reference endpoints is < 20 ms and error rate is < 0.1%.

> **Common mistake:** Treating performance work as done once the average looks good. Always confirm the fix with a load test at 2× expected concurrency, because cache stampede and queue contention only manifest under load.

---

## 11. Key Concepts Summary

```
Performance Engineering & Caching
├── Mental Models
│   ├── Latency  → time for one operation
│   ├── Throughput → operations per second
│   └── Little's Law: L = λ × W
│
├── Profiling Tools
│   ├── cProfile       → deterministic, call graph, dev/test
│   ├── line_profiler  → line-level, known-hot function
│   └── py-spy         → sampling, zero-code-change, production
│
├── Cache Patterns
│   ├── Cache-aside    → app checks cache, lazy populate
│   ├── Write-through  → write to cache + DB synchronously
│   └── Write-behind   → write to cache, flush DB async
│
├── Redis Data Structures
│   ├── String   → JSON blob (policy types, structure combos)
│   ├── Hash     → field-level sync state
│   ├── Sorted Set → priority sync queue
│   ├── List     → FIFO flush queue (write-behind)
│   └── Set      → deduplication (already-queued accounts)
│
├── Cache Stampede Prevention
│   ├── Probabilistic Early Revalidation (XFetch)
│   ├── Distributed Lock (redis-py Lock)
│   └── Cache Warming on reconnect
│
├── Local vs. Distributed Cache
│   ├── Local (TTLCache) → zero latency, per-pod, immutable config
│   └── Redis (Memorystore) → shared, consistent, survives pod restart
│
└── Load Testing
    └── locust → Python tasks, ramp-up, HTML report, CI-friendly
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the difference between latency and throughput? Give an example from AESF where optimizing one does not automatically improve the other.

**2.** What does Little's Law (`L = λ × W`) predict happens to the AESF sync queue depth if Epic API latency increases from 200 ms to 2 s while arrival rate stays constant?

**3.** You run `cProfile` on `process_sync_batch` and see that `requests.Session.send` has `cumtime = 18.4s` out of `20.1s` total. What does this tell you, and what is your next action?

**4.** What is the key difference between `tottime` and `cumtime` in cProfile output? Which should you use to find the most expensive call chain?

**5.** Why is `line_profiler` more useful than `cProfile` once you have identified a suspect function?

**6.** What is a sampling profiler, and why is `py-spy` safer to use in production than `cProfile`?

**7.** Explain the cache-aside pattern. In which scenario does it perform worst, and what is the AESF-specific example of this worst case?

**8.** What is the main trade-off of write-through caching compared to cache-aside?

**9.** Describe a scenario where write-behind caching could cause data loss in AESF. What Redis configuration mitigates this risk?

**10.** You have 5 GKE pods running AESF middleware. A sync state update is written by Pod A. Why would a local (in-process) cache on Pod B be a problem?

**11.** What Redis data structure would you use to implement a priority-ordered sync queue where higher-priority accounts are dequeued first? Name the relevant commands.

**12.** What is cache stampede and why is it a specific risk during Epic reconnect events in AESF?

**13.** Explain the XFetch (Probabilistic Early Revalidation) algorithm. How does the `beta` parameter affect stampede risk?

**14.** How does a distributed lock prevent cache stampede? Write pseudocode for the double-check-after-lock pattern.

**15.** Why should you add TTL jitter when populating multiple reference data keys in Redis? What problem does it prevent?

**16.** In a locust test, the `POST /accounts/:id/sync` endpoint shows p99 = 800 ms under 50 users. What are the two most likely causes in the AESF context, and how would you distinguish between them?

**17.** What is the `wait_time = between(0.1, 0.5)` parameter in Locust and what real-world behavior does it model?

**18.** Why is it dangerous to run a locust load test against AESF staging using the `POST /accounts/:id/sync` endpoint without precautions?

**19.** You notice that `GET /reference/policy-types` has p50 = 3 ms but p99 = 450 ms under load. What is the most likely explanation, and which stampede-prevention technique addresses it?

**20.** A colleague proposes caching full Salesforce Account objects (avg 40 KB each) for all 50,000 accounts in Redis. What concerns would you raise, and what alternative caching strategy would you recommend?

---

### Answers

??? note "Reveal Answers"

    **1.** Latency is the elapsed time for a single operation; throughput is the number of operations completed per second. In AESF, caching Epic API responses reduces the latency of a single reference lookup from ~300 ms to ~1 ms. However, if the sync worker is single-threaded and processes items serially, throughput (items/sec) is bounded by concurrency — reducing per-item latency helps, but adding async concurrency is the throughput lever. These are orthogonal improvements.

    **2.** Little's Law states `L = λ × W`. If `W` (average time in system, proportional to Epic latency) grows 10×, queue depth `L` grows 10× at the same arrival rate `λ`. If the queue has a fixed capacity or a downstream consumer with bounded concurrency, items begin to back up and eventually time out. This is why cache-warming on reconnect is critical: it prevents the 10× latency spike from translating into a 10× queue spike.

    **3.** This tells you that 91.5% of all processing time is spent in HTTP I/O calling the Epic API, not in Python computation. The bottleneck is not algorithm efficiency — it is network round-trips. The correct next action is to implement caching (cache-aside with Redis) for any Epic API call that returns stable reference data, so that repeated calls within the TTL window are served from Redis at ~1 ms instead of 300+ ms.

    **4.** `tottime` is the time spent executing only that function's own code, excluding time spent in any function it calls. `cumtime` is the total time including all callees. To find the most expensive call chain (the path where time is really being spent), sort by `cumtime` — a function may have negligible `tottime` itself but a very high `cumtime` because it calls expensive sub-functions. `tottime` is useful only when you want to confirm that a specific function's own logic (not its callees) is expensive.

    **5.** `cProfile` reports per-function timing but not which line within the function is slow. Once `cProfile` has identified a suspect function, `line_profiler` decorates it and measures each individual line's execution time and call count. This allows you to distinguish between, for example, a slow regex on one line, a slow Pydantic validation on another, and a fast HTTP call on a third — all within the same function.

    **6.** A sampling profiler periodically interrupts the process and records the current call stack, rather than instrumenting every function call. `py-spy` is written in Rust and reads the CPython stack via `/proc/<pid>/mem` without injecting any code into the process. This means overhead is proportional only to sampling frequency (default 100 Hz), not to call rate, making it safe to run against a live production Uvicorn worker. `cProfile`'s overhead grows with function call frequency, which can distort results and slow the service under production load.

    **7.** Cache-aside: the application checks the cache before calling the origin; on a miss it fetches from origin and populates the cache; subsequent reads hit the cache. It performs worst when the cache hit rate is low — specifically, when a large number of distinct keys are requested before any have been warmed, causing every request to be a cache miss and hit the origin. In AESF, the worst case is immediately after service startup or after a Redis flush, when all policy type and structure combination keys are cold and every concurrent Salesforce webhook fires an Epic API call simultaneously.

    **8.** Write-through guarantees that the cache always reflects the latest write (strong consistency), unlike cache-aside which may serve stale data until the TTL expires. The trade-off is write latency: every mutation must complete two writes (origin DB and cache) before acknowledging the client. This doubles write latency compared to cache-aside, which only writes to the origin on the write path. For AESF reference data that is updated infrequently, cache-aside with a short TTL is usually the better balance.

    **9.** In write-behind, updates are acknowledged to the client after writing only to Redis; the flush to PostgreSQL happens asynchronously. If the Redis instance crashes or the GKE pod running the flush worker is evicted before the flush completes, those writes are lost permanently. In AESF, if sync status updates are written behind and Redis loses data, the sync queue state in PostgreSQL becomes stale, leading to duplicate or missed syncs. The mitigation is to enable Redis AOF (Append-Only File) persistence with `appendfsync always` or to use GCP Memorystore with a replica, so writes survive instance restarts.

    **10.** Pod B's local cache holds its own in-memory copy of the sync state that was valid when Pod B last fetched it. When Pod A writes a state update to PostgreSQL (and possibly Redis), Pod B's local cache is unaware of this change and will continue returning its stale copy for any request routed to it. In a GKE deployment with round-robin load balancing, this means ~80% of subsequent requests (those routed to Pods B–E) will observe stale state. Distributed Redis eliminates this: all pods read from and write to the same store, so an update from any pod is immediately visible to all others within the same Redis read.

    **11.** A Redis Sorted Set (`ZSET`) is the correct structure. Each member is an account ID; the score is a numeric value representing priority (lower score = higher priority, so use a timestamp or a negative priority value). Key commands: `ZADD aesf:sync:queue <score> <account_id>` to enqueue, `ZPOPMIN aesf:sync:queue 1` to atomically dequeue the highest-priority (lowest-score) item, and `ZCARD aesf:sync:queue` to check queue depth. The atomic `ZPOPMIN` is important — it prevents two workers from dequeuing the same item.

    **12.** Cache stampede occurs when a popular cached key expires and many concurrent requests all detect a miss at the same instant, then simultaneously race to recompute the value from the origin. During an Epic reconnect, all middleware pods have been unable to populate their caches while Epic was down. The moment Epic becomes available, every queued request fires an Epic API call for the same reference data keys at the same time, potentially overwhelming Epic's rate limiter and causing cascading failures. This is amplified in AESF by the backlog: hundreds of sync items may have accumulated during the outage, all requiring policy type lookups.

    **13.** XFetch works by computing, before the key expires, a probabilistic re-fetch decision based on remaining TTL and the time it took to compute the value. The formula `−delta × beta × log(random())` produces a threshold: if the threshold exceeds the remaining TTL, the current request re-fetches early and refreshes the cache before expiry. A higher `beta` (e.g., 2.0) makes early revalidation more likely, reducing stampede risk at the cost of more frequent origin fetches. A lower `beta` (e.g., 0.5) delays revalidation closer to actual expiry, saving origin calls but increasing stampede risk.

    **14.** A distributed lock ensures only one process can recompute the cache value at a time. Pattern: (1) Check cache — if hit, return. (2) Acquire lock with a short timeout (e.g., 10 s). (3) After acquiring lock, check cache again (double-check) because another process may have populated it while waiting for the lock. (4) If still a miss, fetch from origin and populate cache. (5) Release lock. The double-check is essential: without it, all processes that were waiting for the lock will each re-fetch from origin sequentially after the first holder releases, defeating the purpose.

    **15.** When a service restarts, it populates all reference data keys simultaneously. If all keys have the same TTL (e.g., 3600 s), they will all expire at the same time 1 hour later, creating a synchronized expiry event where all keys become cold simultaneously. This is a self-inflicted stampede. Adding jitter (e.g., `TTL = 3600 + random(0, 300)`) staggers expiration times across a 5-minute window, ensuring that at any given time only a small fraction of keys expire together, spreading the recomputation load smoothly.

    **16.** The two most likely causes are (a) PostgreSQL queue table contention — the sync queue table is being written to and read from by many workers simultaneously, causing lock waits — or (b) the sync POST triggers a synchronous Epic API call in the request path (i.e., the handler is not fully asynchronous and awaits an Epic response before returning). To distinguish them: attach py-spy during the load test and look at the flame graph. If the wide bar is `asyncpg` or `sqlalchemy.execute`, the bottleneck is PostgreSQL. If the wide bar is `httpx.send` or `requests.Session.send`, the bottleneck is a synchronous Epic call in the request handler.

    **17.** `wait_time = between(0.1, 0.5)` configures each simulated user to pause for a random duration between 100 ms and 500 ms between consecutive requests. This models real-world user think time — the delay between a Salesforce user completing one action and triggering the next. Without think time, locust users would fire requests as fast as the server responds, producing unrealistically high concurrency. The `between` distribution also randomizes arrival patterns, preventing the artificial synchronized bursts that a fixed wait time would create.

    **18.** The `POST /accounts/:id/sync` endpoint creates real sync queue items in the PostgreSQL queue table. A locust test with 50 users and `task` weight 2 could generate thousands of queue items in minutes. These items will be picked up by the real ETL worker and sent to the Epic API, potentially creating duplicate records, firing Epic webhooks, or consuming Epic rate limit quota. The staging environment may also share a PostgreSQL database with the real QA Salesforce org. Always either add a load-test header that causes the handler to enqueue to a separate test queue or to no-op entirely, or coordinate with the team to ensure the ETL worker is paused during load tests.

    **19.** The p50 of 3 ms indicates that most requests are served from the Redis cache (cache hits). The p99 of 450 ms indicates that approximately 1% of requests are cache misses that fall through to the Epic API (~400 ms). This is a classic cache stampede signature at the tail: the 1% misses are likely requests that arrive at the moment a key expires, and if they are concurrent, they all hit Epic simultaneously. Probabilistic Early Revalidation (XFetch) addresses this most directly by re-fetching the key before expiry, preventing the synchronized miss event. A distributed lock is a secondary option but adds lock-wait latency to misses.

    **20.** Caching 50,000 accounts at 40 KB each would consume 2 GB of Redis memory — a significant and likely unwarranted cost on GCP Memorystore. More importantly, account data changes frequently (Salesforce updates, Epic sync results), so cached copies would go stale quickly, requiring aggressive TTLs or complex invalidation logic. The alternative is to cache only the data that is truly read-heavy and rarely changes: reference data (policy types, structure combinations, activity codes, ~50–200 KB total), and per-account hot state (sync status, last-synced timestamp) using Hashes (~100–500 bytes per account). Full account objects should be served directly from PostgreSQL with appropriate indexes, not from Redis.

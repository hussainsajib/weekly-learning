# Week 13 — Advanced Python — Concurrency

**Week of:** August 31, 2026
**Estimated study time:** ~2 hours
**Tags:** `python` `async` `concurrency`

---

## Overview

Python's concurrency model is often misunderstood because the language offers three distinct mechanisms — threading, multiprocessing, and asyncio — each targeting a different class of problem. Threading and multiprocessing have been in the standard library since the early 2000s, while asyncio arrived in Python 3.4 and has matured significantly through 3.13. Understanding which tool to reach for, and why, is a critical skill for any senior engineer working on I/O-bound services like the AESF FastAPI middleware.

At the heart of asyncio lies the event loop: a single-threaded scheduler that multiplexes I/O operations without blocking the calling thread. When you write `await some_coroutine()`, you are not blocking — you are yielding control back to the event loop, which can then service other waiting tasks. This cooperative multitasking model is extremely efficient for services like AESF middleware that spend most of their time waiting on external systems: the Epic EHR API, PostgreSQL, and Salesforce. A single event loop thread can handle hundreds of concurrent HTTP connections with far less overhead than an equivalent thread pool.

The danger in this model is that it is cooperative, not preemptive. If any coroutine performs a blocking operation — a synchronous database query, a CPU-heavy computation, even a blocking `time.sleep()` — the event loop freezes entirely. Every other in-flight request queues behind that one blocking call. This is exactly the failure mode seen in AESF middleware when synchronous SQLAlchemy sessions leak into async request handlers. Detecting and eliminating these blocking calls is one of the highest-leverage reliability improvements you can make to the service.

This week builds from the ground up: event loop internals, the coroutine/task/awaitable hierarchy, the relationship between asyncio and threads/processes, and practical FastAPI patterns for writing genuinely non-blocking endpoints. Every section includes a concrete AESF-relevant example because the gap between "understanding asyncio" and "correctly applying it in production" is where most bugs live.

---

## 1. The Event Loop — Internals and Lifecycle

The event loop is a `selectors`-based I/O polling loop backed by the OS's most efficient mechanism: `epoll` on Linux, `kqueue` on macOS, `IOCP` on Windows. Each iteration of the loop does three things: (1) run all ready callbacks, (2) poll I/O selectors for completed operations, (3) schedule newly-ready callbacks. Coroutines become callbacks when they are wrapped in `Task` objects.

```python
import asyncio

# Python 3.13 — get the running loop from within a coroutine
async def inspect_loop() -> None:
    loop = asyncio.get_running_loop()
    print(f"Loop implementation: {type(loop).__name__}")
    print(f"Time: {loop.time():.4f}")  # monotonic clock, seconds since loop start

asyncio.run(inspect_loop())
```

`asyncio.run()` is the standard entry point since Python 3.7. It creates a new event loop, runs the given coroutine to completion, and then closes the loop — including cancelling all remaining tasks and running shutdown hooks. Never nest `asyncio.run()` calls; if you're already inside a running loop (e.g., inside a FastAPI endpoint), use `asyncio.create_task()` or `await` directly.

The loop maintains two queues: a ready queue (callbacks to run immediately on the next iteration) and a scheduled queue (callbacks to run after a delay, backed by a min-heap). `loop.call_soon()` adds to the ready queue; `loop.call_later()` adds to the scheduled queue. Every `await` expression ultimately resolves to one of these callbacks.

**Common mistake:** Calling `asyncio.get_event_loop()` instead of `asyncio.get_running_loop()`. In Python 3.10+ `get_event_loop()` emits a DeprecationWarning when there is no current loop, and in 3.12+ it will raise `RuntimeError` in some contexts. Always use `get_running_loop()` inside a coroutine, or `asyncio.run()` at the top level.

---

## 2. Coroutines, Tasks, and Awaitables

The term "awaitable" refers to any object that can appear after `await`. Python defines three kinds:

| Awaitable type | Created by | Scheduled by |
|----------------|------------|--------------|
| Coroutine | `async def` function call | Must be wrapped in a Task or directly awaited |
| Task | `asyncio.create_task()` | Scheduled immediately on the running loop |
| Future | `loop.create_future()` | Resolved externally (e.g., by a callback) |

A coroutine object is **lazy**: calling an `async def` function returns a coroutine object but executes nothing. Execution starts only when the coroutine is awaited or wrapped in a Task.

```python
import asyncio
import httpx

async def fetch_epic_client(client_id: str, http: httpx.AsyncClient) -> dict:
    """Fetch a single client from Epic EHR API."""
    response = await http.get(
        f"/clients/{client_id}",
        timeout=10.0,
    )
    response.raise_for_status()
    return response.json()

async def fetch_many_clients(client_ids: list[str]) -> list[dict]:
    """Fetch multiple Epic clients concurrently using Tasks."""
    async with httpx.AsyncClient(base_url="https://de21web/api/v1") as http:
        # create_task schedules all coroutines immediately — they run concurrently
        tasks = [
            asyncio.create_task(fetch_epic_client(cid, http), name=f"epic-{cid}")
            for cid in client_ids
        ]
        # gather collects results in order, propagates first exception by default
        results = await asyncio.gather(*tasks, return_exceptions=False)
    return results
```

`asyncio.gather()` vs `asyncio.TaskGroup` (Python 3.11+): `TaskGroup` is preferred in modern code because it has structured concurrency semantics — if any child task raises, all siblings are cancelled before the exception propagates. `gather()` leaves siblings running by default.

```python
async def fetch_many_clients_structured(client_ids: list[str]) -> list[dict]:
    results: list[dict] = []
    async with httpx.AsyncClient(base_url="https://de21web/api/v1") as http:
        async with asyncio.TaskGroup() as tg:
            tasks = [
                tg.create_task(fetch_epic_client(cid, http))
                for cid in client_ids
            ]
        # TaskGroup exits only when all tasks are done or one raised
        results = [t.result() for t in tasks]
    return results
```

**Common mistake:** Awaiting a coroutine sequentially when concurrent execution is needed. `result = await coro_a(); result2 = await coro_b()` runs A then B in serial. Use `create_task` or `gather` to run them concurrently. In AESF, fetching Epic client data and policy data for the same account in serial doubles latency unnecessarily.

---

## 3. Threading vs. Multiprocessing vs. asyncio

Choosing the right concurrency primitive depends on what your bottleneck is:

| Concern | Best primitive | Why |
|---------|---------------|-----|
| Network I/O (HTTP, DB) | asyncio | Single-threaded, no GIL contention, minimal overhead |
| Blocking I/O, legacy sync libs | `ThreadPoolExecutor` | Releases GIL during I/O, unblocks event loop |
| CPU-bound work (parsing, crypto) | `ProcessPoolExecutor` | Bypasses GIL entirely with separate processes |
| Mixed: async + CPU-bound | asyncio + `run_in_executor` | Dispatches CPU work to process pool without blocking loop |

The Global Interpreter Lock (GIL) is the key constraint: CPython only executes Python bytecode on one thread at a time. For I/O-bound work this doesn't matter because threads spend most of their time waiting on OS syscalls (which release the GIL). For CPU-bound work, threading provides no parallelism — you need multiprocessing.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import json

def parse_large_epic_payload(raw: bytes) -> dict:
    """CPU-bound: deserialize and validate a large JSON payload. Runs in process pool."""
    data = json.loads(raw)
    # ... heavy validation logic ...
    return data

async def handle_epic_webhook(raw_body: bytes) -> dict:
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor(max_workers=2) as pool:
        result = await loop.run_in_executor(pool, parse_large_epic_payload, raw_body)
    return result
```

**Common mistake:** Using `threading.Thread` directly in async code. You can start threads, but you cannot safely call coroutines from them without `asyncio.run_coroutine_threadsafe()`. If a thread attempts to call an `async def` function without awaiting it, the coroutine is silently dropped.

---

## 4. `concurrent.futures` — The Unified Interface

`concurrent.futures` provides `ThreadPoolExecutor` and `ProcessPoolExecutor` with an identical interface, making it easy to swap strategies. The `Executor.submit()` method returns a `Future` (not an asyncio Future — a `concurrent.futures.Future`). These integrate with asyncio via `loop.run_in_executor()`.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import psycopg2  # legacy sync driver — illustrative

# Module-level pool reused across requests
_thread_pool = ThreadPoolExecutor(max_workers=10, thread_name_prefix="sync-db")

def sync_query(query: str, params: tuple) -> list[dict]:
    """Blocking DB call — must not run on the event loop thread."""
    conn = psycopg2.connect("postgresql://localhost/aesf")
    cur = conn.cursor()
    cur.execute(query, params)
    return cur.fetchall()

async def async_wrapped_query(query: str, params: tuple) -> list[dict]:
    """Run a blocking query in the thread pool, freeing the event loop."""
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(
        _thread_pool,
        sync_query,
        query,
        params,
    )
```

Note: In AESF middleware you use SQLAlchemy async (`asyncpg` driver), so this pattern applies only when integrating legacy sync dependencies — for example, the Pentaho ETL monitoring sidecar or any sync Salesforce SDK calls. The key insight is that `run_in_executor` is the bridge between the synchronous and asynchronous worlds.

**Common mistake:** Creating a new `ThreadPoolExecutor` on every request. Thread pool creation is expensive. Define the pool at module level (or as a dependency) and reuse it. FastAPI's `lifespan` context is the right place to create and shut down shared pools.

---

## 5. Async Context Managers and Async Iterators

An async context manager implements `__aenter__` and `__aexit__` as coroutines. This is essential for resources that require async setup or teardown — database connection pools, HTTP client sessions, file handles on async filesystems.

```python
import asyncio
from contextlib import asynccontextmanager
from typing import AsyncGenerator
import httpx
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/aesf",
    pool_size=20,
    max_overflow=10,
)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

@asynccontextmanager
async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Async context manager for a scoped SQLAlchemy session."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

async def update_account(account_id: str, data: dict) -> None:
    async with get_db_session() as db:
        # db is an AsyncSession; all operations must be awaited
        result = await db.execute(
            select(Account).where(Account.sf_id == account_id)
        )
        account = result.scalar_one_or_none()
        if account:
            for key, val in data.items():
                setattr(account, key, val)
        # commit happens automatically on context manager exit
```

Async iterators (`__aiter__` / `__anext__`) allow streaming large result sets without loading them fully into memory:

```python
async def stream_pending_sync_records(db: AsyncSession):
    """Stream sync queue records in batches — avoids OOM on large backlogs."""
    result = await db.stream(
        select(SyncQueue).where(SyncQueue.status == "pending").order_by(SyncQueue.created_at)
    )
    async for row in result:
        yield row
```

**Common mistake:** Using a synchronous `with` statement on an async context manager. `with async_ctx_mgr()` will not call `__aenter__`/`__aexit__` as coroutines — it will call them synchronously, returning coroutine objects that are never awaited. Always use `async with`.

---

## 6. Writing Non-Blocking FastAPI Endpoints

FastAPI runs on Starlette/Uvicorn, which runs on an asyncio event loop. When you define an endpoint as `async def`, it runs directly on the event loop thread. When you define it as a plain `def`, FastAPI dispatches it to a thread pool automatically (via `anyio.to_thread.run_sync`).

This means a plain `def` endpoint is actually *safer* for blocking code than a carelessly-written `async def`:

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()

# CORRECT: async endpoint with fully async SQLAlchemy
@app.get("/clients/{client_id}")
async def get_client(
    client_id: str,
    db: AsyncSession = Depends(get_db),
) -> dict:
    result = await db.execute(
        select(Client).where(Client.sf_id == client_id)
    )
    client = result.scalar_one_or_404()
    return client.to_dict()

# CORRECT: sync endpoint for legacy blocking code — FastAPI runs in thread pool
@app.post("/legacy/sync-client")
def sync_client_legacy(client_id: str) -> dict:
    # blocking code is fine here — FastAPI dispatched us off the event loop
    return legacy_sync_epic_call(client_id)

# WRONG: async endpoint that blocks the event loop
@app.get("/clients/{client_id}/bad")
async def get_client_bad(client_id: str) -> dict:
    import time
    time.sleep(2)  # BLOCKS THE ENTIRE EVENT LOOP FOR 2 SECONDS
    return {}
```

For endpoints that must call both async and sync code, use `asyncio.to_thread()` (Python 3.9+) or `loop.run_in_executor()`:

```python
import asyncio

@app.post("/sync-to-epic/{account_id}")
async def sync_account_to_epic(account_id: str) -> dict:
    # Fetch from DB asynchronously
    async with get_db_session() as db:
        account = await get_account(db, account_id)

    # Call a legacy sync Epic SDK function in a thread
    epic_response = await asyncio.to_thread(
        legacy_epic_client.put_client,
        account.to_epic_payload(),
    )
    return {"status": "synced", "epic_id": epic_response["id"]}
```

**Common mistake:** Marking an endpoint `async def` under the assumption it is "faster." If the body contains any blocking calls (sync ORM, `requests` library, `time.sleep`), the async decoration makes things *worse* — it blocks the loop rather than running in the thread pool that `def` would use.

---

## 7. Detecting Sync Code Blocking the Async Loop

Blocking detection is an operational necessity for AESF middleware. A single misfired synchronous DB call during peak load can stall the entire service. There are four approaches, from lightest to most comprehensive.

**Approach 1: `asyncio` debug mode**

```python
# main.py or startup
import asyncio
import logging

logging.basicConfig(level=logging.DEBUG)
asyncio.get_event_loop().set_debug(True)
# Logs a WARNING for any coroutine that takes > 0.1s to yield
# Set threshold: loop.slow_callback_duration = 0.05  # 50ms
```

**Approach 2: `pytest-asyncio` + `asyncio.wait_for` in tests**

```python
import pytest
import asyncio

@pytest.mark.asyncio
async def test_get_client_does_not_block():
    """Endpoint must complete within 500ms even under simulated DB latency."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await asyncio.wait_for(
            ac.get("/clients/SF001"),
            timeout=0.5,
        )
    assert response.status_code == 200
```

**Approach 3: `anyio` backend instrumentation with `trio`**

Running your test suite under `trio` (which `anyio`/FastAPI supports) surfaces blocking calls that `asyncio` tolerates silently, because Trio has stricter cooperative multitasking checks.

**Approach 4: `greenlet`-based blocking detector (production)**

```python
import threading
import time
import asyncio

class BlockingDetector:
    """Periodically checks if the event loop thread is stuck."""

    def __init__(self, threshold_sec: float = 0.1):
        self.threshold = threshold_sec
        self._last_tick = time.monotonic()
        self._running = False

    def tick(self) -> None:
        self._last_tick = time.monotonic()

    def start(self, loop: asyncio.AbstractEventLoop) -> None:
        self._running = True
        loop.call_soon(self._schedule_tick, loop)
        threading.Thread(target=self._monitor, daemon=True).start()

    def _schedule_tick(self, loop: asyncio.AbstractEventLoop) -> None:
        if self._running:
            self.tick()
            loop.call_later(0.05, self._schedule_tick, loop)

    def _monitor(self) -> None:
        while self._running:
            time.sleep(self.threshold)
            gap = time.monotonic() - self._last_tick
            if gap > self.threshold:
                print(f"WARNING: event loop blocked for {gap:.3f}s")
```

**Common mistake:** Relying solely on `asyncio` debug mode in production. Its overhead is significant (it uses `tracemalloc` snapshots) and it should be disabled in prod. Use a lightweight custom detector like the one above, or integrate Datadog APM traces to flag event-loop-blocking spans.

---

## 8. SQLAlchemy Async — Patterns and Pitfalls

SQLAlchemy's async extension (`sqlalchemy.ext.asyncio`) wraps the core sync engine with an `asyncpg`-backed async counterpart. The ORM lazy-loading model does not work in async context — any attribute access that would trigger a lazy load raises `MissingGreenlet`. You must use `selectinload` or `joinedload` to eagerly load relationships.

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload
from sqlalchemy.ext.asyncio import AsyncSession

async def get_account_with_policies(db: AsyncSession, sf_id: str):
    result = await db.execute(
        select(Account)
        .where(Account.sf_id == sf_id)
        .options(
            selectinload(Account.policies).selectinload(Policy.lines),
            selectinload(Account.servicings),
        )
    )
    return result.scalar_one_or_none()
```

For write-heavy operations (bulk inserts from ETL), use `insert().values()` with `execute_many` semantics:

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

async def upsert_sync_queue_batch(
    db: AsyncSession,
    records: list[dict],
) -> None:
    stmt = pg_insert(SyncQueue).values(records)
    stmt = stmt.on_conflict_do_update(
        index_elements=["sf_id", "object_type"],
        set_={"status": stmt.excluded.status, "updated_at": stmt.excluded.updated_at},
    )
    await db.execute(stmt)
    await db.commit()
```

**Common mistake:** Using `Session.refresh()` inside a tight loop. Each refresh is a separate round-trip to PostgreSQL. If you need to refresh 100 records, batch your query with `WHERE id IN (...)` instead of calling `refresh()` per record.

---

## 9. Async httpx — Concurrent Epic API Calls

`httpx.AsyncClient` is the production-grade async HTTP client for AESF middleware. It supports connection pooling, retries via `httpx-retries` or a custom transport, and proper timeout configuration.

```python
import asyncio
import httpx
from typing import Any

EPIC_BASE_URL = "https://de21web/api/v1"
EPIC_TIMEOUT = httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0)

async def make_epic_client() -> httpx.AsyncClient:
    return httpx.AsyncClient(
        base_url=EPIC_BASE_URL,
        timeout=EPIC_TIMEOUT,
        limits=httpx.Limits(max_connections=50, max_keepalive_connections=20),
        headers={"Authorization": f"Bearer {get_epic_token()}"},
    )

async def sync_account_and_contacts(
    account_id: str,
    contact_ids: list[str],
    http: httpx.AsyncClient,
) -> dict[str, Any]:
    """Fan-out: fetch account + all contacts concurrently, then reconcile."""
    async with asyncio.TaskGroup() as tg:
        account_task = tg.create_task(
            http.get(f"/clients/{account_id}"),
            name="fetch-account",
        )
        contact_tasks = [
            tg.create_task(
                http.get(f"/contacts/{cid}"),
                name=f"fetch-contact-{cid}",
            )
            for cid in contact_ids
        ]

    account = account_task.result().json()
    contacts = [t.result().json() for t in contact_tasks]
    return {"account": account, "contacts": contacts}
```

Rate limiting with a semaphore — Epic EHR enforces per-client rate limits. Use `asyncio.Semaphore` to cap concurrency:

```python
_epic_semaphore = asyncio.Semaphore(10)  # max 10 concurrent Epic requests

async def rate_limited_epic_get(http: httpx.AsyncClient, path: str) -> dict:
    async with _epic_semaphore:
        response = await http.get(path)
        response.raise_for_status()
        return response.json()
```

**Common mistake:** Sharing a single `httpx.AsyncClient` instantiated at import time without a lifespan hook. The client's connection pool is tied to the event loop that created it. In tests, each `asyncio.run()` creates a new loop, making the module-level client stale. Use FastAPI's `lifespan` to create and close the client within the same loop.

---

## 10. Structured Concurrency with `asyncio.TaskGroup` and Cancellation

Python 3.11's `TaskGroup` implements structured concurrency: all tasks spawned inside the group are guaranteed to be done (or cancelled) before the `async with` block exits. This eliminates a class of resource-leak bugs where a fire-and-forget task outlives its parent.

```python
import asyncio

async def process_sync_batch(batch: list[str]) -> list[dict]:
    """
    Process a batch of Epic sync operations.
    If any single operation fails, all siblings are cancelled
    and the exception propagates immediately.
    """
    results = []
    try:
        async with asyncio.TaskGroup() as tg:
            tasks = [
                tg.create_task(sync_single_record(record_id))
                for record_id in batch
            ]
    except* httpx.HTTPStatusError as eg:
        # ExceptionGroup handling (Python 3.11+)
        for exc in eg.exceptions:
            print(f"Epic API error: {exc.response.status_code}")
        raise
    return [t.result() for t in tasks]
```

Cancellation propagation is the hardest part of async code to get right. When a `Task` is cancelled, an `asyncio.CancelledError` is raised at the next `await` point inside that task. Code that catches `Exception` without re-raising `CancelledError` will suppress cancellation silently:

```python
# WRONG: swallows CancelledError
async def bad_worker():
    try:
        await asyncio.sleep(10)
    except Exception:  # catches CancelledError too!
        pass

# CORRECT: re-raise CancelledError
async def good_worker():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        # cleanup if needed
        raise  # always re-raise
    except Exception as e:
        handle_error(e)
```

**Common mistake:** Using `asyncio.shield()` to protect a coroutine from cancellation without understanding that the outer task is still cancelled. `shield` only prevents the inner awaitable from receiving the cancellation signal — the outer coroutine still gets cancelled after the shielded operation completes.

---

## 11. Key Concepts Summary

```
Python Concurrency Model
├── asyncio (I/O-bound, single-threaded cooperative)
│   ├── Event Loop
│   │   ├── Ready queue  (call_soon)
│   │   └── Scheduled queue  (call_later, min-heap)
│   ├── Awaitables
│   │   ├── Coroutine  (async def — lazy, not scheduled)
│   │   ├── Task       (create_task — scheduled immediately)
│   │   └── Future     (low-level, resolved by callbacks)
│   ├── Structured Concurrency
│   │   ├── TaskGroup  (Python 3.11+ — preferred)
│   │   └── gather()   (legacy — less safe)
│   └── Bridges to sync world
│       ├── run_in_executor  (thread/process pool)
│       └── asyncio.to_thread  (Python 3.9+)
├── threading (I/O-bound, GIL-constrained)
│   ├── ThreadPoolExecutor  (preferred)
│   └── Direct Thread  (avoid in async context)
└── multiprocessing (CPU-bound, GIL-bypassing)
    └── ProcessPoolExecutor  (preferred)

AESF Middleware Concurrency Patterns
├── Epic API calls  →  httpx.AsyncClient + TaskGroup + Semaphore
├── DB reads        →  SQLAlchemy async + selectinload (no lazy load)
├── DB writes       →  batch upsert, avoid per-row refresh()
├── Legacy sync     →  asyncio.to_thread() or run_in_executor()
└── Blocking detect →  asyncio debug mode (dev) + custom detector (prod)
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the difference between a coroutine and a Task in asyncio?

**2.** Why does `time.sleep(1)` inside an `async def` endpoint block all other requests, even though the endpoint is async?

**3.** In Python 3.13, when should you use `asyncio.get_running_loop()` vs. `asyncio.get_event_loop()`?

**4.** You have 50 account IDs and need to fetch each from the Epic API. Write the high-level structure of a concurrent fetch using `TaskGroup` and a `Semaphore`.

**5.** What is `asyncio.gather(return_exceptions=True)` useful for, and what is the risk of `return_exceptions=False`?

**6.** Why does FastAPI dispatch plain `def` endpoints to a thread pool, and when is this behaviour actually desirable?

**7.** What does `MissingGreenlet` mean in SQLAlchemy async, and how do you fix it?

**8.** Explain the difference between `asyncio.CancelledError` and a regular exception. Why must you never swallow it?

**9.** What is structured concurrency, and why does `TaskGroup` enforce it better than `gather()`?

**10.** A colleague says "I made the database call faster by switching from `def` to `async def`." Is this claim correct? Explain why or why not.

**11.** What is the GIL, and why does it mean that `ThreadPoolExecutor` provides no speedup for CPU-bound Python code?

**12.** Describe two production techniques for detecting event-loop blocking in an AESF FastAPI service.

**13.** What is `asyncio.to_thread()` and how does it differ from `loop.run_in_executor(None, fn)`?

**14.** In the context of `httpx.AsyncClient`, why is it wrong to create the client at module-level without a lifespan hook?

**15.** What is a `Semaphore` in asyncio, and give a concrete reason to use one when calling the Epic EHR API?

**16.** Explain `asyncio.shield()`: what it protects and what it does NOT protect.

**17.** You have a `ProcessPoolExecutor`. What kinds of objects can be passed as arguments to the submitted function, and why?

**18.** What is the difference between `selectinload` and `joinedload` in SQLAlchemy async, and when would you prefer each?

**19.** How does `asyncio.TaskGroup` handle the case where one of its child tasks raises an exception?

**20.** Write the skeleton of a FastAPI `lifespan` context manager that initialises a shared `httpx.AsyncClient` and a `ThreadPoolExecutor` for use across all endpoints.

---

### Answers

??? note "Reveal Answers"

    **1.** A coroutine is a generator-like object returned by calling an `async def` function. It is lazy: no code executes until it is awaited or wrapped in a Task. A Task is a subclass of `asyncio.Future` that wraps a coroutine and schedules it to run on the event loop immediately. Tasks enable true concurrent execution — creating two Tasks means both coroutines progress interleaved through their `await` points, whereas awaiting two coroutines in sequence means the second starts only after the first completes. Use `asyncio.create_task()` whenever you want concurrent execution without waiting for the first to finish.

    **2.** `time.sleep()` is a blocking syscall: it suspends the OS thread for the specified duration. The event loop runs on that same thread, so while the thread is sleeping, the event loop cannot run any other callbacks, schedule I/O, or service any other request. The cooperative nature of asyncio means no other coroutine gets a chance to run until the blocking call returns. Replace `time.sleep()` with `await asyncio.sleep()`, which suspends the coroutine and returns control to the event loop, allowing other work to proceed.

    **3.** `asyncio.get_running_loop()` is the correct call inside a coroutine or any code that knows it is running within an async context. It raises `RuntimeError` if no loop is running, making errors explicit. `asyncio.get_event_loop()` was the older, more permissive API that would create a new loop if none existed. In Python 3.10+ this creation behaviour is deprecated, and in 3.12+ it raises a warning in many cases. As a rule: use `get_running_loop()` inside coroutines, and `asyncio.run()` at the program entry point.

    **4.** Wrap all task creation in an `async with asyncio.TaskGroup() as tg:` block. Before creating tasks, declare `sem = asyncio.Semaphore(10)`. Each task coroutine should begin with `async with sem:` before calling `http.get(...)`. This ensures at most 10 concurrent requests fly to Epic at any moment, while still issuing all 50 requests concurrently in batches constrained by the semaphore. TaskGroup ensures all 50 tasks complete (or one exception cancels the rest) before the block exits.

    **5.** `return_exceptions=True` causes `gather()` to collect exceptions as regular return values rather than propagating them, so all tasks run to completion regardless of individual failures. This is useful when you want partial results — for example, syncing 100 accounts and reporting which ones failed rather than aborting on the first failure. The risk of `return_exceptions=False` (the default) is that the first exception immediately propagates to the caller, but the other tasks continue running in the background with no handle to cancel them, potentially causing resource leaks or duplicate operations.

    **6.** FastAPI (via `anyio`) detects plain `def` endpoints and runs them in a default thread pool so they do not block the event loop thread. This is useful for legacy code or third-party libraries that are inherently synchronous and cannot be easily converted — for example, a sync Salesforce SDK call or a legacy Epic client library. It is preferable to mark such endpoints as plain `def` rather than wrapping them in `async def` with ad hoc `run_in_executor` calls, because FastAPI's automatic dispatch is cleaner and less error-prone.

    **7.** `MissingGreenlet` is raised by SQLAlchemy async when ORM code tries to perform lazy-loading (implicitly issuing a SQL query to load a relationship) from outside a greenlet context — i.e., from pure asyncio code. SQLAlchemy async uses greenlets internally to bridge the sync ORM layer to async I/O. The fix is to eagerly load all relationships you intend to access using `selectinload()` or `joinedload()` in the original query, so no additional SQL is needed after the query returns.

    **8.** `asyncio.CancelledError` is a subclass of `BaseException` (not `Exception` in Python 3.8+), which means `except Exception` does not catch it. When a Task is cancelled, `CancelledError` is injected at the next `await`. If code catches it and does not re-raise, the cancellation is silently suppressed: the task continues running and the entity that requested cancellation (a TaskGroup, a timeout, a signal handler) will hang waiting for the task to stop. Always re-raise `CancelledError` after any necessary cleanup, ensuring cancellation propagates correctly through the task hierarchy.

    **9.** Structured concurrency is the principle that child tasks cannot outlive their parent scope. `TaskGroup` enforces this: you cannot exit the `async with` block until all tasks spawned inside it have finished or been cancelled. `gather()` has no such guarantee — tasks passed to `gather()` are already scheduled before `gather()` is awaited, and if you cancel the `gather()` awaitable, the tasks may or may not be cancelled depending on the Python version and configuration. `TaskGroup` also handles `ExceptionGroup` semantics (Python 3.11+), collecting multiple simultaneous exceptions into one.

    **10.** The claim is incorrect as stated. Switching from `def` to `async def` changes how FastAPI dispatches the endpoint — from a thread pool to the event loop thread directly — but it does not make the underlying operation faster. If the endpoint body still contains synchronous blocking code (sync ORM calls, the `requests` library, file I/O), making it `async def` actually makes performance worse: the blocking now stalls the event loop rather than occupying one thread in the pool. True speedup requires replacing blocking I/O with non-blocking async equivalents (e.g., `asyncpg`, `httpx.AsyncClient`).

    **11.** The Global Interpreter Lock is a mutex in CPython that prevents more than one thread from executing Python bytecode simultaneously. For I/O-bound work (network, disk), threads release the GIL while waiting on OS syscalls, so multiple threads can make progress concurrently even though only one runs Python bytecode at a time. For CPU-bound work (parsing, compression, numerical computation), threads never release the GIL during computation, so a `ThreadPoolExecutor` with N workers runs essentially the same speed as a single thread — only one thread computes at any instant. `ProcessPoolExecutor` bypasses the GIL by spawning separate interpreter processes, each with their own GIL.

    **12.** First: enable `asyncio` debug mode (`loop.set_debug(True)` and `loop.slow_callback_duration = 0.05`) in development and staging. This logs a warning whenever a callback takes longer than 50ms, with the offending coroutine's stack trace. Second: deploy a lightweight custom `BlockingDetector` that runs a background thread, periodically checks the timestamp of the last event loop tick, and emits a Datadog metric or log when the gap exceeds a threshold. This approach has negligible production overhead and provides real-time visibility into blocking events under production load.

    **13.** `asyncio.to_thread(fn, *args)` is a convenience wrapper introduced in Python 3.9 that runs `fn(*args)` in the default executor (a `ThreadPoolExecutor` managed by the event loop) and returns an awaitable. `loop.run_in_executor(None, fn, *args)` does the same thing with `None` meaning "use the default executor." `to_thread` is preferred in modern code for readability and because it accepts keyword arguments via `functools.partial`. The key difference is that `to_thread` does not allow you to specify a custom executor — use `run_in_executor` when you need a dedicated pool with specific sizing or a `ProcessPoolExecutor`.

    **14.** An `httpx.AsyncClient` creates its internal connection pool and binds it to the event loop that was running when the client was created. In a testing environment where each test calls `asyncio.run()`, each run creates a new event loop, making any previously created client stale — connections in its pool reference the old loop. In production, if the ASGI lifespan has not started the loop yet, the module-level client is also misconfigured. The correct pattern is to create the client inside a FastAPI `lifespan` async context manager, where the event loop is already running and the client lifetime is tied to the application lifetime.

    **15.** `asyncio.Semaphore(n)` is a counter-based lock: `async with sem:` decrements the counter (blocking if it reaches zero) and increments it on exit. It limits the number of concurrent coroutines that hold the semaphore at any time. For Epic EHR API calls, the Epic server enforces rate limits (e.g., 10 requests/second per client). Without a semaphore, 100 concurrent tasks could all make Epic API calls simultaneously, triggering HTTP 429 rate-limit errors. A `Semaphore(10)` ensures at most 10 calls are in-flight at once, naturally spreading load and avoiding rate limit breaches.

    **16.** `asyncio.shield(coro_or_future)` wraps the inner awaitable so that if the outer Task is cancelled, the cancellation is NOT forwarded to the inner awaitable — the inner continues running. However, `shield` does NOT protect the outer Task from cancellation: after the inner completes, `CancelledError` is still raised in the outer task at the `await asyncio.shield(...)` point. A common misuse is assuming `shield` makes an operation "uncancellable" — it only decouples the inner awaitable's cancellation from the outer task's, which is useful for cleanup operations (e.g., "cancel the request, but let the DB commit finish").

    **17.** Arguments passed to a `ProcessPoolExecutor` worker function must be picklable, because they are serialized and sent to the child process via `multiprocessing`'s IPC mechanism. This means: built-in types (int, str, list, dict), most dataclasses and plain objects, and numpy arrays all work. Things that do NOT pickle include lambda functions, closures that capture non-picklable state, database connections, file handles, asyncio event loops, and `httpx` clients. For AESF use cases, pass raw data (dicts, bytes, strings) to process pool workers, not ORM objects or network clients.

    **18.** `selectinload` issues a separate `SELECT ... WHERE id IN (...)` query to load the related collection after the parent query completes. It is efficient when loading many parents with many children because it avoids a cartesian product. `joinedload` adds a `JOIN` to the parent query and loads everything in one round-trip, which is more efficient when you have a small number of parents and you know each will have few children. For AESF, prefer `selectinload` for loading policies on an Account (potentially hundreds of policies) and `joinedload` for loading a single related object (e.g., an Account's primary branch).

    **19.** When any child task inside a `TaskGroup` raises an unhandled exception, the TaskGroup immediately cancels all remaining sibling tasks by calling `task.cancel()` on each. It then waits for all siblings to finish (either naturally or by handling their `CancelledError`). After all tasks are settled, the TaskGroup collects all exceptions (from the original failure and any tasks that also raised during cancellation) into an `ExceptionGroup` and raises it. The caller handles it with `except*` syntax. This guarantees no task is silently abandoned and no exception is silently swallowed.

    **20.** The skeleton:
    ```python
    from contextlib import asynccontextmanager
    from concurrent.futures import ThreadPoolExecutor
    import httpx
    from fastapi import FastAPI

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        # Startup
        app.state.http_client = httpx.AsyncClient(
            base_url="https://de21web/api/v1",
            timeout=httpx.Timeout(30.0),
        )
        app.state.thread_pool = ThreadPoolExecutor(max_workers=10)
        yield
        # Shutdown
        await app.state.http_client.aclose()
        app.state.thread_pool.shutdown(wait=True)

    app = FastAPI(lifespan=lifespan)
    ```
    Endpoints access `request.app.state.http_client` and `request.app.state.thread_pool` via the `Request` dependency. This ensures both resources are created on the running event loop and cleaned up gracefully on shutdown.

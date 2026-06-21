# Week 12 — Advanced Python — Internals & CPython

**Week of:** August 24, 2026
**Estimated study time:** ~2 hours
**Tags:** `python` `cpython` `performance`

---

## Overview

Most Python engineers spend years writing production code without ever looking at what CPython is actually doing on their behalf. That gap becomes a liability when you're diagnosing a slow FastAPI endpoint, hunting a memory leak in a long-running Kubernetes pod, or deciding whether `asyncio` will actually parallelize your CPU-bound database fan-out. This week closes that gap by walking through the CPython execution model from source to bytecode to object lifecycle.

The crm-middleware is a particularly good lens for these topics. It runs as a persistent Kubernetes service handling thousands of EHR-to-Salesforce sync events per day. Each request touches SQLAlchemy ORM objects, serializes Pydantic models, and often fans out to multiple downstream HTTP calls. Understanding how CPython manages the memory for those objects, how the GIL interacts with `asyncio`'s event loop, and where cycles accumulate in long-lived services gives you a concrete return on the two hours you'll spend here.

Python 3.13 introduces the experimental free-threaded build (PEP 703), which disables the GIL entirely. This is not yet production-ready for most workloads, but it changes the calculus for CPU-bound parallelism significantly, and it's worth understanding what the GIL was actually protecting before celebrating its removal. We'll cover the GIL's real scope — and the surprising things it does *not* protect.

Finally, profiling. `cProfile` gives you call-count statistics cheaply. `py-spy` is a sampling profiler that attaches to a running process without code changes — invaluable for production middleware where you can't afford `cProfile` overhead on live traffic. You'll leave this week knowing which tool to reach for in which scenario, and how to read the output without being misled by instrumentation artifacts.

---

## 1. The CPython Execution Model

CPython compiles Python source to **bytecode** (`.pyc` files) and then interprets that bytecode in a virtual machine called `ceval.c`. The compilation step is fast and happens implicitly; the bytecode lives in the `__pycache__` directory alongside your source files.

Every function, class body, and module is compiled into a **code object** (`types.CodeType`). Code objects are immutable and carry the bytecode as `co_code`, the constant pool as `co_consts`, and the variable names as `co_varnames`. When a function is *called*, CPython wraps a code object in a **frame object** (`PyFrameObject`) that holds the local variable slots and the evaluation stack for that specific invocation.

```python
import dis

def compute_sync_delay(queue_depth: int, base_ms: float = 50.0) -> float:
    """Estimate EHR sync delay from queue depth."""
    return base_ms * (1 + queue_depth / 100)

dis.dis(compute_sync_delay)
```

Typical output (Python 3.13):

```
  4           0 RESUME                   0

  5           2 LOAD_FAST                1 (base_ms)
              4 LOAD_CONST               1 (1)
              6 LOAD_FAST                0 (queue_depth)
              8 LOAD_CONST               2 (100)
             10 BINARY_OP                11 (/)
             12 BINARY_OP                0 (+)
             14 BINARY_OP                5 (*)
             16 RETURN_VALUE
```

Each line is an **opcode** — a single byte identifying the operation — plus an optional argument. `LOAD_FAST` pushes a local variable onto the evaluation stack. `BINARY_OP` pops two values, applies the operation, and pushes the result. `RETURN_VALUE` pops the top-of-stack and returns it to the caller.

You can inspect a code object directly:

```python
code = compute_sync_delay.__code__
print(code.co_varnames)   # ('queue_depth', 'base_ms')
print(code.co_consts)     # (None, 1, 100, 'Estimate EHR sync delay from queue depth.')
print(code.co_argcount)   # 2
```

**Common mistake:** Assuming that Python's compilation step is equivalent to a "build" in a compiled language. CPython's bytecode is not optimized across function boundaries; the interpreter evaluates each opcode at runtime, making Python fundamentally interpreter-bound. Specializations introduced in Python 3.11–3.13 (adaptive interpreter) provide limited inline caching, but Python is still orders of magnitude slower than native code for CPU-heavy loops.

**Integration platform connection:** When a sync endpoint deserializes a large EHR payload into nested Pydantic models, every attribute access, dict lookup, and list append goes through the bytecode interpreter. This is why moving heavy transformation logic out of the hot path — e.g., offloading to background Celery tasks — can matter more than micro-optimizing the Python code itself.

---

## 2. Bytecode Disassembly in Practice

The `dis` module is your primary tool for understanding what Python actually does with your code. The recursive variant `dis.dis()` handles nested functions and comprehensions; `dis.Bytecode()` gives you a programmatic iterator.

```python
import dis

# Inspect a list comprehension — it compiles to its own code object
expr = compile("[x*2 for x in range(10)]", "<string>", "eval")
dis.dis(expr)
```

```python
# Check the specialization state in Python 3.13 (adaptive interpreter)
# BINARY_OP may be specialized to BINARY_OP_ADD_INT after warmup
import sys

def add_ints(a: int, b: int) -> int:
    return a + b

# Warm up the function so the adaptive interpreter specializes it
for _ in range(100):
    add_ints(1, 2)

dis.dis(add_ints)
# You may now see BINARY_OP_ADD_INT instead of plain BINARY_OP
```

Useful `dis` attributes to know:

| Attribute | Meaning |
|-----------|---------|
| `dis.opname` | List mapping opcode numbers to names |
| `dis.stack_effect(op, arg)` | Net stack depth change for a given opcode |
| `code.co_stacksize` | Maximum stack depth the compiler computed |
| `code.co_flags` | Bitmask for generator, coroutine, nested flags |

**Inspecting a FastAPI route's code object:**

```python
from app.api.v2.sync import sync_ehr_client  # your actual route handler

import dis
dis.dis(sync_ehr_client)

# For async functions, bytecode includes GET_AWAITABLE, SEND, YIELD_VALUE
# showing exactly where coroutine suspension points are
```

**Common mistake:** Treating bytecode output as a performance oracle. Fewer opcodes does not mean faster execution — `CALL_FUNCTION` dispatching into a C extension is far cheaper than a tight loop of pure-Python opcodes. Bytecode disassembly is a diagnostic tool, not a benchmark.

---

## 3. The GIL — What It Actually Blocks

The **Global Interpreter Lock** (GIL) is a mutex that CPython holds whenever it executes Python bytecode. Only one OS thread can hold the GIL at a time, which means only one thread can execute Python bytecode at any moment — even on a multi-core machine.

**What the GIL blocks:**
- Parallel CPU-bound execution across Python threads
- Concurrent bytecode interpretation (e.g., two threads both running a tight numeric loop)

**What the GIL does NOT block:**
- I/O — `socket`, `file.read()`, `requests.get()` all release the GIL while waiting on the OS
- NumPy/pandas operations that drop into C with the GIL released
- `multiprocessing` — each subprocess has its own interpreter and its own GIL
- `asyncio` — it's single-threaded by design; the GIL is largely irrelevant

The GIL is released at two points: (1) every 5ms by default (`sys.getswitchinterval()`), giving other threads a chance to run, and (2) explicitly by C extension code that calls `Py_BEGIN_ALLOW_THREADS`.

```python
import sys
print(sys.getswitchinterval())  # 0.005 (5ms)

# You can tighten this for more responsive thread switching (at the cost of throughput)
sys.setswitchinterval(0.001)
```

**Python 3.13 free-threaded mode:**

```bash
# Build/install the free-threaded variant
python3.13t -c "import sys; print(sys._is_gil_enabled())"
# False — GIL is disabled

# Per-session opt-in at import time
import sys
sys.flags.ignore_environment  # check if running t-build
```

Even with the free-threaded build, shared mutable Python objects require explicit locking — the GIL was implicitly serializing those accesses before.

**Common mistake:** Writing `threading.Thread` code expecting CPU parallelism. In standard CPython, two threads running a pure-Python number-crunching loop will not run faster than one — they'll actually be slower due to GIL contention and context-switch overhead. Use `multiprocessing.Pool` or `concurrent.futures.ProcessPoolExecutor` for CPU parallelism.

**Integration platform connection:** The crm-middleware uses `asyncio` (FastAPI). The GIL is essentially irrelevant here because `asyncio` runs on a single thread — its concurrency comes from cooperative yielding at `await` points, not from threading. The real risk is *blocking the event loop* with synchronous SQLAlchemy calls, which holds the single thread and prevents all other coroutines from running. Use `asyncio.to_thread()` or `run_in_executor()` to move synchronous DB work off the event loop.

---

## 4. Memory Management: Reference Counting

CPython uses **reference counting** as its primary memory management strategy. Every Python object has a `ob_refcnt` field. When a reference is created the count increments; when a reference is released (variable goes out of scope, del, reassignment) the count decrements. When the count reaches zero the object is immediately deallocated.

```python
import sys

obj = [1, 2, 3]
print(sys.getrefcount(obj))   # 2: one for `obj`, one for the getrefcount() argument

alias = obj
print(sys.getrefcount(obj))   # 3

del alias
print(sys.getrefcount(obj))   # 2 again
```

Note that `getrefcount()` always adds 1 because passing `obj` to the function creates a temporary reference.

Reference counting has a key advantage: **deterministic destruction**. When an SQLAlchemy session goes out of scope in a FastAPI dependency, its `__del__` is called immediately (assuming no cycles). This makes resource cleanup predictable.

**Small integer caching:** CPython pre-allocates integers from -5 to 256 as singletons. This is an implementation detail, not a language guarantee.

```python
a = 256
b = 256
print(a is b)   # True — same object

a = 257
b = 257
print(a is b)   # False (usually) — different objects
```

**String interning:** Short strings that look like identifiers are interned automatically. You can force interning with `sys.intern()`.

**Common mistake:** Relying on `__del__` for critical cleanup (e.g., closing a DB connection). If a reference cycle exists, `__del__` may never be called promptly or at all. Always use context managers (`with` statements) for resource management.

**Integration platform connection:** In the crm-middleware, SQLAlchemy `Session` objects should always be acquired via the `get_db` FastAPI dependency (which uses `yield` and a `finally` block), not held as module-level globals. A module-level session that accumulates references from circular imports could live indefinitely, holding a PostgreSQL connection.

---

## 5. Memory Management: Cyclic Garbage Collection

Reference counting cannot handle **reference cycles** — objects that point to each other with no external references:

```python
import gc

class Node:
    def __init__(self, name: str):
        self.name = name
        self.children: list["Node"] = []

# Create a cycle
a = Node("policy")
b = Node("line")
a.children.append(b)
b.children.append(a)   # cycle: a → b → a

del a, b   # refcount never hits 0; leaked without cyclic GC
gc.collect()           # forces cyclic collection
```

CPython's **cyclic garbage collector** (in `gc` module) runs automatically in three generations. Objects that survive a collection get promoted to older generations, which are collected less frequently. The generational hypothesis is that most objects die young.

```python
import gc

# Check collection thresholds
print(gc.get_threshold())   # (700, 10, 10) — generation 0 collects every 700 allocations

# Manually trigger a full collection
gc.collect(2)   # collect all three generations

# Find cycles in your object graph
gc.set_debug(gc.DEBUG_LEAK)
gc.collect()
```

In production, you can profile allocations to detect leaks:

```python
import tracemalloc

tracemalloc.start()

# ... run your workload ...

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics("lineno")
for stat in top_stats[:10]:
    print(stat)
```

**Common mistake:** Disabling the GC (`gc.disable()`) for performance without understanding the consequences. Some high-throughput servers do this and rely on not creating cycles. This is safe only if you can guarantee cycle-free data structures — which SQLAlchemy ORM objects are *not* (they contain backreferences to the session and mapper).

**Integration platform connection:** Long-running middleware pods that process thousands of requests can accumulate cycles from Pydantic model instances that reference each other (e.g., nested response models that include a back-reference to the parent). Enabling `tracemalloc` in staging and watching RSS growth over time is a standard way to catch this.

---

## 6. Descriptors, `__slots__`, and Metaclasses

**Descriptors** are objects that implement `__get__`, `__set__`, or `__delete__`. When a class attribute is a descriptor, Python calls these methods instead of doing a plain dict lookup. This is the mechanism underlying `property`, `classmethod`, `staticmethod`, and SQLAlchemy's `Column`.

```python
class ValidatedField:
    """A descriptor that validates positive integers."""

    def __set_name__(self, owner, name: str):
        self.name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.name, None)

    def __set__(self, obj, value: int):
        if not isinstance(value, int) or value < 0:
            raise ValueError(f"{self.name} must be a non-negative integer")
        setattr(obj, self.name, value)


class SyncJob:
    queue_depth = ValidatedField()

    def __init__(self, queue_depth: int):
        self.queue_depth = queue_depth


job = SyncJob(42)
print(job.queue_depth)   # 42
job.queue_depth = -1     # raises ValueError
```

**`__slots__`** replaces the per-instance `__dict__` with a fixed-layout C array of slots, saving memory and speeding up attribute access:

```python
class EHRRecord:
    __slots__ = ("client_id", "policy_number", "timestamp")

    def __init__(self, client_id: str, policy_number: str, timestamp: float):
        self.client_id = client_id
        self.policy_number = policy_number
        self.timestamp = timestamp


# No __dict__ → ~40–50% less memory per instance for attribute-heavy classes
import sys
print(sys.getsizeof(EHRRecord("C001", "P001", 0.0)))  # smaller than dict-backed class
```

**Metaclasses** control class *creation*. The default metaclass is `type`. You can intercept `__new__` to validate or transform a class at definition time:

```python
class SyncModelMeta(type):
    """Ensure every sync model declares a `sync_direction` class attribute."""

    def __new__(mcs, name, bases, namespace):
        if name != "BaseSyncModel" and "sync_direction" not in namespace:
            raise TypeError(f"{name} must declare `sync_direction`")
        return super().__new__(mcs, name, bases, namespace)


class BaseSyncModel(metaclass=SyncModelMeta):
    pass


class AccountSync(BaseSyncModel):
    sync_direction = "bidirectional"
```

**`__init_subclass__`** is a lighter alternative to metaclasses for subclass hooks:

```python
class BaseSyncModel:
    def __init_subclass__(cls, direction: str = "unknown", **kwargs):
        super().__init_subclass__(**kwargs)
        cls.sync_direction = direction
        print(f"Registered {cls.__name__} with direction={direction}")


class AccountSync(BaseSyncModel, direction="bidirectional"):
    pass
```

**Common mistake:** Using `__slots__` on a class that you later subclass without also defining `__slots__` in the subclass. The subclass will have a `__dict__` again, negating the memory benefit for subclass instances.

---

## 7. The Import System

When Python executes `import foo`, it:

1. Checks `sys.modules` — if `foo` is already there, returns cached module immediately.
2. Finds the module using **finders** in `sys.meta_path` (e.g., `BuiltinImporter`, `FrozenImporter`, `PathFinder`).
3. Loads the module using a **loader** (reads the `.py` file or `.pyc` cache).
4. Executes the module's code in a new namespace.
5. Stores the result in `sys.modules`.

```python
import sys

# Check what's cached
print("fastapi" in sys.modules)   # True if FastAPI has been imported

# Inspect meta_path finders
for finder in sys.meta_path:
    print(type(finder).__name__)

# Force a module reload (useful in development, dangerous in production)
import importlib
import app.core.config
importlib.reload(app.core.config)
```

**Circular imports** are a common pain point in FastAPI projects. They occur when module A imports module B, which imports module A before A has finished initializing:

```python
# models.py
from app.schemas import PolicySchema   # schemas.py imports from models.py → circular!

# Solution: use TYPE_CHECKING guard
from __future__ import annotations
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from app.schemas import PolicySchema
```

**Custom importers** via `importlib.abc.MetaPathFinder` let you intercept imports — useful for plugin systems or lazy-loading heavy dependencies:

```python
import importlib.util
import sys

spec = importlib.util.spec_from_file_location(
    "dynamic_module", "/path/to/module.py"
)
mod = importlib.util.module_from_spec(spec)
sys.modules["dynamic_module"] = mod
spec.loader.exec_module(mod)
```

**Common mistake:** Importing at module level inside a frequently-called function. Each `import` statement still checks `sys.modules`, which is a dict lookup — fast, but not free. Move module-level imports to the top of the file. The exception is avoiding circular imports or importing optional heavy dependencies lazily.

**Integration platform connection:** The crm-middleware's `app/core/config.py` is imported at startup and cached in `sys.modules`. If you accidentally trigger a re-import (e.g., via `importlib.reload` in a middleware hook), you'll get a second `Settings` instance, breaking singleton config assumptions. This is a subtle production footgun.

---

## 8. Profiling with cProfile

`cProfile` is a **deterministic profiler** — it instruments every function call and records exact counts and cumulative times. It's built into the standard library and has near-zero overhead for I/O-heavy code, but can inflate measured times for tight CPU loops.

```python
import cProfile
import pstats
import io

def profile_sync_batch(records: list[dict]) -> None:
    pr = cProfile.Profile()
    pr.enable()

    # --- code under test ---
    for record in records:
        process_ehr_record(record)
    # -----------------------

    pr.disable()

    stream = io.StringIO()
    stats = pstats.Stats(pr, stream=stream)
    stats.sort_stats("cumulative")
    stats.print_stats(20)
    print(stream.getvalue())
```

As a CLI tool:

```bash
python -m cProfile -s cumulative -o profile.out app/scripts/run_sync.py
python -m pstats profile.out
# Inside pstats: sort cumulative; stats 20
```

Visualize with SnakeViz:

```bash
pip install snakeviz
snakeviz profile.out
```

Key columns in `pstats` output:

| Column | Meaning |
|--------|---------|
| `ncalls` | Number of times the function was called |
| `tottime` | Time in the function itself (excluding callees) |
| `cumtime` | Cumulative time including all callees |
| `percall` | `tottime / ncalls` or `cumtime / ncalls` |

**Common mistake:** Sorting by `cumtime` and blaming the top function. High `cumtime` on `asyncio.run()` or `uvicorn.main()` just means "the whole app ran inside this." Look for functions with high `tottime` and high `ncalls` — they represent genuine hot spots.

---

## 9. Profiling with py-spy

`cProfile` requires code changes. **py-spy** is a **sampling profiler** that attaches to a running process via its PID — no code changes, no restarts, safe for production use.

```bash
pip install py-spy

# Attach to a running FastAPI/uvicorn process (requires root or ptrace permissions)
py-spy top --pid 12345

# Record a flamegraph for 60 seconds
py-spy record -o profile.svg --pid 12345 --duration 60

# Profile a script from the start
py-spy record -o profile.svg -- python app/scripts/run_sync.py
```

The flamegraph SVG is interactive: click a frame to zoom, hover for time percentages. Tall narrow towers indicate deep call stacks; wide flat bars indicate time spent in a function.

**Combining cProfile and py-spy:**

| Scenario | Tool |
|----------|------|
| Development: which function is slow? | `cProfile` |
| Production: live process, no restart | `py-spy top` |
| Production: root cause, flamegraph | `py-spy record` |
| Memory leak hunt | `tracemalloc` + `objgraph` |
| GC pressure | `gc.get_count()` + `gc.callbacks` |

**py-spy in Kubernetes:**

```bash
# Exec into the pod
kubectl exec -it crm-middleware-xyz -- bash

# Inside the pod, attach to uvicorn worker
py-spy top --pid $(pgrep -f uvicorn)
```

Note: you may need to add `SYS_PTRACE` capability to the pod's security context for `py-spy` to work.

**Common mistake:** Running `py-spy` on a process that's already at 100% CPU and concluding that the slowest frame *in py-spy* is the bottleneck. If the process is I/O-bound (waiting on Postgres, the EHR system API), `py-spy` will mostly sample idle time in the event loop, not your slow code. Check `async with asyncio.timeout()` and Postgres query plans first.

**Integration platform connection:** During an EHR sync backlog incident, attach `py-spy top` to the crm-middleware pod in staging. If the flamegraph shows wide bands in SQLAlchemy's `execute()` → `psycopg2._psycopg.cursor.execute()`, the bottleneck is query time, not Python. If it shows wide bands in Pydantic's `model_validate()`, you have a schema complexity problem worth optimizing.

---

## 10. Key Concepts Summary

```
CPython Execution Model
├── Source (.py)
│   └── compile() → Code Object (bytecode + consts + varnames)
│       └── call() → Frame Object (locals + eval stack)
│           └── ceval.c → opcode dispatch loop
│
├── Memory Management
│   ├── Reference Counting (ob_refcnt) — immediate dealloc on refcount=0
│   └── Cyclic GC (gc module) — generational, handles cycles
│       └── tracemalloc — allocation snapshots for leak hunting
│
├── Object Model
│   ├── Descriptors (__get__/__set__/__delete__)
│   │   └── property, classmethod, SQLAlchemy Column
│   ├── __slots__ — fixed-layout C array, no __dict__
│   └── Metaclasses / __init_subclass__ — class creation hooks
│
├── GIL
│   ├── Blocks: parallel bytecode execution across threads
│   ├── Does NOT block: I/O, C extensions that release GIL, multiprocessing
│   ├── asyncio: single-threaded cooperative — GIL mostly irrelevant
│   └── Python 3.13t: free-threaded build (PEP 703) — GIL disabled
│
├── Import System
│   ├── sys.modules cache (dict lookup first)
│   ├── sys.meta_path finders → loaders
│   ├── Circular import solutions: TYPE_CHECKING guard, lazy imports
│   └── importlib for dynamic / programmatic imports
│
└── Profiling
    ├── dis — bytecode inspection, not a benchmark
    ├── cProfile — deterministic, call counts, dev/test
    ├── py-spy — sampling, zero instrumentation, production-safe
    └── tracemalloc — memory allocation tracing
```

---

## Quiz — 20 Questions

### Questions

**1.** What is a CPython code object, and how does it differ from a frame object?

**2.** You run `dis.dis(my_func)` and see `LOAD_FAST` for one variable and `LOAD_GLOBAL` for another. What does this tell you about each variable's scope?

**3.** Explain why two Python threads running CPU-bound loops on a 16-core machine will not achieve 2× speedup under standard CPython.

**4.** You have a FastAPI endpoint that calls a synchronous SQLAlchemy query. Why is this dangerous in an async context, and how do you fix it?

**5.** `sys.getrefcount(x)` always returns at least 1 more than you expect. Why?

**6.** A Python object's reference count reaches zero, but its `__del__` method is never called. What could cause this?

**7.** What is the generational hypothesis in garbage collection, and how does CPython's `gc` module use it?

**8.** You define `__slots__ = ("x", "y")` on a class. A subclass inherits from it without defining its own `__slots__`. What happens?

**9.** What is the descriptor protocol, and name three built-in Python features that use it?

**10.** What is the difference between a metaclass and `__init_subclass__`? When would you prefer one over the other?

**11.** Describe the five steps CPython performs when executing `import foo` for the first time.

**12.** How would you diagnose a circular import error in a FastAPI project, and what pattern resolves it without restructuring the modules?

**13.** In `cProfile` output, what is the difference between `tottime` and `cumtime`, and which is more useful for finding the actual bottleneck?

**14.** `py-spy top` on a live middleware pod shows that most time is spent inside `select()` (the OS system call). What does this tell you about the process?

**15.** What is the adaptive interpreter introduced in Python 3.11 and refined in 3.13, and how does it affect the bytecode you see with `dis`?

**16.** You suspect a memory leak in the crm-middleware. Walk through the steps you'd take to confirm it and identify the source.

**17.** Why is `gc.disable()` dangerous in a service that uses SQLAlchemy ORM objects?

**18.** Explain the difference between a data descriptor and a non-data descriptor. Which takes precedence over an instance's `__dict__`?

**19.** In Python 3.13 free-threaded mode (PEP 703), the GIL is disabled. Does this mean you no longer need to worry about thread safety for shared mutable Python objects? Explain.

**20.** You want to profile a specific integration platform sync endpoint under realistic load without modifying the running container. What command do you run, and what are you looking for in the output?

---

### Answers

??? note "Reveal Answers"

    **1.** A **code object** (`types.CodeType`) is a static, immutable artifact produced by the compiler for each function, class body, or module. It stores the bytecode (`co_code`), constant pool (`co_consts`), variable names (`co_varnames`), and other metadata. It has no execution state. A **frame object** (`PyFrameObject`) is created dynamically each time a function is *called*; it wraps a code object and holds the per-call state: local variable values, the evaluation stack, the instruction pointer, and a reference to the enclosing frame. One code object can have many frame objects active simultaneously (e.g., in recursive calls).

    **2.** `LOAD_FAST` means the variable is a **local** to the function — CPython stores it in a fast-access slot in the frame's local array, indexed by position. `LOAD_GLOBAL` means the variable is not local; CPython looks it up in the function's `__globals__` dict (and then `__builtins__`). Local variable access is faster because it's an array index lookup rather than a dict lookup. If a variable appears as both assigned and referenced in a function, CPython always treats it as local — if you reference it before assignment you get `UnboundLocalError`, not a lookup in the enclosing scope.

    **3.** CPython's **Global Interpreter Lock** (GIL) is a mutex that allows only one thread to execute Python bytecode at a time. Even on a 16-core machine with 16 threads, only one thread runs Python at any given moment. The other threads wait for the GIL. For CPU-bound Python code, threading adds overhead (context switches, GIL contention) but no parallelism. The solution is `multiprocessing` or `concurrent.futures.ProcessPoolExecutor`, where each worker process has its own GIL-independent interpreter, or using a C extension (NumPy, etc.) that releases the GIL during computation.

    **4.** FastAPI's async event loop runs on a **single OS thread**. A synchronous SQLAlchemy call (e.g., `session.execute()` with a psycopg2 driver) **blocks that thread** until the database responds — during which time no other coroutines can run, stalling all concurrent requests. The fix is to either (a) use an async SQLAlchemy engine with an async driver (asyncpg) so the query returns control to the event loop while waiting, or (b) wrap the synchronous call in `await asyncio.to_thread(sync_db_call, ...)` or `loop.run_in_executor(None, sync_db_call)` to run it in a thread pool, releasing the event loop.

    **5.** `sys.getrefcount(x)` takes `x` as an argument, which **creates a temporary reference** to the object for the duration of the function call. So the call itself increments the count by 1. The returned value is therefore always at least 1 higher than the "real" reference count at the time of the call. This is documented behavior and not a bug — be aware of it when interpreting refcount values, especially when debugging memory leaks.

    **6.** If the object is part of a **reference cycle**, its reference count never reaches zero even after all external references are removed. The cyclic garbage collector handles these cycles, but its default collection schedule is not immediate — it runs periodically. If `gc` is disabled (`gc.disable()`), cycles are never collected and `__del__` is never called. Additionally, if the object's `__del__` method itself creates a new reference to the object (a "resurrection"), CPython will not call `__del__` again and may leave the object alive indefinitely. In Python 3.4+, objects with `__del__` in cycles are collected safely, but `__del__` is not guaranteed to run at a deterministic time.

    **7.** The **generational hypothesis** states that most objects die young — they are created and released quickly, while objects that survive many collection cycles tend to live much longer. CPython's `gc` module implements three generations (0, 1, 2). New objects start in generation 0, which is collected most frequently (every ~700 allocations by default). Objects that survive a gen-0 collection are promoted to gen-1, which is collected less often. Gen-1 survivors go to gen-2, collected least frequently. This means the GC spends most of its time on small, recently-allocated objects, which is where most garbage actually lives, reducing overhead compared to always scanning the entire heap.

    **8.** The subclass will have a `__dict__` attribute in addition to inheriting the parent's slots. This is because when you define `__slots__` in a class, it suppresses `__dict__` only for *that class*. A subclass that does not define its own `__slots__` gets a `__dict__` by default, which means instances of the subclass can have arbitrary attributes again. The memory savings from `__slots__` are lost for subclass instances. To preserve the benefit across the hierarchy, every class in the chain must explicitly define `__slots__ = ()` (or their own slots).

    **9.** The **descriptor protocol** is a set of special methods (`__get__`, `__set__`, `__delete__`) that, when defined on a class, change how attribute access on *instances of another class* works. When Python resolves an attribute, it checks if the class has a descriptor for that name and calls the descriptor's method instead of returning the raw value. Three built-in features that use it: (1) **`property`** — uses all three methods to create computed/validated attributes; (2) **`classmethod` and `staticmethod`** — use `__get__` to transform the function before returning it; (3) **functions themselves** — `function.__get__(instance, owner)` returns a bound method, which is why `obj.method()` automatically passes `obj` as `self`.

    **10.** A **metaclass** intercepts class *creation* at the `type.__new__` level — it can inspect, reject, or transform any class in the hierarchy. `__init_subclass__` is a classmethod on the base class that is called when a *subclass* is defined; it's simpler and does not require understanding the metaclass machinery. Prefer `__init_subclass__` for straightforward subclass registration, validation, or attribute injection — it's clearer and composes better with other metaclass-using frameworks (e.g., Pydantic, SQLAlchemy both use metaclasses internally; defining your own metaclass can conflict). Use a metaclass when you need to control `__new__` deeply, change `__prepare__` (the namespace dict used during class body execution), or integrate with a framework that requires it.

    **11.** (1) Python checks `sys.modules["foo"]` — if found, returns the cached module immediately without re-executing anything. (2) Python iterates `sys.meta_path` finders, calling `find_spec("foo", ...)` on each until one returns a `ModuleSpec`. (3) The finder's associated **loader** is used to create a new module object and populate `sys.modules["foo"]` with it (done before execution to handle circular imports). (4) The loader calls `exec_module(module)`, which executes the module's code in the module's `__dict__` namespace. (5) The fully initialized module is returned to the caller. Steps 3–4 happen atomically from the perspective of `sys.modules` to prevent infinite recursion on circular imports.

    **12.** A circular import produces `ImportError: cannot import name 'X' from partially initialized module 'Y'`. To diagnose, add `print(f"importing {__name__}")` at the top of suspected modules and trace the order. The canonical fix without restructuring is the **`TYPE_CHECKING` guard**: wrap imports that are only needed for type annotations in `if TYPE_CHECKING: ...`. Since `TYPE_CHECKING` is `False` at runtime, the import is skipped at execution time. Add `from __future__ import annotations` to enable postponed evaluation so the annotations are stored as strings, not evaluated eagerly. For imports needed at runtime (not just for types), the fix usually requires moving the import inside the function that needs it, or restructuring shared models into a separate `models.py` that neither module depends on.

    **13.** `tottime` is the time spent *inside* the function itself, excluding time spent in functions it called. `cumtime` is the total time from the function's start to its return, including all callees. For finding the actual bottleneck (the "leaf" doing real work), sort by **`tottime`** — high `tottime` means that function itself is slow, not just passing work to something deeper. `cumtime` is useful for tracing call chains (why is the top-level function slow?) but misleading for pinpointing the root cause. A function with `cumtime=5s` and `tottime=0.001s` is just an orchestrator; the real work is happening in its callees.

    **14.** `select()` is the OS-level system call that blocks until a file descriptor (socket, pipe) is readable or writable. Seeing most time in `select()` means the process is **I/O-bound and waiting** — it's spending its time blocked on network or disk I/O, not burning CPU. For the crm-middleware, this is actually *normal and healthy* during light load: the event loop is sleeping in `select()` waiting for the next request or database response. If you see this during a period of high backlog, it suggests the process is waiting on upstream systems (the EHR system API, Postgres) and adding more Python concurrency won't help — investigate query latency or EHR system API response times instead.

    **15.** The **adaptive interpreter** (PEP 659, Python 3.11+) makes CPython self-optimizing. After a code path executes enough times ("warms up"), the interpreter replaces generic opcodes with **specialized** variants tailored to the observed types. For example, `BINARY_OP` operating on two integers may be replaced with `BINARY_OP_ADD_INT`, which skips type-dispatch overhead. In Python 3.13, the specialization is more aggressive and covers more opcode families. When you use `dis.dis()`, you may see the specialized opcode names. Importantly, specializations are per-code-object and can be de-optimized if the types change (e.g., first calling with `int, int` then with `float, int`), falling back to the generic opcode. This is transparent to user code.

    **16.** Start by confirming the leak with external metrics: watch the pod's RSS memory in Datadog over time — a leak shows steady growth without plateaus. Inside the pod, enable `tracemalloc` at startup (`tracemalloc.start(nframe=10)`) and expose a debug endpoint that calls `tracemalloc.take_snapshot()` and returns the top allocation sites. Take two snapshots separated by a period of traffic and compare with `snapshot2.compare_to(snapshot1, "lineno")` — allocations that grow between snapshots are suspects. Then use `objgraph.show_growth()` and `objgraph.show_backrefs()` to visualize which object types are accumulating and what is holding references to them. Common culprits in FastAPI services: Pydantic model instances captured in closures, event listeners that register callbacks without deregistering, or global caches without eviction.

    **17.** SQLAlchemy ORM objects contain **backreferences**: a `Session` references its managed objects, and those objects reference their `Session`, their mapper, and potentially each other through relationships. These create reference cycles that reference counting cannot collect. With `gc.disable()`, these cycles are never cleaned up, and memory grows without bound as each request creates new ORM instances. The cyclic GC is specifically designed to handle this pattern cheaply. Disabling it in a long-running service that uses SQLAlchemy is essentially scheduling a memory leak. If you want to reduce GC pause overhead, tune `gc.set_threshold()` to collect less frequently, but do not disable it entirely.

    **18.** A **data descriptor** defines both `__get__` and `__set__` (and/or `__delete__`). A **non-data descriptor** defines only `__get__`. The distinction matters for attribute lookup precedence: when Python resolves `obj.attr`, it first checks the *class* for a **data descriptor** (data descriptors win over instance `__dict__`). If none, it checks the **instance `__dict__`**. If not found there, it falls back to non-data descriptors and class attributes. `property` is a data descriptor (it defines `__set__` even if only to raise `AttributeError`), so it always takes precedence. Functions are non-data descriptors (only `__get__`), so an instance attribute with the same name as a method will shadow the method.

    **19.** No. The GIL was implicitly serializing all Python bytecode execution, which incidentally prevented most data races on Python objects. Without the GIL, **two threads can truly run Python simultaneously**, meaning concurrent reads and writes to a shared list, dict, or custom object can now produce data corruption without explicit locks. In 3.13 free-threaded mode, CPython adds fine-grained per-object locks internally for built-in types (lists, dicts), but user-defined classes sharing mutable state across threads still require `threading.Lock()` or `threading.RLock()`. Think of the free-threaded build as equivalent to writing multithreaded C — you gain CPU parallelism, but you are now responsible for thread safety everywhere shared state exists.

    **20.** Attach py-spy to the running uvicorn process inside the pod: `kubectl exec -it <pod> -- py-spy record -o /tmp/profile.svg --pid $(pgrep -f uvicorn) --duration 60`. Then `kubectl cp <pod>:/tmp/profile.svg ./profile.svg` and open the SVG in a browser. In the flamegraph, look for **wide horizontal bars** in non-I/O frames — these represent CPU time spent in Python code rather than waiting. Specifically watch for wide bands in `pydantic._internal`, SQLAlchemy's `_execute_crud`, or your own serialization logic. Narrow tall stacks with wide bases in `select` or `recv` are normal I/O waiting. If the widest frame is your sync transformation logic, that's the optimization target; if it's Postgres communication, tune your queries or add connection pooling.

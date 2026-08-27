---
tags: [python, concurrency, asyncio, threading, multiprocessing, performance, computer-science]
created: 2026-07-08
aliases: [Concurrency, Threads vs Processes, asyncio, Event Loop, CPU-Bound vs IO-Bound]
---

# Concurrency, Threads, Processes & Async Programming — Deep Dive

## 1. Concurrency vs Parallelism (Foundational Distinction)

> [!note] The Core Difference
> - **Concurrency** = dealing with **multiple tasks at once** — structuring a program to make progress on several tasks by interleaving them (not necessarily simultaneously).
> - **Parallelism** = actually **executing multiple tasks at the exact same time**, physically, using multiple CPU cores.

> [!tip] Analogy
> A single chef juggling three dishes — stirring one pot, chopping while another simmers, checking the oven — is **concurrency** (one "worker," multiple tasks interleaved). Three chefs each cooking a separate dish simultaneously is **parallelism** (multiple workers, truly simultaneous).

```
Concurrency (one core, interleaved):
Task A: ---   ---   ---
Task B:    ---   ---   ---
           (single core switches between them rapidly)

Parallelism (multiple cores, simultaneous):
Task A: ------------------
Task B: ------------------
        (running literally at the same instant, on different cores)
```

> [!info] Key Insight
> Concurrency is about **structure** (how you design the program). Parallelism is about **execution** (how the hardware runs it). You can have concurrency without parallelism (single-core multitasking), and you generally need concurrency-friendly design to *achieve* parallelism.

---

## 2. CPU-Bound vs I/O-Bound Tasks

This distinction determines **which concurrency tool** you should reach for — arguably the most important practical decision in this entire topic.

### 2.1 CPU-Bound Tasks
The bottleneck is **processor computation speed** — the program spends most of its time doing actual calculations, and would run faster with a faster CPU.

**Examples:**
- Mathematical computations (matrix multiplication, numerical simulations)
- Image/video processing and encoding
- Data compression
- Cryptographic hashing
- Sorting/searching massive datasets in memory

```python
def cpu_bound_task(n):
    # Pure computation -- no waiting, CPU is busy the entire time
    total = 0
    for i in range(n):
        total += i ** 2
    return total
```

### 2.2 I/O-Bound Tasks
The bottleneck is **waiting** — for disk reads/writes, network responses, database queries, or user input. The CPU is mostly **idle**, waiting for something external to finish.

**Examples:**
- HTTP requests / API calls
- Database queries
- Reading/writing files
- Network sockets
- Waiting on user input

```python
import time

def io_bound_task():
    # Simulates waiting on a network call -- CPU does nothing during this time
    time.sleep(2)
    return "data received"
```

### 2.3 Why This Distinction Matters

| Task Type | Bottleneck                    | Best Solved By               | Why                                                                                       |
| --------- | ----------------------------- | ---------------------------- | ----------------------------------------------------------------------------------------- |
| CPU-bound | Processor speed               | **Multiprocessing**          | Needs genuine parallel execution across cores; threads are blocked by the GIL (see below) |
| I/O-bound | Waiting on external resources | **Threading** or **asyncio** | CPU is idle during waits anyway — can be released to do other work in the meantime        |

> [!warning] The #1 Concurrency Mistake
> Using `threading` for CPU-bound work in Python **doesn't speed things up** — due to the GIL (next section), threads can't run Python bytecode in true parallel. Use `multiprocessing` for CPU-bound tasks, and `threading`/`asyncio` for I/O-bound tasks.

---

## 3. The Global Interpreter Lock (GIL)

> [!note] What is the GIL?
> The **GIL** is a mutex in CPython (the standard Python implementation) that allows only **one thread to execute Python bytecode at a time**, even on a multi-core machine.

### Why Does the GIL Exist?
- Simplifies memory management (CPython uses reference counting for garbage collection, which isn't thread-safe without a lock)
- Makes single-threaded code fast and C-extension integration simpler
- Historical design decision from early CPython (1990s), still present today

### GIL's Practical Impact

```python
import threading
import time

def cpu_task():
    total = 0
    for i in range(50_000_000):
        total += i
    return total

start = time.time()
t1 = threading.Thread(target=cpu_task)
t2 = threading.Thread(target=cpu_task)
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Threaded: {time.time() - start:.2f}s")
# Often TAKES LONGER (or same time) as running both sequentially,
# because only one thread executes Python bytecode at a time.
```

> [!warning] GIL Only Affects CPU-Bound Python Bytecode
> The GIL is **released** during I/O operations (file reads, network calls, `time.sleep()`) and by many C-extension libraries (NumPy, for instance, releases the GIL during heavy array computations). This is why **threading still helps for I/O-bound tasks** despite the GIL.

### GIL Summary Table

| Aspect | Detail |
|---|---|
| What it is | A lock ensuring only one thread runs Python bytecode at a time |
| Affects | CPython specifically (not Jython, IronPython, or PyPy's alternate modes) |
| Impact on CPU-bound threading | No real speedup — threads compete for the same lock |
| Impact on I/O-bound threading | Minimal — GIL is released during I/O waits |
| How to bypass for CPU-bound work | Use `multiprocessing` (separate processes = separate GILs) |
| Future | Python 3.13+ has an experimental **free-threaded build** (PEP 703) that can disable the GIL |

---

## 3A. Reference Counting & Garbage Collection (Why the GIL Exists)

> [!note] Why This Belongs Here
> The GIL isn't an arbitrary design choice — it exists largely **because of how CPython manages memory**. Understanding reference counting and the garbage collector explains *why* CPython needs a global lock in the first place, and is a very common follow-up interview question after GIL discussions.

### 3A.1 Reference Counting — CPython's Primary Memory Management Strategy

> [!note] Definition
> Every object in CPython has an internal **reference count** — a counter tracking how many references (variables, container elements, etc.) currently point to it. When the count drops to **zero**, the object is **immediately deallocated**.

```python
import sys

a = []
print(sys.getrefcount(a))   # 2 -- one for 'a', one for the temporary arg to getrefcount()

b = a           # new reference to the same object
print(sys.getrefcount(a))   # 3 -- 'a', 'b', and the temp arg

del b
print(sys.getrefcount(a))   # back to 2
```

### How Reference Counts Change

| Action | Effect on Refcount |
|---|---|
| Assigning an object to a new variable | +1 |
| Passing an object as a function argument | +1 (temporarily) |
| Adding an object to a list/dict/set | +1 |
| `del` a variable / variable goes out of scope | -1 |
| Reassigning a variable to something else | -1 on old object |
| Container holding the object is deleted | -1 |

```python
def demo():
    x = [1, 2, 3]     # refcount = 1
    y = x               # refcount = 2
    z = [x]              # refcount = 3 (stored inside another list)
    return               # x, y, z go out of scope -> refcount drops to 0 -> deallocated immediately
```

> [!success] Key Advantage
> Reference counting deallocates objects **immediately and deterministically** the instant their refcount hits zero — unlike generational garbage collectors in languages like Java, there's no unpredictable "GC pause" for most objects. This is why file handles, locks, etc. often get cleaned up promptly when they go out of scope in CPython (though `with` statements should still be used for guaranteed, immediate cleanup — see [[Exception Handling (Python)]]).

### Why Reference Counting Needs the GIL

> [!warning] The Core Problem
> Incrementing/decrementing a refcount is **not atomic** at the machine-instruction level — it's a read-modify-write operation. If two threads modify the same object's refcount **simultaneously without a lock**, you can get a **race condition** that corrupts the count, leading to either:
> - **Premature deallocation** (refcount hits 0 while something still uses the object → crash / memory corruption)
> - **Memory leaks** (refcount never reaches 0 → object never freed)

This is the **primary historical reason CPython uses the GIL**: rather than adding fine-grained locks to every object (which is slow and complex — as CPython's experimental free-threaded build has to do), CPython uses **one big lock** so only one thread touches any Python object (and its refcount) at a time.

```python
# Conceptually, WITHOUT a GIL, this would be a race condition:
# Thread A: reads refcount (5), about to write 6
# Thread B: reads refcount (5), about to write 6
# Both write 6 -- but the "true" count should have been 7!
# Now the object might be freed while Thread B still holds a reference to it.
```

### 3A.2 The Cyclic Garbage Collector (`gc` module)

> [!warning] Reference Counting Alone Can't Handle Cycles
> Reference counting fails when objects reference each other in a **cycle** — their refcounts never reach zero even though nothing outside the cycle can reach them.

```python
class Node:
    def __init__(self):
        self.other = None

a = Node()
b = Node()
a.other = b   # a references b
b.other = a   # b references a -- circular reference!

del a
del b
# Even after both variables are deleted, a and b still reference EACH OTHER
# Their refcounts never hit 0 through reference counting alone!
```

This is where Python's **secondary garbage collector** (the `gc` module) comes in — a **generational, cycle-detecting** collector that runs periodically to find and clean up these reference cycles.

### How the Generational GC Works

```
Generation 0 (youngest) → Generation 1 → Generation 2 (oldest)
     Most objects            Survivors         Long-lived objects
     collected here         from Gen 0         collected least often
```

> [!note] Generational Hypothesis
> Most objects die young. So the GC scans **Generation 0 frequently** (cheap, catches most garbage quickly), and only occasionally promotes surviving objects to Generation 1, then Generation 2 (scanned much less often, since long-lived objects are statistically likely to stay alive).

```python
import gc

print(gc.get_threshold())   # e.g. (700, 10, 10)
# Gen 0 collected every 700 allocations (net of deallocations)
# Gen 1 collected every 10 Gen-0 collections
# Gen 2 collected every 10 Gen-1 collections

gc.collect()   # manually force a full collection (returns number of unreachable objects found)

gc.disable()    # turn off automatic cyclic GC (refcounting still works!)
gc.enable()      # turn it back on
```

### How Cycle Detection Actually Works (Conceptually)
The GC identifies objects only reachable via cycles by:
1. Tracking all "container" objects (lists, dicts, class instances, etc. — things that can hold references to other objects)
2. Temporarily simulating "what if we removed all references coming from *outside* this group of objects?"
3. If an object's refcount would still be non-zero from **external** references, it survives
4. Anything left with an effective refcount of zero (only referenced from within the unreachable cycle) is garbage-collected

> [!info] `__del__` and Cycles — A Classic Gotcha (Python < 3.4)
> In old Python versions, objects in a reference cycle with a custom `__del__` method **couldn't be collected at all** (unsafe to guess destruction order). Since Python 3.4 (PEP 442), the GC can safely handle this in most cases — but it's still best practice to **avoid `__del__` for critical cleanup** and use context managers instead.

### 3A.3 Reference Counting vs Generational GC — Comparison

| Aspect | Reference Counting | Generational GC (`gc` module) |
|---|---|---|
| Handles | Simple ownership (most objects) | Reference **cycles** specifically |
| Timing | Immediate, deterministic (refcount hits 0) | Periodic, runs at generation thresholds |
| Performance cost | Small overhead on every assignment/deletion | Occasional pause to scan generations |
| Can be disabled? | No (core to CPython's memory model) | Yes (`gc.disable()`) — refcounting still works without it |
| Relationship to GIL | **Primary reason the GIL exists** | Runs under the GIL too, but is a separate mechanism |

### 3A.4 Practical Implications for Concurrency

> [!success] Why This Matters for Threads vs Processes
> - **Threads** share the same object graph and refcounts → the GIL is what makes refcount updates safe. This is a big part of why threads can't achieve true CPU parallelism in CPython.
> - **Processes** each have their **own separate memory space and their own refcounts/GC** → no shared refcount contention, which is part of why `multiprocessing` achieves genuine parallelism.
> - **`asyncio`** runs single-threaded, so refcounts are never touched concurrently by design → no GIL contention concern at all, by construction.

> [!tip] Interview-Ready Summary
> *"CPython uses reference counting as its primary memory management strategy, which deallocates objects immediately and deterministically when their count hits zero. Because refcount updates aren't atomic, CPython uses the GIL to make them thread-safe rather than adding per-object locks. Reference counting alone can't detect reference cycles, so CPython supplements it with a generational, cycle-detecting garbage collector in the `gc` module that runs periodically across three generations."*

---

## 4. Threads (the `threading` module)

### What is a Thread?
A **thread** is the smallest unit of execution within a process. Multiple threads within the same process **share the same memory space** (variables, objects) but have their own call stack.

```python
import threading
import time

def worker(name, delay):
    print(f"{name} starting")
    time.sleep(delay)
    print(f"{name} finished")

t1 = threading.Thread(target=worker, args=("Thread-1", 2))
t2 = threading.Thread(target=worker, args=("Thread-2", 2))

t1.start()
t2.start()
t1.join()   # wait for t1 to finish
t2.join()   # wait for t2 to finish

print("Both threads done")
# Total time ~2s (not 4s) -- they run concurrently, waiting overlaps
```

### Thread Pools — `concurrent.futures.ThreadPoolExecutor`
A cleaner, higher-level way to manage multiple threads without manually creating/joining each one.

```python
from concurrent.futures import ThreadPoolExecutor
import requests

urls = ["https://example.com"] * 5

def fetch(url):
    return requests.get(url).status_code

with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch, urls))

print(results)   # e.g. [200, 200, 200, 200, 200] -- fetched concurrently
```

### Race Conditions & Thread Safety

> [!warning] Race Condition
> When multiple threads access/modify **shared state** without synchronization, the outcome depends on unpredictable timing — leading to inconsistent, hard-to-reproduce bugs.

```python
counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1   # NOT atomic! Read-modify-write can be interrupted mid-operation

threads = [threading.Thread(target=increment) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()

print(counter)   # Expected 200000, but often LESS due to race condition
```

### Fixing Race Conditions — `threading.Lock`
```python
counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100000):
        with lock:          # only one thread can hold the lock at a time
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()

print(counter)   # Correctly 200000
```

### Other Synchronization Primitives
| Primitive | Purpose |
|---|---|
| `threading.Lock` | Basic mutual exclusion — one thread at a time |
| `threading.RLock` | Reentrant lock — same thread can acquire it multiple times |
| `threading.Semaphore` | Allows up to N threads to access a resource simultaneously |
| `threading.Event` | Simple flag for signaling between threads |
| `threading.Condition` | Allows threads to wait for a condition to become true |
| `queue.Queue` | Thread-safe FIFO queue — the standard way to pass data between threads safely |

### Threads Summary Table
| Aspect | Detail |
|---|---|
| Memory | Shared across threads within the same process |
| Creation cost | Lightweight (cheaper than processes) |
| Best for | I/O-bound tasks (network calls, file I/O) |
| GIL impact | Blocks true CPU parallelism |
| Risk | Race conditions, deadlocks, need explicit synchronization |

---

## 5. Processes (the `multiprocessing` module)

### What is a Process?
A **process** is an independent instance of a running program with its **own memory space**, own Python interpreter, and own GIL. Processes don't share memory by default — communication requires explicit mechanisms.

```python
import multiprocessing
import time

def cpu_task(n):
    total = 0
    for i in range(n):
        total += i ** 2
    return total

if __name__ == "__main__":
    start = time.time()
    p1 = multiprocessing.Process(target=cpu_task, args=(50_000_000,))
    p2 = multiprocessing.Process(target=cpu_task, args=(50_000_000,))
    p1.start(); p2.start()
    p1.join(); p2.join()
    print(f"Multiprocessing: {time.time() - start:.2f}s")
    # Genuinely FASTER than threading for this -- true parallel execution across cores
```

> [!warning] `if __name__ == "__main__":` Guard Required
> On Windows (and when using the `spawn` start method), child processes re-import the main module. Without this guard, you can trigger **infinite recursive process creation**. Always wrap multiprocessing entry points in this guard.

### Process Pools — `multiprocessing.Pool` / `ProcessPoolExecutor`
```python
from concurrent.futures import ProcessPoolExecutor

def square(n):
    return n * n

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(square, range(10)))
    print(results)   # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

### Inter-Process Communication (IPC)
Since processes don't share memory, you need explicit mechanisms to exchange data:

```python
from multiprocessing import Process, Queue

def worker(q):
    q.put("result from child process")

if __name__ == "__main__":
    q = Queue()
    p = Process(target=worker, args=(q,))
    p.start()
    p.join()
    print(q.get())   # "result from child process"
```

| IPC Mechanism | Use Case |
|---|---|
| `multiprocessing.Queue` | FIFO queue, safe for passing data between processes |
| `multiprocessing.Pipe` | Two-way communication channel between exactly 2 processes |
| `multiprocessing.Value` / `Array` | Shared memory for simple types/arrays across processes |
| `multiprocessing.Manager` | Provides shared, proxy-managed objects (dict, list, etc.) |

> [!info] Serialization Overhead
> Data passed between processes must be **pickled** (serialized) and **unpickled** on the other end — this adds overhead absent in threading (shared memory, no serialization needed).

### Threads vs Processes — Full Comparison

| Aspect | Threads | Processes |
|---|---|---|
| Memory | Shared | Isolated (separate memory space) |
| Creation overhead | Low | High (heavier — full interpreter copy) |
| Communication | Direct (shared variables) — needs locks | IPC required (Queue, Pipe) — needs serialization |
| GIL impact | Limited by GIL for CPU-bound work | Each process has its own GIL — true parallelism |
| Best for | I/O-bound tasks | CPU-bound tasks |
| Crash isolation | One thread crashing can affect the whole process | One process crashing doesn't affect others |
| Scalability | Cheap to spawn many | Expensive — limited by CPU cores/memory |

---

## 6. Async Programming — `asyncio`, `async`/`await`

### What Problem Does Async Solve?

> [!note] Core Idea
> Async programming achieves **concurrency within a single thread**, using **cooperative multitasking** — a task voluntarily yields control (usually while waiting on I/O) so another task can run, instead of the OS switching threads for you (**preemptive multitasking**, as with `threading`).

> [!tip] Why Async Instead of Threads for I/O?
> - No GIL contention, no race conditions, no locks needed (single-threaded)
> - Much lower memory/overhead per "task" compared to OS threads (can run thousands of concurrent tasks cheaply)
> - Explicit control over where a task can be "paused" (`await` points) makes reasoning about concurrency easier

### `async def` and `await`

```python
import asyncio

async def fetch_data(name, delay):
    print(f"{name}: starting")
    await asyncio.sleep(delay)   # yields control -- doesn't block the thread
    print(f"{name}: done")
    return f"{name} result"

async def main():
    result = await fetch_data("Task-1", 2)
    print(result)

asyncio.run(main())
```

> [!info] `async def` Creates a Coroutine
> Calling `fetch_data("Task-1", 2)` does **not** run the function immediately — it returns a **coroutine object**. The function body only executes when the coroutine is **awaited** or scheduled on the event loop.

```python
coro = fetch_data("Task-1", 2)
print(coro)          # <coroutine object fetch_data at 0x...>
# Nothing has run yet! Must await it or wrap in asyncio.run()/create_task()
```

### Running Multiple Coroutines Concurrently

```python
import asyncio

async def fetch_data(name, delay):
    print(f"{name}: starting")
    await asyncio.sleep(delay)
    print(f"{name}: done")
    return f"{name} result"

async def main():
    # Sequential (slow) -- total ~6s
    # await fetch_data("A", 2)
    # await fetch_data("B", 2)
    # await fetch_data("C", 2)

    # Concurrent (fast) -- total ~2s, all run "simultaneously"
    results = await asyncio.gather(
        fetch_data("A", 2),
        fetch_data("B", 2),
        fetch_data("C", 2),
    )
    print(results)

asyncio.run(main())
```

> [!warning] `await` Alone Doesn't Give You Concurrency
> ```python
> await fetch_data("A", 2)   # runs, THEN moves to next line
> await fetch_data("B", 2)   # only starts after A fully finishes -- sequential!
> ```
> To actually run coroutines **concurrently**, you need `asyncio.gather()`, `asyncio.create_task()`, or `asyncio.TaskGroup` (3.11+) — simply chaining `await` calls runs them one after another.

### `asyncio.create_task()` — Scheduling Without Waiting Immediately

```python
async def main():
    task1 = asyncio.create_task(fetch_data("A", 2))   # starts running immediately in background
    task2 = asyncio.create_task(fetch_data("B", 2))
    # do other work here while tasks run...
    result1 = await task1   # now wait for completion
    result2 = await task2
```

### `asyncio.TaskGroup` (Python 3.11+) — Modern, Safer Alternative

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch_data("A", 2))
        task2 = tg.create_task(fetch_data("B", 2))
    # all tasks guaranteed complete here; exceptions properly propagated/grouped
    print(task1.result(), task2.result())
```

> [!success] Why `TaskGroup` is Preferred Over `gather()`
> - Automatically cancels sibling tasks if one fails (structured concurrency)
> - Cleaner exception handling via `ExceptionGroup` (see [[Exception Handling (Python)]])
> - Guarantees all tasks are awaited/cleaned up, avoiding "dangling task" warnings

---

## 7. The Event Loop

### What is the Event Loop?

> [!note] Definition
> The **event loop** is the core engine of `asyncio` — a single-threaded scheduler that continuously: (1) runs ready coroutines until they hit an `await`, (2) parks them, (3) checks which awaited operations have completed, and (4) resumes the appropriate coroutine. It repeats this cycle indefinitely.

```
Event Loop Cycle (conceptual):
┌─────────────────────────────────────────┐
│  1. Pick a ready task from the queue      │
│  2. Run it until it hits an `await`        │
│  3. If awaiting I/O, register a callback    │
│     and move to the next ready task          │
│  4. When I/O completes, mark task as ready    │
│  5. Repeat until all tasks are done             │
└─────────────────────────────────────────┘
```

### How `asyncio.run()` Works
```python
asyncio.run(main())
```
This single call:
1. Creates a new event loop
2. Runs the `main()` coroutine until it completes
3. Closes the event loop afterward
4. Is the **recommended entry point** — manually managing loops (`get_event_loop()`, `loop.run_until_complete()`) is now discouraged for typical application code.

### Event Loop and Blocking Calls

> [!warning] Never Use Blocking Calls Inside Async Code
> ```python
> async def bad_example():
>     time.sleep(2)   # BLOCKS THE ENTIRE EVENT LOOP -- freezes ALL concurrent tasks!
> ```
> `time.sleep()` is a **synchronous, blocking** call — it doesn't yield control back to the event loop, so nothing else can run during those 2 seconds, defeating the entire purpose of async.
>
> **Fix:** always use the async-native equivalent:
> ```python
> async def good_example():
>     await asyncio.sleep(2)   # yields control -- other tasks can run meanwhile
> ```

### Running Blocking/CPU-Bound Code Inside Async — `run_in_executor`
If you must call blocking code (e.g., a synchronous library, or genuinely CPU-heavy work) from async code, offload it to a thread or process pool so it doesn't block the event loop:

```python
import asyncio
import time

def blocking_io():
    time.sleep(2)   # simulate a blocking call
    return "done"

async def main():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, blocking_io)  # runs in a thread pool
    print(result)

asyncio.run(main())
```

---

## 8. Async Ecosystem — Related Concepts

### 8.1 Async Generators & Async Comprehensions
```python
async def async_range(n):
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async def main():
    async for value in async_range(5):
        print(value)

    # Async comprehension
    results = [x async for x in async_range(5)]
```

### 8.2 `async with` — Async Context Managers
```python
class AsyncResource:
    async def __aenter__(self):
        print("Acquiring resource asynchronously")
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Releasing resource asynchronously")

async def main():
    async with AsyncResource() as res:
        print("Using resource")
```

### 8.3 Common Real-World Async Libraries
| Library | Purpose |
|---|---|
| `aiohttp` | Async HTTP client/server |
| `httpx` | HTTP client with both sync and async support |
| `asyncpg` | Async PostgreSQL driver |
| `motor` | Async MongoDB driver |
| `FastAPI` | Async-first web framework built on `asyncio` (via Starlette) |
| `uvicorn` | ASGI server that runs async web apps |

---

## 9. Comparing All Three Approaches

| Aspect | Threading | Multiprocessing | Asyncio |
|---|---|---|---|
| Concurrency model | Preemptive (OS-scheduled) | Preemptive, true parallel | Cooperative (single-threaded) |
| Best for | I/O-bound | CPU-bound | I/O-bound (high concurrency, many tasks) |
| GIL affected? | Yes | No (separate processes) | Yes, but irrelevant (single-threaded by design) |
| Memory | Shared | Isolated | Shared (single thread) |
| Overhead per task | Moderate (OS thread) | High (full process) | Very low (lightweight coroutine) |
| Max practical concurrency | ~hundreds of threads | ~number of CPU cores | Tens of thousands of coroutines |
| Data sharing complexity | Needs locks | Needs IPC/serialization | None needed (single thread, no race conditions) |
| Code complexity | Moderate (locks, race conditions) | Moderate (serialization, IPC) | Requires `async`/`await` throughout the call chain ("async all the way down") |

> [!warning] "Async All the Way Down"
> You cannot `await` inside a regular synchronous function, and calling a blocking synchronous function inside an async function blocks the entire event loop. Once you go async, ideally your **entire I/O call chain** should be async-compatible (or offloaded via `run_in_executor`).

---

## 10. Decision Framework — Which Tool to Use?

```
Is the task CPU-bound (heavy computation)?
├── YES → use multiprocessing (or offload to C extensions like NumPy that release the GIL)
└── NO (I/O-bound) → 
        Is the number of concurrent tasks very large (100s-1000s)?
        ├── YES → use asyncio (lower overhead, scales better)
        └── NO / mixing with blocking libraries → use threading (simpler to integrate with existing sync code)
```

> [!tip] Practical Rules of Thumb
> - **Web scraping / API calls (many, I/O-bound)** → `asyncio` + `aiohttp`/`httpx`
> - **Image/video processing, ML training, number crunching** → `multiprocessing` or NumPy/vectorized operations
> - **GUI apps needing responsiveness while doing I/O** → `threading` (many GUI frameworks aren't async-native)
> - **Mixing a legacy synchronous codebase with a bit of concurrency** → `ThreadPoolExecutor` is often the path of least resistance
> - **Combining CPU-bound and I/O-bound work** → often combine `asyncio` (for I/O) with `run_in_executor` backed by a `ProcessPoolExecutor` (for CPU-heavy parts)

---

## 11. Performance Concepts

### 11.1 Amdahl's Law
Describes the theoretical speedup limit when parallelizing part of a program.

```
Speedup = 1 / ((1 - P) + P/N)

P = proportion of the program that can be parallelized
N = number of processors/cores
```

> [!info] Key Takeaway
> If only 50% of your program can be parallelized (`P = 0.5`), even with infinite cores, you can **never speed up the whole program by more than 2x** — the serial (non-parallelizable) portion becomes the bottleneck.

### 11.2 Context Switching Overhead
Switching between threads/processes has a real cost (saving/restoring state, cache invalidation). Too many threads/processes can lead to **diminishing returns or even slowdowns** due to excessive context switching — this is why asyncio's lightweight coroutines scale better for very high concurrency counts.

### 11.3 Throughput vs Latency
| Term | Meaning |
|---|---|
| **Latency** | Time to complete a single task/request |
| **Throughput** | Number of tasks/requests completed per unit time |

> [!tip]
> Async and concurrency primarily improve **throughput** for I/O-bound workloads (handle many requests while waiting on others) — they don't necessarily reduce the **latency** of any single request.

### 11.4 Deadlocks, Livelocks, and Starvation
| Problem | Description |
|---|---|
| **Deadlock** | Two or more threads/processes wait on each other indefinitely, each holding a resource the other needs |
| **Livelock** | Threads keep changing state in response to each other but make no actual progress |
| **Starvation** | A thread/process is perpetually denied access to a resource because other threads keep "cutting in line" |

```python
# Classic deadlock example
lock_a = threading.Lock()
lock_b = threading.Lock()

def thread_1():
    with lock_a:
        time.sleep(0.1)
        with lock_b:   # waits for lock_b, held by thread_2
            pass

def thread_2():
    with lock_b:
        time.sleep(0.1)
        with lock_a:   # waits for lock_a, held by thread_1 -- DEADLOCK
            pass
```
> [!tip] Avoiding Deadlocks
> Always acquire multiple locks in a **consistent, global order** across all threads to prevent circular waiting.

---

## 12. Best Practices ✅

> [!success] Do's
> - Match the tool to the task: `multiprocessing` for CPU-bound, `asyncio`/`threading` for I/O-bound
> - Use `concurrent.futures` (`ThreadPoolExecutor`/`ProcessPoolExecutor`) instead of manually managing raw threads/processes when possible
> - Always guard `multiprocessing` entry points with `if __name__ == "__main__":`
> - Use `asyncio.TaskGroup` (3.11+) over `gather()` for structured, safer concurrency
> - Use locks/queues for any shared mutable state accessed by multiple threads
> - Offload blocking calls inside async code via `run_in_executor`

> [!failure] Don'ts
> - Don't use threading expecting a CPU-bound speedup — the GIL prevents it
> - Don't call blocking functions (`time.sleep`, synchronous `requests.get`) directly inside `async def` functions
> - Don't share mutable state across threads without synchronization (race conditions)
> - Don't spawn unbounded numbers of threads/processes — use a pool with a sensible `max_workers`
> - Don't assume `await` automatically implies concurrency — sequential `await` calls still run one after another
> - Don't acquire multiple locks in inconsistent order across different threads (deadlock risk)

---

## 13. Quick Revision Summary

| Concept | One-Line Definition |
|---|---|
| **Concurrency** | Structuring a program to handle multiple tasks by interleaving (not necessarily simultaneous) |
| **Parallelism** | Truly simultaneous execution across multiple CPU cores |
| **CPU-bound** | Bottlenecked by computation → solved with `multiprocessing` |
| **I/O-bound** | Bottlenecked by waiting → solved with `threading`/`asyncio` |
| **GIL** | CPython lock allowing only one thread to run Python bytecode at a time |
| **Thread** | Lightweight unit of execution sharing memory with siblings; needs locks for safety |
| **Process** | Independent execution unit with isolated memory; true parallelism, needs IPC |
| **Coroutine** | A function (`async def`) that can pause (`await`) and resume, run on the event loop |
| **Event Loop** | Single-threaded scheduler that runs coroutines cooperatively |
| **`asyncio.gather`/`TaskGroup`** | Run multiple coroutines concurrently within the event loop |
| **Reference Counting** | CPython's primary memory strategy — object freed immediately when refcount hits 0 |
| **Generational GC (`gc`)** | Secondary collector that detects and cleans up reference **cycles** refcounting can't catch |

**Golden Rules:**
1. CPU-bound → `multiprocessing` (bypasses the GIL via separate processes)
2. I/O-bound with few tasks / sync-library integration → `threading`
3. I/O-bound with massive concurrency → `asyncio`
4. Never block the event loop with synchronous/blocking calls
5. `await` alone ≠ concurrency — you need `gather`/`create_task`/`TaskGroup` to actually run things at once
6. Reference counting is why CPython needs the GIL — refcount updates aren't atomic, so one global lock keeps them thread-safe
7. Reference cycles need the generational `gc` module, since plain refcounting can never bring a cycle's count to zero

---

## Related Notes
- [[Exception Handling (Python)]]
- [[self, cls, Static Methods and __new__]]
- [[14 PEP 8 Style Guide]]
- [[15 Memory Management in Python]]
- [[Design Patterns in Python]]
- [[Reference Counting and Garbage Collection Deep Dive]]

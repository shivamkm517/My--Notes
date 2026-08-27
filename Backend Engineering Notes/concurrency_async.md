# Concurrency, Parallelism & Async Programming — Complete Notes

---

## 1. Concurrency vs Parallelism (The Foundation)

Before touching threads, processes, or async, get this distinction rock solid — almost everything else builds on it.

|                              | Concurrency                                        | Parallelism                                          |
| ---------------------------- | -------------------------------------------------- | ---------------------------------------------------- |
| **Definition**               | Dealing with multiple tasks at once (structure)    | Doing multiple tasks at the same time (execution)    |
| **Requires multiple cores?** | No                                                 | Yes                                                  |
| **Analogy**                  | One chef juggling 3 dishes, switching between them | 3 chefs, each cooking their own dish, simultaneously |
| **Goal**                     | Better structure / responsiveness                  | Better raw speed / throughput                        |

> **Key line to remember:** *Concurrency is about structure, parallelism is about execution.* You can have concurrency on a single core (via task-switching), but you can't have true parallelism without multiple cores.

**Real world example:**
- A single cashier (1 core) serving 5 customers by quickly switching between taking orders, printing receipts, and giving change → **concurrency**.
- 5 cashiers (5 cores) each serving their own customer at the same time → **parallelism**.

---

## 2. CPU-Bound vs IO-Bound Tasks

This classification decides *which concurrency tool you should even reach for*. Get this wrong and your "optimization" makes things slower.

### CPU-Bound Tasks
The bottleneck is the **processor** doing heavy computation. The CPU is constantly busy; adding more "waiting" tricks doesn't help.

**Examples:**
- Image/video processing (resizing, encoding)
- Machine learning model training / inference
- Mathematical simulations, cryptography, hashing (bcrypt, password cracking)
- Compiling code
- Data compression (zip/unzip large files)

### IO-Bound Tasks
The bottleneck is **waiting** — for a network response, disk read/write, or database query. The CPU is mostly idle, twiddling its thumbs while something external finishes.

**Examples:**
- Making an API call (waiting for the server to respond)
- Reading/writing files from disk
- Database queries
- Web scraping (fetching hundreds of URLs)
- Chat applications waiting on socket messages

### Why the distinction matters

| Task Type | Bottleneck | Best Tool |
|---|---|---|
| CPU-bound | Processor computation | **Multiprocessing** (true parallel cores) |
| IO-bound | Waiting on external resource | **Threading or Async/asyncio** |

> **Rule of thumb:** If your program is "busy waiting" → go async/threaded. If your program is "busy calculating" → go multiprocessing.

**Real world analogy:** Imagine you're cooking (CPU-bound, you must physically chop/stir) vs waiting for delivery food (IO-bound, you're just waiting for the doorbell). Hiring more cooks (processes) helps the first case. For the second, you don't need more cooks — you just need to not stand at the door; you can do other things while waiting (async).

---

## 3. Threads

A **thread** is the smallest unit of execution within a process. Multiple threads within the same process **share the same memory space**.

### Key characteristics
- Lightweight compared to processes (cheaper to create, less memory overhead)
- Share memory → easy to share data, but risky (race conditions)
- Managed by the OS scheduler
- In **Python**, threads are limited by the **GIL** (Global Interpreter Lock) — see section 3.1

### Pros
- Fast to create/switch (low overhead)
- Easy data sharing between threads (same memory)
- Great for IO-bound tasks

### Cons
- Risk of **race conditions** (two threads modifying shared data at once)
- Need synchronization primitives (locks, semaphores) → adds complexity
- In Python: can't achieve true CPU parallelism due to the GIL

### 3.1 The GIL (Global Interpreter Lock) — Python Specific

The GIL is a mutex in CPython that allows **only one thread to execute Python bytecode at a time**, even on a multi-core machine.

- This means: Python threads **do not** give you CPU parallelism.
- But: while one thread is waiting on IO (like a network call), the GIL is released, so other threads *can* run.
- **Conclusion:** Python threading is great for IO-bound work, useless for CPU-bound work.

**Real world example:** A web scraper downloading 100 web pages using threads. While Thread A is waiting for a server response, the GIL is free, so Thread B can start its own request. Total time drops dramatically compared to doing it one by one.

---

## 4. Processes

A **process** is an independent program in execution, with its **own memory space**, its own Python interpreter, and its own GIL (in Python's case).

### Key characteristics
- Heavier than threads (more memory, slower to spawn)
- No shared memory by default → need explicit **Inter-Process Communication (IPC)** (pipes, queues, shared memory)
- Truly parallel — each process can run on a separate CPU core
- In Python: `multiprocessing` module sidesteps the GIL entirely by using separate interpreter processes

### Pros
- True parallelism — actually uses multiple cores
- No GIL limitation (each process has its own)
- Crash isolation: if one process crashes, others survive

### Cons
- High memory overhead (each process duplicates the interpreter + memory)
- Slower startup/teardown than threads
- Data sharing between processes is expensive (needs serialization via pickling, queues, etc.)

**Real world example:** A video encoding service that splits a video into 4 chunks and encodes each chunk in a separate process, using all 4 CPU cores simultaneously → 4x speedup (roughly). This is a **CPU-bound** task, so processes (not threads) are the correct tool.

### Threads vs Processes — Quick Comparison

| | Threads | Processes |
|---|---|---|
| Memory | Shared | Isolated (separate) |
| Creation cost | Low | High |
| Communication | Easy (shared vars) | Needs IPC (Queue, Pipe) |
| Crash impact | Can bring down whole process | Isolated — one crash doesn't kill others |
| Best for | IO-bound tasks | CPU-bound tasks |
| Python GIL affected? | Yes | No (each process has own GIL) |

---

## 5. Async Programming

Async programming is a **concurrency model within a single thread**. Instead of using OS-level threads/processes to handle multiple tasks, a single thread cooperatively switches between tasks whenever one of them is *waiting* (usually on IO).

### The core idea: Cooperative Multitasking
- In threading, the **OS** decides when to pause/switch a thread (preemptive).
- In async, the **task itself** decides when to give up control — usually at an `await` point, when it's waiting for something.

**Real world analogy:** A single waiter serving multiple tables. Instead of standing at Table 1 waiting for the kitchen to prepare food (blocking), the waiter takes Table 2's order while Table 1's food is cooking, then comes back to Table 1 once it's ready. One person (one thread), many tasks handled concurrently — because the waiter is smart about *when to switch*.

### Why async instead of threads for IO-bound work?
- No OS thread-switching overhead
- No race conditions from shared memory (single thread, no true simultaneous access)
- Can scale to **thousands** of concurrent operations cheaply (imagine 10,000 threads — each has ~8MB stack overhead; 10,000 async tasks cost almost nothing in comparison)

**Real world example:** A chat server handling 50,000 simultaneous open connections. Spinning up 50,000 threads would crash the server (memory + context-switching cost). But 50,000 async coroutines, each just waiting for a message, is totally feasible — this is exactly how frameworks like Node.js, and Python's `asyncio`-based servers, handle massive concurrent connections.

---

## 6. asyncio (Python's Async Framework)

`asyncio` is Python's standard library for writing **single-threaded concurrent code** using coroutines.

### Core building blocks

| Concept | What it is |
|---|---|
| **Coroutine** | A function defined with `async def`, which can be paused/resumed |
| **Event Loop** | The engine that runs and schedules coroutines |
| **Task** | A coroutine wrapped and scheduled to run on the event loop (`asyncio.create_task()`) |
| **Future** | A placeholder for a result that will be available later |

### Basic example

```python
import asyncio

async def fetch_data(name, delay):
    print(f"Start fetching {name}")
    await asyncio.sleep(delay)   # simulates IO wait (like a network call)
    print(f"Done fetching {name}")
    return f"{name}-result"

async def main():
    # Run both "requests" concurrently
    results = await asyncio.gather(
        fetch_data("UserProfile", 2),
        fetch_data("OrderHistory", 3)
    )
    print(results)

asyncio.run(main())
```

**What happens:** Both `fetch_data` calls start almost immediately. Instead of waiting 2s then 3s (5s total, sequential), they run *concurrently*, and total time is ~3s (the longer of the two) — because while one is "sleeping" (waiting), the event loop runs the other.

**Real world example:** An API backend (e.g., built with FastAPI) that needs to call 3 external services (payment gateway, inventory service, shipping service) to build one response. Using `asyncio.gather()`, all 3 calls fire concurrently instead of one after another, cutting response time roughly to the slowest single call instead of the sum of all three.

---

## 7. async / await Syntax

- `async def` → declares a coroutine function. Calling it doesn't run it immediately — it returns a coroutine object.
- `await` → pauses the coroutine at this point, hands control back to the event loop, and resumes once the awaited operation completes.

```python
async def get_user(user_id):
    response = await http_client.get(f"/users/{user_id}")  # pauses here, event loop does other work
    return response.json()
```

### Common mistakes to avoid
1. **Using `time.sleep()` instead of `asyncio.sleep()`** inside async code — `time.sleep()` blocks the *entire* thread (and event loop), defeating the purpose of async entirely.
2. **Calling a blocking library** (like `requests` instead of `aiohttp`/`httpx`) inside async code — it freezes the whole event loop while that call runs.
3. **Forgetting `await`** — calling an async function without `await` just creates a coroutine object; it never actually runs.
4. Mixing CPU-heavy work inside async functions — async doesn't speed up computation, only waiting. A CPU-bound loop inside a coroutine will block the entire event loop.

---

## 8. The Event Loop

The **event loop** is the heart of asyncio — a single-threaded loop that:
1. Keeps track of all pending tasks/coroutines
2. Runs a task until it hits an `await` (a pause point)
3. Switches to another ready task
4. Comes back to the paused task once its awaited operation is done
5. Repeats until all tasks are complete

### Mental model
Think of the event loop like a **restaurant order queue manager**:
- It doesn't cook the food itself
- It just keeps checking: "Is this order ready? No? Move to the next one. Is that one ready? Yes! Serve it."
- Nothing happens in true parallel — it's all about smart switching during idle/waiting time

### Visual flow

```
Task A: ---[running]---(await: waiting on network)-------[resumed]---[done]
Task B: --------------[running]--(await: waiting on disk)----[resumed]--[done]
Event Loop: A runs → A awaits → switch to B → B runs → B awaits → 
            back to A (ready) → A resumes → A finishes → back to B → B finishes
```

**Real world example:** Node.js's entire runtime model is built around a single event loop — this is *why* a single Node.js process can handle tens of thousands of concurrent HTTP requests without spawning thousands of threads. Python's `asyncio`, JavaScript's event loop, and Go's goroutine scheduler (though goroutines are a bit different) all lean on this same core idea: don't block on waiting, do something useful instead.

---

## 9. Performance Concepts

### 9.1 Blocking vs Non-Blocking
- **Blocking call:** Halts the entire thread until it completes (e.g., `time.sleep()`, synchronous `requests.get()`).
- **Non-blocking call:** Returns control immediately, letting other work happen while the operation completes in the background (e.g., `await asyncio.sleep()`).

### 9.2 Throughput vs Latency
- **Latency:** Time to complete a single task (how fast is *one* request?)
- **Throughput:** How many tasks completed per unit time (how many requests/sec can the system handle?)
- Async is excellent at improving **throughput** for IO-bound systems, but doesn't reduce the **latency** of a single operation (a network call still takes as long as it takes).

### 9.3 Context Switching Cost
- **OS thread switching** is relatively expensive (kernel involvement, register/stack saving).
- **Async task switching** happens in user space (no kernel involvement) → far cheaper → enables scaling to thousands of concurrent tasks.

### 9.4 Amdahl's Law (why parallelism has limits)
If a program is 90% parallelizable and 10% must run sequentially, no matter how many cores you throw at it, you can never speed it up more than **10x** — because that sequential 10% is always the bottleneck.

> **Practical takeaway:** Adding more threads/processes doesn't scale performance linearly forever. Always profile before optimizing — figure out if you're actually CPU-bound or IO-bound first.

---

## 10. Choosing the Right Tool — Decision Table

| Scenario | Bottleneck | Tool | Why |
|---|---|---|---|
| Downloading 1000 files from the internet | IO-bound | `asyncio` / threads | Mostly waiting on network |
| Resizing 1000 images | CPU-bound | `multiprocessing` | Heavy computation, needs real cores |
| Web server handling many client connections | IO-bound | `asyncio` (e.g., FastAPI, aiohttp) | Massive concurrency, cheap task switching |
| Training a machine learning model | CPU-bound (or GPU-bound) | `multiprocessing` / GPU parallelism | Pure computation |
| GUI app that must stay responsive while doing a task | Mixed | Threads (for background work) | Keep the main/UI thread free |
| Password hash cracking | CPU-bound | `multiprocessing` | Needs raw CPU power across cores |
| Chat application / real-time notifications | IO-bound | `asyncio` (WebSockets) | Long-lived, mostly-idle connections |

---

## 11. Real-World Case Studies (Summary)

1. **Web scraper (IO-bound):** Scraping 500 URLs sequentially might take ~8 minutes. Using `asyncio` + `aiohttp`, the same job can finish in under 30 seconds, because the program isn't wasting time idly waiting on each response one at a time.

2. **Image processing pipeline (CPU-bound):** A photo-sharing app resizing/compressing thousands of uploaded images. Using `multiprocessing.Pool` to distribute images across all CPU cores cuts processing time roughly proportional to the number of cores available.

3. **Backend API aggregator (IO-bound):** An e-commerce checkout page needs data from the Payment, Inventory, and Shipping microservices. `asyncio.gather()` calls all three concurrently instead of sequentially — response time ≈ slowest single service, not the sum of all three.

4. **Chat/notification server (IO-bound, high concurrency):** WhatsApp-style backend holding open connections for millions of users. Threads would exhaust memory quickly; an async event-loop-based server (or Go's goroutines) handles this efficiently because each idle connection costs very little.

5. **Video transcoding service (CPU-bound):** YouTube-style platform splitting a video into segments and encoding each on a separate process/core in parallel to cut overall processing time.

---

## 12. Quick-Revision Summary Sheet

- **Concurrency** = structure (dealing with many things). **Parallelism** = execution (doing many things at once).
- **CPU-bound** → use **multiprocessing**. **IO-bound** → use **threading or async**.
- **Threads** share memory, lightweight, but limited by Python's GIL for CPU work.
- **Processes** have separate memory, heavier, but achieve true parallelism.
- **Async programming** = single-threaded cooperative multitasking; tasks yield control at `await` points instead of the OS forcibly switching them.
- **Event loop** = the scheduler that manages and resumes coroutines when their awaited operation completes.
- **Never** block the event loop with `time.sleep()` or synchronous IO calls inside async code.
- Async improves **throughput**, not the **latency** of a single call.
- Always identify bottleneck type (CPU vs IO) *before* picking a concurrency tool.

---

### One-Line Mental Anchors (for fast recall)
- *Threads* → many workers, one shared kitchen, watch for collisions.
- *Processes* → many separate kitchens, no collisions, but no shared ingredients either.
- *Async* → one super-efficient worker who never stands idle, always juggling.
- *Event loop* → the manager telling the async worker who to help next.

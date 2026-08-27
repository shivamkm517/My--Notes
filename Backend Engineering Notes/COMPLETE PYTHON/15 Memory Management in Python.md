---
tags: [python, memory-management, garbage-collection, performance, internals, computer-science]
created: 2026-07-08
aliases: [Memory Management, Python Memory Model, Garbage Collection Deep Dive]
---

# Memory Management in Python — Deep Dive

> [!info] Scope
> This note goes below the surface of "Python manages memory for you" — covering the object model, the allocator hierarchy (pymalloc), reference counting internals, the generational garbage collector, common leak patterns, and profiling tools. For the GIL's relationship to reference counting, see [[Concurrency, Threads, Processes and Async Programming]] (Section 3A) — this note goes deeper into the memory model itself.

---

## 1. Everything is an Object — The Foundational Model

> [!note] Core Idea
> In CPython, **every value is a `PyObject`** — including integers, strings, functions, classes, and modules. There is no concept of a Python "primitive type" living outside the object system (unlike C's `int`, `float`, which are raw memory).

Every `PyObject` in CPython's C implementation has, at minimum:
```c
typedef struct _object {
    Py_ssize_t ob_refcnt;      // reference count
    PyTypeObject *ob_type;      // pointer to the type object
} PyObject;
```

> [!info] What This Means Practically
> - Even `x = 5` creates a full Python object with a refcount and a type pointer — not a raw machine integer
> - This is a major reason Python is slower than C for raw numeric work, and why libraries like NumPy exist — they store data in **contiguous raw C arrays**, bypassing per-element `PyObject` overhead

```python
import sys
print(sys.getsizeof(5))        # 28 bytes -- NOT 4 or 8 bytes like a C int!
print(sys.getsizeof(5.0))       # 24 bytes
print(sys.getsizeof("a"))        # 50 bytes
print(sys.getsizeof([]))          # 56 bytes (empty list, before any elements)
```

---

## 2. Variables Are Names, Not Boxes

> [!warning] Critical Mental Model Shift
> In languages like C, a variable is a labeled memory box holding a value. In Python, **a variable is just a name bound to an object that lives elsewhere in memory** — assignment doesn't copy data, it creates a new reference (a new name pointing at the same object).

```python
a = [1, 2, 3]
b = a              # b is now ANOTHER NAME for the SAME list object
b.append(4)
print(a)            # [1, 2, 3, 4] -- 'a' sees the change too! Same object.

print(id(a) == id(b))   # True -- identical memory address
print(a is b)              # True -- same object identity
```

```
Memory model:

a ─┐
    ├──► [1, 2, 3, 4]   (one list object in memory)
b ─┘
```

### `id()` and Object Identity
```python
x = "hello"
print(id(x))   # a specific memory address (implementation detail, CPython-specific)
```
> [!info]
> `id()` returns a unique identifier for an object's lifetime — in CPython, this happens to be the object's **memory address**, but this is an implementation detail, not a language guarantee.

### `is` vs `==`
| Operator | Checks | Example |
|---|---|---|
| `is` | **Identity** — same object in memory | `a is b` |
| `==` | **Equality** — same value (calls `__eq__`) | `a == b` |

```python
a = [1, 2, 3]
b = [1, 2, 3]
print(a == b)   # True -- same VALUE
print(a is b)    # False -- different OBJECTS in memory
```

---

## 3. Mutable vs Immutable Objects — Memory Implications

| Type | Mutable? | Examples |
|---|---|---|
| Mutable | Yes | `list`, `dict`, `set`, custom classes (by default) |
| Immutable | No | `int`, `float`, `str`, `tuple`, `frozenset`, `bool` |

```python
# Immutable: "modifying" actually creates a NEW object
x = 100
print(id(x))      # e.g. 140712834
x += 1
print(id(x))       # DIFFERENT id -- a new int object was created, x now points to it

# Mutable: modifying changes the SAME object in place
lst = [1, 2, 3]
print(id(lst))      # e.g. 140712900
lst.append(4)
print(id(lst))        # SAME id -- the object was mutated in place
```

> [!warning] The Classic Mutable Default Argument Trap
> ```python
> def add_item(item, bucket=[]):    # bucket is created ONCE, at function definition time
>     bucket.append(item)
>     return bucket
>
> print(add_item(1))   # [1]
> print(add_item(2))    # [1, 2] -- SURPRISE! Same list object reused across calls
> ```
> This happens because default argument objects are created **once** and persist across calls — a direct consequence of Python's object/reference model. Fix: use `None` as the sentinel default and create a new object inside the function.

### Shallow Copy vs Deep Copy
```python
import copy

original = [[1, 2], [3, 4]]

shallow = copy.copy(original)          # new outer list, but inner lists are SHARED
shallow[0].append(99)
print(original)   # [[1, 2, 99], [3, 4]] -- original affected!

deep = copy.deepcopy(original)          # recursively copies EVERYTHING
deep[0].append(100)
print(original)     # unaffected by deep copy
```

| Copy Type | Behavior |
|---|---|
| Assignment (`b = a`) | No copy — new name, same object |
| Shallow copy (`copy.copy()`, slicing `lst[:]`) | New outer container, but nested objects are shared references |
| Deep copy (`copy.deepcopy()`) | Recursively creates fully independent copies of everything |

---

## 4. Stack vs Heap in Python

> [!note] Where Things Actually Live
> - **Call stack**: stores each function call's **frame** — local variable *names* (references), return addresses, not the actual objects themselves
> - **Heap**: where all Python **objects** actually live (ints, lists, strings, custom instances) — managed by CPython's memory allocators

```python
def func():
    x = [1, 2, 3]    # 'x' (the reference/name) lives on the stack frame for func()
    return x           # the LIST OBJECT itself lives on the heap

result = func()
# func()'s stack frame is destroyed after return,
# but the list object survives on the heap because 'result' still references it
```

> [!info]
> Unlike C/C++, Python developers rarely think about stack vs heap explicitly — the interpreter and reference counting/GC handle object lifetime automatically. But understanding this model clarifies *why* objects can outlive the function that created them (as long as something still references them).

---

## 5. CPython's Memory Allocator — `pymalloc`

> [!note] Why Not Just Use `malloc()`/`free()` Directly?
> General-purpose C allocators (`malloc`) are optimized for arbitrary-sized, long-lived allocations. Python programs create **huge numbers of small, short-lived objects** (ints, small tuples, etc.) — so CPython layers its own specialized allocator, **pymalloc**, on top of the OS/C allocator for small objects (< 512 bytes).

### The Three-Tier Allocator Hierarchy

```
┌─────────────────────────────────────────────┐
│                    Arena                        │  256 KB chunk, requested from OS via malloc/mmap
│  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │   Pool     │  │   Pool     │  │   Pool     │  │  4 KB each, subdivided from an Arena
│  │ ┌───┬───┐  │  │ ┌───┬───┐  │  │ ┌───┬───┐  │  │
│  │ │Blk│Blk│  │  │ │Blk│Blk│  │  │ │Blk│Blk│  │  │  Fixed-size Blocks (8, 16, 24, ... up to 512 bytes)
│  │ └───┴───┘  │  │ └───┴───┘  │  │ └───┴───┘  │  │
│  └───────────┘  └───────────┘  └───────────┘    │
└─────────────────────────────────────────────┘
```

| Level | Size | Purpose |
|---|---|---|
| **Arena** | 256 KB | Large chunk requested from the OS; contains multiple pools |
| **Pool** | 4 KB | Subdivided from an arena; each pool serves objects of ONE specific size class |
| **Block** | 8–512 bytes (in steps of 8) | The actual unit handed out to your Python object |

> [!info] Why This Design?
> - Avoids calling the (relatively slow) OS-level allocator for every tiny object
> - Reduces memory fragmentation by grouping same-sized objects together
> - Objects **larger than 512 bytes** bypass pymalloc and go directly to the system `malloc()`

### Arena Reclamation
When all pools within an arena become empty (all their objects freed), CPython can **return the entire arena to the OS** — this is one of the few ways Python actually shrinks its memory footprint at the OS level (rather than just marking memory "free" for internal reuse).

---

## 6. Reference Counting — The Deeper Mechanics

> [!info] Cross-Reference
> Covered at an introductory level in [[Concurrency, Threads, Processes and Async Programming]] (Section 3A). This section goes further into the mechanics and edge cases.

### `Py_INCREF` and `Py_DECREF`
At the C level, every time a reference is created or destroyed, CPython calls:
```c
Py_INCREF(obj);    // ob_refcnt += 1
Py_DECREF(obj);     // ob_refcnt -= 1; if ob_refcnt == 0, deallocate immediately
```

### Small Integer Caching (Interning)
CPython pre-allocates and caches small integers in the range **-5 to 256** at startup — these are singleton objects, never garbage collected during the program's life.

```python
a = 100
b = 100
print(a is b)   # True -- both point to the SAME cached int object

x = 1000
y = 1000
print(x is y)    # False (usually) -- outside the cached range, separate objects
                   # (though CPython may still sometimes fold identical literals in the same code object)
```

> [!warning] Never Rely on `is` for Integer/Value Comparison
> The caching range (-5 to 256) is a CPython **implementation detail**, not a language guarantee. Always use `==` for value comparisons; reserve `is` strictly for identity checks (e.g., `is None`).

### String Interning
Python also interns certain strings — particularly short strings that look like identifiers (letters, digits, underscores) — to save memory, since identifier-like strings (variable names, dict keys, attribute names) repeat constantly.

```python
a = "hello"
b = "hello"
print(a is b)   # True -- typically interned (identifier-like literal)

c = "hello world!"
d = "hello world!"
print(c is d)    # False (usually) -- contains a space/punctuation, not auto-interned

# Force interning manually:
import sys
e = sys.intern("hello world!")
f = sys.intern("hello world!")
print(e is f)     # True -- now guaranteed to be the same object
```

> [!info] Why Interning Matters for Memory
> If a program creates millions of dictionary keys or attribute names that are identical strings, interning means they all point to **one shared string object** instead of millions of duplicate objects — a substantial memory saving.

### Reference Counting Overhead Example
```python
import sys

class Node:
    def __init__(self, value):
        self.value = value

n = Node(5)
print(sys.getrefcount(n))   # 2 (the variable 'n' + the temp arg to getrefcount)

container = [n, n, n]         # THREE references to the SAME object, not 3 copies
print(sys.getrefcount(n))     # 5 (2 + 3 from the list)
```

---

## 7. The Generational Garbage Collector — Deeper Mechanics

> [!info] Cross-Reference
> Basic cycle-detection concept covered in [[Concurrency, Threads, Processes and Async Programming]] (Section 3A). Here we go deeper into thresholds, tuning, and debugging.

### Why Only Container Objects Are Tracked
The GC only needs to track objects **capable of holding references to other objects** (lists, dicts, sets, class instances, tuples-of-objects, etc.) — because only these can participate in a reference cycle. Simple objects like `int`, `str`, and `float` are never tracked by the cyclic GC (they can't reference other objects, so they can't form cycles).

```python
import gc

x = 5
print(gc.is_tracked(x))       # False -- int, can't form cycles

lst = [1, 2, 3]
print(gc.is_tracked(lst))       # True -- container object

t = (1, 2, 3)
print(gc.is_tracked(t))          # False -- tuple of only immutable/simple values may be untracked
                                    # (CPython optimizes this when it can prove no cycle is possible)
```

### Generational Thresholds in Detail

```python
import gc
print(gc.get_threshold())    # (700, 10, 10) by default
```

- **Generation 0** collected after **700 net allocations** (allocations minus deallocations of tracked objects) since the last Gen-0 collection
- **Generation 1** collected after **10** Gen-0 collections have occurred
- **Generation 2** collected after **10** Gen-1 collections have occurred

```python
gc.set_threshold(1000, 15, 15)   # tune collection frequency (rarely needed in practice)
```

> [!tip] When Tuning Actually Matters
> For performance-critical, allocation-heavy code (e.g., tight loops creating many short-lived container objects), some teams **disable the GC temporarily** (`gc.disable()`) during a hot section and re-enable it afterward, relying purely on reference counting during that window — since refcounting alone still frees non-cyclic garbage immediately.

### Inspecting and Debugging the GC

```python
import gc

gc.collect()                 # force full collection; returns count of unreachable objects collected
gc.get_count()                 # current object counts per generation, e.g. (233, 5, 1)
gc.get_stats()                  # detailed per-generation stats (collections, collected, uncollectable)

gc.set_debug(gc.DEBUG_STATS)     # print GC activity to stderr as it runs -- useful for diagnosing GC overhead
```

### `gc.garbage` — Uncollectable Objects
```python
gc.garbage   # list of objects the GC found in cycles but couldn't safely free
              # (rare in modern Python 3.4+, mostly historical relevance re: __del__ + cycles)
```

---

## 8. Weak References — Referencing Without Owning

> [!note] Definition
> A **weak reference** (`weakref` module) lets you reference an object **without increasing its reference count** — meaning the object can still be garbage collected even while a weak reference to it exists.

```python
import weakref

class Cache:
    def __init__(self, value):
        self.value = value

obj = Cache("expensive data")
weak = weakref.ref(obj)

print(weak())        # <Cache object> -- call it to access the referenced object
del obj
print(weak())          # None -- the object was collected; the weak ref doesn't keep it alive
```

### Why Use Weak References?

> [!success] Primary Use Case — Breaking Reference Cycles / Caches
> - **Caches**: you want to cache computed results, but don't want the cache itself to prevent memory from ever being freed
> - **Observer patterns**: an object holding a list of "listeners" shouldn't keep those listeners alive forever just by virtue of being subscribed
> - **Parent-child relationships**: e.g. a child node referencing its parent — if both used strong references, you'd create a reference cycle needing the GC; a weak reference from child→parent avoids this entirely

```python
import weakref

class TreeNode:
    def __init__(self, value, parent=None):
        self.value = value
        self._parent = weakref.ref(parent) if parent else None   # weak ref -- avoids cycle

    @property
    def parent(self):
        return self._parent() if self._parent else None
```

### `WeakValueDictionary` / `WeakKeyDictionary`
```python
import weakref

cache = weakref.WeakValueDictionary()

class BigObject:
    pass

obj = BigObject()
cache["key"] = obj
print("key" in cache)   # True

del obj
print("key" in cache)     # False -- entry automatically removed once the object was collected
```

---

## 9. `__slots__` — Reducing Per-Instance Memory Overhead

> [!note] Default Behavior
> By default, every instance of a Python class has a `__dict__` — a dictionary storing its instance attributes. Dictionaries have significant memory overhead (hash table structure) compared to a fixed set of named slots.

```python
import sys

class RegularPoint:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class SlottedPoint:
    __slots__ = ('x', 'y')     # no __dict__ created; fixed, fast attribute storage
    def __init__(self, x, y):
        self.x = x
        self.y = y

r = RegularPoint(1, 2)
s = SlottedPoint(1, 2)

print(sys.getsizeof(r.__dict__))    # e.g. 104 bytes just for the dict overhead!
print(hasattr(s, '__dict__'))         # False -- no dict at all
```

### Benefits and Tradeoffs

| Aspect | Regular Class (`__dict__`) | `__slots__` Class |
|---|---|---|
| Memory per instance | Higher (dict overhead) | Lower (fixed-size array-like storage) |
| Attribute access speed | Slightly slower (dict lookup) | Slightly faster (direct slot access) |
| Can add new attributes dynamically | Yes | No (only pre-declared slots allowed) |
| Supports multiple inheritance with slots | Complicated (all bases must be compatible) | Restricted |
| When to use | Default case; flexibility needed | Millions of instances of a simple, fixed-attribute class (e.g., data points, graph nodes) |

> [!tip] When `__slots__` Actually Matters
> The memory savings become significant when you're creating **hundreds of thousands or millions of instances** (e.g., parsing a huge dataset into objects). For typical application code with a handful of instances, the difference is negligible — don't add `__slots__` prematurely at the cost of flexibility.

---

## 10. Memory Leaks in Python — How They Actually Happen

> [!warning] "Python Doesn't Leak Memory" is a Myth
> Python **can** leak memory — not in the C sense of losing a raw pointer, but by **keeping references alive longer than intended**, preventing refcounts from ever reaching zero.

### Common Leak Patterns

#### 1. Growing Global/Module-Level Collections
```python
_cache = {}

def process(key, value):
    _cache[key] = value    # never removed -- grows forever across the program's life
```

#### 2. Reference Cycles Combined with `gc.disable()`
```python
import gc
gc.disable()   # if cyclic garbage collection is off, cycles NEVER get cleaned up
```

#### 3. Unbounded Caching (`functools.lru_cache` without `maxsize`)
```python
from functools import lru_cache

@lru_cache(maxsize=None)   # unbounded -- keeps EVERY unique call's result forever
def expensive(x):
    return x ** 2
```

#### 4. Closures Capturing Large Objects Unintentionally
```python
def make_handler(large_dataset):
    def handler(event):
        return len(large_dataset)     # closure keeps 'large_dataset' alive as long as 'handler' exists
    return handler

# If 'handler' is stored somewhere long-lived (e.g., an event listener registry),
# 'large_dataset' can never be freed, even if you no longer need it.
```

#### 5. Circular References with `__del__` (Rare, Legacy Concern)
As discussed in Section 7 — mostly resolved since Python 3.4, but still worth avoiding `__del__` for critical logic.

#### 6. Threads/Timers That Never Terminate, Holding References
```python
import threading

class Worker:
    def __init__(self, data):
        self.data = data
        self.timer = threading.Timer(9999, self.run)
        self.timer.start()   # if never cancelled, keeps 'self' (and 'self.data') alive indefinitely
    def run(self):
        pass
```

### Detecting Leaks — Practical Workflow
1. Take a memory snapshot (`tracemalloc`)
2. Run the suspected leaking operation repeatedly
3. Take another snapshot, diff against the first
4. Look for object counts/types that grow unexpectedly and never shrink

---

## 11. Profiling & Debugging Memory — Tools

### 11.1 `sys.getsizeof()` — Single Object Size
```python
import sys
print(sys.getsizeof([1, 2, 3]))   # size of the list object itself
                                     # NOTE: does NOT include the size of objects it references!
```
> [!warning] `getsizeof()` is Shallow
> For a list of large objects, `getsizeof()` only reports the size of the list's own internal array of pointers — not the memory used by the objects those pointers point to. Use `sys.getsizeof()` recursively (or a library) for a true deep size.

### 11.2 `tracemalloc` — Built-in Memory Profiler (Python 3.4+)
```python
import tracemalloc

tracemalloc.start()

# ... run your code ...
data = [str(i) for i in range(100000)]

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

for stat in top_stats[:5]:
    print(stat)   # shows which lines allocated the most memory
```

### Comparing Two Snapshots (Leak Detection)
```python
import tracemalloc

tracemalloc.start()
snapshot1 = tracemalloc.take_snapshot()

# ... run the suspected leaking code ...

snapshot2 = tracemalloc.take_snapshot()
diff = snapshot2.compare_to(snapshot1, 'lineno')

for stat in diff[:5]:
    print(stat)   # shows what GREW between the two snapshots
```

### 11.3 `objgraph` — Visualizing Object Graphs (Third-Party)
```python
import objgraph

objgraph.show_most_common_types(limit=10)     # what types have the most live instances
objgraph.show_growth()                          # what's grown since the last call (great for leak hunting)
objgraph.show_backrefs([obj], filename='refs.png')   # visualize what's holding a reference to obj
```

### 11.4 `memory_profiler` (Third-Party) — Line-by-Line Memory Usage
```python
from memory_profiler import profile

@profile
def my_func():
    a = [1] * 1000000
    b = [2] * 2000000
    del b
    return a
# Run with: python -m memory_profiler script.py
# Outputs per-line memory delta
```

### 11.5 `gc` Module for Live Diagnostics
```python
import gc

print(len(gc.get_objects()))    # total number of objects currently tracked by the GC
gc.collect()
print(gc.garbage)                  # objects GC found but couldn't free (should normally be empty)
```

---

## 12. Memory Views and Buffer Protocol (Advanced)

For large binary data (bytes, arrays, NumPy arrays), copying data is expensive. Python's **buffer protocol** and `memoryview` allow **zero-copy** access to underlying memory.

```python
data = bytearray(b"hello world")
view = memoryview(data)

print(view[0])          # 104 -- byte value, no copy made
sub_view = view[0:5]      # still no copy -- just a view into the same buffer

sub_view[0] = 72           # modifies the ORIGINAL bytearray through the view
print(data)                  # bytearray(b'Hello world')
```

> [!success] Why This Matters for Performance
> Without `memoryview`, slicing a large `bytes`/`bytearray` object creates a **full copy**. For large datasets (e.g., processing gigabyte-scale binary files or network buffers), `memoryview` avoids that copy entirely — critical in high-performance I/O code.

---

## 13. Object Lifecycle — Full Picture, Start to Finish

```
1. Object created (e.g. via a class's __new__)
        │
        ▼
2. Reference count initialized (typically 1)
        │
        ▼
3. Object gains/loses references as the program runs
   (Py_INCREF / Py_DECREF called automatically)
        │
        ▼
4a. Refcount hits 0
        │                       4b. Object is part of an
        ▼                           unreachable cycle
5a. Immediately deallocated         │
    (via tp_dealloc)                 ▼
                              5b. Detected by generational GC
                                  during a collection pass
                                       │
                                       ▼
                              6b. Deallocated by the GC
                                  (or added to gc.garbage
                                  if unsafe to free, rare)
```

---

## 14. Best Practices ✅

> [!success] Do's
> - Use `del` or let variables go out of scope to release references promptly when no longer needed
> - Use `weakref` for caches, observer patterns, and parent-child back-references to avoid unnecessary cycles
> - Use `__slots__` for classes instantiated in very large numbers with fixed attributes
> - Use `tracemalloc` (built-in, no dependency) as your first tool when investigating memory growth
> - Set a `maxsize` on `functools.lru_cache` unless you deliberately want unbounded caching
> - Use `memoryview` for zero-copy slicing of large binary buffers
> - Break intentional reference cycles explicitly (or rely on the generational GC) rather than leaving `gc` disabled indefinitely

> [!failure] Don'ts
> - Don't rely on `is` for value comparisons — the small-int/string interning ranges are CPython implementation details, not guarantees
> - Don't use mutable objects as default function arguments
> - Don't disable the GC (`gc.disable()`) globally in long-running applications unless you have a specific, measured reason and a plan to manage cycles yourself
> - Don't assume `sys.getsizeof()` gives you the full memory footprint of a nested structure — it's shallow
> - Don't let closures capture large objects unintentionally in long-lived callbacks/handlers
> - Don't add `__slots__` prematurely to small, low-volume classes — it costs you flexibility for negligible benefit

---

## 15. Quick Revision Summary

| Concept | Key Point |
|---|---|
| Object model | Everything is a `PyObject` with a refcount + type pointer |
| Variables | Names bound to objects, not boxes holding values |
| `is` vs `==` | Identity vs equality — never confuse the two |
| Mutable vs immutable | Mutation happens in-place only for mutable types |
| pymalloc | 3-tier allocator (Arena → Pool → Block) for small objects (<512 bytes) |
| Reference counting | Immediate, deterministic deallocation at refcount 0 |
| Small int/string interning | CPython caches small ints (-5 to 256) and identifier-like strings |
| Generational GC | Catches reference **cycles** refcounting can't; 3 generations, tuned thresholds |
| `weakref` | Reference without incrementing refcount — avoids cycles, enables self-cleaning caches |
| `__slots__` | Removes per-instance `__dict__`, saves memory at scale, costs flexibility |
| Memory leaks | Usually unintended long-lived references, not "true" leaks like in C |
| `tracemalloc` | Built-in snapshot-diffing tool — the standard first stop for leak hunting |
| `memoryview` | Zero-copy access to large binary buffers |

**Golden Rules:**
1. Python doesn't prevent leaks — it prevents *dangling pointers*. You can still leak by holding references too long.
2. Refcounting handles the common case immediately; the generational GC exists solely for cycles.
3. `is` = identity, `==` = equality — conflating them is a top source of subtle bugs.
4. Profile before optimizing — `tracemalloc`/`objgraph` tell you what's actually growing, don't guess.
5. `__slots__` and `weakref` are scale tools — reach for them when instance counts or graph complexity actually justify it.

---

## Related Notes
- [[Concurrency, Threads, Processes and Async Programming]]
- [[Exception Handling (Python)]]
- [[self, cls, Static Methods and __new__]]
- [[Four Pillars of OOP]]
- [[PEP 8 Style Guide]]

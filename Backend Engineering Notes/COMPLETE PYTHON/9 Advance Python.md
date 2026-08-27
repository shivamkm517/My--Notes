---
tags: [python, programming, notes]
title: Python Advanced Concepts
---

# Python Advanced Concepts

A deep-dive reference covering Decorators, Generators, Iterators, Context Managers, Lambda, Closures, `*args`/`**kwargs`, Deep/Shallow Copy, and Garbage Collection.

---

## Table of Contents

- [[#1. Closures]]
- [[#2. Decorators]]
- [[#3. Iterators]]
- [[#4. Generators]]
- [[#5. Context Managers]]
- [[#6. Lambda Functions]]
- [[#7. args (Arguments)]]
- [[#8. kwargs (Keyword Arguments)]]
- [[#9. Shallow Copy]]
- [[#10. Deep Copy]]
- [[#11. Garbage Collection]]

---

## 1. Closures

### What is a Closure?
A **closure** is a function object that "remembers" values from its enclosing lexical scope even after that outer scope has finished executing. It happens when:

1. There is a nested function (a function inside another function).
2. The nested function references a variable from the enclosing function.
3. The enclosing function returns the nested function.

### Example

```python
def outer_function(msg):
    # 'msg' is a free variable captured by the closure
    def inner_function():
        print(f"Message: {msg}")
    return inner_function

greet = outer_function("Hello, Obsidian!")
greet()  # Output: Message: Hello, Obsidian!
```

Even though `outer_function` has already returned, `inner_function` still has access to `msg`. This is because `inner_function` carries a **closure** — a reference to the environment in which it was created.

### Checking the closure

```python
print(greet.__closure__)          # (<cell at 0x...: str object at 0x...>,)
print(greet.__closure__[0].cell_contents)  # 'Hello, Obsidian!'
```

### Practical Use Case: Counters

```python
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

counter1 = make_counter()
print(counter1())  # 1
print(counter1())  # 2

counter2 = make_counter()
print(counter2())  # 1 (independent state!)
```

> [!note] Key Point
> `nonlocal` is required to **modify** a variable from the enclosing scope. Without it, Python would treat `count` inside `counter()` as a new local variable and raise an `UnboundLocalError`.

### Why Closures Matter
- They enable **data hiding** (encapsulation without classes).
- They are the foundation for **decorators**.
- Useful in callback-based and functional programming patterns.

---

## 2. Decorators

### What is a Decorator?
A **decorator** is a function that takes another function (or class) as input, adds some functionality, and returns a modified version of it — **without changing the original function's source code**. Decorators are a direct application of closures.

### Basic Syntax

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Something before the function runs")
        result = func(*args, **kwargs)
        print("Something after the function runs")
        return result
    return wrapper

@my_decorator
def say_hello(name):
    print(f"Hello, {name}!")

say_hello("World")
```

Output:
```
Something before the function runs
Hello, World!
Something after the function runs
```

The `@my_decorator` syntax is just syntactic sugar for:

```python
say_hello = my_decorator(say_hello)
```

### Preserving Metadata with `functools.wraps`

Without `wraps`, the wrapped function loses its original name and docstring.

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # preserves func.__name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### Decorators with Arguments

To make a decorator itself accept arguments, you need an extra layer of nesting.

```python
def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    print(f"Hi {name}")

greet("Amit")
# Hi Amit
# Hi Amit
# Hi Amit
```

### Class-Based Decorators

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call {self.count} of {self.func.__name__}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hi():
    print("Hi!")

say_hi()  # Call 1 of say_hi \n Hi!
say_hi()  # Call 2 of say_hi \n Hi!
```

### Common Built-in Decorators
| Decorator | Purpose |
|---|---|
| `@staticmethod` | Method that doesn't access instance/class state |
| `@classmethod` | Method that receives the class (`cls`) instead of instance |
| `@property` | Turns a method into a read-only attribute |
| `@functools.lru_cache` | Caches function results (memoization) |
| `@functools.wraps` | Preserves original function metadata in decorators |

### Real-World Use Cases
- Logging
- Timing / performance measurement
- Authentication & authorization checks
- Caching (memoization)
- Input validation
- Rate limiting

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper
```

---

## 3. Iterators

### What is an Iterator?
An **iterator** is an object that implements the **iterator protocol**:
- `__iter__()` → returns the iterator object itself.
- `__next__()` → returns the next value, raising `StopIteration` when exhausted.

### Iterable vs Iterator
- **Iterable**: An object capable of returning its members one at a time (implements `__iter__`). Examples: `list`, `tuple`, `dict`, `str`.
- **Iterator**: The object produced by calling `iter()` on an iterable — it keeps state and implements `__next__`.

```python
my_list = [1, 2, 3]
my_iter = iter(my_list)   # my_list is iterable; my_iter is the iterator

print(next(my_iter))  # 1
print(next(my_iter))  # 2
print(next(my_iter))  # 3
print(next(my_iter))  # raises StopIteration
```

### Building a Custom Iterator

```python
class CountDown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self  # the object is its own iterator

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for num in CountDown(5):
    print(num)  # 5 4 3 2 1
```

### How `for` loops actually work
A `for` loop is syntactic sugar around the iterator protocol:

```python
iterator = iter(some_iterable)
while True:
    try:
        item = next(iterator)
    except StopIteration:
        break
    else:
        # loop body
        print(item)
```

> [!tip]
> This is why you can only loop through certain iterators (like generators or file objects) **once** — they don't reset their internal state.

---

## 4. Generators

### What is a Generator?
A **generator** is a special, simpler way to create iterators. Instead of manually implementing `__iter__` and `__next__`, you write a function using the `yield` keyword. Calling a generator function does **not** run the function body immediately — it returns a **generator object** that produces values lazily, on demand.

### Generator Function

```python
def count_up_to(limit):
    count = 1
    while count <= limit:
        yield count
        count += 1

gen = count_up_to(5)
print(next(gen))  # 1
print(next(gen))  # 2

for num in gen:   # continues from where it left off: 3, 4, 5
    print(num)
```

### Key Behavior
- Execution **pauses** at `yield` and resumes right after it on the next call to `next()`.
- Local state (variables, loop position) is preserved between calls.
- Raises `StopIteration` automatically when the function ends.

### Generator Expressions
Like list comprehensions, but lazy — use `()` instead of `[]`.

```python
squares = (x**2 for x in range(1_000_000))  # doesn't compute anything yet
print(next(squares))  # 0
print(next(squares))  # 1
```

### Why Use Generators? (Memory Efficiency)

```python
# List comprehension — builds the ENTIRE list in memory
squares_list = [x**2 for x in range(10_000_000)]  # heavy memory usage

# Generator — computes ONE value at a time
squares_gen = (x**2 for x in range(10_000_000))    # nearly no memory usage
```

Generators are ideal for:
- Reading large files line by line.
- Streaming data / infinite sequences.
- Pipelines where you chain multiple lazy transformations.

### `send()`, `throw()`, and `close()`

```python
def echo():
    while True:
        received = yield
        print(f"Got: {received}")

gen = echo()
next(gen)          # prime the generator (advance to first yield)
gen.send("Hello")  # Got: Hello
gen.close()        # stops the generator
```

### `yield from` — Delegating to Sub-generators

```python
def inner():
    yield 1
    yield 2

def outer():
    yield from inner()
    yield 3

print(list(outer()))  # [1, 2, 3]
```

### Generators are Iterators
Every generator automatically satisfies the iterator protocol — that's why `for` loops, `list()`, `sum()`, etc. all work on generators directly.

---

## 5. Context Managers

### What is a Context Manager?
A **context manager** is an object that defines the runtime context to be established when executing a `with` statement — handling **setup** and **teardown** (cleanup) automatically, even if an exception occurs. The most common example is file handling.

```python
with open("file.txt", "r") as f:
    data = f.read()
# file is automatically closed here, even if an error occurred above
```

### The Protocol: `__enter__` and `__exit__`

```python
class ManagedFile:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode

    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file

    def __exit__(self, exc_type, exc_value, traceback):
        if self.file:
            self.file.close()
        # return True to suppress exceptions, False/None to propagate them
        return False

with ManagedFile("test.txt", "w") as f:
    f.write("Hello, World!")
```

### `__exit__` Parameters
| Parameter | Meaning |
|---|---|
| `exc_type` | Exception class (or `None` if no exception) |
| `exc_value` | Exception instance |
| `traceback` | Traceback object |

If `__exit__` returns `True`, any exception raised inside the `with` block is **suppressed**.

### Using `contextlib.contextmanager` (Simpler Approach)

```python
from contextlib import contextmanager

@contextmanager
def managed_file(filename, mode):
    f = open(filename, mode)
    try:
        yield f          # code before yield = __enter__
    finally:
        f.close()        # code after yield = __exit__ (always runs)

with managed_file("test.txt", "w") as f:
    f.write("Hello!")
```

### Multiple Context Managers

```python
with open("in.txt") as infile, open("out.txt", "w") as outfile:
    outfile.write(infile.read())
```

### Real-World Use Cases
- File handling (auto-close)
- Database connections (auto-commit/rollback + close)
- Thread locks (`with lock: ...`)
- Timing code blocks
- Temporarily changing state (e.g., suppressing warnings, changing working directory)

```python
import contextlib

with contextlib.suppress(FileNotFoundError):
    os.remove("nonexistent_file.txt")  # no error raised if missing
```

---

## 7. `*args` (Arguments)

### What is `*args`?
`*args` allows a function to accept **any number of positional arguments**. Inside the function, `args` becomes a **tuple** of all the extra positional values passed in.

### Example

```python
def add_all(*args):
    print(type(args))  # <class 'tuple'>

### Unpacking with `*` when Calling a Function

```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))  # unpacks list into 3 separate args → 6
```

> [!note]
> `*args` is just a **naming convention** — the important part is the `*`. You could name it `*values` or `*items`; `args` is simply the community standard.

---

## 8. `**kwargs` (Keyword Arguments)

### What is `**kwargs`?
`**kwargs` allows a function to accept **any number of keyword arguments** (name=value pairs). Inside the function, `kwargs` becomes a **dictionary**.

### Example

```python
def print_info(**kwargs):
    print(type(kwargs))  # <class 'dict'>
    for key, value in kwargs.items():
        print(f"{key}: {value}")

print_info(name="Amit", age=25, city="Bhopal")
# name: Amit
# age: 25
# city: Bhopal
```

### Combining `*args` and `**kwargs`

Correct parameter ordering in a function definition:

```python
def full_example(a, b, *args, c=10, **kwargs):
    print("a, b:", a, b)
    print("args:", args)
    print("c:", c)
    print("kwargs:", kwargs)

full_example(1, 2, 3, 4, c=99, d=5, e=6)
# a, b: 1 2
# args: (3, 4)
# c: 99
# kwargs: {'d': 5, 'e': 6}
```

### Parameter Order Rule
```
def func(positional, *args, keyword_only, **kwargs):
```
1. Standard positional/keyword parameters
2. `*args`
3. Keyword-only parameters (with defaults)
4. `**kwargs`

### Unpacking a Dictionary into Function Calls

```python
def greet(name, greeting):
    print(f"{greeting}, {name}!")

data = {"name": "Amit", "greeting": "Hello"}
greet(**data)  # Hello, Amit!
```

### Why Use `*args`/`**kwargs`?
- Writing flexible/generic functions (e.g., wrapper functions in decorators).
- Forwarding arguments to another function without knowing its exact signature.
- Building APIs where the number of inputs may vary.

```python
def wrapper(*args, **kwargs):
    return original_function(*args, **kwargs)
```

---

## 9. Shallow Copy

### What is a Shallow Copy?
A **shallow copy** creates a new outer object, but **does not recursively copy nested objects** — it copies references to them. So the top-level container is independent, but nested/mutable objects inside are **shared** between the original and the copy.

### Creating a Shallow Copy

```python
import copy

original = [[1, 2, 3], [4, 5, 6]]
shallow = copy.copy(original)
# or: shallow = original.copy()
# or: shallow = list(original)
# or: shallow = original[:]
```

### Demonstrating the "Shared Reference" Problem

```python
original = [[1, 2, 3], [4, 5, 6]]
shallow = copy.copy(original)

shallow.append([7, 8, 9])   # only affects 'shallow' (new top-level list)
print(original)  # [[1, 2, 3], [4, 5, 6]]        <- unaffected
print(shallow)   # [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

shallow[0][0] = 999          # modifies a NESTED list — shared!
print(original)  # [[999, 2, 3], [4, 5, 6]]      <- CHANGED!
print(shallow)   # [[999, 2, 3], [4, 5, 6], [7, 8, 9]]
```

> [!warning]
> Both `original[0]` and `shallow[0]` point to the **same inner list object** in memory. Modifying a nested element affects both.

### Visual Model
```
original ──▶ [ ref_A, ref_B ]
shallow  ──▶ [ ref_A, ref_B ]     (new outer list, same inner refs)
                 │       │
                 ▼       ▼
              [1,2,3]  [4,5,6]   (shared objects)
```

---

## 10. Deep Copy

### What is a Deep Copy?
A **deep copy** recursively copies **every nested object**, creating a fully independent clone. Changes to the copy (at any depth) never affect the original, and vice versa.

### Creating a Deep Copy

```python
import copy

original = [[1, 2, 3], [4, 5, 6]]
deep = copy.deepcopy(original)

deep[0][0] = 999
print(original)  # [[1, 2, 3], [4, 5, 6]]   <- unaffected!
print(deep)       # [[999, 2, 3], [4, 5, 6]]
```

### Visual Model
```
original ──▶ [ ref_A, ref_B ]
                 │       │
                 ▼       ▼
              [1,2,3]  [4,5,6]

deep     ──▶ [ ref_C, ref_D ]     (entirely new objects, all levels)
                 │       │
                 ▼       ▼
              [1,2,3]  [4,5,6]   (independent copies)
```

### Shallow vs Deep Copy — Summary Table

| Aspect | Shallow Copy | Deep Copy |
|---|---|---|
| Function | `copy.copy(obj)` | `copy.deepcopy(obj)` |
| Top-level object | New | New |
| Nested objects | Shared references | Fully independent copies |
| Speed | Faster | Slower (recursive) |
| Memory | Less | More |
| Use case | Flat/simple structures, or when sharing nested data is fine | Nested/complex structures needing full independence |

### Custom Copy Behavior
You can control how your own classes are copied by implementing `__copy__` and `__deepcopy__`.

```python
class Node:
    def __init__(self, value, children=None):
        self.value = value
        self.children = children or []

    def __deepcopy__(self, memo):
        new_node = Node(self.value)
        new_node.children = copy.deepcopy(self.children, memo)
        return new_node
```

> [!tip]
> The `memo` dictionary passed to `__deepcopy__` prevents infinite loops when copying objects with **circular references**.

---

## 11. Garbage Collection

### What is Garbage Collection (GC)?
**Garbage collection** is Python's automatic memory management system that reclaims memory occupied by objects that are no longer needed/reachable by the program, preventing memory leaks.

Python uses **two main mechanisms**:
1. **Reference Counting** (primary mechanism)
2. **Generational Garbage Collector** (handles cyclic references)

### 1. Reference Counting
Every object in Python keeps a count of how many references point to it. When the count drops to **zero**, the object is immediately deallocated.

```python
import sys

a = []
print(sys.getrefcount(a))  # 2 (one from 'a', one from getrefcount's own argument)

b = a
print(sys.getrefcount(a))  # 3 (now 'a', 'b', and the temp argument reference it)

del b
print(sys.getrefcount(a))  # 2 (back down)
```

When an object's reference count hits 0, CPython frees its memory **immediately** — this is why simple objects are cleaned up predictably and fast.

### 2. The Problem: Circular/Cyclic References
Reference counting alone **cannot** detect cycles — when two or more objects reference each other but nothing else references them.

```python
class Node:
    def __init__(self):
        self.ref = None

a = Node()
b = Node()
a.ref = b   # a references b
b.ref = a   # b references a

del a
del b
# Even though 'a' and 'b' variables are deleted,
# the two Node objects still reference EACH OTHER
# → reference count never reaches 0 → memory leak (without cyclic GC)
```

This is where Python's **generational garbage collector** (the `gc` module) comes in — it specifically scans for and collects these reference cycles.

### The `gc` Module

```python
import gc

# Check if garbage collection is enabled
print(gc.isenabled())  # True (by default)

# Manually trigger a collection
gc.collect()

# Get current generation thresholds
print(gc.get_threshold())  # e.g. (700, 10, 10)

# Get objects tracked by GC
print(len(gc.get_objects()))

# Disable / enable automatic GC
gc.disable()
gc.enable()
```

### Generational Garbage Collection
Python's cyclic GC divides objects into **3 generations** based on how long they've survived:

| Generation | Description |
|---|---|
| **Gen 0** | New objects. Collected most frequently. |
| **Gen 1** | Objects that survived one Gen-0 collection. Collected less often. |
| **Gen 2** | Long-lived objects that survived Gen-1 collections. Collected least often. |

**Why generations?** Empirically, most objects die young. So it's more efficient to frequently scan new/short-lived objects (Gen 0) and rarely scan old/long-lived ones (Gen 2) — this is the **generational hypothesis**.

Each generation has a **threshold** — when the count of new allocations minus deallocations exceeds the threshold, a collection is triggered for that generation.

```python
import gc
print(gc.get_threshold())  
# (700, 10, 10) → Gen0 threshold=700, Gen1=10, Gen2=10
```

### Manually Controlling GC

```python
import gc

gc.disable()          # turn off automatic cyclic GC
# ... performance-critical code that creates lots of short-lived objects ...
gc.enable()            # turn it back on
gc.collect()           # force a full collection
```

### `__del__` and Finalizers

```python
class Resource:
    def __del__(self):
        print("Resource cleaned up!")

r = Resource()
del r  # "Resource cleaned up!" printed (ref count hits 0)
```

> [!warning]
> Avoid relying on `__del__` for critical cleanup (like closing files/connections) — its execution timing isn't guaranteed, especially with reference cycles. Prefer **context managers** (`with` statement) for deterministic cleanup instead.

### Weak References (Breaking Cycles Intentionally)
The `weakref` module lets you reference an object **without increasing its reference count** — useful for caches and observer patterns to avoid unwanted cycles.

```python
import weakref

class Data:
    pass

obj = Data()
weak_ref = weakref.ref(obj)

print(weak_ref())  # <__main__.Data object at ...>
del obj
print(weak_ref())  # None (object was collected since no strong refs remain)
```

### Summary
| Mechanism | Handles | Trigger |
|---|---|---|
| Reference Counting | Simple, non-circular references | Immediate (count hits 0) |
| Generational GC | Circular/cyclic references | Periodic, based on generation thresholds |
| `gc.collect()` | Manual, forced collection | On demand |
| `weakref` | Avoiding unwanted reference retention | N/A (prevention, not collection) |

---

## Quick Reference Cheat Sheet

| Concept | One-Line Definition |
|---|---|
| **Closure** | A function that remembers variables from its enclosing scope |
| **Decorator** | A function that wraps another function to extend its behavior |
| **Iterator** | An object implementing `__iter__` + `__next__` for sequential access |
| **Generator** | A function using `yield` to lazily produce a sequence of values |
| **Context Manager** | An object with `__enter__`/`__exit__` for automatic setup/cleanup (`with`) |
| **Lambda** | A small anonymous, single-expression function |
| **`*args`** | Collects extra positional arguments into a tuple |
| **`**kwargs`** | Collects extra keyword arguments into a dict |
| **Shallow Copy** | Copies the outer object; nested objects are shared |
| **Deep Copy** | Recursively copies everything; fully independent clone |
| **Garbage Collection** | Automatic reclamation of unused memory (ref counting + cyclic GC) |

---

*Tags:* #python #decorators #generators #iterators #context-managers #lambda #closures #copy #garbage-collection

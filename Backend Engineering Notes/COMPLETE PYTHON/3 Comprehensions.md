---
tags: [python, comprehensions, programming, computer-science]
created: 2026-07-08
aliases: [Comprehensions, List Comprehension, Dict Comprehension, Set Comprehension]
---

# Python Comprehensions — Deep Dive

## 1. What is a Comprehension?

A **comprehension** is a concise, expressive syntax in Python for creating a new collection (list, set, dict) — or a generator — by transforming and/or filtering elements from an existing iterable, in a **single readable line**.

> [!note] Core Idea
> Comprehensions are essentially syntactic sugar over a `for` loop + optional `if` condition + an expression, but they are more **Pythonic**, often faster, and reduce boilerplate.

### General Syntax Template
```python
[expression for item in iterable if condition]   # list
{expression for item in iterable if condition}   # set
{key_expr: value_expr for item in iterable if condition}  # dict
(expression for item in iterable if condition)   # generator (NOT a tuple!)
```

> [!warning] Common Misconception
> There is **no "tuple comprehension"** in Python. Using `()` with comprehension syntax creates a **generator expression**, not a tuple. To get a tuple, wrap a generator expression (or list comprehension) with `tuple(...)`.
>
> There is also **no direct "frozenset comprehension"** syntax — you build it by wrapping a set comprehension (or generator) with `frozenset(...)`.

---

## 2. List Comprehension

Creates a new **list** by evaluating an expression for each item in an iterable.

### Basic Syntax
```python
squares = [x**2 for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

### Equivalent traditional loop
```python
squares = []
for x in range(10):
    squares.append(x**2)
```

### With Condition (Filtering)
```python
even_squares = [x**2 for x in range(10) if x % 2 == 0]
# [0, 4, 16, 36, 64]
```

### With if-else (Conditional Expression — NOT filtering)
```python
labels = ["even" if x % 2 == 0 else "odd" for x in range(5)]
# ['even', 'odd', 'even', 'odd', 'even']
```

> [!tip] Placement Rule
> - `if` **after** the `for` clause → filters items (may omit some)
> - `if...else` **before** the `for` clause (as part of the expression) → transforms every item, never omits any

### Nested Loops in List Comprehension
```python
pairs = [(x, y) for x in range(3) for y in range(2)]
# [(0,0), (0,1), (1,0), (1,1), (2,0), (2,1)]
```
Equivalent to:
```python
pairs = []
for x in range(3):
    for y in range(2):
        pairs.append((x, y))
```

### Nested Comprehension (list of lists — e.g., flattening a matrix)
```python
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flattened = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### Comprehension Producing a Matrix (2D list)
```python
matrix = [[i * j for j in range(3)] for i in range(3)]
# [[0, 0, 0], [0, 1, 2], [0, 2, 4]]
```

> [!warning] Common Pitfall — Nested Mutable Default
> `[[0]*3]*3` creates **3 references to the same inner list** (aliasing bug). Use a comprehension instead:
> ```python
> grid = [[0 for _ in range(3)] for _ in range(3)]  # safe, independent rows
> ```

---

## 3. Set Comprehension

Creates a new **set** — automatically removes duplicates, unordered.

### Basic Syntax
```python
unique_squares = {x**2 for x in [1, 1, 2, 2, 3, 3]}
# {1, 4, 9}
```

### With Condition
```python
vowels_found = {ch for ch in "hello world" if ch in "aeiou"}
# {'o', 'e'}
```

> [!info] Key Property
> Since sets are **unordered and unindexed**, and automatically deduplicate, set comprehensions are ideal when you need **uniqueness** without caring about order.

### Difference from List Comprehension
| Aspect | List Comprehension | Set Comprehension |
|---|---|---|
| Syntax | `[ ]` | `{ }` |
| Duplicates | Allowed | Automatically removed |
| Order | Preserved (insertion order) | Not guaranteed |
| Mutable? | Yes | Yes |
| Hashable requirement | No | Elements must be hashable |

---

## 4. Dictionary Comprehension

Creates a new **dict** by specifying both a key expression and a value expression.

### Basic Syntax
```python
squares_dict = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

### With Condition
```python
even_squares_dict = {x: x**2 for x in range(10) if x % 2 == 0}
# {0: 0, 2: 4, 4: 16, 6: 36, 8: 64}
```

### Swapping Keys and Values
```python
original = {'a': 1, 'b': 2, 'c': 3}
swapped = {v: k for k, v in original.items()}
# {1: 'a', 2: 'b', 3: 'c'}
```

### Building from Two Lists (using zip)
```python
keys = ['name', 'age', 'city']
values = ['Alice', 30, 'Delhi']
merged = {k: v for k, v in zip(keys, values)}
# {'name': 'Alice', 'age': 30, 'city': 'Delhi'}
```

### Merging/Transforming an Existing Dict
```python
prices = {'apple': 100, 'banana': 40, 'mango': 90}
discounted = {item: price * 0.9 for item, price in prices.items()}
```

> [!warning] Pitfall
> If duplicate keys are generated during the comprehension, **the last value wins** (silently overwrites earlier ones):
> ```python
> {x % 3: x for x in range(6)}
> # {0: 3, 1: 4, 2: 5}  -- earlier values for each key are overwritten
> ```

---

## 5. Generator Expression (the "Tuple-like" Comprehension)

Uses `()` — but produces a **lazy generator object**, NOT a tuple.

### Basic Syntax
```python
gen = (x**2 for x in range(10))
print(gen)          # <generator object <genexpr> at 0x...>
print(list(gen))     # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

### Key Characteristics
- **Lazy evaluation** — values are computed **on demand**, one at a time, not stored in memory all at once
- Can only be **iterated once** (exhausted after first full pass)
- Much more **memory-efficient** for large datasets
- Commonly used directly inside function calls without extra parentheses:
```python
total = sum(x**2 for x in range(1000000))  # no need for extra ()
any(x > 100 for x in data)
all(x % 2 == 0 for x in numbers)
```

### To Get an Actual Tuple
```python
squares_tuple = tuple(x**2 for x in range(10))
# (0, 1, 4, 9, 16, 25, 36, 49, 64, 81)
```

### Generator vs List Comprehension

| Aspect | List Comprehension | Generator Expression |
|---|---|---|
| Brackets | `[ ]` | `( )` |
| Evaluation | Eager (all at once) | Lazy (on demand) |
| Memory usage | Stores all elements | Stores only current state |
| Reusable? | Yes (list persists) | No (exhausted after one iteration) |
| Speed (one-time use) | Slightly slower to build fully | Faster to start, avoids full memory allocation |
| Use case | Need to access elements multiple times / need indexing | Large/infinite sequences, single-pass processing, pipelines |

> [!tip] When to Use a Generator
> Use generator expressions when working with **large datasets**, **streaming data**, or when you only need to iterate **once** — this avoids loading the entire result into memory.

---

## 6. Frozenset "Comprehension"

There's no dedicated syntax — build a `frozenset` by wrapping a set comprehension or generator with `frozenset()`.

```python
fs = frozenset(x**2 for x in range(10) if x % 2 == 0)
# frozenset({0, 4, 16, 36, 64})
```

### Why Use frozenset?
- **Immutable** version of a set — cannot be modified after creation
- **Hashable** — can be used as a dictionary key or as an element inside another set (regular sets cannot)
- Useful for representing fixed, constant collections (e.g., valid states, allowed values)

```python
# frozenset as a dict key (impossible with a regular set)
graph_lookup = {
    frozenset({'A', 'B'}): 5,
    frozenset({'B', 'C'}): 3
}
```

### Regular Set vs Frozenset

| Aspect | `set` | `frozenset` |
|---|---|---|
| Mutable | Yes (add/remove) | No |
| Hashable | No | Yes |
| Can be dict key | No | Yes |
| Can be set element | No | Yes |
| Comprehension syntax | `{ }` | None (must wrap `frozenset(...)`) |

---

## 7. Comparison Table — All Comprehension Types

| Type | Syntax | Result Type | Ordered? | Duplicates? | Mutable? |
|---|---|---|---|---|---|
| List Comprehension | `[expr for x in it]` | `list` | Yes | Allowed | Yes |
| Set Comprehension | `{expr for x in it}` | `set` | No | Removed | Yes |
| Dict Comprehension | `{k:v for x in it}` | `dict` | Yes (insertion) | Keys unique | Yes |
| Generator Expression | `(expr for x in it)` | `generator` | Yes (lazy) | Allowed | N/A (lazy) |
| "Tuple comprehension" | `tuple(expr for x in it)` | `tuple` | Yes | Allowed | **No** |
| "Frozenset comprehension" | `frozenset(expr for x in it)` | `frozenset` | No | Removed | **No** |

---

## 8. Nested Comprehensions (All Types Combined)

```python
# Nested list comprehension: transpose a matrix
matrix = [[1, 2, 3], [4, 5, 6]]
transposed = [[row[i] for row in matrix] for i in range(len(matrix[0]))]
# [[1, 4], [2, 5], [3, 6]]

# Set of tuples via nested comprehension
coord_set = {(x, y) for x in range(2) for y in range(2) if x != y}
# {(0, 1), (1, 0)}

# Dict comprehension with nested value expression
nested_dict = {x: [y for y in range(x)] for x in range(4)}
# {0: [], 1: [0], 2: [0, 1], 3: [0, 1, 2]}
```

---

## 9. Multiple `for` and `if` Clauses (Order Matters)

Comprehensions can chain multiple `for` and `if` clauses — evaluated **left to right, exactly like nested loops**.

```python
result = [x*y for x in range(3) for y in range(3) if x != y if (x+y) % 2 == 0]
```
Equivalent to:
```python
result = []
for x in range(3):
    for y in range(3):
        if x != y:
            if (x + y) % 2 == 0:
                result.append(x * y)
```

> [!tip]
> The **leftmost `for`** is the **outermost loop**. Each subsequent `for`/`if` is nested one level deeper.

---

## 10. The Walrus Operator (`:=`) in Comprehensions

Introduced in Python 3.8, allows assignment **within** a comprehension expression — useful to avoid recomputation.

```python
# Without walrus (computes square twice)
results = [y for x in data if (y := expensive_func(x)) > 10]

# y is computed once per x, then reused in both the condition and the expression
```

```python
data = [1, 2, 3, 4, 5]
big_squares = [square for x in data if (square := x**2) > 10]
# [16, 25]
```

---

## 11. Scoping Rules

> [!info] Important
> Comprehensions in Python 3 have their **own local scope** (like a mini-function). Variables defined inside (the loop variable) do **not leak** into the enclosing scope — unlike old Python 2 behavior.

```python
x = 100
squares = [x for x in range(5)]
print(x)  # 100 (unaffected — comprehension has its own scope)
```

But comprehensions **can access variables from the enclosing scope** (closures):
```python
multiplier = 10
scaled = [x * multiplier for x in range(5)]  # multiplier accessed from outer scope
```

---

## 12. Performance Considerations

> [!success] Why Comprehensions Are Often Faster
> - Implemented in C internally (optimized bytecode: `LIST_APPEND`, etc.) rather than repeated Python-level `.append()` calls
> - Avoid repeated attribute lookups (`list.append`) on every iteration
> - Fewer lines of interpreted bytecode overall

```python
import timeit

# Loop version
def loop_version():
    result = []
    for x in range(10000):
        result.append(x**2)
    return result

# Comprehension version
def comp_version():
    return [x**2 for x in range(10000)]

# Comprehension is typically ~30-50% faster than equivalent explicit loop
```

> [!warning] But Don't Over-Optimize Readability Away
> If a comprehension becomes deeply nested or hard to read (e.g., 3+ levels, multiple conditions), prefer a regular loop with clear variable names — **readability > micro-performance**.

---

## 13. Common Real-World Use Cases

```python
# Filter valid emails
emails = ["a@x.com", "invalid", "b@y.com"]
valid_emails = [e for e in emails if "@" in e]

# Flatten a list of lists
nested = [[1,2],[3,4],[5,6]]
flat = [n for sub in nested for n in sub]

# Word frequency skeleton using dict comprehension
words = ["apple", "banana", "apple", "cherry"]
word_set = {w for w in words}                     # unique words (set comp)
word_lengths = {w: len(w) for w in word_set}       # dict comp

# Remove None / falsy values
data = [1, None, 2, "", 3, 0, 4]
cleaned = [x for x in data if x]

# Create a mapping of index -> value
values = ['a', 'b', 'c']
index_map = {i: v for i, v in enumerate(values)}

# Extract unique domains from emails (set + frozenset)
emails2 = ["a@x.com", "b@x.com", "c@y.com"]
domains = frozenset(e.split('@')[1] for e in emails2)
# frozenset({'x.com', 'y.com'})
```

---

## 14. Best Practices ✅

> [!success] Do's
> - Use comprehensions for **simple, single-expression transformations/filters**
> - Prefer generator expressions for **large data** or **single-pass** operations to save memory
> - Use `frozenset()` when you need an **immutable, hashable** unique collection
> - Use dict comprehensions to cleanly **invert, filter, or remap** dictionaries
> - Use the walrus operator to avoid **recomputing expensive expressions**

> [!failure] Don'ts
> - Don't nest comprehensions more than 2 levels deep — it hurts readability drastically
> - Don't use comprehensions purely for **side effects** (e.g., `[print(x) for x in data]`) — use a plain `for` loop instead; this creates a wasted list
> - Don't assume `()` creates a tuple — it creates a generator
> - Don't forget that **set/dict comprehensions don't preserve arbitrary order** guarantees the way lists do
> - Don't overuse multiple chained `if`/`for` clauses when a helper function would be clearer

---

## 15. Quick Revision Summary

| You want... | Use |
|---|---|
| Ordered, duplicate-allowed collection | List comprehension `[ ]` |
| Unique, unordered collection | Set comprehension `{ }` |
| Key-value mapping | Dict comprehension `{k:v}` |
| Memory-efficient, lazy, single-pass sequence | Generator expression `( )` |
| Immutable tuple from a comprehension | `tuple(... for ... in ...)` |
| Immutable, hashable unique collection | `frozenset(... for ... in ...)` |

**Golden Rules:**
1. `()` ≠ tuple comprehension → it's a generator
2. No native frozenset comprehension → wrap with `frozenset()`
3. `if` after `for` = filter; `if/else` before `for` = transform
4. Comprehensions have their own scope (Python 3)
5. Prefer readability over cramming everything into one line

---

## Related Notes
- [[Generators and Iterators]]
- [[Python Data Structures Overview]]
- [[Lambda Functions and map/filter/reduce]]
- [[15 Memory Management in Python]]
- [[6 Exception Handling]]

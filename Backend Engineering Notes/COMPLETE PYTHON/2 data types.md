---
tags: [python, data-structures, memory, performance, notes]
title: Python Data Types Deep Dive
---

# Python Data Types Deep Dive
---

## Table of Contents

- [[#1. Strings (str)]]
- [[#2. Lists]]
- [[#3. Tuples]]
- [[#4. Sets]]
- [[#5. Dictionaries]]
- [[#6. Frozensets]]
- [[#7. Cross-Type Comparison]]
- [[#8. Which Method/Type is Faster and Why — Summary]]
- [[#9. Best Practices]]

---

## 1. Strings (`str`)

### 1.1 Memory Model & Internals

A Python `str` is an **immutable sequence of Unicode code points**. Since CPython 3.3 (PEP 393), strings use a **flexible internal representation** — the interpreter picks the smallest fixed-width storage that can hold every character in the string:

| Internal Representation | Used When | Bytes per Character |
|---|---|---|
| `Py_UCS1` (Latin-1) | All characters fit in 0–255 (e.g., ASCII, Latin-1 text) | 1 byte |
| `Py_UCS2` | Contains characters up to U+FFFF | 2 bytes |
| `Py_UCS4` | Contains characters beyond U+FFFF (e.g., emoji, rare scripts) | 4 bytes |

```python
import sys
print(sys.getsizeof("hello"))     # 54 bytes (ASCII, 1 byte/char + overhead)
print(sys.getsizeof("héllo"))     # 74 bytes (needs UCS2 due to 'é' beyond ASCII)
print(sys.getsizeof("😀"))         # 80 bytes (needs UCS4 for the emoji)
```

> [!note] Why This Matters
> This is why a string of pure ASCII text is far more memory-efficient than one containing even a single emoji or non-Latin character — the **entire string** upgrades to the wider representation, not just the special characters.

### 1.2 Immutability & String Interning

Strings are **immutable** — any "modification" (concatenation, `.upper()`, slicing, etc.) actually creates a **brand-new string object** in memory; the original is untouched.

```python
s = "hello"
s2 = s.upper()
print(s is s2)   # False — a completely new object was created
```

CPython also performs **string interning** — certain strings (short identifier-like strings, string literals in code) are cached and reused, so identical values can share the same memory object.

```python
a = "hello"
b = "hello"
print(a is b)     # True — both reference the SAME interned object (compile-time literal)

c = "".join(["h", "e", "l", "l", "o"])
print(a is c)      # False — built at runtime, not automatically interned

import sys
d = sys.intern(c)
print(a is d)      # True — after manual interning
```

> [!tip]
> Interning mainly benefits identifier-like strings (variable names, dict keys) and is why `is` comparisons on short strings can seem to "work" — but you should **always use `==` for string equality**, never rely on `is`, since interning behavior is an implementation detail, not a guarantee.

### 1.3 Why Immutability Matters for Performance

Because strings are immutable, repeatedly concatenating in a loop creates a **new string object every time**, copying all previous characters:

```python
# BAD — O(n²) overall: each += creates a full copy
result = ""
for word in words:
    result += word    # allocates a new string, copies old + new content each time

# GOOD — O(n) overall: builds a list, joins ONCE at the end
result = "".join(words)
```

> [!warning]
> `str.join()` is dramatically faster than repeated `+=` in a loop because it calculates the total required size **once** and performs a **single allocation and copy pass**, rather than reallocating and copying on every iteration.

### 1.4 Complete String Methods Reference

| Method | Return Type | Description |
|---|---|---|
| `s.capitalize()` | `str` | First character uppercase, rest lowercase |
| `s.casefold()` | `str` | Aggressive lowercasing for caseless matching (better than `.lower()` for some Unicode) |
| `s.center(width, fillchar=' ')` | `str` | Center-align string within given width |
| `s.count(sub, start, end)` | `int` | Count non-overlapping occurrences of substring |
| `s.encode(encoding='utf-8')` | `bytes` | Encode string into bytes |
| `s.endswith(suffix)` | `bool` | Check if string ends with given suffix |
| `s.expandtabs(tabsize=8)` | `str` | Replace tabs with spaces |
| `s.find(sub, start, end)` | `int` | Index of first occurrence, or `-1` if not found |
| `s.format(*args, **kwargs)` | `str` | Format string using `{}` placeholders |
| `s.format_map(mapping)` | `str` | Like `.format()` but takes a mapping directly |
| `s.index(sub, start, end)` | `int` | Like `.find()` but raises `ValueError` if not found |
| `s.isalnum()` | `bool` | True if all characters are alphanumeric |
| `s.isalpha()` | `bool` | True if all characters are alphabetic |
| `s.isascii()` | `bool` | True if all characters are ASCII |
| `s.isdecimal()` | `bool` | True if all characters are decimal digits |
| `s.isdigit()` | `bool` | True if all characters are digits |
| `s.isidentifier()` | `bool` | True if string is a valid Python identifier |
| `s.islower()` | `bool` | True if all cased characters are lowercase |
| `s.isnumeric()` | `bool` | True if all characters are numeric |
| `s.isprintable()` | `bool` | True if all characters are printable |
| `s.isspace()` | `bool` | True if all characters are whitespace |
| `s.istitle()` | `bool` | True if string is titlecased |
| `s.isupper()` | `bool` | True if all cased characters are uppercase |
| `s.join(iterable)` | `str` | Join elements of an iterable using `s` as separator |
| `s.ljust(width, fillchar=' ')` | `str` | Left-justify within given width |
| `s.lower()` | `str` | Convert to lowercase |
| `s.lstrip(chars=None)` | `str` | Strip leading whitespace/characters |
| `s.maketrans(x, y, z)` | `dict` (static method) | Build a translation table for `.translate()` |
| `s.partition(sep)` | `tuple` | Split into `(before, sep, after)` at first occurrence |
| `s.removeprefix(prefix)` | `str` | Remove prefix if present (Python 3.9+) |
| `s.removesuffix(suffix)` | `str` | Remove suffix if present (Python 3.9+) |
| `s.replace(old, new, count=-1)` | `str` | Replace occurrences of a substring |
| `s.rfind(sub, start, end)` | `int` | Like `.find()` but searches from the right |
| `s.rindex(sub, start, end)` | `int` | Like `.index()` but searches from the right |
| `s.rjust(width, fillchar=' ')` | `str` | Right-justify within given width |
| `s.rpartition(sep)` | `tuple` | Like `.partition()` but splits at the LAST occurrence |
| `s.rsplit(sep=None, maxsplit=-1)` | `list` | Split from the right |
| `s.rstrip(chars=None)` | `str` | Strip trailing whitespace/characters |
| `s.split(sep=None, maxsplit=-1)` | `list` | Split string into a list of substrings |
| `s.splitlines(keepends=False)` | `list` | Split at line boundaries |
| `s.startswith(prefix)` | `bool` | Check if string starts with given prefix |
| `s.strip(chars=None)` | `str` | Strip leading and trailing whitespace/characters |
| `s.swapcase()` | `str` | Swap uppercase ↔ lowercase |
| `s.title()` | `str` | Titlecase the string ("Hello World") |
| `s.translate(table)` | `str` | Apply a translation table (from `.maketrans()`) |
| `s.upper()` | `str` | Convert to uppercase |
| `s.zfill(width)` | `str` | Pad with leading zeros |

### 1.5 Method Performance Comparison

| Comparison                                                  | Winner       | Why                                                                                                          |
| ----------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------ |
| `s.replace()` vs `re.sub()` for simple substitution         | `.replace()` | No regex engine overhead — a straightforward memory scan/copy                                                |
| `s += x` in a loop vs `"".join(list)`                       | `.join()`    | Avoids O(n²) repeated allocation/copy; single-pass allocation                                                |
| `s.find()` vs `s.index()` when substring might be absent    | `.find()`    | `.index()` raises an exception on failure — exception handling has overhead; `.find()` just returns -1       |
| `in` operator (`"x" in s`) vs `.find()` for existence check | `in`         | Slightly more direct C-level check; also more readable                                                       |
| f-strings vs `.format()` vs `%` formatting                  | f-strings    | Evaluated at compile time into efficient bytecode; `.format()` and `%` involve more runtime parsing overhead |

```python
# Fast substring existence check
if "sub" in my_string:      # preferred
    ...

# Slower / more verbose equivalent
if my_string.find("sub") != -1:
    ...
```

---

## 2. Lists

### 2.1 Memory Model & Internals

A Python `list` is implemented as a **dynamic array** (like C++'s `vector` or Java's `ArrayList`) — internally, it's a contiguous array of **pointers** to Python objects (not the objects themselves inline).

```
list object header
   │
   ▼
 ┌────┬────┬────┬────┬────┐
 │ptr0│ptr1│ptr2│ptr3│... │   ← array of pointers (contiguous memory)
 └────┴────┴────┴────┴────┘
   │     │     │
   ▼     ▼     ▼
 [10]  ["hi"] [3.14]         ← actual objects, scattered anywhere in memory
```

This is why a Python list can hold **mixed types** — it's just storing references/pointers, not the values inline.

### 2.2 Dynamic Resizing (Over-Allocation)

Lists **over-allocate** memory to make appends efficient. When a list's internal array is full and a new element is appended, CPython allocates a **larger** block (roughly ~1.125x plus a small constant, growth pattern is not a simple doubling) and copies existing pointers over.

```python
import sys

lst = []
prev_size = sys.getsizeof(lst)
for i in range(10):
    lst.append(i)
    size = sys.getsizeof(lst)
    if size != prev_size:
        print(f"len={len(lst)}: size grew to {size} bytes")
        prev_size = size
```

This shows that the underlying array doesn't grow by exactly 1 slot per append — it grows in **chunks**, so most `.append()` calls are O(1) amortized (no resize needed), and only occasionally does a resize (with an O(n) copy) occur.

> [!note] Amortized O(1) Append
> Because resizes happen exponentially less often as the list grows, the **average** cost per append across many operations is O(1), even though any individual append that triggers a resize is O(n). This is called **amortized constant time**.

### 2.3 Why Insertion/Deletion at the Front is Slow

```python
lst = [1, 2, 3, 4, 5]
lst.insert(0, 0)   # O(n) — EVERY existing element must shift right by one slot
del lst[0]           # O(n) — every remaining element shifts left by one slot
lst.append(6)         # O(1) amortized — no shifting needed
lst.pop()              # O(1) — removes from the end, no shifting needed
```

```
Before insert(0, 0):  [1, 2, 3, 4, 5]
                        │  │  │  │
                        ▼  ▼  ▼  ▼
After insert(0, 0):  [0, 1, 2, 3, 4, 5]   ← everything shifted right by one slot
```

### 2.4 Complete List Methods Reference

| Method | Return Type | Time Complexity | Description |
|---|---|---|---|
| `lst.append(x)` | `None` | O(1) amortized | Add element to the end |
| `lst.extend(iterable)` | `None` | O(k) for k new elements | Append all elements from another iterable |
| `lst.insert(i, x)` | `None` | O(n) | Insert element at index `i`, shifting others |
| `lst.remove(x)` | `None` | O(n) | Remove first occurrence of value `x`; raises `ValueError` if absent |
| `lst.pop(i=-1)` | element's type | O(1) at end, O(n) elsewhere | Remove and return element at index (default: last) |
| `lst.clear()` | `None` | O(n) | Remove all elements |
| `lst.index(x, start, end)` | `int` | O(n) | Index of first occurrence of `x`; raises `ValueError` if absent |
| `lst.count(x)` | `int` | O(n) | Count occurrences of `x` |
| `lst.sort(key=None, reverse=False)` | `None` | O(n log n) | Sort the list **in place** |
| `lst.reverse()` | `None` | O(n) | Reverse the list **in place** |
| `lst.copy()` | `list` | O(n) | Shallow copy of the list |

### 2.5 Method Performance Comparison

| Comparison | Winner | Why |
|---|---|---|
| `.append(x)` vs `.insert(0, x)` | `.append()` | O(1) amortized vs O(n) — no shifting required |
| `.pop()` vs `.pop(0)` | `.pop()` (no args) | Removing the last element needs no shifting; removing the first shifts all remaining elements left |
| `list.sort()` vs `sorted(list)` | `.sort()` (if you don't need the original preserved) | `.sort()` mutates in place, no extra list allocation; `sorted()` must build a new list, using more memory |
| `+` (concatenation) vs `.extend()` for building up a list | `.extend()` | `+` creates a brand-new list each time; `.extend()` grows the existing list's buffer in place |
| Membership test `x in list` vs `x in set` | `set` (for large collections) | List membership is O(n) linear scan; set membership is O(1) average via hashing |
| List comprehension vs manual `for` loop with `.append()` | List comprehension | Optimized bytecode (`LIST_APPEND` in a tight loop without the general attribute-lookup overhead of calling `.append()` repeatedly) |

```python
# Fast — list comprehension
squares = [x**2 for x in range(1000)]

# Slower — same result, more overhead per iteration
squares = []
for x in range(1000):
    squares.append(x**2)
```

### 2.6 Timsort — Python's Sorting Algorithm

`list.sort()` and `sorted()` both use **Timsort**, a hybrid stable sorting algorithm combining **merge sort** and **insertion sort**, designed to perform well on real-world data (which often has existing runs of order).

- **Time complexity**: O(n log n) worst case, O(n) best case (already sorted data).
- **Stable**: equal elements retain their relative order.
- **Adaptive**: detects and exploits existing sorted "runs" in the data to reduce work.

---

## 3. Tuples

### 3.1 Memory Model & Internals

A `tuple` is also an array of pointers, structurally similar to a list — **but immutable and fixed-size**. Because its size never changes after creation, CPython can allocate the **exact** amount of memory needed, with **no over-allocation**.

```python
import sys
print(sys.getsizeof([1, 2, 3]))   # 88 bytes (list — has extra capacity reserved)
print(sys.getsizeof((1, 2, 3)))   # 64 bytes (tuple — allocated exactly, no slack)
```

> [!note] Why Tuples Are More Memory-Efficient
> Lists reserve extra capacity to make future appends cheap; tuples never grow, so Python skips that reservation entirely — resulting in a smaller memory footprint for the same elements.

### 3.2 Immutability

```python
t = (1, 2, 3)
t[0] = 99   # TypeError: 'tuple' object does not support item assignment
```

> [!warning] Tuples Aren't FULLY Immutable if They Contain Mutable Elements
> A tuple's own structure (the sequence of references) can't change, but if it contains a **mutable object** (like a list), that inner object can still be mutated:
> ```python
> t = ([1, 2], "hello")
> t[0].append(3)   # totally legal! the tuple still "points" to the same list object
> print(t)  # ([1, 2, 3], 'hello')
> ```

### 3.3 Why Tuples Can Be Faster Than Lists

- **No dynamic resizing logic** — allocated once at exact size, so creation is often faster.
- **Can be used as dictionary keys / set elements** — since they're hashable (as long as all their contents are hashable), unlike lists.
- **Constant folding** — CPython can sometimes precompute tuple literals containing only constants **at compile time**, embedding them directly into bytecode; lists cannot benefit from this due to mutability.

```python
import dis
dis.dis(compile("(1, 2, 3)", "", "eval"))
# Shows a single LOAD_CONST for the whole tuple — precomputed at compile time!

dis.dis(compile("[1, 2, 3]", "", "eval"))
# Shows individual LOAD_CONST for each element, then BUILD_LIST — built at runtime
```

### 3.4 Complete Tuple Methods Reference

Tuples support only **two** methods (since they're immutable, there's no add/remove/sort/etc.):

| Method | Return Type | Time Complexity | Description |
|---|---|---|---|
| `t.count(x)` | `int` | O(n) | Count occurrences of `x` |
| `t.index(x, start, end)` | `int` | O(n) | Index of first occurrence of `x`; raises `ValueError` if absent |

> [!note]
> Beyond these two, tuples support the general sequence operations available to any sequence type: indexing (`t[0]`), slicing (`t[1:3]`), concatenation (`t1 + t2`), repetition (`t * 3`), membership (`x in t`), iteration, `len()`, `min()`, `max()`, and unpacking (`a, b, c = t`).

### 3.5 Named Tuples (Bonus)

`collections.namedtuple` (or `typing.NamedTuple`) gives tuples **named field access** while keeping all the memory/performance benefits of regular tuples.

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
print(p.x, p.y)     # 3 4
print(p[0], p[1])    # 3 4 — still works as a regular tuple too
```

---

## 4. Sets

### 4.1 Memory Model & Internals

A `set` is implemented as a **hash table** — internally similar to a dictionary but storing only keys (no associated values). Elements are placed into "buckets" (slots) determined by `hash(element) % table_size`, enabling average **O(1)** membership testing, insertion, and deletion.

```
Hash table (simplified):
Index:  0        1        2        3        4        5        6        7
       [ ]     ["cat"]   [ ]    ["dog"]    [ ]       [ ]    ["bird"]  [ ]
```

### 4.2 How Hashing Determines Placement

```python
s = {"cat", "dog", "bird"}
# Internally, each element's hash() value determines its bucket index
print(hash("cat"))    # some large integer, e.g. -2036258821
```

When you check `"cat" in s`, Python computes `hash("cat")`, jumps **directly** to the corresponding bucket, and checks for equality — no need to scan every element, unlike a list.

### 4.3 Collision Handling

When two different elements hash to the same bucket (a "collision"), CPython uses **open addressing** — it probes a sequence of alternative slots (using a specific perturbation formula) until it finds an empty one.

```python
# Conceptually:
index = hash(x) % table_size
while table[index] is occupied and table[index] != x:
    index = next_probe(index)   # follow the probing sequence
```

> [!note]
> This is why sets require elements to be **hashable** (must implement `__hash__` and be immutable in practice) — mutable objects like lists cannot be set elements, because their hash would change if mutated, breaking the bucket placement.

### 4.4 Why Sets Resize (Load Factor)

Like dictionaries, sets automatically **resize** (grow their internal table) once they get sufficiently full (their "load factor" crosses a threshold, typically around 2/3 full in CPython), to keep collision rates — and therefore lookup times — low.

### 4.5 Complete Set Methods Reference

| Method | Return Type | Time Complexity | Description |
|---|---|---|---|
| `s.add(x)` | `None` | O(1) average | Add element `x` to the set |
| `s.remove(x)` | `None` | O(1) average | Remove `x`; raises `KeyError` if absent |
| `s.discard(x)` | `None` | O(1) average | Remove `x` if present; does nothing if absent (no error) |
| `s.pop()` | element's type | O(1) | Remove and return an **arbitrary** element |
| `s.clear()` | `None` | O(n) | Remove all elements |
| `s.copy()` | `set` | O(n) | Shallow copy |
| `s.union(*others)` | `set` | O(len(s) + sum of others) | All elements from `s` and all others (also via `\|`) |
| `s.update(*others)` | `None` | O(sum of others) | Add elements from other iterables **in place** (also via `\|=`) |
| `s.intersection(*others)` | `set` | O(min(len(s), len(other))) | Elements common to `s` and all others (also via `&`) |
| `s.intersection_update(*others)` | `None` | similar | Keep only common elements, in place (also via `&=`) |
| `s.difference(*others)` | `set` | O(len(s)) | Elements in `s` but not in others (also via `-`) |
| `s.difference_update(*others)` | `None` | O(len(other)) | Remove elements found in others, in place (also via `-=`) |
| `s.symmetric_difference(other)` | `set` | O(len(s) + len(other)) | Elements in either set, but not both (also via `^`) |
| `s.symmetric_difference_update(other)` | `None` | similar | In-place symmetric difference (also via `^=`) |
| `s.issubset(other)` | `bool` | O(len(s)) | True if `s` ⊆ `other` (also via `<=`) |
| `s.issuperset(other)` | `bool` | O(len(other)) | True if `s` ⊇ `other` (also via `>=`) |
| `s.isdisjoint(other)` | `bool` | O(min(len(s), len(other))) | True if `s` and `other` share no elements |

### 4.6 Method Performance Comparison

| Comparison | Winner | Why |
|---|---|---|
| `x in set` vs `x in list` | `set` | O(1) average hash lookup vs O(n) linear scan |
| `.remove()` vs `.discard()` when uncertain if element exists | `.discard()` | Avoids the overhead of exception handling/try-except when absence is expected/common |
| `set(list)` deduplication vs manual loop with a list and `in` check | `set(list)` | Manual approach is O(n²) (checking `in` on a growing list each time); using a set is O(n) overall |
| `set` intersection (`&`) vs manually looping and checking membership | `&` operator | Implemented in optimized C, iterates the **smaller** set and checks membership in the larger — highly optimized |

```python
# Fast deduplication — O(n)
unique = list(set(my_list))

# Slow deduplication — O(n²)
unique = []
for item in my_list:
    if item not in unique:   # 'in' on a list is O(n), done n times = O(n²)
        unique.append(item)
```

> [!warning]
> `set(list)` deduplication does **not preserve order** (sets are unordered). If order matters, use `dict.fromkeys(my_list)` instead — dicts preserve insertion order (Python 3.7+) while still deduplicating via hashing.
> ```python
> unique_ordered = list(dict.fromkeys(my_list))
> ```

---

## 5. Dictionaries

### 5.1 Memory Model & Internals

A `dict` is a **hash table** mapping keys to values, using the same hashing/bucket principles as sets — but each slot stores a **(key, hash, value)** triple instead of just a key.

Since Python 3.6+ (an implementation detail formalized in 3.7), dictionaries maintain **insertion order** via a clever two-part internal structure:

```
1. A "sparse" array of indices (into the dense table below) — sized for fast hash lookups
2. A "dense" array storing (hash, key, value) entries IN INSERTION ORDER
```

```
Sparse index table:      Dense entry table (insertion order preserved):
[ -, 1, -, 0, -, 2 ]      [0]: (hash("b"), "b", 2)
                          [1]: (hash("a"), "a", 1)
                          [2]: (hash("c"), "c", 3)
```

This "compact dict" design (introduced in Python 3.6) both **saves memory** (the sparse index array uses smaller integer types) and **preserves insertion order** as a side effect, which was later made an official language guarantee.

### 5.2 Hashing and Lookup

```python
d = {"name": "Amit", "age": 25}
# d["name"] lookup:
# 1. Compute hash("name")
# 2. Use hash to find the slot in the sparse index table
# 3. Follow that index into the dense table
# 4. Verify the key actually matches (handles collisions)
# 5. Return the associated value
```

This gives **O(1) average-case** lookup, insertion, and deletion — regardless of dictionary size (until a resize is triggered).

### 5.3 Why Dictionary Keys Must Be Hashable

Just like set elements, dict keys must be hashable (implement `__hash__`, and be effectively immutable) — this is why lists can't be dict keys, but tuples (of hashable elements) can.

```python
d = {(1, 2): "point A"}   # fine — tuple is hashable
d2 = {[1, 2]: "point A"}  # TypeError: unhashable type: 'list'
```

### 5.4 Complete Dictionary Methods Reference

| Method | Return Type | Time Complexity | Description |
|---|---|---|---|
| `d.get(key, default=None)` | value's type or `default` | O(1) average | Get value for `key`; returns `default` instead of raising if missing |
| `d.setdefault(key, default=None)` | value's type | O(1) average | Get value for `key`; if missing, insert `key: default` and return `default` |
| `d.pop(key, default)` | value's type | O(1) average | Remove `key` and return its value; returns `default` (or raises `KeyError`) if missing |
| `d.popitem()` | `tuple (key, value)` | O(1) | Remove and return the **last inserted** key-value pair (LIFO order, Python 3.7+) |
| `d.update(other)` | `None` | O(len(other)) | Merge another dict/iterable of pairs into `d`, in place |
| `d.clear()` | `None` | O(n) | Remove all items |
| `d.copy()` | `dict` | O(n) | Shallow copy |
| `d.keys()` | `dict_keys` (view) | O(1) to create | Dynamic view of all keys |
| `d.values()` | `dict_values` (view) | O(1) to create | Dynamic view of all values |
| `d.items()` | `dict_items` (view) | O(1) to create | Dynamic view of all (key, value) pairs |
| `dict.fromkeys(iterable, value=None)` | `dict` (class method) | O(n) | Build a new dict with given keys, all mapped to the same value |

### 5.5 Views Are Dynamic (Live) — Not Snapshots

```python
d = {"a": 1, "b": 2}
keys = d.keys()
print(list(keys))    # ['a', 'b']

d["c"] = 3
print(list(keys))     # ['a', 'b', 'c'] — the view reflects the CURRENT state automatically
```

> [!note]
> `.keys()`, `.values()`, `.items()` return **view objects**, not lists — they're memory-cheap and stay synchronized with the dict. Wrap in `list(...)` if you need a fixed snapshot.

### 5.6 Method Performance Comparison

| Comparison | Winner | Why |
|---|---|---|
| `d[key]` vs `d.get(key)` when key is known to exist | `d[key]` | Slightly faster — `.get()` has extra function-call and default-handling overhead |
| `d.get(key, default)` vs `try/except KeyError` for a possibly-missing key | `.get()` (if misses are frequent) | Exceptions have real overhead when actually raised; `.get()` avoids that entirely. (If misses are RARE, try/except can actually be faster since the try block itself is nearly free when no exception occurs.) |
| `key in d` vs `key in d.keys()` | `key in d` | Equivalent performance in CPython (both are O(1) hash lookups) but `in d` avoids creating/considering a view object, and is the idiomatic form |
| `dict` vs `list` for lookups by identifier | `dict` | O(1) average hash lookup vs O(n) linear scan through a list of pairs |
| `d.setdefault()` vs manual `if key not in d: d[key] = default` | `.setdefault()` | Single hash lookup instead of two (one for the `in` check, one for the assignment) |

```python
# Efficient counting pattern using setdefault
counts = {}
for word in words:
    counts[word] = counts.get(word, 0) + 1

# Even better — collections.Counter is implemented in optimized C
from collections import Counter
counts = Counter(words)
```

---

## 6. Frozensets

### 6.1 Memory Model & Internals

A `frozenset` is the **immutable counterpart to `set`** — identical hash-table internals (same bucket/probing mechanics), but once created, its contents can never change. Because it's immutable, **`frozenset` itself is hashable**, allowing it to be used as a dict key or as an element of another set — something a regular `set` cannot do.

```python
s = {1, 2, 3}
fs = frozenset(s)

d = {fs: "immutable set as a key"}   # works! frozenset is hashable
d2 = {s: "regular set as a key"}      # TypeError: unhashable type: 'set'
```

### 6.2 Why Frozensets Can Be Slightly More Memory-Efficient

Similar to the tuple/list relationship: since a `frozenset`'s size is fixed at creation, CPython can avoid reserving the extra "growth headroom" a mutable `set` keeps for future `.add()` calls.

```python
import sys
print(sys.getsizeof({1, 2, 3}))            # e.g. 216 bytes (set — has growth capacity)
print(sys.getsizeof(frozenset({1, 2, 3}))) # e.g. 216 bytes for small sets; the gap widens more noticeably for larger collections, and frozensets also skip mutation-related bookkeeping
```

### 6.3 Complete Frozenset Methods Reference

Frozensets support all the **non-mutating** methods that regular sets do — everything except `.add()`, `.remove()`, `.discard()`, `.pop()`, `.clear()`, and the `_update()` in-place variants.

| Method | Return Type | Time Complexity | Description |
|---|---|---|---|
| `fs.union(*others)` | `frozenset` | O(len(fs) + sum of others) | All elements from `fs` and others (also via `\|`) |
| `fs.intersection(*others)` | `frozenset` | O(min(len(fs), len(other))) | Common elements (also via `&`) |
| `fs.difference(*others)` | `frozenset` | O(len(fs)) | Elements in `fs` but not others (also via `-`) |
| `fs.symmetric_difference(other)` | `frozenset` | O(len(fs) + len(other)) | Elements in either, not both (also via `^`) |
| `fs.copy()` | `frozenset` | O(1) | Returns the SAME object (since it's immutable, no real copy needed!) |
| `fs.issubset(other)` | `bool` | O(len(fs)) | True if `fs` ⊆ `other` |
| `fs.issuperset(other)` | `bool` | O(len(other)) | True if `fs` ⊇ `other` |
| `fs.isdisjoint(other)` | `bool` | O(min(len(fs), len(other))) | True if no elements in common |

> [!tip] Interesting Detail
> `frozenset.copy()` actually just **returns the same object** (increments the reference count) rather than creating a genuine duplicate — since the object can never change, there's no risk in sharing it, making `.copy()` an O(1) no-op essentially.

### 6.4 When to Use Frozenset Over Set

| Use Case | Why Frozenset |
|---|---|
| Dictionary keys that represent a group/combination | Needs to be hashable; regular sets aren't |
| Set of sets (nested sets) | Inner sets must be hashable to belong to an outer set |
| Representing a fixed, constant collection that shouldn't change | Communicates immutability intent; prevents accidental mutation |
| Function default arguments (avoiding mutable default pitfalls) | Immutable, so safe as a default value |

```python
# Example: grouping students by the FROZENSET of courses they're enrolled in
groups = {
    frozenset({"Math", "Physics"}): ["Amit", "Priya"],
    frozenset({"Biology", "Chemistry"}): ["Rahul"],
}
```

---

## 7. Cross-Type Comparison

### 7.1 Mutability & Hashability

| Type | Mutable? | Hashable? | Ordered? |
|---|---|---|---|
| `str` | No | Yes | Yes (sequence order) |
| `list` | Yes | No | Yes (sequence order) |
| `tuple` | No (but can contain mutables) | Yes, if all elements are hashable | Yes (sequence order) |
| `set` | Yes | No | No (arbitrary iteration order) |
| `dict` | Yes | No | Yes (insertion order, Python 3.7+) |
| `frozenset` | No | Yes | No (arbitrary iteration order) |

### 7.2 Time Complexity Comparison Table

| Operation | `list` | `tuple` | `set` | `dict` | `str` |
|---|---|---|---|---|---|
| Indexing (`x[i]`) | O(1) | O(1) | N/A | N/A | O(1) |
| Membership (`x in y`) | O(n) | O(n) | O(1) avg | O(1) avg (keys) | O(n) |
| Append/Add | O(1) amortized | N/A (immutable) | O(1) avg | O(1) avg | N/A (immutable) |
| Insert at front | O(n) | N/A | N/A (unordered) | N/A (unordered) | N/A (immutable) |
| Delete | O(n) (or O(1) at end) | N/A | O(1) avg | O(1) avg | N/A (immutable) |
| Search by value/key | O(n) | O(n) | O(1) avg | O(1) avg (by key) | O(n) |
| Concatenation | O(k) for k new elements | O(n+m), new tuple | O(n+m) via union | O(m) via update | O(n+m), new string |

### 7.3 Memory Footprint (Illustrative, CPython, 64-bit)

```python
import sys
print(sys.getsizeof([1, 2, 3]))              # ~88 bytes — list (extra capacity)
print(sys.getsizeof((1, 2, 3)))              # ~64 bytes — tuple (exact size)
print(sys.getsizeof({1, 2, 3}))              # ~216 bytes — set (hash table overhead)
print(sys.getsizeof(frozenset({1, 2, 3})))   # ~216 bytes — similar table, no growth slack
print(sys.getsizeof({"a": 1, "b": 2}))       # ~232 bytes — dict (hash table + values)
```

> [!note]
> Sets/dicts have higher **per-element overhead** than lists/tuples because of the hash table infrastructure (buckets, collision handling) — but this overhead buys O(1) average lookup instead of O(n).

---

## 8. Which Method/Type is Faster and Why — Summary

### 8.1 "Should I Use a List or a Set/Dict for Membership Testing?"

**Always a set/dict for repeated membership checks.** A list requires scanning every element (O(n)) until a match is found or the list is exhausted. A set/dict computes a hash and jumps directly to the relevant bucket (O(1) average) — the performance gap grows dramatically as the collection size increases.

```python
import time

big_list = list(range(1_000_000))
big_set = set(big_list)

# List membership: scans up to a million elements
start = time.perf_counter()
999_999 in big_list
print(f"List: {time.perf_counter() - start:.6f}s")   # noticeably slow

# Set membership: near-instant hash lookup
start = time.perf_counter()
999_999 in big_set
print(f"Set: {time.perf_counter() - start:.6f}s")     # dramatically faster
```

### 8.2 "Should I Use a Tuple or List for Fixed Data?"

**Tuple**, when the data won't change — slightly less memory (no over-allocation), marginally faster to create (no resizing logic needed), and can double as a dict key/set element if all contents are hashable.

### 8.3 "Should I Use `frozenset` or `set`?"

**`set`** for anything you'll mutate. **`frozenset`** when you need hashability (as a dict key, or nested inside another set) or want to signal/enforce that the collection is fixed.

### 8.4 "Why is String Concatenation with `+=` in a Loop Slow?"

Because strings are immutable — every `+=` allocates an entirely new string and copies both the old content and the new piece into it. Across n iterations, this becomes O(n²) total work. `"".join(list_of_strings)` computes the final size once and does a single allocation + copy pass, making it O(n) overall.

### 8.5 "Why Are Dict/Set Insertions Usually O(1), But Sometimes Slow?"

Because of periodic **resizing**. As more elements are added, the hash table's load factor increases, raising collision probability and thus lookup/insertion time. Once a threshold is crossed, Python allocates a larger table and **rehashes every existing element** into it — an O(n) operation, but one that happens rarely enough (amortized across many O(1) insertions) that the average cost per insertion remains O(1).

---

## 9. Best Practices

- ✅ Use **lists** for ordered, mutable collections where you'll be appending/iterating; avoid frequent front-insertion/deletion.
- ✅ Use **tuples** for fixed, heterogeneous, or "record-like" data (e.g., coordinates, database rows) — and whenever you need a hashable sequence.
- ✅ Use **sets** for membership testing, deduplication, and set algebra (union/intersection/difference) — never for anything requiring order.
- ✅ Use **dictionaries** for key-based lookups; rely on insertion-order guarantee (Python 3.7+) only when you explicitly want that behavior documented, not as an implicit assumption in older-compatible code.
- ✅ Use **frozensets** when a set needs to be a dict key, a set element, or simply must never change.
- ✅ Prefer `"".join(list)` over repeated `+=` for building strings in loops.
- ✅ Prefer `x in some_set` over `x in some_list` for any membership check performed more than a handful of times.
- ✅ Use `collections.Counter`, `collections.defaultdict`, and `collections.namedtuple` where applicable — they're implemented in optimized C and cover extremely common patterns.
- ✅ Remember: `list`/`dict`/`set` are **mutable and unhashable**; `tuple` (of hashables)/`frozenset`/`str` are **immutable and hashable** — this single distinction drives most "which type should I use here?" decisions.

---

*Tags:* #python #data-structures #memory #performance #internals #cpython
---
tags: [python, recursion, algorithms, data-structures, notes]
title: Recursion in Python
---

# Recursion in Python

A deep-dive reference on recursion — how it works internally, its types, and how to apply it across different data types (numbers, strings, lists, dictionaries, trees, and graphs).

---

## Table of Contents

- [[#1. What is Recursion]]
- [[#2. How Recursion Works Internally (The Call Stack)]]
- [[#3. Anatomy of a Recursive Function]]
- [[#4. Types of Recursion]]
- [[#5. Recursion vs Iteration]]
- [[#6. Recursion with Numbers]]
- [[#7. Recursion with Strings]]
- [[#8. Recursion with Lists]]
- [[#9. Recursion with Tuples & Sets]]
- [[#10. Recursion with Dictionaries]]
- [[#11. Recursion with Nested/Mixed Data Structures]]
- [[#12. Recursion with Trees]]
- [[#13. Recursion with Graphs]]
- [[#14. Recursion Limit & Stack Overflow]]
- [[#15. Tail Recursion & Why Python Doesn't Optimize It]]
- [[#16. Memoization & Optimizing Recursion]]
- [[#17. Common Pitfalls]]
- [[#18. Best Practices]]
- [[#19. Quick Reference Cheat Sheet]]

---

## 1. What is Recursion?

**Recursion** is a technique where a function solves a problem by calling **itself** with a smaller/simpler version of the same problem, until it reaches a case simple enough to solve directly.

Every recursive solution has two essential parts:

1. **Base Case** — the condition under which the function stops calling itself and returns a direct answer. Without this, recursion never terminates.
2. **Recursive Case** — the part where the function calls itself with a smaller/simpler input, moving it closer to the base case.

```python
def factorial(n):
    if n == 0:            # base case
        return 1
    return n * factorial(n - 1)   # recursive case

print(factorial(5))  # 120
```

> [!note] Mental Model
> Recursion mirrors **mathematical induction**: solve the smallest case directly, and assume the function already correctly solves "n-1" to build the solution for "n".

---

## 2. How Recursion Works Internally (The Call Stack)

Each function call — recursive or not — creates a new **stack frame** containing its local variables, parameters, and the return address. These frames are pushed onto the **call stack** and popped off as functions return.

### Tracing `factorial(3)`

```
Call:                          Stack (top → bottom):
factorial(3)                   factorial(3)
  factorial(2)                 factorial(2)
                                factorial(3)
    factorial(1)                factorial(1)
                                 factorial(2)
                                 factorial(3)
      factorial(0) → returns 1  factorial(0)
                                 factorial(1)
                                 factorial(2)
                                 factorial(3)
```

### Unwinding (Popping the Stack)

```
factorial(0) returns 1
factorial(1) returns 1 * 1 = 1
factorial(2) returns 2 * 1 = 2
factorial(3) returns 3 * 2 = 6
```

```python
def factorial(n, depth=0):
    print("  " * depth + f"Calling factorial({n})")
    if n == 0:
        print("  " * depth + "Base case hit, returning 1")
        return 1
    result = n * factorial(n - 1, depth + 1)
    print("  " * depth + f"Returning {result} for factorial({n})")
    return result

factorial(3)
```

Output:
```
Calling factorial(3)
  Calling factorial(2)
    Calling factorial(1)
      Calling factorial(0)
      Base case hit, returning 1
    Returning 1 for factorial(1)
  Returning 2 for factorial(2)
Returning 6 for factorial(3)
```

> [!warning]
> Each recursive call consumes **memory** (a new stack frame). Very deep recursion can exhaust the stack and raise `RecursionError: maximum recursion depth exceeded`. See [[#14. Recursion Limit & Stack Overflow]].

---

## 3. Anatomy of a Recursive Function

```python
def recursive_function(input):
    if base_condition(input):        # 1. Base case check
        return base_result

    smaller_input = reduce(input)    # 2. Shrink the problem
    result = recursive_function(smaller_input)  # 3. Recursive call
    return combine(input, result)    # 4. Combine result with current step
```

| Step | Purpose |
|---|---|
| **Base case** | Stops the recursion; prevents infinite calls |
| **Shrinking the input** | Ensures each call moves closer to the base case |
| **Recursive call** | The function invoking itself |
| **Combining step** | How the sub-result is used to build the current level's answer |

---

## 4. Types of Recursion

### A. Direct Recursion
A function calls **itself** directly.

```python
def countdown(n):
    if n <= 0:
        print("Done!")
        return
    print(n)
    countdown(n - 1)
```

### B. Indirect (Mutual) Recursion
Two or more functions call **each other** in a cycle.

```python
def is_even(n):
    if n == 0:
        return True
    return is_odd(n - 1)

def is_odd(n):
    if n == 0:
        return False
    return is_even(n - 1)

print(is_even(10))  # True
```

### C. Linear Recursion
The function makes **one** recursive call per invocation.

```python
def sum_n(n):
    if n == 0:
        return 0
    return n + sum_n(n - 1)   # single recursive call
```

### D. Tree (Multiple/Branching) Recursion
The function makes **more than one** recursive call per invocation, forming a tree of calls.

```python
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)   # TWO recursive calls
```

Call tree for `fibonacci(4)`:
```
                fib(4)
              /        \
          fib(3)        fib(2)
         /     \        /    \
     fib(2)   fib(1)  fib(1) fib(0)
     /    \
  fib(1) fib(0)
```

> [!warning]
> Tree recursion can be **exponentially slow** due to repeated recomputation of the same sub-problems (notice `fib(2)` and `fib(1)` computed multiple times). See [[#16. Memoization & Optimizing Recursion]].

### E. Tail Recursion
The recursive call is the **very last operation** in the function — nothing happens after it returns.

```python
def factorial_tail(n, accumulator=1):
    if n == 0:
        return accumulator
    return factorial_tail(n - 1, n * accumulator)   # tail call — nothing after it
```

Compare with non-tail recursion:
```python
def factorial(n):
    if n == 0:
        return 1
    return n * factorial(n - 1)   # multiplication happens AFTER the recursive call returns — NOT tail recursive
```

### F. Nested Recursion
A recursive call's **argument** is itself a recursive call.

```python
def nested(n):
    if n > 100:
        return n - 10
    return nested(nested(n + 11))   # recursive call inside a recursive call

print(nested(95))  # 91 (classic "McCarthy 91 function")
```

---

## 5. Recursion vs Iteration

| Aspect | Recursion | Iteration |
|---|---|---|
| Mechanism | Function calls itself | Loop (`for`/`while`) repeats a block |
| Memory | Uses call stack — O(depth) extra memory | Typically O(1) extra memory |
| Speed | Generally slower (function call overhead) | Generally faster |
| Readability | Often cleaner for naturally recursive problems (trees, divide & conquer) | Often cleaner for simple repetition |
| Risk | Stack overflow on deep recursion | No stack risk (bounded by loop conditions) |
| Best for | Trees, graphs, divide-and-conquer, backtracking | Simple counting, linear scans |

### Same Problem, Both Approaches

```python
# Recursive
def sum_recursive(n):
    if n == 0:
        return 0
    return n + sum_recursive(n - 1)

# Iterative
def sum_iterative(n):
    total = 0
    for i in range(1, n + 1):
        total += i
    return total
```

> [!tip]
> Every recursive algorithm **can** be rewritten iteratively (often using an explicit stack to simulate the call stack), but the recursive version is frequently far more readable for inherently recursive structures like trees.

---

## 6. Recursion with Numbers

### Factorial

```python
def factorial(n):
    if n < 0:
        raise ValueError("Factorial not defined for negative numbers")
    if n == 0:
        return 1
    return n * factorial(n - 1)
```

### Fibonacci

```python
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

### Greatest Common Divisor (Euclidean Algorithm)

```python
def gcd(a, b):
    if b == 0:
        return a
    return gcd(b, a % b)

print(gcd(48, 18))  # 6
```

### Power Function (Fast Exponentiation)

```python
def power(base, exp):
    if exp == 0:
        return 1
    if exp % 2 == 0:
        half = power(base, exp // 2)
        return half * half
    return base * power(base, exp - 1)

print(power(2, 10))  # 1024 — computed in O(log n) calls instead of O(n)
```

### Digit Sum

```python
def digit_sum(n):
    if n < 10:
        return n
    return n % 10 + digit_sum(n // 10)

print(digit_sum(12345))  # 15
```

---

## 7. Recursion with Strings

### Reverse a String

```python
def reverse_string(s):
    if len(s) <= 1:
        return s
    return reverse_string(s[1:]) + s[0]

print(reverse_string("hello"))  # "olleh"
```

### Check Palindrome

```python
def is_palindrome(s):
    if len(s) <= 1:
        return True
    if s[0] != s[-1]:
        return False
    return is_palindrome(s[1:-1])

print(is_palindrome("racecar"))  # True
print(is_palindrome("hello"))    # False
```

### Count Occurrences of a Character

```python
def count_char(s, char):
    if not s:
        return 0
    return (1 if s[0] == char else 0) + count_char(s[1:], char)

print(count_char("mississippi", "s"))  # 4
```

### Generate All Permutations

```python
def permutations(s):
    if len(s) <= 1:
        return [s]
    result = []
    for i, char in enumerate(s):
        remaining = s[:i] + s[i+1:]
        for perm in permutations(remaining):
            result.append(char + perm)
    return result

print(permutations("abc"))
# ['abc', 'acb', 'bac', 'bca', 'cab', 'cba']
```

---

## 8. Recursion with Lists

### Sum of a List

```python
def sum_list(lst):
    if not lst:               # empty list = base case
        return 0
    return lst[0] + sum_list(lst[1:])

print(sum_list([1, 2, 3, 4, 5]))  # 15
```

### Find Maximum

```python
def find_max(lst):
    if len(lst) == 1:
        return lst[0]
    max_of_rest = find_max(lst[1:])
    return lst[0] if lst[0] > max_of_rest else max_of_rest

print(find_max([3, 7, 2, 9, 4]))  # 9
```

### Reverse a List

```python
def reverse_list(lst):
    if len(lst) <= 1:
        return lst
    return [lst[-1]] + reverse_list(lst[:-1])

print(reverse_list([1, 2, 3, 4]))  # [4, 3, 2, 1]
```

### Binary Search (Divide and Conquer)

```python
def binary_search(lst, target, low=0, high=None):
    if high is None:
        high = len(lst) - 1
    if low > high:
        return -1   # base case: not found
    mid = (low + high) // 2
    if lst[mid] == target:
        return mid
    elif lst[mid] < target:
        return binary_search(lst, target, mid + 1, high)
    else:
        return binary_search(lst, target, low, mid - 1)

print(binary_search([1, 3, 5, 7, 9, 11], 7))  # 3
```

### Merge Sort (Divide and Conquer)

```python
def merge_sort(lst):
    if len(lst) <= 1:
        return lst    # base case: single element is already sorted

    mid = len(lst) // 2
    left = merge_sort(lst[:mid])
    right = merge_sort(lst[mid:])

    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result

print(merge_sort([5, 2, 9, 1, 7, 3]))  # [1, 2, 3, 5, 7, 9]
```

### Flatten a Nested List

```python
def flatten(lst):
    result = []
    for item in lst:
        if isinstance(item, list):
            result.extend(flatten(item))   # recursive call for nested list
        else:
            result.append(item)
    return result

print(flatten([1, [2, 3, [4, 5]], 6, [7, [8, [9]]]]))
# [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## 9. Recursion with Tuples & Sets

Tuples are immutable but can be recursed over just like lists (via slicing, which creates new tuples).

```python
def sum_tuple(t):
    if not t:
        return 0
    return t[0] + sum_tuple(t[1:])

print(sum_tuple((1, 2, 3, 4)))  # 10
```

Sets have no order/indexing, so recursion typically works by **popping** an element (on a copy) or converting to a list first.

```python
def sum_set(s):
    if not s:
        return 0
    s = set(s)          # work on a copy to avoid mutating caller's set
    elem = s.pop()
    return elem + sum_set(s)

print(sum_set({1, 2, 3, 4}))  # 10
```

> [!note]
> Since sets are unordered, recursive processing order is arbitrary — this is fine for operations like `sum`/`max` that don't depend on order, but unsuitable if you need deterministic sequencing.

---

## 10. Recursion with Dictionaries

### Sum All Values

```python
def sum_dict_values(d):
    if not d:
        return 0
    key = next(iter(d))          # grab one key
    value = d[key]
    remaining = {k: v for k, v in d.items() if k != key}
    return value + sum_dict_values(remaining)

print(sum_dict_values({"a": 1, "b": 2, "c": 3}))  # 6
```

### Recursively Sum Nested Dictionary Values

```python
def deep_sum(d):
    total = 0
    for value in d.values():
        if isinstance(value, dict):
            total += deep_sum(value)   # recurse into nested dict
        elif isinstance(value, (int, float)):
            total += value
    return total

data = {
    "a": 1,
    "b": {"c": 2, "d": {"e": 3, "f": 4}},
    "g": 5
}
print(deep_sum(data))  # 15
```

### Search for a Key in a Nested Dictionary

```python
def find_key(d, target_key):
    if target_key in d:
        return d[target_key]
    for value in d.values():
        if isinstance(value, dict):
            result = find_key(value, target_key)
            if result is not None:
                return result
    return None

nested = {"a": 1, "b": {"c": {"target": 42}}}
print(find_key(nested, "target"))  # 42
```

---

## 11. Recursion with Nested/Mixed Data Structures

Real-world data (e.g., parsed JSON) often mixes lists, dicts, and primitives. Recursion is the natural tool to traverse arbitrary nesting.

```python
def deep_flatten_values(data):
    """Recursively collect all primitive values from any nested combo of dict/list."""
    values = []
    if isinstance(data, dict):
        for value in data.values():
            values.extend(deep_flatten_values(value))
    elif isinstance(data, list):
        for item in data:
            values.extend(deep_flatten_values(item))
    else:
        values.append(data)
    return values

data = {
    "name": "Amit",
    "scores": [90, 85, [70, 60]],
    "meta": {"active": True, "tags": ["python", "dev"]}
}
print(deep_flatten_values(data))
# ['Amit', 90, 85, 70, 60, True, 'python', 'dev']
```

### Recursively Count All Elements (JSON-like structure)

```python
def count_elements(data):
    if isinstance(data, dict):
        return sum(count_elements(v) for v in data.values())
    elif isinstance(data, list):
        return sum(count_elements(item) for item in data)
    else:
        return 1

print(count_elements({"a": [1, 2, {"b": 3, "c": [4, 5]}]}))  # 5
```

---

## 12. Recursion with Trees

Trees are **inherently recursive** structures — every subtree is itself a tree — making recursion the most natural traversal technique.

### Binary Tree Node Definition

```python
class TreeNode:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

#         1
#        / \
#       2   3
#      / \
#     4   5
root = TreeNode(1, TreeNode(2, TreeNode(4), TreeNode(5)), TreeNode(3))
```

### Depth-First Traversals

```python
def preorder(node):     # Root -> Left -> Right
    if node is None:
        return []
    return [node.value] + preorder(node.left) + preorder(node.right)

def inorder(node):       # Left -> Root -> Right
    if node is None:
        return []
    return inorder(node.left) + [node.value] + inorder(node.right)

def postorder(node):     # Left -> Right -> Root
    if node is None:
        return []
    return postorder(node.left) + postorder(node.right) + [node.value]

print(preorder(root))   # [1, 2, 4, 5, 3]
print(inorder(root))    # [4, 2, 5, 1, 3]
print(postorder(root))  # [4, 5, 2, 3, 1]
```

### Tree Height / Max Depth

```python
def tree_height(node):
    if node is None:
        return 0
    return 1 + max(tree_height(node.left), tree_height(node.right))

print(tree_height(root))  # 3
```

### Count Nodes / Sum of Values

```python
def count_nodes(node):
    if node is None:
        return 0
    return 1 + count_nodes(node.left) + count_nodes(node.right)

def sum_tree(node):
    if node is None:
        return 0
    return node.value + sum_tree(node.left) + sum_tree(node.right)

print(count_nodes(root))  # 5
print(sum_tree(root))     # 15
```

### Search in a Binary Search Tree (BST)

```python
def bst_search(node, target):
    if node is None:
        return False
    if node.value == target:
        return True
    elif target < node.value:
        return bst_search(node.left, target)
    else:
        return bst_search(node.right, target)
```

### Check if a Tree is Balanced

```python
def is_balanced(node):
    def check(node):
        if node is None:
            return 0
        left_height = check(node.left)
        if left_height == -1:
            return -1
        right_height = check(node.right)
        if right_height == -1:
            return -1
        if abs(left_height - right_height) > 1:
            return -1
        return 1 + max(left_height, right_height)
    return check(node) != -1
```

---

## 13. Recursion with Graphs

Graphs may contain **cycles**, so recursive traversal requires tracking **visited nodes** to avoid infinite loops.

### Graph Representation (Adjacency List)

```python
graph = {
    "A": ["B", "C"],
    "B": ["A", "D"],
    "C": ["A", "D"],
    "D": ["B", "C", "E"],
    "E": ["D"]
}
```

### Depth-First Search (DFS)

```python
def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()
    if node in visited:
        return visited          # base case: already visited, avoid cycles
    visited.add(node)
    print(node)
    for neighbor in graph[node]:
        dfs(graph, neighbor, visited)
    return visited

dfs(graph, "A")
# A B D C E   (order may vary depending on adjacency list order)
```

### Detecting a Cycle in a Directed Graph

```python
def has_cycle(graph, node, visited, rec_stack):
    visited.add(node)
    rec_stack.add(node)

    for neighbor in graph.get(node, []):
        if neighbor not in visited:
            if has_cycle(graph, neighbor, visited, rec_stack):
                return True
        elif neighbor in rec_stack:      # back edge found -> cycle!
            return True

    rec_stack.remove(node)   # backtrack
    return False
```

### Counting Connected Components (Undirected Graph)

```python
def count_components(graph):
    visited = set()
    count = 0

    def dfs(node):
        visited.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                dfs(neighbor)

    for node in graph:
        if node not in visited:
            dfs(node)
            count += 1
    return count
```

> [!warning]
> Always maintain a **visited set** when recursing over graphs. Without it, cycles cause **infinite recursion** and an eventual `RecursionError`.

---

## 14. Recursion Limit & Stack Overflow

Python enforces a maximum recursion depth (default **1000**) to prevent the interpreter itself from crashing due to a C stack overflow.

```python
import sys

print(sys.getrecursionlimit())  # 1000 (default)

def infinite_recursion(n):
    return infinite_recursion(n + 1)   # no base case!

infinite_recursion(0)
# RecursionError: maximum recursion depth exceeded
```

### Adjusting the Limit (Use with Caution)

```python
import sys
sys.setrecursionlimit(5000)
```

> [!warning]
> Raising the limit doesn't add actual stack memory — it only changes when Python *complains*. Setting it too high can crash the Python process itself with a segmentation fault since the underlying C stack still has a hard limit.

### When You Hit the Limit, Consider:
1. **Convert to iteration** using an explicit stack/queue.
2. **Increase efficiency** (e.g., memoization to reduce call depth/count).
3. **Use tail-recursion-style + trampolining** (see below) if deep recursion is unavoidable.

### Trampolining (Simulating Tail-Call Optimization)

Since Python doesn't optimize tail calls, you can manually avoid stack growth using a "trampoline" pattern:

```python
def factorial_trampoline(n, acc=1):
    while True:
        if n == 0:
            return acc
        n, acc = n - 1, n * acc   # loop instead of recursive call — no new stack frame

print(factorial_trampoline(10000))  # works fine, no RecursionError
```

---

## 15. Tail Recursion & Why Python Doesn't Optimize It

**Tail Call Optimization (TCO)** is a compiler/interpreter technique where a tail-recursive call **reuses the current stack frame** instead of creating a new one — effectively turning recursion into a loop internally, using constant stack space.

```python
def factorial_tail(n, acc=1):
    if n == 0:
        return acc
    return factorial_tail(n - 1, n * acc)   # tail call: nothing happens after it returns
```

> [!warning] Important
> **CPython does NOT implement tail call optimization**, by design — Guido van Rossum has stated this is intentional, since TCO makes tracebacks less informative (frames disappear, obscuring the call history during debugging). This means even a "tail recursive" function in Python still consumes stack frames and can hit `RecursionError` just like any other recursive function.

If you truly need TCO-like behavior in Python, use:
- **Iteration** (the standard, idiomatic solution).
- **Trampolining** (manual simulation, shown above).
- Third-party decorators (e.g., some libraries implement trampoline decorators), though these add complexity and are rarely necessary.

---

## 16. Memoization & Optimizing Recursion

Naive recursive solutions (like plain Fibonacci) often **recompute the same sub-problem many times**. **Memoization** caches results of expensive calls so repeated calls return instantly.

### Without Memoization (Exponential Time — O(2ⁿ))

```python
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

fib(35)  # noticeably slow
```

### Manual Memoization with a Dictionary

```python
def fib_memo(n, cache={}):
    if n in cache:
        return cache[n]
    if n <= 1:
        return n
    cache[n] = fib_memo(n - 1, cache) + fib_memo(n - 2, cache)
    return cache[n]

fib_memo(35)  # instant
```

> [!warning] Mutable Default Argument Pitfall
> Using a mutable default argument (`cache={}`) works here because we intentionally want it to persist across calls, but this is generally considered risky/unconventional — see the "Common Pitfalls" section in general Python gotchas. Prefer the `functools.lru_cache` approach below for production code.

### Using `functools.lru_cache` (Recommended)

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

print(fib(50))  # instant, thanks to automatic caching
```

### Using `functools.cache` (Python 3.9+, simpler alias)

```python
from functools import cache

@cache
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)
```

### Time Complexity Comparison

| Approach | Time Complexity | Space Complexity |
|---|---|---|
| Naive recursive Fibonacci | O(2ⁿ) | O(n) (call stack depth) |
| Memoized recursive Fibonacci | O(n) | O(n) (cache + stack) |
| Iterative Fibonacci | O(n) | O(1) |

---

## 17. Common Pitfalls

| Pitfall | Why It's a Problem | Fix |
|---|---|---|
| Missing base case | Causes infinite recursion → `RecursionError` | Always define a clear, reachable base case |
| Base case never reached | Input doesn't shrink correctly toward the base case | Ensure each recursive call moves strictly closer to the base case |
| Recomputing overlapping sub-problems | Exponential slowdowns (e.g., naive Fibonacci) | Use memoization (`lru_cache` or manual dict) |
| Recursing on mutable objects without copying | Accidentally mutates shared state across calls | Pass copies, or use immutable structures/slicing |
| Ignoring cycles in graphs | Infinite recursion on cyclic structures | Track visited nodes |
| Assuming Python optimizes tail recursion | Still hits `RecursionError` for deep tail calls | Convert to iteration or use trampolining |
| Deeply recursing on large inputs (e.g., huge lists) | Risk of hitting the recursion limit | Prefer iteration for simple linear scans over huge inputs |
| Using mutable default arguments for memoization casually | Can cause subtle bugs if reused across unrelated calls | Prefer `functools.lru_cache`/`cache` |

---

## 18. Best Practices

- ✅ Always define the **base case first**, before writing the recursive case.
- ✅ Make sure each recursive call **strictly reduces** the problem size.
- ✅ Use **memoization** (`functools.lru_cache`) whenever sub-problems overlap.
- ✅ Track **visited nodes** when recursing over graphs to avoid cycles.
- ✅ Prefer **iteration** for simple linear operations on large inputs to avoid stack limits.
- ✅ Use recursion where it naturally fits the problem's structure: trees, graphs, divide-and-conquer, backtracking, combinatorics (permutations/subsets).
- ✅ Keep recursive functions **pure** (avoid side effects on shared mutable state) when possible — makes reasoning and debugging much easier.
- ✅ For deep/performance-critical recursion, consider trampolining or converting to an explicit-stack iterative version.

---

## 19. Quick Reference Cheat Sheet

```python
# Template
def recursive_fn(input):
    if base_case(input):
        return base_result
    return combine(input, recursive_fn(smaller(input)))

# Numbers
factorial(n) -> n * factorial(n-1),        base: n == 0
fibonacci(n) -> fib(n-1) + fib(n-2),       base: n <= 1
gcd(a, b)    -> gcd(b, a % b),             base: b == 0

# Strings
reverse(s)     -> reverse(s[1:]) + s[0],   base: len(s) <= 1
is_palindrome  -> compare ends, recurse on s[1:-1]

# Lists
sum_list(lst)  -> lst[0] + sum_list(lst[1:]),  base: empty list
binary_search  -> divide search range in half each call

# Trees
traverse(node) -> combine(node.value, traverse(node.left), traverse(node.right))
                                              base: node is None

# Graphs
dfs(node, visited) -> mark visited, recurse into unvisited neighbors
                                              base: node already visited
```

| Type | Description |
|---|---|
| Direct | Function calls itself |
| Indirect/Mutual | Two+ functions call each other |
| Linear | One recursive call per invocation |
| Tree/Branching | Multiple recursive calls per invocation |
| Tail | Recursive call is the last operation (not optimized in Python) |
| Nested | Recursive call's argument is itself a recursive call |

---

*Tags:* #python #recursion #algorithms #data-structures #trees #graphs #memoization

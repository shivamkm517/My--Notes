---
tags: [python, pep8, style-guide, best-practices, programming]
created: 2026-07-08
aliases: [PEP 8, Python Style Guide, PEP8]
---

# PEP 8 — Python Style Guide Deep Dive

## 1. What is PEP 8?

**PEP 8** (Python Enhancement Proposal 8) is the **official style guide for Python code**, originally written by Guido van Rossum, Barry Warsaw, and Nick Coghlan. It defines conventions for writing readable, consistent Python code.

> [!note] Core Philosophy
> *"Code is read much more often than it is written."* PEP 8 exists to make Python code **consistent across projects and developers**, so anyone can read anyone else's code without friction.

> [!info] Where to Find It
> Official reference: `https://peps.python.org/pep-0008/`. Most teams also enforce it automatically via linters/formatters rather than manual review (see [[#12. Tooling — Enforcing PEP 8 Automatically]]).

### Guido's Own Caveat
> [!tip] "A Foolish Consistency is the Hobgoblin of Little Minds"
> PEP 8 itself says: know when to be inconsistent. If following a rule would make code **less** readable, or breaks compatibility with surrounding code that already violates the rule, it's fine to deviate.

---

## 2. Code Layout

### 2.1 Indentation
- Use **4 spaces per indentation level** — never tabs.
- Never mix tabs and spaces (Python 3 disallows this outright and raises `TabError`).

```python
# Correct
def greet(name):
    if name:
        print(f"Hello, {name}")

# Wrong -- inconsistent indentation
def greet(name):
  if name:
      print(f"Hello, {name}")
```

### 2.2 Continuation Lines (Wrapping Long Expressions)
Align wrapped elements either with the opening delimiter, or use a hanging indent.

```python
# Aligned with opening delimiter
result = some_function(arg_one, arg_two,
                        arg_three, arg_four)

# Hanging indent (extra indentation to distinguish from body)
result = some_function(
    arg_one, arg_two,
    arg_three, arg_four,
)

# Closing bracket options -- either matches first char of the line
# that starts the construct, or the first non-whitespace char:
result = some_function(
    arg_one, arg_two,
)
```

### 2.3 Maximum Line Length
- Limit all lines to a maximum of **79 characters** (72 for docstrings/comments).
- Many modern teams relax this to **88 or 100** (e.g., via `black`'s default of 88) — but PEP 8's official recommendation is 79.

```python
# Use backslash or (preferably) parentheses for line continuation
total = (first_variable + second_variable
         + third_variable + fourth_variable)
```

> [!warning] Avoid Backslash Continuation
> ```python
> total = first_variable + \
>         second_variable   # works but fragile -- easy to break with trailing whitespace
> ```
> Prefer implicit continuation inside parentheses/brackets/braces instead.

### 2.4 Blank Lines
| Context | Blank Lines |
|---|---|
| Between top-level function/class definitions | 2 blank lines |
| Between methods inside a class | 1 blank line |
| To separate logical sections within a function | 1 blank line (sparingly) |

```python
class Foo:

    def method_one(self):
        pass

    def method_two(self):
        pass


class Bar:
    pass


def standalone_function():
    pass
```

### 2.5 Imports

- Imports go **at the top of the file**, after module docstrings/comments.
- One import per line (except `from x import a, b, c`).
- Group imports in this order, each group separated by a blank line:
  1. Standard library imports
  2. Third-party imports
  3. Local application/library imports

```python
# Correct
import os
import sys

import requests
import numpy as np

from myapp.models import User
from myapp.utils import helper_function

# Wrong
import os, sys   # multiple imports on one line
```

> [!tip] Avoid Wildcard Imports
> ```python
> from module import *   # avoid -- pollutes namespace, hides where names come from
> ```

---

## 3. Whitespace

### 3.1 Avoid Extraneous Whitespace
```python
# Correct
spam(ham[1], {eggs: 2})
x = 1
y = 2

# Wrong
spam( ham[ 1 ], { eggs: 2 } )    # extra space inside brackets
x             = 1                  # extra space to "align" (avoid this)
```

### 3.2 Whitespace Around Operators

```python
# Correct
i = i + 1
x = x * 2 - 1
c = (a + b) * (a - b)

# Wrong
i=i+1
x = x*2 - 1
```

**Exception:** when operators of different priority are used, PEP 8 permits (but doesn't require) removing whitespace around the operator with **higher priority**:
```python
# Acceptable -- extra space removed around * to show it binds tighter
hypot2 = x*x + y*y
```

### 3.3 No Space Before, Space After Commas/Colons/Semicolons
```python
# Correct
x, y = 1, 2
def func(a, b): pass

# Wrong
x , y = 1 , 2
def func(a ,b): pass
```

### 3.4 Slice Colons Act as a Binary Operator
```python
# Correct
ham[1:9]
ham[1:9:3]
ham[lower:upper], ham[lower:upper:], ham[lower::step]
ham[lower+offset : upper+offset]     # equal spacing on both sides when expressions are complex

# Wrong
ham[1: 9]
ham[1 :9]
ham[lower : : upper]
```

### 3.5 No Space Around `=` for Keyword Arguments / Default Values
```python
# Correct
def complex(real, imag=0.0):
    return magic(r=real, i=imag)

# Wrong
def complex(real, imag = 0.0):
    return magic(r = real, i = imag)
```

> [!info] Exception
> When a parameter has **both** a type annotation and a default value, DO use spaces around `=`:
> ```python
> def func(x: int = 5) -> int:   # correct -- spaces around = when annotated
>     return x
> ```

---

## 4. Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Module/Package | `lowercase_with_underscores` (short, all lowercase) | `my_module.py` |
| Class | `CapWords` / `PascalCase` | `class BankAccount:` |
| Exception | `CapWords`, usually ending in `Error` | `class ValidationError(Exception):` |
| Function | `lowercase_with_underscores` | `def calculate_total():` |
| Variable | `lowercase_with_underscores` | `total_price = 100` |
| Constant | `UPPERCASE_WITH_UNDERSCORES` | `MAX_RETRIES = 5` |
| Method | `lowercase_with_underscores` | `def get_balance(self):` |
| Protected attribute (convention) | `_leading_underscore` | `self._internal_state` |
| Private attribute (name-mangled) | `__leading_underscore` | `self.__secret` |
| "Magic"/dunder methods | `__double_leading_and_trailing__` | `__init__`, `__str__` |
| Type variables (generics) | `CapWords`, short | `T`, `KT`, `VT` |

> [!warning] Avoid Ambiguous Single-Character Names
> Never use lowercase `l` (letter el), uppercase `O`, or uppercase `I` as single-character variable names — in some fonts they're indistinguishable from `1` and `0`.
> ```python
> l = 1   # avoid: looks like "I" or "1" in many fonts
> ```

### 4.1 Function and Method Argument Naming
- Always use `self` for the first argument of instance methods.
- Always use `cls` for the first argument of class methods.
- See [[self, cls, Static Methods and __new__]] for the deep dive on why.

### 4.2 Avoid Shadowing Built-ins
```python
# Avoid overwriting built-in names
list = [1, 2, 3]     # bad -- shadows the built-in list() type
type = "dog"           # bad -- shadows built-in type()

# Better
items_list = [1, 2, 3]
animal_type = "dog"
```

---

## 5. Comments and Docstrings

### 5.1 Block Comments
```python
# This is a block comment explaining the logic below.
# Each line starts with '# ' and stays aligned with the code.
x = compute_something()
```

### 5.2 Inline Comments
Use sparingly; separate from statement by **at least 2 spaces**.

```python
x = x + 1  # increment counter
```

> [!warning]
> Inline comments that just restate the obvious are noise:
> ```python
> x = x + 1  # increment x by 1   -- unnecessary, adds no value
> ```

### 5.3 Docstrings (PEP 257)
Every public module, function, class, and method should have a docstring.

```python
def calculate_area(radius):
    """Calculate the area of a circle.

    Args:
        radius (float): The radius of the circle.

    Returns:
        float: The area of the circle.
    """
    return 3.14159 * radius ** 2
```

```python
class BankAccount:
    """Represents a simple bank account with deposit/withdraw operations."""

    def deposit(self, amount):
        """Add funds to the account balance."""
        ...
```

> [!tip] One-liner Docstring Rule
> For simple functions, a one-line docstring is fine, but it should still use triple quotes and end with a period:
> ```python
> def add(a, b):
>     """Return the sum of a and b."""
>     return a + b
> ```

---

## 6. Programming Recommendations

### 6.1 Comparisons

```python
# Correct -- use 'is'/'is not' for None comparisons
if x is None:
    ...
if x is not None:
    ...

# Wrong
if x == None:
    ...
```

```python
# Correct -- use truthiness for emptiness checks
if not my_list:
    ...

# Wrong
if len(my_list) == 0:
    ...
if my_list == []:
    ...
```

```python
# Correct -- use isinstance() for type checks
if isinstance(obj, int):
    ...

# Wrong
if type(obj) == int:
    ...
```

### 6.2 Boolean Comparisons
```python
# Correct
if is_valid:
    ...
if not is_valid:
    ...

# Wrong
if is_valid == True:
    ...
if is_valid is True:   # also discouraged unless truly checking identity to the singleton
    ...
```

### 6.3 Exception Handling Style
```python
# Correct -- catch specific exceptions
try:
    value = int(user_input)
except ValueError:
    print("Invalid input")

# Wrong -- bare except catches EVERYTHING including SystemExit, KeyboardInterrupt
try:
    value = int(user_input)
except:
    print("Something went wrong")
```
> [!info] See Also
> Full breakdown in [[Exception Handling (Python)]].

### 6.4 Use `.startswith()` / `.endswith()` Instead of Slicing
```python
# Correct
if name.startswith("Mr."):
    ...

# Wrong
if name[:3] == "Mr.":
    ...
```

### 6.5 Context Managers for Resource Management
```python
# Correct
with open("file.txt") as f:
    data = f.read()

# Wrong
f = open("file.txt")
data = f.read()
f.close()   # easy to forget, or skipped if an exception occurs first
```

### 6.6 Function Annotations (Type Hints)
PEP 8 recommends consistent use of type hints where the codebase has adopted them (formalized further in PEP 484).

```python
def greet(name: str, times: int = 1) -> str:
    return (f"Hello, {name}! " * times).strip()
```

---

## 7. Trailing Commas

Use a trailing comma when the closing bracket is on its own line — helps produce cleaner diffs in version control.

```python
# Correct
my_list = [
    1,
    2,
    3,
]

# Also acceptable (single line, no trailing comma needed)
my_list = [1, 2, 3]
```

> [!warning] Watch Out for Single-Element Tuples
> ```python
> t = (1,)     # correct -- this is a tuple
> t = (1)       # WRONG -- this is just an int in parentheses, not a tuple!
> ```

---

## 8. String Quotes

PEP 8 doesn't mandate single vs double quotes — pick one and **be consistent** within a project. However:
- If a string contains a quote character, use the other quote type to avoid backslash-escaping.

```python
text1 = 'He said "hello"'     # use single quotes to avoid escaping
text2 = "It's a nice day"      # use double quotes to avoid escaping
```

> [!info] Modern Convention
> Many teams standardize on **double quotes** (this is also `black`'s default), reserving single quotes for nested strings.

---

## 9. Function/Method Design Guidelines

### 9.1 Return Consistency
Be consistent — either **all** return statements in a function return an expression, or **none** do.

```python
# Correct
def get_value(flag):
    if flag:
        return 1
    return None    # explicit None, consistent style

# Avoid mixing
def get_value(flag):
    if flag:
        return 1
    # implicit None fallthrough -- inconsistent with the explicit return above
```

### 9.2 Default Argument Values — Avoid Mutable Defaults
```python
# Wrong -- classic Python pitfall
def append_item(item, items=[]):
    items.append(item)
    return items

# Correct
def append_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

> [!warning] Why This Matters
> Default argument values are evaluated **once**, at function definition time — not on every call. A mutable default is **shared across all calls** that don't explicitly pass a value, leading to subtle, hard-to-track bugs.

---

## 10. Layout of Operators When Wrapping Lines

PEP 8 (updated in later revisions, following Knuth's convention) recommends breaking **before** binary operators, not after — for better readability, aligning operators with their operands.

```python
# Correct (operator at the start of the continuation line)
income = (gross_wages
          + taxable_interest
          + dividends
          - qualified_dividends
          - ira_deduction)

# Also historically acceptable, but discouraged now
income = (gross_wages +
          taxable_interest +
          dividends -
          qualified_dividends -
          ira_deduction)
```

---

## 11. Class Design Conventions

```python
class MyClass:
    """A well-formatted example class."""

    class_variable = "shared"    # class-level attribute

    def __init__(self, value):
        self.value = value        # instance attribute

    def public_method(self):
        """Public methods have simple, descriptive docstrings."""
        return self._helper()

    def _helper(self):             # leading underscore = internal/protected
        return self.value * 2

    @staticmethod
    def utility_function(x):
        return x + 1

    @classmethod
    def from_string(cls, s):
        return cls(int(s))
```

> [!tip] Method Order Convention (commonly followed, not strict PEP 8)
> 1. `__init__` / dunder methods
> 2. Public methods
> 3. Protected/private methods (`_helper`)
> 4. Static/class methods at the bottom, or grouped near related public methods

---

## 12. Tooling — Enforcing PEP 8 Automatically

Manually tracking every rule is impractical — teams rely on automated tools.

| Tool | Purpose |
|---|---|
| `flake8` | Lints code, flags PEP 8 violations + basic bugs |
| `pylint` | More thorough linting, style + code smell detection |
| `black` | Auto-formatter — rewrites code to a consistent style (opinionated, minimal config) |
| `autopep8` | Auto-formatter that specifically targets PEP 8 compliance |
| `isort` | Automatically sorts and groups imports per PEP 8 conventions |
| `ruff` | Extremely fast, modern all-in-one linter/formatter (increasingly popular, Rust-based) |

```bash
# Common workflow
pip install black flake8 isort
black .        # auto-format code
isort .         # sort imports
flake8 .         # check remaining issues
```

> [!success] Practical Takeaway
> In real-world teams, you rarely enforce PEP 8 by memory — you configure `black`/`ruff` + a pre-commit hook, and let tooling handle formatting automatically, freeing you to focus on logic during code review.

---

## 13. PEP 8 vs Other Related PEPs

| PEP | Topic |
|---|---|
| PEP 8 | Style guide (this note) |
| PEP 257 | Docstring conventions |
| PEP 484 | Type hints |
| PEP 20 | "The Zen of Python" (guiding philosophy — `import this`) |

```python
import this
# Prints "The Zen of Python" -- guiding principles behind Python's design,
# closely related in spirit to PEP 8's emphasis on readability.
```

---

## 14. Best Practices ✅

> [!success] Do's
> - Use 4 spaces for indentation, never tabs
> - Keep lines ≤ 79 characters (or your team's agreed limit, e.g. 88 for `black`)
> - Follow naming conventions strictly (`snake_case` for functions/variables, `PascalCase` for classes)
> - Use `is`/`is not` for `None` comparisons, truthiness for emptiness checks
> - Add docstrings to public modules, classes, and functions
> - Automate style enforcement with `black`/`flake8`/`ruff` instead of manual policing

> [!failure] Don'ts
> - Don't mix tabs and spaces
> - Don't use bare `except:` clauses
> - Don't use mutable objects as default argument values
> - Don't shadow built-in names (`list`, `type`, `id`, `str`, etc.)
> - Don't write wildcard imports (`from module import *`) in application code
> - Don't sacrifice readability just to satisfy a rule — PEP 8 explicitly allows judgment calls

---

## 15. Quick Revision Summary

| Category | Rule |
|---|---|
| Indentation | 4 spaces, no tabs |
| Line length | 79 characters (max), 72 for docstrings |
| Blank lines | 2 between top-level defs, 1 between methods |
| Imports | Grouped: stdlib → third-party → local; one per line |
| Naming | `snake_case` (functions/vars), `PascalCase` (classes), `UPPER_CASE` (constants) |
| Whitespace | No space inside brackets; space after commas; no space around `=` for defaults |
| Comparisons | `is None`, truthiness for empty checks, `isinstance()` for types |
| Docstrings | Triple-quoted, every public function/class/module |
| Mutable defaults | Never use `[]`/`{}` directly as a default argument |
| Tooling | Automate with `black`, `flake8`, `isort`, or `ruff` |

**Golden Rule:** *Readability counts.* When in doubt, favor the option a future reader (including future-you) will understand fastest.

---

## Related Notes
- [[self, cls, Static Methods and __new__]]
- [[Four Pillars of OOP]]
- [[Exception Handling (Python)]]
- [[Python Comprehensions]]
- [[Type Hints and Static Typing in Python]]

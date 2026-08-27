---
title: Python Fundamentals — Complete Refresher
tags: [python, fundamentals, basics, cheatsheet]
related: "[[NumPy_Cheat_Sheet]], [[Pandas_Data_Handling_Notes]], [[Python_File_Handling_Notes]]"
---

# Python Fundamentals Refresher

> [!info] Scope
> Variables, data types, input/output, type casting, operators, conditionals, loops, functions, and an OOP overview — the core building blocks of Python, explained in depth.

---

## 1. Variables

A variable is a name bound to a value stored in memory. Python is **dynamically typed** — you don't declare a type; it's inferred at runtime.

```python
x = 10
name = "Alice"
is_active = True
```

### Variable Naming Rules

- Must start with a letter or underscore (`_`), not a digit.
- Can contain letters, digits, underscores.
- Case-sensitive (`age` ≠ `Age`).
- Cannot be a reserved keyword (`if`, `for`, `class`, etc.).

```python
_valid = 1
valid_2 = 2
2invalid = 3     # SyntaxError
```

### Multiple Assignment

```python
a, b, c = 1, 2, 3        # unpacking
x = y = z = 0             # same value to multiple variables
a, b = b, a                # swap without a temp variable
```

### Dynamic Typing

```python
x = 5        # x is int
x = "hello"  # now x is str — completely valid, type changes at runtime
```

> [!note]
> Python variables are **references (labels)** to objects in memory, not containers holding the value directly. `x = 5` means "point the name `x` at the integer object `5`."

---

## 2. Data Types

### Numeric Types

```python
a = 10          # int — whole numbers, arbitrary precision
b = 3.14        # float — decimal numbers (IEEE 754 double precision)
c = 2 + 3j      # complex — real + imaginary part
```
### Floating precision issue


A better way to say it is:

> **Python stores floating-point (decimal) numbers in binary (base 2). Since many decimal fractions cannot be represented exactly in binary, Python stores the closest possible approximation, which can lead to small rounding errors.**

For example:

```python
x = 0.1
print(format(x, ".20f"))
```

Output:

```text
0.10000000000000000555
```

Python did **not intentionally change** `0.1` to a different value. Instead:

1. You write:
    
    ```python
    0.1
    ```
    
2. Python converts it to binary:
    
    ```
    0.000110011001100110011001100...
    ```
    
    This repeats forever.
    
3. A `float` has only **64 bits** available, so Python stores only the first 53 bits of precision (including the implicit leading bit) and rounds the rest.
    
4. When you print it with high precision, you see the tiny approximation:
    
    ```
    0.10000000000000000555
    ```
    

The same happens with `0.2`:

```
0.2
↓
0.20000000000000001110...
```

So when you add them:

```python
print(0.1 + 0.2)
```

Python is really adding:

```
0.10000000000000000555
+
0.20000000000000001110
----------------------
0.30000000000000004440
```

which is displayed as:

```
0.30000000000000004
```

### One important correction

It's **not every decimal number** that has this issue.

Some decimal numbers **can** be represented exactly in binary, for example:

```python
0.5   # Exact
0.25  # Exact
0.75  # Exact
1.0   # Exact
2.5   # Exact
```

because their binary representations terminate:

```
0.5   = 0.1₂
0.25  = 0.01₂
0.75  = 0.11₂
```

But numbers like these **cannot** be represented exactly:

```python
0.1
0.2
0.3
0.6
0.7
```

because their binary expansions repeat forever.
### How to compare floats correctly

Instead of:

```python
a = 0.1 + 0.2
if a == 0.3:    
	print("Equal")
```

use:

```python
import math

a = 0.1 + 0.2
print(math.isclose(a, 0.3))
```

Output:

```
True
```

`math.isclose()` checks whether two floating-point values are close enough within a tolerance.

---

# For exact decimal arithmetic

Python provides the `decimal` module:

```python
from decimal import Decimala 

a= Decimal("0.1")
b = Decimal("0.2")
print(a + b)
```

Output:

```
0.3
```

Notice the values are passed as **strings**. If you write `Decimal(0.1)`, the floating-point approximation has already been introduced.
### `str` — String

```python
s = "Hello, World!"
s2 = 'single quotes work too'
s3 = """triple quotes
for multi-line strings"""
```

Strings are **immutable** — any "modification" creates a new string object.

```python
s.upper()          # "HELLO, WORLD!"
s.lower()
s.strip()          # remove leading/trailing whitespace
s.replace("Hello", "Hi")
s.split(",")       # -> ['Hello', ' World!']
s.find("World")    # index of substring, -1 if not found
len(s)             # length of string
s[0]               # indexing -> 'H'
s[0:5]             # slicing -> 'Hello'
s[::-1]            # reverse string
f"{a} and {b}"     # f-string formatting (Python 3.6+)
"{} and {}".format(a, b)   # .format() method
```

### `bool` — Boolean

```python
t = True
f = False
```

- Internally, `bool` is a subclass of `int`: `True == 1`, `False == 0`.
- Truthy/Falsy: `0`, `0.0`, `""`, `[]`, `{}`, `()`, `None`, `set()` are all **falsy**; virtually everything else is **truthy**.

```python
if []:
    print("won't run")   # empty list is falsy
if "text":
    print("runs")          # non-empty string is truthy
```

### `None` — Null / Absence of Value

```python
x = None
```

- `None` is a singleton object representing "no value."
- Compare using `is None` / `is not None`, **not** `== None`.

```python
if x is None:
    print("x has no value")
```
### 1. `None` is an object

Many beginners think `None` is a keyword like `if` or `for`, but it's actually a **singleton object**.

```python
x = Noneprint(type(x))
```

Output:

```python
<class 'NoneType'>
```

So:

- `None` is an object.
- Its type is `NoneType`.
- `NoneType` has only **one instance**, which is `None`.

---

# 2. Singleton object

A singleton means **only one object exists**.

```python
a = None
b = None
print(a is b)
```

Output:

```python
True
```

Both variables point to the exact same object in memory.

You can verify this:

```python
print(id(a))
print(id(b))
```

The IDs will be identical.

---

# 3. Why use `is` instead of `==`

Always compare with `None` using `is`:

```python
if x is None:    
	...
```

Not:

```python
if x == None:   # Avoid    ...
```

### Why?

`==` checks **value equality**, and classes can override how it works.

```python
class Weird:    
	def __eq__(self, other):        
		return
		 
Trueobj = Weird()
print(obj == None)   # Trueprint(obj is None)   # False
```

`is` checks **object identity**, which is what you want for the singleton `None`.
### Checking Type

```python
type(x)               # returns the type object, e.g. <class 'int'>
isinstance(x, int)     # returns True/False, preferred for type checks
isinstance(x, (int, float))   # check against multiple types
```

---

## 3. Input and Output

### Output — `print()`

```python
print("Hello")
print("Value:", 10, "Name:", "Alice")     # multiple args, space-separated by default
print("A", "B", sep="-")                  # custom separator -> "A-B"
print("No newline", end=" ")              # custom line ending (default is '\n')
print(f"Age: {25}, Name: {'Bob'}")        # f-string, most common modern style
```

**Formatting numbers:**

```python
pi = 3.14159265
print(f"{pi:.2f}")        # 2 decimal places -> "3.14"
print(f"{1000000:,}")     # thousands separator -> "1,000,000"
print(f"{0.25:.1%}")      # percentage format -> "25.0%"
print(f"{42:5d}")         # width padding -> "   42"
print(f"{42:05d}")        # zero-padded -> "00042"
```

### Input — `input()`

```python
name = input("Enter your name: ")   # ALWAYS returns a string
age = int(input("Enter your age: "))  # must manually cast to int/float if needed
```

> [!important]
> `input()` always returns a `str`, even if the user types a number. You must explicitly cast it (`int()`, `float()`) before doing arithmetic.

```python
# Reading multiple values from one line
x, y = input("Enter two numbers: ").split()
x, y = int(x), int(y)

# Reading a list of numbers
nums = list(map(int, input("Enter numbers: ").split()))
```

---

## 4. Type Casting

Converting one data type into another — **implicit** (automatic) or **explicit** (manual).

### Implicit Casting (done automatically by Python)

```python
x = 5          # int
y = 2.5        # float
z = x + y      # int automatically becomes float -> 7.5, z is float
```

### Explicit Casting (manual, using built-in functions)

```python
int("10")        # str -> int -> 10
int(3.9)         # float -> int -> 3 (TRUNCATES, does not round)
float("3.14")    # str -> float -> 3.14
str(100)         # int -> str -> "100"
bool(0)          # -> False
bool(1)          # -> True
bool("")         # -> False
bool("text")     # -> True
list("abc")      # str -> list -> ['a', 'b', 'c']
tuple([1,2,3])   # list -> tuple -> (1, 2, 3)
set([1,1,2])     # list -> set -> {1, 2}
```

> [!warning] Common casting errors
> ```python
> int("3.14")     # ValueError: invalid literal for int()
> int("abc")      # ValueError
> ```
> To convert a numeric string with decimals to int, cast to float first: `int(float("3.14"))`.

---

## 5. Operators

### Arithmetic Operators

| Operator | Meaning | Example |
|---|---|---|
| `+` | Addition | `5 + 2` → `7` |
| `-` | Subtraction | `5 - 2` → `3` |
| `*` | Multiplication | `5 * 2` → `10` |
| `/` | Division (always returns float) | `5 / 2` → `2.5` |
| `//` | Floor division (rounds down) | `5 // 2` → `2` |
| `%` | Modulus (remainder) | `5 % 2` → `1` |
| `**` | Exponentiation | `5 ** 2` → `25` |
## Division always returns a float

```python
print(6 / 3)
```

Output:

```python
2.0
```

Even though the mathematical result is an integer.

Use `//` if you want integer floor division.
### Comparison (Relational) Operators

| Operator | Meaning |
|---|---|
| `==` | Equal to |
| `!=` | Not equal to |
| `>` | Greater than |
| `<` | Less than |
| `>=` | Greater than or equal |
| `<=` | Less than or equal |

### Logical Operators

```python
True and False   # False — both must be True
True or False    # True — at least one True
not True         # False — inverts
```

### Assignment Operators

```python
x = 5
x += 1     # x = x + 1
x -= 1     # x = x - 1
x *= 2     # x = x * 2
x /= 2     # x = x / 2
x //= 2    # x = x // 2
x %= 2     # x = x % 2
x **= 2    # x = x ** 2
```

### Identity Operators

```python
a is b          # True if a and b are the SAME object in memory
a is not b      # True if they are different objects
```

> [!warning] `is` vs `==`
> `==` checks **value equality**. `is` checks **identity** (same object in memory). Always use `==` to compare values, and `is` only for `None`/singleton checks.
> ```python
> a = [1, 2, 3]
> b = [1, 2, 3]
> a == b     # True  (same values)
> a is b     # False (different objects in memory)
> ```

### Membership Operators

```python
'a' in 'abc'          # True
3 in [1, 2, 3]        # True
'x' not in 'abc'      # True
```

### Bitwise Operators

| Operator | Meaning |
|---|---|
| `&` | Bitwise AND |
| `\|` | Bitwise OR |
| `^` | Bitwise XOR |
| `~` | Bitwise NOT |
| `<<` | Left shift |
| `>>` | Right shift |

---

# Step 1: Everything starts with bits

A computer stores integers as a sequence of **bits** (0s and 1s).

For example, using 8 bits:

```
Decimal    Binary

0          00000000
1          00000001
2          00000010
3          00000011
4          00000100
5          00000101
6          00000110
7          00000111
8          00001000
```

When you use a bitwise operator, Python applies the operation **bit by bit**.

---

# Bitwise AND (`&`)

Rule:

```
1 & 1 = 1

1 & 0 = 0

0 & 1 = 0

0 & 0 = 0
```

Think:

> **AND only keeps a bit if both bits are 1.**

Example:

```python
a = 5
b = 3

print(a & b)
```

Binary:

```
5 = 0101

3 = 0011
```

Compare each column:

```
  0101
& 0011
------
  0001
```

Result:

```
1
```

---

## Common use

Checking whether a bit is set.

Example:

```python
x = 13      # 1101

mask = 4    # 0100

print(x & mask)
```

```
1101
0100
----
0100
```

Since the result is not zero, that bit was set.

---

# Bitwise OR (`|`)

Rule:

```
1 | 1 = 1

1 | 0 = 1

0 | 1 = 1

0 | 0 = 0
```

Think:

> **OR sets a bit if either bit is 1.**

Example:

```python
5 | 3
```

```
0101

0011
----
0111
```

Result:

```
7
```

---

## Common use

Turning on specific bits.

---

# Bitwise XOR (`^`)

Rule:

```
1 ^ 1 = 0

1 ^ 0 = 1

0 ^ 1 = 1

0 ^ 0 = 0
```

Think:

> **XOR is 1 only when the bits are different.**

Example:

```
5 = 0101

3 = 0011
```

```
0101

0011
----
0110
```

Result:

```
6
```

---

## XOR properties (important)

```
a ^ a = 0

a ^ 0 = a

a ^ b ^ b = a
```

This is why XOR is useful for finding a unique element.

Example:

```python
nums = [2, 3, 2, 5, 5]

ans = 0

for n in nums:
    ans ^= n

print(ans)
```

Output:

```
3
```

All duplicate values cancel each other out.

---

# Bitwise NOT (`~`)

This is the most confusing operator.

Many people think it simply flips bits.

It **does flip bits**, but Python represents negative integers using **two's complement**.

Example:

```python
print(~5)
```

Binary (8-bit illustration):

```
5

00000101
```

Flip every bit:

```
11111010
```

This bit pattern represents:

```
-6
```

So:

```python
~5
```

returns

```
-6
```

### Shortcut formula

```
~x = -(x + 1)
```

Examples:

```
~5 = -6

~8 = -9

~0 = -1
```

---

# Left Shift (`<<`)

Moves every bit to the left.

Example:

```python
5 << 1
```

```
5

00000101
```

Shift left:

```
00001010
```

Result:

```
10
```

Notice:

```
5 × 2 = 10
```

Shift again:

```
00010100
```

Result:

```
20
```

So:

```
x << n

=

x × (2ⁿ)
```

Examples:

```
5 << 2 = 20

5 << 3 = 40
```

---

# Right Shift (`>>`)

Moves bits to the right.

Example:

```
20

00010100
```

Shift right:

```
00001010
```

Result:

```
10
```

Shift again:

```
00000101
```

Result:

```
5
```

So:

```
x >> n

=

floor(x / 2ⁿ)
```

For positive integers.

---

# Why are shifts fast?

Multiplication:

```
37 × 2
```

requires arithmetic.

Shifting:

```
37 << 1
```

just moves bits.

Modern CPUs make both operations very fast, but shifts are still fundamental in low-level programming.

---

# Where are bitwise operators used?

### Permissions

```
Read     = 001

Write    = 010

Execute  = 100
```

A user with read and write permissions:

```
011
```

---

### Networking

IP addresses

Subnet masks

Routing tables

---

### Graphics

RGB colors

```
Red

Green

Blue
```

stored in bits.

---

### Compression

ZIP

PNG

JPEG

---

### Cryptography

Encryption algorithms rely heavily on XOR.

---

### Competitive Programming

- Bit masking
    
- Subset generation
    
- State compression
    
- Dynamic Programming with bitmasks
    

---

# Operator precedence

From highest to lowest among bitwise operators:

```
~

<< >>

&

^

|
```

Example:

```python
5 | 2 & 1
```

evaluates as:

```python
5 | (2 & 1)
```

because `&` has higher precedence than `|`.

---

# Summary Table

|Operator|Meaning|Example|
|---|---|---|
|`&`|AND|Keeps bits that are `1` in both numbers|
|`|`|OR|
|`^`|XOR|Sets a bit if the bits are different|
|`~`|NOT|Flips bits (`~x = -(x + 1)`)|
|`<<`|Left shift|Multiplies by `2ⁿ` (for non-overflowing integers)|
|`>>`|Right shift|Divides by `2ⁿ` using floor division for non-negative integers|

### Operator Precedence (high to low, simplified)

```
()                    - parentheses
**                    - exponent
+x, -x, ~x            - unary
*, /, //, %           - multiplication/division
+, -                  - addition/subtraction
<<, >>                - bitwise shifts
&                     - bitwise AND
^                     - bitwise XOR
|                     - bitwise OR
==, !=, >, <, >=, <=  - comparisons
not                   - logical NOT
and                   - logical AND
or                    - logical OR
```

---

## 6. Conditional Statements

### `if`

```python
age = 20
if age >= 18:
    print("Adult")
```

### `if-else`

```python
if age >= 18:
    print("Adult")
else:
    print("Minor")
```

### `if-elif-else`

```python
score = 75
if score >= 90:
    grade = 'A'
elif score >= 75:
    grade = 'B'
elif score >= 60:
    grade = 'C'
else:
    grade = 'F'
```

### Nested Conditions

```python
if age >= 18:
    if has_id:
        print("Entry allowed")
    else:
        print("Need ID")
else:
    print("Too young")
```

### Ternary (Conditional) Expression

```python
status = "Adult" if age >= 18 else "Minor"
```
#### Example 1

Normal `if-elif-else`:

```python
if marks >= 90:
    grade = "A"
elif marks >= 75:
    grade = "B"
else:    
	grade = "C"
```

Ternary:

```python
grade = "A" if marks >= 90 else "B" if marks >= 75 else "C"
```
#### Example 2 (Multiple conditions)

Normal:

```python
if x > 0:
    result = "Positive"
elif x < 0:
    result = "Negative"
else:
    result = "Zero"
```

Ternary:

```python
result = "Positive" if x > 0 else "Negative" if x < 0 else "Zero"
```



### `switch-case` Equivalent in Python

> [!important]
> Python had **no native `switch` statement** until Python 3.10, which introduced **structural pattern matching** via `match-case`. Before that, developers used `if-elif` chains or dictionaries to simulate switch behavior.

**Modern way — `match-case` (Python 3.10+):**

```python
day = 3
match day:
    case 1:
        result = "Monday"
    case 2:
        result = "Tuesday"
    case 3:
        result = "Wednesday"
    case _:                      # default case (like 'else')
        result = "Unknown"
```

**`match` also supports pattern matching on structure, not just values:**

```python
command = ("move", 10, 20)
match command:
    case ("move", x, y):
        print(f"Move to {x},{y}")
    case ("stop",):
        print("Stopping")
    case _:
        print("Unknown command")
```

**Older/dictionary-based simulation (works in all Python versions):**

```python
def switch_case(day):
    cases = {
        1: "Monday",
        2: "Tuesday",
        3: "Wednesday"
    }
    return cases.get(day, "Unknown")     # .get() provides the default
```

---

## 7. Loops

### `for` Loop

```python
for i in range(5):          # 0,1,2,3,4
    print(i)

for i in range(2, 10, 2):   # start, stop(exclusive), step -> 2,4,6,8
    print(i)

for char in "hello":
    print(char)

for item in [1, 2, 3]:
    print(item)

for key, value in {"a": 1, "b": 2}.items():
    print(key, value)

for index, value in enumerate(["a", "b", "c"]):   # get index + value
    print(index, value)

for x, y in zip([1,2,3], ['a','b','c']):          # iterate multiple lists together
    print(x, y)
```

### `while` Loop

```python
i = 0
while i < 5:
    print(i)
    i += 1
```

```python
while True:               # infinite loop
    response = input("Continue? (y/n): ")
    if response == 'n':
        break
```

### Loop Control Statements

```python
for i in range(10):
    if i == 5:
        break          # exit the loop entirely
    print(i)

for i in range(10):
    if i % 2 == 0:
        continue       # skip rest of this iteration, go to next
    print(i)

for i in range(5):
    pass                # placeholder, does nothing (syntax filler)
```

### `else` Clause on Loops (Python-specific feature)

```python
for i in range(5):
    print(i)
else:
    print("Loop completed without break")   # runs only if loop wasn't broken
```

> [!note]
> The `else` block on a `for`/`while` loop executes **only if the loop completes normally** (no `break`). Uncommon but useful for search patterns.

### Nested Loops

```python
for i in range(3):
    for j in range(3):
        print(i, j)
```

---

## 8. Functions Overview

### Defining and Calling

```python
def greet(name):
    return f"Hello, {name}!"

greet("Alice")
```

### Default Arguments

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

greet("Bob")                  # uses default -> "Hello, Bob!"
greet("Bob", "Hi")             # overrides default -> "Hi, Bob!"
```

### Positional vs Keyword Arguments

```python
def describe(name, age):
    print(f"{name} is {age}")

describe("Alice", 30)              # positional
describe(age=30, name="Alice")     # keyword — order doesn't matter
```

### `*args` and `**kwargs`

> Internally *args creates a tuple
```python
def total(*args):                  # collects extra positional args into a tuple
    return sum(args)

total(1, 2, 3, 4)    # -> 10

# **kwargs takes internally as dictionary
def show_info(**kwargs):           # collects extra keyword args into a dict
    for key, value in kwargs.items():
        print(key, value)

show_info(name="Alice", age=30)
```
Basically "* and ** " do the tuple and dictionary unpacking in internally 
## Closure

#### Definition

> **A closure is a function that remembers and can access variables from its enclosing scope, even after the enclosing function has finished executing.**

This is the key idea:

> **The outer function is gone, but its variables are still available to the inner function.**

---

### Normal function

```python
def outer():
    x = 10
    print(x)
    
outer()
```

Memory:

```
Call outer()
Local Scope-----------x = 10
```

When `outer()` finishes:

```
Local Scope-----------Destroyed
```

The variable `x` disappears.

---

# Closure example

```python
def outer():
    x = 10
    def inner():
        print(x)
    return inner
```

Now:

```python
f = outer()
```

People often think:

```
outer() finished↓x destroyed
```

But that's **not** what happens.

Python notices that `inner()` still needs `x`.

So instead:

```
outer()-----------------x = 10        \         \          inner()
```

When `outer()` returns:

```
f │ ▼inner()x = 10
```

The variable `x` stays alive.

---

### Complete example

```python
def outer():
    x = 100
    def inner():
        print(x)
        return inner

f = outer()
f()
```

Output:

```
100
```

Notice:

```
outer() already finished.
Yet x still exists.
```

That is a closure.

---
### Return Values

```python
def add(a, b):
    return a + b               # single return value

def divide(a, b):
    return a // b, a % b       # multiple return values -> returned as a tuple

quotient, remainder = divide(10, 3)
```

- A function with no `return` statement returns `None` by default.

### Variable Scope

```python
x = 10                       # global scope

def func():
    x = 5                    # local scope — separate variable, shadows global
    print(x)                 # 5

func()
print(x)                     # 10 (unchanged)

def modify_global():
    global x
    x = 99                   # explicitly modifies the global variable

modify_global()
print(x)                     # 99
```

### Lambda (Anonymous) Functions

```python
square = lambda x: x ** 2
square(5)          # -> 25

add = lambda a, b: a + b
add(2, 3)           # -> 5

# Common use: as a key function
sorted([(1,'b'), (2,'a')], key=lambda pair: pair[1])
```

### Docstrings

```python
def greet(name):
    """
    Returns a greeting message for the given name.

    Parameters:
        name (str): The name to greet.

    Returns:
        str: A greeting string.
    """
    return f"Hello, {name}!"

print(greet.__doc__) # access the docstring programmatically
print(greet.doc()) # it returns list    
```

### Recursion

```python
def factorial(n):
    if n == 0:
        return 1
    return n * factorial(n - 1)

factorial(5)   # -> 120
```

---

## 9. OOP (Object-Oriented Programming) Overview

### Classes and Objects

```python
class Person:
    def __init__(self, name, age):     # constructor
        self.name = name                # instance attribute
        self.age = age

    def greet(self):                    # instance method
        return f"Hi, I'm {self.name}"

p = Person("Alice", 30)     # create an object (instance)
p.greet()                    # -> "Hi, I'm Alice"
```

- `self` refers to the current instance — always the first parameter of instance methods.
- `__init__` is the constructor, automatically called when creating an object.

### The Four Pillars of OOP

#### 1. Encapsulation — bundling data and methods, restricting direct access

```python
class Account:
    def __init__(self, balance):
        self._balance = balance          # convention: "protected" (single underscore)
        self.__pin = "1234"              # "private" (double underscore -> name mangling)

    def get_balance(self):
        return self._balance

    def deposit(self, amount):
        if amount > 0:
            self._balance += amount
```

> [!note]
> Python has no true private members — `_var` is a convention meaning "internal use," and `__var` triggers **name mangling** (`_ClassName__var`) to avoid accidental access, but it's still technically reachable.

#### 2. Inheritance — a class derives from another

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "Some sound"

class Dog(Animal):                  # Dog inherits from Animal
    def speak(self):                # method overriding
        return f"{self.name} says Woof!"

class Puppy(Dog):                   # multi-level inheritance
    def speak(self):
        return super().speak() + " (in a small voice)"   # super() calls parent method

d = Dog("Rex")
d.speak()      # -> "Rex says Woof!"
```

```python
class A: pass
class B: pass
class C(A, B): pass    # multiple inheritance — C inherits from both A and B
```

#### 3. Polymorphism — same interface, different behavior

```python
class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow!"

animals = [Dog("Rex"), Cat("Whiskers")]
for a in animals:
    print(a.speak())     # each object responds differently to the same method call
```

#### 4. Abstraction — hiding implementation details behind a simple interface

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass                          # subclasses MUST implement this

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return 3.14159 * self.radius ** 2

# Shape()          # TypeError: can't instantiate abstract class
c = Circle(5)
c.area()            # -> 78.53975
```

### Class Attributes vs Instance Attributes

```python
class Dog:
    species = "Canis familiaris"       # class attribute — shared by ALL instances

    def __init__(self, name):
        self.name = name                # instance attribute — unique per object

d1 = Dog("Rex")
d2 = Dog("Max")
d1.species       # "Canis familiaris" (shared)
d1.name          # "Rex" (unique to d1)
```

### Class Methods and Static Methods

```python
class MyClass:
    count = 0

    def __init__(self):
        MyClass.count += 1

    @classmethod
    def get_count(cls):              # operates on the class itself, not an instance
        return cls.count

    @staticmethod
    def utility_function(x, y):      # doesn't need self or cls, just grouped logically
        return x + y

MyClass.get_count()
MyClass.utility_function(2, 3)
```

### Dunder (Magic/Special) Methods

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __str__(self):                # controls str(obj) / print(obj)
        return f"Point({self.x}, {self.y})"

    def __repr__(self):                # controls repr(obj), used in debugging/console
        return f"Point(x={self.x}, y={self.y})"

    def __eq__(self, other):           # controls == comparison
        return self.x == other.x and self.y == other.y

    def __add__(self, other):          # controls + operator (operator overloading)
        return Point(self.x + other.x, self.y + other.y)

    def __len__(self):                 # controls len(obj)
        return 2
```

### Property Decorators (Getter/Setter)

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):                  # acts like an attribute but runs code
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius can't be negative")
        self._radius = value

c = Circle(5)
c.radius            # calls the getter -> 5
c.radius = 10       # calls the setter
```

---

## 10. Quick Reference Cheat Table

| Concept | Example |
|---|---|
| Variable assignment | `x = 10` |
| Type check | `type(x)`, `isinstance(x, int)` |
| f-string | `f"{name} is {age}"` |
| Input as int | `int(input())` |
| Explicit cast | `int()`, `float()`, `str()`, `bool()` |
| Comparison | `==`, `!=`, `is`, `in` |
| if-elif-else | `if x: ... elif y: ... else: ...` |
| Modern switch | `match x: case 1: ... case _: ...` |
| For loop | `for i in range(n):` |
| While loop | `while cond:` |
| List comprehension | `[x for x in range(10) if x%2==0]` |
| Function | `def f(a, b=1, *args, **kwargs): return ...` |
| Lambda | `lambda x: x**2` |
| Class | `class C: def __init__(self): ...` |
| Inheritance | `class Child(Parent):` |
| Abstract method | `@abstractmethod` |

---

> [!tip] See also
> [[NumPy_Cheat_Sheet]] · [[NumPy_Advanced_Notes]] · [[Pandas_Data_Handling_Notes]] · [[Python_File_Handling_Notes]]

[^1]: 

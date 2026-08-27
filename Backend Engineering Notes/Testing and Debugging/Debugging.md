## 🐞 Debugging in Python 

---

### What is Debugging?

**Debugging** is the process of finding, understanding, and fixing errors (bugs) in a program.

A **bug** is any mistake in the code that causes:

- Incorrect output
    
- Program crash
    
- Unexpected behavior
    
- Performance issues
    

> Writing code is only half the job. A significant part of software development is debugging.

---

# Types of Errors

## 1. Syntax Errors

Occur when Python cannot understand the code because it violates Python's grammar.

Example:

```python
if True
    print("Hello")
```

Output:

```
SyntaxError: expected ':'
```

Characteristics:

- Detected before execution
    
- Program does not start
    
- Easy to identify from the traceback
    

---

## 2. Runtime Errors (Exceptions)

Occur while the program is running.

Example:

```python
print(10 / 0)
```

Output:

```
ZeroDivisionError
```

Examples:

- IndexError
    
- KeyError
    
- ValueError
    
- TypeError
    
- ZeroDivisionError
    
- FileNotFoundError
    

---

## 3. Logical Errors

The program runs successfully but produces incorrect results.

Example:

```python
def area(length, width):
    return length + width
```

Output:

```
Wrong Answer
```

No exception occurs, making logical errors the hardest to find.

---

# Debugging Process

```
Identify Problem
        ↓
Reproduce Problem
        ↓
Locate Bug
        ↓
Understand Cause
        ↓
Fix Code
        ↓
Test Again
```

Never fix a bug before understanding **why** it happened.

---

# Reading Error Messages

Example:

```python
numbers = [1, 2, 3]

print(numbers[5])
```

Output:

```
IndexError: list index out of range
```

Break it down:

```
IndexError
```

Type of error.

```
list index out of range
```

Reason.

```
File "...", line 3
```

Location.

Always read the traceback **from bottom to top**, because the last line usually contains the actual exception.

---

# Print Debugging

The simplest debugging technique.

Example:

```python
a = 5
b = 10

print(a)
print(b)

print(a + b)
```

Useful for checking:

- Variable values
    
- Loop iterations
    
- Function calls
    
- Conditional branches
    

---

# Debugging with `repr()`

Sometimes `print()` hides important characters.

Example:

```python
text = "Hello\nWorld"

print(text)
```

Output:

```
Hello
World
```

Using `repr()`:

```python
print(repr(text))
```

Output:

```
'Hello\nWorld'
```

Useful for debugging strings and invisible characters.

---

# Using Assertions

Assertions verify assumptions during development.

Example:

```python
age = 20

assert age >= 18
```

If false:

```
AssertionError
```

Custom message:

```python
assert age >= 18, "Age must be at least 18"
```

Use assertions to catch impossible states while developing.

---

# Using the Python Debugger (`pdb`)

Python includes a built-in debugger.

```python
import pdb

pdb.set_trace()
```

Execution pauses, allowing inspection.

Common commands:

| Command  | Description            |
| -------- | ---------------------- |
| `n`      | Next line              |
| `s`      | Step into function     |
| `c`      | Continue execution     |
| `l`      | List source code       |
| `p var`  | Print variable         |
| `pp var` | Pretty-print variable  |
| `q`      | Quit debugger          |
| `bt`     | Show stack trace       |
| `where`  | Current stack location |

Example:

```python
import pdb

a = 5
b = 10

pdb.set_trace()

print(a + b)
```

---

# Using `breakpoint()`

Since Python 3.7:

```python
breakpoint()
```

Equivalent to:

```python
import pdb

pdb.set_trace()
```

Preferred because it is cleaner and configurable.

---

# Debugging in VS Code

Useful features:

- Breakpoints
    
- Step Over
    
- Step Into
    
- Step Out
    
- Continue
    
- Restart
    
- Stop
    

Panels:

- Variables
    
- Watch
    
- Call Stack
    
- Debug Console
    

---

# Understanding the Call Stack

Every function call creates a stack frame.

Example:

```python
def A():
    B()

def B():
    C()

def C():
    print(10 / 0)

A()
```

Call stack:

```
A()
 ↓
B()
 ↓
C()
 ↓
ZeroDivisionError
```

The traceback reflects this call stack.

---

# Common Debugging Strategies

## Binary Search Debugging

If the codebase is large:

1. Test halfway.
    
2. Decide which half contains the bug.
    
3. Repeat.
    

This narrows the search quickly.

---

## Rubber Duck Debugging

Explain your code line by line, even to an inanimate object. Verbalizing the logic often reveals mistakes.

---

## Divide and Conquer

Break a large problem into smaller parts and verify each independently.

---

## Isolate the Bug

Create the smallest program that still reproduces the problem. This is often called a **Minimal Reproducible Example (MRE)**.

---

# Common Mistakes While Debugging

❌ Guessing instead of investigating.

❌ Changing many things at once.

❌ Ignoring the traceback.

❌ Assuming a library is at fault before checking your own code.

❌ Not testing edge cases after a fix.

---

# Debugging Recursive Functions

Always verify:

- Base case
    
- Recursive call
    
- Return value
    
- Progress toward the base case
    

Helpful print statements:

```python
def factorial(n):
    print("Entering:", n)

    if n == 0:
        return 1

    result = n * factorial(n - 1)

    print("Returning:", result)

    return result
```

---

# Debugging Loops

Check:

- Loop condition
    
- Loop variable updates
    
- Off-by-one errors
    
- Infinite loops
    

Example:

```python
for i in range(5):
    print(i)
```

Output:

```
0
1
2
3
4
```

Remember that `range(5)` stops before `5`.

---

# Debugging Functions

Verify:

- Arguments received
    
- Return values
    
- Side effects
    
- Local variable changes
    

Example:

```python
def add(a, b):
    print(a, b)
    return a + b
```

---

# Debugging Collections

Print or inspect collections to verify their contents.

```python
numbers = [1, 2, 3]

print(numbers)
```

For dictionaries:

```python
student = {
    "name": "John",
    "age": 20
}

print(student)
```

---

# Debugging with Unit Tests

Automated tests can catch bugs early.

```python
def add(a, b):
    return a + b

assert add(2, 3) == 5
assert add(-1, 1) == 0
```

---

# Best Practices

- Read the complete traceback.
    
- Reproduce the bug consistently.
    
- Change one thing at a time.
    
- Verify assumptions with prints, logging, or a debugger.
    
- Use descriptive variable names.
    
- Write small, testable functions.
    
- Add tests after fixing a bug to prevent regressions.
    

---

# Interview Questions

1. What is debugging?
    
2. What is the difference between syntax, runtime, and logical errors?
    
3. What is a traceback?
    
4. What is the call stack?
    
5. What is `pdb`?
    
6. What is `breakpoint()`?
    
7. Difference between `print()` debugging and logging?
    
8. What is an assertion?
    
9. What is a breakpoint?
    
10. What is a Minimal Reproducible Example (MRE)?
    
11. Why is logging preferred over `print()` in production?
    
12. How do you debug recursion?
    

---

# Key Takeaways

- Debugging is the systematic process of locating and fixing bugs.
    
- Errors are commonly classified as **syntax**, **runtime**, and **logical**.
    
- Learn to read tracebacks carefully—they usually point you to the source of the problem.
    
- Start with simple tools (`print()`, `repr()`, `assert`), then use more advanced tools (`logging`, `breakpoint()`, `pdb`, IDE debuggers`) as needed.
    
- A disciplined debugging process saves time and leads to more reliable code.
---
tags: [python, error-handling, exceptions, programming, computer-science]
created: 2026-07-08
aliases: [Exception Handling, Python Exceptions, Error Handling]
---

# Exception Handling in Python

## 1. What is an Exception?

An **exception** is an event that occurs during program execution that disrupts the normal flow of instructions. In Python, an exception is an **object** — an instance of a class that inherits (directly or indirectly) from `BaseException`.

> [!note] Key Idea
> When something goes wrong, Python **raises** an exception object. If nothing catches it, the program terminates and prints a **traceback**. Exception handling lets you intercept that object and respond gracefully instead of crashing.

### Why Exception Handling Matters
- Prevents abrupt program termination
- Separates error-handling logic from normal business logic
- Allows graceful fallback / recovery behavior
- Enables guaranteed resource cleanup (files, sockets, DB connections)
- Provides meaningful, actionable feedback instead of raw crashes
- Improves debuggability via tracebacks and custom messages

---

## 2. Types of Errors (General Classification)

### 2.1 Syntax Errors
Detected **before** execution, while Python parses the code. The interpreter refuses to even start running.

```python
if True
    print("missing colon")
# SyntaxError: expected ':'
```

Common causes: missing colons, mismatched brackets/quotes, bad indentation, misspelled keywords.

### 2.2 Runtime Errors (Exceptions)
The code is syntactically valid, but something goes wrong **while running**, depending on input/state/environment. This is what "exception handling" deals with.

```python
a, b = 10, 0
print(a / b)   # ZeroDivisionError — raised only when this line actually runs
```

### 2.3 Logical Errors
The program runs fine, raises no exception, but produces **incorrect results** due to flawed logic.

```python
def average(a, b):
    return a + b  # bug: forgot to divide by 2
```

> [!warning] Important
> Logical errors are the hardest to catch because Python doesn't raise anything — the code "works." These are found through testing, assertions, and code review, not `try/except`.

### Summary Table

| Error Type | Detected When | Detected By | Example |
|---|---|---|---|
| Syntax Error | Before execution (parse time) | Python parser | Missing `:` |
| Runtime Error (Exception) | During execution | Python interpreter | `ZeroDivisionError` |
| Logical Error | Runs, but wrong output | Testing / review | Wrong formula |

---

## 3. Python Exception Hierarchy

All exceptions inherit from `BaseException`. In practice, you almost always work with `Exception` and its subclasses.

```
BaseException
├── SystemExit          (raised by sys.exit())
├── KeyboardInterrupt    (Ctrl+C)
├── GeneratorExit
└── Exception            ← base class for almost all "normal" exceptions
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   ├── OverflowError
    │   └── FloatingPointError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── ValueError
    ├── TypeError
    ├── AttributeError
    ├── NameError
    │   └── UnboundLocalError
    ├── OSError (aka IOError, EnvironmentError)
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   ├── TimeoutError
    │   └── ConnectionError
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── StopIteration
    ├── RuntimeError
    │   ├── RecursionError
    │   └── NotImplementedError
    └── AssertionError
```

> [!warning] Never catch `BaseException` casually
> `BaseException` includes `SystemExit` and `KeyboardInterrupt`. Catching it broadly (`except BaseException:`) can swallow a user's Ctrl+C or prevent a clean program exit. Catch `Exception` instead, unless you have a very specific reason not to.

> [!info] No Checked vs Unchecked Distinction
> Unlike Java, Python does **not** have "checked exceptions" that the compiler forces you to declare or handle. **All exceptions in Python are effectively unchecked** — nothing stops you from not catching one; it simply propagates up and crashes the program if unhandled.

---

## 4. Common Built-in Exceptions

| Exception | Raised When |
|---|---|
| `ZeroDivisionError` | Dividing by zero |
| `ValueError` | Right type, but invalid value (e.g. `int("abc")`) |
| `TypeError` | Operation applied to an inappropriate type (e.g. `"a" + 5`) |
| `IndexError` | Sequence index out of range |
| `KeyError` | Dictionary key not found |
| `AttributeError` | Object has no such attribute/method |
| `NameError` | Variable/name not defined |
| `FileNotFoundError` | File path doesn't exist |
| `ImportError` / `ModuleNotFoundError` | Module can't be imported |
| `StopIteration` | Iterator has no more items (used internally by `for` loops) |
| `RecursionError` | Maximum recursion depth exceeded |
| `AssertionError` | An `assert` statement fails |
| `PermissionError` | Insufficient OS permission for the operation |
| `TimeoutError` | Operation exceeded its time limit |
| `OverflowError` | Numeric result too large to represent |
| `NotImplementedError` | Abstract method not overridden by subclass |

---

## 5. Basic `try` / `except` Syntax

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"Cannot divide by zero: {e}")
```

### Catching Multiple Specific Exceptions
```python
try:
    value = int(input("Enter a number: "))
    result = 10 / value
except ZeroDivisionError:
    print("You can't divide by zero.")
except ValueError:
    print("That wasn't a valid number.")
```

### Catching Multiple Exceptions in One Block
```python
try:
    risky_operation()
except (ValueError, TypeError) as e:
    print(f"Invalid input or type: {e}")
```

### Catching Any Exception (Generic Handler)
```python
try:
    risky_operation()
except Exception as e:
    print(f"Something went wrong: {type(e).__name__}: {e}")
```

> [!tip] Ordering Rule
> Always put **specific exceptions before generic ones**. Python checks `except` blocks top-to-bottom and stops at the first match — a broad `except Exception` placed first will silently swallow everything below it.

---

## 6. `else` and `finally` Clauses

Python's `try` statement supports **four** blocks: `try`, `except`, `else`, `finally`.

```python
try:
    result = 10 / 2
except ZeroDivisionError:
    print("Division failed")
else:
    print(f"Success! Result = {result}")   # runs ONLY if no exception occurred
finally:
    print("This always runs — success, failure, or return")
```

| Block | Runs When |
|---|---|
| `try` | Always attempted first |
| `except` | Only if a matching exception is raised inside `try` |
| `else` | Only if **no** exception was raised in `try` |
| `finally` | **Always** — regardless of exception, return, or break |

> [!tip] Why use `else`?
> Code in `else` only runs if `try` succeeded, keeping the "happy path" separate from the exception-prone code — this avoids accidentally catching exceptions raised by code that should run *after* the risky operation succeeds.

> [!success] `finally` for Cleanup
> `finally` is ideal for releasing resources (closing files, network connections, releasing locks) since it runs no matter what — even if a `return`, `break`, or unhandled exception occurs.

```python
def read_file(path):
    f = None
    try:
        f = open(path)
        return f.read()
    except FileNotFoundError:
        print("File not found")
        return None
    finally:
        if f:
            f.close()
            print("File closed")
```

---

## 7. Raising Exceptions Manually — `raise`

```python
def withdraw(balance, amount):
    if amount > balance:
        raise ValueError("Insufficient funds!")
    return balance - amount
```

### Re-raising an Exception
```python
try:
    risky_operation()
except ValueError as e:
    print("Logging the error...")
    raise   # re-raises the SAME exception, preserving the original traceback
```

### Exception Chaining (`raise ... from ...`)
Used to indicate one exception was caused by another — preserves both tracebacks for debugging.

```python
try:
    connect_to_db()
except ConnectionError as e:
    raise RuntimeError("Failed to initialize application") from e
```

Output shows:
```
ConnectionError: ...
The above exception was the direct cause of the following exception:
RuntimeError: Failed to initialize application
```

### Suppressing Chaining Context
```python
raise ValueError("Custom error") from None
# Hides the "during handling of the above exception" context
```

---

## 8. Custom (User-Defined) Exceptions

Create custom exceptions by subclassing `Exception` (not `BaseException`).

```python
class InsufficientBalanceError(Exception):
    """Raised when a withdrawal exceeds the available balance."""
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        message = f"Cannot withdraw {amount}; balance is only {balance}."
        super().__init__(message)


def withdraw(balance, amount):
    if amount > balance:
        raise InsufficientBalanceError(balance, amount)
    return balance - amount


try:
    withdraw(100, 150)
except InsufficientBalanceError as e:
    print(e)
    print(f"Shortfall: {e.amount - e.balance}")
```

### Building a Custom Exception Hierarchy
```python
class AppError(Exception):
    """Base exception for this application."""

class ValidationError(AppError):
    """Raised when input validation fails."""

class DatabaseError(AppError):
    """Raised when a database operation fails."""

# Catching the base class also catches all subclasses:
try:
    ...
except AppError as e:
    print(f"Application error occurred: {e}")
```

> [!tip] Best Practice
> Define a **base exception class** for your application/module (e.g., `AppError`), then derive specific exceptions from it. This lets calling code catch broadly (`except AppError`) or narrowly (`except ValidationError`) as needed.

---

## 9. The `assert` Statement

Used for internal sanity checks / debugging assumptions — **not** for validating user input in production (asserts can be stripped out with `python -O`).

```python
def divide(a, b):
    assert b != 0, "b must not be zero"
    return a / b
```

If the condition is `False`, Python raises `AssertionError` with the given message.

> [!warning]
> Never use `assert` for validating untrusted input (e.g., user data, API payloads) in production code — assertions are disabled when Python runs with the `-O` (optimize) flag, silently skipping the check.

---

## 10. Context Managers — `with` Statement

The **preferred way** to manage resources that need guaranteed cleanup, replacing manual `try/finally`.

```python
with open("data.txt") as f:
    contents = f.read()
# file is automatically closed, even if an exception occurs inside the block
```

### Multiple Context Managers
```python
with open("in.txt") as fin, open("out.txt", "w") as fout:
    fout.write(fin.read())
```

### Writing Your Own Context Manager (class-based)
```python
class ManagedResource:
    def __enter__(self):
        print("Acquiring resource")
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        print("Releasing resource")
        if exc_type is not None:
            print(f"Handled exception: {exc_type.__name__}")
        return False  # False = don't suppress the exception; True = suppress it
```

### Writing Your Own Context Manager (using `contextlib`)
```python
from contextlib import contextmanager

@contextmanager
def managed_resource():
    print("Acquiring resource")
    try:
        yield "resource_handle"
    finally:
        print("Releasing resource")

with managed_resource() as r:
    print(f"Using {r}")
```

> [!success] Why prefer `with` over `try/finally`?
> It's shorter, less error-prone (impossible to forget the cleanup step), and clearly signals "this resource has managed acquisition/release" to anyone reading the code.

---

## 11. Exception Propagation & Traceback

If an exception isn't caught in the current function, it propagates up the call stack until a matching `except` handles it, or it reaches the top and crashes the program.

```python
def level3():
    return 1 / 0

def level2():
    return level3()

def level1():
    return level2()

level1()
```

```
Traceback (most recent call last):
  File "app.py", line 10, in <module>
    level1()
  File "app.py", line 8, in level1
    return level2()
  File "app.py", line 5, in level2
    return level3()
  File "app.py", line 2, in level3
    return 1 / 0
ZeroDivisionError: division by zero
```

> [!tip] Reading a Traceback
> Read **bottom to top**: the last line is the actual error type + message; the lines above trace the call path (outermost call first, innermost/actual failure point last).

---

## 12. Useful Exception Object Attributes

```python
try:
    1 / 0
except ZeroDivisionError as e:
    print(type(e).__name__)   # 'ZeroDivisionError'
    print(e.args)              # ('division by zero',)
    print(str(e))               # 'division by zero'
    import traceback
    traceback.print_exc()       # prints full traceback without stopping the program
```

### Getting the Full Traceback as a String (for logging)
```python
import traceback

try:
    1 / 0
except ZeroDivisionError:
    error_log = traceback.format_exc()
    # log error_log to a file / monitoring system
```

---

## 13. Exception Groups (Python 3.11+)

Python 3.11 introduced `ExceptionGroup` and the `except*` syntax to handle **multiple unrelated exceptions raised together** (common in concurrent/async code).

```python
try:
    raise ExceptionGroup("multiple failures", [
        ValueError("bad value"),
        TypeError("bad type"),
    ])
except* ValueError as eg:
    print(f"Caught ValueErrors: {eg.exceptions}")
except* TypeError as eg:
    print(f"Caught TypeErrors: {eg.exceptions}")
```

> [!info]
> This is primarily useful with `asyncio.TaskGroup` and other concurrent constructs where several tasks can fail independently and you want to handle each failure type separately.

---

## 14. Best Practices ✅

> [!success] Do's
> - Catch **specific exceptions**, not bare `except:` or overly broad `except Exception:`, whenever possible
> - Use `with` statements / context managers for anything requiring cleanup (files, locks, connections)
> - Create a **custom exception hierarchy** for domain-specific errors in larger applications
> - Log exceptions with full context (`traceback.format_exc()`) rather than just printing the message
> - Use `raise ... from ...` to preserve the original cause when re-raising as a different exception type
> - Validate/sanitize input **before** risky operations where feasible ("fail fast")
> - Use `finally` (or `with`) to guarantee cleanup regardless of success/failure

> [!failure] Don'ts
> - Don't use a bare `except:` (catches literally everything, including `KeyboardInterrupt` and `SystemExit`) — use `except Exception:` at minimum
> - Don't silently swallow exceptions (`except Exception: pass`) — this hides real bugs and makes debugging painful
> - Don't use exceptions for **normal control flow** (e.g., raising an exception just to break out of a loop when a flag/`break` would do)
> - Don't use `assert` to validate untrusted/user input in production
> - Don't catch an exception you can't meaningfully handle — let it propagate instead
> - Don't expose raw internal exception messages/tracebacks directly to end-users in production systems (log internally, show a friendly message externally)

---

## 15. Quick Revision Summary

- **3 error categories:** Syntax (parse-time), Runtime/Exceptions (execution-time), Logical (wrong output, no crash)
- **Hierarchy:** `BaseException` → `Exception` → specific exception classes (`ValueError`, `TypeError`, `KeyError`, etc.)
- **No checked/unchecked distinction** in Python — everything is effectively unchecked
- **Core keywords:** `try`, `except`, `else`, `finally`, `raise`, `assert`, `with`
- **`else`** = runs only on success; **`finally`** = always runs (cleanup)
- **Custom exceptions**: subclass `Exception`, build a hierarchy with a common base class
- **`with`** statements are the preferred, safer alternative to manual `try/finally` for resource management
- **Exception chaining** (`raise ... from ...`) preserves the original cause for debugging
- **Python 3.11+**: `ExceptionGroup` + `except*` for handling multiple simultaneous exceptions
- **Golden rule:** be specific in what you catch, never swallow errors silently, always clean up resources

---

## Related Notes
- [[Python Comprehensions]]
- [[Debugging Techniques]]
- [[Context Managers Deep Dive]]
- [[Logging and Error Monitoring]]
- [[Custom Exception Design Patterns]]

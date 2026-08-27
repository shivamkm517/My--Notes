---
tags: [python, logging, programming, notes]
title: Python Logging Module
---

# Python Logging Module

A deep-dive reference on Python's built-in `logging` module — why it's better than `print()`, how it works internally, and how to configure it properly for real projects.

---

## Table of Contents

- [[#1. Why Use Logging Instead of print()]]
- [[#2. Core Concepts & Architecture]]
- [[#3. Log Levels]]
- [[#4. Basic Usage]]
- [[#5. Loggers]]
- [[#6. Handlers]]
- [[#7. Formatters]]
- [[#8. Filters]]
- [[#9. Logger Hierarchy & Propagation]]
- [[#10. Configuring Logging]]
- [[#11. Logging Exceptions]]
- [[#12. Rotating & Timed Log Files]]
- [[#13. Logging in Multi-Module Projects]]
- [[#14. Logging in Multi-Threaded/Multi-Process Apps]]
- [[#15. Best Practices]]
- [[#16. Common Pitfalls]]
- [[#17. Quick Reference Cheat Sheet]]

---

## 1. Why Use Logging Instead of `print()`

| `print()` | `logging` |
|---|---|
| No severity levels | Has levels (DEBUG, INFO, WARNING, ERROR, CRITICAL) |
| Always goes to stdout | Can go to console, file, network, email, etc. |
| Can't be turned off selectively | Can filter by level/module without touching code |
| No timestamps/context by default | Can auto-include timestamp, module, line number, thread, etc. |
| Hard to disable in production | Easily disabled/redirected via configuration |
| Not thread-safe by design | Thread-safe by design |

> [!tip]
> Rule of thumb: use `print()` for quick, throwaway debugging in a script you'll delete. Use `logging` for anything that will run more than once or that someone else (including future-you) will need to debug.

---

## 2. Core Concepts & Architecture

The `logging` module has **four main components** that work together:

```
Your Code
   │
   ▼
Logger  ──────►  decides IF a message should be processed (based on level)
   │
   ▼
Filter(s) ─────►  optional, fine-grained filtering
   │
   ▼
Handler ───────►  decides WHERE the message goes (console, file, network...)
   │
   ▼
Formatter ─────►  decides HOW the message looks (timestamp, format string)
   │
   ▼
Output (console / file / etc.)
```

| Component | Role |
|---|---|
| **Logger** | The entry point your code calls (`logger.info(...)`). Decides whether a message is "important enough" to process, based on its level. |
| **Handler** | Sends the log record to a destination (console, file, socket, email, etc.). A logger can have multiple handlers. |
| **Formatter** | Defines the exact text layout of a log message (timestamp format, fields shown). Attached to a handler. |
| **Filter** | Optional extra logic to allow/block specific records beyond simple level checks. |

---

## 3. Log Levels

Levels indicate the **severity/importance** of a log message. Each level has a numeric value — a logger/handler only processes messages **at or above** its configured level.

| Level | Numeric Value | When to Use |
|---|---|---|
| `DEBUG` | 10 | Detailed diagnostic info, useful only when diagnosing problems |
| `INFO` | 20 | Confirmation that things are working as expected |
| `WARNING` | 30 | Something unexpected happened, or a potential problem (default level) |
| `ERROR` | 40 | A serious problem — the software failed to do something |
| `CRITICAL` | 50 | A very serious error — the program itself may be unable to continue |

```python
import logging

logging.debug("This is a debug message")
logging.info("This is an info message")
logging.warning("This is a warning message")
logging.error("This is an error message")
logging.critical("This is a critical message")
```

By default, the root logger's level is `WARNING`, so running the above **only shows WARNING, ERROR, and CRITICAL**:
```
WARNING:root:This is a warning message
ERROR:root:This is an error message
CRITICAL:root:This is a critical message
```

> [!note]
> You can also define **custom levels** with `logging.addLevelName()`, though this is rarely needed — the 5 standard levels cover almost all use cases.

---

## 4. Basic Usage

### The Quick Way — `basicConfig()`

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S"
)

logging.debug("Debugging info")
logging.info("Informational message")
logging.warning("Warning message")
logging.error("Error message")
logging.critical("Critical message")
```

Output:
```
2026-07-07 10:15:32 - root - DEBUG - Debugging info
2026-07-07 10:15:32 - root - INFO - Informational message
2026-07-07 10:15:32 - root - WARNING - Warning message
2026-07-07 10:15:32 - root - ERROR - Error message
2026-07-07 10:15:32 - root - CRITICAL - Critical message
```

> [!warning]
> `basicConfig()` only has an effect the **first time** it's called (unless you pass `force=True` in Python 3.8+). If the root logger already has handlers attached, subsequent calls are silently ignored.

```python
logging.basicConfig(level=logging.DEBUG, force=True)  # forces reconfiguration
```

### `basicConfig()` Common Parameters

| Parameter | Purpose |
|---|---|
| `level` | Minimum severity level to process |
| `format` | Format string for messages |
| `datefmt` | Format for `%(asctime)s` |
| `filename` | Log to a file instead of console |
| `filemode` | `'a'` (append, default) or `'w'` (overwrite) |
| `handlers` | List of handler objects to attach |
| `force` | Remove and re-add handlers (Python 3.8+) |

```python
logging.basicConfig(
    filename="app.log",
    filemode="a",
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
```

---

## 5. Loggers

### Getting a Logger
Never log directly with `logging.info()` etc. in real applications — instead, **get a named logger per module**, typically using `__name__`:

```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

logger.debug("Debug message from this module")
```

> [!tip] Why `__name__`?
> Using `__name__` automatically names the logger after the module's dotted path (e.g. `myapp.utils.db`), which makes it trivial to identify **where** a log message came from and to configure logging **per-module** later.

### Logger Methods

```python
logger.debug(msg)
logger.info(msg)
logger.warning(msg)
logger.error(msg)
logger.critical(msg)
logger.exception(msg)   # like error(), but auto-includes traceback (use inside except blocks)
logger.log(level, msg)  # generic method, level passed explicitly
```

### Checking if a Level is Enabled (Performance Optimization)

```python
if logger.isEnabledFor(logging.DEBUG):
    logger.debug("Expensive computation: %s", expensive_function())
```

This avoids calling `expensive_function()` unnecessarily when DEBUG logging is disabled — though passing arguments lazily (see below) usually solves this already.

### Lazy Argument Formatting (Important!)

```python
# GOOD — string formatting only happens if the log level is enabled
logger.debug("User %s logged in at %s", username, timestamp)

# BAD — f-string is ALWAYS evaluated, even if DEBUG is disabled
logger.debug(f"User {username} logged in at {timestamp}")
```

> [!warning] Performance Tip
> Always prefer `%s`-style lazy formatting with extra arguments over f-strings/`.format()` in log calls. The string interpolation is deferred until (and unless) the message actually needs to be emitted, saving CPU cycles in production when DEBUG/INFO logs are disabled.

---

## 6. Handlers

Handlers determine **where** log messages actually go. A single logger can have **multiple handlers**, each with its own level and formatter.

### Common Built-in Handlers

| Handler | Sends logs to... |
|---|---|
| `StreamHandler` | Console (stdout/stderr) |
| `FileHandler` | A file |
| `RotatingFileHandler` | A file that rotates once it hits a size limit |
| `TimedRotatingFileHandler` | A file that rotates on a time interval (daily, hourly, etc.) |
| `SMTPHandler` | Sends log records via email |
| `SysLogHandler` | Sends logs to a Unix syslog daemon |
| `HTTPHandler` | Sends logs to a web server via GET/POST |
| `QueueHandler` | Sends logs to a queue (useful for multiprocessing) |
| `NullHandler` | Discards all records (used in libraries to avoid "no handler" warnings) |

### Example: Console + File Handlers Together

```python
import logging

logger = logging.getLogger("my_app")
logger.setLevel(logging.DEBUG)

# Console handler — only show INFO and above
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)

# File handler — capture everything, including DEBUG
file_handler = logging.FileHandler("app.log")
file_handler.setLevel(logging.DEBUG)

# Attach formatters
formatter = logging.Formatter("%(asctime)s - %(levelname)s - %(message)s")
console_handler.setFormatter(formatter)
file_handler.setFormatter(formatter)

# Add handlers to the logger
logger.addHandler(console_handler)
logger.addHandler(file_handler)

logger.debug("This goes to the file only")
logger.info("This goes to both console and file")
```

> [!note]
> The **logger's level** acts as the first filter; each **handler's level** acts as an additional filter on top of that. A message must pass *both* checks to be emitted by a given handler.

---

## 7. Formatters

A `Formatter` controls the exact text layout of each log record.

### Common Format Attributes

| Attribute | Description |
|---|---|
| `%(asctime)s` | Human-readable timestamp |
| `%(name)s` | Name of the logger |
| `%(levelname)s` | Text level (DEBUG, INFO, etc.) |
| `%(levelno)s` | Numeric level |
| `%(message)s` | The actual log message |
| `%(filename)s` | Filename where the log call was made |
| `%(funcName)s` | Function name where the log call was made |
| `%(lineno)d` | Line number of the log call |
| `%(module)s` | Module name |
| `%(process)d` | Process ID |
| `%(thread)d` | Thread ID |
| `%(threadName)s` | Thread name |

### Example

```python
formatter = logging.Formatter(
    "%(asctime)s | %(name)s | %(levelname)-8s | %(filename)s:%(lineno)d | %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S"
)
```

Output:
```
2026-07-07 10:20:11 | my_app | INFO     | main.py:42 | Server started successfully
```

### JSON-Style Formatting (for log aggregation tools)

```python
import logging
import json

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        return json.dumps(log_data)

handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
```

---

## 8. Filters

Filters give **fine-grained control** beyond simple level thresholds — e.g., only allow messages containing a certain keyword, or from a certain module.

```python
import logging

class OnlyErrorsFilter(logging.Filter):
    def filter(self, record):
        return record.levelno == logging.ERROR  # only allow ERROR, block everything else

logger = logging.getLogger("my_app")
handler = logging.StreamHandler()
handler.addFilter(OnlyErrorsFilter())
logger.addHandler(handler)
```

Filters can be attached to **both loggers and handlers**, and can also be used to **inject extra contextual data** into records:

```python
class ContextFilter(logging.Filter):
    def filter(self, record):
        record.user = "amit"   # adds a custom field to every record
        return True

logger.addFilter(ContextFilter())

formatter = logging.Formatter("%(asctime)s - %(user)s - %(message)s")
```

---

## 9. Logger Hierarchy & Propagation

Loggers are organized in a **dot-separated hierarchy**, mirroring Python's package/module structure. `myapp.db` is a **child** of `myapp`, which is a child of the **root logger**.

```
root
 └── myapp
      ├── myapp.db
      └── myapp.utils
```

### How Propagation Works
By default, a log record processed by a child logger **propagates upward** to all its ancestor loggers' handlers too (in addition to its own).

```python
import logging

logging.basicConfig(level=logging.INFO)  # configures the ROOT logger

logger = logging.getLogger("myapp.db")
logger.info("Query executed")  
# This message propagates up to root's handler and gets printed,
# even though "myapp.db" itself has no handlers attached.
```

### Disabling Propagation

```python
logger = logging.getLogger("myapp.db")
logger.propagate = False   # stop messages from bubbling up to ancestor loggers
```

### Effective Level
If a logger has no explicit level set (`NOTSET`), it inherits the **effective level** from its nearest ancestor that does have one.

```python
logger.getEffectiveLevel()
```

> [!tip]
> This hierarchy is what makes per-module control possible in large codebases: you can set the root logger to `WARNING` globally, but bump a specific noisy module like `myapp.db` up to `DEBUG` only when you're actively debugging it.

```python
logging.getLogger("myapp.db").setLevel(logging.DEBUG)
```

---

## 10. Configuring Logging

There are three main ways to configure logging, in increasing order of flexibility:

### A. `basicConfig()` — Simple Scripts
(See [[#4. Basic Usage]] above.) Good for small scripts and quick tools.

### B. Dictionary Config (`dictConfig`) — Recommended for Real Apps

```python
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "level": "DEBUG",
            "formatter": "standard",
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "INFO",
            "formatter": "standard",
            "filename": "app.log",
            "maxBytes": 1024 * 1024,   # 1 MB
            "backupCount": 3,
        },
    },
    "loggers": {
        "myapp": {
            "handlers": ["console", "file"],
            "level": "DEBUG",
            "propagate": False,
        },
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
logger = logging.getLogger("myapp")
logger.info("App started using dictConfig")
```

> [!tip]
> `dictConfig` is the modern, preferred approach — it's easy to load from JSON/YAML files, keeps configuration separate from code, and is what most frameworks (Django, Flask) use internally.

### C. File-Based Config (`fileConfig`) — Legacy `.ini` Style

```ini
; logging.ini
[loggers]
keys=root,myapp

[handlers]
keys=consoleHandler

[formatters]
keys=simpleFormatter

[logger_root]
level=WARNING
handlers=consoleHandler

[logger_myapp]
level=DEBUG
handlers=consoleHandler
qualname=myapp
propagate=0

[handler_consoleHandler]
class=StreamHandler
level=DEBUG
formatter=simpleFormatter
args=(sys.stdout,)

[formatter_simpleFormatter]
format=%(asctime)s - %(name)s - %(levelname)s - %(message)s
```

```python
import logging.config
logging.config.fileConfig("logging.ini")
logger = logging.getLogger("myapp")
```

> [!note]
> `fileConfig` is older and less flexible than `dictConfig` (e.g., harder to configure filters). Prefer `dictConfig` for new projects.

---

## 11. Logging Exceptions

### `logger.exception()` — Automatically Includes Traceback

```python
import logging

logger = logging.getLogger(__name__)

try:
    1 / 0
except ZeroDivisionError:
    logger.exception("Division failed")  # logs at ERROR level + full traceback
```

Output:
```
ERROR:__main__:Division failed
Traceback (most recent call last):
  File "example.py", line 6, in <module>
    1 / 0
ZeroDivisionError: division by zero
```

> [!warning]
> `logger.exception()` should **only** be called from within an `except` block — it relies on `sys.exc_info()` being active. Calling it outside an exception handler will log "NoneType: None" instead of a real traceback.

### Manually Including Traceback with `exc_info`

```python
try:
    risky_operation()
except Exception:
    logger.error("Something went wrong", exc_info=True)  # same effect as .exception()
```

### Logging Without Interrupting Program Flow

```python
try:
    result = risky_operation()
except Exception:
    logger.exception("Operation failed, continuing with default value")
    result = default_value
```

---

## 12. Rotating & Timed Log Files

Log files can grow indefinitely if not managed — Python provides handlers to automatically rotate them.

### `RotatingFileHandler` — Rotate by File Size

```python
from logging.handlers import RotatingFileHandler
import logging

handler = RotatingFileHandler(
    "app.log",
    maxBytes=5 * 1024 * 1024,  # 5 MB per file
    backupCount=5              # keep app.log, app.log.1, ..., app.log.5
)

logger = logging.getLogger("myapp")
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

Once `app.log` hits 5 MB, it's renamed to `app.log.1`, and a fresh `app.log` is started. Older backups shift (`app.log.1` → `app.log.2`, etc.), and anything beyond `backupCount` is deleted.

### `TimedRotatingFileHandler` — Rotate by Time Interval

```python
from logging.handlers import TimedRotatingFileHandler

handler = TimedRotatingFileHandler(
    "app.log",
    when="midnight",   # rotate daily at midnight
    interval=1,
    backupCount=7       # keep last 7 days
)
```

| `when` value | Rotation Interval |
|---|---|
| `'S'` | Seconds |
| `'M'` | Minutes |
| `'H'` | Hours |
| `'D'` | Days |
| `'midnight'` | Daily, at midnight |
| `'W0'`–`'W6'` | Weekly (0=Monday) |

---

## 13. Logging in Multi-Module Projects

### Recommended Pattern

**`main.py`**
```python
import logging.config
from config import LOGGING_CONFIG
import module_a

logging.config.dictConfig(LOGGING_CONFIG)
logger = logging.getLogger(__name__)

logger.info("Application starting")
module_a.do_something()
```

**`module_a.py`**
```python
import logging

logger = logging.getLogger(__name__)  # name = "module_a"

def do_something():
    logger.debug("Doing something in module_a")
```

> [!tip]
> Configure logging **once**, centrally (typically in your entry-point script, e.g. `main.py` or `app.py`). Every other module should simply call `logging.getLogger(__name__)` and use it — never call `basicConfig()` or attach handlers inside library/utility modules.

### For Library Authors
If you're writing a **library** (not an application), add a `NullHandler` to prevent "No handlers could be found" warnings for users who haven't configured logging themselves:

```python
import logging
logging.getLogger(__name__).addHandler(logging.NullHandler())
```

---

## 14. Logging in Multi-Threaded/Multi-Process Apps

### Threading
The `logging` module is **thread-safe** by default — multiple threads can safely call the same logger/handler without corrupting output, since locks are used internally.

```python
import logging
import threading

logging.basicConfig(level=logging.INFO, format="%(threadName)s: %(message)s")
logger = logging.getLogger(__name__)

def worker():
    logger.info("Worker thread running")

threads = [threading.Thread(target=worker) for _ in range(3)]
for t in threads:
    t.start()
```

### Multiprocessing (Requires Extra Care)
Regular handlers are **not safe** across separate processes (e.g., writing to the same file from multiple processes can interleave/corrupt output). Use `QueueHandler` + `QueueListener` instead:

```python
import logging
from logging.handlers import QueueHandler, QueueListener
import multiprocessing

def worker_process(queue):
    logger = logging.getLogger("worker")
    logger.addHandler(QueueHandler(queue))
    logger.setLevel(logging.INFO)
    logger.info("Message from a subprocess")

if __name__ == "__main__":
    log_queue = multiprocessing.Queue()
    stream_handler = logging.StreamHandler()
    listener = QueueListener(log_queue, stream_handler)
    listener.start()

    p = multiprocessing.Process(target=worker_process, args=(log_queue,))
    p.start()
    p.join()

    listener.stop()
```

All worker processes push log records into a shared queue; a single listener process/thread consumes the queue and writes to the actual handler — avoiding race conditions.

---

## 15. Best Practices

- ✅ Use `logging.getLogger(__name__)` per module — never the root logger directly in application code.
- ✅ Configure logging **once**, centrally, at the application's entry point.
- ✅ Use lazy `%s` formatting instead of f-strings in log calls.
- ✅ Use `logger.exception()` inside `except` blocks to auto-capture tracebacks.
- ✅ Set appropriate levels: `DEBUG` for development, `INFO`/`WARNING` for production.
- ✅ Use `RotatingFileHandler`/`TimedRotatingFileHandler` to prevent unbounded log growth.
- ✅ Use `dictConfig` for non-trivial projects — keep config separate from code (e.g., in YAML/JSON).
- ✅ Add a `NullHandler` in reusable library code.
- ✅ Include contextual info (module, function, line number) in formatters for easier debugging.
- ✅ Use `QueueHandler`/`QueueListener` for multiprocessing scenarios.

---

## 16. Common Pitfalls

| Pitfall | Why It's a Problem | Fix |
|---|---|---|
| Calling `logging.basicConfig()` in every module | Only the first call has any effect; leads to confusing behavior | Configure once, centrally |
| Using f-strings in log calls | String is always evaluated, even if the log level is disabled | Use `%s` lazy formatting |
| Using `logger.exception()` outside an `except` block | Produces meaningless/empty traceback | Only call inside exception handlers |
| Forgetting `propagate = False` on a child logger with its own handlers | Causes **duplicate log lines** (once from child, once via propagation to root) | Set `propagate = False` when appropriate |
| Attaching multiple handlers repeatedly (e.g., in a loop or repeated function calls) | Causes duplicate log lines for every attached handler | Check `logger.handlers` before adding, or configure once at startup |
| Writing to the same log file from multiple processes directly | File corruption / interleaved, garbled lines | Use `QueueHandler` + `QueueListener` |
| Setting root logger level but not handler level (or vice versa) | Messages silently disappear because both must allow the message through | Ensure both logger AND handler levels are set correctly |

---

## 17. Quick Reference Cheat Sheet

```python
import logging

# 1. Get a logger
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

# 2. Create a handler
handler = logging.StreamHandler()  # or FileHandler("app.log")
handler.setLevel(logging.DEBUG)

# 3. Create and attach a formatter
formatter = logging.Formatter("%(asctime)s - %(name)s - %(levelname)s - %(message)s")
handler.setFormatter(formatter)

# 4. Attach the handler to the logger
logger.addHandler(handler)

# 5. Log messages
logger.debug("Debug detail")
logger.info("Informational message")
logger.warning("Something looks off")
logger.error("An error occurred")
logger.critical("Critical failure!")

# 6. Log an exception with traceback
try:
    1 / 0
except ZeroDivisionError:
    logger.exception("Math error occurred")
```

| Level | Method | Numeric |
|---|---|---|
| DEBUG | `logger.debug()` | 10 |
| INFO | `logger.info()` | 20 |
| WARNING | `logger.warning()` | 30 |
| ERROR | `logger.error()` | 40 |
| CRITICAL | `logger.critical()` | 50 |

---

*Tags:* #python #logging #debugging #best-practices

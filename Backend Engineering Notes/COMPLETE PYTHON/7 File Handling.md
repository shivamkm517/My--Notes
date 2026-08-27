---
title: Python File Handling — Complete Notes
tags: [python, file-handling, io, cheatsheet]
---

# Python File Handling — Complete Notes

> [!info] Scope
> Covers opening/closing files, read/write modes, text vs binary, context managers, file object methods, `pathlib`, directory operations, CSV/JSON/pickle, exception handling, encodings, buffering, temp files, and common gotchas.

---

## 1. Opening a File — `open()`

```python
f = open("file.txt", mode="r", encoding="utf-8")
```

**Syntax:**
```python
open(file, mode='r', buffering=-1, encoding=None, errors=None,
     newline=None, closefd=True, opener=None)
```

| Parameter   | Meaning                                                                               |
| ----------- | ------------------------------------------------------------------------------------- |
| `file`      | Path (string or path-like object)                                                     |
| `mode`      | Read/write/append/binary mode (see below)                                             |
| `buffering` | `0` = no buffering, `1` = line buffering, `>1` = buffer size in bytes, `-1` = default |
| `encoding`  | Text encoding (e.g. `'utf-8'`) — only for text mode                                   |
| `errors`    | How encoding errors are handled (`'strict'`, `'ignore'`, `'replace'`)                 |
| `newline`   | Controls universal newline handling                                                   |

---

## 2. File Modes

| Mode | Meaning |
|---|---|
| `'r'` | Read (default). Error if file doesn't exist |
| `'w'` | Write. **Overwrites/truncates** existing file, creates if not present |
| `'a'` | Append. Writes at end of file, creates if not present |
| `'x'` | Exclusive creation. Fails if file already exists |
| `'r+'` | Read and write. File must exist |
| `'w+'` | Write and read. Truncates existing file |
| `'a+'` | Append and read |
| `'b'` | Binary mode (combine, e.g. `'rb'`, `'wb'`, `'ab'`) |
| `'t'` | Text mode (default, e.g. `'rt'`) |

> [!warning] `'w'` truncates immediately
> Opening a file in `'w'` mode **erases its contents instantly**, even before you write anything. Use `'a'` if you want to preserve existing content.

---

## 3. Closing a File

```python
f = open("file.txt", "r")
data = f.read()
f.close()   # always close to release the OS file handle
```

> [!danger] Don't forget to close
> Unclosed files can cause data loss (unflushed write buffers), resource leaks, and locked files on some OSes. Always prefer the context manager below.

---

## 4. The `with` Statement (Context Manager) — Preferred Way

```python
with open("file.txt", "r", encoding="utf-8") as f:
    data = f.read()
# file is automatically closed here, even if an exception occurs
```

**Multiple files at once:**
```python
with open("in.txt") as fin, open("out.txt", "w") as fout:
    fout.write(fin.read())
```

> [!tip] Always use `with`
> It guarantees the file closes even if an exception is raised inside the block — equivalent to a `try/finally`.

---

## 5. Reading From Files

```python
with open("file.txt", "r") as f:
    content = f.read()          # reads entire file as one string
```

```python
with open("file.txt", "r") as f:
    content = f.read(10)        # read only first 10 characters/bytes
```

```python
with open("file.txt", "r") as f:
    line = f.readline()         # reads a single line (includes '\n')
```

```python
with open("file.txt", "r") as f:
    lines = f.readlines()       # list of all lines, each with '\n'
```

**Iterating line by line (most memory-efficient for large files):**
```python
with open("file.txt", "r") as f:
    for line in f:
        print(line.strip())     # strip() removes trailing '\n'
```

---

## 6. Writing to Files

```python
with open("file.txt", "w") as f:
    f.write("Hello, World!\n")   # write() does NOT add newline automatically
```

```python
with open("file.txt", "w") as f:
    lines = ["line1\n", "line2\n", "line3\n"]
    f.writelines(lines)          # writes list of strings, no auto newlines
```

**Appending:**
```python
with open("file.txt", "a") as f:
    f.write("This gets added at the end\n")
```

> [!warning]
> `writelines()` does NOT insert `\n` between items automatically — you must include it in each string yourself.

---

## 7. File Pointer / Cursor Control

```python
f = open("file.txt", "r")

f.tell()          # returns current cursor position (byte offset)
f.seek(0)         # move cursor to the start of file
f.seek(5)         # move cursor to byte 5
f.seek(0, 2)      # move to end of file (whence=2 means "relative to end")
```

**`seek(offset, whence)`** — `whence`:
| Value | Meaning |
|---|---|
| `0` | Absolute position from start (default) |
| `1` | Relative to current position |
| `2` | Relative to end of file |

> [!note]
> In text mode, `seek()` with non-zero offsets and arbitrary `whence` values can behave inconsistently across platforms — safest to use `0`/`2` reliably only in binary mode.

---

## 8. Binary Files

```python
with open("image.png", "rb") as f:
    data = f.read()          # returns bytes object

with open("copy.png", "wb") as f:
    f.write(data)
```

**Reading in chunks (good for large files):**
```python
with open("bigfile.bin", "rb") as f:
    while chunk := f.read(4096):     # walrus operator, Python 3.8+
        process(chunk)
```

---

## 9. Encoding & Newlines

```python
open("file.txt", "r", encoding="utf-8")
open("file.txt", "r", encoding="latin-1")
open("file.txt", "r", encoding="utf-8", errors="ignore")   # skip bad bytes
open("file.txt", "r", encoding="utf-8", errors="replace")  # replace with '�'
```

> [!important]
> Always specify `encoding='utf-8'` explicitly when opening text files. Without it, Python uses the OS default encoding, which differs across platforms (e.g. `cp1252` on Windows vs `utf-8` on Linux) — a common source of bugs.

**Newline handling:**
```python
open("file.txt", "r", newline="")     # disables universal newline translation
open("file.txt", "w", newline="\n")   # force Unix-style line endings on write
```

---

## 10. File Object Attributes & Methods Reference

| Method / Attribute | Description |
|---|---|
| `f.read(size=-1)` | Read `size` chars/bytes, or entire file if omitted |
| `f.readline()` | Read one line |
| `f.readlines()` | Read all lines into a list |
| `f.write(s)` | Write string/bytes, returns number of chars/bytes written |
| `f.writelines(list)` | Write list of strings (no auto newline) |
| `f.close()` | Close the file |
| `f.closed` | `True` if file is closed |
| `f.flush()` | Force write buffer to disk without closing |
| `f.tell()` | Current cursor position |
| `f.seek(offset, whence)` | Move cursor |
| `f.seekable()` | Whether file supports seeking |
| `f.readable()` | Whether file is open for reading |
| `f.writable()` | Whether file is open for writing |
| `f.name` | File path/name |
| `f.mode` | Mode file was opened in |
| `f.encoding` | Encoding used (text mode only) |
| `f.truncate(size=None)` | Resize file to `size` bytes (or current position) |

---

## 11. Exception Handling with Files

```python
try:
    with open("file.txt", "r") as f:
        data = f.read()
except FileNotFoundError:
    print("File does not exist")
except PermissionError:
    print("No permission to access file")
except IsADirectoryError:
    print("Expected a file, got a directory")
except UnicodeDecodeError:
    print("Encoding mismatch while reading file")
except OSError as e:
    print(f"OS error occurred: {e}")
```

**Common file-related exceptions:**
| Exception | When it's raised |
|---|---|
| `FileNotFoundError` | File doesn't exist (subclass of `OSError`) |
| `PermissionError` | Insufficient permissions |
| `IsADirectoryError` | Tried to open a directory as a file |
| `NotADirectoryError` | Expected directory, got file |
| `FileExistsError` | Using `'x'` mode on an existing file |
| `UnicodeDecodeError` | Decoding bytes with wrong encoding |
| `OSError` | Base class for OS-related I/O errors |

---

## 12. Working with Paths — `os.path` (legacy) vs `pathlib` (modern)

### `os.path` (older style)

```python
import os

os.path.exists("file.txt")
os.path.isfile("file.txt")
os.path.isdir("folder")
os.path.join("folder", "file.txt")
os.path.abspath("file.txt")
os.path.basename("/a/b/file.txt")    # 'file.txt'
os.path.dirname("/a/b/file.txt")     # '/a/b'
os.path.splitext("file.txt")         # ('file', '.txt')
os.path.getsize("file.txt")          # size in bytes
```

### `pathlib` (recommended, Python 3.4+)

```python
from pathlib import Path

p = Path("folder/file.txt")

p.exists()
p.is_file()
p.is_dir()
p.name          # 'file.txt'
p.stem          # 'file'
p.suffix        # '.txt'
p.parent        # Path('folder')
p.resolve()     # absolute path

p.read_text(encoding="utf-8")
p.write_text("hello", encoding="utf-8")
p.read_bytes()
p.write_bytes(b"data")

Path("folder") / "file.txt"     # path joining with '/' operator

for file in Path(".").iterdir():
    print(file)

for file in Path(".").glob("*.txt"):
    print(file)

for file in Path(".").rglob("*.py"):   # recursive glob
    print(file)
```

> [!tip] Prefer `pathlib` in new code
> It's object-oriented, cross-platform, and generally more readable than string-based `os.path`.

---

## 13. Directory & File System Operations — `os` / `shutil`

```python
import os
import shutil

os.mkdir("new_folder")            # create single directory
os.makedirs("a/b/c")              # create nested directories
os.makedirs("a/b/c", exist_ok=True)  # no error if exists

os.rmdir("folder")                # remove empty directory
os.removedirs("a/b/c")            # remove nested empty directories
shutil.rmtree("folder")           # remove directory and all contents

os.rename("old.txt", "new.txt")   # rename/move file
os.remove("file.txt")             # delete a file
os.unlink("file.txt")             # same as os.remove

shutil.copy("src.txt", "dst.txt")       # copy file (data only)
shutil.copy2("src.txt", "dst.txt")      # copy file + metadata
shutil.copytree("src_dir", "dst_dir")   # copy entire directory tree
shutil.move("src.txt", "dst_folder/")   # move file or directory

os.listdir(".")                   # list directory contents
os.getcwd()                       # current working directory
os.chdir("/some/path")            # change working directory
os.path.getsize("file.txt")       # file size in bytes
```

---

## 14. Checking File Existence Safely

```python
import os
from pathlib import Path

if os.path.exists("file.txt"):
    ...

if Path("file.txt").exists():
    ...
```

> [!tip] EAFP vs LBYL
> Pythonic style often prefers **"Easier to Ask Forgiveness than Permission"** (try/except) over **"Look Before You Leap"** (checking existence first), since the file's state can change between the check and the actual operation (race condition):
> ```python
> try:
>     with open("file.txt") as f:
>         data = f.read()
> except FileNotFoundError:
>     data = None
> ```

---

## 15. Working with CSV Files

```python
import csv

# Reading
with open("data.csv", "r", newline="") as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)          # each row is a list of strings

# Reading as dictionaries
with open("data.csv", "r", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["column_name"])

# Writing
with open("out.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "age"])
    writer.writerows([["Alice", 30], ["Bob", 25]])

# Writing from dictionaries
with open("out.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "age"])
    writer.writeheader()
    writer.writerow({"name": "Alice", "age": 30})
```

> [!warning]
> Always pass `newline=""` when opening files for `csv` module use — otherwise extra blank lines appear on Windows due to newline translation.

---

## 16. Working with JSON Files

```python
import json

# Reading JSON from file
with open("data.json", "r") as f:
    data = json.load(f)          # returns dict/list

# Writing JSON to file
with open("out.json", "w") as f:
    json.dump(data, f, indent=4)

# String <-> object conversion (no file involved)
json_string = json.dumps(data, indent=4)
data = json.loads(json_string)
```

| Function | Purpose |
|---|---|
| `json.load(f)` | Parse JSON from a file object |
| `json.loads(s)` | Parse JSON from a string |
| `json.dump(obj, f)` | Write JSON to a file object |
| `json.dumps(obj)` | Convert object to JSON string |

---

## 17. Serialization with `pickle` (Python objects, binary)

```python
import pickle

# Save any Python object to disk
with open("data.pkl", "wb") as f:
    pickle.dump(my_object, f)

# Load it back
with open("data.pkl", "rb") as f:
    my_object = pickle.load(f)
```

> [!danger] Security warning
> **Never unpickle data from an untrusted source.** Pickle can execute arbitrary code during deserialization — it is not safe for data received over a network or from unknown users.

---

## 18. Temporary Files & Directories

```python
import tempfile

# Temporary file (auto-deleted when closed, by default)
with tempfile.TemporaryFile(mode="w+") as f:
    f.write("temp data")
    f.seek(0)
    print(f.read())

# Named temp file (has a visible path on disk)
with tempfile.NamedTemporaryFile(delete=False) as f:
    print(f.name)

# Temporary directory
with tempfile.TemporaryDirectory() as tmpdir:
    print(tmpdir)   # deleted automatically when block exits
```

---

## 19. File Locking / Concurrent Access (brief note)

> [!note]
> Python's standard `open()` does not lock files against concurrent access by default. For cross-process file locking, use third-party/platform libraries:
> - `fcntl.flock()` (Unix/Linux only)
> - `msvcrt.locking()` (Windows only)
> - `filelock` (cross-platform third-party package)

---

## 20. Buffering

```python
open("file.txt", "r", buffering=0)    # unbuffered (binary mode only)
open("file.txt", "r", buffering=1)    # line-buffered (text mode)
open("file.txt", "r", buffering=8192) # fixed buffer size in bytes
```

- `buffering=-1` (default): system chooses (typically 4KB–8KB blocks).
- Line buffering (`1`) only works meaningfully in text mode.
- `f.flush()` forces buffered data to be written immediately without closing the file.

---

## 21. Common Gotchas Summary

| Gotcha | Fix |
|---|---|
| Forgetting to close files | Always use `with` |
| `'w'` mode silently erasing data | Use `'a'` to append, or check first |
| Not specifying `encoding` | Always pass `encoding='utf-8'` explicitly |
| Extra blank lines in CSV on Windows | Pass `newline=""` when opening for `csv` |
| `writelines()` missing newlines | Add `'\n'` manually to each string |
| Reading huge files with `.read()` | Iterate line-by-line or read in chunks instead |
| Unpickling untrusted data | Never do it — security risk |
| Relying on OS default encoding | Explicit `encoding=` avoids cross-platform bugs |
| Race conditions with existence checks | Prefer try/except (EAFP) over `if exists()` (LBYL) |

---

## 22. Quick Reference Cheat Table

| Task | Code |
|---|---|
| Open & read whole file | `open(f, 'r', encoding='utf-8').read()` |
| Read line by line | `for line in open(f): ...` |
| Write (overwrite) | `open(f, 'w').write(text)` |
| Append | `open(f, 'a').write(text)` |
| Read binary | `open(f, 'rb').read()` |
| Check existence | `Path(f).exists()` |
| Delete file | `os.remove(f)` |
| Copy file | `shutil.copy(src, dst)` |
| Read JSON | `json.load(open(f))` |
| Read CSV | `csv.reader(open(f, newline=''))` |
| Pickle save/load | `pickle.dump(obj, f)` / `pickle.load(f)` |

---

> [!tip] See also
> [[NumPy_Cheat_Sheet]] and [[NumPy_Advanced_Notes]] for related Python data-handling references.

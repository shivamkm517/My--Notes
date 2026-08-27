---
tags: [python, modules, packages, notes]
title: Python Modules and Packages
---

# Python Modules and Packages

A deep-dive reference on how Python organizes code — modules, packages, imports, and how the import system actually works under the hood.

---

## Table of Contents

- [[#1. What is a Module]]
- [[#2. Ways to Import a Module]]
- [[#3. How Python Finds Modules (The Import System)]]
- [[#4. The `__name__ == "__main__"` Pattern]]
- [[#5. Module Attributes]]
- [[#6. Reloading Modules]]
- [[#7. What is a Package]]
- [[#8. Regular Packages vs Namespace Packages]]
- [[#9. `__init__.py` in Depth]]
- [[#10. Absolute vs Relative Imports]]
- [[#11. Subpackages & Nested Package Structures]]
- [[#12. `__all__` and Controlling Wildcard Imports]]
- [[#13. The `sys.path` and `PYTHONPATH`]]
- [[#14. Standard Library Highlights]]
- [[#15. Installing & Managing Third-Party Packages]]
- [[#16. Creating & Distributing Your Own Package]]
- [[#17. Circular Imports]]
- [[#18. Compiled Files: `__pycache__` and `.pyc`]]
- [[#19. Common Pitfalls]]
- [[#20. Best Practices]]
- [[#21. Quick Reference Cheat Sheet]]

---

## 1. What is a Module?

A **module** is simply a single `.py` file containing Python code — functions, classes, variables, or runnable statements — that can be **imported** and reused in other files.

### Creating a Module

**`mymath.py`**
```python
# mymath.py
PI = 3.14159

def add(a, b):
    return a + b

def square(x):
    return x ** 2
```

### Using the Module

**`main.py`**
```python
import mymath

print(mymath.PI)          # 3.14159
print(mymath.add(2, 3))   # 5
print(mymath.square(4))   # 16
```

> [!note]
> The module's name is just the filename without `.py`. So `mymath.py` becomes the module `mymath`.

### Why Modules Matter
- **Organization** — split large programs into logical, manageable files.
- **Reusability** — write code once, use it across many projects.
- **Namespacing** — avoid naming collisions (`mymath.add` vs some other `add`).
- **Maintainability** — easier to test, debug, and update in isolation.

---

## 2. Ways to Import a Module

### A. Basic Import

```python
import mymath
mymath.add(2, 3)
```

### B. Import with Alias

```python
import mymath as mm
mm.add(2, 3)
```

### C. Import Specific Names

```python
from mymath import add, PI
print(add(2, 3))   # no need for "mymath." prefix
print(PI)
```

### D. Import Specific Name with Alias

```python
from mymath import add as sum_two
print(sum_two(2, 3))
```

### E. Wildcard Import (Import Everything)

```python
from mymath import *
print(add(2, 3))
print(square(4))
```

> [!warning]
> Wildcard imports (`from module import *`) pollute the current namespace and make it unclear where a name came from — this hurts readability and can silently overwrite existing names. Avoid in production code; see [[#12. `__all__` and Controlling Wildcard Imports]].

### F. Conditional / Optional Imports

```python
try:
    import ujson as json   # faster JSON library, if available
except ImportError:
    import json            # fallback to standard library
```

### G. Dynamic Imports with `importlib`

```python
import importlib

module_name = "mymath"
mymath = importlib.import_module(module_name)
print(mymath.add(2, 3))
```

> [!tip]
> `importlib.import_module()` is useful when the module name is only known at runtime (e.g., loading plugins dynamically).

---

## 3. How Python Finds Modules (The Import System)

When you write `import mymath`, Python searches through a specific, ordered list of locations stored in `sys.path`:

```
1. The directory containing the input script (or the current directory, in interactive mode)
2. PYTHONPATH environment variable directories (if set)
3. The standard library directories
4. Site-packages (where pip-installed third-party packages live)
```

```python
import sys
print(sys.path)
# ['', '/usr/lib/python3.11', '/usr/lib/python3.11/site-packages', ...]
```

### The Import Process (Simplified)
1. Check `sys.modules` — has this module already been imported? If yes, reuse the cached module object (imports are **cached**, so a module's top-level code runs only **once** per process, no matter how many times it's imported elsewhere).
2. If not cached, search `sys.path` in order for a matching module/package.
3. Once found, execute the module's code top-to-bottom, and store the resulting module object in `sys.modules`.
4. Bind the module (or specific names) into the importing namespace.

```python
import sys
print("mymath" in sys.modules)  # False, before first import
import mymath
print("mymath" in sys.modules)  # True, after import
```

> [!note]
> This caching behavior is why placing print statements or heavy computations at a module's top level should be done carefully — that code runs once, at first import, not every time the module is referenced afterward.

---

## 4. The `__name__ == "__main__"` Pattern

Every module has a built-in `__name__` attribute:
- If the file is run **directly** (`python mymath.py`), `__name__` is set to `"__main__"`.
- If the file is **imported** by another module, `__name__` is set to the module's actual name (e.g., `"mymath"`).

```python
# mymath.py
def add(a, b):
    return a + b

if __name__ == "__main__":
    # This block only runs when mymath.py is executed directly,
    # NOT when it's imported elsewhere.
    print("Running tests...")
    print(add(2, 3))
```

```bash
python mymath.py       # prints "Running tests..." and "5"
```

```python
import mymath           # does NOT print anything — the __main__ block is skipped
```

> [!tip]
> This pattern is the standard way to make a file usable **both** as an importable module and as a standalone script (e.g., for quick manual testing or CLI entry points).

---

## 5. Module Attributes

Every module object carries useful metadata attributes automatically:

```python
import mymath

print(mymath.__name__)   # 'mymath'
print(mymath.__file__)   # '/path/to/mymath.py'
print(mymath.__doc__)    # module docstring, if defined
print(dir(mymath))       # lists all names defined in the module
```

### Module Docstrings

```python
"""
mymath.py

A small collection of math helper functions.
"""

PI = 3.14159

def add(a, b):
    """Return the sum of a and b."""
    return a + b
```

```python
print(mymath.__doc__)
# '\nmymath.py\n\nA small collection of math helper functions.\n'
```

---

## 6. Reloading Modules

Since modules are cached in `sys.modules`, re-importing an already-imported module does **not** re-execute its code — useful to know, especially in interactive sessions (like Jupyter) where you've edited a file.

```python
import importlib
import mymath

# ... edit mymath.py on disk ...

importlib.reload(mymath)   # forces re-execution of the module's code
```

> [!warning]
> `importlib.reload()` only updates the module object itself. Any objects created **before** the reload from the old version of the module (e.g., class instances) still reference the **old** class definitions — this can create confusing inconsistencies. Reloading is mainly a development/debugging convenience, not something to rely on in production logic.

---

## 7. What is a Package?

A **package** is a way of organizing **related modules** together into a directory hierarchy. Technically, a package is simply a directory that Python treats as a single importable unit (traditionally, one containing an `__init__.py` file).

### Example Package Structure

```
mypackage/
├── __init__.py
├── math_utils.py
├── string_utils.py
└── file_utils.py
```

### Using the Package

```python
import mypackage.math_utils
print(mypackage.math_utils.add(2, 3))

# or
from mypackage import math_utils
print(math_utils.add(2, 3))

# or
from mypackage.math_utils import add
print(add(2, 3))
```

> [!note]
> Think of it this way: a **module** is one file; a **package** is a folder of modules (which may itself contain further folders/subpackages), giving you a namespaced, hierarchical way to organize a larger codebase.

---

## 8. Regular Packages vs Namespace Packages

### Regular Packages (Traditional, Explicit)
Contains an `__init__.py` file — even if empty — which explicitly marks the directory as a package.

```
mypackage/
├── __init__.py    ← required for a "regular" package
└── module_a.py
```

### Namespace Packages (Implicit, PEP 420, Python 3.3+)
A directory **without** an `__init__.py` can still function as a package — Python 3.3+ supports "namespace packages," which allow a single logical package to be **split across multiple directories/distributions**.

```
# No __init__.py needed
mypackage/
└── module_a.py
```

| Aspect | Regular Package | Namespace Package |
|---|---|---|
| Requires `__init__.py` | Yes | No |
| Can span multiple directories/distributions | No | Yes |
| Import behavior | Faster, more predictable | Slightly more overhead (searches multiple paths) |
| Common use case | Standard application/library structure | Large plugin ecosystems (e.g., splitting a package across multiple installable pieces) |

> [!tip]
> Unless you're specifically building a plugin-style ecosystem meant to be extended by separately-installed packages, prefer **regular packages** with an explicit `__init__.py` — it's clearer and more predictable.

---

## 9. `__init__.py` in Depth

`__init__.py` runs automatically whenever the package (or anything inside it) is first imported. It can be empty, or used to:

### A. Mark a Directory as a Package (can be empty)
```python
# mypackage/__init__.py
# (can be completely empty)
```

### B. Control What Gets Imported with the Package

```python
# mypackage/__init__.py
from .math_utils import add, square
from .string_utils import reverse

# Now users can do:
# from mypackage import add, square, reverse
# instead of:
# from mypackage.math_utils import add, square
```

### C. Define Package-Level Variables/Metadata

```python
# mypackage/__init__.py
__version__ = "1.0.0"
__author__ = "Amit"
```

```python
import mypackage
print(mypackage.__version__)  # '1.0.0'
```

### D. Run Package Initialization Logic

```python
# mypackage/__init__.py
print("Initializing mypackage...")   # runs once, on first import of anything in the package
```

---

## 10. Absolute vs Relative Imports

### Absolute Imports (Recommended)
Specify the full path from the project's top-level package.

```python
# Inside mypackage/subpackage/module_b.py
from mypackage.math_utils import add
```

### Relative Imports
Use dots to reference modules relative to the **current package's location**.

```python
# Inside mypackage/subpackage/module_b.py
from . import module_a          # same directory (sibling module)
from .. import math_utils       # one level up (parent package)
from ..string_utils import reverse  # one level up, specific import
```

| Dots | Meaning |
|---|---|
| `.` | Current package |
| `..` | Parent package |
| `...` | Grandparent package |

### Example Structure

```
myapp/
├── __init__.py
├── main.py
├── core/
│   ├── __init__.py
│   ├── engine.py
│   └── config.py
└── utils/
    ├── __init__.py
    └── helpers.py
```

```python
# Inside myapp/core/engine.py

from .config import settings          # relative: same package (core)
from ..utils.helpers import log       # relative: parent, then into utils
from myapp.utils.helpers import log   # absolute equivalent
```

> [!warning]
> Relative imports **only work inside packages** — running a file directly with `python module_b.py` when it uses relative imports raises `ImportError: attempted relative import with no known parent package`. Relative imports require the file to be run/imported as part of a package (e.g., via `python -m myapp.core.engine`).

> [!tip] Best Practice
> Prefer **absolute imports** for clarity, especially in larger projects — they make it immediately obvious where a name comes from, and they don't break if a file is moved to a different depth in the package tree. Relative imports are more common inside self-contained packages/libraries.

---

## 11. Subpackages & Nested Package Structures

Packages can contain other packages, forming deep hierarchies:

```
ecommerce/
├── __init__.py
├── orders/
│   ├── __init__.py
│   ├── models.py
│   └── services.py
├── payments/
│   ├── __init__.py
│   ├── stripe_gateway.py
│   └── paypal_gateway.py
└── users/
    ├── __init__.py
    └── auth.py
```

```python
from ecommerce.orders.models import Order
from ecommerce.payments.stripe_gateway import charge_card
from ecommerce.users.auth import login
```

Each folder in the hierarchy needs its own `__init__.py` (for regular packages) to be recognized as a sub-package.

---

## 12. `__all__` and Controlling Wildcard Imports

`__all__` is a list of strings defined in a module or package's `__init__.py` that explicitly controls what gets imported when someone uses `from module import *`.

```python
# mymath.py
__all__ = ["add", "square"]   # only these are exported via *

PI = 3.14159       # NOT exported via *, since it's not listed
SECRET = "hidden"  # NOT exported via *

def add(a, b):
    return a + b

def square(x):
    return x ** 2

def _internal_helper():   # convention: leading underscore = "private", excluded from * anyway
    pass
```

```python
from mymath import *
print(add(2, 3))     # works — listed in __all__
print(square(4))     # works — listed in __all__
print(PI)             # NameError! not in __all__, and wildcard import doesn't bring it in
```

> [!note]
> `__all__` **only** affects `from module import *`. Direct imports like `from mymath import PI` or `import mymath; mymath.PI` still work regardless of `__all__`.

### Convention: Leading Underscore = "Private"
Names starting with a single underscore (`_internal_helper`) are a **convention** signaling "internal use only." They are automatically excluded from wildcard imports (even without `__all__`), but are still fully accessible via explicit import — Python doesn't enforce true privacy.

---

## 13. The `sys.path` and `PYTHONPATH`

`sys.path` is the list of directories Python searches (in order) when resolving imports.

```python
import sys
print(sys.path)
```

### Modifying `sys.path` at Runtime

```python
import sys
sys.path.append("/path/to/my/custom/modules")
import my_custom_module
```

> [!warning]
> Manually appending to `sys.path` is a quick hack, useful for scripts/notebooks, but is generally discouraged for real projects — prefer proper packaging (installable via `pip install -e .`) or setting the `PYTHONPATH` environment variable instead.

### Using the `PYTHONPATH` Environment Variable

```bash
export PYTHONPATH="/path/to/my/project:$PYTHONPATH"
python main.py
```

This tells Python to also search the given directory when resolving imports, without modifying code.

### Running Modules with `-m`

```bash
python -m myapp.core.engine
```

The `-m` flag runs a module as a script **while still treating it as part of its package** — this is why it's the standard way to run files that use relative imports.

---

## 14. Standard Library Highlights

Python ships with a huge collection of built-in modules/packages — no installation needed.

| Module | Purpose |
|---|---|
| `os` | Operating system interaction (files, paths, environment variables) |
| `sys` | Interpreter internals (path, arguments, exit) |
| `math` | Mathematical functions |
| `datetime` | Dates and times |
| `random` | Random number generation |
| `json` | JSON encoding/decoding |
| `re` | Regular expressions |
| `collections` | Specialized container types (`defaultdict`, `Counter`, `namedtuple`, `deque`) |
| `itertools` | Efficient looping/combinatorics tools |
| `functools` | Higher-order function tools (`lru_cache`, `reduce`, `partial`, `wraps`) |
| `pathlib` | Object-oriented filesystem paths |
| `logging` | Application logging |
| `argparse` | Command-line argument parsing |
| `unittest` | Built-in testing framework |
| `subprocess` | Running external commands/processes |
| `typing` | Type hints |
| `threading` / `multiprocessing` | Concurrency |

```python
from collections import Counter
from pathlib import Path
from itertools import combinations

print(Counter("mississippi"))
# Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
```

---

## 15. Installing & Managing Third-Party Packages

### `pip` — The Standard Package Installer

```bash
pip install requests                # install a package
pip install requests==2.31.0        # install a specific version
pip install "requests>=2.28,<3.0"   # version range
pip uninstall requests              # remove a package
pip list                            # list installed packages
pip show requests                   # show details of an installed package
pip freeze > requirements.txt       # export exact installed versions
pip install -r requirements.txt     # install from a requirements file
```

### Virtual Environments (Isolating Dependencies)

```bash
python -m venv venv               # create a virtual environment named 'venv'

# Activate it:
source venv/bin/activate          # macOS/Linux
venv\Scripts\activate              # Windows

pip install requests               # installs only inside this isolated environment
deactivate                          # exit the virtual environment
```

> [!tip]
> Always use a **virtual environment** per project. It prevents dependency version conflicts between different projects and keeps your global Python installation clean.

### Where Do Installed Packages Go?

```python
import site
print(site.getsitepackages())
# e.g. ['/path/to/venv/lib/python3.11/site-packages']
```

---

## 16. Creating & Distributing Your Own Package

### Modern Project Structure (using `pyproject.toml`)

```
mypackage_project/
├── pyproject.toml
├── README.md
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── math_utils.py
│       └── string_utils.py
└── tests/
    └── test_math_utils.py
```

### Minimal `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "mypackage"
version = "1.0.0"
description = "A small utility package"
authors = [{name = "Amit", email = "amit@example.com"}]
dependencies = []

[tool.setuptools.packages.find]
where = ["src"]
```

### Installing Locally in "Editable" Mode (for development)

```bash
pip install -e .
```

This links the package into your environment so changes to the source are immediately reflected, without needing to reinstall.

### Building & Publishing to PyPI

```bash
pip install build twine
python -m build                     # creates dist/*.whl and dist/*.tar.gz
twine upload dist/*                 # uploads to PyPI (requires an account)
```

> [!note]
> `setup.py`/`setup.cfg` were the traditional way to define package metadata; modern Python packaging has largely shifted to the declarative `pyproject.toml` format (PEP 621), which is now the recommended standard.

---

## 17. Circular Imports

A **circular import** happens when two (or more) modules import each other, directly or indirectly, creating a dependency loop.

### Example of the Problem

**`a.py`**
```python
import b

def func_a():
    return b.func_b()
```

**`b.py`**
```python
import a          # circular! a imports b, and b imports a

def func_b():
    return a.func_a()
```

```python
import a
# ImportError or AttributeError, depending on which module is imported first
# and how far each module's top-level code got before the circular reference was hit
```

### Why It Happens
When `a.py` is imported, Python starts executing it. It hits `import b`, so it pauses `a` and starts executing `b.py`. `b.py` then hits `import a` — but `a` is only **partially initialized** at this point (still mid-execution), so any names not yet defined in `a` at that point are unavailable, often causing an `AttributeError` or `ImportError`.

### Fixes

**1. Import inside the function (deferred import):**
```python
# b.py
def func_b():
    import a         # deferred until func_b() is actually called, by which point a is fully loaded
    return a.func_a()
```

**2. Restructure code to remove the cycle:**
Move shared logic into a third module that both `a.py` and `b.py` import from, instead of importing each other directly.

```
shared.py    ← common logic used by both
a.py         → imports shared
b.py         → imports shared
```

**3. Import only what's needed, at the top, if there's no true circular dependency (just reorder):**
Sometimes reordering imports or using `import module` (not `from module import name`) resolves partial-initialization issues, since `import module` doesn't require the specific name to exist yet at import time.

> [!tip]
> Circular imports are usually a sign of tightly coupled modules — treat them as a nudge to reconsider your module boundaries and responsibilities, not just something to patch around.

---

## 18. Compiled Files: `__pycache__` and `.pyc`

When a module is imported, CPython compiles its source to **bytecode** and caches it as a `.pyc` file inside a `__pycache__` directory, to speed up future imports (skipping re-compilation if the source hasn't changed).

```
mypackage/
├── __init__.py
├── math_utils.py
└── __pycache__/
    ├── __init__.cpython-311.pyc
    └── math_utils.cpython-311.pyc
```

- The filename encodes the Python version (`cpython-311`) so different interpreter versions don't clash.
- Python automatically detects if the `.py` source has changed (via timestamp/hash) and recompiles as needed — you generally never need to manage these files manually.
- It's standard practice to add `__pycache__/` and `*.pyc` to `.gitignore`.

```gitignore
__pycache__/
*.pyc
*.pyo
```

---

## 19. Common Pitfalls

| Pitfall | Why It's a Problem | Fix |
|---|---|---|
| Naming your own file the same as a standard library module (e.g., `random.py`, `json.py`) | Your file **shadows** the real module, breaking anything that imports the real one | Never name personal scripts after standard library modules |
| Using wildcard imports (`from module import *`) carelessly | Pollutes namespace, causes silent name collisions, hurts readability | Import explicit names, or define `__all__` |
| Relative imports run as a standalone script | Raises `ImportError: attempted relative import with no known parent package` | Run via `python -m package.module`, or use absolute imports |
| Circular imports | `ImportError`/`AttributeError` due to partial initialization | Restructure code, use deferred imports, or extract shared logic |
| Forgetting `__init__.py` when a namespace package isn't intended | Directory not recognized where explicit packaging is expected (though Python 3.3+ often treats it as a namespace package by default) | Add `__init__.py` for regular packages |
| Heavy computation at module top-level | Runs unexpectedly at import time (once), possibly slowing down startup or causing side effects | Wrap in functions, guard with `if __name__ == "__main__":` |
| Modifying `sys.path` haphazardly in scripts | Fragile, hard to reproduce across environments | Use virtual environments + proper packaging instead |

---

## 20. Best Practices

- ✅ One module = one clear responsibility; one package = a cohesive group of related modules.
- ✅ Use `if __name__ == "__main__":` to separate importable logic from script-only behavior.
- ✅ Prefer **absolute imports** for clarity in application code; use relative imports mainly within self-contained libraries/packages.
- ✅ Use `__all__` to explicitly define a module/package's public API.
- ✅ Always use a **virtual environment** per project.
- ✅ Pin dependency versions (`requirements.txt` or `pyproject.toml`) for reproducibility.
- ✅ Avoid circular imports by extracting shared logic into a separate module.
- ✅ Never shadow standard library module names with your own files.
- ✅ Use `pyproject.toml` for new packaging projects (modern standard, replacing `setup.py`).
- ✅ Keep `__init__.py` files lightweight — mainly for re-exporting a clean public API, not heavy logic.

---

## 21. Quick Reference Cheat Sheet

```python
# Import styles
import module
import module as alias
from module import name
from module import name as alias
from module import *          # avoid in production
from . import sibling_module              # relative: same package
from .. import parent_module              # relative: parent package
from package.subpackage import module     # absolute

# Introspection
module.__name__
module.__file__
module.__doc__
dir(module)

# Reloading
import importlib
importlib.reload(module)

# Dynamic import
importlib.import_module("module_name")

# Package essentials
__init__.py       # marks a regular package, controls public API
__all__ = [...]   # controls `from module import *` behavior

# Environment
python -m venv venv
source venv/bin/activate
pip install package_name
pip freeze > requirements.txt

# Running with relative imports
python -m package.module
```

| Concept | Summary |
|---|---|
| **Module** | A single `.py` file of reusable code |
| **Package** | A directory of related modules (with or without `__init__.py`) |
| **Regular package** | Has `__init__.py`; explicit and predictable |
| **Namespace package** | No `__init__.py`; can span multiple directories (PEP 420) |
| **Absolute import** | Full path from the project root |
| **Relative import** | Path relative to current package (`.`, `..`) |
| **`__all__`** | Controls what `import *` exposes |
| **Circular import** | Two modules importing each other, causing partial-init errors |
| **`sys.path`** | Ordered list of directories Python searches for modules |
| **Virtual environment** | Isolated Python environment per project |

---

*Tags:* #python #modules #packages #imports #packaging
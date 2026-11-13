# Python Modules and Packages — Beginner to Advanced

> **Audience:** A learner starting from the basics and progressing to building, testing, and publishing reusable Python packages.  
> **Baseline:** Python 3.10+ for the examples unless a section states otherwise.  
> **Scope:** Python modules, imports, packages, import resolution, project structure, packaging, and distribution.

---

# Part I — Understand the Building Blocks

## 1. Why Modules and Packages Exist

Imagine writing an entire application in one file. As the application grows, that file becomes difficult to read, test, debug, and reuse. Functions and classes also become harder to locate, and unrelated code can accidentally use the same variable or function name.

Python solves this by letting you split code into **modules** and organize modules into **packages**.

A useful mental model is:

| Term | Meaning | Example |
|---|---|---|
| Module | One importable unit of Python code | `billing.py` |
| Package | A way to organize related modules under a package name | `shop/` |
| Subpackage | A package nested inside another package | `shop/delivery/` |
| Distribution package | The installable artifact users install with a tool such as `pip` | `sample-shop-utils` |

**Important distinction:** the word *package* is used in two ways in Python discussions. A Python package is an importable namespace that contains modules. A *distribution package* is what is built and installed. Their names do not have to match: `pip install pillow` provides the import name `PIL`.

## 2. What Is a Module?

A module is an importable unit of Python code. The most common kind is a `.py` file, but modules can also be built into the interpreter, implemented as extension modules, or provided by custom import loaders.

For now, think of a module as a Python file that groups related code.

### 2.1 Create a module

Project layout:

```text
project/
├── calculator.py
└── main.py
```

`calculator.py`:

```python
# calculator.py

TAX_RATE = 0.18


def add(a, b):
    return a + b


def calculate_tax(amount):
    return amount * TAX_RATE
```

`main.py`:

```python
# main.py

import calculator

subtotal = calculator.add(100, 50)
tax = calculator.calculate_tax(subtotal)

print(subtotal)
print(tax)
```

**Line-by-line explanation**

1. `import calculator` tells Python to find and load the module named `calculator`.
2. `calculator.add(...)` calls the `add` function that belongs to that module.
3. `calculator.calculate_tax(...)` accesses another name in the same module.
4. The `calculator.` prefix makes the origin of each function clear and reduces the risk of name collisions.

Run it from the `project/` directory:

```bash
python main.py
```

Expected output:

```text
150
27.0
```

**Why create a module?** The calculation logic can now be reused by another file without copying the functions.

## 3. What Is a Package?

A package organizes related modules under a common name. A regular package usually has an `__init__.py` file.

Example:

```text
project/
├── main.py
└── shop/
    ├── __init__.py
    ├── billing.py
    └── inventory.py
```

`shop/billing.py`:

```python
# shop/billing.py

def create_invoice(order_id):
    return f"Invoice created for order {order_id}"
```

`shop/inventory.py`:

```python
# shop/inventory.py

def get_item_count():
    return 25
```

`main.py`:

```python
# main.py

from shop.billing import create_invoice
from shop.inventory import get_item_count

print(create_invoice("ORD-1001"))
print(get_item_count())
```

**Line-by-line explanation**

1. `shop` is the package name because it is the directory containing the package.
2. `billing` and `inventory` are modules inside the `shop` package.
3. `from shop.billing import create_invoice` loads the `shop.billing` module and binds its `create_invoice` name in the current file.
4. The second import does the same for `get_item_count`.
5. The caller can use those functions directly without writing the full module path each time.

Expected output:

```text
Invoice created for order ORD-1001
25
```

### 3.1 What does `__init__.py` do?

For a **regular package**, `__init__.py` marks the directory as a package and runs when that package is imported for the first time in the current interpreter process.

It may be empty:

```python
# shop/__init__.py
# This file can be empty.
```

It can also expose a convenient public API. That pattern is explained in Section 9.

**Do not confuse these statements:**

- A regular package normally contains `__init__.py`.
- A namespace package can exist without `__init__.py`; see Section 40.
- A directory containing `.py` files is not automatically importable from every working directory. Its parent directory must be discoverable by Python's import system.

## 4. Module Names, File Names, and Import Names

The file name usually supplies the final part of a module's import name:

| File | Typical import name |
|---|---|
| `calculator.py` | `calculator` |
| `shop/billing.py` | `shop.billing` |
| `shop/delivery/tracking.py` | `shop.delivery.tracking` |

The dot represents a package boundary. It is not a filesystem separator in Python import syntax.

A normal import name must be a valid Python identifier at each dotted component. Avoid file names that conflict with standard-library or third-party module names, such as `json.py`, `random.py`, `typing.py`, or `requests.py`.

---

# Part II — Learn Import Syntax

## 5. The Main Import Forms

Python has several import forms. They all load modules, but they bind different names in the current namespace.

### 5.1 `import module`

```python
import math

result = math.sqrt(25)
print(result)
```

Line by line:

1. `import math` loads the `math` module if it has not already been loaded.
2. It binds the name `math` in the current namespace.
3. `math.sqrt(25)` looks up `sqrt` inside the module and calls it.

Expected output:

```text
5.0
```

This is often the clearest style because the module that provides the function is visible at the call site.

### 5.2 `from module import name`

```python
from math import sqrt

result = sqrt(25)
print(result)
```

Line by line:

1. Python imports `math`.
2. It obtains the `sqrt` attribute from that module.
3. It binds the name `sqrt` directly in the current namespace.
4. You call `sqrt(...)` without the `math.` prefix.

Use this when the imported name is clear and does not create ambiguity.

**Important:** `from module import name` binds the object that the module exposes at that time. It does not create a live alias that automatically follows later reassignment of `module.name`.

### 5.3 Import a submodule

```python
import shop.billing

message = shop.billing.create_invoice("ORD-20")
print(message)
```

With the statement `import shop.billing`, Python binds the top-level name `shop` in the current namespace. The submodule is reachable as `shop.billing`.

You can bind the submodule to a shorter local name with an alias:

```python
import shop.billing as billing

print(billing.create_invoice("ORD-20"))
```

Here, `billing` refers to the `shop.billing` module.

### 5.4 `from package import submodule`

```python
from shop import billing

print(billing.create_invoice("ORD-20"))
```

This form can import a submodule even when `billing` was not already defined as an attribute in `shop`. Python checks the package for the attribute and, when appropriate, tries to import the submodule.

### 5.5 Aliases with `as`

```python
import statistics as stats
from pathlib import Path as FilePath

print(stats.mean([10, 20, 30]))
print(FilePath("reports") / "summary.txt")
```

An alias changes only the local name used by this file. It does not rename the installed module or package.

Common standard-library aliases include `import numpy as np` and `import pandas as pd` in projects that use those third-party libraries. Use conventional aliases consistently rather than inventing many abbreviations.

### 5.6 Wildcard imports: `from module import *`

```python
from math import *

print(sqrt(25))
```

This imports names into the current namespace. It can make it unclear where a name came from, and it can overwrite an existing name.

**Recommendation:** avoid wildcard imports in normal application code. Prefer explicit imports:

```python
from math import pi, sqrt

print(sqrt(25))
print(pi)
```

Wildcard imports are occasionally useful in interactive sessions, but they are rarely a good choice in maintainable modules.

## 6. What Exactly Gets Bound by Each Import?

This distinction is important when reading larger projects or debugging imports.

| Statement | Name(s) bound in the current file |
|---|---|
| `import math` | `math` |
| `import shop.billing` | `shop` |
| `import shop.billing as billing` | `billing` |
| `from shop import billing` | `billing` |
| `from shop.billing import create_invoice` | `create_invoice` |

The imported module object is stored in `sys.modules`; the statement also determines which reference is placed in the current namespace.

### 6.1 Why `from ... import ...` can become stale after reload

```python
import importlib
import calculator
from calculator import add

old_add = add

importlib.reload(calculator)

# calculator.add refers to the function now present in the reloaded module.
# The local name `add` still refers to the previously imported function.
print(add is calculator.add)
print(old_add is add)
```

Expected output in this simple example:

```text
False
True
```

`reload()` re-executes code in the module's existing module object. It does not update references that other modules previously copied with `from calculator import add`. For live development, prefer accessing functions through the module (`calculator.add`) when reload behavior matters. Reloading has additional caveats; see Section 13.

---

# Part III — Public APIs and Package Structure

## 7. What Is `__all__`?

`__all__` is a module-level list of names that should be imported by `from module import *`.

Example `billing.py`:

```python
# billing.py

__all__ = ["create_invoice", "calculate_tax"]


def create_invoice():
    return "Invoice created"


def calculate_tax(amount):
    return amount * 0.18


def _internal_helper():
    return "Internal implementation detail"


def debug_state():
    return "Debug information"
```

Caller:

```python
from billing import *

print(create_invoice())
print(calculate_tax(100))
```

Only `create_invoice` and `calculate_tax` are imported by the wildcard statement because they are listed in `__all__`.

**Important distinctions:**

- `__all__` affects wildcard imports only.
- It does not prevent an explicit import such as `from billing import debug_state`.
- If `__all__` is absent, wildcard import normally excludes names beginning with `_`.
- `__all__` is not an access-control or security mechanism.

## 8. The Role of `__init__.py`

A package's `__init__.py` runs when the package is imported. It can expose a clean public API so callers do not need to know the internal file layout.

Example layout:

```text
shop/
├── __init__.py
├── billing.py
└── inventory.py
```

`shop/__init__.py`:

```python
from .billing import create_invoice, calculate_tax
from .inventory import get_item_count

__all__ = [
    "create_invoice",
    "calculate_tax",
    "get_item_count",
]

__version__ = "1.0.0"
```

Line by line:

1. `from .billing ...` uses a relative import to import names from the sibling `billing.py` module.
2. `from .inventory ...` imports the public inventory function.
3. `__all__` identifies the package-level names intended for wildcard import.
4. `__version__` is an ordinary module-level variable. The name is a convention, not a Python keyword.

Callers can now write:

```python
from shop import create_invoice

print(create_invoice())
```

instead of importing from the implementation module directly.

### 8.1 Keep package initialization lightweight

When a caller imports `shop.billing`, Python initializes the parent package first. That means `shop/__init__.py` runs before `shop/billing.py`.

Avoid putting these in `__init__.py` unless genuinely necessary:

- Network calls or database connections.
- Expensive computations.
- Large optional dependencies imported only for a rarely used feature.
- Application startup logic or long-running work.

Re-exporting a small set of stable public names is useful. Importing every submodule eagerly can make imports slower and can create circular dependencies.

## 9. Naming and Organizing a Public API

A good package separates its **public API** from its internal implementation.

Example:

```text
shop/
├── __init__.py
├── billing.py
├── inventory.py
└── _internal/
    ├── __init__.py
    └── tax_rules.py
```

Conventions:

- Names without a leading underscore are normally public unless documentation says otherwise.
- A leading underscore (`_internal_helper`, `_internal/`) signals that a name is internal by convention.
- A leading underscore does not make a name inaccessible.
- Double-leading-underscore name mangling is primarily a class feature; it is not a module privacy mechanism.

Avoid exposing every implementation detail from `__init__.py`. Export only the functions, classes, and constants callers should rely on.

---

# Part IV — Absolute and Relative Imports

## 10. Absolute Imports

An absolute import starts from a top-level importable name.

Given:

```text
project/
├── main.py
└── shop/
    ├── __init__.py
    ├── billing.py
    └── inventory.py
```

Inside `shop/billing.py`:

```python
from shop.inventory import get_item_count
```

The path begins at the top-level package `shop`. Python must be able to find the directory containing `shop` through the import system, commonly via the project root or an installed distribution.

**Benefits:**

- Makes the origin of the import explicit.
- Is easy to search and understand in a large codebase.
- Avoids ambiguity when many packages contain files with the same name.

## 11. Relative Imports

A relative import starts from the current module's package.

```python
from .inventory import get_item_count
```

This means: import `inventory` from the same package as the current module.

The dots indicate package levels:

| Syntax | Meaning |
|---|---|
| `from . import billing` | Import `billing` from the current package |
| `from .inventory import get_item_count` | Import from a sibling module in the current package |
| `from ..shared import format_price` | Go to the parent package, then import `shared` |
| `from ...common import helper` | Go up two package levels, then import `common` |

Relative import syntax must use `from ... import ...`. `import .inventory` is invalid syntax.

A relative import cannot go above the top-level package.

### 11.1 Example in a nested package

```text
shop/
├── __init__.py
├── billing.py
└── delivery/
    ├── __init__.py
    └── tracking.py
```

Inside `shop/delivery/tracking.py`:

```python
from ..billing import create_invoice


def get_tracking_message(order_id):
    invoice_message = create_invoice(order_id)
    return f"Tracking started. {invoice_message}"
```

Explanation:

1. `tracking.py` belongs to the `shop.delivery` package.
2. One dot would refer to `shop.delivery`.
3. Two dots (`..`) move to the parent package, `shop`.
4. Python then imports `create_invoice` from `shop.billing`.

## 12. The Relative-Import Execution Trap

A module that contains a relative import generally needs to be executed with package context.

Suppose `shop/billing.py` contains:

```python
from .inventory import get_item_count


def print_stock():
    print(get_item_count())
```

Running it directly by file path:

```bash
python shop/billing.py
```

usually fails with an error similar to:

```text
ImportError: attempted relative import with no known parent package
```

Why? Python is executing the file as a standalone script, not as `shop.billing` within the `shop` package.

Run it as a module from the directory above `shop`:

```bash
python -m shop.billing
```

This gives the module package context so its relative import can be resolved.

**Important:** when a module is run with `python -m shop.billing`, its code executes with `__name__ == "__main__"`, even though its `__spec__.name` identifies it as `shop.billing`. If the same module is later imported as `shop.billing`, it can be executed a second time under a second module name. For an application entry point, prefer a package-level `__main__.py` (Section 17) or a console script declared in the project metadata (Section 27).

### 12.1 Absolute or relative — which should you use?

Both are valid. A practical rule is:

- Use absolute imports when they make the package path clearer.
- Use explicit relative imports for closely related modules inside a package when they improve readability.
- Avoid implicit relative imports. Python 3 does not support the old Python 2 behavior where a bare import could silently mean a sibling module.
- Follow the conventions already used consistently by the project.

PEP 8 recommends absolute imports but also recognizes explicit relative imports as an acceptable alternative.

---

# Part V — How Python Finds and Loads Modules

## 13. `sys.modules`: The Module Cache

Python keeps track of modules it has loaded in a dictionary named `sys.modules`.

```python
import sys
import math

print("math" in sys.modules)
print(sys.modules["math"] is math)
```

Expected output:

```text
True
True
```

Line by line:

1. `import sys` loads the module that exposes interpreter-level information.
2. `import math` loads (or reuses) the math module.
3. `"math" in sys.modules` checks whether the import name has an entry in the cache.
4. `sys.modules["math"]` retrieves the module object associated with that name.
5. `is` checks object identity, not value equality.

### 13.1 Why caching matters

On a normal successful import, Python does not execute the same module code again for every later import under the same module name. It reuses the module object in `sys.modules`.

A simplified model is:

1. Check `sys.modules` for the requested module name.
2. If it is already loaded, reuse that module object.
3. Otherwise, ask the import system to find a module specification and a loader.
4. Create the module object and place it in `sys.modules` before executing its code.
5. The loader executes the module code.
6. The import statement binds names in the importing module.

The insertion before execution is important for circular imports: another module can find a partially initialized module while it is still being executed.

**Precision:** caching is by import name, not by source-file path. The same file imported under two different names can be executed twice and produce two different module objects. See Section 16.

## 14. Inspecting Modules: `dir()`, `hasattr()`, and `__file__`

### 14.1 `dir(module)`

```python
import math

names = dir(math)
print("sqrt" in names)
print("factorial" in names)
```

`dir()` returns a useful list of names visible on the object. It is helpful for exploration, but it is not a guarantee that a name is a stable public API.

### 14.2 `hasattr(module, name)`

```python
import math

if hasattr(math, "factorial"):
    print(math.factorial(5))
```

`hasattr()` is the more direct check when the question is simply whether an attribute can be accessed.

**Caveat:** attribute access can execute code. For modules with a custom `__getattr__`, `hasattr()` can trigger lazy loading or other behavior.

### 14.3 `__file__`

```python
import json

print(json.__file__)
```

For many file-backed modules, `__file__` shows the path from which the module was loaded. Not all modules have a meaningful `__file__`; built-in modules and some namespace packages do not.

Use it when debugging accidental module shadowing (Section 18).

## 15. Checking Whether an Optional Dependency Is Available

The usual approach is to attempt the import and handle failure:

```python
try:
    import numpy as np
except ImportError:
    np = None

if np is None:
    print("NumPy is not installed; using the fallback implementation.")
else:
    print(np.array([1, 2, 3]))
```

Line by line:

1. `try` starts an operation that may fail.
2. Python attempts to import `numpy` and bind it as `np`.
3. If the dependency is unavailable, `ImportError` is caught and `np` is set to `None`.
4. The later `if` chooses the fallback or NumPy implementation.

`ModuleNotFoundError` is a subclass of `ImportError`. Catching `ImportError` is common for optional dependencies, but be careful: it can also catch an import failure *inside* an installed dependency. If you need to distinguish “package not installed” from “package found but one of its own imports failed,” catch `ModuleNotFoundError` and inspect `exc.name`.

Example of a more specific check:

```python
try:
    import numpy as np
except ModuleNotFoundError as exc:
    if exc.name != "numpy":
        raise
    np = None
```

Here, a missing dependency imported internally by NumPy is not silently treated as though NumPy itself were absent.

### 15.1 `importlib.util.find_spec()`

You can ask the import system whether it can find a module without importing that module itself:

```python
from importlib.util import find_spec

spec = find_spec("json")

if spec is not None:
    print("A module specification was found.")
else:
    print("The module could not be found.")
```

`find_spec()` is useful for discovery, but it is not a replacement for handling import errors. A module may be discoverable and still fail when executed. Also, checking a dotted name such as `package.submodule` may import the parent package as part of finding the submodule.

## 16. `sys.path`: Where Python Searches

`sys.path` is the list of locations used by the standard path-based import finder.

```python
import sys

for entry in sys.path:
    print(entry)
```

The exact list depends on how Python was launched, the environment, the installation, and configuration. A simplified view of common entries is:

1. The script directory when executing a script by path, or the current directory for some launch modes such as `-m` and the interactive shell.
2. Directories from the `PYTHONPATH` environment variable, when set.
3. Standard-library locations.
4. Site-package directories containing installed third-party distributions.

**Do not treat this as a complete universal ordering.** Python first consults `sys.modules` and the finders in `sys.meta_path`. The path-based finder uses `sys.path` for top-level imports and a package's `__path__` for its submodules. Built-in and frozen import finders are also part of the import system.

### 16.1 What is `sys.path[0]`?

The first entry varies by launch mode:

- `python script.py`: normally the directory containing `script.py`.
- `python -m package.module`: normally the current working directory.
- Interactive shell: commonly an empty string, representing the current working directory.

Python 3.11+ provides `-P` and `PYTHONSAFEPATH=1` to avoid automatically prepending a potentially unsafe path at startup. These options can help prevent accidental shadowing, but they do not replace good project structure.

### 16.2 Should you modify `sys.path` manually?

You may see code like:

```python
import sys
sys.path.append("/some/project/folder")
```

This can be useful for short experiments or specialized tooling, but it is usually not the right fix for application structure. It creates environment-dependent behavior and can make a program import different code on different machines.

For a project you develop locally, use a virtual environment and an editable install instead:

```bash
python -m pip install -e .
```

## 17. `__name__ == "__main__"` and `__main__.py`

### 17.1 The main guard

Every module has a `__name__` attribute.

- When a file is executed directly as `python billing.py`, its `__name__` is exactly `"__main__"`.
- When it is imported as `shop.billing`, its `__name__` is `"shop.billing"`.

This is why the following pattern is so common:

```python
# billing.py

def create_invoice():
    return "Invoice created"


def main():
    print(create_invoice())


if __name__ == "__main__":
    main()
```

When executed directly, the condition is true and `main()` runs. When imported as `shop.billing`, the condition is false, so command-line behavior does not run as an import side effect.

### 17.2 Run a package with `python -m package`

To make a package runnable as a whole, add `__main__.py`:

```text
shop/
├── __init__.py
├── __main__.py
└── billing.py
```

`shop/__main__.py`:

```python
from .billing import create_invoice


def main():
    print(create_invoice())


if __name__ == "__main__":
    main()
```

Run it from the directory containing `shop/`:

```bash
python -m shop
```

Python imports the `shop` package and executes `shop/__main__.py` as the program entry point. This is a standard way to support `python -m shop`.

### 17.3 Useful module attributes

```python
print(__name__)
print(__package__)
print(__file__)
print(__spec__)
print(__doc__)
```

| Attribute | Purpose |
|---|---|
| `__name__` | Module's runtime name; `"__main__"` when executed as the main program |
| `__package__` | Package context used for resolving relative imports |
| `__file__` | File path for many file-backed modules; not defined for every module |
| `__spec__` | The module's import specification (`ModuleSpec`) |
| `__doc__` | Module docstring, or `None` when there is no docstring |

For a package, `__path__` describes where Python looks for its submodules. A normal module does not have a package `__path__`.

**Remember:** `python -m shop.billing` runs `shop.billing` as `__main__`; `python -m shop` runs `shop.__main__`. Prefer the latter for a package-level command so the main application logic has one clear entry point.

## 18. How the Import System Works Internally

You do not need to implement Python's import machinery to use it, but understanding its stages makes errors easier to diagnose.

### 18.1 Finders, loaders, and `ModuleSpec`

- A **finder** determines whether it can locate a module and returns a `ModuleSpec`.
- A **loader** knows how to create or execute the module.
- A **ModuleSpec** carries information about how the module should be loaded, including its name, loader, and origin when applicable.

The main finder sequence is represented by `sys.meta_path`. The path-based finder is one of these finders; it consults `sys.path` or a package's `__path__`.

### 18.2 A simplified import sequence

1. Python checks `sys.modules`.
2. If the module is not already present, Python asks the finders in `sys.meta_path` for a specification.
3. A finder returns a `ModuleSpec` if it can handle the name.
4. Python creates the module object and inserts it in `sys.modules` before executing it.
5. The loader executes the module's code.
6. Import machinery makes a successfully imported submodule available as an attribute of its parent package.

This explains why circular imports can observe a module before all of its top-level statements have run.

### 18.3 Inspect a module specification

```python
import importlib.util

spec = importlib.util.find_spec("json")

if spec is not None:
    print(spec.name)
    print(spec.origin)
    print(spec.loader)
```

`origin` may be a path, a special marker, or `None`, depending on the module type. Do not assume all modules come from `.py` files.

## 19. Bytecode Cache: `__pycache__` and `.pyc`

When Python imports source code from a `.py` file, it may compile that source into bytecode and store the bytecode cache under `__pycache__/`.

Example:

```text
shop/
├── __pycache__/
│   └── billing.cpython-313.pyc
└── billing.py
```

Key points:

- `.pyc` files are an import-time optimization that can avoid recompiling unchanged source.
- The cache file name is tied to the Python implementation/version tag.
- Python validates the cache using timestamp-based or hash-based rules.
- You can delete `__pycache__`; Python can regenerate the cache when needed.
- `python -B` or `PYTHONDONTWRITEBYTECODE=1` disables writing bytecode cache files.
- The top-level script executed by file path is not cached in the same way as an imported module.

Do not edit `.pyc` files manually. They are generated artifacts, not source files.

## 20. Reloading a Module with `importlib.reload()`

```python
import importlib
import math

reloaded_math = importlib.reload(math)
print(reloaded_math is math)
```

The returned module object is normally the same object that was passed in.

Important caveats:

- `reload()` re-executes the module code in the existing module object's namespace.
- Names removed from the new source may remain in the module dictionary if the new execution does not overwrite them.
- References previously copied using `from module import name` are not automatically rebound.
- Existing instances of classes defined in the module continue to use their old class objects.
- Reloading is not thread-safe and is not a normal production deployment mechanism.

Use reload mainly for interactive experimentation. Restarting the process is usually safer when code changes affect many references.

---

# Part VI — Import Errors and Troubleshooting

## 21. Circular Imports

A circular import occurs when modules depend on one another during initialization.

Example dependency cycle:

```text
user.py  ---> imports order.py
order.py ---> imports user.py
```

### 21.1 A minimal demonstration

Create these files in a package called `shop`.

`shop/user.py`:

```python
from .order import Order


class User:
    def __init__(self, user_id):
        self.user_id = user_id
```

`shop/order.py`:

```python
from .user import User


class Order:
    pass
```

When `shop.user` is imported:

1. Python begins executing `user.py`.
2. Before defining `User`, it imports `shop.order`.
3. `order.py` then tries to import `User` from `shop.user`.
4. `shop.user` is already in `sys.modules`, but it is only partially initialized.
5. `User` has not been defined yet, so the `from` import fails.

The error commonly mentions a *partially initialized module*. This is not a deadlock; it is an initialization-order problem.

### 21.2 Solution A: Move the import inside the function that needs it

```python
# shop/order.py

class Order:
    def assign_to_user(self, user_id):
        from .user import User
        return User(user_id)
```

The import is delayed until `assign_to_user()` is called. It works if `shop.user` has completed initialization by then.

**Caution:** a local import can be a practical fix, but it does not automatically make a poor module dependency design good. Use it only when delaying the dependency is appropriate.

### 21.3 Solution B: Move shared definitions to a third module

```text
shop/
├── __init__.py
├── user.py
├── order.py
└── models.py
```

`shop/models.py`:

```python
class User:
    def __init__(self, user_id):
        self.user_id = user_id


class Order:
    def __init__(self, order_id):
        self.order_id = order_id
```

Both `user.py` and `order.py` can import the shared model definitions without importing each other. This is often the cleaner design.

### 21.4 Solution C: Import only for type checking

If the dependency exists only because of type annotations, use `TYPE_CHECKING`:

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from .user import User


class Order:
    def assign_to_user(self, user: User) -> None:
        ...
```

Line by line:

1. `from __future__ import annotations` postpones evaluation of annotations.
2. `TYPE_CHECKING` is `False` at runtime and treated as true by static type checkers.
3. The `User` import is available to the type checker but does not execute at runtime.
4. The annotation documents the expected type without introducing a runtime import cycle.

Use this only when the imported symbol is genuinely needed for typing, not when runtime code needs the class.

## 22. Module Shadowing and Naming Conflicts

A local file can accidentally shadow a standard-library or third-party module.

Problematic project layout:

```text
project/
├── random.py
└── main.py
```

`main.py`:

```python
import random

print(random.randint(1, 10))
```

Python may import the local `random.py` rather than the standard-library `random` module. If that local file does not define `randint`, the code fails.

### 22.1 Diagnose the problem

```python
import random

print(random.__file__)
```

If the path points to your project rather than the expected standard-library location, you have likely shadowed the module.

### 22.2 Fix it

1. Rename the local file, for example to `random_helpers.py`.
2. Remove stale `__pycache__` files for the old name if needed.
3. Restart the Python process so the wrong module is no longer cached.
4. Re-run the import and check its path again.

**Important distinction:** built-in and frozen modules are found by earlier import finders and generally cannot be shadowed in the same way. Many pure-Python standard-library modules, such as `random` and `json`, can be shadowed by an earlier path entry.

## 23. Common Import Exceptions

| Exception | Meaning | Typical cause |
|---|---|---|
| `ModuleNotFoundError` | A requested module could not be found | Package is not installed or import path is wrong |
| `ImportError` | Import operation failed | Requested name is unavailable, or module initialization failed |
| `AttributeError` | Object has no requested attribute | Wrong module/object or partially initialized module accessed through an attribute |
| `SyntaxError` | Source code could not be parsed | Invalid Python syntax in an imported file |
| `OSError` | An operating-system operation failed | A file or shared-library loading operation failed |

`ModuleNotFoundError` is a subclass of `ImportError`. Read the complete traceback: the last line alone may not identify the original cause.

---
# Part VII — Build an Installable Python Package

## 24. Import Package vs. Distribution Package

These terms are related but different.

| Concept | Example | Used by |
|---|---|---|
| Distribution name | `sample-shop-utils` | `pip install sample-shop-utils` |
| Import package name | `shop` | `import shop` |
| Module name | `shop.billing` | `from shop.billing import create_invoice` |

The distribution name is declared in project metadata. The import name comes from the Python package/module structure. They may be different.

## 25. Why Use `pyproject.toml`?

`pyproject.toml` is the modern project configuration file used by Python packaging tools and many development tools.

It commonly contains three important areas:

| Table | Responsibility |
|---|---|
| `[build-system]` | Declares the build backend and its build-time requirements |
| `[project]` | Declares project metadata and dependencies |
| `[tool.*]` | Stores configuration specific to tools and build backends |

The `[dependency-groups]` table can also be used for development dependencies by supporting tools. It is distinct from installable extras under `[project.optional-dependencies]`.

For new projects, prefer a `pyproject.toml`-based build configuration instead of relying on direct `setup.py` commands. `setup.py` can still exist for advanced backend configuration, but commands such as `python setup.py install` and `python setup.py sdist` should be replaced by modern frontend commands such as `python -m pip install .` and `python -m build`.

## 26. A Complete Example Project

We will create a small package named `sample-shop-utils` that users install but import as `shop`.

### 26.1 Project structure

```text
sample_shop_project/
├── pyproject.toml
├── README.md
├── LICENSE
├── src/
│   └── shop/
│       ├── __init__.py
│       ├── billing.py
│       ├── cli.py
│       └── data/
│           └── tax_rates.json
└── tests/
    └── test_billing.py
```

Why use the `src/` layout?

- The import package is kept under `src/` rather than directly beside `pyproject.toml`.
- Running tests from the project root does not accidentally import the source package merely because the current directory is on the import path.
- You normally install the project (often in editable mode) before importing it.
- This helps expose packaging mistakes earlier.

A flat layout can be valid too. The `src/` layout is a useful choice for reusable packages because it makes the distinction between the repository root and importable source code explicit.

### 26.2 Create `src/shop/billing.py`

```python
# src/shop/billing.py

def calculate_tax(amount: float, rate: float = 0.18) -> float:
    if amount < 0:
        raise ValueError("amount must not be negative")
    if rate < 0:
        raise ValueError("rate must not be negative")
    return amount * rate


def create_invoice(order_id: str, amount: float) -> dict:
    tax = calculate_tax(amount)
    return {
        "order_id": order_id,
        "subtotal": amount,
        "tax": tax,
        "total": amount + tax,
    }
```

Explanation:

1. The function annotations document the expected values and return type.
2. The checks reject negative amounts and tax rates.
3. `calculate_tax()` keeps tax logic in one reusable function.
4. `create_invoice()` reuses that function and returns a dictionary representing the invoice.

### 26.3 Create `src/shop/__init__.py`

```python
# src/shop/__init__.py

from .billing import calculate_tax, create_invoice

__all__ = ["calculate_tax", "create_invoice"]
__version__ = "1.0.0"
```

This makes the two functions available from the package root:

```python
from shop import create_invoice

print(create_invoice("ORD-1001", 100))
```

### 26.4 Create a command-line function

`src/shop/cli.py`:

```python
from .billing import create_invoice


def main() -> None:
    invoice = create_invoice("ORD-1001", 100.0)
    print(invoice)


if __name__ == "__main__":
    main()
```

The `main()` function is the command-line entry point. It returns `None` because its purpose is to print the result rather than return a shell status code.

### 26.5 Add a package data file

`src/shop/data/tax_rates.json`:

```json
{
  "standard": 0.18,
  "reduced": 0.05
}
```

A non-Python resource such as JSON, a template, or a certificate must be included in the built distribution if the package needs it at runtime. Section 32 shows how to configure and read package data.

### 26.6 Add a test

`tests/test_billing.py`:

```python
import pytest

from shop import calculate_tax, create_invoice


def test_calculate_tax():
    assert calculate_tax(100) == pytest.approx(18)


def test_create_invoice():
    invoice = create_invoice("ORD-1001", 100)
    assert invoice["total"] == pytest.approx(118)


def test_negative_amount_is_rejected():
    with pytest.raises(ValueError):
        calculate_tax(-1)
```

This test file demonstrates three behaviors: the standard calculation, the invoice total, and validation of invalid input.

## 27. Configure `pyproject.toml`

Create this file at the project root:

```toml
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "sample-shop-utils"
version = "1.0.0"
description = "A small example package for learning Python packaging"
readme = "README.md"
requires-python = ">=3.10"
license = "MIT"
license-files = ["LICENSE"]
authors = [
  { name = "Example Author" }
]

[project.optional-dependencies]
dev = [
  "pytest>=8",
  "build>=1",
  "twine>=5"
]

[project.scripts]
shop-cli = "shop.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-data]
shop = ["data/*.json"]
```

### 27.1 Explain each table

**`[build-system]`**

- `requires` lists the packages needed to run the build backend.
- `build-backend` names the backend that creates the distribution artifacts.
- `setuptools.build_meta` is the setuptools backend.

**`[project]`**

- `name` is the distribution name used by package installers.
- `version` is the release version.
- `description` is a short summary.
- `readme` identifies the README used as project metadata.
- `requires-python` communicates supported Python versions.
- `license` is an SPDX license expression. `license-files` identifies license files to include.
- `authors` contains project author metadata.

**`[project.optional-dependencies]`**

- Defines optional installation extras.
- `dev` is an extra containing tools needed by contributors.
- A user can request it with `python -m pip install "sample-shop-utils[dev]"`.

**`[project.scripts]`**

- `shop-cli` is the command users run after installation.
- `shop.cli:main` means: import `shop.cli`, then call its `main` function.
- The installer creates the command wrapper in the environment's scripts directory.

**`[tool.setuptools.packages.find]`**

- Tells setuptools to search for packages under `src/`.
- It discovers the `shop` package and its subpackages.

**`[tool.setuptools.package-data]`**

- Includes JSON files inside the `shop/data/` directory in the distribution.
- Without an appropriate package-data configuration, non-Python files may be missing from the wheel.

The setuptools version requirement is deliberately `>=77` because this example uses the modern SPDX string form for `project.license` and `license-files` support. Check the backend documentation if you change build backends.

### 27.2 `README.md` and `LICENSE`

Add a short README describing the package. Add the actual license text that matches the `license = "MIT"` declaration. Do not declare a license you do not have permission to use.

---

# Part VIII — Virtual Environments, Install, Test, and Build

## 28. Create a Virtual Environment

A virtual environment isolates project dependencies from other Python projects.

From the project root:

```bash
python -m venv .venv
```

Activate it on macOS/Linux:

```bash
source .venv/bin/activate
```

Activate it on Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Upgrade pip in the active environment:

```bash
python -m pip install --upgrade pip
```

Use `python -m pip` rather than an ambiguous `pip` command when you want to make sure pip belongs to the Python interpreter you are running.

## 29. Install the Project in Editable Mode

Install the package and its development tools:

```bash
python -m pip install -e ".[dev]"
```

What this does:

- `-e` requests an editable (development) installation.
- `.` means the current project directory.
- `[dev]` requests the optional `dev` extra from `pyproject.toml`.

With an editable installation, changes to Python source files are generally available without reinstalling. Changes to project metadata, entry points, or build configuration may require reinstalling.

**Important:** editable installs can behave differently from regular installs depending on the backend. They are for development; they are not proof that a built wheel contains every required file. Test a regular wheel installation before release.

Verify the import:

```bash
python -c "import shop; print(shop.__version__)"
```

Run the command-line entry point:

```bash
shop-cli
```

Run the tests:

```bash
python -m pytest
```

## 30. Build Source and Wheel Distributions

Install the `build` package (included in the `dev` extra above), then run:

```bash
python -m build
```

By default, this creates a `dist/` directory with two artifacts:

| Artifact | Meaning |
|---|---|
| `.tar.gz` source distribution (sdist) | Contains source and metadata; the installer/build frontend builds it during installation |
| `.whl` wheel | A built distribution that can be installed directly |

A wheel file name follows a pattern like:

```text
sample_shop_utils-1.0.0-py3-none-any.whl
```

The tags mean:

- `py3`: compatible with the Python 3 interpreter family indicated by the tag.
- `none`: no interpreter-specific ABI requirement is declared.
- `any`: not tied to a specific operating system or CPU platform.

A package containing compiled extensions may need multiple platform- and interpreter-specific wheels.

## 31. Inspect the Built Artifacts

Use the actual artifact names created in your `dist/` folder:

```bash
python -m zipfile -l dist/sample_shop_utils-1.0.0-py3-none-any.whl
```

List the source distribution:

```bash
tar tf dist/sample_shop_utils-1.0.0.tar.gz
```

Validate package metadata and README rendering:

```bash
python -m twine check dist/*
```

Look for:

- Python package files.
- Required package data such as JSON or templates.
- Project metadata and license files.
- Unexpected files such as local virtual environments, secrets, or temporary artifacts.

## 32. Include and Read Package Data Correctly

Do not assume a package's data file is available through a path relative to the current working directory. The program may be launched from anywhere, or the package may be installed in a way that does not map directly to a normal source directory.

Use `importlib.resources`:

```python
import json
from importlib.resources import files


def load_tax_rates():
    resource = files("shop").joinpath("data", "tax_rates.json")
    text = resource.read_text(encoding="utf-8")
    return json.loads(text)


rates = load_tax_rates()
print(rates["standard"])
```

Line by line:

1. `files("shop")` returns a traversable representation of the package's resources.
2. `joinpath(...)` locates the JSON resource within the package.
3. `read_text(encoding="utf-8")` reads its contents as text.
4. `json.loads(...)` parses the text into a Python dictionary.
5. The final line reads the standard tax rate.

This approach is more portable than assuming the resource is a regular file next to `__file__`. If a third-party API requires a real filesystem path, use `importlib.resources.as_file()` as a context manager; the resource may be extracted to a temporary location and that path is only guaranteed for the context's lifetime.

## 33. Test the Wheel in a Clean Environment

The clean-wheel test checks what an end user installs, rather than what happens to work only inside your repository.

1. Build the wheel using Section 30.
2. Create a separate virtual environment.
3. Install the wheel by its actual file name.
4. Run an import and a smoke test outside the project root.

macOS/Linux example:

```bash
python -m venv .venv-wheel-test
source .venv-wheel-test/bin/activate
python -m pip install dist/sample_shop_utils-1.0.0-py3-none-any.whl
cd /tmp
python -c "from shop import create_invoice; print(create_invoice('ORD-1', 100))"
```

Windows PowerShell: activate `.venv-wheel-test\Scripts\Activate.ps1`, install the wheel using its exact path, then run the import check from a directory outside the repository.

The test should confirm that:

- `shop` imports successfully.
- The public API works.
- The console command is installed if you test it.
- Package data can be read after installation.

## 34. Source Distribution vs. Wheel

| Comparison | Source distribution (sdist) | Wheel |
|---|---|---|
| Main purpose | Distribute source for building | Distribute a built installable artifact |
| Installation | Usually requires a build step | Normally unpacked/installed directly |
| Contents | Source files and build metadata | Files needed for the installed distribution |
| Portability | Depends on build requirements and source compatibility | Depends on wheel tags |

For a pure-Python package, one generic wheel is often sufficient. For packages with compiled extensions, build wheels for supported platforms and interpreter/ABI combinations. Publishing both an sdist and suitable wheels is common practice.

---
# Part IX — Dependencies and Publishing

## 35. Declare Dependencies

Runtime dependencies are declared in `[project].dependencies`:

```toml
[project]
dependencies = [
  "requests>=2.28,<3",
  "pydantic>=2.0"
]
```

Optional capabilities can be declared as extras:

```toml
[project.optional-dependencies]
postgres = ["psycopg[binary]>=3"]
dev = ["pytest>=8", "build>=1", "twine>=5"]
```

Install an extra with:

```bash
python -m pip install "sample-shop-utils[postgres]"
```

### 35.1 Understand version specifiers

| Specifier | Meaning |
|---|---|
| `requests==2.32.3` | Exactly version 2.32.3 |
| `requests>=2.28` | Version 2.28 or newer |
| `requests>=2.28,<3` | At least 2.28 but below 3 |
| `requests~=2.28` | Compatible release range `>=2.28, ==2.*` |
| `tomli>=2; python_version < "3.11"` | Install only when the environment marker matches |

Use version constraints intentionally. Overly tight pins in a reusable library can prevent it from working alongside other libraries. An application has different needs: reproducible deployment usually calls for a lock file or fully resolved environment snapshot.

`pip freeze` can capture the installed environment, but it includes every installed package and is not automatically a carefully designed dependency specification. Tools such as `uv`, `pip-tools`, Poetry, and other lockfile workflows can help applications record a reproducible resolution.

## 36. Distribution Metadata vs. Import Metadata

To read an installed distribution's version, use `importlib.metadata`:

```python
from importlib.metadata import version, PackageNotFoundError

try:
    installed_version = version("sample-shop-utils")
except PackageNotFoundError:
    print("The distribution is not installed in this environment.")
else:
    print(installed_version)
```

The argument is the **distribution name**, not necessarily the import package name. For example, the installed distribution might be `sample-shop-utils` while the code imports `shop`.

## 37. Publish to TestPyPI First

TestPyPI is a separate package index used to rehearse a release. It has separate accounts and package records from the production PyPI index.

Typical workflow:

1. Build and inspect the artifacts.
2. Create a TestPyPI account and configure an API token.
3. Upload the artifacts.
4. Install the package in a clean environment from TestPyPI and verify it.

Upload with Twine:

```bash
python -m twine upload --repository testpypi dist/*
```

When prompted, use `__token__` as the username and the TestPyPI token as the password.

Install from TestPyPI:

```bash
python -m pip install \
  --index-url https://test.pypi.org/simple/ \
  --extra-index-url https://pypi.org/simple/ \
  sample-shop-utils
```

The extra index allows ordinary dependencies that are not uploaded to TestPyPI to be found on PyPI. Be careful with multiple indexes in environments that use private packages: pip considers candidates from configured indexes together, so do not use this pattern as a substitute for a secure private-index strategy.

## 38. Publish to PyPI

After a successful rehearsal, upload the same verified artifacts to PyPI:

```bash
python -m twine upload dist/*
```

PyPI does not allow the same project version file to be uploaded again after it has been published, even if the file is later removed. If a release contains a mistake, create a new version and rebuild the artifacts.

### 38.1 Trusted Publishing

For automated releases, PyPI Trusted Publishing can use OpenID Connect (OIDC) from a supported CI platform to obtain short-lived publishing credentials. This avoids storing a long-lived PyPI API token as a CI secret. It requires configuring the trusted publisher on PyPI and a compatible workflow; it is not enabled automatically by adding a `pyproject.toml` setting.

## 39. Versioning and Releasing Updates

A common version format is:

```text
MAJOR.MINOR.PATCH
```

Typical semantic-versioning interpretation:

- **MAJOR:** incompatible public API change.
- **MINOR:** backward-compatible feature.
- **PATCH:** backward-compatible bug fix.

Python package versions must also conform to Python's version-specifier rules (PEP 440). For example, `1.1.0rc1` is a release candidate.

A typical release process:

1. Update the version in the authoritative metadata source.
2. Run the test suite.
3. Remove old build artifacts from `dist/` so stale files are not uploaded.
4. Build a fresh sdist and wheel.
5. Inspect the artifact contents and run metadata checks.
6. Test the built distribution.
7. Upload the verified artifacts.

If you use Git, tag the release so the source code and published artifacts can be related to a commit.

---

# Part X — Advanced Package Features

## 40. Namespace Packages

Since Python 3.3, Python supports implicit namespace packages (PEP 420). A namespace package can span multiple directories and does not require an `__init__.py` file in each namespace portion.

A common use case is when separate distributions provide different subpackages under one shared top-level namespace.

Example layout from two independently installed distributions:

```text
Distribution A:
site-packages/
└── company/
    └── core/
        └── __init__.py

Distribution B:
site-packages/
└── company/
    └── analytics/
        └── __init__.py
```

There is no `company/__init__.py`. Python can combine the namespace portions so callers can import both:

```python
import company.core
import company.analytics
```

Key points:

- The namespace directory itself has no `__init__.py`.
- Its `__path__` can cover multiple locations.
- A namespace package may not have a single `__file__`.
- If a regular package named `company` is found, it takes precedence over namespace portions found elsewhere; the regular package does not merge with those portions.
- Do not omit `__init__.py` accidentally in an ordinary application package. Choose namespace packaging deliberately and configure the build backend to include the intended packages.

Use namespace packages when multiple distributions genuinely need to contribute to one import namespace. For a normal application or a single library, a regular package is usually simpler.

## 41. Dynamic Imports with `importlib.import_module()`

Sometimes the module name is known only at runtime, for example from a configuration file or a plugin selection.

```python
from importlib import import_module

module_name = "json"
module = import_module(module_name)

payload = '{"status": "ok"}'
data = module.loads(payload)

print(data["status"])
```

Line by line:

1. Import the standard-library helper `import_module`.
2. Store the requested module's name as a string.
3. Load the module dynamically.
4. Call `loads()` from the module object.
5. Read a value from the returned dictionary.

For trusted, known module names this is straightforward. Do not blindly import arbitrary names supplied by an untrusted user: importing a module executes its top-level code.

`importlib.import_module()` is preferred over directly using the lower-level built-in `__import__()` for most dynamic import use cases.

## 42. Lazy Imports with Package-Level `__getattr__`

A package can define module-level `__getattr__` (PEP 562) to resolve attributes on demand.

Example `shop/__init__.py`:

```python
from importlib import import_module

__all__ = ["create_invoice", "analytics"]


def __getattr__(name):
    if name == "create_invoice":
        from .billing import create_invoice
        return create_invoice

    if name == "analytics":
        module = import_module(".analytics", __name__)
        globals()[name] = module  # cache the resolved module on the package
        return module

    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")
```

What happens:

1. Normal attribute lookup on `shop` happens first.
2. If `analytics` is not already present, Python calls `shop.__getattr__("analytics")`.
3. The implementation imports the submodule only at that point.
4. It stores the result in the package globals so future lookup is direct.
5. Unknown names raise `AttributeError`, as expected for a missing attribute.

Lazy loading can reduce import time when an optional or expensive feature is rarely used. It also makes package behavior more complex. Use it only when measurement or a real optional-dependency requirement justifies it.

## 43. Discovering Plugins with Entry Points

A plugin system lets separately installed distributions advertise functionality to an application. Python packaging entry points provide metadata for this purpose.

A plugin distribution can declare an entry point in its `pyproject.toml`:

```toml
[project.entry-points."shop.plugins"]
example = "shop_example_plugin:register"
```

The application can discover and load the plugin:

```python
from importlib.metadata import entry_points

plugins = entry_points(group="shop.plugins")

for plugin_entry in plugins:
    plugin_callable = plugin_entry.load()
    plugin_callable()
```

Line by line:

1. `entry_points` reads installed distribution metadata.
2. `group="shop.plugins"` selects only entry points registered for this plugin group.
3. Each returned entry point describes a registered component.
4. `.load()` imports the referenced object.
5. The example calls the loaded function.

Only load plugins from distributions you trust. Loading a plugin imports third-party code and executes its module-level code.

## 44. Typing Information in Distributions

If a library provides inline type annotations and wants type checkers to treat it as a typed package, it can include a `py.typed` marker file in the package.

Example:

```text
src/
└── shop/
    ├── __init__.py
    ├── billing.py
    └── py.typed
```

For a setuptools package, include it as package data:

```toml
[tool.setuptools.package-data]
shop = ["py.typed", "data/*.json"]
```

The `py.typed` file is normally empty. It signals that the package's inline type information is intended for use by type checkers. Follow PEP 561 for more advanced cases, including stub-only distributions.

## 45. Measuring Import Time

Do not guess which import makes startup slow. Measure it.

```bash
python -X importtime -c "import shop"
```

This prints import timing information to standard error, including self time and cumulative time for imports. Use it to identify expensive import chains before adding lazy-loading complexity.

---

# Part XI — Import Style and Engineering Practices

## 46. PEP 8 Import Ordering

A common import grouping is:

1. Standard-library imports.
2. Third-party imports.
3. Local application/package imports.

Example:

```python
# Standard library
import json
import os
from pathlib import Path

# Third-party
import requests

# Local application
from shop.billing import create_invoice
```

Keep a blank line between groups. Prefer one import per line for ordinary `import` statements and avoid wildcard imports in production code.

Tools such as Ruff and isort can organize and validate imports automatically.

## 47. Common Project Files

| File/folder | Purpose |
|---|---|
| `pyproject.toml` | Build metadata and tool configuration |
| `README.md` | Project documentation |
| `LICENSE` | License text |
| `src/` | Importable package source in a src-layout project |
| `tests/` | Automated tests |
| `.venv/` | Local virtual environment; do not commit it |
| `dist/` | Built distribution files; usually do not commit it |
| `build/` | Temporary build output; usually do not commit it |
| `__pycache__/` | Generated bytecode cache; usually do not commit it |

A common `.gitignore` excerpt:

```gitignore
.venv/
__pycache__/
*.py[cod]
.pytest_cache/
build/
dist/
*.egg-info/
```

Do not put credentials, API tokens, or private keys in source files or package metadata.

## 48. Common Packaging and Importing Pitfalls

| Symptom | Likely cause | Practical fix |
|---|---|---|
| `ModuleNotFoundError` for your own package | Package is not installed or project root is not on the correct import path | Activate the intended environment and install with `python -m pip install -e .` |
| Works in repository, fails after installation | Source-tree import masked a missing packaged file | Build a wheel and test it in a clean environment |
| `attempted relative import with no known parent package` | A package module was run directly by file path | Run it with `python -m package.module` or provide a package entry point |
| `partially initialized module` | Circular import or local module shadowing | Inspect import order and `module.__file__`; consider restructuring modules |
| Wrong version is imported | Multiple environments or duplicate module names | Check `sys.executable`, `sys.path`, and `module.__file__` |
| Package data missing after install | Data files were not included in the wheel | Configure package data and inspect the wheel contents |
| CLI command not found | Environment not activated, entry point not configured, or package not reinstalled | Check `[project.scripts]` and reinstall the package |
| Old files appear in release | `dist/` contains stale artifacts | Delete `dist/` and rebuild before uploading |
| `from module import name` is stale after reload | Local name still references the previous object | Re-import the name or access it through the module object |
| `isinstance()` unexpectedly fails across imports | Same source file imported under different module names | Use one consistent package import path and avoid running a module both as a script and by its package name |

## 49. Useful Commands Cheat Sheet

| Goal | Command |
|---|---|
| Check Python version | `python --version` |
| Create a venv | `python -m venv .venv` |
| Upgrade pip | `python -m pip install --upgrade pip` |
| Install current project | `python -m pip install .` |
| Install editable project | `python -m pip install -e .` |
| Install development extra | `python -m pip install -e ".[dev]"` |
| Run tests | `python -m pytest` |
| Run a module | `python -m package.module` |
| Run a package | `python -m package` |
| Build sdist and wheel | `python -m build` |
| Inspect installed metadata | `python -m pip show distribution-name` |
| Inspect import path | `python -c "import sys; print(sys.path)"` |
| Inspect a module origin | `python -c "import json; print(json.__file__)"` |
| Measure import time | `python -X importtime -c "import package"` |

---

# Part XII — Practice and Quick Reference

## 50. Practice Exercises

Try solving each exercise before looking at documentation or an existing implementation.

### Exercise 1 — Create a simple module

Create `string_utils.py` with:

- `reverse_text(value)` that returns a reversed string.
- `count_words(value)` that returns the number of whitespace-separated words.

Import both functions from another file and call them.

### Exercise 2 — Build a package

Create this package:

```text
school/
├── __init__.py
├── students.py
└── courses.py
```

Implement one function in each module. Re-export both functions from `school/__init__.py` and import them from the package root.

### Exercise 3 — Absolute and relative imports

Create a nested package:

```text
company/
├── __init__.py
├── employees.py
└── payroll/
    ├── __init__.py
    └── salary.py
```

Use one absolute import and one relative import. Run the module with `python -m company.payroll.salary` from the correct directory.

### Exercise 4 — Main guard

Create a module containing a function and a `main()` function. Verify that `main()` runs when the file is executed but does not run when another module imports it.

### Exercise 5 — Diagnose a circular import

Create two modules that import each other. Observe the partially initialized module error. Refactor the shared class into a third module.

### Exercise 6 — Inspect the import system

Use `sys.modules`, `sys.path`, `module.__file__`, and `importlib.util.find_spec()` to inspect the `json` module.

### Exercise 7 — Build an installable package

Create a `src/`-layout package with `pyproject.toml`, install it in editable mode, run a test, and build a wheel.

### Exercise 8 — Include package data

Add a JSON configuration file to your package. Include it in the wheel and read it with `importlib.resources.files()`.

### Exercise 9 — Create a console command

Add a `main()` function and configure `[project.scripts]`. Install the project and run the generated command.

### Exercise 10 — Test a wheel outside the repository

Build the wheel, install it in a clean virtual environment, move outside the source tree, then import and use the package.


## 51. Commonly Used Standard-Library Modules and APIs

This is the practical map of the tools used throughout this guide. You do not need to memorize every function; know which module to consult for each job.

| Module / API | Common use |
|---|---|
| `sys` | Inspect interpreter state, `sys.path`, `sys.modules`, `sys.meta_path`, and `sys.executable` |
| `importlib` | Import modules dynamically and reload modules |
| `importlib.util` | Find module specifications with `find_spec()` |
| `importlib.resources` | Read data files shipped inside packages |
| `importlib.metadata` | Read installed distribution metadata and discover entry points |
| `venv` | Create isolated Python environments |
| `runpy` | Run a module or script using Python's execution mechanisms; mainly useful for advanced tooling |

### 51.1 Inspect the active Python environment with `sys`

```python
import sys

print(sys.executable)       # Exact Python interpreter running this program
print(sys.version)          # Full interpreter version/build information
print(sys.path)             # Search locations used by the path-based finder
print("json" in sys.modules)  # Whether json is already loaded by this name
```

### 51.2 Find a module with `importlib.util`

```python
from importlib.util import find_spec

spec = find_spec("json")
if spec is None:
    print("json cannot be found")
else:
    print("Found:", spec.name)
    print("Origin:", spec.origin)
```

This checks discovery, not whether importing the module will successfully execute all of its code.

### 51.3 Read package resources with `importlib.resources`

```python
from importlib.resources import files

resource = files("shop").joinpath("data", "tax_rates.json")
contents = resource.read_text(encoding="utf-8")
print(contents)
```

Use this rather than building a resource path from the current working directory.

### 51.4 Read installed package metadata with `importlib.metadata`

```python
from importlib.metadata import metadata, version

print(version("sample-shop-utils"))
project_metadata = metadata("sample-shop-utils")
print(project_metadata["Name"])
```

These functions read metadata for an installed distribution. They do not import the Python package itself.

### 51.5 When would you use `runpy`?

`runpy.run_module("package.module", run_name="__main__")` executes a module's code using module execution semantics and returns its resulting global namespace. It is useful for tooling that needs to run modules programmatically. For ordinary application commands, prefer the shell form `python -m package.module` because it is easier to understand and debug.

## 52. Quick Reference

| Concept | Syntax / API | Purpose |
|---|---|---|
| Import a module | `import module` | Bind module object by its top-level name |
| Import a name | `from module import name` | Bind a specific object directly |
| Alias | `import module as alias` | Use a local alternative name |
| Package initializer | `__init__.py` | Initialize a regular package |
| Wildcard exports | `__all__` | Control names imported by `import *` |
| Module cache | `sys.modules` | See loaded modules by import name |
| Search path | `sys.path` | Path-based search locations |
| Module metadata | `__name__`, `__package__`, `__spec__` | Inspect runtime/import context |
| Main guard | `if __name__ == "__main__":` | Prevent command-line code from running on import |
| Package runner | `__main__.py` | Support `python -m package` |
| Dynamic import | `importlib.import_module()` | Import from a runtime string |
| Module discovery | `importlib.util.find_spec()` | Ask whether a module can be found |
| Reload module | `importlib.reload()` | Re-execute an imported module (with caveats) |
| Package data | `importlib.resources.files()` | Read resources attached to a package |
| Distribution version | `importlib.metadata.version()` | Read installed distribution metadata |
| Build config | `pyproject.toml` | Declare build system and project metadata |
| Editable install | `python -m pip install -e .` | Install for local development |
| Build artifacts | `python -m build` | Build sdist and wheel |
| CLI command | `[project.scripts]` | Create an installed command wrapper |
| Plugin discovery | `importlib.metadata.entry_points()` | Discover installed plugins |
| Namespace package | No `__init__.py` at namespace level | Allow multiple distributions to contribute to a namespace |

# Final Notes and Common Gotchas

- A module is an importable unit; a `.py` file is the most common implementation, not the only one.
- A regular package normally has `__init__.py`; a namespace package deliberately does not.
- An import executes module-level code the first time that module name is loaded in the process.
- `sys.modules` is keyed by import name, not by physical source-file path.
- `import package.module` binds the top-level package name; `import package.module as alias` binds the alias.
- `from module import name` binds the object reference into the caller's namespace.
- `__all__` controls wildcard imports; it does not provide privacy or block explicit imports.
- Importing a submodule first initializes its parent package, so keep `__init__.py` lightweight.
- Relative imports need package context; run package modules with `python -m` rather than by file path.
- The main guard prevents command-line behavior from running as a side effect of an ordinary import.
- A circular import is an initialization-order issue involving partially initialized modules, not a deadlock.
- A local file can shadow a standard-library or third-party module if it is found earlier in the import search process.
- Editable installation is useful during development, but a built wheel must be tested separately.
- Distribution names and import names are separate concepts.
- Non-Python files must be deliberately included in the distribution if runtime code needs them.
- Use `importlib.resources` rather than assuming package data lives at a path relative to the current working directory.
- Use modern packaging frontends such as `python -m build`, `python -m pip install`, and Twine rather than deprecated direct `setup.py` commands.

These checks validate the examples in the stated environment. They do not guarantee that every third-party build backend, operating system, Python version, or CI configuration behaves identically.

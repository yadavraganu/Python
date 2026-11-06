# Python Modules and Packages

A module is a single `.py` file, a package is a directory of modules. Code modularity lets you break large applications into smaller, manageable, and reusable pieces.

## 1. Definitions and Architecture

### Modules
A module is a single Python file containing executable code, function definitions, classes, or variables.

* **File Name**: Any file ending in `.py` (e.g., `calculator.py`).
* **Purpose**: Groups related code together to avoid naming conflicts and maximize reusability.

### Packages
A package is a collection of modules organized in a directory structure.

* **Folder Name**: Any directory Python can find on `sys.path` (its name must be a valid identifier to be imported with normal `import` syntax).
* **The `__init__.py` file**: Marks the directory as a **regular package**. (Since Python 3.3 a directory *without* it can still be imported as an implicit namespace package, see section 10, but include `__init__.py` unless you specifically want that behavior.)
  * It can be completely empty.
  * It runs initialization code when the package is first imported.
  * It can use `__all__` to control what gets exported during wildcard (`*`) imports.

### Typical Directory Structure
```
my_project/                 # Project Root Folder
│
├── main.py                 # Main entry script
│
├── utilities.py            # A Module
│
└── shop/                   # A Package
    ├── __init__.py         # Makes 'shop' a package
    ├── billing.py          # Module inside 'shop'
    ├── inventory.py        # Module inside 'shop'
    │
    └── delivery/           # A Sub-package (nested package)
        ├── __init__.py     # Makes 'delivery' a sub-package
        └── tracking.py     # Module inside 'delivery'
```

## 2. The Import Mechanism

Python provides several syntax options to bring external code into your active workspace.

### Standard Import
Imports the entire module. You must use dot (`.`) syntax to access anything inside it.

* **Syntax**: `import package.module`
```python
import shop.billing
# Accessing a function requires the full path
shop.billing.create_invoice()
```

**What actually gets bound:** `import shop.billing` binds only the top-level name `shop` in your namespace; `billing` is reached as an attribute of it, which is why the full dotted path is required. (`import shop.billing as b` binds `b` directly to the submodule.) Similarly, `from shop import billing` works even though `billing` is a module rather than a name defined inside `shop`: Python first looks for an attribute called `billing` on the package and, if it isn't there, tries importing `shop.billing` as a submodule.

### Specific `from ... import`
Imports specific parts (functions, classes, variables) directly into your namespace. No module-name prefix needed.

* **Syntax**: `from package.module import items`
```python
from shop.billing import create_invoice
create_invoice()
```

### Alias Imports (`as`)
Renames a module or function locally — shortens long names or avoids naming conflicts.

* **Syntax**: `import module as alias`
```python
import shop.delivery.tracking as track
from shop.inventory import get_item_count as check_stock

track.locate_package()
items = check_stock()
```

### Wildcard Imports (`from ... import *`)
Imports everything from a module directly into your namespace.

* **Syntax**: `from module import *`
* **Warning**: Avoid in production code — it causes "namespace pollution" because you can't easily tell where a function came from, and it may silently overwrite existing names.

## 3. Controlling Exports with `__all__`

When someone uses a wildcard import, `__all__` in that module (or the package's `__init__.py`) controls exactly what gets exported.

```python
# Inside shop/billing.py
__all__ = ['create_invoice', 'calculate_tax']  # Only these two will be exported

def create_invoice():
    pass

def calculate_tax():
    pass

def _internal_helper():
    # Excluded: not in __all__ (underscore names are also skipped by import * when __all__ is absent)
    pass

def hidden_bonus_function():
    # Clean code, but excluded because it's not listed in __all__
    pass
```

If another file runs `from shop.billing import *`, it only gets `create_invoice` and `calculate_tax`.

### Defining a Package's Public API in `__init__.py`

`__init__.py` is also where a package decides what its *public surface* looks like. Re-exporting names there lets callers skip the internal file layout:

```python
# shop/__init__.py
from .billing import create_invoice, calculate_tax
from .inventory import get_item_count

__all__ = ["create_invoice", "calculate_tax", "get_item_count"]
__version__ = "1.0.0"
```
```python
# Callers now write this...
from shop import create_invoice
# ...instead of this, and you can reorganize billing.py later without breaking them
from shop.billing import create_invoice
```
Importing any submodule (`import shop.billing`) always runs `shop/__init__.py` **first**, so keep it light: heavy work or many eager imports there slows down every import of the package.

## 4. Checking Import State with `in`

`in` is a plain membership operator — it's never part of import syntax itself, but it's very useful for checking the status and contents of imports.

### Verifying Loaded Modules with `sys.modules`
On import, Python loads the module into memory and caches it in the `sys.modules` dictionary.

```python
import sys

# Check if 'json' has been imported anywhere in the application
if "json" in sys.modules:
    print("The JSON module is cached and ready to use.")
```

### Checking Module Content with `dir()`
`dir()` lists all functions, classes, and variables inside a module — useful for checking a feature exists before using it (e.g., across Python versions).

```python
import math

# Check if the 'factorial' function exists in this version of Python
if "factorial" in dir(math):
    print(math.factorial(5))
```
`hasattr(math, "factorial")` is the more idiomatic spelling of the same check.

### Checking Whether a Module Is Available (Optional Dependencies)

Two common patterns:

```python
# 1. Try the import, fall back gracefully
try:
    import numpy as np
except ImportError:          # ModuleNotFoundError is a subclass of ImportError
    np = None

# 2. Ask without importing the module itself
import importlib.util
if importlib.util.find_spec("numpy") is not None:
    print("numpy is installed")
```
The first is the standard way to support optional dependencies; the second is handy when you only need to know whether something is installed.

## 5. Absolute vs. Relative Imports

| Metric | Absolute Imports | Relative Imports |
|---|---|---|
| Path Origin | Starts from a top-level package/module found on `sys.path` (usually your project root) | Starts from the current module's own package |
| Syntax Style | Explicit full path using words | Uses single (`.`) or double (`..`) dots |
| PEP 8 Status | Preferred for readability and safety | Acceptable for complex, deeply nested packages |
| Execution | Works wherever the top-level package is importable | Only works when the module is imported as part of a package (not run as a plain script) |

### Absolute Imports
Outlines the exact path from the project root to the desired resource.

```python
# Working inside: shop/billing.py
from shop.inventory import get_item_count  # Clear, explicit, and readable
```

### Relative Imports
Detects the target module's location based on where the current file is.

* `.` (one dot) — the current package.
* `..` (two dots) — the parent package (one level up); each extra dot climbs one more level.
* Relative imports use the `from ... import ...` form only (`import .inventory` is a syntax error), and they can't climb above the top-level package.

```python
# Working inside: shop/billing.py
from .inventory import get_item_count  # Look in the same folder for inventory.py
```
```python
# Working inside: shop/delivery/tracking.py
from ..billing import create_invoice   # Go up to 'shop', then look for billing.py
```

### The Relative Import Trap
Relative imports only work when Python runs your code **as part of a package**. If you `cd` directly into `shop/` and run:

```bash
python billing.py
```

you'll get:
```
ImportError: attempted relative import with no known parent package
```

**The fix:** run from the project root using the module flag (`-m`):
```bash
python -m shop.billing
```

Running the file by path from the project root (`python shop/billing.py`) fails the same way, because Python treats it as a standalone script with no package context. This example assumes the flat layout from section 1; with the `src/` layout in section 8, first run `pip install -e .`, after which `python -m shop.billing` works from any directory.

## 6. Module Lifecycle and Resolution

### Module Caching and Re-execution
Python executes a module's code exactly once per process, then caches the resulting module object in `sys.modules`.

* Subsequent imports of the same module don't re-run the file — they read it from the cache.
* To forcefully re-run a module (e.g., live debugging in a shell), use `importlib`:
```python
import importlib
import shop.billing

importlib.reload(shop.billing)  # Force re-executes the file
```

**`reload` gotcha:** it re-executes the module and updates the existing module object in place, but any name you previously bound with `from shop.billing import create_invoice` still points at the **old** function object. Only code that accesses it through the module (`shop.billing.create_invoice()`) sees the new version. This is one reason `import module` is friendlier to live-reloading than `from module import name`.

### The Double-Import Problem

`sys.modules` caches by **name**, not by file. If one file is reachable under two different names, it is executed twice and you get two separate module objects:
- Putting both `project/` and `project/shop/` on `sys.path` lets you `import billing` *and* `import shop.billing`: two names, one file.
- Running `python shop/billing.py` executes the file as `__main__`; if another module then does `import shop.billing`, the file runs **again** under its real name. The result is duplicate classes, duplicate global state, and baffling `isinstance` failures.

Avoid it by using consistent absolute imports and exactly one import root on `sys.path`.

### How Python Locates Modules (`sys.path`)
On `import xyz`, Python searches an ordered list of directory paths in `sys.path`:

1. **The Current Directory** — the folder containing the script you launched.
2. **PYTHONPATH** — an optional environment variable with extra directory paths.
3. **Standard Library** — the interpreter's library directories (where `os`, `json`, `random`, etc. live).
4. **Third-Party Packages (`site-packages`)** — where `pip`-installed packages live.

Before `sys.path` is consulted at all, Python checks `sys.modules` and then the **built-in** and **frozen** modules (such as `sys` and `builtins`, which are compiled into the interpreter). That's why a local `sys.py` can't shadow the real `sys`, whereas a local `random.py` or `json.py` can.

If nothing matches across all of these, Python raises `ModuleNotFoundError`. You can inspect or extend the search path directly:
```python
import sys
print(sys.path)
sys.path.append("/custom/path")   # ad-hoc addition, resolved at runtime
```

**Precision on the first entry:** `sys.path[0]` is the **directory of the script you ran** (for `python script.py`), the current working directory (for `python -m package.module`), or an empty string in the interactive shell. Editing `sys.path` inside application code is usually a smell; prefer an editable install (`pip install -e .`) so your package is importable the normal way. Python 3.11+ also offers `python -P` / `PYTHONSAFEPATH=1` to stop the script directory (or cwd) from being prepended automatically, which closes off the accidental-shadowing problem described in the troubleshooting section.

### Bytecode Caching — `__pycache__` and `.pyc` files
When a module is imported (not run directly as `__main__`), CPython compiles its source into bytecode and caches it as a `.pyc` file inside a `__pycache__` folder next to the source (e.g., `__pycache__/billing.cpython-311.pyc`).

* On the next import, Python compares the source file's timestamp/hash against the cached bytecode; if unchanged, it skips recompilation and loads the cached bytecode directly — faster startup.
* `.pyc` files are version-specific (tied to the interpreter version in the filename) and are safe to delete; Python regenerates them automatically.
* This caching is purely an import-time optimization — it has no effect on how the *script you actually run* (`python billing.py`) is executed, since the top-level script itself isn't cached this way.
* You can stop `.pyc` files being written with `python -B` or `PYTHONDONTWRITEBYTECODE=1`, and `__pycache__/` belongs in your `.gitignore`.

## 7. `__name__ == "__main__"` — Script vs. Import Behavior

Every module has a built-in `__name__` attribute:

* If the file is **run directly** (`python billing.py`), Python sets `__name__ = "__main__"`.
* If the file is **imported** by another module, `__name__` is set to the module's actual name (e.g., `"shop.billing"`).

This is why the following guard is one of the most common patterns in Python:

```python
# billing.py
def create_invoice():
    return "Invoice created"

def main():
    print(create_invoice())

if __name__ == "__main__":
    main()   # only runs when billing.py is executed directly, not when imported
```

**Why it matters (frequent interview question):** without the guard, importing `billing` elsewhere (`import shop.billing`) would also immediately execute `main()` — any test/demo code at module level runs as a side effect of import, which is almost never what you want. The guard lets a file serve double duty as both a reusable module and a standalone script.

Other module-level introspection attributes worth knowing:
```python
print(__name__)      # "__main__" or the module's dotted name
print(__file__)      # the file path this code was loaded from
print(__doc__)        # the module's top-level docstring, if any
print(__package__)    # the package this module belongs to (None when run as a plain script, '' for a top-level imported module)
print(__spec__)       # the ModuleSpec the import system used (None for a script run by path)
```
Packages additionally have `__path__`, the list of directories searched for their submodules (`import shop; print(shop.__path__)`); plain modules don't.

## 8. Packaging Your Code — Step-by-Step Guide

Goal: turn the `shop` package into something anyone can `pip install`. Modern packaging is driven by a single `pyproject.toml` file (replacing the older `setup.py`), plus a **build backend** that turns your source into installable files.

**Terminology first (a common source of confusion):**
- **Distribution name**: what you `pip install` (e.g., `my-custom-shop-pkg`), set by `name` in `pyproject.toml`.
- **Import name**: what you `import` (e.g., `shop`), determined by the folder under `src/`.
- They don't have to match, but keeping them close makes life easier. Names must also be **unique on PyPI**, so check availability before committing to one.

### Step 1 — Create the Project Layout

```
my_shared_package/
├── pyproject.toml          # project metadata + build configuration
├── README.md               # shown on the PyPI page
├── LICENSE                 # your license text
├── src/
│   └── shop/               # the importable package
│       ├── __init__.py
│       ├── __main__.py     # optional: enables `python -m shop`
│       └── billing.py
└── tests/
    └── test_billing.py
```

**Why the `src/` layout?** With `src/`, your package is *not* importable from the project root by accident, so you are forced to install it (even in editable mode) before importing it. That surfaces layout and configuration mistakes early, instead of hiding them behind the current directory happening to be on `sys.path`.

### Step 2 — Create and Activate a Virtual Environment

Isolate this project's dependencies so they don't collide with other projects or your system Python:

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
python -m pip install --upgrade pip
```

### Step 3 — Write the Package Code

```python
# src/shop/__init__.py
from .billing import create_invoice

__all__ = ["create_invoice"]
__version__ = "1.0.0"
```
```python
# src/shop/billing.py
def create_invoice():
    return "Invoice created"
```
```python
# src/shop/__main__.py   (the target of the shop-cli entry point in Step 4)
from .billing import create_invoice

def main():
    print(create_invoice())

if __name__ == "__main__":
    main()
```
```python
# tests/test_billing.py
from shop import create_invoice

def test_create_invoice():
    assert create_invoice() == "Invoice created"
```

### Step 4 — Write `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=77.0.0"]        # what pip installs to build your project
build-backend = "setuptools.build_meta"   # which backend does the building

[project]
name = "my-custom-shop-pkg"              # the DISTRIBUTION name (what users pip install)
version = "1.0.0"
description = "An ecommerce utility package"
readme = "README.md"
requires-python = ">=3.10"
license = "MIT"                           # SPDX expression (PEP 639)
authors = [{ name = "Your Name", email = "you@example.com" }]
classifiers = ["Programming Language :: Python :: 3"]
dependencies = ["requests>=2.28.0"]       # runtime dependencies

[project.optional-dependencies]
dev = ["pytest>=8", "build", "twine"]     # installed with: pip install -e ".[dev]"

[project.scripts]
shop-cli = "shop.__main__:main"           # creates a `shop-cli` command on install

[project.urls]
Homepage = "https://github.com/you/my-custom-shop-pkg"

[tool.setuptools.packages.find]
where = ["src"]                           # look for packages inside src/
```

What each part does:
- **`[build-system]`**: names the backend (PEP 517/518). Setuptools is one option; `hatchling`, `flit-core`, and `poetry-core` work through the same table, so swapping backends doesn't change the rest of your file.
- **`[project]`**: standard metadata (PEP 621), understood by every backend and tool.
- **`dependencies`** vs **`optional-dependencies`**: the first is installed for every user; the second is opt-in extras like dev tools.
- **`[project.scripts]`**: turns `shop.__main__:main` into a real command-line program on the user's PATH.
- **`[tool.setuptools.packages.find]`**: tells setuptools to look under `src/`. Recent setuptools can auto-discover a `src/` layout even without it, but being explicit avoids surprises as the layout grows.
- Version numbers here are illustrative; check current guidance for your backend.

*Optional, single-source the version:* instead of repeating `1.0.0` in two places, declare it dynamic and let setuptools read it from the package:
```toml
[project]
dynamic = ["version"]        # and delete the static `version = ...` line

[tool.setuptools.dynamic]
version = { attr = "shop.__version__" }
```

### Step 5 — Install in Editable Mode and Try It

```bash
pip install -e ".[dev]"
python -c "import shop; print(shop.__version__)"
shop-cli
```
An **editable install** links your source folder into the environment, so code edits take effect immediately with no reinstall. If `import shop` works from *any* directory, the package is being discovered correctly. It does **not** prove the packaging is complete: editable installs can hide mistakes such as a data file that won't make it into the wheel, which is exactly what Steps 8 and 9 check.

You can also read the *installed* version from package metadata instead of a hard-coded string:
```python
from importlib.metadata import version
print(version("my-custom-shop-pkg"))   # takes the DISTRIBUTION name, not the import name
```

### Step 6 — Run Your Tests

```bash
pytest
```
Because of the `src/` layout, the tests import `shop` through the installed (editable) package rather than accidentally through the current directory. To test exactly what your users get, also run the suite against the built wheel in a clean environment (Step 9), ideally in CI.

### Step 7 — Build the Distribution Files

```bash
python -m build
```
This creates a `dist/` folder containing two artifacts:

| File | Type | What it is |
|---|---|---|
| `my_custom_shop_pkg-1.0.0.tar.gz` | **sdist** (source distribution) | Your source plus metadata; pip must run the build backend to install it |
| `my_custom_shop_pkg-1.0.0-py3-none-any.whl` | **wheel** (built distribution) | A pre-built zip that pip installs directly, so it's fast and needs no build step |

(The exact spelling of the name part, hyphens vs. underscores, varies with your backend version.)

Reading the wheel filename: `name-version-pythontag-abitag-platformtag`. `py3-none-any` means **pure Python**, working on any OS and any Python 3. Packages with compiled C extensions produce platform-specific wheels instead. Publish **both** files: pip prefers the wheel and falls back to the sdist.

### Step 8 — Inspect and Validate the Artifacts

```bash
python -m zipfile -l dist/*.whl      # list what actually got packaged
tar tf dist/*.tar.gz                  # list the sdist contents
twine check dist/*                    # validates metadata and the README rendering
```
Look for missing files (e.g., data files, which need extra config; see the pitfalls below) *before* uploading anything.

### Step 9 — Test the Built Wheel in a Clean Environment

```bash
python -m venv /tmp/verify-env
source /tmp/verify-env/bin/activate
pip install dist/*.whl
python -c "import shop; print(shop.create_invoice())"
```
This proves the wheel works **without** your source tree or dev environment lying around.

### Step 10 — Publish to TestPyPI First (a Safe Rehearsal)

1. Create accounts on [test.pypi.org](https://test.pypi.org) and [pypi.org](https://pypi.org) (they are separate accounts) and set up 2FA (PyPI requires it).
2. Create an **API token** on TestPyPI.
3. Upload:
```bash
twine upload --repository testpypi dist/*
# username: __token__
# password: <your API token, including the pypi- prefix>
```
4. Install it back from TestPyPI to verify:
```bash
pip install --index-url https://test.pypi.org/simple/ \
            --extra-index-url https://pypi.org/simple/ my-custom-shop-pkg
```
The `--extra-index-url` matters: your dependencies (like `requests`) live on the real PyPI, not TestPyPI.

If `--repository testpypi` isn't recognized in your setup, pass the URL explicitly (`twine upload --repository-url https://test.pypi.org/legacy/ dist/*`) or define the repository in `~/.pypirc`.

### Step 11 — Publish to the Real PyPI

```bash
twine upload dist/*
```
Then `pip install my-custom-shop-pkg` works for anyone in the world.

**Important:** PyPI **never lets you re-upload a version**, even after deleting it. A mistake means bumping the version number, so the TestPyPI rehearsal is worth the extra minute.

**Better than long-lived tokens:** use **Trusted Publishing**. You register your GitHub repository and workflow with PyPI, and the release workflow gets a short-lived credential automatically through OIDC (typically via the `pypa/gh-action-pypi-publish` action), so there's no secret token to store or leak.

### Step 12 — Releasing Updates

1. Bump the version (follow **semantic versioning**: `MAJOR.MINOR.PATCH`, where a breaking change bumps MAJOR, a new feature bumps MINOR, and a fix bumps PATCH). Pre-releases look like `1.1.0rc1` (PEP 440).
2. Commit and tag the release in git (`git tag v1.1.0`).
3. **Delete the old `dist/` folder** (`rm -rf dist/`), or you'll re-upload stale files.
4. Rebuild (`python -m build`), check (`twine check dist/*`), and upload.

### Tooling Alternatives

The steps above use the standard, backend-agnostic tools (`build` + `twine`), so they work everywhere. All-in-one tools wrap the same workflow: **`uv`** (`uv build`, `uv publish`), **Hatch** (`hatch build`, `hatch publish`), and **Poetry** (`poetry build`, `poetry publish`). Knowing the underlying steps makes any of them easy to pick up.

### Declaring Dependencies: Version Specifiers and Requirements Files

| Specifier | Meaning |
|---|---|
| `requests==2.32.3` | Exactly this version |
| `requests>=2.28` | At least this version |
| `requests>=2.28,<3` | A range (typical: allow updates, block the next major version) |
| `requests~=2.28` | "Compatible release": same as `>=2.28, ==2.*` |
| `tomli>=2; python_version < "3.11"` | Environment marker: install only when the condition holds |

**Libraries vs. applications:** a *library* (like this package) should declare **loose** ranges in `pyproject.toml` so it can coexist with other packages. An *application* should **pin exact versions** for reproducible deployments (via `pip freeze > requirements.txt`, or a lock file from `uv`, `pip-tools`, or Poetry) and install them with `pip install -r requirements.txt`.

### Common Packaging Pitfalls

| Symptom | Likely cause and fix |
|---|---|
| `ModuleNotFoundError` after installing your own package | The package wasn't discovered (folder not under `src/`, or the `where = ["src"]` setting doesn't match your layout), or you installed into a different environment than the one you're running (check `python -m pip` and `which python`) |
| `shop-cli` fails with an `ImportError` or "command not found" | The `[project.scripts]` target must point at a real importable function (`shop.__main__:main`); reinstall after changing it |
| Works in your repo, breaks for users | You tested against loose source instead of the installed package (the reason for `src/` layout and Step 9) |
| Non-`.py` files (templates, JSON, data) missing from the wheel | They aren't included by default; declare them, e.g. `[tool.setuptools.package-data] shop = ["data/*.json"]` |
| `400 File already exists` on upload | That version was already published; bump the version and rebuild |
| Old code uploaded after a rebuild | Stale files in `dist/`; clear it before building |
| `pip install` name differs from `import` name | Expected: distribution name and import name are separate (`pip install pillow` → `import PIL`) |
| Secrets committed to git | Never commit API tokens; use Trusted Publishing or CI secrets, and keep `dist/` and `.venv/` in `.gitignore` |

## 9. Common Troubleshooting Guide

### Circular Imports

* **Cause**: Module A imports Module B at the top of its file, and Module B imports Module A at the top of *its* file. Nothing deadlocks or freezes. Python simply finds the other module **partially initialized** in `sys.modules`: it exists, but the names defined *after* the import line haven't been created yet.
```python
# --- user.py ---
from .order import Order      # starts executing order.py right here...
class User: pass              # ...so User doesn't exist yet when order.py needs it
```
```python
# --- order.py ---
from .user import User        # user.py is only partially initialized at this point
class Order: pass
```
* **Symptom**: `ImportError: cannot import name 'User' from partially initialized module 'shop.user' (most likely due to a circular import)`. With the plain form (`import shop.user`, then `shop.user.User`) you'd instead get an `AttributeError` at the moment the missing name is accessed.

* **Solution A — Deferred (inline) imports:** move the import inside the function/method that actually needs it.
```python
# --- order.py ---
class Order:
    def assign_to_user(self):
        from .user import User  # deferred until call time, so the cycle never occurs at import
        pass
```
* **Solution B — Structural redesign:** move shared classes/functions into an independent third module (e.g., `models.py`) that both sides can safely import.
* **Solution C — `TYPE_CHECKING` guard (when the import is only for type hints):** most circular imports in modern code exist purely for annotations. Import the name only for the type checker, never at runtime:
```python
# --- order.py ---
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:            # False at runtime, True for mypy/Pyright
    from .user import User

class Order:
    def assign_to_user(self, user: User) -> None:   # annotation is never evaluated at runtime
        ...
```

### Shadowing / Name Conflicts

* **Cause**: A local file shares a name with a standard-library or third-party module (e.g., a script named `random.py`, `json.py`, `email.py`, or `requests.py`). Since the script's directory comes first in `sys.path`, Python imports your file instead of the real module. (Truly built-in modules compiled into the interpreter, like `sys`, are found before `sys.path` is searched and can't be shadowed this way.)
* **Symptoms**: `AttributeError: module 'requests' has no attribute 'get'`, or, when the shadowing file itself imports that name, `AttributeError: partially initialized module 'random' has no attribute 'randint' (most likely due to a circular import)`. The word "circular" is misleading there; the real cause is the name clash.
* **Diagnose**: print the module's file location (`import random; print(random.__file__)`). If it points into your project, that's the culprit.
* **Solution**: Rename the local file to something unique (e.g., `random_demo.py`) and delete its stale copy in `__pycache__`.

## 10. Advanced Tip: Namespace Packages

Since Python 3.3 (PEP 420), you can create a package **without** an `__init__.py` file — a **namespace package**.

* **Why use it?** It lets large companies or open-source projects split a single package across completely different directories or separate git repositories.
* **Example**: Company A distributes `company.core`; Company B distributes `company.auth`. When a user installs both, Python seamlessly merges them under the single `company` namespace, even though they originated from separate codebases.
* **Caveat:** the shared directory must have **no** `__init__.py` in any portion. If one portion has an `__init__.py` in `company/`, it becomes a regular package and the first one found wins instead of merging. Namespace packages have no single `__file__`, and their `__path__` spans several directories.
* **Gotcha:** because a missing `__init__.py` no longer causes an error, forgetting one in an ordinary project silently produces an implicit namespace package. Imports may still work, but test discovery, packaging, and type checkers can behave differently, so include `__init__.py` unless you *want* a namespace package.

## 11. Import Style and Ordering (PEP 8)

- Put imports at the **top of the file**, one per line for plain `import` statements.
- Group them in this order, with a blank line between groups: **standard library**, then **third-party**, then **local application** imports.
```python
import os
import sys

import requests

from shop.billing import create_invoice
```
- Prefer **absolute imports**; use explicit relative imports only within a package. (Python 3 removed *implicit* relative imports, where a bare `import sibling` used to find a neighboring file.)
- Avoid `from module import *` outside interactive sessions and package `__init__` re-exports.
- Tools like `isort` and `ruff` enforce this ordering automatically.

## 12. Dynamic and Lazy Imports

### Importing by String Name
When the module to load isn't known until runtime (plugin systems, config-driven code), use `importlib.import_module` instead of the low-level `__import__`:

```python
import importlib

module_name = "shop.billing"
billing = importlib.import_module(module_name)
billing.create_invoice()
```

### Lazy Loading
Imports cost startup time. Two techniques to defer them:
- **Inline import** inside the function that needs it (also useful for breaking cycles, but repeated calls pay a small dictionary lookup, not a full re-import, since the module is cached).
- **Module-level `__getattr__` (PEP 562)** lets a package load a heavy submodule only on first access:
```python
# shop/__init__.py
def __getattr__(name):
    if name == "analytics":
        import importlib
        return importlib.import_module(".analytics", __name__)
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")
```

### Measuring Import Cost
```bash
python -X importtime -c "import shop"
```
Prints (to stderr) how long each import took, self and cumulative, which is the quickest way to find what's making a CLI tool slow to start.

## 13. Making a Package Runnable with `__main__.py`

Adding a `__main__.py` file lets you run the **package itself** with `python -m`:

```
shop/
├── __init__.py
├── __main__.py     # executed by: python -m shop
└── billing.py
```
```python
# shop/__main__.py
from .billing import create_invoice

def main():
    print(create_invoice())

if __name__ == "__main__":
    main()
```
```bash
python -m shop
```
This is the standard way to ship a command-line entry point inside a package, and it runs with the package context set up, so relative imports like `from .billing import ...` work (unlike running a file directly).

## Notes & Gotchas

- **`if __name__ == "__main__":`** is the single most-asked module concept — know both *what* it does and *why* (prevents import-time side effects).
- **Modules are only executed once per process** — top-level code (including print statements, class/function definitions) runs at import time, not each time you reference the module afterward. (More precisely: once per *name* in `sys.modules`; the same file imported under two names runs twice.)
- **Circular imports** are usually a design smell — needing an inline import to "fix" one is often a sign two modules are too tightly coupled and share responsibilities that belong in a third module.
- **`__pycache__`/.pyc files** are safe to delete; Python regenerates them and uses source timestamps/hashes to know when to recompile.
- **Wildcard imports (`from x import *`) and `__all__`** are frequently paired questions — know that `__all__` only affects `import *`, not `from module import specific_name`, which always works regardless of `__all__`.
- **A package needs `__init__.py`** for a "regular" package; omitting it deliberately creates a **namespace package** instead — a common point of confusion.
- **`sys.path` order matters** — the current directory is checked before site-packages, which is exactly why local files can accidentally shadow standard-library or third-party modules of the same name.
- **A circular import isn't a deadlock**; it's a *partially initialized module*. `from X import Y` fails with `ImportError` (the name doesn't exist *yet*), while `import X; X.Y` fails with `AttributeError`. Being able to explain that distinction is a strong answer.
- **`TYPE_CHECKING` is the cleanest fix when a cycle exists only for type hints**, since the import never runs at runtime.
- **`import module` vs `from module import name`:** the second binds the object itself, so it goes stale after `importlib.reload()`, and it's also where partial-initialization errors surface.
- **Importing a submodule runs every parent package's `__init__.py` first**, so `import shop.delivery.tracking` executes `shop/__init__.py`, then `shop/delivery/__init__.py`, then `tracking.py`.
- **`python -m package` needs a `__main__.py`**, and running with `-m` (rather than by file path) is what makes relative imports work.
- **Prefer an editable install over editing `sys.path`** to make your own package importable during development.
- **Distribution name vs import name:** `pip install my-custom-shop-pkg` and `import shop` are independent (think `pip install pillow` / `import PIL`).
- **sdist vs wheel:** an sdist is source that needs a build step at install time; a wheel is pre-built and installs directly. Publish both, and pip prefers the wheel.
- **PyPI versions are immutable**: you can never re-upload a version, so rehearse on TestPyPI and bump the version to fix a bad release.
- **`pyproject.toml` separates three concerns:** `[build-system]` (how to build), `[project]` (metadata and dependencies), and `[tool.*]` (backend/tool settings).
- **The double-import problem:** running `python shop/billing.py` executes it as `__main__`, and a later `import shop.billing` executes it again as a separate module object. Always run package code with `python -m`.
- **Built-in modules (`sys`, `builtins`) can't be shadowed by a local file; pure-Python stdlib modules (`random`, `json`) can.** A "partially initialized module ... circular import" error on a stdlib name usually means you shadowed it.
- **`ImportError` vs `ModuleNotFoundError`:** the latter is a subclass raised when no such module exists at all; a bare `ImportError` usually means the module was found but the *name* wasn't (as in a circular import).
- **Libraries declare loose dependency ranges; applications pin exact versions.**
- **`importlib.metadata.version("dist-name")`** reads the installed version from package metadata (using the distribution name, not the import name).

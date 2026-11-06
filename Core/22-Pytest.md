## 1. How pytest Works

Understanding the pipeline explains most "why did pytest do that?" moments.

1. **Startup:** pytest finds the **rootdir** and a **config file**, loads built-in and third-party plugins, and imports `conftest.py` files.
2. **Collection:** it walks the filesystem and imports test modules, building a tree: *Session → Directory/Package → Module → Class → Function*. Each leaf is a test **item** with a **node ID** like `tests/test_cart.py::TestCart::test_add`. **Parametrization happens here**, so a parametrized test becomes several items.
3. **Run, per item:** *setup* (fixtures are created) → *call* (your test body) → *teardown* (fixtures are finalized in reverse order). Each phase can pass, fail, or be skipped, and a failure in setup is reported as an **error**, not a failure.
4. **Reporting:** results are summarized, and the process exits with a code.

| Exit code | Meaning |
|---|---|
| 0 | All tests passed |
| 1 | Some tests failed |
| 2 | Run interrupted (e.g. Ctrl+C) |
| 3 | Internal error |
| 4 | Command-line usage error |
| 5 | No tests were collected |

Exit code 5 matters in CI: a typo in a path can make "zero tests ran" look like a pass if your script ignores the code.

## 2. Installing, Running, and Project Layout

```bash
pip install pytest

pytest                              # discover and run everything from the rootdir
python -m pytest                    # same, but also adds the current directory to sys.path
pytest tests/test_cart.py           # one file
pytest tests/test_cart.py::TestCart::test_add          # one test, by node ID
pytest "tests/test_pricing.py::test_apply_discount[100-25-75.0]"   # one parametrized case (quote the brackets)
```

**`pytest` vs `python -m pytest`:** the second form puts your current directory on `sys.path`, which can make imports "just work" locally and then fail in CI. Prefer an editable install (below) so both behave the same.

### Recommended layout (`src/` layout)

```
my_project/
├── pyproject.toml
├── src/
│   └── shop/
│       ├── __init__.py
│       ├── pricing.py
│       └── cart.py
└── tests/
    ├── conftest.py            # shared fixtures and hooks
    ├── test_pricing.py
    ├── test_cart.py
    └── integration/
        ├── conftest.py        # fixtures only for this directory
        └── test_checkout.py
```

Make the package importable by installing it in editable mode:

```bash
pip install -e .
```
or, as a lighter alternative, tell pytest where to look with the `pythonpath` option (section 3).

### Import modes and duplicate file names

By default pytest uses `--import-mode=prepend`: it inserts each test file's root directory at the front of `sys.path` and imports it. Two consequences:

- **Without `__init__.py`**, test files must have **unique base names** across the whole suite. Two `tests/a/test_utils.py` and `tests/b/test_utils.py` files raise an *import file mismatch* error.
- **With `__init__.py`** in the test directories, modules are imported by their full package path, so duplicate base names are fine.

`--import-mode=importlib` avoids touching `sys.path` and relaxes both rules (it was substantially improved in 8.1). Tests can't import each other in that mode, which is usually the right constraint.

## 3. Configuration

pytest reads one configuration file. **The first match wins**, searching the starting directory and then each parent, in this precedence order (current docs):

1. `pytest.toml`
2. `.pytest.toml`
3. `pytest.ini`
4. `.pytest.ini`
5. `pyproject.toml` (with a `[tool.pytest]` or `[tool.pytest.ini_options]` table)
6. `tox.ini` (needs a `[pytest]` section)
7. `setup.cfg` (needs a `[tool:pytest]` section)

`-c FILE` forces a specific file, and its directory becomes the rootdir. Since 9.0 pytest also **warns** if configuration is spread across more than one file, because only one is ever used. The header of every run prints `rootdir:` and `configfile:` so you can always see which was picked.

### Native TOML (pytest 9.0+)

```toml
# pyproject.toml
[tool.pytest]
minversion = "9.0"
testpaths = ["tests"]
pythonpath = ["src"]
addopts = ["-ra", "--strict-markers"]
markers = [
    "slow: marks tests as slow (deselect with -m 'not slow')",
    "integration: needs external services",
]
filterwarnings = ["error"]
```

Values use real TOML types (lists are lists). A standalone `pytest.toml` or `.pytest.toml` uses a `[pytest]` table instead of `[tool.pytest]`.

### INI-compatibility mode (works on all supported versions)

```toml
# pyproject.toml
[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["tests"]
pythonpath = ["src"]
addopts = "-ra --strict-markers"
markers = [
    "slow: marks tests as slow",
    "integration: needs external services",
]
```

Here every value is treated as a string or list of strings. The two tables **cannot be used together**, and `[tool.pytest.ini_options]` remains supported.

```ini
# pytest.ini
[pytest]
minversion = 8.0
testpaths = tests
addopts = -ra --strict-markers
markers =
    slow: marks tests as slow
    integration: needs external services
```

### Options worth knowing

| Option | Purpose |
|---|---|
| `testpaths` | Where to look when you run plain `pytest` |
| `pythonpath` | Extra `sys.path` entries (e.g. `src`) |
| `addopts` | Command-line flags always applied |
| `markers` | Register custom markers |
| `filterwarnings` | Turn warnings into errors, or ignore specific ones |
| `python_files`, `python_classes`, `python_functions` | Change the discovery naming rules |
| `norecursedirs` | Directories to skip |
| `minversion`, `required_plugins` | Fail early if pytest or a plugin is too old or missing |
| `xfail_strict` / `strict_xfail` | Make unexpected passes of `xfail` tests fail |
| `log_cli`, `log_cli_level` | Show log records live while tests run |
| `tmp_path_retention_count` (default 3), `tmp_path_retention_policy` (`all`, `failed`, `none`; default `all`) | How many runs of `tmp_path` directories to keep, and for which outcomes |
| `collect_imported_tests` (8.4+) | Set `false` to collect only tests *defined* in a file, not ones it merely imports |

**Strict mode (9.0+):** the single option `strict = true` turns on `strict_config` (config problems are errors), `strict_markers` (unknown markers are errors), `strict_xfail` (an unexpected pass of an `xfail` test fails) and `strict_parametrization_ids` (duplicate parametrization IDs are an error) together. The docs advise enabling it only when you pin your pytest version, because future releases may add more strictness options to it. The individual flags work on older versions too (`--strict-markers`, `--strict-config`).

Override any option from the command line with `-o name=value`, and put one-off flags in the `PYTEST_ADDOPTS` environment variable.

## 4. Test Discovery and Selection

By default pytest collects:

- files named `test_*.py` or `*_test.py`
- functions and methods whose names start with `test`
- classes whose names start with `Test` **and have no `__init__`** (otherwise pytest warns it can't collect them)

A function named `check_total` is **silently never run**. That is the most common "my test isn't running" cause. `pytest --collect-only -q` prints exactly what was collected.

### Selecting tests

```bash
pytest -k "cart and not slow"       # by name (substring, case-insensitive) with and/or/not
pytest -m "integration"             # by marker
pytest -m "not slow"                # marker expressions
pytest --lf                         # only the tests that failed last run
pytest --ff                         # failed first, then the rest
pytest --nf                         # new files/tests first
pytest --sw                         # stepwise: stop at first failure, resume from there next time
pytest -x                           # stop after the first failure
pytest --maxfail=3                  # stop after 3 failures
pytest @args.txt                    # read arguments from a file (8.2+)
```

`--lf`, `--ff`, `--nf` and `--sw` rely on the cache stored in `.pytest_cache/` (add it to `.gitignore`; clear it with `--cache-clear`).

### Class-based tests

```python
class TestCart:
    def test_new_cart_is_empty(self):
        assert len(Cart()) == 0
```

pytest creates a **new instance of the class for every test method**, so instance attributes set in one test never leak into another. Don't write `__init__`; use fixtures for setup.

## 5. Writing Assertions

pytest uses the plain `assert` statement. At import time it **rewrites** the assertion in test modules to capture the intermediate values, so a failure explains itself:

```python
def test_discount():
    assert apply_discount(100, 25) == 70
```
A failure looks roughly like this (exact layout varies by version):
```
>       assert apply_discount(100, 25) == 70
E       assert 75.0 == 70
E        +  where 75.0 = apply_discount(100, 25)
```
Lists, dicts, sets and long strings get a readable diff. In 9.1 the `assertion_text_diff_style` option can show string failures as separate `Left:` and `Right:` blocks instead of an `ndiff`. Use `-v` or `-vv` for more detail.

### Where rewriting applies

Rewriting happens for **test modules, `conftest.py` files, and plugins** that pytest imports. A helper module that tests import (say `tests/helpers.py` with its own `assert`s) is **not** rewritten, so its assertion failures are bare. Fix that by registering it *before* it is imported:

```python
# conftest.py
import pytest
pytest.register_assert_rewrite("tests.helpers")
```
Disable rewriting with `--assert=plain` (or a `PYTEST_DONT_REWRITE` marker in a module docstring) if it ever causes trouble.

### Custom failure messages

```python
assert response.status_code == 200, f"unexpected body: {response.text}"
```

### Classic traps

```python
assert (x == 1, "x should be one")   # always true! A non-empty tuple is truthy.
```
Python emits a `SyntaxWarning` ("assertion is always true, perhaps remove parentheses?") for this. Write `assert x == 1, "message"`.

### Comparing floats: `pytest.approx`

```python
import pytest

def test_float_sum():
    assert 0.1 + 0.2 == pytest.approx(0.3)             # default tolerance: relative 1e-6, absolute 1e-12
    assert 2.0 == pytest.approx(2.05, abs=0.1)         # explicit absolute tolerance
    assert [0.1 + 0.2, 1.0] == pytest.approx([0.3, 1.0])            # sequences
    assert {"a": 0.1 + 0.2} == pytest.approx({"a": 0.3})            # dicts (values)
    assert float("nan") == pytest.approx(float("nan"), nan_ok=True)
```
`approx` only supports equality comparison. Since 9.1 it also compares `datetime` and `timedelta` values, but you must pass an explicit `abs` tolerance as a `timedelta`.

### Tests must return `None` (8.4+)

Since 8.4 a test that returns anything other than `None` **fails** (it used to warn), and a test function containing `yield` is an explicit error. Don't `return` a boolean from a test; assert it.

## 6. Testing Exceptions and Warnings

### `pytest.raises`

```python
import pytest

def test_discount_rejects_bad_percent():
    with pytest.raises(ValueError, match="between 0 and 100"):
        apply_discount(100, 150)
```

- `match` is searched in `str(exception)` with **`re.search`**, so it's a regex: escape metacharacters (`re.escape(...)`) when you want a literal match. A `|` in a pattern means alternation.
- Keep the `with` block to **one statement that should raise**. If setup code inside the block can raise the same exception, the test passes for the wrong reason.
- Capture details with `as`:

```python
def test_error_details():
    with pytest.raises(KeyError) as excinfo:
        Cart().remove("missing")
    assert excinfo.type is KeyError
    assert excinfo.value.args == ("missing",)
```
`excinfo` is an `ExceptionInfo`: `.type`, `.value`, `.traceback`, and `.match(regex)`.

For exception groups, `pytest.RaisesGroup` (8.4+) is the counterpart of `except*`:

```python
def test_group():
    with pytest.RaisesGroup(ValueError, TypeError):
        raise ExceptionGroup("boom", [ValueError("a"), TypeError("b")])
```

### Warnings

```python
import warnings, pytest

def test_old_api_warns():
    with pytest.warns(DeprecationWarning, match="use new_api"):
        warnings.warn("use new_api instead", DeprecationWarning)

def test_no_warning_emitted(recwarn):
    do_something()
    assert len(recwarn) == 0          # `recwarn` records all warnings in the test
```
`pytest.deprecated_call()` is shorthand for expecting a `DeprecationWarning` or `PendingDeprecationWarning`. To treat warnings as errors project-wide, use `filterwarnings = ["error"]` in config or run with `-W error`; to relax one test, use `@pytest.mark.filterwarnings("ignore:some text:UserWarning")`.

### Imperative outcomes

```python
pytest.skip("reason")                       # skip from inside a test or fixture
pytest.xfail("known bug")                   # mark as expected failure imperatively
pytest.fail("explicit failure message")
np = pytest.importorskip("numpy", minversion="1.26")    # skip if missing or too old
```
At module level, `pytest.skip("reason", allow_module_level=True)` skips the whole file.

## 7. Fixtures

A **fixture** is a function that provides a resource to a test. Instead of calling it, the test **declares it as a parameter by name** and pytest injects the result. This is dependency injection, and it replaces `setUp`/`tearDown` with something composable.

```python
import pytest
from shop.cart import Cart

@pytest.fixture
def cart():
    return Cart()

def test_empty_cart_total(cart):
    assert cart.total == 0
```

### Setup and teardown with `yield`

```python
@pytest.fixture
def db():
    conn = FakeDB()          # setup
    yield conn               # the test runs here; this value is what the test receives
    conn.close()             # teardown: runs even if the test failed
```
Code after `yield` always runs. Wrap risky setup in `try`/`finally` if part of the setup can fail after something is already allocated.

### Ordering rules

```python
@pytest.fixture
def a():
    print("setup a"); yield; print("teardown a")

@pytest.fixture
def b(a):                    # b depends on a
    print("setup b"); yield; print("teardown b")

def test_order(b):
    print("test")
# setup a, setup b, test, teardown b, teardown a
```

- A fixture's dependencies are set up first, and **teardown runs in reverse order of setup**.
- **Higher-scoped fixtures are set up before lower-scoped ones**, and `autouse` fixtures come before others within the same scope.
- A fixture is created at most once per its scope, no matter how many tests or other fixtures request it.

### Scopes

| Scope | The fixture is created... |
|---|---|
| `function` (default) | once per test |
| `class` | once per test class |
| `module` | once per test file |
| `package` | once per test package |
| `session` | once per run |

```python
@pytest.fixture(scope="session")
def expensive_client():
    client = connect()       # slow, so share it
    yield client
    client.close()
```

**A fixture can only depend on fixtures of the same or a wider scope.** A `session` fixture requesting a `function` fixture raises `ScopeMismatch`, because the narrower value would be torn down while the wider one still holds it.

**Mutable shared state is the classic scope bug.** If a `module`- or `session`-scoped fixture returns a list or dict and one test mutates it, later tests see the mutation, and tests pass alone but fail together. Return immutable data, a factory, or keep that fixture function-scoped.

**Dynamic scope:** pass a callable that receives `(fixture_name, config)`:

```python
def determine_scope(fixture_name, config):
    return "session" if config.getoption("--reuse-db", None) else "function"

@pytest.fixture(scope=determine_scope)
def database(): ...
```

### Fixtures that take parameters

```python
@pytest.fixture(params=["sqlite", "memory"], ids=["sql", "mem"])
def backend(request):
    return make_backend(request.param)       # `request.param` holds the current value

def test_roundtrip(backend):                  # runs twice: test_roundtrip[sql], test_roundtrip[mem]
    assert backend.get(backend.put("x")) == "x"
```
Every test that uses `backend` runs once per param. This is the right tool when **the dependency varies** (which database, which client), and `@pytest.mark.parametrize` is right when **the data varies**.

### The `request` object

`request` is a built-in fixture describing the requesting context:

| Attribute / method | Use |
|---|---|
| `request.param` | Current value in a parametrized fixture |
| `request.node` | The test item (read markers with `request.node.get_closest_marker("name")`) |
| `request.config` | The pytest config (`request.config.getoption("--flag")`) |
| `request.addfinalizer(fn)` | Register teardown to run after the fixture (alternative to `yield`) |
| `request.getfixturevalue("name")` | Request another fixture **dynamically** at run time |
| `request.scope`, `request.module`, `request.cls`, `request.instance` | Information about where the fixture is being used |

```python
@pytest.fixture
def user(request):
    marker = request.node.get_closest_marker("admin")
    return make_user(admin=marker is not None)
```
`getfixturevalue` is useful when the choice of fixture depends on a parameter. **Don't call it during teardown** to pull in a fixture that wasn't already requested: that's deprecated in 9.1 and becomes an error in pytest 10.

### Autouse and `usefixtures`

```python
@pytest.fixture(autouse=True)
def clean_environment(monkeypatch):
    monkeypatch.delenv("APP_DEBUG", raising=False)
```
An **autouse** fixture runs for every test in its scope without being requested. It is powerful and invisible, so keep it to genuinely universal setup. To apply a fixture to specific tests without taking its value, use the marker:

```python
@pytest.mark.usefixtures("clean_environment")
class TestSettings: ...
```

### Factory fixtures

When a test needs several variants of an object, return a function:

```python
@pytest.fixture
def make_cart():
    carts = []
    def _make(**items):
        cart = Cart()
        for sku, price in items.items():
            cart.add(sku, price)
        carts.append(cart)
        return cart
    yield _make
    for c in carts:          # teardown can clean up everything the factory made
        c.clear()

def test_total_of_two_items(make_cart):
    cart = make_cart(apple=1.5, pear=2.0)
    assert cart.total == 3.5
```

### `conftest.py` and fixture visibility

Fixtures defined in a `conftest.py` are available to every test in that directory **and below**, with **no import**. This is the standard way to share fixtures.

- Multiple `conftest.py` files can exist at different depths.
- A fixture with the same name defined **closer to the test overrides** the one further away (conftest → module → class), and the closer fixture can request the overridden one by its own name.
- Don't import fixtures from one test file into another. Move them to a `conftest.py`.
- `pytest_plugins = [...]` is only allowed in the top-level (rootdir) `conftest.py`.

```python
# tests/conftest.py
@pytest.fixture
def client():
    return Client("prod-like")

# tests/integration/conftest.py
@pytest.fixture
def client(client):                 # overrides, but wraps the parent fixture
    client.enable_debug()
    return client
```

### Class-scoped fixtures (a 9.1 change)

A fixture with `scope="class"` defined as an **instance method** sets attributes on a different instance than the one the test methods get, which silently breaks it. Defining one without `@classmethod` is deprecated in 9.1. Use a `@classmethod`, or a normal class-scoped fixture function that returns a value.

### Naming and inspecting fixtures

```python
@pytest.fixture(name="cart")         # avoids shadowing warnings when the function is named differently
def make_default_cart(): ...
```
```bash
pytest --fixtures                  # list available fixtures with docstrings
pytest --fixtures-per-test         # which fixtures each test uses
pytest --setup-show                # print fixture setup/teardown as it happens
```
In `--setup-show`, the letter before a fixture name is its scope: `F` function, `C` class, `M` module, `P` package, `S` session.

## 8. Parametrization

### Basics

```python
import pytest
from shop.pricing import apply_discount

@pytest.mark.parametrize(
    "price, percent, expected",
    [
        (100, 0, 100.0),
        (100, 25, 75.0),
        (19.99, 10, 17.99),
        (0, 50, 0.0),
        (100, 100, 0.0),
    ],
)
def test_apply_discount(price, percent, expected):
    assert apply_discount(price, percent) == expected
```
Each row becomes its own test, with an ID built from the values (`test_apply_discount[100-25-75.0]`), so a failure points at the exact case.

### Test IDs

```python
@pytest.mark.parametrize(
    "text, expected",
    [
        pytest.param("a", 1, id="single-char"),
        pytest.param("", 0, id="empty"),
        pytest.param("abc", 4, marks=pytest.mark.xfail(reason="off-by-one bug #12"), id="miscounted"),
    ],
)
def test_length(text, expected):
    assert len(text) == expected
```
- Pass `ids=[...]` (a list) or `ids=some_function` to name cases, or set `id=` on each `pytest.param`.
- `pytest.param(..., marks=...)` attaches `skip`/`xfail`/custom marks to one case.
- **Duplicate IDs:** by default pytest makes clashing IDs unique by appending `0`, `1`, ... With `strict_parametrization_ids` (part of strict mode, 9.0+) a duplicate is a collection **error** instead. Note that `1` and `"1"` produce the same ID (`1-2`), so rows that differ only by type collide; give them explicit `id=`s.

### Stacking decorators multiplies cases

```python
@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [10, 20])
def test_grid(x, y): ...             # 4 tests: every combination
```

### Indirect parametrization

Route a parameter *through a fixture* so setup can use it:

```python
@pytest.fixture
def user(request):
    return make_user(role=request.param)

@pytest.mark.parametrize("user", ["admin", "guest"], indirect=True)
def test_permissions(user): ...
```

### Generating parameters dynamically

```python
# conftest.py
def pytest_generate_tests(metafunc):
    if "backend_name" in metafunc.fixturenames:
        metafunc.parametrize("backend_name", discover_backends())
```

### Parametrize pitfalls

- **Pass a real collection** (a list or tuple) as the argvalues, not a generator or iterator. An iterator is exhausted after the first collection pass, so a second collection (e.g. running `pytest.main()` twice, or a class-level decorator) gets no cases and tests go missing. This is deprecated in 9.1.
- A single argument name with a trailing comma (`"arg,"`) now behaves like the tuple form `("arg",)`, treating each value as a tuple to unpack (fixed in 9.1).
- Don't compute expected values with the same logic as the code under test.

### Subtests (9.0+)

Parametrization needs the cases at collection time. When you only learn them while the test runs, use the built-in `subtests` fixture: each failing subtest is reported on its own and the loop continues.

```python
def test_all_files_have_docstring(subtests):
    for path in find_python_files():
        with subtests.test(path=str(path)):
            assert has_docstring(path)
```
The reporting format of subtests may still change, but the usage is stable. It merges the former `pytest-subtests` plugin into core.

## 9. Markers

### Built-in markers

```python
import sys, pytest

@pytest.mark.skip(reason="not implemented yet")
def test_future(): ...

@pytest.mark.skipif(sys.platform == "win32", reason="POSIX only")
def test_posix_paths(): ...

@pytest.mark.xfail(raises=ZeroDivisionError, strict=True, reason="bug #42")
def test_known_bug():
    1 / 0
```

| Marker | Effect |
|---|---|
| `skip` | Never runs the test |
| `skipif(condition, reason=...)` | Skips when the condition is true. With a boolean condition `reason` is required |
| `xfail(condition, reason, raises, run, strict)` | Expected to fail: failure is reported as **xfail**; an unexpected pass is **xpass** |
| `parametrize` | See section 8 |
| `usefixtures` | Apply fixtures without taking their values |
| `filterwarnings` | Per-test warning filters |

- **`raises=`** limits the expectation to specific exception types, so an unrelated crash still fails the test.
- **`strict=True`** (or `xfail_strict` / `strict_xfail` config) turns an unexpected pass into a failure, which stops stale `xfail` markers from hiding fixed bugs.
- **`run=False`** doesn't execute the test at all (useful if it crashes the interpreter).

### Custom markers

Register them, or pytest warns about unknown marks (and `--strict-markers` turns that into an error):

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = ["slow: takes more than a second", "integration: needs external services"]
```
```python
@pytest.mark.slow
def test_big_import(): ...
```
```bash
pytest -m "not slow"
pytest -m "integration and not slow"
pytest --markers                     # list all registered markers
```

### Marking classes and whole modules

```python
pytestmark = pytest.mark.integration          # every test in this module

@pytest.mark.slow                              # every test in this class
class TestBulkImport: ...
```
`pytestmark = [pytest.mark.a, pytest.mark.b]` takes a list.

## 10. Built-in Fixtures

| Fixture | Scope | What it gives you |
|---|---|---|
| `tmp_path` | function | A unique `pathlib.Path` directory per test |
| `tmp_path_factory` | session | Make shared temp dirs: `tmp_path_factory.mktemp("data")` |
| `monkeypatch` | function | Safely patch attributes, dict items, env vars, `cwd`, `sys.path` |
| `capsys`, `capfd` | function | Capture stdout/stderr (Python level / file-descriptor level) |
| `capsysbinary`, `capfdbinary` | function | Same, but bytes |
| `capteesys` | function | Like `capsys`, but output still reaches the terminal |
| `caplog` | function | Captured log records |
| `recwarn` | function | Records warnings |
| `request` | function | Info about the requesting test (section 7) |
| `pytestconfig` | session | The config object |
| `cache` | session | Persist values across runs (`cache.get("k", None)`, `cache.set("k", v)`) |
| `subtests` | function | Subtest support (9.0+) |
| `record_property` | function | Add custom properties to the report/JUnit XML |

### `tmp_path`

```python
import json
from shop.config import load_settings

def test_load_settings(tmp_path):
    path = tmp_path / "settings.json"
    path.write_text(json.dumps({"debug": False, "name": "shop"}))
    assert load_settings(path) == {"debug": False, "name": "shop"}
```
Directories live under the system temp dir in `pytest-of-<user>/pytest-N`. pytest keeps the last **3** runs by default (`tmp_path_retention_count`) and, with the default `tmp_path_retention_policy = "all"`, keeps them regardless of outcome. Set the policy to `failed` to keep only directories from failing tests, or `none`. `--basetemp=DIR` uses a fixed location and **deletes it first**, so never point it at a directory with data you care about. The older `tmpdir` fixture returns a `py.path.local`; prefer `tmp_path`.

### `monkeypatch`

```python
def test_env_override(tmp_path, monkeypatch):
    path = tmp_path / "s.json"
    path.write_text('{"debug": false}')
    monkeypatch.setenv("APP_DEBUG", "1")
    assert load_settings(path)["debug"] is True
```

| Method | Use |
|---|---|
| `setattr(obj, "name", value)` or `setattr("pkg.mod.name", value)` | Replace an attribute (raises if it doesn't exist, unless `raising=False`) |
| `delattr(obj, "name")` | Remove an attribute |
| `setitem(dict, key, value)`, `delitem` | Patch a dict |
| `setenv(name, value)`, `delenv(name, raising=True)` | Environment variables |
| `syspath_prepend(path)` | Add to `sys.path` |
| `chdir(path)` | Change the working directory |
| `context()` | A scoped patcher, for use inside wider-scoped fixtures |

Everything is **undone automatically** when the test finishes. A function-scoped `monkeypatch` can't be used by a `module`/`session` fixture. Use `with pytest.MonkeyPatch.context() as mp:` there.

### `capsys` and `caplog`

```python
import logging
from shop.cli import main
from shop.orders import process

def test_cli_prints_greeting(capsys):
    main(["Pune"])
    captured = capsys.readouterr()                # a (out, err) pair
    assert captured.out == "hello Pune\n"

def test_slow_order_is_logged(caplog):
    with caplog.at_level(logging.WARNING):
        process(7, seconds=6.0)
    assert "slow order 7" in caplog.text
    assert caplog.records[0].levelname == "WARNING"
```
`caplog` exposes `.text`, `.records`, `.record_tuples`, `.messages`, `.set_level(...)`, `.at_level(...)` and `.clear()`. From 9.1, logs from **non-propagating loggers** are captured too (previously only records reaching the root logger were).

Output captured during a passing test is hidden; on failure pytest shows it under "Captured stdout/log call". Use `-s` (`--capture=no`) to stream it live, and `breakpoint()` still works under capture.

## 11. Mocking and Test Isolation

### The code under test

```python
# shop/weather.py
import json
import urllib.request

def get_json(url):
    with urllib.request.urlopen(url, timeout=5) as resp:
        return json.load(resp)

def temperature(city):
    data = get_json(f"https://api.example.com/weather?city={city}")
    return data["temp_c"]
```

### Prefer isolation by design

The easiest code to test takes its collaborators as arguments:

```python
def temperature(city, fetch=None):
    fetch = fetch or get_json                      # looked up at CALL time, so patching still works
    return fetch(f"https://api.example.com/weather?city={city}")["temp_c"]

def test_temperature_injected():
    assert temperature("Pune", fetch=lambda url: {"temp_c": 21}) == 21
```
No patching needed. Note the `fetch=None` idiom: writing `fetch=get_json` as the default would bind the *original* function when `def` runs, so later patching `get_json` would have no effect (default values are evaluated once, see `02-Functions.md`). Reach for patching when you can't change the signature.

### `monkeypatch` for simple replacements

```python
def test_temperature_patched(monkeypatch):
    monkeypatch.setattr("shop.weather.get_json", lambda url: {"temp_c": 5})
    assert temperature("Pune") == 5
```

### `unittest.mock` when you need assertions on calls

```python
from unittest.mock import patch

def test_temperature_calls_api():
    with patch("shop.weather.get_json", return_value={"temp_c": 21}) as fake:
        assert temperature("Pune") == 21
    fake.assert_called_once_with("https://api.example.com/weather?city=Pune")
```

### The `pytest-mock` plugin

`pytest-mock` wraps the same API in a `mocker` fixture that **undoes patches automatically**, so you don't need `with` blocks or decorators:

```python
def test_temperature_with_mocker(mocker):
    fake = mocker.patch("shop.weather.get_json", return_value={"temp_c": 21})
    assert temperature("Pune") == 21
    fake.assert_called_once()
    spy = mocker.spy(shop.weather, "temperature")   # wraps the real function and records calls
```

### Rule 1: patch where the name is *looked up*, not where it's defined

```python
# shop/report.py
from shop.weather import get_json        # report.py now holds its OWN reference

def status(url):
    return get_json(url)
```
`patch("shop.weather.get_json")` replaces the attribute in `shop.weather`, but `shop.report.get_json` still points at the original function. Patch **`"shop.report.get_json"`** instead. If code uses `import shop.weather` and calls `shop.weather.get_json(...)`, patching `shop.weather.get_json` works.

### Rule 2: use `autospec` so mocks match the real signature

```python
with patch("shop.weather.get_json", autospec=True) as fake:
    fake.return_value = {"temp_c": 1}
    fake()                                # TypeError: the real function needs a `url` argument
```
A plain `MagicMock` accepts any call, which hides bugs when the real signature changes.

### Controlling behavior

```python
fake.return_value = {"temp_c": 21}
fake.side_effect = TimeoutError                      # raise on every call
fake.side_effect = [{"temp_c": 1}, TimeoutError]     # first call returns, second raises
fake.side_effect = lambda url: {"temp_c": len(url)}  # compute from arguments
```
Useful assertions: `assert_called_once_with(...)`, `assert_not_called()`, `call_count`, `call_args`, `call_args_list`. Use `AsyncMock` for coroutines and `mock_open` for file reads.

### Mocking guidance

- Mock at **boundaries** (network, clock, filesystem, third-party services), not your own pure logic.
- Prefer a **fake** (a small working in-memory implementation) over a mock when behavior matters; tests that assert exact call sequences break whenever you refactor.
- For HTTP, libraries such as `responses` (for `requests`) or `respx` (for `httpx`) are less brittle than hand-patching.
- For time, pass a clock in, or use a library like `time-machine` or `freezegun`.
- Seed randomness (`random.seed(...)`) or inject the generator.

## 12. Async Tests

pytest has no built-in event loop. **Since 8.4, an `async def` test fails** (it used to warn and skip) if no suitable plugin is installed. The usual choice is `pytest-asyncio`:

```python
import asyncio
import pytest
import pytest_asyncio

@pytest_asyncio.fixture
async def client():
    c = await make_client()
    yield c
    await c.close()

@pytest.mark.asyncio
async def test_fetch(client):
    result = await client.fetch("x")
    assert result == "x"
```

- The default mode is **strict**: tests need `@pytest.mark.asyncio` and async fixtures need `@pytest_asyncio.fixture`.
- **Auto mode** (`asyncio_mode = auto`, or `--asyncio-mode=auto`) handles every async test and fixture without markers; it's convenient but can conflict with other async plugins.
- Since pytest-asyncio 1.0 the old `event_loop` fixture is gone, and the marker takes `loop_scope=` (e.g. `@pytest.mark.asyncio(loop_scope="module")`) to share a loop across tests.
- `anyio` ships its own pytest plugin if you use `anyio` or `trio`.

---

## 13. Plugins You'll Actually Use

| Plugin | Why |
|---|---|
| `pytest-cov` | Coverage: `pytest --cov=shop --cov-report=term-missing --cov-fail-under=90 --cov-branch` |
| `pytest-xdist` | Parallel runs: `pytest -n auto` |
| `pytest-mock` | The `mocker` fixture (section 11) |
| `pytest-asyncio` | Async tests (section 12) |
| `pytest-timeout` | Fail hung tests: `--timeout=30` or `@pytest.mark.timeout(5)` |
| `pytest-randomly` | Shuffles test order and reseeds `random`, exposing hidden order dependence |
| `hypothesis` | Property-based testing |
| `pytest-django`, `pytest-flask` | Framework integration (e.g. `@pytest.mark.django_db`) |
| `pytest-benchmark` | A `benchmark` fixture for timing code |
| `pytest-rerunfailures` | `--reruns N`. Use it to quarantine known-flaky tests, not as a cure |

### Running in parallel with `pytest-xdist`

```bash
pytest -n auto                      # one worker per CPU
pytest -n 4 --dist loadscope        # keep tests of one module/class on the same worker
```
Tests must not share mutable external state (the same file, port, or database row). A `session`-scoped fixture is created **once per worker**, not once overall. Unique resources per worker can use the `worker_id` fixture (`"gw0"`, `"gw1"`, ...; `"master"` when not running under xdist).

### Property-based testing with Hypothesis

```python
from hypothesis import given, strategies as st

@given(st.integers(), st.integers())
def test_addition_is_commutative(a, b):
    assert a + b == b + a
```
Hypothesis generates many inputs, then shrinks a failing one to a minimal example, which is excellent for parsers, serializers, and anything with a round-trip property.

## 14. Customizing pytest with Hooks and `conftest.py`

pytest is built on **hooks**: functions named `pytest_*` that it calls at defined points. Put them in `conftest.py` (or in a plugin).

### Adding a command-line option and a slow-test switch

```python
# conftest.py
import pytest

def pytest_addoption(parser):
    parser.addoption("--runslow", action="store_true", default=False, help="run slow tests")

def pytest_configure(config):
    config.addinivalue_line("markers", "slow: mark test as slow to run")

def pytest_collection_modifyitems(config, items):
    if config.getoption("--runslow"):
        return
    skip_slow = pytest.mark.skip(reason="need --runslow option to run")
    for item in items:
        if "slow" in item.keywords:
            item.add_marker(skip_slow)
```
Define `pytest_addoption` in the **top-level** `conftest.py` (the one at the rootdir, or next to the tests you pass on the command line). Conftest files are loaded after the built-in and third-party plugins have initialized, so their hooks can't take part in other plugins' `pytest_addoption` calls; only initial conftests can add command-line options.

A fixture can read the option through `request.config.getoption("--runslow")` or the `pytestconfig` fixture.

### Reading a test's outcome from a fixture

```python
# conftest.py
import pytest

phase_report_key = pytest.StashKey[dict]()

@pytest.hookimpl(wrapper=True, tryfirst=True)
def pytest_runtest_makereport(item, call):
    report = yield                                   # let the other hooks build the report
    item.stash.setdefault(phase_report_key, {})[report.when] = report
    return report                                    # a wrapper must return the (possibly modified) result

@pytest.fixture
def screenshot_on_failure(request):
    yield
    report = request.node.stash[phase_report_key]
    if "call" in report and report["call"].failed:
        save_screenshot(request.node.name)
```
This uses the **new-style hook wrapper** (`wrapper=True`), which needs pluggy 1.2 or newer; recent pytest versions require that anyway. The older `hookwrapper=True` style, which uses `outcome = yield; outcome.get_result()`, still works.

### Other useful hooks

| Hook | Use |
|---|---|
| `pytest_generate_tests(metafunc)` | Parametrize dynamically |
| `pytest_collection_modifyitems(config, items)` | Reorder, deselect, or mark collected tests |
| `pytest_report_header(config)` | Add lines to the run header |
| `pytest_assertrepr_compare(config, op, left, right)` | Customize assertion failure text for your own types |
| `pytest_sessionstart` / `pytest_sessionfinish` | Run code once at the beginning/end of a run |
| `pytest_runtest_setup` / `call` / `teardown` | Per-phase hooks |

**Hooks configured with markers** (the old `@pytest.mark.tryfirst` style, deprecated since 7.2) are scheduled for removal in pytest 10. Use `@pytest.hookimpl(tryfirst=True)`.

### Writing a reusable plugin

Package a module of hooks and fixtures and expose it through an entry point:

```toml
# pyproject.toml of the plugin package
[project.entry-points.pytest11]
myplugin = "mypackage.pytest_plugin"
```
pytest loads every `pytest11` entry point automatically on startup. For declaring fixtures imperatively (rare, for plugin authors), 9.1 adds `pytest.register_fixture()`. The `pytester` fixture, enabled with `pytest_plugins = ["pytester"]` in your top-level `conftest.py`, runs pytest inside a test so you can test plugins end to end.

## 15. Working With `unittest` and Legacy Style

pytest runs `unittest.TestCase` classes unchanged, including `setUp`/`tearDown`, so you can adopt it incrementally.

```python
import unittest, pytest

@pytest.mark.usefixtures("clean_environment")     # fixtures can't be method arguments in a TestCase
class TestLegacy(unittest.TestCase):
    def test_something(self):
        self.assertEqual(1 + 1, 2)
```

| | `unittest` | pytest |
|---|---|---|
| Assertions | `self.assertEqual(a, b)` and friends | plain `assert` with rewriting |
| Setup | `setUp` / `tearDown` methods | fixtures (composable, scoped) |
| Test classes | must subclass `TestCase` | plain functions (classes optional) |
| Parametrization | loops or `subTest` | `@pytest.mark.parametrize`, subtests (9.0+) |
| Sharing setup | inheritance | `conftest.py` and fixture overriding |
| Extensions | few | a large plugin ecosystem |

xunit-style methods (`setup_method`, `teardown_method`, `setup_class`, `setup_module`) still work, but fixtures are more flexible. nose-style `setup()`/`teardown()` support was removed in pytest 8.0.

## 16. Debugging and Reading Output

### Controlling output

| Flag | Effect |
|---|---|
| `-v`, `-vv`, `-q` | More or less detail (`-vv` shows full diffs) |
| `-ra` | Show a short summary of **all non-passing** outcomes at the end |
| `-rA` | Summary of **all** outcomes, including passes |
| `-rfE` | Only failures and errors (letters: `f` failed, `E` error, `s` skipped, `x` xfailed, `X` xpassed, `p` passed) |
| `--tb=short` / `long` / `line` / `no` / `native` | Traceback style |
| `-l` | Show local variables in tracebacks |
| `-s` | Don't capture output |
| `--durations=10` | List the 10 slowest tests |
| `--log-cli-level=INFO` | Stream logs live |
| `--junitxml=report.xml` | CI-friendly XML report |
| `--max-warnings N` (9.1+) | Fail the run if more than N warnings occur |

A line like `FAILED tests/test_pricing.py::test_apply_discount[100-25-70] - assert 75.0 == 70` in the summary is a complete pointer: paste the node ID after `pytest` to rerun just that case.

### Interactive debugging

```bash
pytest --pdb                  # drop into pdb at the first failure
pytest -x --pdb               # stop at the first failure and debug it
pytest --trace                # start pdb at the beginning of each test
pytest --lf -x --pdb          # rerun only last time's failures, debugging the first
```
A plain `breakpoint()` call inside a test also works.

### Finding out why something wasn't collected or ran oddly

```bash
pytest --collect-only -q      # list collected node IDs
pytest --setup-show           # see fixtures being created/destroyed
pytest -p no:cacheprovider    # disable a plugin (-p no:NAME) to rule it out
pytest --version -v           # shows loaded plugins
```
Environment variables: `PYTEST_ADDOPTS` (extra flags), `PYTEST_PLUGINS` (extra plugin modules), `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` (don't load installed plugins; handy for isolating a plugin conflict), and `PYTEST_CURRENT_TEST` (set while a test runs, useful for hang diagnosis).

## 17. Running pytest Well in CI

- **Pin pytest and key plugins, but keep them moving.** A new pytest minor release can change behavior and add deprecation warnings (9.1 added several), yet old versions stop getting fixes: an advisory about pytest's temporary-directory handling was addressed in 9.0.3 (April 2026).
- **Fail on warnings** (`filterwarnings = ["error"]`, or `--max-warnings`) so deprecations are fixed while they are cheap. Whitelist third-party noise specifically.
- Use **`--strict-markers --strict-config`** (or `strict = true` with a pinned version) so typos in markers and config fail loudly.
- **Check the exit code**, and treat 5 (no tests collected) as a failure.
- **Collect coverage** with `pytest-cov`; enforce a floor with `--cov-fail-under`.
- **Parallelize** with `pytest-xdist` once the suite takes more than a minute, after making sure tests are isolated.
- **Keep the suite deterministic:** no sleeping to wait for things, no dependence on test order, no real network. Run with `pytest-randomly` occasionally to catch order coupling.
- **Publish `--junitxml`** so CI can show per-test results and trends.
- **Slow tests:** mark them (`@pytest.mark.slow`) and run the fast set on every commit and the full set on merge or nightly.

## 18. Common Pitfalls

| Symptom | Likely cause and fix |
|---|---|
| A test never runs | Its name doesn't start with `test`, or the file doesn't match `test_*.py` / `*_test.py`. Check with `--collect-only` |
| `fixture 'x' not found` | The fixture isn't in the test's module or a parent `conftest.py`, or the name is misspelled |
| `ScopeMismatch` | A wider-scoped fixture requests a narrower-scoped one. Widen the dependency's scope or narrow yours |
| Tests pass alone, fail together | Shared mutable state in a wide-scoped fixture, module-level globals, or test-order dependence |
| `import file mismatch` | Two test files with the same base name and no `__init__.py` (default `prepend` import mode) |
| Patch has no effect | You patched where the name is defined, not where it's imported (section 11, rule 1) |
| `assert (a, "msg")` always passes | Parentheses made a non-empty tuple. Write `assert a, "msg"` |
| `pytest.raises` passes unexpectedly | The `with` block contains more than the failing call, so unrelated code raised |
| Output missing from a passing test | Output is captured; use `-s` or `-rP` |
| Test returns a value and fails (8.4+) | Replace `return cond` with `assert cond` |
| Class tests lose state between methods | A new instance is created per test method; use fixtures |
| Parametrized tests vanish on a second run | argvalues was a generator or iterator; use a list (deprecated in 9.1) |
| Helper module asserts show no detail | Not rewritten; `pytest.register_assert_rewrite("module")` before import |
| `autouse` fixture has surprising effects | Autouse is invisible. Scope it narrowly or use `usefixtures` |

## 19. Interview Questions and Answers

**What is a fixture, and why use one instead of `setUp`?**
A function whose result is injected into tests by parameter name. Unlike `setUp`, fixtures are composable (they can request other fixtures), scoped (function through session), requested only by tests that need them, and cleaned up with `yield`.

**How does pytest find tests?**
It collects `test_*.py` / `*_test.py` files, `test*` functions, and `Test*` classes without `__init__`, starting from `testpaths` or the arguments. Naming rules are configurable (`python_files`, `python_classes`, `python_functions`).

**Explain fixture scopes and the ScopeMismatch error.**
Scopes are `function`, `class`, `module`, `package`, `session`. A fixture lives as long as its scope. A wide-scoped fixture can't depend on a narrow one because the dependency would be destroyed while it was still in use.

**`parametrize` vs a parametrized fixture?**
Use `parametrize` when the *data* varies for a single test. Use fixture `params` when a *dependency* varies and many tests should run against each variant.

**How do you share fixtures across files?**
Put them in a `conftest.py`; it needs no import and applies to its directory and everything below. Closer definitions override farther ones.

**How does assertion introspection work?**
pytest rewrites the assertion statements in test modules, conftests and plugins at import time (through an import hook), recording sub-expression values so a failure can show them. Plain helper modules aren't rewritten unless registered.

**`monkeypatch` vs `unittest.mock.patch`?**
`monkeypatch` is a fixture for simple replacement of attributes, env vars, and dict items, undone automatically. `mock.patch` (or `mocker`) additionally gives call recording and assertions, with `autospec` for signature checking. Either way, patch where the name is looked up.

**How do you test that code raises an exception, and check the message?**
`with pytest.raises(ExcType, match="regex"):`, where `match` uses `re.search` on `str(exc)`. Use `as excinfo` for the type, value and traceback.

**`skip` vs `xfail`?**
`skip` doesn't run the test. `xfail` runs it and expects a failure: failure becomes *xfail* (not an error), and an unexpected pass becomes *xpass*, which fails the run if `strict=True`.

**How do you run only some tests?**
By node ID (`file::Class::test[param]`), by name (`-k`), by marker (`-m`), by last result (`--lf`, `--ff`), or by config (`testpaths`).

**How do you handle flaky tests?**
Find the cause (shared state, time, ordering, network); isolate with fixtures and fakes; reproduce with `pytest-randomly` and a fixed seed; as a stopgap, mark known-flaky tests and use `pytest-rerunfailures`, tracking them as debt.

**How do you speed up a slow suite?**
Profile with `--durations`, widen the scope of expensive-but-safe fixtures, avoid real I/O, parallelize with `pytest-xdist`, and split slow tests behind a marker.

**What changed recently in pytest?**
8.4: tests returning non-`None` fail, `RaisesGroup`, async tests fail without a plugin. 9.0: built-in subtests, native TOML config, strict mode, Python 3.9 dropped. 9.1: deprecations for generator argvalues, instance-method class fixtures, `getfixturevalue` during teardown; new `--max-warnings` and `pytest.register_fixture`.

## 20. Version Notes

| Version | Notable changes (8.4, 9.0 and 9.1 are from the official release notes; 8.1 and 8.2 are from a distribution changelog summary, so check the official changelog if you depend on them) |
|---|---|
| 8.1 | Namespace-package support; improved `--import-mode=importlib` |
| 8.2 | Arguments from a file with `@filename`; `PYTEST_VERSION` environment variable |
| 8.4 | Tests returning non-`None` now **fail**; `yield` in a test is an error; async tests fail without a plugin; `pytest.RaisesGroup`/`RaisesExc`; `collect_imported_tests` option; Python 3.8 dropped |
| 9.0 | Built-in **subtests**; native TOML config (`[tool.pytest]`, `pytest.toml`); **strict mode**; `strict_parametrization_ids`; warning when config is split across files; Python 3.9 dropped |
| 9.1 | Deprecations: generators/iterators as parametrize argvalues, class-scoped fixtures as instance methods, `request.getfixturevalue()` during teardown, `pytest.console_main`, `--pastebin`; `pytest.register_fixture()`; `--max-warnings`; `assertion_text_diff_style`; `approx` for `datetime`/`timedelta`; `caplog` captures non-propagating loggers; fixture-override resolution by visibility (mainly affects plugins); Python 3.15 support |

Deprecated items are scheduled for removal in **pytest 10**. Run your suite with warnings enabled to find them early.

## Quick Reference

**Most-used commands**
```bash
pytest -x --lf -vv -ra --tb=short        # re-run last failures, verbose, stop at first failure
pytest -k "cart and not slow" -m "not integration"
pytest --collect-only -q                  # what would run?
pytest --setup-show --fixtures-per-test   # understand fixtures
pytest -n auto --cov=shop --cov-report=term-missing
```

**Mental model**
- Collection builds items (parametrization happens here); each item then runs setup → call → teardown.
- Fixtures are injected by **name**, resolved from the test outward through class, module, `conftest.py` files, then plugins.
- Scope controls lifetime; teardown is the reverse of setup.
- Patch where a name is **used**; prefer injected dependencies and fakes.
- Register markers, enable strictness, pin versions, and check the exit code in CI.

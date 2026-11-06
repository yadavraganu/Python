### What Is a Context Manager?
A **context manager** is a construct that handles the setup and teardown of resources automatically. It's most commonly used with the `with` statement.
```python
with open('file.txt', 'r') as f:
    data = f.read()
# File is automatically closed here
```
This ensures the file is closed even if an error occurs during reading.

### Why Use Context Managers?

- **Automatic cleanup**: Frees resources like files, sockets, or DB connections.
- **Exception-safe**: Cleanup happens even if an error is raised.
- **Cleaner code**: Replaces verbose `try-finally` blocks.
- **Custom control**: You can manage any resource using `__enter__` and `__exit__` methods.

### Real-World Use Cases

- **File handling**: `open()`
- **Database connections**: `psycopg2.connect()`
- **Thread locks**: `threading.Lock()`
- **Web scraping**: Closing browser sessions
- **Socket programming**: Managing open sockets
- **Subprocesses**: Cleaning up child processes

### The Protocol, Precisely

A context manager is any object implementing two methods:
- **`__enter__(self)`** — runs at the start of the `with` block; its return value is what `as` binds.
- **`__exit__(self, exc_type, exc_value, traceback)`** — always runs when the block ends, whether it ended normally or via an exception. If an exception occurred, the three arguments describe it; otherwise all three are `None`.

The return value of `__exit__` decides what happens to an exception:
- **Truthy** → the exception is **suppressed**; execution continues after the `with` block as if nothing happened.
- **Falsy (including no explicit `return`, i.e. `None`)** → the exception **propagates** normally.

<br></br>

### Custom Context Manager with Exception Handling
```python
class SafeDivision:
    def __init__(self, a, b):
        self.a = a
        self.b = b

    def __enter__(self):
        print("Entering context: preparing to divide")
        return self

    def divide(self):
        print("➗ Performing division")
        return self.a / self.b

    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type:
            print(f"Exception occurred: {exc_value}")
        print("Exiting context: cleaning up")
        # Suppress exception if handled
        return True  # Change to False to propagate exception
```
### Usage with Explanation

#### Case 1: No Exception
```python
with SafeDivision(10, 2) as sd:
    result = sd.divide()
    print(f"Result: {result}")
```
**Execution Flow:**
1. `__init__` is called → sets `a = 10`, `b = 2`
2. `__enter__` is called → prints "Entering context"
3. `divide()` is called → prints "Performing division"
4. `__exit__` is called → prints "Exiting context"

#### Case 2: With Exception (division by zero)
```python
with SafeDivision(10, 0) as sd:
    result = sd.divide()
    print(f"Result: {result}")
```
**Execution Flow:**
1. `__init__` is called → sets `a = 10`, `b = 0`
2. `__enter__` is called → prints "Entering context"
3. `divide()` raises `ZeroDivisionError`
4. `__exit__` is called → catches exception, prints it
5. Exception is **suppressed** because `__exit__` returns `True`

### The Danger of Unconditionally Returning `True`

`SafeDivision.__exit__` returns `True` **no matter what exception occurred** — not just a `ZeroDivisionError`. That means it silently swallows *any* bug raised inside the block:

```python
with SafeDivision(10, 2) as sd:
    result = 10 / 0            # a real bug, unrelated to "safe division"
    print(undefined_name)       # NameError — also silently swallowed!
print("Program continues as if nothing happened")   # this really does print
```
Both errors vanish without a trace. This is a genuine footgun, not just a style nitpick — a context manager that suppresses exceptions should **check `exc_type`** and only suppress the specific error(s) it knows how to handle, re-raising (or returning falsy for) everything else:

```python
def __exit__(self, exc_type, exc_value, traceback):
    if exc_type is ZeroDivisionError:
        print(f"Handled: {exc_value}")
        return True       # only THIS specific error is suppressed
    return False           # everything else propagates normally
```

### Summary of Method Calls

| Method         | When It's Called                          |
|----------------|--------------------------------------------|
| `__init__`     | When the context manager object is created |
| `__enter__`    | At the start of the `with` block           |
| `divide()`     | Inside the `with` block                    |
| `__exit__`     | At the end of the `with` block or on error |

<br></br>

## Combining Multiple Context Managers

A single `with` statement can manage several context managers at once — either comma-separated (works in all versions) or, since Python 3.10, wrapped in parentheses for easier multi-line formatting:

```python
with open("in.txt") as f_in, open("out.txt", "w") as f_out:
    f_out.write(f_in.read())

# Python 3.10+ — same thing, parenthesized
with (
    open("in.txt") as f_in,
    open("out.txt", "w") as f_out,
):
    f_out.write(f_in.read())
```

**Order matters, and it's a stack:** managers are **entered left to right**, and **exited in reverse (LIFO) order** — confirmed by running it:
```
enter A
enter B
inside block
exit B
exit A
```

**Exception visibility across managers:** if the block raises, `__exit__` is called on each manager in reverse order, and the *first* one that returns truthy stops the exception from reaching the ones further out. In other words, an inner manager can suppress an exception before an outer manager's `__exit__` even sees it:

```python
class CM:
    def __init__(self, name, suppress=False):
        self.name, self.suppress = name, suppress

    def __enter__(self):
        print(f"enter {self.name}")
        return self

    def __exit__(self, exc_type, exc_value, tb):
        print(f"exit {self.name}, exc={exc_type.__name__ if exc_type else None}")
        return self.suppress

with CM("outer"), CM("inner", suppress=True):
    raise ValueError("boom")
# exit inner, exc=ValueError   <- inner sees it and suppresses
# exit outer, exc=None          <- outer's __exit__ runs, but sees NOTHING happened
```

<br></br>

## How `@contextmanager` Works

`contextlib.contextmanager` turns a generator function into a context manager by splitting it at `yield`:
- **Before `yield`** — setup code (opening a file, acquiring a lock).
- **After `yield`** — teardown code (closing the file, releasing the lock).

### Example: Logging Context Manager
```python
from contextlib import contextmanager

@contextmanager
def log_context(name):
    print(f"Entering: {name}")
    try:
        yield
    except Exception as e:
        print(f"Exception in {name}: {e}")
        raise  # Optional: re-raise if you want the error to propagate
    finally:
        print(f"Exiting: {name}")
```
Usage:
```python
with log_context("Test Block"):
    print("Doing work")
    # raise ValueError("Oops!")  # Uncomment to test exception handling
```

### Execution Flow

| Line | When It Runs |
|------|--------------|
| `print("Entering")` | Immediately when `with` starts |
| `yield`             | Pauses to run the block inside `with` |
| `except` block      | Runs if an exception occurs inside the block |
| `finally` block     | Always runs when the block ends (even on error) |

### Key Points

- You must `yield` exactly once.
- The value yielded is returned to the `with` block (e.g., a file object).
- If an exception occurs inside the `with` block, it's **thrown into the generator at the `yield` point** — this is the generator `.throw()` mechanism from the Iterators & Generators notes, which is *why* wrapping `yield` in `try/except`/`finally` works at all here.
- You can suppress or log exceptions inside the `except` block; to suppress, the generator must **not re-raise** (swallow it instead of calling `raise`).

<br></br>

### `@contextmanager` Objects Are Single-Use, Not Reentrant

A generator-based context manager can only be safely used **once**. Reusing the same instance for a second `with` block fails, because the underlying generator has already been exhausted:

```python
c = log_context("demo")
with c:
    print("first use")
with c:                       # reusing the SAME object
    print("second use")
```
This raises an error on the second `__enter__` (the exact exception is a CPython implementation detail of `_GeneratorContextManager`'s internal bookkeeping, currently an `AttributeError`, not a documented, stable error type — don't pattern-match on it). **The fix is simple: call the decorated function again** to get a *fresh* context manager each time, rather than storing and reusing one instance:
```python
with log_context("demo"):
    print("first use")
with log_context("demo"):     # a brand-new generator, works fine
    print("second use")
```
A **class-based** context manager (`__enter__`/`__exit__` written by hand) doesn't have this restriction automatically, but is still typically written for single use unless you deliberately design it to reset its state in `__enter__`.

<br></br>

### Using a `@contextmanager` Function as a Decorator

Since Python 3.2, a function decorated with `@contextmanager` can also be used directly as a **function decorator**, with no `with` statement needed — each call gets its own fresh generator, so this doesn't hit the reentrancy issue above:

```python
from contextlib import contextmanager

@contextmanager
def timed(label):
    print(f"start {label}")
    yield
    print(f"end {label}")

@timed("my_func")
def my_func():
    print("running my_func")

my_func()   # start my_func / running my_func / end my_func
my_func()    # works again — a new generator is created per call
```

## What If `__exit__` Itself Raises?

If cleanup code in `__exit__` raises a **new** exception, that new exception replaces the original — but Python automatically chains them via `__context__`, so the original isn't lost, just no longer the "active" one:

```python
class Bad:
    def __enter__(self): return self
    def __exit__(self, exc_type, exc_value, tb):
        raise RuntimeError("cleanup failed")

try:
    with Bad():
        raise ValueError("original error")
except RuntimeError as e:
    print(e)                 # cleanup failed
    print(e.__context__)      # original error — still reachable, and shown in the traceback as "During handling of the above exception..."
```

<br></br>

## Useful `contextlib` Helpers

Beyond `@contextmanager`, the standard library ships several ready-made context managers worth knowing:

- **`contextlib.suppress(*exceptions)`** — a cleaner alternative to a `try/except: pass` block for expected, safely-ignorable exceptions:
  ```python
  from contextlib import suppress
  with suppress(FileNotFoundError):
      open("does_not_exist.txt")
  print("continues cleanly")
  ```
- **`contextlib.closing(obj)`** — wraps any object that has a `.close()` method but isn't itself a context manager, guaranteeing `.close()` runs at block exit:
  ```python
  from contextlib import closing
  with closing(some_legacy_connection) as conn:
      conn.query(...)
  # conn.close() is called automatically
  ```
- **`contextlib.ExitStack`** — manages a **dynamic** number of context managers (not known until runtime), all cleaned up in reverse order when the stack exits, even if the list length varies per call:
  ```python
  from contextlib import ExitStack
  filenames = ["a.txt", "b.txt", "c.txt"]
  with ExitStack() as stack:
      files = [stack.enter_context(open(fn)) for fn in filenames]
      # every opened file is closed automatically, in reverse order, on exit
  ```
- **`contextlib.nullcontext(value=None)`** — a no-op context manager, useful for conditionally skipping a real context manager without duplicating code:
  ```python
  from contextlib import nullcontext
  cm = real_lock if need_lock else nullcontext()
  with cm:
      do_work()
  ```
<br></br>

## Async Context Managers

For resources that need `await` during setup/teardown (a network connection, an async DB session), the protocol has async counterparts: `__aenter__`/`__aexit__`, used with `async with`.

```python
class AsyncConnection:
    async def __aenter__(self):
        print("connecting...")
        return self
    async def __aexit__(self, exc_type, exc_value, tb):
        print("disconnecting...")
        return False

async def main():
    async with AsyncConnection():
        print("using the connection")
```

`contextlib.asynccontextmanager` is the async equivalent of `@contextmanager`, for writing one with a generator function instead of a full class:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def acquire_resource():
    print("async setup")
    yield "resource"
    print("async teardown")

async def main():
    async with acquire_resource() as res:
        print("got:", res)
```
All the same rules apply — single `yield`, exceptions thrown in at the `yield` point, single-use per generator instance — just with `await`-capable setup/teardown.

<br></br>

## File Handling with `@contextmanager`

```python
from contextlib import contextmanager

@contextmanager
def open_file(filename, mode):
    print("Opening file")
    f = open(filename, mode)
    try:
        yield f  # This is where the file is used inside the `with` block
    except Exception as e:
        print(f"Error: {e}")
        raise
    finally:
        print("Closing file")
        f.close()
```
### Usage
```python
with open_file("example.txt", "w") as file:
    file.write("Hello, Anurag!")
```
### Execution Flow
1. `open_file()` is called → file is opened
2. `yield f` → control passes to the `with` block
3. After block ends or error occurs → `finally` closes the file

<br></br>

## Database Connection with `@contextmanager`
```python
from contextlib import contextmanager
import sqlite3

@contextmanager
def db_connection(db_name):
    print("🔌 Connecting to database")
    conn = sqlite3.connect(db_name)
    try:
        yield conn  # Use the connection inside the block
        conn.commit()
        print("Committed changes")
    except Exception as e:
        conn.rollback()
        print(f"Rolled back due to: {e}")
        raise
    finally:
        print("Closing connection")
        conn.close()
```
### Usage
```python
with db_connection("test.db") as conn:
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER, name TEXT)")
    cursor.execute("INSERT INTO users VALUES (?, ?)", (1, "Anurag"))
```
### Execution Flow
| Step | What Happens |
|------|--------------|
| `db_connection()` | Opens DB connection |
| `yield conn` | Executes SQL inside `with` block |
| `commit()` | Saves changes if no error |
| `rollback()` | Reverts changes if error occurs |
| `close()` | Always closes connection |

<br></br>

## Notes & Gotchas

- **`__exit__`'s return value is the whole story**: truthy suppresses, falsy/`None` propagates. Interviewers love asking "what happens if `__exit__` returns `True`?"
- **Never suppress unconditionally** — check `exc_type` and only swallow the specific exception(s) you intend to handle; an unconditional `return True` hides unrelated bugs, as demonstrated above.
- **Multiple context managers exit in reverse (LIFO) order**, and an inner one suppressing an exception hides it from outer ones entirely.
- **A `@contextmanager` generator is thrown the exception via `.throw()`** at its `yield` line — this is the same mechanism covered in the Iterators & Generators notes, not a separate special case.
- **`@contextmanager`-based context managers are single-use** — reusing the same instance for a second `with` block fails; call the decorated function again for a fresh one each time.
- **A `@contextmanager` function can also be used as a plain decorator**, since 3.2 — each call still gets a fresh generator, so it doesn't hit the single-use issue.
- **If `__exit__` itself raises, the new exception chains to the original via `__context__`** — the original isn't lost, just no longer the active exception.
- **Know the `contextlib` toolbox**: `suppress` (cleaner than `try/except: pass`), `closing` (for non-CM objects with `.close()`), `ExitStack` (a dynamic/variable number of managers), and `nullcontext` (a conditional no-op).
- **Async resources use `__aenter__`/`__aexit__` and `async with`**, with `asynccontextmanager` as the generator-based shortcut — same rules, just awaitable setup/teardown.

## 1. The Exception Hierarchy

All exceptions in Python are objects, and every exception class inherits — directly or indirectly — from `BaseException`.

```
BaseException
 ├── SystemExit
 ├── KeyboardInterrupt
 ├── GeneratorExit
 └── Exception                 # <- what you should almost always catch/subclass
      ├── ArithmeticError
      │    ├── ZeroDivisionError
      │    └── OverflowError
      ├── LookupError
      │    ├── IndexError
      │    └── KeyError
      ├── ValueError
      ├── TypeError
      ├── AttributeError
      ├── NameError
      │    └── UnboundLocalError
      ├── OSError (aka IOError)
      │    ├── FileNotFoundError
      │    ├── PermissionError
      │    └── TimeoutError
      ├── RuntimeError
      │    ├── RecursionError
      │    └── NotImplementedError
      └── StopIteration
```

#### Why `SystemExit`, `KeyboardInterrupt`, and `GeneratorExit` aren't under `Exception`

These three inherit from `BaseException`, not `Exception`. That way, a broad `except Exception:` won't accidentally catch the user pressing Ctrl+C or a normal `sys.exit()` call.

#### Why a bare `except:` is bad practice

A bare `except:` catches everything, including `BaseException`. That means it also swallows these exit and interrupt signals, so your program can become hard to stop.

#### Other exceptions worth knowing

- `ImportError`, with its subclass `ModuleNotFoundError`
- `AssertionError`
- `EOFError`
- `MemoryError`
- `SyntaxError`
- `StopAsyncIteration`
- `UnicodeError`, a kind of `ValueError`, so `UnicodeDecodeError` is also a `ValueError`
- `json.JSONDecodeError`, also a `ValueError`
- The `Warning` family

#### Two more things to remember

- `IOError` is just another name for `OSError`.
- Since Python 3.8, `asyncio.CancelledError` inherits from `BaseException`. So `except Exception:` won't swallow task cancellation either.

**One-line rule:** catch `Exception` (or something more specific), never a bare `except:`, so Ctrl+C, `sys.exit()`, and task cancellation still work.

## 2. Basic try/except/else/finally

```python
try:
    result = 10 / x
except ZeroDivisionError:
    print("Can't divide by zero")
else:
    # Runs only if the try block raised NOTHING
    print(f"Result: {result}")
finally:
    # Always runs — success, exception, or even a return/break in try
    print("Cleanup runs regardless")
```

- **`except`** — handles a specific error.
- **`else`** — runs only when no exception occurred; keeps the "happy path" separate from error handling and from the code that might raise.
- **`finally`** — always runs, used for guaranteed cleanup (closing files, releasing locks), even if the `try` block returns or re-raises.

#### Catching Multiple / Specific Exceptions

```python
try:
    risky_call()
except (ValueError, TypeError) as e:      # tuple = catch either
    print(f"Bad input: {e}")
except KeyError as e:
    print(f"Missing key: {e}")
except Exception as e:                     # generic fallback — keep it last, and narrow
    print(f"Unexpected: {e}")
```

Order matters: Python checks `except` clauses top to bottom and uses the **first match** — so put more specific exceptions before more general ones (a broad one listed first would shadow the specific ones below it).

## 3. Raising Exceptions

```python
def set_age(age):
    if age < 0:
        raise ValueError(f"age cannot be negative, got {age}")
```

### Re-raising

```python
try:
    do_something()
except ValueError:
    log.warning("bad value, re-raising")
    raise            # re-raises the SAME exception with its original traceback intact
```

### Exception Chaining — `raise ... from ...`
Exception chaining links a new exception to the one that caused it, so the traceback shows both.
#### Three forms

- **Implicit:** raising inside an `except` block automatically sets `__context__`. Traceback says "During handling of the above exception, another exception occurred."
- **Explicit (`raise X from e`):** sets `__cause__`. Traceback says "The above exception was the direct cause of the following exception." Use it to wrap low-level errors in domain errors.
- **Suppressed (`raise X from None`):** hides the original from the traceback (`__suppress_context__ = True`), but `__context__` is still stored.

**Example**
```python
class ConfigError(Exception):
    pass

def load_port(value):
    try:
        return int(value)
    except ValueError as e:
        raise ConfigError(f"Invalid port: {value!r}") from e
```
#### Remember

- Use `from e` when translating errors, so debugging info isn't lost.
- Use `from None` only when the original is pure noise to the caller.
- Wrapping without `from` works but makes it look like a bug in your handler.

### Adding Context Without Changing the Exception — `add_note()`

`add_note()` attaches extra text that shows up in the traceback, which is handy for annotating an error as it passes through a layer and then re-raising it unchanged:

```python
try:
    try:
        int("x")
    except ValueError as e:
        e.add_note("while parsing record 7")
        raise                      # same exception, now carrying the note
except ValueError as e:
    print(e.__notes__)             # ['while parsing record 7']
```

## 4. Custom Exceptions

Define your own exceptions by subclassing `Exception` (never `BaseException` directly, unless you specifically want to bypass broad `except Exception` handlers — rare and usually wrong).

```python
class InsufficientFundsError(Exception):
    """Raised when a withdrawal exceeds the available balance."""
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        super().__init__(
            f"Cannot withdraw {amount}; balance is only {balance}"
        )

def withdraw(balance, amount):
    if amount > balance:
        raise InsufficientFundsError(balance, amount)
    return balance - amount
```

### Gotcha: Custom `__init__` Arguments Break Pickling and Copying

An exception remembers only what you pass to `super().__init__()` (stored in `args`). `InsufficientFundsError` above passes a single formatted message, but its `__init__` needs **two** arguments. When the exception is pickled or copied, Python rebuilds it by calling the class with `args` — one argument — and that fails:

```python
import pickle
e = InsufficientFundsError(10, 50)
print(e.args)                      # ('Cannot withdraw 50; balance is only 10',)
pickle.loads(pickle.dumps(e))      # TypeError: __init__() missing 1 required positional argument: 'amount'
```
`copy.copy(e)` fails the same way, and so does anything that moves exceptions between processes (`multiprocessing`, `ProcessPoolExecutor`): in a worker pool the failure surfaces as a confusing `TypeError` while the parent unpickles the worker's exception. The fix is to keep `args` aligned with the `__init__` signature and build the message in `__str__`:

```python
class InsufficientFundsError(Exception):
    def __init__(self, balance, amount):
        super().__init__(balance, amount)     # args == (balance, amount), so the exception can be rebuilt
        self.balance = balance
        self.amount = amount

    def __str__(self):
        return f"Cannot withdraw {self.amount}; balance is only {self.balance}"
```

Custom exception **hierarchies** are common in larger codebases so callers can catch broadly or narrowly:

```python
class AppError(Exception):          # base for all app-specific errors
    pass

class ValidationError(AppError):
    pass

class NotFoundError(AppError):
    pass

# caller can do: except AppError:  to catch anything from this app
```

## 5. `finally`, Context Managers, and Resource Cleanup

`finally` guarantees cleanup, but the idiomatic tool for resource management is the **context manager** (`with` statement), which calls cleanup automatically via `__exit__` — even on an exception.

```python
# Manual approach
f = open("file.txt")
try:
    data = f.read()
finally:
    f.close()

# Idiomatic approach — same guarantee, less code
with open("file.txt") as f:
    data = f.read()
```

**How `with` interacts with exceptions:** `__exit__(exc_type, exc_value, traceback)` is called even if the block raises. If `__exit__` returns a truthy value, the exception is **suppressed**; otherwise it propagates after cleanup runs.

```python
class Suppressor:
    def __enter__(self): return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        return True   # swallows any exception raised inside the block

with Suppressor():
    raise ValueError("this will be silently suppressed")
```

## 6. EAFP vs. LBYL

Two philosophies for handling conditions that might fail:

- **LBYL** ("Look Before You Leap") — check first, then act:
  ```python
  if key in d:
      value = d[key]
  ```
- **EAFP** ("Easier to Ask Forgiveness than Permission") — try it, handle failure:
  ```python
  try:
      value = d[key]
  except KeyError:
      value = default
  ```

**Python idiom strongly favors EAFP** — it avoids a race condition between the check and the action (relevant for files/threads), and it's often faster when failures are rare, since exceptions have near-zero cost on the success path. This is a frequent "why does Python prefer try/except over if-checks" interview question.

## 7. `assert` Statements

```python
def divide(a, b):
    assert b != 0, "b must not be zero"
    return a / b
```

- Raises `AssertionError` if the condition is falsy.
- **Not for validating user input or production-critical invariants** — `assert` statements are stripped out entirely when Python is run with the `-O` (optimize) flag, so they must never be relied on for security or required business logic.

## 8. Inspecting and Logging Exceptions

```python
import traceback, logging

try:
    risky_call()
except Exception:
    traceback.print_exc()                 # prints full traceback to stderr
    tb_str = traceback.format_exc()        # same, as a string
    logging.exception("risky_call failed")  # logs message + traceback automatically
```

`logging.exception(...)` must be called from inside an `except` block — it automatically attaches the current exception's traceback, and is generally preferred over `print()` for anything beyond a quick script.

## 9. `StopIteration` and Generators

`StopIteration` is a normal, expected exception — it's how iterators signal "no more items," raised internally by `next()` and caught automatically by `for` loops.

```python
it = iter([1, 2])
next(it)         # 1
next(it)         # 2
next(it)         # raises StopIteration
next(it, "done")  # "done" — pass a default to avoid the exception
```

An unhandled `StopIteration` escaping a generator body is converted into a `RuntimeError`, to prevent it from silently terminating an enclosing `for` loop unexpectedly.

## 10. Exception Groups

For situations where multiple unrelated exceptions need to be raised together (e.g., concurrent tasks each failing independently):

```python
try:
    raise ExceptionGroup("multiple failures", [ValueError("bad value"), TypeError("bad type")])
except* ValueError as eg:
    print("caught ValueErrors:", eg.exceptions)
except* TypeError as eg:
    print("caught TypeErrors:", eg.exceptions)
```

`except*` matches exceptions by type within the group without stopping at the first match, unlike a normal `except`.

## Notes & Gotchas

- **Never use a bare `except:`** — it also catches `SystemExit` and `KeyboardInterrupt`. Use `except Exception:` at minimum if you must catch broadly.
- **`except Exception as e:`** — the exception object `e` is deleted automatically at the end of the `except` block (a CPython/PEP 3110 detail); trying to reference it afterward raises `NameError` (`UnboundLocalError`, a subclass, inside a function).
- **`finally` overriding `return`:** a `return` inside `finally` silently swallows any exception or return value from the `try`/`except` — a classic "gotcha" question.
  ```python
  def f():
      try:
          return 1
      finally:
          return 2   # this wins — f() returns 2, and any exception is lost too
  ```
  Python 3.14 emits a `SyntaxWarning` for `return`, `break` or `continue` that exits a `finally` block (PEP 765). Behavior is unchanged, but the pattern is now officially discouraged.
- **Custom exceptions should call `super().__init__(...)`** so the message is captured and `str(exception)` works. If `__init__` takes extra arguments, pass them all through so `args` matches the signature, or the exception can't be pickled or copied (see the gotcha in section 4).
- **`raise` with no argument** only works inside an `except` block (re-raises the currently-handled exception); outside one it raises `RuntimeError: No active exception to reraise`.
- **Exceptions are relatively cheap on the success path** in Python, which is part of why EAFP is idiomatic — unlike some languages where exceptions carry heavy overhead regardless of whether they're thrown.
- **`else` after `try` is easy to forget** but is the right place for code that should only run when nothing went wrong, keeping it out of the `try` block (so it isn't accidentally caught by your own `except`).
- **`str(KeyError("a"))` is `'a'` — with the quotes.** `KeyError` shows the `repr` of its key, unlike most exceptions where `str(e)` is the plain message, so be careful when parsing it.
- **`except*` restrictions:** `return`, `break` and `continue` aren't allowed inside an `except*` block, and one `try` can't mix `except` with `except*` (both are `SyntaxError`s).

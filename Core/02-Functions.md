## 1. The Anatomy of a Function

Every function in Python is an instance of the `function` class. When you define one, Python creates an object in memory and binds it to the name you provided — the `def` statement is really just an assignment.

```python
def greet(name: str) -> str:
    """Returns a greeting string."""
    return f"Hello, {name}!"

# Functions are objects!
print(greet.__doc__)     # Access the docstring
print(greet.__name__)    # Access the function name
print(greet.__annotations__)  # {'name': <class 'str'>, 'return': <class 'str'>}
print(type(greet))       # <class 'function'>
```

### Functions Are First-Class Objects

Because a function is just an object, you can:

- **Assign it to another name:**
  ```python
  say_hi = greet
  say_hi("Anurag")
  ```
- **Store it in a data structure:**
  ```python
  ops = {"greet": greet, "upper": str.upper}
  ```
- **Pass it as an argument** to another function (enables `map`, `filter`, `sorted(key=...)`, callbacks):
  ```python
  sorted(names, key=len)
  ```
- **Return it from another function** — this is exactly what makes closures and decorators possible.
- **Attach arbitrary attributes to it**, since it's a regular object with a `__dict__`:
  ```python
  greet.calls = 0
  ```

### Argument Types

Python offers a lot of flexibility in how you pass data:

* **Positional:** `func(1, 2)`
* **Keyword:** `func(a=1, b=2)`
* **Default Values:** `def func(a=10):`
* **Variadic (`*args`):** Collects extra positional arguments into a tuple.
* **Keyword Variadic (`**kwargs`):** Collects extra keyword arguments into a dictionary.
* **Unpacking at the call site** (the reverse of collecting):
  ```python
  def add(a, b, c): return a + b + c
  nums = [1, 2, 3]
  add(*nums)                 # unpack a list/tuple into positional args
  kwargs = {"a": 1, "b": 2, "c": 3}
  add(**kwargs)               # unpack a dict into keyword args
  ```

### ⚠️ The Mutable Default Argument Gotcha

Default argument values are evaluated **once**, at function-definition time — not on every call. If the default is mutable, every call that doesn't override it **shares the same object**.

```python
def add_item(item, bucket=[]):   # bucket is created ONCE
    bucket.append(item)
    return bucket

print(add_item(1))   # [1]
print(add_item(2))   # [1, 2]  <- surprise! Same list reused.
```

**Fix — use `None` as the sentinel default:**
```python
def add_item(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```
This is one of the most commonly asked "what's wrong with this code" interview questions.

## 2. Advanced Parameter Control

Modern Python (3.8+) lets you strictly enforce how arguments are passed using special delimiters:

```python
def complex_func(pos_only, /, standard, *, kw_only):
    pass

# /  -> Everything before this MUST be positional.
# *  -> Everything after this MUST be keyword-only.
```

**Full parameter ordering rule** (the order Python enforces in a `def` signature):

```python
def f(pos_only, /, pos_or_kw, *args, kw_only, **kwargs):
    pass
```
1. Positional-only (before `/`)
2. Positional-or-keyword (standard)
3. `*args` (extra positional) — or a bare `*` if you want keyword-only params without collecting extras
4. Keyword-only (after `*` or `*args`)
5. `**kwargs` (extra keyword) — always last

**Defaults and ordering:** a positional parameter without a default can't follow one that has a default (`def f(a=1, b)` is a `SyntaxError`), but keyword-only parameters (after `*`) can mix required and defaulted ones freely: `def g(a=1, *, b)` is fine.

**Why use positional-only (`/`)?** It lets a library author rename a parameter later without breaking callers, and it's how many CPython builtins (like `len`) are defined internally.

## 3. How Arguments Are Actually Passed

A classic interview question: *"Is Python pass-by-value or pass-by-reference?"* The accurate answer is **neither** — Python uses **"pass by object reference"** (also called "pass by assignment"):

- The function parameter becomes a new local name bound to the **same object** the caller passed.
- If you **mutate** that object in place (`list.append`, `dict[key] = ...`), the caller sees the change, because there's only one underlying object.
- If you **reassign** the parameter to a new object, that only rebinds the local name — the caller's variable is untouched.

```python
def mutate(lst):
    lst.append(4)          # mutates the shared object — caller sees it

def reassign(lst):
    lst = [9, 9, 9]          # rebinds the LOCAL name only — caller unaffected

L = [1, 2, 3]
mutate(L)
print(L)      # [1, 2, 3, 4]

reassign(L)
print(L)      # [1, 2, 3, 4]  — unchanged
```

## 4. Scoping and the LEGB Rule

When you reference a variable inside a function, Python searches in a specific order:

1. **L**ocal — inside the current function.
2. **E**nclosing — inside any nested "parent" functions (closures).
3. **G**lobal — at the top level of the module.
4. **B**uilt-in — reserved names like `len`, `range`, `print`.

Assigning to a name anywhere in a function body makes Python treat it as local for the *whole* function (see `UnboundLocalError` in `03-Scopes.md`, and how closures use `nonlocal` in `08-Closures.md`) unless declared `global` or `nonlocal`.

## 5. Anonymous Functions — `lambda`

A `lambda` is a restricted, single-expression function with no name and no `return` keyword (the expression's value is returned implicitly).

```python
square = lambda x: x * x
sorted(pairs, key=lambda p: p[1])
```

Use a `lambda` for short throwaway callbacks (`key=`, `sorted`, `map`/`filter`); use `def` when the logic needs a docstring, multiple statements, or a name that helps readability/tracebacks.

## 6. Higher-Order Functions

A higher-order function is a function that either:

- takes another function as an argument, or
- returns a function as a result.

This is one of the most useful ideas in Python because it lets you write generic logic that works with behavior, not just values.

- Example 1: Takes a function as an argument — `map`

```python
numbers = [1, 2, 3, 4]
squares = list(map(lambda x: x * x, numbers))
print(squares)   # [1, 4, 9, 16]
```

`map` applies the function to every item in the iterable.

- Example 2: Takes a function as an argument — `filter`

```python
devices = ["phone", "tablet", "laptop", "watch"]
long_names = list(filter(lambda name: len(name) >= 6, devices))
print(long_names)   # ['tablet', 'laptop']
```

`filter` keeps only the items for which the function returns `True`.

- Example 3: Takes a function as an argument — `sorted(key=...)`

```python
students = [("Anurag", 25), ("Amit", 18), ("Riya", 22)]
ordered = sorted(students, key=lambda student: student[1])
print(ordered)
# [('Amit', 18), ('Riya', 22), ('Anurag', 25)]
```

Here, the function decides how the list should be ordered.

- Example 4: Takes a function as an argument — `reduce`

```python
from functools import reduce

numbers = [1, 2, 3, 4]
total = reduce(lambda acc, x: acc + x, numbers, 0)
print(total)   # 10
```

`reduce` repeatedly applies a function to combine values into one result.

- Example 5: Returns a function

```python
def make_multiplier(factor):
    def multiply(x):
        return x * factor
    return multiply

triple = make_multiplier(3)
print(triple(5))   # 15
```

This is the classic pattern behind closures and decorators: a function creates and returns another function that captures some state.

In practice, list comprehensions are often more readable than `map`/`filter`.

## 7. `functools.partial` and `functools.wraps`

- **`partial`** — pre-fills some arguments of a function, returning a new callable:
  ```python
  from functools import partial
  power_of_2 = partial(pow, 2)   # pow(2, exponent)
  power_of_2(10)                  # 1024
  ```
- **`wraps`** — used inside decorators to preserve the wrapped function's `__name__`, `__doc__`, etc. (without it, introspecting a decorated function shows the wrapper's metadata instead of the original's):
  ```python
  from functools import wraps

  def my_decorator(func):
      @wraps(func)
      def wrapper(*args, **kwargs):
          return func(*args, **kwargs)
      return wrapper
  ```

## 8. Recursion

A function that calls itself, with a **base case** to stop and a **recursive case** that moves toward it.

```python
def factorial(n):
    if n <= 1:          # base case
        return 1
    return n * factorial(n - 1)   # recursive case
```

- Python has no built-in tail-call optimization — deep recursion can raise `RecursionError`.
- Default recursion limit is 1000 (`sys.getrecursionlimit()`), adjustable via `sys.setrecursionlimit()`, but raising it risks a real C stack overflow rather than a clean Python exception.
- Prefer an iterative rewrite or `functools.lru_cache` for recursive solutions with overlapping subproblems (classic example: naive recursive Fibonacci is exponential; memoized is linear).

## 9. Callables Beyond `def`

`callable(obj)` checks whether `obj()` is valid. Things other than plain functions can be callable:

```python
class Adder:
    def __init__(self, n):
        self.n = n
    def __call__(self, x):     # makes instances callable
        return x + self.n

add5 = Adder(5)
add5(10)          # 15
callable(add5)     # True
```

## 10. Functions That Don't Run When Called: Generators and Coroutines

Adding `yield` to a `def` (or writing `async def`) changes what *calling* the function does: the body does **not** run. Instead you get an object that runs the body later.

```python
def gen():
    print("body started")
    yield 1

g = gen()        # nothing printed — the body hasn't started
next(g)          # prints "body started" and returns 1
```
- `def` containing `yield` → a **generator function**; the returned generator is a lazy iterator (see `10-Iterators-Generators.md`).
- `async def` → a **coroutine function**; the returned coroutine object only runs when it is awaited or scheduled, and forgetting `await` is a classic bug (see `20-Async-Programming.md`).
- In both cases the function itself is callable, but the object it returns is not.

## 11. Introspection and the `inspect` Module

For building frameworks or debugging complex systems, `inspect` lets you look "inside" a function object — its signature, source, and live stack frames.

```python
import inspect

def my_func(a, b=5): pass

sig = inspect.signature(my_func)
print(sig)                    # (a, b=5)
print(list(sig.parameters))   # ['a', 'b']
print(sig.parameters)         # mappingproxy(OrderedDict({'a': <Parameter "a">, 'b': <Parameter "b=5">})) — exact repr varies by version

print(inspect.getsource(my_func))    # prints the source code (needs the source file: raises OSError in a bare REPL or `python -c`)
print(inspect.isfunction(my_func))   # True
```

Much of the same information is available directly as function attributes:

```python
print(my_func.__defaults__)             # (5,)  — default values, evaluated once at def time
print(my_func.__code__.co_varnames)     # ('a', 'b')  — parameter and local variable names
print(my_func.__closure__)              # None — or a tuple of cells if the function closes over variables
```

## Notes & Gotchas

- **Mutable default arguments** persist across calls — always default to `None` and create the mutable object inside the body.
- **Python is "pass by object reference,"** not pass-by-value or pass-by-reference — mutation vs. reassignment behave differently, as shown in section 3.
- **A function with no explicit `return` returns `None`.**
- **Multiple return values** are really just one tuple being returned and unpacked: `def f(): return 1, 2` → `a, b = f()`.
- **`*args`/`**kwargs` naming is convention, not syntax** — the `*`/`**` matters, not the names `args`/`kwargs`.
- **Docstrings vs comments:** a docstring (`"""..."""` right after `def`) is stored in `__doc__` and accessible via `help()`; a `#` comment is not.
- **`lambda` can't contain statements** (no `=` assignment statements, no `if`/`for` blocks) — only a single expression. Assignment *expressions* (`:=`) are allowed: `lambda x: (y := x * 2) + 1`.

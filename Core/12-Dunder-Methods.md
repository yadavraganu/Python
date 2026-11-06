"Dunder" = **d**ouble **under**score. These methods aren't called directly in normal code — Python's syntax and built-in functions call them *for* you: `obj + other` calls `obj.__add__(other)`, `len(obj)` calls `obj.__len__()`, `for x in obj` calls `obj.__iter__()`. This is how user-defined classes hook into Python's built-in operators and functions, and it's the same mechanism behind operator overloading, the container types, and the `with`/`for`/`in` statements.

This file focuses on **object lifecycle, representation, comparison, containers, and callables**. Attribute-access dunders (`__getattr__` and friends) get the deepest treatment, in their own section below, since they're the most subtle and most interview-tested group. A few categories are covered in full elsewhere and only summarized here, with a pointer — see the table at the end.

<br></br>

## 1. Object Lifecycle — `__new__`, `__init__`, `__del__`

### `__new__` vs. `__init__`
- **`__new__(cls, ...)`** actually **creates** the instance (it's a `staticmethod`, called before the instance exists) and must **return** it.
- **`__init__(self, ...)`** **initializes** an already-created instance; it returns `None` implicitly.

For ordinary classes you almost never touch `__new__` — `object.__new__` handles it. You override it when:
- **Subclassing an immutable type** (`int`, `str`, `tuple`), since the value must be fixed at creation time, before `__init__` could mutate it:
```python
class PositiveInt(int):
    def __new__(cls, value):
        if value <= 0:
            raise ValueError("must be positive")
        return super().__new__(cls, value)   # int is immutable — set the value here, not in __init__

PositiveInt(5)      # 5
PositiveInt(-1)      # ValueError: must be positive
```
- **Implementing a singleton** or controlling whether a new instance is created at all.
- **Metaclass-free customization of instance creation**, since `__new__` runs before `__init__` and can return an instance of a *different* class entirely (Python only calls `__init__` afterward if the returned object actually is an instance of `cls`).

### `__del__` — The Finalizer
Called when an object is about to be garbage-collected. **Do not rely on it for critical cleanup** (closing files, releasing locks) — its timing is not guaranteed:
- In reference-counted CPython it usually runs promptly when the refcount hits zero, but objects in a **reference cycle** are only collected (and finalized) when the cycle-detecting GC runs — see `16-Garbage-Collection.md`.
- At interpreter shutdown, globals and modules may already be torn down when `__del__` runs, causing confusing `NameError`s inside it.
- Prefer a **context manager** (`__enter__`/`__exit__`, see `13-Context-Manager.md`) for deterministic cleanup instead.

<br></br>

## 2. String Representation — `__repr__`, `__str__`, `__format__`

- **`__repr__`** — the unambiguous, developer-facing representation. Convention: `eval(repr(obj)) == obj` should ideally hold, or at least `repr` should look like valid Python. This is what you see in a REPL and inside containers (`print([obj])` always uses each element's `repr`, never `str`).
- **`__str__`** — the readable, user-facing representation, used by `print(obj)` and `str(obj)`.
- **Fallback rule, confirmed by running it:** `str()` falls back to `__repr__` if `__str__` isn't defined; `repr()` **never** falls back to `__str__` — with neither defined, you get the default `<ClassName object at 0x...>`.

```python
class OnlyRepr:
    def __repr__(self): return "OnlyRepr()"

print(str(OnlyRepr()))    # "OnlyRepr()" — falls back to __repr__

class OnlyStr:
    def __str__(self): return "readable"

print(repr(OnlyStr()))    # <__main__.OnlyStr object at 0x...> — does NOT fall back to __str__
```
**Rule of thumb:** always define `__repr__`; add `__str__` only if a different, friendlier display is actually useful.

`__format__(self, format_spec)` backs `format(obj, spec)` and f-string format specs (`f"{obj:>10}"`), letting a class define its own mini-language for alignment, precision, or units.

<br></br>

## 3. Comparison and Hashing

| Method | Triggered by |
|---|---|
| `__eq__`, `__ne__` | `==`, `!=` (`__ne__` is auto-derived from `__eq__` if you don't define it) |
| `__lt__`, `__le__`, `__gt__`, `__ge__` | `<`, `<=`, `>`, `>=` |
| `__hash__` | `hash(obj)`; required for use as a dict key / set member |

**The `__eq__`/`__hash__` trap, confirmed by running it:** defining `__eq__` without `__hash__` sets `__hash__` to `None` automatically, making instances **unhashable**:
```python
class NoHash:
    def __eq__(self, other): return True

print(NoHash.__hash__)     # None
{NoHash()}                   # TypeError: unhashable type: 'NoHash'
```
If instances need to go in a `set`/dict key, define both, keeping the invariant **equal objects must have equal hashes**. Writing all four ordering methods by hand is repetitive — `functools.total_ordering` (covered in `11-Object-Oriented-Program.md`) derives the rest from `__eq__` plus one of them. Arithmetic operators (`__add__`, reflected forms like `__radd__`, and in-place forms like `__iadd__`) are also covered there in depth, since they fit naturally alongside operator-overloading design, not attribute access.

<br></br>

## 4. Container / Sequence Protocol

| Method | Triggered by |
|---|---|
| `__len__` | `len(obj)` |
| `__getitem__` | `obj[key]` |
| `__setitem__` | `obj[key] = value` |
| `__delitem__` | `del obj[key]` |
| `__contains__` | `x in obj` — **falls back to `__iter__`** (linear scan) if undefined |
| `__iter__`, `__next__` | `for x in obj`, `iter(obj)`, `next(obj)` — full protocol detailed in `10-Iterators-Generators.md` |
| `__reversed__` | `reversed(obj)` |

**`__contains__` fallback, confirmed by running it:** without `__contains__`, Python uses `__iter__` and checks each item in turn:
```python
class IterOnly:
    def __iter__(self):
        return iter([1, 2, 3])

2 in IterOnly()   # True — found via iteration, no __contains__ needed
```

<br></br>

## 5. Truthiness — `__bool__`

Backs `bool(obj)` and any implicit truth test (`if obj:`). **Fallback chain, confirmed by running it:** if `__bool__` is undefined, Python tries `__len__` (zero means falsy); with neither defined, every instance is truthy by default.
```python
class HasLenOnly:
    def __len__(self): return 0

bool(HasLenOnly())   # False — no __bool__, so __len__ is used, and 0 is falsy

class Neither: pass
bool(Neither())        # True — no __bool__ or __len__, so the default applies
```

<br></br>

## 6. Callable Objects — `__call__`

Defining `__call__` lets instances be invoked like functions, with `callable(obj)` returning `True`:
```python
class Adder:
    def __init__(self, n): self.n = n
    def __call__(self, x): return x + self.n

add5 = Adder(5)
add5(10)          # 15
callable(add5)      # True
```
This is the same mechanism behind class-based decorators (see `09-Decorators.md`) and any "function-like" object that needs to carry state.

<br></br>

## 7. Context Manager Protocol — `__enter__`, `__exit__`

Backs the `with` statement. Covered in full depth, including the exception-suppression rules and common pitfalls, in `13-Context-Manager.md` — included here only for completeness of the dunder-method overview.

<br></br>

## 8. Attribute Access — The Full Protocol

These methods control how attributes are read, assigned, and deleted on an object — the most commonly misunderstood dunder category, and a frequent source of "why is this infinitely recursing" bugs.

### `__getattr__(self, name)`
Called **only when normal lookup fails** — the attribute isn't found via the instance's `__dict__`, the class, or any parent class. Useful for dynamic attribute generation or friendlier errors for misspelled names. Must raise `AttributeError` if it can't handle the name (letting the `AttributeError` propagate normally).

### `__setattr__(self, name, value)`
Called **unconditionally** for every attribute assignment (`obj.name = value`). Because it's unconditional, assigning directly to `self.name` inside your own `__setattr__` implementation recurses into itself infinitely — use `object.__setattr__(self, name, value)` to bypass your override and actually store the value.

### `__delattr__(self, name)`
Called on every `del obj.name`. Same infinite-recursion trap as `__setattr__`; bypass with `object.__delattr__(self, name)`.

### `__getattribute__(self, name)`
Called **unconditionally for every attribute access**, before the instance `__dict__`/class/`__getattr__` chain is even consulted — this is the actual entry point for `obj.name`, implementing the full lookup algorithm described in `17-Internals.md` (data descriptor → instance `__dict__` → non-data descriptor/class attribute → `__getattr__`). Overriding it is the riskiest of the four, since calling `self.name` anywhere inside it recurses into itself; always go through `object.__getattribute__(self, name)`.

**Key relationship:** `__getattr__` is only reached as a **fallback from `__getattribute__`** — specifically, when `__getattribute__` (default or overridden) raises `AttributeError`. If you override `__getattribute__` and it never raises `AttributeError` for a missing name, your `__getattr__` will never run at all.

### `__dir__(self)`
Called by `dir(obj)`; must return a list of attribute-name strings. Lets a class customize what shows up in introspection/autocomplete, independent of what actually resolves via `__getattribute__`.

```python
class WithCustomDir:
    def __init__(self):
        self.visible = 1
    def __dir__(self):
        return ["visible", "virtual_helper"]   # advertise a name with no real backing attribute

dir(WithCustomDir())   # includes 'virtual_helper' even though no such attribute is ever set
```

### Full Worked Example — Traced and Verified

```python
class SimpleAttributeDemo:
    def __init__(self, value):
        object.__setattr__(self, 'stored_value', value)
        object.__setattr__(self, 'my_id', id(self))

    def __getattribute__(self, name):
        print(f"--> __getattribute__ called for: '{name}'")
        if name in ['stored_value', 'my_id']:
            return object.__getattribute__(self, name)
        try:
            result = object.__getattribute__(self, name)
            return result
        except AttributeError:
            raise

    def __getattr__(self, name):
        print(f"--> __getattr__ called for: '{name}' (fallback)")
        if name == "dynamic_attr":
            return "Hello from dynamic_attr!"
        else:
            raise AttributeError(f"'{type(self).__name__}' object has no attribute '{name}'")

    def __setattr__(self, name, value):
        print(f"--> __setattr__ called for: '{name}' = '{value}'")
        object.__setattr__(self, name, value)

    def __delattr__(self, name):
        print(f"--> __delattr__ called for: '{name}'")
        if name == 'stored_value':
            raise AttributeError(f"Deletion of '{name}' is not allowed.")
        object.__delattr__(self, name)
```

**Reading an existing attribute:**
```python
my_obj = SimpleAttributeDemo(123)
print(f"Initial stored_value: {my_obj.stored_value}")
```
```
--> __getattribute__ called for: 'stored_value'
Initial stored_value: 123
```

**Setting a new attribute, then reading it back:**
```python
my_obj.my_data = "Hello World"
print(f"Reading my_obj.my_data: {my_obj.my_data}")
```
```
--> __setattr__ called for: 'my_data' = 'Hello World'
--> __getattribute__ called for: 'my_data'
Reading my_obj.my_data: Hello World
```

**Reading a dynamically-handled attribute — `__getattribute__` fails first, THEN `__getattr__` runs:**
```python
print(f"Reading my_obj.dynamic_attr: {my_obj.dynamic_attr}")
```
```
--> __getattribute__ called for: 'dynamic_attr'
--> __getattr__ called for: 'dynamic_attr' (fallback)
Reading my_obj.dynamic_attr: Hello from dynamic_attr!
```

**Reading a truly non-existent attribute:**
```python
try:
    print(my_obj.non_existent_key)
except AttributeError as e:
    print(f"Caught expected error: {e}")
```
```
--> __getattribute__ called for: 'non_existent_key'
--> __getattr__ called for: 'non_existent_key' (fallback)
Caught expected error: 'SimpleAttributeDemo' object has no attribute 'non_existent_key'
```

**Deleting attributes — a protected one is blocked:**
```python
del my_obj.my_data           # allowed
try:
    del my_obj.stored_value   # blocked by this class's own rule
except AttributeError as e:
    print(f"Caught error: {e}")
```
```
--> __delattr__ called for: 'my_data'
--> __delattr__ called for: 'stored_value'
Caught error: Deletion of 'stored_value' is not allowed.
```
Note: `__delattr__` here only *raises* — it doesn't print any extra warning message before doing so, so the only lines produced are the `-->` trace line and the caught error, exactly as shown. (An earlier version of this example implied an extra "WARNING" line was printed; running the code confirms it isn't — the raised `AttributeError`'s message is the only indication, caught and printed by the surrounding `except` block.)

**See also:** descriptors (`__get__`/`__set__`/`__delete__` on a *class*, not an instance) interact with this same lookup chain and take priority over instance `__dict__` for data descriptors — covered in full in `14-Python-Descriptors.md`.

<br></br>

## Quick Reference: Dunder Categories

| Category | Key methods | Triggered by | Covered in depth |
|---|---|---|---|
| Object lifecycle | `__new__`, `__init__`, `__del__` | Instance creation/destruction | This file |
| Representation | `__repr__`, `__str__`, `__format__` | `repr()`, `str()`/`print()`, `format()`/f-strings | This file |
| Comparison | `__eq__`, `__lt__`, ..., `__hash__` | `==`, `<`, ..., `hash()` | This file + `11-Object-Oriented-Program.md` (`total_ordering`) |
| Arithmetic | `__add__`, `__radd__`, `__iadd__`, ... | `+`, reflected `+`, `+=` | `11-Object-Oriented-Program.md` |
| Containers | `__len__`, `__getitem__`, `__contains__`, ... | `len()`, `obj[k]`, `in` | This file |
| Iteration | `__iter__`, `__next__` | `for`, `iter()`, `next()` | `10-Iterators-Generators.md` |
| Truthiness | `__bool__` | `bool()`, `if obj:` | This file |
| Callable | `__call__` | `obj(...)` | This file + `09-Decorators.md` |
| Context manager | `__enter__`, `__exit__` | `with` | `13-Context-Manager.md` |
| Attribute access | `__getattr__`, `__setattr__`, `__delattr__`, `__getattribute__`, `__dir__` | `obj.x`, `obj.x = v`, `del obj.x`, `dir(obj)` | This file (section 8) |
| Descriptors | `__get__`, `__set__`, `__delete__` | Attribute access on a *class* attribute | `14-Python-Descriptors.md` |

<br></br>

## Notes & Gotchas Worth Knowing for Interviews

- **`__new__` creates, `__init__` initializes** — override `__new__` mainly for immutable-type subclassing or controlling whether/what instance gets created at all.
- **Never rely on `__del__` for critical cleanup** — reference cycles delay it until the cycle collector runs, and it may not run in a sensible order (or at all) during interpreter shutdown. Use a context manager instead.
- **`str()` falls back to `__repr__`; `repr()` never falls back to `__str__`** — verified directly above. Always implement `__repr__`.
- **Defining `__eq__` without `__hash__` makes instances unhashable** — `__hash__` is set to `None` automatically, a very common "why can't I put this in a set" bug.
- **`__contains__` falls back to `__iter__`**, and `__bool__` falls back to `__len__` — both are "if the specific one is missing, Python tries a more general protocol" patterns worth recognizing as a theme across dunders.
- **`__getattribute__` runs for every single attribute access; `__getattr__` only runs when that fails** — mixing these two up is the single most common point of confusion in this topic.
- **Inside a custom `__setattr__`/`__delattr__`/`__getattribute__`, always go through `object.__xxx__(self, ...)`** to avoid infinite recursion — this is the most common bug when people first implement these.

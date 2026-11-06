# Python Data Types

Python's data types can be classified into several categories based on their properties and use cases. Everything in Python is an **object** — every type below is a class, and `type(x)` returns that class.

## 1. Numeric Types

Represent numbers and support arithmetic operations.

### Integral Types
| Type | Description |
|------|-------------|
| `int` | Arbitrary-precision integers (no overflow) |
| `bool` | Boolean values (`True`/`False`), subclass of `int` |

### Non-Integral Types
| Type | Description |
|------|-------------|
| `float` | Floating-point numbers (IEEE 754 double precision) |
| `complex` | Complex numbers with real and imaginary parts (`3+4j`) |
| `Decimal` | High-precision decimal arithmetic (`decimal` module) |
| `Fraction` | Rational numbers with exact arithmetic (`fractions` module) |

**Integer Caching:** CPython pre-allocates and reuses small integers from `-5` to `256`. This is a CPython implementation detail (never rely on it), but it explains a classic `is` vs `==` puzzle. Which result you see depends on *how the number was created*, so the reliable way to demonstrate it is with values built at **runtime**:

```python
print(int("256") is int("256"))   # True  — both come from the small-int cache
print(int("257") is int("257"))   # False — two separate int objects
print(int("-5") is int("-5"))     # True  — the cache starts at -5
print(int("-6") is int("-6"))     # False
```

**Why not just write `a = 257; b = 257`?** Two identical literals inside the *same* script or function are usually merged into one shared constant by the compiler, so `a is b` prints `True` even beyond 256. The same two lines typed one at a time in the interactive REPL (each line compiled separately) print `False`. That makes the literal-based version of this demo unreliable, which is exactly why `==` is the right way to compare numbers.

### Numeric Gotchas Worth Knowing

```python
import math

print(0.1 + 0.2 == 0.3)               # False — binary floats can't represent 0.1/0.2 exactly (the sum is 0.30000000000000004)
print(math.isclose(0.1 + 0.2, 0.3))   # True  — compare floats with a tolerance
# For exact decimal arithmetic (money), use Decimal("0.1") + Decimal("0.2") == Decimal("0.3")

nan = float("nan")
print(nan == nan)                     # False — NaN is not equal to anything, even itself
print(nan in [nan])                   # True  — `in` checks identity (`is`) before `==`
print(math.isnan(nan))                # True  — the correct NaN test

print(-7 // 2, -7 % 2, 7 % -2)        # -4 1 -1 — // floors toward -infinity; % takes the sign of the divisor
print(round(2.5), round(3.5))         # 2 4 — round() uses banker's rounding (ties go to the even number)
print(True + True)                    # 2 — bool is an int subclass
```

Two more worth knowing: `int` has arbitrary precision (so no overflow, but memory grows with the value: `sys.getsizeof(1)` is 28 bytes versus 40 for `10**30` on 64-bit CPython 3.12), and since Python 3.11 converting an `int` to or from a `str` with more than 4,300 digits raises `ValueError` as a denial-of-service guard (adjustable with `sys.set_int_max_str_digits`).

## 2. Sequence Types

Ordered collections, accessed by index.

| Type | Mutability | Description |
|------|-----------|-------------|
| `str` | Immutable | Text/Unicode strings |
| `list` | Mutable | Ordered, dynamic collection of items |
| `tuple` | Immutable | Ordered, fixed-size collection |
| `range` | Immutable | Sequence of numbers (memory-efficient, lazy) |

**String interning:** CPython reuses string objects in two situations: identifier-like literals are interned, and identical literals within one compilation unit share a single constant (including simple constant expressions such as `"a" * 100`, which the compiler folds). Strings **built at runtime** are separate objects unless you intern them explicitly:

```python
import sys

n = 100
print(("a" * n) is ("a" * n))                        # False — built at runtime: two objects
print(sys.intern("a" * n) is sys.intern("a" * n))     # True  — interned explicitly
print(("a" * 100) is ("a" * 100))                      # True  — constant-folded at compile time
print(("a" * n) == ("a" * n))                          # True  — value equality is what you should compare
```
See `17-Internals.md` for how interning works.

## 3. Mapping Type

Associates keys with values; fast average-case O(1) lookup by key.

| Type | Mutability | Description |
|------|-----------|-------------|
| `dict` | Mutable | Key-value pairs (hash table), insertion-ordered since 3.7 |

## 4. Set Types

Unordered collections of unique, hashable items. Support mathematical set operations (union, intersection, difference).

| Type | Mutability | Description |
|------|-----------|-------------|
| `set` | Mutable | Unordered collection of unique items |
| `frozenset` | Immutable | Immutable, hashable version of `set` |

## 5. Binary Types

Handle raw binary data.

| Type | Mutability | Description |
|------|-----------|-------------|
| `bytes` | Immutable | Sequence of integers 0–255 |
| `bytearray` | Mutable | Mutable sequence of bytes |
| `memoryview` | — | Zero-copy view over a buffer-protocol object (e.g. `bytes`, `bytearray`, `array`) |

**Text vs. bytes:** `str` is a sequence of Unicode *characters*; `bytes` is a sequence of raw *bytes*. Moving between them requires an explicit encoding:

```python
s = "é"
print(len(s), len(s.encode("utf-8")))           # 1 2 — one character, two bytes in UTF-8
print(s.encode("utf-8"), s.encode("latin-1"))   # b'\xc3\xa9' b'\xe9' — the bytes depend on the encoding
print(b"\xc3\xa9".decode("utf-8"))              # é
# b"\xff".decode("utf-8") raises UnicodeDecodeError (a subclass of ValueError)
```
Always state the encoding explicitly (`open(path, encoding="utf-8")`) rather than relying on the platform default.

## 6. Boolean Type

- `bool` — subclass of `int`; `True == 1`, `False == 0`. Only two instances exist (`True`, `False`), so `is` comparisons on booleans are always safe.

## 7. Callable Types

Objects callable with `()`.

- User-defined functions
- Built-in functions/methods
- Classes (calling a class invokes `__new__`/`__init__`)
- Instances with a `__call__` method
- Generator functions and coroutine functions (`async def`) — but calling one returns a generator/coroutine **object**, and those objects are **not** callable themselves (`callable(gen_obj)` is `False`)
- Lambda functions

## 8. Singleton / Sentinel Types

- `None` — absence of a value; the sole instance of `NoneType`
- `NotImplemented` — returned by a comparison/arithmetic dunder when the operation isn't supported for the given types (signals Python to try the reflected method)
- `Ellipsis` (`...`) — used in slicing, stub files, and type hints (`Callable[..., int]`)

## 9. Enumerations *(often missed but common in interviews)*

`enum` module types provide named, immutable, comparable constants.

| Type | Description |
|------|-------------|
| `Enum` | Base class for enumerations |
| `IntEnum` | Enum members that also behave as `int` |
| `Flag` / `IntFlag` | Support bitwise combination of members |
| `auto()` | Auto-assigns values |

```python
from enum import Enum

class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3

Color.RED is Color.RED   # True — enum members are singletons
```

## 10. Structured/Container Helpers (from `collections`)

Not "types" in the numeric-sequence-mapping sense, but frequently tested alongside data types:

| Type | Description |
|------|-------------|
| `namedtuple` | Tuple subclass with named fields |
| `defaultdict` | `dict` subclass with a default factory for missing keys |
| `OrderedDict` | `dict` subclass with explicit reordering methods (less needed since 3.7) |
| `Counter` | `dict` subclass for counting hashable items |
| `deque` | Double-ended queue, O(1) appends/pops from both ends |

## 11. Mutability Overview

| Category | Mutable | Immutable |
|----------|---------|-----------|
| Sequences | `list`, `bytearray` | `str`, `tuple`, `range`, `bytes` |
| Sets | `set` | `frozenset` |
| Mappings | `dict` | — |
| Numbers | — | `int`, `float`, `complex`, `bool`, `Decimal`, `Fraction` |
| Special | — | `None`, `NotImplemented`, `Ellipsis`, `Enum` members |

## 12. Type Checking & Conversion

```python
type(obj)                        # exact type
isinstance(obj, type_or_tuple)   # flexible — respects inheritance
issubclass(SubCls, BaseCls)      # class-level relationship check
```

**`type()` vs `isinstance()`:** `isinstance()` is preferred in most code because it accounts for subclassing (e.g. `isinstance(True, int)` → `True`), whereas `type(obj) == int` would reject a `bool` or any subclass.

### Common Conversions
`int()`, `float()`, `str()`, `list()`, `tuple()`, `dict()`, `set()`, `frozenset()`, `bool()`, `bytes()`, `bytearray()`, `complex()`

## 13. Key Characteristics by Category

### Ordered vs Unordered
- **Ordered:** `str`, `list`, `tuple`, `range`, `bytes`, `bytearray`, `dict` (3.7+, insertion order — but not "sorted")
- **Unordered:** `set`, `frozenset`

### Subscriptable (support indexing)
- Sequences: `str`, `list`, `tuple`, `range`, `bytes`, `bytearray`
- Mappings: `dict` (by key)
- Sets are **not** subscriptable (no defined order)

### Iterable
All sequences, sets, dicts (iterates over keys), and generators.

### Hashable
Only immutable, hash-stable types are hashable and usable as dict keys / set members: `int`, `float`, `str`, `tuple` (if all elements are hashable), `frozenset`, `bool`, `Enum` members, `None`. Lists, dicts, and sets are **unhashable**.

```python
hash((1, 2, 3))     # OK — tuple of hashables
hash([1, 2, 3])      # TypeError: unhashable type: 'list'
{[1, 2]: "x"}         # TypeError — can't use a list as a dict key
```

## 14. Copying Objects: Alias vs. Shallow vs. Deep

Assignment never copies — it binds another name to the **same object**. Real copying has two levels:

```python
import copy

orig = [[1, 2], [3, 4]]
alias   = orig                  # same object
shallow = copy.copy(orig)       # new outer list, SAME inner lists (also: orig.copy(), orig[:], list(orig))
deep    = copy.deepcopy(orig)   # new outer list AND new inner lists

orig[0].append(99)
print(alias)     # [[1, 2, 99], [3, 4]]  — alias is orig
print(shallow)   # [[1, 2, 99], [3, 4]]  — the inner list is shared, so the change shows up
print(deep)      # [[1, 2], [3, 4]]      — fully independent
```
- A **shallow** copy duplicates only the outer container; the elements are still shared references. For flat containers of immutables (ints, strings) that is enough.
- A **deep** copy recursively copies everything and remembers what it has already copied, so it handles self-referencing structures. It is slower, so use it only when nested mutable data must be independent.
- Classes can customize copying by defining `__copy__` and `__deepcopy__`.

## 15. Notes & Gotchas
- **Duck Typing:** Python cares about what an object *can do* (its methods/protocol), not its declared type.
- **Everything is an object:** functions, classes, and modules all have a `type()`.
- **`is` vs `==`:** `is` checks identity (same object in memory); `==` checks value equality (calls `__eq__`). Never use `is` to compare numbers or strings for value equality except with singletons like `None`.
- **Mutable default arguments:** `def f(x=[]):` — the default list is created once and shared across calls, a classic bug source.
- **Type hints (3.5+):** `def func(x: int) -> str:` — not enforced at runtime, purely for tooling/clarity (`mypy`, IDEs).
- **`NotImplemented` vs `NotImplementedError`:** the former is a value returned from dunder methods; the latter is an exception raised in abstract methods. Frequently confused.

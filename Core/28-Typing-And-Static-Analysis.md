# Type Hints and Static Analysis

Type hints make interfaces easier to understand, enable editor support, and let static analyzers catch many defects before runtime. They do **not** automatically validate values passed to ordinary Python functions.

**Version baseline:** examples target Python 3.12+. Native generic function/class syntax and the `type` statement require Python 3.12+. Features introduced in later releases are marked.

## 1. What annotations do—and do not do

Annotations are metadata. Python itself does not reject an incorrect argument simply because a function is annotated.

```python
def add(left: int, right: int) -> int:
    return left + right

print(add(2, 3))          # 5
print(add("a", "b"))      # 'ab' at runtime; a type checker should flag this call
```

Use annotations to communicate the intended contract. Use runtime validation when untrusted data crosses a boundary (HTTP request, file, database, message queue, CLI).

## 2. Core annotation vocabulary

| Type | Meaning |
|---|---|
| `int`, `str`, `float`, `bool` | Built-in scalar types. |
| `list[str]` | List whose elements are strings. |
| `dict[str, int]` | Mapping from string keys to integer values. |
| `tuple[str, int]` | Fixed-length tuple with positional element types. |
| `tuple[int, ...]` | Tuple of any length containing integers. |
| `set[str]` | Set of strings. |
| `str \| None` | String or `None` (equivalent to `Optional[str]`). |
| `A \| B` | Union of possible types (Python 3.10+). |
| `Any` | Disable static checking for this value; use sparingly. |
| `object` | Any Python object, but only operations valid for every object are allowed. |
| `Never` | A type with no possible values, often used for unreachable code. |
| `NoReturn` | A function that never returns normally (for example, it always raises). |

`Any` and `object` are not interchangeable:
- `Any` opts out of type checking and permits arbitrary operations.
- `object` is safe but restrictive; narrow it with `isinstance()` or another check before using type-specific operations.

```python
from typing import Any

def dynamic(value: Any) -> None:
    value.not_a_real_method()      # static checker allows it because value is Any

def safe(value: object) -> None:
    if isinstance(value, str):
        print(value.upper())
```

## 3. Collection interfaces: type the contract, not just the implementation

Accept the broadest useful interface. A function that only iterates should usually accept `Iterable[T]`, not `list[T]`.

```python
from collections.abc import Iterable, Mapping, Sequence

def total(values: Iterable[int]) -> int:
    return sum(values)

def display_names(users: Sequence[str]) -> None:
    for name in users:
        print(name)

def get_region(settings: Mapping[str, str]) -> str:
    return settings.get("region", "unknown")
```

Common interfaces from `collections.abc`:
- `Iterable[T]`: can be iterated.
- `Iterator[T]`: produces values with `next()`.
- `Sequence[T]`: ordered, sized, indexable collection.
- `Mapping[K, V]`: read-oriented key/value interface.
- `MutableMapping[K, V]`: mapping that also supports mutation.
- `Callable[[A, B], R]`: callable taking arguments and returning `R`.
- `AsyncIterable[T]`, `AsyncIterator[T]`: asynchronous iteration.

Do not annotate every parameter as a concrete `list` or `dict` when a read-only or iterable interface communicates the actual requirement better.

## 4. Union types, optional values, and narrowing

```python
def normalize(value: str | bytes | None) -> str:
    if value is None:
        return ""
    if isinstance(value, bytes):
        return value.decode("utf-8")
    return value.strip()
```

A type checker narrows `value` after the `None` and `bytes` checks. Avoid `Optional[T]` when `T | None` is clearer, although both mean the same thing.

Use `Literal` when only a fixed set of values is valid:

```python
from typing import Literal

Mode = Literal["read", "write"]

def open_mode(mode: Mode) -> None:
    ...
```

`Literal` is primarily a static constraint. Add runtime validation if the value comes from untrusted input.

## 5. Type aliases and native generic syntax

A type alias gives a complex type a meaningful name. Python 3.12 introduced the `type` statement and native type parameters.

```python
type UserId = int
type Row = dict[str, str | int | None]
type Matrix[T] = list[list[T]]

def first[T](values: list[T]) -> T | None:
    return values[0] if values else None
```

Before Python 3.12, use assignment-based aliases and `TypeVar`:

```python
from typing import TypeVar

T = TypeVar("T")
UserId = int
Row = dict[str, str | int | None]

def first(values: list[T]) -> T | None:
    return values[0] if values else None
```

Generic classes can also use native syntax (3.12+):

```python
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()
```

Generics describe relationships between types. For example, `first(list[str])` returns a `str`, while `first(list[int])` returns an `int`.

## 6. `TypedDict` for dictionary-shaped records

A `TypedDict` describes the expected keys and value types of a dictionary. It is useful when a JSON/API payload remains a dictionary rather than becoming a class instance.

```python
from typing import NotRequired, TypedDict

class UserPayload(TypedDict):
    id: int
    name: str
    email: NotRequired[str]

def display_user(user: UserPayload) -> str:
    return f"{user['id']}: {user['name']}"

payload: UserPayload = {"id": 7, "name": "Ana"}
print(display_user(payload))
```

`TypedDict` does not validate a runtime dict. Validate decoded input before treating it as a `UserPayload`.

Useful variations:
- `total=False` makes all keys optional by default.
- `Required[T]` and `NotRequired[T]` override requiredness per key.
- `ReadOnly[T]` (Python 3.13+) expresses a read-only item to static type checkers; it does not freeze a runtime dict.

## 7. `Protocol`: structural typing

A protocol defines the attributes and methods an object must provide, without requiring explicit inheritance. This is often the right abstraction for pluggable implementations and dependency inversion.

```python
from typing import Protocol

class Sender(Protocol):
    def send(self, message: str) -> None: ...

class ConsoleSender:
    def send(self, message: str) -> None:
        print(message)

def notify(sender: Sender, message: str) -> None:
    sender.send(message)

notify(ConsoleSender(), "ready")
```

This is checked by a static type checker. Runtime `isinstance()` checks require `@runtime_checkable` and only verify member presence, not the full method signature. See `15-Protocols.md` for detailed behavior and its comparison with ABCs.

## 8. `Self`, `ClassVar`, `Final`, and `NewType`

```python
from typing import ClassVar, Final, NewType, Self

UserId = NewType("UserId", int)
DEFAULT_TIMEOUT: Final[int] = 10

class Query:
    created: ClassVar[int] = 0

    def where(self, clause: str) -> Self:
        self.clause = clause
        return self

def load_user(user_id: UserId) -> None:
    ...
```

- `Self` (Python 3.11+) means the current enclosing class, useful for fluent methods and class-aware return types.
- `ClassVar` says an annotated attribute belongs to the class, not each instance.
- `Final` tells type checkers that a name should not be reassigned or overridden.
- `NewType` creates a static distinction between otherwise identical underlying types. At runtime, calling the `NewType` function returns the original value; it does not create a wrapper object.

These are static hints, not enforcement mechanisms.

## 9. `ParamSpec` and `TypeVar` for decorators

A decorator that accepts and returns `Callable[..., Any]` loses useful signature information. `ParamSpec` preserves the parameter types.

```python
from collections.abc import Callable
from functools import wraps
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def log_calls(func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_calls
def multiply(left: int, right: int) -> int:
    return left * right
```

The static signature remains `multiply(int, int) -> int`. `ParamSpec` and `Concatenate` are useful when a decorator adds, removes, or preserves parameters. See `09-Decorators.md` for runtime decorator behavior.

## 10. Overloads for APIs with related signatures

`@overload` tells static checkers that several call signatures are supported. The final implementation is what executes at runtime.

```python
from typing import overload

@overload
def lookup(key: int) -> str: ...

@overload
def lookup(key: str) -> int: ...

def lookup(key: int | str) -> str | int:
    if isinstance(key, int):
        return f"user-{key}"
    return len(key)
```

Overloads should describe the real implementation accurately. They do not create multiple runtime dispatch branches by themselves.

## 11. `Annotated` and metadata

`Annotated[T, metadata]` attaches extra metadata while retaining the underlying type `T` for tools that do not interpret the metadata.

```python
from typing import Annotated

Port = Annotated[int, "TCP port from 1 through 65535"]

def start_server(port: Port) -> None:
    ...
```

The string here is documentation metadata only; it does not validate the range. A framework or validation library may interpret metadata, but ordinary Python does not.

## 12. Type narrowing and unreachable code

Use `isinstance`, `is None`, `assert`, and well-designed validation helpers to narrow types. `typing.cast()` tells the checker to treat an expression as a given type but performs **no runtime check**.

```python
from typing import cast

raw: object = "hello"
text = cast(str, raw)    # no conversion or validation occurs
print(text.upper())
```

For exhaustiveness checks, `assert_never()` can help static analyzers identify an unhandled union case:

```python
from typing import Literal, assert_never

State = Literal["new", "done"]

def render(state: State) -> str:
    if state == "new":
        return "Pending"
    if state == "done":
        return "Complete"
    assert_never(state)
```

The exact diagnostics depend on the checker and configuration.

## 13. Type annotations and Python 3.14+

Before Python 3.14, function and class annotations were generally evaluated when the definition executed, unless `from __future__ import annotations` postponed them as strings. Python 3.14 changed the default to deferred annotation evaluation (PEP 649 / PEP 749), with the `annotationlib` module providing explicit inspection formats.

Practical guidance:
- Do not assume that `__annotations__` is always a simple eagerly evaluated dictionary in every supported Python version.
- Use `typing.get_type_hints()` when you need resolved type hints and understand that resolving annotations may evaluate expressions.
- For tooling that must inspect annotations without evaluating them, use the version-appropriate `annotationlib` APIs on Python 3.14+.
- Avoid side effects in annotation expressions. Type annotations should describe the API, not execute application logic.

## 14. Running a static type checker

A practical workflow is:
1. Add annotations at public function and module boundaries first.
2. Run a checker (`mypy` or `pyright`) in CI.
3. Fix real mismatches rather than silencing every warning with `Any` or `cast`.
4. Keep runtime validation for external input.
5. Type-check tests and examples where feasible.

Example mypy configuration:

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
warn_unused_ignores = true
warn_redundant_casts = true
check_untyped_defs = true
no_implicit_optional = true
```

The `python_version` value must match the language baseline supported by the project. Pin the checker and configuration in CI so local and CI results agree.

## 15. Common interview mistakes

- Saying that Python enforces annotations at runtime.
- Using `Any` where `object` would provide safer checking.
- Typing a read-only parameter as `list[T]` when it only needs `Iterable[T]` or `Sequence[T]`.
- Believing `cast()` converts or validates a value.
- Treating `TypedDict` as a runtime validation class.
- Assuming `@overload` dispatches at runtime.
- Assuming a `@runtime_checkable` protocol checks full signatures.
- Using `Final` or `ClassVar` as if they were runtime enforcement.
- Ignoring annotation evaluation changes in Python 3.14+.

## 16. Practice questions

1. What is the difference between `Any` and `object`?
2. Why should a function accept `Iterable[T]` instead of `list[T]` when it only loops?
3. When would you use `TypedDict`, a dataclass, or a `Protocol`?
4. What do generics add beyond annotations using `Any`?
5. How does `ParamSpec` help type decorators?
6. What does `cast()` do at runtime?
7. What is the difference between static checking and runtime validation?
8. What changed about annotation evaluation in Python 3.14?


## 17. Advanced narrowing and variadic typing

### `TypeGuard` and `TypeIs`

Use a type guard when a custom predicate proves that a value has a more specific shape.

```python
from typing import TypeGuard

def is_string_list(values: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(value, str) for value in values)

items: list[object] = ["a", "b"]
if is_string_list(items):
    # A static checker narrows items to list[str] in this branch.
    print(" ".join(items))
```

`TypeGuard` is useful when the narrowed type is not a subtype of the input type, such as narrowing `list[object]` to `list[str]` (mutable lists are invariant).

`TypeIs` (Python 3.13+, or `typing_extensions` on earlier versions) is a related form for predicates that narrow to a subtype of the input type. In addition to narrowing the true branch, a checker can narrow the false branch by excluding the tested type. Use `TypeIs` when that stronger subtype relationship is valid; do not claim that either annotation performs runtime validation by itself.

### `Unpack` with `TypedDict` for keyword arguments

```python
from typing import TypedDict, Unpack

class MovieOptions(TypedDict):
    title: str
    year: int

def create_movie(**options: Unpack[MovieOptions]) -> str:
    return f"{options['title']} ({options['year']})"

assert create_movie(title="Arrival", year=2016) == "Arrival (2016)"
```

This describes the supported keyword names and types to a static checker. Runtime callers can still pass invalid values unless the implementation validates them.

### `TypeVarTuple` for variadic generic shapes

`TypeVarTuple` models a variable number of type parameters, useful when a tuple's length and element types are part of the contract.

```python
from typing import TypeVarTuple

Ts = TypeVarTuple("Ts")

def move_last_to_front(values: tuple[*Ts, int]) -> tuple[int, *Ts]:
    return (values[-1], *values[:-1])

assert move_last_to_front(("name", True, 7)) == (7, "name", True)
```

Use variadic generics only when preserving heterogeneous tuple shapes matters. For ordinary sequences with homogeneous items, `Sequence[T]` or `list[T]` is simpler.

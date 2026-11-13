# Dataclasses: Deep Dive and Interview Guide

`dataclasses` is part of the standard library. It reduces boilerplate for classes whose primary purpose is to store structured data, while still allowing validation, methods, inheritance, and custom behavior.

The decorator reads class annotations to identify fields. It does **not** enforce those annotations at runtime.

## 1. Basic dataclass

```python
from dataclasses import dataclass

@dataclass
class User:
    user_id: int
    name: str
    active: bool = True

user = User(7, "Ana")
print(user)            # User(user_id=7, name='Ana', active=True)
print(user.name)       # Ana
```

By default, `@dataclass` generates:
- `__init__`
- `__repr__`
- `__eq__`

It can generate ordering methods, hashing behavior, frozen assignment behavior, and slots depending on its options.

## 2. Field ordering and defaults

A field without a default cannot follow a field with a default in the generated initializer.

```python
from dataclasses import dataclass

@dataclass
class Product:
    sku: str
    quantity: int = 0
```

The equivalent invalid definition would place a required field after `quantity` and raises a `TypeError` while decorating the class. Reorder the fields or make the later field keyword-only.

## 3. Mutable defaults: use `default_factory`

Do not use a mutable object as a shared default. Dataclasses reject many unhashable defaults, but `default_factory` is the correct explicit pattern.

```python
from dataclasses import dataclass, field

@dataclass
class Cart:
    items: list[str] = field(default_factory=list)
    metadata: dict[str, str] = field(default_factory=dict)

first = Cart()
second = Cart()
first.items.append("book")

print(first.items)     # ['book']
print(second.items)    # []
```

The factory is called once per instance. It should be callable with no arguments.

## 4. `field()` options

```python
from dataclasses import dataclass, field

@dataclass
class ApiClientConfig:
    base_url: str
    timeout: float = 5.0
    token: str = field(default="", repr=False, compare=False)
    headers: dict[str, str] = field(default_factory=dict)
```

Common `field()` arguments:
- `default` / `default_factory`: choose one to provide the field value.
- `init=False`: omit the field from the generated initializer.
- `repr=False`: omit it from the generated representation (useful for secrets).
- `compare=False`: omit it from generated equality and ordering comparisons.
- `hash=False`: exclude it from a generated hash, subject to the dataclass hash rules.
- `kw_only=True`: make the field keyword-only.
- `metadata`: an optional read-only mapping for third-party tools.

Avoid putting secrets into `repr`; also avoid logging the full dataclass unless sensitive fields are excluded.

## 5. Derived fields and validation with `__post_init__`

`__post_init__` runs after the generated initializer assigns fields. Use it for validation and derived state.

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)

    def __post_init__(self) -> None:
        if self.width <= 0 or self.height <= 0:
            raise ValueError("dimensions must be positive")
        self.area = self.width * self.height

rectangle = Rectangle(4, 5)
print(rectangle.area)  # 20
```

If the class defines its own `__init__`, dataclasses do not generate one, and `__post_init__` is not automatically called by a custom initializer.

## 6. `InitVar` for initialization-only values

`InitVar[T]` makes a value an initializer parameter that is passed to `__post_init__` but is not stored as a normal dataclass field.

```python
from dataclasses import InitVar, dataclass, field

@dataclass
class User:
    name: str
    raw_password: InitVar[str]
    password_length: int = field(init=False)

    def __post_init__(self, raw_password: str) -> None:
        if len(raw_password) < 12:
            raise ValueError("password is too short")
        self.password_length = len(raw_password)

user = User("Ana", "correct-horse")
print(user.name, user.password_length)
```

This example demonstrates initialization flow only; do not store plaintext passwords in production. Use a dedicated password-hashing scheme.

## 7. `ClassVar` is not an instance field

```python
from dataclasses import dataclass
from typing import ClassVar

@dataclass
class Job:
    default_queue: ClassVar[str] = "normal"
    job_id: int
    name: str
```

`default_queue` is a class attribute, not included in the generated initializer, `repr`, equality, or `dataclasses.fields()`.

## 8. Frozen, ordered, and keyword-only dataclasses

```python
from dataclasses import dataclass

@dataclass(frozen=True, order=True)
class Coordinate:
    x: int
    y: int

a = Coordinate(1, 2)
b = Coordinate(2, 1)

print(a < b)           # True: ordering follows the field order (x, then y)
# a.x = 10             # FrozenInstanceError
```

- `frozen=True` prevents ordinary field assignment/deletion; it is shallow emulated immutability, not deep immutability.
- `order=True` generates `<`, `<=`, `>`, `>=` using field declaration order. Equality and ordering generally require the same dataclass type.
- `kw_only=True` (Python 3.10+) makes generated initializer parameters keyword-only. This can make APIs more resilient when fields are added.
- `match_args=True` (default) can expose positional fields for structural pattern matching.

Frozen dataclasses can be hashable when the equality/hash configuration permits it. Do not put mutable state inside a frozen object and assume the entire object graph is immutable.

## 9. Equality and hashing

The default dataclass uses field-based equality. With `eq=True` and `frozen=True`, a hash is normally generated. With `eq=True` and `frozen=False`, instances are normally unhashable because mutable objects should not be used as hash keys when their equality-relevant state can change.

You can set `eq=False`, or explicitly control `unsafe_hash`, but do so only after understanding the equality/hash contract:
- Equal objects must have equal hashes.
- A key's hash and equality behavior must remain stable while it is in a dict or set.
- `unsafe_hash=True` does not make mutable fields safe.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class CacheKey:
    tenant_id: int
    resource: str

cache = {CacheKey(1, "orders"): "cached"}
print(cache[CacheKey(1, "orders")])  # cached
```

## 10. Inheritance and field ordering

Dataclasses inherit fields from dataclass base classes. The final field order follows the dataclass field collection rules, and a required non-keyword-only field cannot follow a defaulted field in the generated initializer—even if the fields come from different classes.

A common solution is to make subclass-only parameters keyword-only:

```python
from dataclasses import dataclass, field

@dataclass
class Base:
    name: str
    enabled: bool = True

@dataclass
class Specialized(Base):
    endpoint: str = field(default="", kw_only=True)
```

Design inheritance around a coherent data model; do not use dataclass inheritance merely to share a few unrelated fields.

## 11. Slots and memory

`slots=True` (Python 3.10+) asks dataclasses to create a slotted class. It can reduce per-instance memory and prevent arbitrary attributes, but it has inheritance, weak-reference, and introspection implications.

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    x: float
    y: float

point = Point(1, 2)
print(point.x, point.y)
```

Measure real workloads before using slots as an optimization. See `11-Object-Oriented-Program.md` for general `__slots__` behavior.

## 12. Useful dataclasses APIs

```python
from dataclasses import asdict, dataclass, fields, is_dataclass, replace

@dataclass
class User:
    user_id: int
    name: str

user = User(7, "Ana")

print(is_dataclass(user))                 # True
print([field.name for field in fields(user)])  # ['user_id', 'name']
print(asdict(user))                        # {'user_id': 7, 'name': 'Ana'}

renamed = replace(user, name="Asha")
print(renamed)                             # User(user_id=7, name='Asha')
```

- `fields()` returns field metadata.
- `asdict()` recursively converts dataclasses and deep-copies other objects as needed; it may be expensive for large object graphs.
- `astuple()` is the tuple-oriented counterpart.
- `replace()` creates a new instance with selected field values changed.
- `is_dataclass()` checks a class or instance.
- `make_dataclass()` constructs a dataclass dynamically; ordinary class definitions are usually clearer.

## 13. Dataclass vs other data containers

| Tool | Best fit | Runtime characteristics |
|---|---|---|
| `dict` | Dynamic key/value data | Flexible, no declared schema by default. |
| `TypedDict` | Typed dictionary payloads | Static description; still a regular `dict` at runtime. |
| `NamedTuple` / `namedtuple` | Small immutable tuple-like records | Tuple unpacking/indexing and named fields. |
| `dataclass` | Domain records with fields and behavior | Generated methods, defaults, validation hooks, options. |
| Regular class | Custom lifecycle, invariants, or behavior-heavy objects | Maximum control, more boilerplate. |

## 14. Common mistakes

- Using `items: list[str] = []` rather than `field(default_factory=list)`.
- Assuming annotations perform runtime validation.
- Assuming `frozen=True` deeply freezes nested lists or dicts.
- Using `unsafe_hash=True` to silence an error without ensuring stable hash/equality.
- Assuming `asdict()` is a shallow, zero-copy conversion.
- Forgetting that `ClassVar` is excluded from dataclass fields.
- Using a dataclass for a type with complex lifecycle and invariants where a regular class is clearer.
- Exposing passwords, API keys, or tokens in the generated `repr`.

## 15. Practice questions

1. What does `@dataclass` generate by default?
2. Why does `default_factory` solve the mutable-default problem?
3. What is the difference between `InitVar` and a normal field?
4. Why is a mutable dataclass normally unhashable?
5. What does `frozen=True` guarantee, and what does it not guarantee?
6. When would you use `TypedDict`, `NamedTuple`, or a dataclass?
7. What trade-offs come with `slots=True`?
8. How do dataclass inheritance and keyword-only fields interact?

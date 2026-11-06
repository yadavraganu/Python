# Python Protocols - Structural Subtyping Guide

In Python, a **Protocol** is an explicit way to define an interface or behavioral contract using **structural subtyping** (often called static duck typing). Introduced in Python 3.8 via the `typing.Protocol` class, protocols let you specify the exact methods and attributes an object must have without requiring classes to explicitly inherit from it.

If a class has the members a protocol declares (matching names, with compatible signatures and types), static type checkers such as mypy and Pyright accept it as a valid subtype automatically. The idea is specified in **PEP 544**.

**The duck-typing connection:** Python has always had runtime duck typing ("if it walks like a duck..."), but that gave type checkers nothing to verify ahead of time. Protocols bring the same flexibility to *static* analysis, hence "static duck typing." Protocols are checked by type checkers, not enforced by the interpreter (see the runtime section below).

## Basic Implementation Pattern

To implement a protocol:
1. Define a class that inherits from `typing.Protocol`
2. List the expected methods or attributes using an ellipsis (`...`) as a placeholder
3. Use that protocol as a type hint

### 1. Define the Protocol

```python
from typing import Protocol

class Document(Protocol):
    title: str  # Expected attribute

    def extract_text(self) -> str:
        """Expected method"""
        ...
```
### 2. Create Conforming Classes (No Inheritance Needed)

You do not need to inherit from `Document`. You only need to match its structure.

```python
class PDFBook:
    def __init__(self, title: str, content: str):
        self.title = title
        self.content = content

    def extract_text(self) -> str:
        return f"PDF Content: {self.content}"


class WordFile:
    def __init__(self, title: str, text: str):
        self.title = title
        self.text = text

    def extract_text(self) -> str:
        return f"Word Content: {self.text}"
```

### 3. Use the Protocol as a Type Hint

```python
def print_summary(doc: Document) -> None:
    # Type checkers verify that 'title' and 'extract_text' exist
    print(f"Reading: {doc.title}")
    print(doc.extract_text())

# Both work flawlessly with static type checkers
print_summary(PDFBook("Python Guide", "Hello World"))
print_summary(WordFile("Notes", "Some text"))
```
## Advanced Implementation Rules

### Defining Properties (Read-Only vs. Writable Attributes)

By default, protocol variables are assumed to be writable. If an attribute should only be readable, define it using Python's `@property` decorator.

```python
from typing import Protocol

class UserProfile(Protocol):
    @property
    def username(self) -> str: ...  # Read-only attribute
    
    email: str  # Writable attribute (must support read AND write)
```
### Protocol Methods Can Have Bodies

A protocol method doesn't have to be a stub. It can contain real code, but **only classes that explicitly subclass the protocol inherit it**. A class that merely matches the protocol's shape inherits nothing:

```python
from typing import Protocol

class Greeter(Protocol):
    def name(self) -> str: ...

    def greet(self) -> str:
        return f"Hello, {self.name()}!"        # a concrete default

class Explicit(Greeter):                         # explicit subclass: inherits greet()
    def name(self) -> str:
        return "Ana"

class Structural:                                # matches the shape, no inheritance
    def name(self) -> str:
        return "Ana"

print(Explicit().greet())                        # Hello, Ana!
print(hasattr(Structural(), "greet"))            # False — structural matches get no default bodies
```
(A type checker will also flag `Structural` as *not* matching `Greeter`, since `greet` is missing.) For shared implementation across many classes, an ABC or mixin is usually the clearer tool.

### Generic Protocols

Protocols can be made generic using `typing.Generic` or Python 3.12+'s native generic syntax. This is highly useful for abstracting data structures or repositories.

```python
from typing import Protocol, TypeVar

T = TypeVar('T')

class Repository(Protocol[T]):
    def get_by_id(self, item_id: int) -> T: ...
    def save(self, item: T) -> None: ...
```

On Python 3.12+ the same protocol can be written with the native generic syntax, with no `TypeVar` needed:

```python
class Repository[T](Protocol):
    def get_by_id(self, item_id: int) -> T: ...
    def save(self, item: T) -> None: ...
```
Type checkers also validate **variance** in protocols: a type variable used only in return positions should be declared covariant (`TypeVar("T_co", covariant=True)`), and one used only in parameters contravariant. `T` above appears in both, so it stays invariant.

### Subclassing and Extending Protocols

You can extend an existing protocol to create a more specific interface. A class must satisfy both protocols to conform to the extended version.

```python
from typing import Protocol

class Reader(Protocol):
    def read(self) -> bytes: ...

class ReadWriter(Reader, Protocol):  # Inherits read(), adds write()
    def write(self, data: bytes) -> None: ...
```
## Key Features & Runtime Mechanisms

- **Implicit Compliance**: Classes conform to a protocol simply by having matching methods and attributes.
- **Static Verification**: They are primarily meant for static analysis tools like mypy to catch bugs before code runs.
- **Runtime Type Checking (@runtime_checkable)**: By default, protocols vanish at runtime and cannot be used with `isinstance()`. If you want to use them for runtime checks, decorate the protocol with `@runtime_checkable`.

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Greeter(Protocol):
    def greet(self) -> str: ...

class Human:
    def greet(self) -> str:
        return "Hello!"

obj = Human()
print(isinstance(obj, Greeter))  # Returns True
```
**What `isinstance()` against a runtime-checkable protocol actually checks** (tested on 3.12):

- Only that each member **exists**. It does not check signatures: a class whose `greet` takes two required arguments still passes, and so does one where `greet = 5`.
- A member set to `None` counts as *not supported*, so `greet = None` fails the check.
- `isinstance()` works for protocols with data members, but `issubclass()` raises `TypeError: Protocols with non-method members don't support issubclass()`. It works for method-only protocols.
- Without the decorator, `isinstance()` raises `TypeError: Instance and class checks can only be used with @runtime_checkable protocols`.
- Python 3.12 made these checks use `inspect.getattr_static`, so they no longer trigger properties or `__getattr__` as a side effect, and they run much faster than before.

Because of this, a passing `isinstance` check is weaker than a static check; use it for dispatching on capability, not as a full contract guarantee. (Python 3.13 added `typing.is_protocol()` and `typing.get_protocol_members()` for introspecting protocols.)

## Python's Built-In Protocols

Python uses structural subtyping internally. The `collections.abc` and `typing` modules provide many built-in protocols that you use daily without realizing it:

- **Iterable**: Requires `__iter__`
- **Iterator**: Requires `__next__` and `__iter__`
- **Sized**: Requires `__len__`
- **Container**: Requires `__contains__`
- **ContextManager**: Requires `__enter__` and `__exit__`
- **Callable**: Represents callable objects
- **Hashable**: Objects that can be used as dictionary keys
- **Reversible**: Objects that support `__reversed__`

```python
from collections.abc import Sized

def print_length(item: Sized) -> None:
    print(len(item))  # Works for lists, strings, dicts, etc.

print_length([1, 2, 3])       # ✓ Works
print_length("hello")         # ✓ Works
print_length({"a": 1})        # ✓ Works
```
Prefer importing these from `collections.abc`; the `typing.Sized`-style names are deprecated aliases (since Python 3.9). For the contextual example, `typing.ContextManager` is likewise an alias of `contextlib.AbstractContextManager`. The `collections.abc` classes use `__subclasshook__` (the pre-`Protocol` mechanism) so that `isinstance([], Sized)` is `True` and `isinstance(5, Iterable)` is `False` without any inheritance. `typing.Protocol` generalized that pattern into something you can use for your own interfaces and that static checkers understand.
## Protocol vs. Abstract Base Class (ABC)

| Feature | Protocol (typing.Protocol) | Abstract Base Class (abc.ABC) |
|-|-|-|
| Typing Style | Structural (Looks like a duck) | Nominal (Named inheritance) |
| Coupling | Decoupled: Classes don't know the protocol exists. | Tightly Coupled: Classes must explicitly inherit. |
| Implementation | Methods may have bodies, but only explicit subclasses inherit them. | Full concrete/shared logic, inherited by every subclass. |
| Enforcement | Static (type checker) only, unless `@runtime_checkable` or an explicit subclass uses `@abstractmethod`. | Instantiating a subclass that misses an `@abstractmethod` raises `TypeError` immediately. |
| Best Case | Third-party libraries, flexible APIs, modern type hints. | Internal frameworks, core shared logic hierarchies. |

## Callable Protocols

Protocols are especially useful for defining function signatures. Use `Callable` for simple functions or create custom callable protocols for complex behavior.

```python
from typing import Protocol, Callable

# Simple approach using Callable
def execute_operation(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

# Protocol approach for more complex callable behavior
class Transformer(Protocol):
    """Protocol for objects that transform data"""
    def transform(self, data: str) -> str: ...
    def validate(self) -> bool: ...

class UppercaseTransformer:
    def transform(self, data: str) -> str:
        return data.upper()
    
    def validate(self) -> bool:
        return True

def apply_transformer(transformer: Transformer, text: str) -> str:
    if transformer.validate():
        return transformer.transform(text)
    return text
```

A protocol can also define `__call__`, describing "any object callable with this signature" while allowing extra members, which `Callable[..., ...]` can't express:

```python
class ScoredCallable(Protocol):
    weight: float
    def __call__(self, x: int) -> int: ...
```

## Protocol Composition

Combine multiple protocols to create more specific requirements.

```python
from typing import Protocol

class Readable(Protocol):
    def read(self) -> str: ...

class Writable(Protocol):
    def write(self, data: str) -> None: ...

class Closeable(Protocol):
    def close(self) -> None: ...

# Composed protocol
class FileHandler(Readable, Writable, Closeable, Protocol):
    """Combines all three protocols"""
    pass

class TextFile:
    def read(self) -> str:
        return "content"
    
    def write(self, data: str) -> None:
        pass
    
    def close(self) -> None:
        pass

# Type checker accepts TextFile as FileHandler
def process_file(handler: FileHandler) -> None:
    content = handler.read()
    handler.write(content.upper())
    handler.close()
```
## Structural vs. Nominal Subtyping

### Nominal Subtyping (Traditional OOP)
Requires explicit inheritance relationship.

```python
class Animal:
    def speak(self) -> str:
        raise NotImplementedError

class Dog(Animal):  # MUST explicitly inherit from Animal
    def speak(self) -> str:
        return "Woof!"

def make_sound(animal: Animal) -> None:
    print(animal.speak())

dog = Dog()
make_sound(dog)  # ✓ Works because Dog inherits from Animal
```

### Structural Subtyping (Protocols)
Only requires matching structure, no explicit relationship.

```python
from typing import Protocol

class Speaker(Protocol):
    def speak(self) -> str: ...

class Dog:  # NO inheritance from Speaker
    def speak(self) -> str:
        return "Woof!"

def make_sound(speaker: Speaker) -> None:
    print(speaker.speak())

dog = Dog()
make_sound(dog)  # ✓ Works! Type checker accepts it because structure matches
```
## Explicit vs. Implicit Conformance

Structural matching is the default, but you are **allowed** to subclass a protocol explicitly:

```python
class Dog(Speaker):          # explicit: Dog is now nominally a subtype of Speaker too
    def speak(self) -> str:
        return "Woof!"
```
What explicit subclassing does and doesn't do (tested on 3.12):

- It **inherits** any method bodies the protocol defines (see above), and documents your intent.
- It does **not** force you to implement the members. A subclass that forgets `speak` instantiates fine, and `speak()` quietly returns `None`, because a `...` body is just a method returning `None`.
- To make forgetting a runtime error, mark the protocol method `@abstractmethod`; then instantiating a subclass that misses it raises `TypeError: Can't instantiate abstract class ...`. (A type checker will flag the missing method statically either way.)
- A protocol class itself can never be instantiated: `Speaker()` raises `TypeError: Protocols cannot be instantiated`.
- Other, unrelated classes can still conform structurally without inheriting; explicit inheritance doesn't make the protocol nominal-only.

## Common Pitfalls & Best Practices
### Best Practices

1. **Use Protocols for public APIs** - Allow flexibility in implementations
   ```python
   def save_data(writer: Writer) -> None:
       writer.write(b"data")  # Works with any object that has write()
   ```
2. **Prefer small, focused protocols** - Single responsibility principle
   ```python
   # Good
   class Readable(Protocol):
       def read(self) -> bytes: ...
   
   class Writable(Protocol):
       def write(self, data: bytes) -> None: ...
   ```
3. **Use @runtime_checkable sparingly** - Primarily for static type checking
   ```python
   from typing import Protocol, runtime_checkable
   
   @runtime_checkable
   class Drawable(Protocol):
       def draw(self) -> None: ...
   ```
4. **Document protocol requirements clearly**
   ```python
   class Serializable(Protocol):
       """
       Objects that can be converted to bytes.
       
       Methods:
           to_bytes() -> bytes: Convert to binary representation
       """
       def to_bytes(self) -> bytes: ...
   ```
### Common Pitfalls

1. **Forgetting @runtime_checkable for isinstance() checks**
   ```python
   # ✗ Raises TypeError at runtime without @runtime_checkable
   if isinstance(obj, MyProtocol):
       pass
   ```
2. **Creating overly complex protocols**
   ```python
   # ✗ Too many unrelated methods
   class BadProtocol(Protocol):
       def read(self) -> str: ...
       def write(self, data: str) -> None: ...
       def delete(self) -> None: ...
       def compress(self) -> bytes: ...
   ```
3. **Using a protocol to share implementation across many classes**
   ```python
   class Base(Protocol):
       def process(self) -> None:
           self.validate()        # legal, but only explicit subclasses of Base inherit it
   ```
   A protocol method body is only inherited through explicit subclassing, so classes that merely *match* `Base` never get `process()`. If you need shared logic every implementer must receive, use an ABC or a mixin.
4. **Assuming an explicit subclass is forced to implement the protocol**
   ```python
   class Dog(Speaker):    # forgets speak(); instantiates anyway unless Speaker.speak is @abstractmethod
       pass
   ```
   Explicit inheritance is fine, but don't treat it as enforcement. Rely on the type checker, or use `@abstractmethod` when you need a runtime guarantee.
## Real-World Examples

### Example 1: Plugin System

```python
from typing import Protocol

class Plugin(Protocol):
    """Interface for extensible plugins"""
    name: str
    version: str
    
    def initialize(self) -> None: ...
    def execute(self, data: dict) -> dict: ...
    def shutdown(self) -> None: ...

class ImageProcessorPlugin:
    name = "ImageProcessor"
    version = "1.0"
    
    def initialize(self) -> None:
        print("Loading PIL...")
    
    def execute(self, data: dict) -> dict:
        # Process image
        return {"result": "processed"}
    
    def shutdown(self) -> None:
        print("Cleaning up...")

def load_plugin(plugin: Plugin) -> None:
    plugin.initialize()
    result = plugin.execute({})
    plugin.shutdown()
```

### Example 2: Data Serialization

```python
import json
import pickle
from typing import Any, Protocol

class Serializer(Protocol):
    # Any object that can serialize data to bytes and back.
    # Bytes (not str) because formats like pickle produce arbitrary bytes, not text.
    def serialize(self, data: Any) -> bytes: ...
    def deserialize(self, raw: bytes) -> Any: ...

class JSONSerializer:
    def serialize(self, data: Any) -> bytes:
        return json.dumps(data).encode("utf-8")

    def deserialize(self, raw: bytes) -> Any:
        return json.loads(raw.decode("utf-8"))

class PickleSerializer:
    def serialize(self, data: Any) -> bytes:
        return pickle.dumps(data)

    def deserialize(self, raw: bytes) -> Any:
        return pickle.loads(raw)       # only unpickle data you trust: it can execute arbitrary code

def save_to_file(filename: str, data: dict, serializer: Serializer) -> None:
    # Works with any serializer that matches the Serializer protocol
    with open(filename, "wb") as f:
        f.write(serializer.serialize(data))

def load_from_file(filename: str, serializer: Serializer) -> Any:
    with open(filename, "rb") as f:
        return serializer.deserialize(f.read())

# Both JSONSerializer and PickleSerializer work seamlessly
save_to_file("data.json", {"key": "value"}, JSONSerializer())
save_to_file("data.pkl", {"key": "value"}, PickleSerializer())
print(load_from_file("data.json", JSONSerializer()))     # {'key': 'value'}
print(load_from_file("data.pkl", PickleSerializer()))    # {'key': 'value'}
```

### Example 3: Logger Abstraction

```python
from typing import Protocol

class Logger(Protocol):
    """Protocol for logging implementations"""
    def debug(self, msg: str) -> None: ...
    def info(self, msg: str) -> None: ...
    def warning(self, msg: str) -> None: ...
    def error(self, msg: str) -> None: ...

class ConsoleLogger:
    def debug(self, msg: str) -> None:
        print(f"[DEBUG] {msg}")
    
    def info(self, msg: str) -> None:
        print(f"[INFO] {msg}")
    
    def warning(self, msg: str) -> None:
        print(f"[WARNING] {msg}")
    
    def error(self, msg: str) -> None:
        print(f"[ERROR] {msg}")

class FileLogger:
    def __init__(self, filename: str):
        self.file = filename
    
    def debug(self, msg: str) -> None:
        with open(self.file, 'a') as f:
            f.write(f"[DEBUG] {msg}\n")
    
    def info(self, msg: str) -> None:
        with open(self.file, 'a') as f:
            f.write(f"[INFO] {msg}\n")
    
    def warning(self, msg: str) -> None:
        with open(self.file, 'a') as f:
            f.write(f"[WARNING] {msg}\n")
    
    def error(self, msg: str) -> None:
        with open(self.file, 'a') as f:
            f.write(f"[ERROR] {msg}\n")

def run_application(logger: Logger) -> None:
    logger.info("Starting application...")
    logger.debug("Debug mode enabled")
    logger.warning("Low memory")
    logger.error("Connection failed")

# Use either logger without changing application code
run_application(ConsoleLogger())
run_application(FileLogger("app.log"))
```
## When to Use Protocols
### **Use Protocols when:**
- Working with third-party libraries or external code you can't modify
- Building flexible APIs that accept multiple types
- Defining interfaces for modern, type-safe Python code
- You want decoupled, composable type contracts
- Creating plugin systems or extensible architectures
- Supporting duck typing with static type checking
### **Avoid Protocols when:**
- You need shared implementation logic (use ABC instead)
- Working with legacy code that doesn't support type hints
- You require strict runtime enforcement of contracts
- Creating internal class hierarchies with `isinstance()` checks
- Simplicity matters more than flexibility

## Interview Gotchas

- **Protocol vs ABC in one sentence:** a Protocol checks *shape* (structural, static duck typing); an ABC checks *lineage* (nominal inheritance) and can enforce `@abstractmethod` at instantiation.
- **Protocols are for the type checker by default**; at runtime `isinstance()` needs `@runtime_checkable`, and then it only checks that members exist, never their signatures.
- **Explicit subclassing is allowed**: it inherits default bodies but doesn't force implementation unless you use `@abstractmethod`.
- **Structural matches inherit nothing**, including any default method bodies the protocol defines.
- **`issubclass()` fails for protocols with data members**; `isinstance()` works.
- **Use `collections.abc` for the built-in protocols** (`Iterable`, `Sized`, ...); the `typing` aliases are deprecated.
- **Protocols formalize a pattern the standard library already used** (`__subclasshook__` in `collections.abc`).

## Summary
Python Protocols provide a powerful way to define interfaces using structural typing, allowing for flexible and decoupled code while maintaining strong static type checking. They're especially valuable in modern Python for building extensible systems, working with third-party libraries, and creating APIs that accept any object with the right structure.

# Part I — Foundations

### 1. Classes and Objects
A **class** is a blueprint; an **object** (instance) is a concrete thing built from that blueprint.A class defines structure/behavior; an instance is one concrete object built from it — many instances can share one class.
```python
class Dog:
    pass

my_dog = Dog()          # my_dog is an instance of Dog
print(type(my_dog))     # <class '__main__.Dog'>
```
### 2. The `__init__` Constructor and Attributes
```python
class Dog:
    def __init__(self, name, breed):
        self.name = name        # instance attribute
        self.breed = breed

d1 = Dog("Rex", "Labrador")
d2 = Dog("Milo", "Poodle")
```
- `self` refers to the instance being created/operated on — it is passed automatically.
- **Instance attributes** belong to one object; **class attributes** are shared by all instances.

```python
class Dog:
    species = "Canis familiaris"    # class attribute (shared)

    def __init__(self, name):
        self.name = name             # instance attribute (per object)

d1 = Dog("Rex")
print(Dog.species)        # accessed via class
print(d1.species)         # accessed via instance too

d1.species = "Wolf"      # assigning through an instance creates an INSTANCE attribute that shadows the class one
print(d1.species, Dog.species)   # Wolf Canis familiaris — the shared class attribute is unchanged
```
### 3. Instance, Class, and Static Methods

| Type | Decorator | First param | Use case |
| --- | --- | --- | --- |
| Instance method | none | `self` | Operates on one object's data |
| Class method | `@classmethod` | `cls` | Operates on the class itself; alt. constructors |
| Static method | `@staticmethod` | none | Utility function logically grouped with the class |
```python
class Dog:
    count = 0

    def __init__(self, name):
        self.name = name
        Dog.count += 1

    def bark(self):                       # instance method
        return f"{self.name} says Woof!"

    @classmethod
    def from_string(cls, data):           # alternate constructor
        name = data.split("-")[0]
        return cls(name)

    @staticmethod
    def is_valid_name(name):              # standalone helper
        return isinstance(name, str) and len(name) > 0
```
### 4. Encapsulation & Properties

Python doesn't enforce strict private access, but uses naming conventions:

- `name` — public
- `_name` — "protected" (convention: internal use, still accessible)
- `__name` — "private" (name-mangled to `_ClassName__name`)

```python
class Account:
    def __init__(self, balance):
        self.__balance = balance         # private

    def deposit(self, amount):
        if amount > 0:
            self.__balance += amount

    def get_balance(self):
        return self.__balance

acc = Account(100)
acc.deposit(50)
print(acc.get_balance())     # 150
# print(acc.__balance)       # AttributeError
```
#### Properties — Controlled Attribute Access
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius can't be negative")
        self._radius = value

    @property
    def area(self):                      # read-only computed property
        return 3.14159 * self._radius ** 2

c = Circle(5)
c.radius = 10        # calls the setter
print(c.area)         # calls the getter, computed on the fly
```

# Part II — Inheritance & Polymorphism

### 5. Inheritance and `super()`
A subclass inherits attributes and methods from a parent (base) class.
```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        raise NotImplementedError("Subclass must implement this")

class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow"

class Dog(Animal):
    def speak(self):
        return f"{self.name} says Woof"

for a in [Cat("Whiskers"), Dog("Rex")]:
    print(a.speak())
```
#### `super()` — calling the parent class
```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

class Manager(Employee):
    def __init__(self, name, salary, team_size):
        super().__init__(name, salary)   # reuse parent's init
        self.team_size = team_size
```
#### Multiple Inheritance & MRO
```python
class A:
    def greet(self): print("A")

class B:
    def greet(self): print("B")

class C(A, B):
    pass

C().greet()              # "A" — resolved via Method Resolution Order
print(C.__mro__)         # (C, A, B, object)
```
Python uses the **C3 linearization algorithm** to compute MRO — left-to-right, depth-first, but consistent.

#### The Diamond Problem, Explicitly

The classic multiple-inheritance issue: if `B` and `C` both inherit from `A`, and `D` inherits from both `B` and `C`, which version of a shared method should `D` use?

```python
class A:
    def greet(self): return "A"

class B(A):
    def greet(self): return "B"

class C(A):
    def greet(self): return "C"

class D(B, C):
    pass

print(D().greet())     # "B" — C3 linearization, not naive depth-first
print(D.__mro__)        # (D, B, C, A, object)
```

A naive depth-first search (like some older languages use) would visit `A` via `B` *before* even reaching `C`, risking `A` being checked twice or in an inconsistent order across a larger hierarchy. **C3 linearization** guarantees a *consistent, monotonic* ordering — a class always appears before its parents, and the relative order between siblings (`B` before `C`, since that's the order given in `class D(B, C)`) is preserved. If no consistent linearization is possible (contradictory base-class orderings), Python raises `TypeError: Cannot create a consistent method resolution order`.

### 6. Polymorphism and Duck Typing

Different classes respond to the same method call in their own way.

```python
class Shape:
    def area(self):
        raise NotImplementedError

class Square(Shape):
    def __init__(self, side):
        self.side = side
    def area(self):
        return self.side ** 2

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self):
        return 3.14159 * self.radius ** 2

shapes = [Square(4), Circle(3)]
for s in shapes:
    print(s.area())      # same call, different behavior — polymorphism
```
Python also supports **duck typing**: "if it walks like a duck and quacks like a duck…" — no explicit type checking is required, only that the object has the needed method.

# Part III — Abstraction & Composition

### 7. Abstraction (`abc`)
Use the `abc` module to define abstract base classes that force subclasses to implement certain methods.
```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount):
        pass

class CreditCard(PaymentMethod):
    def pay(self, amount):
        print(f"Paid {amount} via credit card")

# PaymentMethod()          # TypeError: can't instantiate abstract class
card = CreditCard()
card.pay(100)
```

#### Abstract Properties

`@abstractmethod` can stack with `@property` — useful when subclasses must provide a value, not just a callable method:

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @property
    @abstractmethod
    def area(self):
        ...

class Square(Shape):
    def __init__(self, side):
        self.side = side

    @property
    def area(self):
        return self.side ** 2

# Shape()          # TypeError — area is still abstract
sq = Square(4)
print(sq.area)      # 16 — accessed like an attribute, not sq.area()
```

The decorator order matters: `@property` must be the outermost decorator so `Shape.area` is recognized as a property by callers, while `@abstractmethod` marks it as unimplemented underneath.

#### Virtual Subclasses — `ABCMeta.register()`

An ABC can also accept a class as a "virtual subclass" **without** that class inheriting from it at all — `isinstance()`/`issubclass()` checks pass, but no method implementations or MRO slot are inherited. This is a nominal-typing escape hatch for structural-style registration, distinct from the `typing.Protocol` approach used elsewhere in this material.

```python
from abc import ABC, abstractmethod

class Flyer(ABC):
    @abstractmethod
    def fly(self): ...

class Superman:                 # doesn't inherit from Flyer at all
    def fly(self):
        return "Up, up, and away!"

Flyer.register(Superman)
print(issubclass(Superman, Flyer))   # True — registered, not inherited
print(isinstance(Superman(), Flyer))  # True
```

### 8. Composition vs. Inheritance ("has-a" vs "is-a")

#### 8.1 The Core Distinction

- **Inheritance ("is-a")** — a `Cat` *is an* `Animal`. The subclass inherits the parent's interface and implementation, and the relationship is fixed at class-definition time.
- **Composition ("has-a")** — a `Car` *has an* `Engine`. One object holds a reference to another and delegates work to it, and that reference can be swapped at runtime.

```python
class Engine:
    def start(self):
        return "Engine started"

class Car:
    def __init__(self):
        self.engine = Engine()        # Car "has-a" Engine

    def start(self):
        return self.engine.start()   # delegation: Car forwards the call

car = Car()
print(car.start())
```

#### 8.2 Why Composition Is Often More Flexible

The example above hard-codes `Engine`. With composition, you can instead **inject** the component, letting the same `Car` work with different engines without touching `Car`'s code at all:

```python
class ElectricEngine:
    def start(self):
        return "Silent electric hum..."

class GasEngine:
    def start(self):
        return "Vroom! Engine roaring."

class Car:
    def __init__(self, engine):
        self.engine = engine           # dependency injected, not hard-coded

    def start(self):
        return self.engine.start()

tesla = Car(ElectricEngine())
mustang = Car(GasEngine())
print(tesla.start())      # Silent electric hum...
print(mustang.start())    # Vroom! Engine roaring.
```

This is the **strategy pattern** in miniature: behavior is swapped by passing in a different object, not by creating a new subclass for every combination.

#### 8.3 Where Inheritance Breaks Down

Inheritance models a rigid, single hierarchy. Problems appear when a "thing" needs to combine behaviors that don't nest cleanly:

```python
# Awkward: does a FlyingCar inherit from Car or Plane? Both?
class Car:
    def drive(self): return "Driving"

class Plane:
    def fly(self): return "Flying"

class FlyingCar(Car, Plane):   # multiple inheritance gets tangled fast
    pass
```

With composition, `FlyingCar` simply *has* the capabilities it needs, with no ambiguity about a "true" parent:

```python
class FlyingCar:
    def __init__(self):
        self.car = Car()
        self.plane = Plane()

    def drive(self):
        return self.car.drive()

    def fly(self):
        return self.plane.fly()
```

#### 8.4 Composition Enables Runtime Flexibility

Because a composed object is just an attribute, it can be **replaced after construction** — something a fixed inheritance hierarchy can't do:

```python
class GasEngine:
    def start(self): return "Vroom! Engine roaring."

class ElectricEngine:
    def start(self): return "Silent electric hum..."

class Car:                                   # the injected-engine Car from 8.2
    def __init__(self, engine):
        self.engine = engine

    def start(self):
        return self.engine.start()

car = Car(GasEngine())
print(car.start())          # Vroom! Engine roaring.

car.engine = ElectricEngine()   # swap the component at runtime
print(car.start())              # Silent electric hum...
```

#### 8.5 Aggregation — a Looser Form of Composition

Composition usually implies the container **owns** the component's lifecycle (the `Engine` doesn't outlive the `Car`). **Aggregation** is the looser cousin: the objects are associated, but each can exist independently.

```python
class Driver:
    def __init__(self, name):
        self.name = name

class Car:
    def __init__(self, driver):
        self.driver = driver      # Car "uses" a Driver, but doesn't own it

alice = Driver("Alice")
car = Car(alice)
# alice existed before the car and can exist after it — aggregation, not ownership
```

#### 8.6 "Favor Composition Over Inheritance" — the Practical Rule

This well-known design principle doesn't mean *never* use inheritance — it means: reach for composition first, and use inheritance only when there's a genuine, stable "is-a" relationship where the subclass should be substitutable for the parent everywhere (see the Liskov Substitution Principle in Part V).

| Question | Leans toward |
| --- | --- |
| Does the relationship ever need to change at runtime? | Composition |
| Are you inheriting just to reuse a method, not because of a real "is-a" relationship? | Composition |
| Would multiple inheritance be needed to express it? | Composition (use separate components or mixins) |
| Is the subclass truly a more specific version of the parent, always substitutable for it? | Inheritance |

#### 8.7 Comparing the Two Side by Side

```python
# Inheritance approach — tightly coupled, fixed at class definition
class Bird:
    def move(self):
        return "Flying"

class Eagle(Bird):
    pass

# Composition approach — flexible, swappable at runtime
class FlyingBehavior:
    def move(self):
        return "Flying"

class WalkingBehavior:
    def move(self):
        return "Walking"

class Bird:
    def __init__(self, movement):
        self.movement = movement

    def move(self):
        return self.movement.move()

eagle = Bird(FlyingBehavior())
penguin = Bird(WalkingBehavior())   # no awkward "Penguin(Bird)" that can't fly
```

The composition version sidesteps a classic OOP trap: a `Penguin(Bird)` subclass that inherits a `fly()` method it can't actually honor — a Liskov Substitution violation. Composing in the right `movement` behavior avoids that problem entirely.
### 9. Mixins
A **mixin** is a small class meant to be combined with others via multiple inheritance to add one piece of reusable behavior — it isn't meant to stand alone.
```python
class JSONExportMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class ComparableMixin:
    def __eq__(self, other):
        return self.__dict__ == other.__dict__

class User(JSONExportMixin, ComparableMixin):
    def __init__(self, name, age):
        self.name = name
        self.age = age

u = User("Ana", 30)
print(u.to_json())          # {"name": "Ana", "age": 30}
```
Mixins keep single-purpose behavior reusable across unrelated class hierarchies without deep inheritance chains.

# Part IV — Making Classes Feel Native

## 10. Dunder (Magic) Methods

These let your objects integrate with built-in Python syntax and functions.

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):               # developer-facing representation
        return f"Vector({self.x}, {self.y})"

    def __str__(self):                # user-facing string
        return f"({self.x}, {self.y})"

    def __add__(self, other):         # enables v1 + v2
        return Vector(self.x + other.x, self.y + other.y)

    def __eq__(self, other):          # enables v1 == v2
        return self.x == other.x and self.y == other.y

    def __len__(self):                # enables len(v)
        return int((self.x**2 + self.y**2) ** 0.5)

v1, v2 = Vector(1, 2), Vector(3, 4)
print(v1 + v2)      # (4, 6)
print(v1 == v2)      # False
```

Common dunder methods:

| Method | Purpose |
| --- | --- |
| `__init__` | Constructor |
| `__repr__` / `__str__` | String representations |
| `__eq__`, `__lt__`, `__gt__` | Comparisons |
| `__add__`, `__sub__`, `__mul__` | Arithmetic operators |
| `__len__` | `len(obj)` |
| `__getitem__` / `__setitem__` | Indexing, `obj[key]` |
| `__iter__` / `__next__` | Iteration support |
| `__enter__` / `__exit__` | Context managers (`with` statement) |
| `__call__` | Makes an instance callable like a function |
| `__hash__` | Enables use in sets/dict keys |

**Note on `__eq__` and `__hash__`:** defining `__eq__` sets `__hash__` to `None` unless you define it too — meaning your objects become unhashable. If instances should be usable in a `set` or as dict keys, define both consistently (equal objects must have equal hashes).

#### Reflected and In-Place Operators

`__add__` only handles `left_operand + right`. Two related families of dunders fill in the rest:

- **Reflected operators** (`__radd__`, `__rmul__`, ...) — Python tries `left.__add__(right)` first; if that returns `NotImplemented` (e.g., `left` doesn't know how to combine with `right`'s type), Python then tries `right.__radd__(left)`. This is what lets `5 + my_vector` work even though `int.__add__` has no idea what a `Vector` is.
  ```python
  class Vector:
      def __init__(self, x): self.x = x
      def __add__(self, other):
          if isinstance(other, Vector):
              return Vector(self.x + other.x)
          return NotImplemented
      def __radd__(self, other):     # handles other + self, e.g. 5 + Vector(1)
          return self.__add__(other)
  ```
- **In-place operators** (`__iadd__`, `__imul__`, ...) — back `+=`, `*=`, etc. If undefined, Python falls back to the regular operator plus reassignment (`x = x + y`). Defining `__iadd__` lets you mutate in place instead of creating a new object — important for mutable types where `+=` should modify, not replace (this is exactly why `list += [x]` mutates the original list while `list = list + [x]` creates a new one).

#### `functools.total_ordering` — Less Boilerplate for Comparisons

Implementing every comparison dunder (`__lt__`, `__le__`, `__gt__`, `__ge__`) by hand is repetitive. Supply just `__eq__` and one ordering method, and let the decorator fill in the rest:

```python
from functools import total_ordering

@total_ordering
class Money:
    def __init__(self, amount):
        self.amount = amount
    def __eq__(self, other):
        return self.amount == other.amount
    def __lt__(self, other):
        return self.amount < other.amount

# __le__, __gt__, __ge__ are all derived automatically
print(Money(5) < Money(10))    # True
print(Money(5) >= Money(10))    # False — derived from __lt__ and __eq__
```

## 11. Iterators and Context Managers

### The iterator protocol

```python
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self                 # an iterator returns itself

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for n in Countdown(3):
    print(n)                        # 3, 2, 1
```

### Custom context managers

```python
class ManagedFile:
    def __init__(self, path):
        self.path = path

    def __enter__(self):
        self.file = open(self.path, "w")
        return self.file

    def __exit__(self, exc_type, exc_value, traceback):
        self.file.close()
        return False        # False = don't suppress exceptions

with ManagedFile("notes.txt") as f:
    f.write("Hello!")
# file is automatically closed here, even if an error occurred
```

## 12. Class Design Tools

### `@dataclass` — reduce boilerplate for data-holding classes

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p = Point(1, 2)
print(p)             # Point(x=1, y=2) — __repr__ generated automatically
```

`@dataclass` also generates `__eq__` (comparing all fields) for free, and takes options for common needs:

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, order=True)   # frozen = immutable; order = adds comparison dunders
class Point:
    x: int
    y: int

p1, p2 = Point(1, 2), Point(1, 3)
print(p1 < p2)         # True — compares field tuples (1, 2) < (1, 3)
# p1.x = 99            # FrozenInstanceError — frozen=True blocks mutation

@dataclass
class Cart:
    items: list = field(default_factory=list)   # NOT items=[] — same mutable-default trap as functions
```

`field(default_factory=list)` is required instead of a bare `items: list = []` for exactly the same reason `def f(x=[])` is a bug in regular functions — a plain mutable default would be created once and shared across every instance.

### `__slots__` — restrict attributes, save memory

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y
```

---

# Part V — Design & Errors

## 13. Custom Exceptions

Exceptions are ordinary classes — build a hierarchy by inheriting from `Exception` so callers can catch broadly or narrowly.

```python
class InsufficientFundsError(Exception):
    """Raised when a withdrawal exceeds the available balance."""
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        super().__init__(f"Cannot withdraw {amount}; balance is {balance}")

class Account:
    def __init__(self, balance):
        self.balance = balance

    def withdraw(self, amount):
        if amount > self.balance:
            raise InsufficientFundsError(self.balance, amount)
        self.balance -= amount

try:
    Account(50).withdraw(100)
except InsufficientFundsError as e:
    print(e)                    # Cannot withdraw 100; balance is 50
```

Custom exception hierarchies let you define a base error for a module (`class AppError(Exception)`) with specific subclasses beneath it, so callers can catch at whichever level of granularity they need.

## 14. SOLID Principles in Python

- **Single Responsibility** — a class should have one reason to change.
- **Open/Closed** — extend behavior via subclassing/composition rather than modifying existing code.
- **Liskov Substitution** — subclasses should be usable wherever the parent class is expected.
- **Interface Segregation** — favor small, focused abstract base classes over large ones.
- **Dependency Inversion** — depend on abstractions (e.g., an ABC), not concrete implementations.

---

# Part VI — Under the Hood

## 15. `__new__` vs `__init__`

`__new__` actually **creates** the object; `__init__` **initializes** it after creation. `__new__` is rarely overridden, but understanding it clarifies how instantiation works.

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)     # create only once
        return cls._instance

    def __init__(self, value):
        self.value = value      # still runs every time Singleton(...) is called

a = Singleton(1)
b = Singleton(2)
print(a is b)          # True — same object
print(a.value)          # 2 — __init__ ran again and overwrote it
```

`__new__` is a static method (implicitly) that returns the new instance; Python then calls `__init__` on that instance automatically — but only if `__new__` returned an instance of the class.

## 16. Attribute Access Hierarchy & Descriptors

### 16.1 The Attribute Lookup Order

When you write `obj.attr`, Python follows a defined search order, orchestrated by the default `__getattribute__` method every object inherits from `object`:

1. **Data descriptors** found on the type (and its MRO) — checked first, take priority over instance `__dict__`
2. **The instance's own `__dict__`**
3. **Non-data descriptors and other class attributes** found on the type (and its MRO)
4. **`__getattr__`** on the class, if defined — only called as a last resort, when normal lookup fails

```python
class Demo:
    class_attr = "class level"

    def __init__(self):
        self.instance_attr = "instance level"

d = Demo()
print(d.instance_attr)   # found in d.__dict__
print(d.class_attr)      # not in d.__dict__, found via type(d).__mro__
```

### 16.2 `__getattribute__` vs `__getattr__`

| Method | Called | Purpose |
| --- | --- | --- |
| `__getattribute__` | On **every** attribute access | Implements the full lookup algorithm — rarely overridden |
| `__getattr__` | Only when normal lookup **fails** (raises `AttributeError`) | Fallback for missing attributes, e.g. lazy loading, proxies |

```python
class Lazy:
    def __getattr__(self, name):
        print(f"{name} not found normally, computing it...")
        value = name.upper()
        setattr(self, name, value)   # cache it for next time
        return value

l = Lazy()
print(l.hello)    # triggers __getattr__
print(l.hello)    # now in __dict__, __getattr__ NOT called again
```

### 16.3 Descriptors — the mechanism behind `@property`

A **descriptor** is any object whose class defines one of `__get__`, `__set__`, or `__delete__`. Descriptors are how Python implements `@property`, methods, `@classmethod`, `@staticmethod`, and `__slots__` internally.

**Non-data descriptor** — only `__get__`:

```python
class NonDataDescriptor:
    def __get__(self, obj, owner):
        return "computed value"

class A:
    attr = NonDataDescriptor()

a = A()
print(a.attr)          # "computed value"
a.__dict__['attr'] = "instance override"
print(a.attr)           # "instance override" — instance dict wins for non-data descriptors
```

**Data descriptor** — `__get__` *and* `__set__` (or `__delete__`):

```python
class DataDescriptor:
    def __get__(self, obj, owner):
        return obj._value
    def __set__(self, obj, value):
        print(f"Setting to {value}")
        obj._value = value

class B:
    attr = DataDescriptor()
    def __init__(self, v):
        self.attr = v         # calls __set__

b = B(10)
b.__dict__['attr'] = "override attempt"
print(b.attr)          # still uses the descriptor — data descriptors always win
```

The key rule: **data descriptors always beat instance `__dict__`**; non-data descriptors lose to it.

### 16.4 Building `@property` from Scratch

`property` is literally a built-in data descriptor:

```python
class MyProperty:
    def __init__(self, fget=None, fset=None):
        self.fget = fget
        self.fset = fset

    def __get__(self, obj, owner):
        if obj is None:
            return self
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

class Temperature:
    def _get_celsius(self):
        return self._celsius
    def _set_celsius(self, value):
        self._celsius = value
    celsius = MyProperty(_get_celsius, _set_celsius)
```

This is essentially what `@property` does for you with nicer syntax.

### 16.5 Why Methods Are Descriptors Too

Functions are non-data descriptors — that's literally how `self` gets bound automatically:

```python
class Foo:
    def bar(self):
        return "hi"

f = Foo()
print(Foo.__dict__['bar'])      # a plain function
print(f.bar)                     # <bound method Foo.bar of <Foo object>>
```

Accessing `f.bar` triggers `function.__get__(f, Foo)`, which returns a *bound method* — that's where `self` comes from.

### 16.6 Practical Uses of Descriptors

Descriptors shine for reusable validation/behavior across many attributes:

```python
class Positive:
    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, obj, owner):
        return getattr(obj, self.name)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} must be positive")
        setattr(obj, self.name, value)

class Product:
    price = Positive()
    quantity = Positive()

    def __init__(self, price, quantity):
        self.price = price
        self.quantity = quantity

p = Product(10, 5)
p.price = -5      # raises ValueError
```

`__set_name__` (added in Python 3.6) lets the descriptor learn what attribute name it was assigned to, without you passing it manually.

### 16.7 Full Priority Summary

From highest to lowest priority when resolving `instance.attr`:

1. Data descriptor on the class/MRO (`__get__` + `__set__`/`__delete__`)
2. Instance `__dict__`
3. Non-data descriptor or plain class attribute on the class/MRO (methods, `@classmethod`, `@staticmethod`, plain values)
4. `__getattr__` (only if all the above fail)

## 17. `__init_subclass__` — A Lighter Alternative to Metaclasses

Before reaching for a metaclass, check whether `__init_subclass__` covers the need. It's a classmethod, implicitly defined via `object`, that Python calls automatically **whenever a subclass is created** — no metaclass required.

```python
class Plugin:
    registry = []

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry.append(cls)    # auto-register every subclass on creation

class AudioPlugin(Plugin):
    pass

class VideoPlugin(Plugin):
    pass

print(Plugin.registry)   # [<class 'AudioPlugin'>, <class 'VideoPlugin'>]
```

This solves a large fraction of the problems people traditionally used metaclasses for — plugin registries, validating subclass structure, injecting shared setup — with far simpler syntax and no risk of metaclass-conflict errors when combined with other base classes. Reach for `__init_subclass__` first; drop to a real metaclass only if you specifically need to control class *creation itself* (e.g., altering the namespace before the class object exists, as in the `UpperAttrMeta` example below).

## 18. A Glimpse of Metaclasses

A **metaclass** is "the class of a class" — it controls how classes themselves are created, the same way a class controls how instances are created. `type` is the default metaclass for every class in Python.

```python
class UpperAttrMeta(type):
    def __new__(mcs, name, bases, namespace):
        # uppercase all non-dunder attribute names at class-creation time
        upper_namespace = {
            (key.upper() if not key.startswith("__") else key): value
            for key, value in namespace.items()
        }
        return super().__new__(mcs, name, bases, upper_namespace)

class Config(metaclass=UpperAttrMeta):
    timeout = 30

print(Config.TIMEOUT)     # 30 — attribute was renamed at class-creation time
```

Metaclasses are powerful but rarely needed directly — frameworks like Django (models) and ORMs use them internally. As the saying goes: "if you're wondering whether you need metaclasses, you don't" — reach for class decorators or `__init_subclass__` first for simpler customization needs.

---

# Appendix

## Common Pitfalls

1. **Mutable default arguments** — `def __init__(self, items=[])` shares *one* list across every instance that doesn't pass `items`. Use `items=None` and set `self.items = items or []` inside instead.
2. **Mutable class attributes** — a class attribute that's a list/dict is shared by all instances unless you deliberately override it per instance in `__init__`.
3. **Forgetting `super().__init__()`** in subclasses — parent state silently never gets set up.
4. **Overusing inheritance** where composition would be simpler and more flexible ("favor composition over inheritance").
5. **Comparing with `==` after defining `__eq__` but not `__hash__`** — makes instances unhashable, breaking use in sets/dicts.
6. **Confusing `@staticmethod` and `@classmethod`** — use `@classmethod` when you need the class itself (e.g., alternate constructors), `@staticmethod` when you don't need `self` or `cls` at all.
7. **`@dataclass` mutable defaults** — `items: list = []` fails at class-definition time in a dataclass (it explicitly rejects bare mutable defaults); use `field(default_factory=list)` instead.
8. **Reaching for a metaclass by default** — most "customize class creation" needs are better served by `__init_subclass__` or a simple class decorator; save metaclasses for when you truly need to intercept the class object's creation itself.

## Practice Exercises

1. Build a `Library` class composed of `Book` objects; implement `__len__` and `__iter__` so you can do `len(library)` and `for book in library`.
2. Create an abstract `Shape` base class with an abstract `area()` method, then implement `Rectangle` and `Triangle` subclasses.
3. Write a `Temperature` descriptor that stores the value in Celsius internally but exposes both `.celsius` and `.fahrenheit` properties.
4. Design a small custom exception hierarchy for a `FileParser` class (e.g., `ParseError` → `MissingHeaderError`, `InvalidRowError`).
5. Implement a `LoggingMixin` that any class can inherit to gain a `.log(message)` method printing `"[ClassName] message"`.

## Quick Reference

| Concept | Keyword / Tool |
| --- | --- |
| Class definition | `class Name:` |
| Constructor | `__init__(self, ...)` |
| Object creation | `__new__(cls, ...)` |
| Inheritance | `class Child(Parent):` |
| Call parent method | `super().method()` |
| Encapsulation | `_protected`, `__private`, `@property` |
| Polymorphism | Method overriding, duck typing |
| Abstraction | `from abc import ABC, abstractmethod` |
| Composition | Storing another object as an attribute |
| Mixin | Multiple inheritance for one reusable behavior |
| Operator overloading | `__add__`, `__eq__`, etc. |
| Class-level method | `@classmethod` |
| Utility method | `@staticmethod` |
| Iteration protocol | `__iter__`, `__next__` |
| Context manager | `__enter__`, `__exit__` |
| Custom exceptions | `class MyError(Exception):` |
| Descriptor | `__get__`, `__set__`, `__delete__` |
| Metaclass | `class Meta(type):` |
| Class-creation hook (lighter than metaclass) | `__init_subclass__` |
| Reflected operator | `__radd__`, `__rmul__`, etc. |
| In-place operator | `__iadd__`, `__imul__`, etc. |
| Fill in comparison dunders | `@functools.total_ordering` |
| Virtual subclass (no inheritance) | `ABCMeta.register()` |
| Immutable / auto-ordered dataclass | `@dataclass(frozen=True, order=True)` |

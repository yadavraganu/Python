## Contents

- [Part I — Foundations](#part-i--foundations)

  1. [What OOP is and when to use it](#1-what-oop-is-and-when-to-use-it)
  2. [Classes and objects](#2-classes-and-objects)
  3. [Instance creation, `self`, and `__init__`](#3-instance-creation-self-and-__init__)
  4. [Instance attributes vs class attributes](#4-instance-attributes-vs-class-attributes)
  5. [Instance, class, and static methods](#5-instance-class-and-static-methods)
  6. [Encapsulation, naming conventions, and properties](#6-encapsulation-naming-conventions-and-properties)
- [Part II — Inheritance and Polymorphism](#part-ii--inheritance-and-polymorphism)
  
  7. [Inheritance and method overriding](#7-inheritance-and-method-overriding)
  8. [`super()` and cooperative inheritance](#8-super-and-cooperative-inheritance)
  9. [Multiple inheritance and MRO](#9-multiple-inheritance-and-mro)
  10. [Polymorphism and duck typing](#10-polymorphism-and-duck-typing)
  11. [Structural typing with `Protocol`](#11-structural-typing-with-protocol)
- [Part III — Abstraction and Composition](#part-iii--abstraction-and-composition)
  
  12. [Abstract base classes (`abc`)](#12-abstract-base-classes-abc)
  13. [Composition, dependency injection, and delegation](#13-composition-dependency-injection-and-delegation)
  14. [Aggregation vs composition](#14-aggregation-vs-composition)
  15. [Mixins](#15-mixins)
- [Part IV — Making Classes Behave Like Native Python Objects](#part-iv--making-classes-behave-like-native-python-objects)
  
  16. [Dunder methods and object representations](#16-dunder-methods-and-object-representations)
  17. [Operator overloading and `NotImplemented`](#17-operator-overloading-and-notimplemented)
  18. [Equality, ordering, and hashing](#18-equality-ordering-and-hashing)
  19. [Iteration and container protocols](#19-iteration-and-container-protocols)
  20. [Context managers](#20-context-managers)
  21. [Dataclasses](#21-dataclasses)
  22. [`__slots__`](#22-__slots__)
- [Part V — Design and Error Handling](#part-v--design-and-error-handling)
  
  23. [Custom exceptions](#23-custom-exceptions)
  24. [SOLID principles](#24-solid-principles)
  25. [Choosing inheritance, composition, ABCs, and Protocols](#25-choosing-inheritance-composition-abcs-and-protocols)
- [Part VI — Python's Object Model](#part-vi--pythons-object-model)
  
  26. [`__new__` vs `__init__`](#26-__new__-vs-__init__)
  27. [Attribute lookup](#27-attribute-lookup)
  28. [Descriptors](#28-descriptors)
  29. [`__init_subclass__`](#29-__init_subclass__)
  30. [Metaclasses](#30-metaclasses)

---

# Part I — Foundations

## 1. What OOP Is and When to Use It

Object-oriented programming (OOP) is a way to organize code around **objects**. An object combines:

- **State:** the data the object currently holds, such as an account balance.
- **Behavior:** the operations the object can perform, such as deposit or withdraw.

A **class** defines how objects of a particular kind are created and what data and behavior they provide. An **instance** is one concrete object created from that class.

### A simple comparison

A procedural design might keep data and functions separately:

```python
balance = 100


def deposit(current_balance, amount):
    return current_balance + amount


balance = deposit(balance, 50)
print(balance)  # 150
```

Line by line:

1. `balance = 100` creates a variable holding the current balance.
2. `deposit(...)` is a function that receives the current balance and the amount to add.
3. The function returns a new balance; the caller stores that result.

An object-oriented version keeps the balance and the operations that work on it together:

```python
class Account:
    def __init__(self, balance):
        self.balance = balance

    def deposit(self, amount):
        self.balance += amount


account = Account(100)
account.deposit(50)
print(account.balance)  # 150
```

Line by line:

1. `class Account:` defines a new class named `Account`.
2. `def __init__(self, balance):` defines initialization logic that runs when an instance is constructed.
3. `self.balance = balance` stores the starting balance on this particular account object.
4. `def deposit(self, amount):` defines an operation that acts on an account instance.
5. `self.balance += amount` updates that instance's state.
6. `Account(100)` creates an account with an initial balance of 100.
7. `account.deposit(50)` calls the method on that account.
8. `print(account.balance)` reads the updated state.

OOP is useful when the program has entities with state, behavior, and relationships. It is not automatically better for every problem. A short calculation or data transformation may be clearer as a normal function.

## 2. Classes and Objects

```python
class Dog:
    """Represent a dog. This class currently has no custom behavior."""
    pass


first_dog = Dog()
second_dog = Dog()

print(type(first_dog).__name__)  # Dog
print(first_dog is second_dog)   # False
```

Line by line:

1. `class Dog:` begins a class definition.
2. The string on the next line is the class docstring. It documents the class.
3. `pass` is a placeholder statement; it lets the class body be syntactically valid without adding behavior.
4. `Dog()` creates a new instance.
5. Calling `Dog()` a second time creates another, distinct instance.
6. `type(first_dog)` returns the object's class; `.__name__` reads the class name.
7. `is` checks object identity, not value equality. The two instances are different objects.

Important distinctions:

- `Dog` refers to the class.
- `Dog()` creates an instance.
- `first_dog` is a variable referring to that instance.
- `type(first_dog) is Dog` checks whether the instance's exact type is `Dog`.
- `isinstance(first_dog, Dog)` also returns `True` for instances of subclasses of `Dog`.

## 3. Instance Creation, `self`, and `__init__`

`self` is the conventional name for the first parameter of an instance method. It refers to the instance on which the method is called. Python supplies it automatically when the method is accessed through an instance.

`__init__` is an **initializer**, not technically the method that creates the object. It initializes an instance after it has been created. Object creation is handled by `__new__`; that distinction is explained in Part VI.

```python
class Dog:
    def __init__(self, name, breed):
        self.name = name
        self.breed = breed

    def describe(self):
        return f"{self.name} is a {self.breed}."


rex = Dog("Rex", "Labrador")
print(rex.describe())  # Rex is a Labrador.
```

Line by line:

1. `class Dog:` defines the class.
2. `def __init__(self, name, breed):` declares the initialization parameters. `self` is the newly created instance; the caller supplies `name` and `breed`.
3. `self.name = name` creates or assigns an instance attribute named `name`.
4. `self.breed = breed` stores a second instance attribute.
5. `def describe(self):` defines an instance method.
6. The f-string reads attributes from the same instance.
7. `Dog("Rex", "Labrador")` constructs an object and invokes its initializer.
8. `rex.describe()` is effectively bound to `rex`; Python passes `rex` as `self` automatically.

The following two calls are equivalent in effect:

```python
class Dog:
    def __init__(self, name):
        self.name = name

    def describe(self):
        return f"Dog: {self.name}"


rex = Dog("Rex")

# These are equivalent ways to call the bound method:
print(rex.describe())
print(Dog.describe(rex))
```

The second form is rarely used in normal application code, but it makes the automatic `self` argument visible.

## 4. Instance Attributes vs Class Attributes

An **instance attribute** is stored separately on each instance. A **class attribute** is defined on the class and is available to instances unless an instance attribute with the same name shadows it.

```python
class Dog:
    species = "Canis familiaris"  # class attribute

    def __init__(self, name):
        self.name = name           # instance attribute


rex = Dog("Rex")
milo = Dog("Milo")

print(rex.name)       # Rex
print(milo.name)      # Milo
print(rex.species)    # Canis familiaris
print(Dog.species)    # Canis familiaris

rex.species = "Wolf"  # creates an instance attribute; it does not edit Dog.species
print(rex.species)     # Wolf
print(milo.species)    # Canis familiaris
print(Dog.species)     # Canis familiaris
```

Line by line:

1. `species` is defined in the class body, so it belongs to the class.
2. `self.name` is assigned during initialization, so each instance receives its own `name`.
3. `rex` and `milo` are separate instances and therefore have separate `name` attributes.
4. `rex.species` is found on the class because `rex` has no instance attribute named `species` yet.
5. `Dog.species` reads the class attribute directly.
6. Assigning `rex.species` creates an attribute in `rex`'s instance namespace. It shadows the class attribute for that instance only.

### A mutable class-attribute trap

Mutable class attributes such as lists and dictionaries are shared. This can be intentional for class-wide state, but it is often a bug when you meant each instance to have its own collection.

```python
class Team:
    members = []  # shared by every Team instance — usually not what you want


team_a = Team()
team_b = Team()
team_a.members.append("Asha")

print(team_b.members)  # ['Asha'] — both instances see the same list
```

Use an instance attribute for per-object state:

```python
class Team:
    def __init__(self):
        self.members = []  # a new list is created for each instance


team_a = Team()
team_b = Team()
team_a.members.append("Asha")

print(team_a.members)  # ['Asha']
print(team_b.members)  # []
```

## 5. Instance, Class, and Static Methods

Python has three common method forms.

| Method kind | Decorator | First parameter | Typical purpose |
|---|---|---|---|
| Instance method | None | `self` | Read or change one instance's state. |
| Class method | `@classmethod` | `cls` | Work with the class; often used for alternative constructors. |
| Static method | `@staticmethod` | None automatically | A utility related conceptually to the class but needing neither instance nor class state. |

```python
class Dog:
    created_count = 0

    def __init__(self, name, breed=None):
        self.name = name
        self.breed = breed
        type(self).created_count += 1

    def bark(self):
        return f"{self.name} says woof!"

    @classmethod
    def from_record(cls, record):
        name, breed = record.split(",")
        return cls(name.strip(), breed.strip())

    @staticmethod
    def is_valid_name(name):
        return isinstance(name, str) and bool(name.strip())


rex = Dog("Rex")
other = Dog.from_record("Milo, Poodle")
print(rex.bark())                    # Rex says woof!
print(other.name, other.breed)       # Milo Poodle
print(Dog.is_valid_name("Fido"))     # True
print(Dog.created_count)             # 2
```

Line by line:

1. `created_count` is class-level state.
2. `__init__` stores the instance name and increments the counter.
3. `type(self)` refers to the actual runtime class of the instance. This is useful when subclasses are involved; using `Dog.created_count` would always update `Dog` directly.
4. `bark` is an instance method because it reads `self.name`.
5. `@classmethod` changes method binding: Python passes the class as `cls`.
6. `from_record` parses a simple comma-separated string and calls `cls(...)` to construct an object of the class through which it was called. This supports subclasses better than hard-coding `Dog(...)`.
7. The `from_record` method calls `cls(...)`, so the same constructor and validation path is used for regular construction and alternate construction.
8. `@staticmethod` means Python does not automatically pass `self` or `cls`.
9. `is_valid_name` checks that the input is a non-empty string after trimming whitespace.
10. The final statements demonstrate each method type.

**Rule of thumb:** use an instance method by default. Use a class method when the class itself is needed. Use a static method only when grouping a helper in the class improves organization; a module-level function may be just as clear.

## 6. Encapsulation, Naming Conventions, and Properties

Encapsulation means keeping an object's internal state and the operations that maintain its validity together. Python relies more on conventions than on enforced access modifiers.

| Form | Meaning in Python |
|---|---|
| `name` | Public attribute; intended for normal use. |
| `_name` | Non-public implementation detail by convention. It is still accessible. |
| `__name` | Name-mangled to reduce accidental name clashes in subclasses. It is not a secure private field. |

```python
class Account:
    def __init__(self, opening_balance):
        if opening_balance < 0:
            raise ValueError("Opening balance cannot be negative")
        self._balance = opening_balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self._balance += amount

    def get_balance(self):
        return self._balance


account = Account(100)
account.deposit(50)
print(account.get_balance())  # 150
```

Line by line:

1. `Account` groups balance data with operations that change it.
2. The initializer validates the opening balance before storing it.
3. `_balance` is marked non-public by convention. Callers should normally use the methods/properties provided by the class.
4. `deposit` validates the amount before changing state.
5. `get_balance` provides a controlled read operation.
6. The final lines create an account, deposit money, and read the result.

### Why a leading double underscore is not true privacy

```python
class Account:
    def __init__(self, balance):
        self.__balance = balance

    def get_balance(self):
        return self.__balance


account = Account(100)
print(account.get_balance())             # 100
# print(account.__balance)               # AttributeError
print(account._Account__balance)         # 100 — name-mangled, not inaccessible
```

`__balance` is transformed to `_Account__balance` by name mangling. This mainly prevents accidental attribute-name collisions in subclasses. Do not use it to protect secrets or enforce security.

### Properties: attribute syntax with validation

A property lets a class expose method logic using attribute syntax. It is useful when a value needs validation, computation, or controlled access.

```python
class Circle:
    def __init__(self, radius):
        self.radius = radius  # assignment calls the property setter

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if not isinstance(value, (int, float)) or isinstance(value, bool):
            raise TypeError("radius must be a number")
        if value < 0:
            raise ValueError("radius cannot be negative")
        self._radius = value

    @property
    def area(self):
        return 3.141592653589793 * self._radius ** 2


circle = Circle(5)
circle.radius = 10
print(circle.radius)  # 10
print(circle.area)    # 314.1592653589793
```

Line by line:

1. The initializer assigns through `self.radius`, so the same validation is applied during construction and later updates.
2. `@property` makes `radius` readable as `circle.radius`.
3. The getter returns the internal `_radius` value.
4. `@radius.setter` defines what happens when a caller writes `circle.radius = value`.
5. The type check rejects strings and booleans. In Python, `bool` is a subclass of `int`, which is why it is checked explicitly.
6. The range check rejects negative radii.
7. Only after validation does the setter update the internal value.
8. `area` is a read-only computed property: callers use `circle.area`, not `circle.area()`.

A property should normally be quick and unsurprising. Avoid hiding expensive I/O or major side effects behind simple attribute access.

---

# Part II — Inheritance and Polymorphism

## 7. Inheritance and Method Overriding

Inheritance lets a subclass reuse or specialize behavior from a base class. Use it when the relationship is genuinely an **is-a** relationship and the subclass can honor the base class's contract.

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "Some generic animal sound"


class Cat(Animal):
    def speak(self):
        return f"{self.name} says meow"


class Dog(Animal):
    def speak(self):
        return f"{self.name} says woof"


animals = [Cat("Whiskers"), Dog("Rex")]
for animal in animals:
    print(animal.speak())
```

Line by line:

1. `Animal` is the base class.
2. Its initializer stores the shared `name` attribute.
3. `speak` provides a default implementation.
4. `class Cat(Animal)` declares `Cat` as a subclass of `Animal`.
5. `Cat.speak` overrides the inherited method with cat-specific behavior.
6. `Dog` does the same for dog-specific behavior.
7. The list contains different object types that share the `speak` interface.
8. The loop calls the same method on each object; the runtime object determines which implementation runs.

This is **method overriding**: the subclass defines a method with the same name to specialize inherited behavior.

## 8. `super()` and Cooperative Inheritance

`super()` gives access to the next implementation in the method resolution order (MRO). It is often used to extend a parent initializer without duplicating its logic.

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary


class Manager(Employee):
    def __init__(self, name, salary, team_size):
        super().__init__(name, salary)  # run the next initializer in the MRO
        self.team_size = team_size


manager = Manager("Asha", 90000, 6)
print(manager.name, manager.salary, manager.team_size)
```

Line by line:

1. `Employee` defines fields common to employees.
2. `Manager(Employee)` inherits from `Employee`.
3. The manager initializer first calls `super().__init__(name, salary)` to initialize the inherited fields.
4. It then adds the manager-specific `team_size` field.
5. The instance therefore has all three attributes.

`super()` does not simply mean “call my direct parent.” More precisely, it follows the next class in the MRO. This matters with multiple inheritance.

### Cooperative multiple inheritance

When several classes participate in a hierarchy, each initializer should accept the arguments it needs, pass the remaining arguments onward, and call `super()` exactly once.

```python
class Base:
    def __init__(self, **kwargs):
        self.ready = True
        super().__init__(**kwargs)


class Named(Base):
    def __init__(self, *, name, **kwargs):
        self.name = name
        super().__init__(**kwargs)


class Timestamped(Base):
    def __init__(self, *, created_at, **kwargs):
        self.created_at = created_at
        super().__init__(**kwargs)


class Record(Named, Timestamped):
    def __init__(self, *, name, created_at):
        super().__init__(name=name, created_at=created_at)


record = Record(name="invoice-1", created_at="2026-01-01")
print(record.name, record.created_at, record.ready)
print([cls.__name__ for cls in Record.__mro__])
```

Line by line:

1. `Base` accepts leftover keyword arguments and initializes the shared base state.
2. `Named` consumes the keyword-only `name` argument.
3. `Named` forwards any unconsumed keyword arguments with `super()`.
4. `Timestamped` consumes `created_at` and forwards anything remaining.
5. `Record(Named, Timestamped)` combines the behaviors.
6. Its initializer forwards both required values into the cooperative chain.
7. Python follows `Record`'s MRO, so each participating initializer runs once in a consistent order.
8. The final print displays the values and the MRO.

This pattern requires all participating classes to cooperate. Mixing a class that does not call `super()` into the chain can interrupt initialization.

## 9. Multiple Inheritance and MRO

Python supports multiple inheritance: a class can inherit from more than one base class. The **method resolution order (MRO)** determines where Python looks for attributes and methods.

```python
class A:
    def greet(self):
        return "A"


class B:
    def greet(self):
        return "B"


class C(A, B):
    pass


obj = C()
print(obj.greet())  # A
print([cls.__name__ for cls in C.__mro__])  # ['C', 'A', 'B', 'object']
```

Line by line:

1. `A` defines `greet`.
2. `B` defines a method with the same name.
3. `C(A, B)` lists `A` before `B` in its bases.
4. When `obj.greet()` is evaluated, Python checks `C`, then `A`, then `B`, then `object`.
5. Since `A` defines `greet`, its method is selected.

### The diamond problem

```python
class A:
    def greet(self):
        return "A"


class B(A):
    def greet(self):
        return "B"


class C(A):
    def greet(self):
        return "C"


class D(B, C):
    pass


print(D().greet())  # B
print([cls.__name__ for cls in D.__mro__])  # ['D', 'B', 'C', 'A', 'object']
```

Both `B` and `C` inherit from `A`; `D` inherits from both. Python's C3 linearization algorithm creates a consistent MRO that preserves the declared ordering constraints and monotonicity. For this hierarchy, `B` is considered before `C`, and the common ancestor `A` appears once after both.

**Do not describe C3 as simply “left-to-right depth-first.”** That shortcut is misleading. C3 merges the MROs of the bases with the declared base-class order while preserving consistency. If Python cannot construct a consistent MRO, class creation raises `TypeError`.

Useful inspection tools:

```python
class A:
    pass


class B(A):
    pass


class C(A):
    pass


class D(B, C):
    pass


print(D.__mro__)   # tuple of classes in lookup order
print(D.mro())     # list form of the MRO
print(D.__bases__) # direct base classes only: (B, C)
```

## 10. Polymorphism and Duck Typing

**Polymorphism** means different object types can be used through a common interface. The caller focuses on what an object can do, rather than the exact class it belongs to.

```python
class Square:
    def __init__(self, side):
        self.side = side

    def area(self):
        return self.side ** 2


class Circle:
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return 3.141592653589793 * self.radius ** 2


def print_area(shape):
    print(shape.area())


shapes = [Square(4), Circle(3)]
for shape in shapes:
    print_area(shape)
```

Line by line:

1. `Square` and `Circle` are unrelated classes.
2. Each class provides an `area()` method.
3. `print_area` does not check the object's class. It assumes the object supports `area()`.
4. Both objects can be passed to the same function.
5. The method implementation used depends on the actual object.

This is also an example of **duck typing**: if an object provides the operations the code needs, it can be used. Explicit inheritance is not required.

A practical implication: avoid checking `type(value) is SomeClass` unless exact type identity is genuinely required. Prefer using the required behavior or `isinstance()` when nominal type checks are appropriate.

## 11. Structural Typing with `Protocol`

Duck typing is a runtime programming style. `typing.Protocol` lets you describe the required interface for static type checkers without forcing implementations to inherit from a shared base class.

```python
from typing import Protocol


class HasArea(Protocol):
    def area(self) -> float:
        ...


class Square:
    def __init__(self, side: float):
        self.side = side

    def area(self) -> float:
        return self.side * self.side


def display_area(shape: HasArea) -> None:
    print(shape.area())


display_area(Square(4))  # Square does not need to inherit from HasArea
```

Line by line:

1. `Protocol` is imported from `typing`.
2. `HasArea(Protocol)` declares the interface required by the consumer.
3. The `...` body indicates that the protocol describes a method signature rather than providing useful runtime behavior.
4. `Square` implements a matching `area()` method without inheriting from `HasArea`.
5. The function annotation says that `display_area` expects an object with the `HasArea` interface.
6. A static type checker can verify that `Square` satisfies the required shape.

By default, a protocol is primarily a static typing mechanism. `isinstance(obj, HasArea)` is not generally enabled unless the protocol is decorated with `@runtime_checkable`; even then, runtime checks only test attribute presence, not full signatures or behavior.

---

# Part III — Abstraction and Composition

## 12. Abstract Base Classes (`abc`)

An **abstract base class (ABC)** defines a common interface and can prevent incomplete subclasses from being instantiated.

Use an ABC when you want explicit nominal inheritance and runtime enforcement that abstract methods or properties are implemented.

```python
from abc import ABC, abstractmethod


class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount):
        """Process a payment."""
        raise NotImplementedError


class CreditCard(PaymentMethod):
    def pay(self, amount):
        return f"Paid {amount} using a credit card"


card = CreditCard()
print(card.pay(100))
# PaymentMethod()  # TypeError: abstract class cannot be instantiated
```

Line by line:

1. `ABC` is the helper base class for defining an abstract base class.
2. `abstractmethod` marks a method that concrete subclasses must implement.
3. `PaymentMethod(ABC)` declares the abstract base class.
4. `pay` describes the required operation.
5. `CreditCard` inherits the contract.
6. It implements `pay`, so it can be instantiated.
7. Attempting to instantiate `PaymentMethod` directly raises `TypeError`.

### Abstract properties

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    @property
    @abstractmethod
    def area(self):
        """Return the area of the shape."""
        raise NotImplementedError


class Square(Shape):
    def __init__(self, side):
        self.side = side

    @property
    def area(self):
        return self.side ** 2


square = Square(4)
print(square.area)  # 16
```

`@property` is the outer decorator and `@abstractmethod` marks the property getter as abstract. A concrete subclass implements a property with the same interface. The caller reads `square.area` like an attribute.

### Virtual subclasses with `register()`

An ABC can register an unrelated class as a **virtual subclass**. This affects `isinstance()` and `issubclass()` checks, but it does not add methods to the registered class or insert the ABC into its MRO.

```python
from abc import ABC, abstractmethod


class Flyer(ABC):
    @abstractmethod
    def fly(self):
        raise NotImplementedError


class BirdLike:
    def fly(self):
        return "Flying"


Flyer.register(BirdLike)

print(issubclass(BirdLike, Flyer))  # True
print(isinstance(BirdLike(), Flyer))  # True
print(Flyer in BirdLike.__mro__)  # False
```

Important: `register()` does not verify that `BirdLike` actually implements `fly`. It is a registration promise made by the programmer, not an implementation check. `typing.Protocol` is often a better fit when describing structural interfaces for static type checking.

## 13. Composition, Dependency Injection, and Delegation

**Composition** means an object stores references to other objects and uses them to do work. It models a **has-a** or **uses-a** relationship.

```python
class Engine:
    def start(self):
        return "Engine started"


class Car:
    def __init__(self, engine):
        self.engine = engine  # Car stores an Engine-like object

    def start(self):
        return self.engine.start()  # delegates the work to the engine


car = Car(Engine())
print(car.start())  # Engine started
```

Line by line:

1. `Engine` defines a component with a `start()` operation.
2. `Car.__init__` receives an engine object from the caller.
3. `self.engine = engine` stores that object as part of the car's state.
4. `Car.start` delegates the operation to the component.
5. The caller interacts with the car's simpler public interface.

### Dependency injection: choose the implementation outside the class

```python
class ElectricEngine:
    def start(self):
        return "Silent electric hum"


class GasEngine:
    def start(self):
        return "Vroom!"


class Car:
    def __init__(self, engine):
        self.engine = engine

    def start(self):
        return self.engine.start()


car_a = Car(ElectricEngine())
car_b = Car(GasEngine())

print(car_a.start())  # Silent electric hum
print(car_b.start())  # Vroom!
```

`Car` does not need to know the concrete engine class. The caller injects the dependency. This makes the class easier to reuse and test.

The same design is useful for databases, file storage, payment providers, notification senders, and API clients. A test can inject a lightweight fake object instead of contacting a real external service.

### Strategy pattern

When an object delegates an operation to a replaceable behavior object, the design is a simple form of the **Strategy pattern**.

```python
class CashDiscount:
    def apply(self, total):
        return total * 0.95


class NoDiscount:
    def apply(self, total):
        return total


class Checkout:
    def __init__(self, pricing_strategy):
        self.pricing_strategy = pricing_strategy

    def final_total(self, total):
        return self.pricing_strategy.apply(total)


checkout = Checkout(CashDiscount())
print(checkout.final_total(100))  # 95.0

checkout.pricing_strategy = NoDiscount()  # swap behavior at runtime
print(checkout.final_total(100))  # 100
```

The strategy object is replaceable without changing the `Checkout` class. The strategy classes need not inherit from a common parent as long as they provide the expected `apply()` method.

## 14. Aggregation vs Composition

These terms are often used in UML and object-oriented design:

- **Composition** describes a strong whole-part relationship. The whole conceptually owns or controls the part's lifecycle.
- **Aggregation** describes a looser association. The related objects can exist independently.

```python
class Engine:
    pass


class Car:
    def __init__(self):
        self.engine = Engine()  # created as an implementation detail of this Car


class Driver:
    def __init__(self, name):
        self.name = name


class Taxi:
    def __init__(self, driver):
        self.driver = driver  # receives an independently created Driver


car = Car()
driver = Driver("Asha")
taxi = Taxi(driver)
```

The `Car` creates its engine internally, while `Taxi` receives an existing driver. These examples illustrate the design distinction, but Python does not enforce ownership or object lifetimes. In Python, references and garbage collection determine when objects can be reclaimed; an engine object can technically outlive a car if another reference to it exists.

## 15. Mixins

A **mixin** is a small class intended to add one focused behavior to another class through inheritance. A mixin is usually not a complete standalone object.

```python
import json


class JSONExportMixin:
    def to_json(self):
        return json.dumps(self.to_dict())


class User(JSONExportMixin):
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def to_dict(self):
        return {"name": self.name, "age": self.age}


user = User("Ana", 30)
print(user.to_json())  # {"name": "Ana", "age": 30}
```

Line by line:

1. `json` is a standard-library module for JSON serialization.
2. `JSONExportMixin` adds a `to_json()` capability.
3. The mixin calls `self.to_dict()` rather than serializing every value in `__dict__`. This makes the data explicitly controlled by the concrete class.
4. `User` inherits the mixin and provides the required `to_dict()` method.
5. A `User` instance can now call `to_json()`.

Good mixins are small, focused, and have clearly documented expectations. Avoid mixins that depend on many undocumented attributes or create complicated initialization chains.

---

# Part IV — Making Classes Behave Like Native Python Objects

## 16. Dunder Methods and Object Representations

Dunder means **double underscore**. Special methods such as `__repr__` and `__len__` let user-defined objects participate in Python syntax and built-in operations.

Do not call special methods directly in normal code when a built-in exists. Prefer `len(obj)` over `obj.__len__()`, `str(obj)` over `obj.__str__()`, and `a + b` over `a.__add__(b)`.

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return f"User(name={self.name!r}, age={self.age!r})"

    def __str__(self):
        return f"{self.name} ({self.age})"


user = User("Asha", 30)
print(repr(user))  # User(name='Asha', age=30)
print(str(user))   # Asha (30)
print(user)        # Asha (30), because print uses str()
```

- `__repr__` should aim for an unambiguous, developer-friendly representation. When practical, it should look like a valid expression that could recreate the object.
- `__str__` is the friendlier display form for users.
- If `__str__` is not defined, Python falls back to `__repr__` when available.

### Common special methods

| Method | Used by / purpose |
|---|---|
| `__new__` | Creates an instance. |
| `__init__` | Initializes an instance. |
| `__repr__`, `__str__` | `repr(obj)`, `str(obj)`, and display. |
| `__bool__` | `bool(obj)` and truth-value testing. |
| `__len__` | `len(obj)`; also used for truth testing if `__bool__` is absent. |
| `__eq__`, `__lt__`, etc. | Equality and ordering comparisons. |
| `__hash__` | Hashing for dictionaries and sets. |
| `__add__`, `__sub__`, `__mul__` | Arithmetic operators. |
| `__getitem__`, `__setitem__`, `__delitem__` | Subscription such as `obj[key]`. |
| `__iter__`, `__next__` | Iteration. |
| `__contains__` | Membership such as `item in obj`. |
| `__enter__`, `__exit__` | Synchronous `with` statement. |
| `__call__` | Makes an instance callable: `obj(...)`. |
| `__get__`, `__set__`, `__delete__` | Descriptor protocol. |

### A callable object

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, value):
        return self.factor * value


triple = Multiplier(3)
print(triple(10))  # 30
```

`__call__` makes `triple(10)` behave like calling a function. This is useful for small stateful function-like objects, callbacks, and configurable operations.

## 17. Operator Overloading and `NotImplemented`

Operator overloading lets a class define how operators such as `+` behave for its instances.

A correct implementation should:

- Handle supported operand types.
- Return `NotImplemented` for unsupported operand types, allowing Python to try the other operand's reflected method.
- Avoid pretending unrelated objects can be added.

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        if not isinstance(other, Vector):
            return NotImplemented
        return Vector(self.x + other.x, self.y + other.y)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"


v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)  # Vector(4, 6)
```

Line by line:

1. `Vector` stores two coordinates.
2. `__add__` is called for `left + right` when the left operand's type supports the operation.
3. `isinstance` checks whether the other object is also a `Vector` (including a subclass).
4. `NotImplemented` is a special return value that tells Python this method does not support the operand combination. It is not the same as raising `NotImplementedError`.
5. For two vectors, a new vector is returned with coordinate-wise sums.
6. `__repr__` makes the result easy to inspect.

### Reflected operators

For `left + right`, Python normally tries the left operand's `__add__`. If that cannot handle the operation, Python may try the right operand's `__radd__`.

This example intentionally supports adding a number to a vector in either order:

```python
class Vector:
    def __init__(self, x, y=0):
        self.x = x
        self.y = y

    def __add__(self, other):
        if isinstance(other, Vector):
            return Vector(self.x + other.x, self.y + other.y)
        if isinstance(other, (int, float)):
            return Vector(self.x + other, self.y)
        return NotImplemented

    def __radd__(self, other):
        # For number + vector, reuse the number + vector rule.
        return self.__add__(other)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"


print(Vector(1, 2) + Vector(3, 4))  # Vector(4, 6)
print(Vector(1, 2) + 5)             # Vector(6, 2)
print(5 + Vector(1, 2))             # Vector(6, 2)
```

Why the third expression works:

1. Python first asks `int.__add__` to add `5` and the vector.
2. The integer implementation does not know how to add this vector and returns `NotImplemented`.
3. Python then tries `Vector.__radd__(5)`.
4. `__radd__` delegates to the vector's supported number-addition logic.
5. A new `Vector(6, 2)` is returned.

The earlier common mistake is to write `__radd__` but have it call a `__add__` that only accepts vectors. That still returns `NotImplemented` for `5 + vector`, so the operation fails.

### In-place operators

`+=` may call `__iadd__`. If the type does not implement it, Python may fall back to ordinary addition and assignment (`x = x + y`). Whether an operation mutates the original object depends on its implementation and type.

```python
class NumberBox:
    def __init__(self, value):
        self.value = value

    def __iadd__(self, amount):
        self.value += amount
        return self


box = NumberBox(10)
box += 5
print(box.value)  # 15
```

The method mutates the existing object and returns it. For immutable types, an in-place operation commonly creates and returns a new value instead.

## 18. Equality, Ordering, and Hashing

`==` calls equality behavior, typically `__eq__`. `is` checks identity: whether two references point to the exact same object.

```python
class User:
    def __init__(self, user_id):
        self.user_id = user_id

    def __eq__(self, other):
        if not isinstance(other, User):
            return NotImplemented
        return self.user_id == other.user_id


first = User(10)
second = User(10)
third = first

print(first == second)  # True: same logical ID
print(first is second)  # False: different instances
print(first is third)   # True: same instance
```

### The `__eq__` and `__hash__` contract

If a class defines `__eq__` but does not define a compatible `__hash__`, Python normally makes instances unhashable. This prevents objects with value-based equality from accidentally violating dictionary/set requirements.

The rule is: **if two objects compare equal, they must have equal hash values**. Hash values do not need to be unique for unequal objects.

```python
class User:
    def __init__(self, user_id):
        self.user_id = user_id

    def __eq__(self, other):
        if not isinstance(other, User):
            return NotImplemented
        return self.user_id == other.user_id

    def __hash__(self):
        return hash(self.user_id)


users = {User(1), User(1), User(2)}
print(len(users))  # 2
```

Only define `__hash__` if the fields used for equality are stable while the object is in a set or used as a dictionary key. Mutating a hash-relevant field after insertion can make the object difficult to find in the collection.

### Ordering and `functools.total_ordering`

```python
from functools import total_ordering


@total_ordering
class Money:
    def __init__(self, amount):
        self.amount = amount

    def __eq__(self, other):
        if not isinstance(other, Money):
            return NotImplemented
        return self.amount == other.amount

    def __lt__(self, other):
        if not isinstance(other, Money):
            return NotImplemented
        return self.amount < other.amount


print(Money(5) < Money(10))   # True
print(Money(5) <= Money(5))   # True, generated by total_ordering
print(Money(5) >= Money(10))  # False, generated by total_ordering
```

`@total_ordering` fills in the other ordering methods when the class supplies `__eq__` and one of `__lt__`, `__le__`, `__gt__`, or `__ge__`. It reduces boilerplate but can make comparisons slightly slower and less direct than implementing all operations yourself.

## 19. Iteration and Container Protocols

An **iterable** is an object you can get an iterator from. An **iterator** is the object that produces values one at a time and remembers its current position.

- An iterable implements `__iter__()` and returns an iterator.
- An iterator implements `__iter__()` returning itself and `__next__()` returning the next item.
- When no values remain, `__next__()` raises `StopIteration`.

### Implementing an iterable collection

```python
class Countdown:
    def __init__(self, start):
        self.start = start

    def __iter__(self):
        current = self.start
        while current > 0:
            yield current
            current -= 1


countdown = Countdown(3)
print(list(countdown))  # [3, 2, 1]
print(list(countdown))  # [3, 2, 1] — a fresh iteration starts from the beginning
```

Line by line:

1. The constructor saves the starting number, but does not store a changing iteration position on the collection itself.
2. `__iter__` is a generator function because it contains `yield`.
3. Each call to `iter(countdown)` creates a fresh generator with its own local `current` variable.
4. The loop yields the current value, suspending the generator until the next value is requested.
5. The counter decreases after each yield.
6. When the condition becomes false, the generator ends and raises `StopIteration` automatically.
7. `list(...)` repeatedly asks for values until iteration ends.

### A direct iterator implementation

```python
class CountdownIterator:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        value = self.current
        self.current -= 1
        return value


iterator = CountdownIterator(3)
print(next(iterator))  # 3
print(next(iterator))  # 2
print(next(iterator))  # 1
# next(iterator)       # StopIteration
```

A direct iterator is stateful and is normally exhausted after one pass. By contrast, an iterable collection can return a fresh iterator each time. This distinction matters when designing reusable containers.

### Useful container methods

```python
class Playlist:
    def __init__(self, songs):
        self._songs = list(songs)

    def __len__(self):
        return len(self._songs)

    def __iter__(self):
        return iter(self._songs)

    def __getitem__(self, index):
        return self._songs[index]

    def __contains__(self, song):
        return song in self._songs


playlist = Playlist(["Song A", "Song B"])
print(len(playlist))           # 2
print(playlist[0])             # Song A
print("Song B" in playlist)    # True
print(list(playlist))          # ['Song A', 'Song B']
```

The special methods connect the class to built-ins: `len`, iteration, indexing, and membership. Define only the protocols that match the meaning of your object.

## 20. Context Managers

A context manager controls setup and cleanup around a block of code. The `with` statement is commonly used for files, locks, database transactions, and other resources.

### Class-based context manager

```python
class ManagedFile:
    def __init__(self, path):
        self.path = path
        self.file = None

    def __enter__(self):
        self.file = open(self.path, "w", encoding="utf-8")
        return self.file

    def __exit__(self, exc_type, exc_value, traceback):
        if self.file is not None:
            self.file.close()
        return False  # do not suppress exceptions


with ManagedFile("oop-example.txt") as file:
    file.write("Hello from a context manager.\n")
```

Line by line:

1. The initializer stores the file path and starts with no open file.
2. `__enter__` runs when execution enters the `with` block.
3. It opens the file and returns the file object that is assigned to `file` after `as`.
4. `__exit__` runs when the block exits, including when an exception occurs inside the block.
5. The file is closed if it was opened.
6. Returning `False` means an exception is not suppressed; it will continue propagating.

For ordinary file work, prefer Python's built-in `open()` context manager directly:

```python
with open("oop-example.txt", "w", encoding="utf-8") as file:
    file.write("The built-in file object is already a context manager.\n")
```

### Function-based context managers with `contextlib`

```python
from contextlib import contextmanager


@contextmanager
def managed_value():
    print("setup")
    try:
        yield "resource"
    finally:
        print("cleanup")


with managed_value() as value:
    print(value)
```

`@contextmanager` turns a generator function into a context manager. Code before `yield` is setup; the yielded value is bound after `as`; code in `finally` runs during cleanup even if the block raises an exception.

## 21. Dataclasses

The `dataclasses` module reduces boilerplate for classes mainly used to store data. A dataclass can generate methods such as `__init__`, `__repr__`, and `__eq__` from annotated fields.

```python
from dataclasses import dataclass


@dataclass
class Point:
    x: int
    y: int


point = Point(3, 4)
print(point)  # Point(x=3, y=4)
print(point == Point(3, 4))  # True
```

Line by line:

1. Import `dataclass`.
2. `@dataclass` decorates the class and generates common data-oriented methods.
3. `x: int` and `y: int` declare fields with type annotations. The annotations are useful to readers and type checkers; Python does not automatically enforce them at runtime.
4. The generated initializer accepts `x` and `y`.
5. The generated representation displays field names and values.
6. The generated equality compares instances of the same dataclass type by their fields.

### Mutable fields: always use a factory

```python
from dataclasses import dataclass, field


@dataclass
class Cart:
    items: list[str] = field(default_factory=list)


first = Cart()
second = Cart()
first.items.append("book")

print(first.items)   # ['book']
print(second.items)  # [] — a separate list
```

`default_factory=list` tells the dataclass to call `list()` for each new instance. Do not use a shared mutable object as the default for per-instance state.

### Frozen and ordered dataclasses

```python
from dataclasses import dataclass


@dataclass(frozen=True, order=True)
class Coordinate:
    x: int
    y: int


first = Coordinate(1, 2)
second = Coordinate(1, 3)
print(first < second)  # True: fields are compared in declaration order
# first.x = 10          # FrozenInstanceError
```

- `frozen=True` prevents normal attribute assignment and deletion; it emulates immutability rather than making Python objects absolutely immutable.
- `order=True` generates ordering methods based on field order.
- Dataclass ordering generally requires both objects to be of the same dataclass type.

### `__post_init__` for validation or derived fields

```python
from dataclasses import dataclass


@dataclass
class Rectangle:
    width: float
    height: float

    def __post_init__(self):
        if self.width <= 0 or self.height <= 0:
            raise ValueError("Width and height must be positive")

    @property
    def area(self):
        return self.width * self.height


rectangle = Rectangle(4, 5)
print(rectangle.area)  # 20
```

The generated `__init__` assigns fields and then calls `__post_init__`. Use this hook for validation or derived initialization that belongs to the dataclass.

## 22. `__slots__`

By default, ordinary instances usually have an instance dictionary (`__dict__`) where attributes are stored. `__slots__` declares a fixed set of instance attributes and can reduce per-instance memory use in suitable classes. It also prevents adding arbitrary new attributes unless a dictionary is otherwise available through inheritance or explicitly included.

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x = x
        self.y = y


point = Point(2, 3)
print(point.x, point.y)  # 2 3
# point.z = 4             # AttributeError: no slot named z
# print(point.__dict__)   # AttributeError: no instance dictionary
```

Important limitations:

- A base class with a normal instance dictionary can make that dictionary available to subclasses.
- A subclass that does not define slots may regain an instance dictionary.
- Multiple inheritance involving different slotted layouts has restrictions.
- Slots can affect weak references unless a `__weakref__` slot is provided where needed.
- Use slots because profiling shows a benefit or because a fixed attribute layout is useful, not as a default requirement for every class.

Dataclasses also support `@dataclass(slots=True)` in Python 3.10+.

---

# Part V — Design and Error Handling

## 23. Custom Exceptions

Custom exceptions make errors from your application or library easier to understand and catch. Custom exception classes should usually inherit from `Exception`, not directly from `BaseException`.

```python
class AppError(Exception):
    """Base class for application-specific errors."""


class InsufficientFundsError(AppError):
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        super().__init__(f"Cannot withdraw {amount}; balance is {balance}")


class Account:
    def __init__(self, balance):
        self.balance = balance

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self.balance:
            raise InsufficientFundsError(self.balance, amount)
        self.balance -= amount


try:
    Account(50).withdraw(100)
except InsufficientFundsError as error:
    print(error)  # Cannot withdraw 100; balance is 50
```

Line by line:

1. `AppError` is a common base class for errors raised by this application.
2. `InsufficientFundsError` specializes that error category.
3. The initializer stores useful structured context on the exception.
4. `super().__init__(message)` initializes the base `Exception` with a readable message.
5. `withdraw` validates the amount and then checks the account balance.
6. `raise` creates and raises the specific exception when the balance is insufficient.
7. `try` marks code that may fail.
8. `except InsufficientFundsError` handles only this specific error; other exceptions are not accidentally hidden.

A useful exception hierarchy often looks like `AppError` → `InputError`, `NotFoundError`, `PermissionError`, etc. Keep exception classes focused and provide enough context for callers to handle or report the problem.

## 24. SOLID Principles

SOLID is a set of design guidelines, not a Python language feature or a checklist that every small class must satisfy.

| Principle | Meaning | Python-oriented example |
|---|---|---|
| **S — Single Responsibility** | A class should have one cohesive responsibility and one main reason to change. | Separate invoice calculation from email delivery. |
| **O — Open/Closed** | Prefer designs that allow new behavior to be added without repeatedly changing stable, tested code. | Inject a new pricing strategy instead of adding many conditionals to checkout. |
| **L — Liskov Substitution** | A subtype should preserve the expectations of the base type. | A `Square` should not violate a `Rectangle` API's promised independent width/height behavior. |
| **I — Interface Segregation** | Prefer small interfaces focused on the needs of their clients. | Use `Readable` and `Writable` protocols instead of one oversized interface. |
| **D — Dependency Inversion** | High-level policy should depend on abstractions, not hard-coded low-level details. | A service accepts a storage interface rather than creating a concrete database client internally. |

### Dependency inversion example

```python
from typing import Protocol


class Notifier(Protocol):
    def send(self, message: str) -> None:
        ...


class AlertService:
    def __init__(self, notifier: Notifier):
        self.notifier = notifier

    def alert(self, message: str) -> None:
        self.notifier.send(message)


class ConsoleNotifier:
    def send(self, message: str) -> None:
        print(f"ALERT: {message}")


service = AlertService(ConsoleNotifier())
service.alert("Disk space is low")
```

`AlertService` depends on the behavior `send(message)` rather than on a specific notifier class. A different notifier can be injected without changing the alert service.

## 25. Choosing Inheritance, Composition, ABCs, and Protocols

| Need | Usually consider | Why |
|---|---|---|
| A true, stable **is-a** relationship with shared behavior | Inheritance | The subtype can reuse and specialize the base implementation. |
| An object needs another object to perform work | Composition | Dependencies can be replaced and tested independently. |
| Runtime-enforced abstract contract and nominal family | ABC | Subclasses must implement abstract members before instantiation. |
| Shared interface for unrelated classes, especially for static typing | `Protocol` | Implementations do not need to inherit from the protocol. |
| One small behavior reused across class hierarchies | Mixin | Adds a focused capability through inheritance. |

A practical rule is to prefer the simplest design that communicates the relationship. Do not introduce inheritance, ABCs, descriptors, or metaclasses just to make a solution look more object-oriented.

---

# Part VI — Python's Object Model

The topics in this part explain mechanisms that power Python classes. They are useful for debugging frameworks and understanding advanced behavior, but most application code does not need to implement them directly.

## 26. `__new__` vs `__init__`

- `__new__(cls, ...)` creates and returns an instance. It is commonly used for immutable types or unusual object-creation control.
- `__init__(self, ...)` initializes an already-created instance and must return `None`.
- Python calls `__init__` after `__new__` only when `__new__` returns an instance of the class (or an appropriate subclass).

```python
class Example:
    def __new__(cls, value):
        print("1. __new__: creating object")
        instance = super().__new__(cls)
        return instance

    def __init__(self, value):
        print("2. __init__: initializing object")
        self.value = value


item = Example(10)
print(item.value)
```

Expected output:

```text
1. __new__: creating object
2. __init__: initializing object
10
```

Line by line:

1. Calling `Example(10)` starts the object construction process.
2. `__new__` receives the class as `cls`.
3. `super().__new__(cls)` delegates allocation to the base implementation.
4. The new instance is returned.
5. Python then calls `__init__` on that instance.
6. The initializer stores the value.

### Why overriding `__new__` is uncommon

Most classes can use the default `object.__new__`. Override `__new__` only when controlling creation itself is necessary, such as for some immutable subclasses. A singleton implemented with `__new__` is possible, but it has initialization, testing, and thread-safety complications; dependency injection is often simpler.

## 27. Attribute Lookup

When you write `obj.attribute`, Python performs attribute lookup. A useful simplified model for ordinary instance access is:

1. A **data descriptor** on the class or its MRO.
2. The instance's `__dict__`, if it has one.
3. A **non-data descriptor** or ordinary class attribute on the class or its MRO.
4. `__getattr__`, if normal lookup fails and the class defines it.

This is a simplified view of the normal object attribute path; custom `__getattribute__`, metaclasses, and special cases can change details.

```python
class Demo:
    class_value = "stored on class"

    def __init__(self):
        self.instance_value = "stored on instance"


demo = Demo()
print(demo.instance_value)  # found in demo.__dict__
print(demo.class_value)     # found on Demo after instance lookup misses
```

### `__getattribute__` vs `__getattr__`

- `__getattribute__` is invoked for normal attribute access and controls the lookup operation. Override it rarely, and delegate to `object.__getattribute__` to avoid recursion.
- `__getattr__` is a fallback called when ordinary lookup raises `AttributeError`.

```python
class LazyValues:
    def __getattr__(self, name):
        print(f"Computing missing attribute: {name}")
        value = name.upper()
        setattr(self, name, value)  # cache the result on the instance
        return value


values = LazyValues()
print(values.status)  # computes and prints STATUS
print(values.status)  # reads cached instance attribute; no second message
```

`__getattr__` is useful for proxies and lazy values, but using it for every normal field can make classes harder to understand and debug.

## 28. Descriptors

A **descriptor** is an object whose class defines one or more of these methods:

- `__get__(self, instance, owner)`
- `__set__(self, instance, value)`
- `__delete__(self, instance)`

A descriptor is a **data descriptor** if its type defines `__set__` and/or `__delete__`. Otherwise, if it defines `__get__`, it is a **non-data descriptor**. Data descriptors take precedence over an instance dictionary; non-data descriptors can be overridden by an instance attribute.

Descriptors are used by properties, methods, `classmethod`, `staticmethod`, and slotted attributes.

### Non-data descriptor

```python
class NonDataDescriptor:
    def __get__(self, instance, owner):
        return "value from descriptor"


class Example:
    value = NonDataDescriptor()


obj = Example()
print(obj.value)  # value from descriptor
obj.__dict__["value"] = "value from instance"
print(obj.value)  # value from instance
```

The descriptor only has `__get__`, so it is non-data. The instance dictionary can shadow it.

### Data descriptor

```python
class DataDescriptor:
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance._value

    def __set__(self, instance, value):
        instance._value = value


class Example:
    value = DataDescriptor()

    def __init__(self, value):
        self.value = value  # calls DataDescriptor.__set__


obj = Example(10)
obj.__dict__["value"] = "shadow attempt"
print(obj.value)  # 10 — data descriptor wins
```

The `instance is None` branch handles access through the class, such as `Example.value`. Returning `self` is a common choice for custom descriptors.

### Building a small property-like descriptor

```python
class ValidatedNumber:
    def __set_name__(self, owner, name):
        self.public_name = name
        self.storage_name = "_" + name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return getattr(instance, self.storage_name)

    def __set__(self, instance, value):
        if not isinstance(value, (int, float)) or isinstance(value, bool):
            raise TypeError(f"{self.public_name} must be numeric")
        if value < 0:
            raise ValueError(f"{self.public_name} cannot be negative")
        setattr(instance, self.storage_name, value)


class Product:
    price = ValidatedNumber()
    quantity = ValidatedNumber()

    def __init__(self, price, quantity):
        self.price = price
        self.quantity = quantity


product = Product(10, 5)
print(Product.price)  # descriptor object
print(product.price)  # 10
# product.price = -1  # ValueError: price cannot be negative
```

Line by line:

1. `__set_name__` is called automatically when Python creates the owner class and assigns the descriptor to an attribute.
2. It remembers both the public attribute name and the private storage attribute name.
3. `__get__` receives `instance=None` when the attribute is accessed through the class; returning the descriptor itself makes class-level inspection possible.
4. When accessed through an instance, the getter reads the stored value.
5. `__set__` validates the incoming value before saving it.
6. `Product.price` returns the descriptor; `product.price` returns the instance's stored value.

Descriptors are powerful when the same validation or attribute behavior must be reused across many classes or fields. For one or two ordinary attributes, `@property` is usually simpler.

## 29. `__init_subclass__`

`__init_subclass__` is a class-creation hook called when a subclass is created. It often provides a simpler alternative to a metaclass for subclass registration or validation.

```python
class Plugin:
    registry = []

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry.append(cls)


class AudioPlugin(Plugin):
    pass


class VideoPlugin(Plugin):
    pass


print([plugin.__name__ for plugin in Plugin.registry])
# ['AudioPlugin', 'VideoPlugin']
```

Line by line:

1. `Plugin.registry` is shared class-level storage for discovered subclasses.
2. Python calls `Plugin.__init_subclass__` while creating each direct or indirect subclass.
3. `super().__init_subclass__(**kwargs)` allows other classes in the MRO to participate in class initialization.
4. `cls` is the newly created subclass, not an instance.
5. The class is appended to the registry.
6. Defining `AudioPlugin` and `VideoPlugin` automatically registers them.

For production registries, consider abstract base classes, duplicate registration, import order, and whether abstract intermediate classes should be included.

## 30. Metaclasses

A **metaclass** is the class used to create a class. Most Python classes use `type` as their metaclass. A metaclass can customize class creation, but it is advanced and should be used only when simpler tools such as class decorators or `__init_subclass__` are insufficient.

```python
class RequiredFieldMeta(type):
    def __new__(mcls, name, bases, namespace):
        if name != "BaseRecord" and "record_id" not in namespace:
            raise TypeError("Classes must define record_id")
        return super().__new__(mcls, name, bases, namespace)


class BaseRecord(metaclass=RequiredFieldMeta):
    pass


class UserRecord(BaseRecord):
    record_id = "user"


print(UserRecord.record_id)  # user
```

Line by line:

1. `RequiredFieldMeta` inherits from `type`, the usual metaclass.
2. Its `__new__` receives the metaclass, class name, base classes, and the prepared class namespace.
3. The validation is skipped for the base class itself.
4. A concrete subclass must define `record_id` directly in its class body.
5. If the requirement is not met, raising `TypeError` stops class creation.
6. Otherwise `super().__new__` creates the class object.
7. `UserRecord` meets the requirement and is created successfully.

A metaclass can make frameworks expressive, but it also adds complexity and can conflict with other metaclasses. Prefer a class decorator or `__init_subclass__` when they solve the problem.

---

# Appendices

## Common Built-ins and Standard-Library Tools

| Tool | Purpose | Example |
|---|---|---|
| `type(obj)` | Get an object's exact type. | `type(user)` |
| `isinstance(obj, Class)` | Check whether an object is an instance of a class or subclass. | `isinstance(user, User)` |
| `issubclass(A, B)` | Check whether class `A` derives from `B` or is registered as its virtual subclass. | `issubclass(Admin, User)` |
| `super()` | Continue lookup in the MRO. | `super().__init__()` |
| `getattr(obj, name[, default])` | Read an attribute by name. | `getattr(user, "name", "Unknown")` |
| `setattr(obj, name, value)` | Assign an attribute by name. | `setattr(user, "active", True)` |
| `hasattr(obj, name)` | Check for an attribute. | `hasattr(user, "name")` |
| `vars(obj)` | Return the object's `__dict__`, when available. | `vars(user)` |
| `dir(obj)` | List names available on an object. | `dir(user)` |
| `callable(obj)` | Check whether an object can be called. | `callable(user)` |
| `id(obj)` | Get an identity value for the object's lifetime. | `id(user)` |
| `repr(obj)`, `str(obj)` | Get representations. | `repr(user)` |
| `len(obj)`, `iter(obj)`, `next(it)` | Work with collection and iterator protocols. | `len(playlist)` |

Common modules:

| Module | Useful types / decorators | When to use it |
|---|---|---|
| `abc` | `ABC`, `abstractmethod` | Abstract base classes and virtual subclass registration. |
| `typing` | `Protocol`, `TypedDict`, type aliases | Type annotations and structural interfaces for static checking. |
| `dataclasses` | `dataclass`, `field`, `asdict`, `replace` | Data-holding classes with generated boilerplate. |
| `functools` | `total_ordering`, `cached_property`, `singledispatchmethod` | Reusable decorators and dispatch helpers. |
| `contextlib` | `contextmanager`, `ExitStack`, `suppress` | Building and combining context managers. |
| `collections.abc` | `Iterable`, `Iterator`, `Sequence`, `Mapping` | Standard abstract interfaces for collection behavior. |
| `enum` | `Enum`, `IntEnum`, `StrEnum` (Python 3.11+) | Represent a fixed set of named values. |

## 1. Patterns in Python Are Different

The classic patterns were described for languages like C++ and Java, where everything must be a class and functions can't be passed around. Python has first-class functions, duck typing, modules, generators, decorators and context managers, and those language features make many patterns shrink to a few lines or disappear.

| Classic pattern | Often just this in Python |
|---|---|
| Singleton | A module-level object (modules are imported once), or `functools.cache` on a getter |
| Factory Method / Abstract Factory | A function, a class stored in a variable, or a `dict` registry |
| Builder | Keyword arguments or a `dataclass` (a fluent builder only when construction is truly step-by-step) |
| Prototype | `copy.copy()` and `copy.deepcopy()` |
| Strategy | Passing a function |
| Command | A callable, or `functools.partial` |
| Iterator | A generator |
| Decorator | `@decorator` syntax for functions (wrapper classes for objects) |
| Observer | A list of callbacks |
| Visitor | `functools.singledispatch` |
| Proxy | `__getattr__`, `weakref.proxy` |
| Flyweight | Interning, `lru_cache`, `__slots__` |
| Adapter | A thin wrapper class, `functools.partial`, or a lambda |
| Facade | A module or class exposing a small API |

**The rule of thumb:** use the simple Pythonic form first. Reach for the full class-based pattern when you need what it adds: several operations per object, state, a shared interface, or a place for subclasses to plug in.

### Three categories

- **Creational:** how objects get created (Singleton, Factory Method, Abstract Factory, Builder, Prototype, Object Pool).
- **Structural:** how objects are composed into larger structures (Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy).
- **Behavioral:** how objects communicate and share responsibility (Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor, Interpreter).

---

## 2. The Principles Behind the Patterns (SOLID)

Patterns are applications of a few design principles. If you remember the principles you can often re-derive the pattern.

| Principle | Meaning | Patterns that serve it |
|---|---|---|
| **S**ingle Responsibility | A class has one reason to change | Facade, Strategy, Command |
| **O**pen/Closed | Open for extension, closed for modification | Strategy, Decorator, Template Method, Visitor |
| **L**iskov Substitution | Subtypes must be usable wherever the base type is | Factory Method, Adapter, State |
| **I**nterface Segregation | Prefer small, focused interfaces | Adapter, Bridge, `Protocol` |
| **D**ependency Inversion | Depend on abstractions, not concrete classes | Factory, Dependency Injection, Strategy |

Two lines of thought run through almost every pattern: **program to an interface, not an implementation**, and **favor composition over inheritance**. In Python an "interface" is usually a `typing.Protocol` (see `15-Protocals.md`) or an `abc.ABC` (see `11-Object-Oriented-Program.md`), and duck typing means you often need neither.

Here is Dependency Inversion and Open/Closed in one small example. `SignupService` depends on the idea of a notifier, not on email or SMS, so you can add a new channel without touching it:

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, to: str, text: str) -> None: ...

class EmailNotifier:
    def send(self, to, text):
        print(f"EMAIL to {to}: {text}")

class SmsNotifier:
    def send(self, to, text):
        print(f"SMS to {to}: {text}")

class SignupService:
    def __init__(self, notifier: Notifier):         # depends on the abstraction
        self.notifier = notifier

    def register(self, user):
        print(f"registered {user}")
        self.notifier.send(user, "Welcome!")

SignupService(EmailNotifier()).register("ana@example.com")
SignupService(SmsNotifier()).register("+91-98765")      # new behavior, SignupService unchanged
```

**Output:**

```text
registered ana@example.com
EMAIL to ana@example.com: Welcome!
registered +91-98765
SMS to +91-98765: Welcome!
```

---

## 3. Choosing a Pattern: Start From the Problem

| You need to... | Consider |
|---|---|
| Have exactly one shared instance | Singleton (or just a module) |
| Let subclasses or config decide which class to create | Factory Method |
| Create families of related objects that must match | Abstract Factory |
| Build a complex object step by step | Builder |
| Create objects by copying an existing one | Prototype |
| Reuse expensive objects instead of creating them | Object Pool |
| Make an incompatible interface fit the one you expect | Adapter |
| Vary two independent dimensions without a class explosion | Bridge |
| Treat a tree of objects and single objects the same way | Composite |
| Add behavior to an object without subclassing | Decorator |
| Hide a messy subsystem behind a simple API | Facade |
| Save memory by sharing identical state | Flyweight |
| Control access to, or defer the creation of, an object | Proxy |
| Pass a request along until someone handles it | Chain of Responsibility |
| Turn an action into an object (queue it, log it, undo it) | Command |
| Traverse a collection without exposing its internals | Iterator |
| Stop many objects from talking to each other directly | Mediator |
| Save and restore an object's state | Memento |
| Notify many objects when one changes | Observer |
| Change behavior as internal state changes | State |
| Swap algorithms at runtime | Strategy |
| Fix the skeleton of an algorithm but let steps vary | Template Method |
| Add operations to a class hierarchy without editing it | Visitor |
| Evaluate sentences in a small language | Interpreter |

---

# Part I: Creational Patterns

Creational patterns hide *how* objects are made, so the rest of the code doesn't depend on concrete classes or construction details.

---

## 4. Singleton

**Intent:** guarantee that a class has exactly one instance and give everyone access to it.

**Use when:** one shared resource is genuinely required: application configuration, a connection pool, a logger registry, a hardware handle.

### The Pythonic Singleton: a module-level object

A module's code runs once, on first import, and Python caches the module, so an object created at module level is already a singleton. This is the simplest, most reliable form:

```python
class _Settings:
    def __init__(self):
        self.debug = False
        self.values = {}

settings = _Settings()          # created exactly once, when the module is first imported

def configure(**kwargs):
    settings.values.update(kwargs)

configure(env="prod")
a = settings
b = settings
print(a is b, settings.values)
```

**Output:**

```text
True {'env': 'prod'}
```

Every module that does `from myapp.config import settings` gets this same object (see `07-Modules-And-Packages.md` for why: `sys.modules` caches imports).

### Singleton with `__new__` (and the classic trap)

```python
class Bad:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, name):
        print(f"Bad.__init__({name!r})")
        self.name = name

a, b = Bad("first"), Bad("second")
print(a is b, a.name)            # same object, but __init__ ran twice and overwrote the state


class Good:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self, name):
        if self._initialized:            # guard: initialize only once
            return
        print(f"Good.__init__({name!r})")
        self.name = name
        self._initialized = True

a, b = Good("first"), Good("second")
print(a is b, a.name)
```

**Output:**

```text
Bad.__init__('first')
Bad.__init__('second')
True second
Good.__init__('first')
True first
```

**The trap:** Python calls `__init__` every time you call the class, even when `__new__` returns the existing instance. Without the guard, the second call silently resets your state.

### A thread-safe Singleton with a metaclass

A metaclass intercepts the class call itself, so `__init__` runs only once and you don't need a guard. The lock plus a second check ("double-checked locking") stops two threads from both creating an instance:

```python
import threading
import time
from concurrent.futures import ThreadPoolExecutor

class SingletonMeta(type):
    _instances = {}
    _lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:                 # fast path: no lock once created
            with cls._lock:
                if cls not in cls._instances:         # re-check: another thread may have won
                    cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    created = 0

    def __init__(self, url):
        time.sleep(0.01)             # slow startup makes a race easy to hit
        Database.created += 1
        self.url = url

with ThreadPoolExecutor(max_workers=8) as pool:
    instances = list(pool.map(lambda i: Database("postgres://db"), range(20)))

print("distinct instances:", len({id(obj) for obj in instances}))
print("constructor ran:", Database.created, "time(s)")
```

**Output:**

```text
distinct instances: 1
constructor ran: 1 time(s)
```

### A cached getter

`functools.cache` (or `lru_cache(maxsize=None)`) turns a factory function into a lazily created singleton with one decorator:

```python
from functools import cache

class Client:
    def __init__(self):
        print("connecting...")

@cache
def get_client():
    return Client()

print(get_client() is get_client())
```

**Output:**

```text
connecting...
True
```

### Borg (Monostate): share state, not identity

Some people want "one set of state" rather than "one object". The Borg pattern gives every instance the same `__dict__`:

```python
class Borg:
    _shared_state = {}

    def __init__(self):
        self.__dict__ = self._shared_state

a, b = Borg(), Borg()
a.color = "blue"
print(b.color, a is b)           # state is shared, but they are different objects
```

**Output:**

```text
blue False
```

### Pitfalls

- **A Singleton is global state in disguise.** It hides dependencies, couples code together, and makes tests leak into each other. Prefer *passing the shared object in* (see Dependency Injection in section 28).
- **Subclassing is awkward**, and reinitialization (above) is easy to get wrong.
- **Tests need a reset.** Provide a way to clear the instance, or inject a fake instead.
- Multiprocessing: each process has its own "singleton".

---

## 5. Factory Method

**Intent:** define an interface for creating an object, but let subclasses decide which class to instantiate. The creator calls a *factory method* instead of a constructor.

**Use when:** code needs to create objects but shouldn't know their concrete class, or you want subclasses (or configuration) to choose.

```python
import csv
import io
import json
from abc import ABC, abstractmethod

class Reader(ABC):                                  # the product interface
    @abstractmethod
    def read(self, text: str) -> list[dict]: ...

class CsvReader(Reader):
    def read(self, text):
        return list(csv.DictReader(io.StringIO(text)))

class JsonReader(Reader):
    def read(self, text):
        return json.loads(text)

class Importer(ABC):                                # the creator
    @abstractmethod
    def create_reader(self) -> Reader:              # <-- the factory method
        ...

    def load(self, text: str) -> list[dict]:
        rows = self.create_reader().read(text)      # shared logic, works with ANY reader
        print(f"{type(self).__name__}: loaded {len(rows)} rows")
        return rows

class CsvImporter(Importer):
    def create_reader(self):
        return CsvReader()

class JsonImporter(Importer):
    def create_reader(self):
        return JsonReader()

print(CsvImporter().load("name,age\nAna,31\nRaj,28"))
print(JsonImporter().load('[{"name": "Mei", "age": 40}]'))
```

**Output:**

```text
CsvImporter: loaded 2 rows
[{'name': 'Ana', 'age': '31'}, {'name': 'Raj', 'age': '28'}]
JsonImporter: loaded 1 rows
[{'name': 'Mei', 'age': 40}]
```

`Importer.load` never mentions CSV or JSON. Adding an XML importer means adding two small classes and changing nothing that exists.

### The Pythonic versions

Classes are objects in Python, so a plain function or a `dict` registry often does the job without a creator hierarchy:

```python
import json

READERS = {}

def register(name):
    def decorator(cls):
        READERS[name] = cls
        return cls
    return decorator

@register("json")
class JsonReader:
    def read(self, text):
        return json.loads(text)

@register("upper")
class UpperReader:
    def read(self, text):
        return text.upper()

def make_reader(kind):
    try:
        return READERS[kind]()
    except KeyError:
        raise ValueError(f"unknown reader {kind!r}; choose from {sorted(READERS)}") from None

print(make_reader("json").read('{"a": 1}'))
print(make_reader("upper").read("hello"))
try:
    make_reader("xml")
except ValueError as e:
    print(e)
```

**Output:**

```text
{'a': 1}
HELLO
unknown reader 'xml'; choose from ['json', 'upper']
```

Alternative constructors written as `@classmethod`s are Python's everyday factory methods:

```python
from dataclasses import dataclass
import math

@dataclass
class Point:
    x: float
    y: float

    @classmethod
    def from_polar(cls, r, theta):
        return cls(r * math.cos(theta), r * math.sin(theta))

    @classmethod
    def from_string(cls, text):
        x, y = (float(part) for part in text.split(","))
        return cls(x, y)

print(Point.from_string("3,4"))
print(Point.from_polar(2, 0))
```

**Output:**

```text
Point(x=3.0, y=4.0)
Point(x=2.0, y=0.0)
```

**Pitfalls:** don't build a creator hierarchy when a function would do, and don't let the registry become a hidden global that tests can't isolate.

---

## 6. Abstract Factory

**Intent:** provide an interface for creating *families* of related objects without naming their concrete classes, so the products you get always match.

**Use when:** you have several products that must be used together (a UI theme's button and checkbox, one database vendor's connection and cursor), and you want to switch the whole family at once.

```python
from abc import ABC, abstractmethod

class Button(ABC):
    @abstractmethod
    def render(self) -> str: ...

class Checkbox(ABC):
    @abstractmethod
    def render(self) -> str: ...

class LightButton(Button):
    def render(self): return "[ Light Button ]"

class LightCheckbox(Checkbox):
    def render(self): return "[ ] Light Checkbox"

class DarkButton(Button):
    def render(self): return "[# Dark Button #]"

class DarkCheckbox(Checkbox):
    def render(self): return "[x] Dark Checkbox"

class WidgetFactory(ABC):                         # the abstract factory
    @abstractmethod
    def create_button(self) -> Button: ...
    @abstractmethod
    def create_checkbox(self) -> Checkbox: ...

class LightFactory(WidgetFactory):
    def create_button(self): return LightButton()
    def create_checkbox(self): return LightCheckbox()

class DarkFactory(WidgetFactory):
    def create_button(self): return DarkButton()
    def create_checkbox(self): return DarkCheckbox()

def render_form(factory: WidgetFactory) -> list[str]:      # client code knows only the interfaces
    return [factory.create_button().render(), factory.create_checkbox().render()]

for factory in (LightFactory(), DarkFactory()):
    print(type(factory).__name__, render_form(factory))
```

**Output:**

```text
LightFactory ['[ Light Button ]', '[ ] Light Checkbox']
DarkFactory ['[# Dark Button #]', '[x] Dark Checkbox']
```

Because `render_form` only receives a factory, it can never accidentally mix a light button with a dark checkbox.

**Pythonic version:** a family is just a group of callables, so a `dict` of tuples (or a `dataclass` of classes) often replaces the whole hierarchy.

**Abstract Factory vs. Factory Method:** Factory Method creates *one* product and uses inheritance; Abstract Factory creates a *family* of products and uses composition (the factory object is passed in).

---

## 7. Builder

**Intent:** separate the construction of a complex object from its representation, so the same process can build different results and you can assemble an object step by step.

**Use when:** an object has many optional parts, constructor signatures get long, or construction needs ordering and validation.

```python
class QueryBuilder:
    def __init__(self, table):
        self._table = table
        self._columns = ["*"]
        self._conditions = []
        self._params = []
        self._order = None
        self._limit = None

    def select(self, *columns):
        self._columns = list(columns)
        return self                                   # returning self enables chaining

    def where(self, condition, *params):
        self._conditions.append(condition)
        self._params.extend(params)
        return self

    def order_by(self, column, descending=False):
        self._order = f"{column} {'DESC' if descending else 'ASC'}"
        return self

    def limit(self, n):
        if n < 0:
            raise ValueError("limit must not be negative")
        self._limit = n
        return self

    def build(self):
        sql = f"SELECT {', '.join(self._columns)} FROM {self._table}"
        if self._conditions:
            sql += " WHERE " + " AND ".join(self._conditions)
        if self._order:
            sql += f" ORDER BY {self._order}"
        if self._limit is not None:
            sql += f" LIMIT {self._limit}"
        return sql, tuple(self._params)

sql, params = (
    QueryBuilder("users")
    .select("id", "name")
    .where("age > ?", 30)
    .where("city = ?", "Pune")
    .order_by("name")
    .limit(10)
    .build()
)
print(sql)
print(params)
```

**Output:**

```text
SELECT id, name FROM users WHERE age > ? AND city = ? ORDER BY name ASC LIMIT 10
(30, 'Pune')
```

Values travel separately as **bind parameters** (`?`), never glued into the SQL text, which is what prevents SQL injection. Table and column *names* can't be bound, so validate those yourself.

### The Pythonic alternative: keyword arguments and a frozen dataclass

When there's no real step-by-step process, defaults plus keyword arguments are simpler than a builder, and `dataclasses.replace` gives you "the same but with one change":

```python
from dataclasses import dataclass, field, replace

@dataclass(frozen=True)
class HttpRequest:
    url: str
    method: str = "GET"
    headers: dict = field(default_factory=dict)
    timeout: float = 10.0

base = HttpRequest("https://api.example.com/items")
slow = replace(base, timeout=60.0)
post = replace(base, method="POST", headers={"Content-Type": "application/json"})
print(base)
print(slow.timeout, post.method)
```

**Output:**

```text
HttpRequest(url='https://api.example.com/items', method='GET', headers={}, timeout=10.0)
60.0 POST
```

**Pitfalls:** a builder that is just a pile of setters adds nothing over keyword arguments; make `build()` the place that validates the whole object, and let `build()` produce a fresh object so the builder can be reused.

---

## 8. Prototype

**Intent:** create new objects by copying an existing one (the prototype) instead of building from scratch.

**Use when:** construction is expensive or complicated, or you want many objects that start from a configured template.

Python has this built in as the `copy` module. The key subtlety is **shallow vs. deep copy**:

```python
import copy

class Report:
    def __init__(self, title, sections):
        self.title = title
        self.sections = sections            # a mutable list

template = Report("Quarterly", ["Summary", "Details"])

shallow = copy.copy(template)               # new Report, but the SAME sections list
deep = copy.deepcopy(template)              # new Report AND a new sections list

shallow.sections.append("Appendix")
print("template sections:", template.sections)     # changed by the shallow copy!
print("deep sections:    ", deep.sections)          # independent
print(shallow.sections is template.sections, deep.sections is template.sections)
```

**Output:**

```text
template sections: ['Summary', 'Details', 'Appendix']
deep sections:     ['Summary', 'Details']
True False
```

A **prototype registry** stores configured templates and clones them on demand:

```python
import copy

class Document:
    def __init__(self, title, tags):
        self.title = title
        self.tags = tags

    def __repr__(self):
        return f"Document({self.title!r}, tags={self.tags})"

class PrototypeRegistry:
    def __init__(self):
        self._prototypes = {}

    def register(self, name, prototype):
        self._prototypes[name] = prototype

    def clone(self, name, **overrides):
        obj = copy.deepcopy(self._prototypes[name])
        for key, value in overrides.items():
            setattr(obj, key, value)
        return obj

registry = PrototypeRegistry()
registry.register("invoice", Document("Invoice", ["finance", "draft"]))

a = registry.clone("invoice", title="Invoice #1")
b = registry.clone("invoice", title="Invoice #2")
a.tags.append("urgent")
print(a)
print(b)
```

**Output:**

```text
Document('Invoice #1', tags=['finance', 'draft', 'urgent'])
Document('Invoice #2', tags=['finance', 'draft'])
```

You can control copying by defining `__copy__` and `__deepcopy__` on your class (see `01-Data-Types.md` for copy semantics).

**Pitfalls:** shallow copies share mutable children; `deepcopy` is slower and can't copy some objects (open files, locks, sockets).

---

## 9. Object Pool

**Intent:** keep a set of ready-to-use objects and hand them out and take them back, instead of creating and destroying them each time.

**Use when:** objects are expensive to create (database connections, threads, large buffers) and are used for short periods.

```python
import contextlib
import queue

class Connection:
    def __init__(self, number):
        self.number = number

    def query(self, sql):
        return f"conn{self.number} ran: {sql}"

class PoolExhausted(Exception):
    pass

class ConnectionPool:
    def __init__(self, size, timeout=0.1):
        self._free = queue.Queue()
        self._timeout = timeout
        for number in range(1, size + 1):
            self._free.put(Connection(number))

    @contextlib.contextmanager
    def connection(self):
        try:
            conn = self._free.get(timeout=self._timeout)      # blocks until one is free
        except queue.Empty:
            raise PoolExhausted("no free connections") from None
        try:
            yield conn
        finally:
            self._free.put(conn)                              # ALWAYS returned, even on error

pool = ConnectionPool(size=2)

with pool.connection() as c1:
    print(c1.query("SELECT 1"))

with pool.connection() as c2:                  # the queue is first-in-first-out, so the longest-idle connection comes out
    print(c2.query("SELECT 2"))

with pool.connection() as a, pool.connection() as b:
    print("holding", a.number, "and", b.number)
    try:
        with pool.connection():
            pass
    except PoolExhausted as e:
        print("third request:", e)
```

**Output:**

```text
conn1 ran: SELECT 1
conn2 ran: SELECT 2
holding 1 and 2
third request: no free connections
```

The context manager (see `13-Context-Manager.md`) guarantees the connection goes back to the pool. In real code use the pool that ships with your driver or `concurrent.futures.ThreadPoolExecutor` for threads rather than writing your own.

**Pitfalls:** objects must be reset before reuse, leaked (never returned) objects starve the pool, and a pool that grows without bound defeats its purpose.

---

# Part II: Structural Patterns

Structural patterns are about *composition*: how to combine classes and objects into larger structures while keeping them flexible.

---

## 10. Adapter

**Intent:** convert the interface of a class into the interface that clients expect, so classes that couldn't work together can.

**Use when:** you want to use an existing class (often third-party or legacy code you can't change) whose interface doesn't match what your code expects.

```python
import xml.etree.ElementTree as ET
from typing import Protocol

class DataSource(Protocol):                       # the interface our code expects
    def fetch(self) -> list[dict]: ...

class LegacyXmlClient:                            # third-party code we cannot change
    def get_rows_xml(self) -> str:
        return "<rows><row id='1' name='Ana'/><row id='2' name='Raj'/></rows>"

class XmlSourceAdapter:                           # makes LegacyXmlClient look like a DataSource
    def __init__(self, client):
        self._client = client                     # composition: wrap, don't modify

    def fetch(self):
        root = ET.fromstring(self._client.get_rows_xml())
        return [dict(row.attrib) for row in root]

class ListSource:                                 # already matches the interface
    def fetch(self):
        return [{"id": "3", "name": "Mei"}]

def collect(sources: list[DataSource]) -> list[dict]:
    rows = []
    for source in sources:
        rows.extend(source.fetch())
    return rows

print(collect([XmlSourceAdapter(LegacyXmlClient()), ListSource()]))
```

**Output:**

```text
[{'id': '1', 'name': 'Ana'}, {'id': '2', 'name': 'Raj'}, {'id': '3', 'name': 'Mei'}]
```

`collect` works with the XML client and the plain list source through one interface, and the legacy class was never edited.

### Object adapter vs. class adapter

The version above is an **object adapter** (it *holds* the legacy object). A **class adapter** inherits from it instead:

```python
import xml.etree.ElementTree as ET

class LegacyXmlClient:
    def get_rows_xml(self):
        return "<rows><row id='1'/></rows>"

class XmlSourceClassAdapter(LegacyXmlClient):     # inherits the legacy behavior
    def fetch(self):
        root = ET.fromstring(self.get_rows_xml())
        return [dict(row.attrib) for row in root]

print(XmlSourceClassAdapter().fetch())
```

**Output:**

```text
[{'id': '1'}]
```

Prefer the object adapter: it follows "composition over inheritance" and works even if the legacy class is final or has several versions.

### Function adapters

For a single function whose signature doesn't fit, `functools.partial` or a lambda is the whole adapter:

```python
from functools import partial

def legacy_log(level, message, *, prefix=""):
    print(f"{prefix}[{level}] {message}")

info = partial(legacy_log, "INFO", prefix="app: ")    # now takes just (message)
info("started")
```

**Output:**

```text
app: [INFO] started
```

**Pitfalls:** an adapter that also changes behavior is no longer an adapter (it is a Decorator or Facade). Keep it a pure translation of the interface.

---

## 11. Bridge

**Intent:** split a class that varies in two independent ways into two hierarchies (an *abstraction* and an *implementation*) joined by composition, so each can change on its own.

**Use when:** you'd otherwise need a subclass for every combination (every notification type times every delivery channel).

Without a bridge, 2 notification kinds and 2 channels means 4 classes, and each new channel doubles that. With a bridge it is 2 + 2:

```python
from abc import ABC, abstractmethod

class Channel(ABC):                               # the implementation side
    @abstractmethod
    def deliver(self, text: str) -> None: ...

class EmailChannel(Channel):
    def deliver(self, text):
        print(f"EMAIL: {text}")

class SmsChannel(Channel):
    def deliver(self, text):
        print(f"SMS: {text}")

class Notification:                               # the abstraction side
    def __init__(self, channel: Channel):
        self.channel = channel                    # <-- the bridge

    def send(self, message):
        raise NotImplementedError

class Alert(Notification):
    def send(self, message):
        self.channel.deliver(f"ALERT! {message}")

class Reminder(Notification):
    def send(self, message):
        self.channel.deliver(f"Reminder: {message}")

for note in (Alert(EmailChannel()), Alert(SmsChannel()), Reminder(EmailChannel())):
    note.send("disk is 90% full")
```

**Output:**

```text
EMAIL: ALERT! disk is 90% full
SMS: ALERT! disk is 90% full
EMAIL: Reminder: disk is 90% full
```

Adding a `PushChannel` touches neither `Alert` nor `Reminder`, and adding a `Warning` notification touches no channel.

**Bridge vs. Adapter:** an Adapter is applied *after the fact* to make existing, incompatible classes fit. A Bridge is designed *up front* to keep two dimensions independent.

---

## 12. Composite

**Intent:** compose objects into tree structures and let clients treat single objects and groups of objects uniformly.

**Use when:** your data is naturally a tree (file systems, UI components, org charts, expression trees, menus) and you want one interface for leaves and containers.

```python
from abc import ABC, abstractmethod

class Node(ABC):
    def __init__(self, name):
        self.name = name

    @property
    @abstractmethod
    def size(self) -> int: ...

    @abstractmethod
    def render(self, indent=0) -> str: ...

class File(Node):                                 # leaf
    def __init__(self, name, size):
        super().__init__(name)
        self._size = size

    @property
    def size(self):
        return self._size

    def render(self, indent=0):
        return f"{' ' * indent}{self.name} ({self.size})"

class Directory(Node):                            # composite
    def __init__(self, name, children=()):
        super().__init__(name)
        self.children = list(children)

    def add(self, node):
        self.children.append(node)
        return self

    @property
    def size(self):
        return sum(child.size for child in self.children)    # recursion hides the tree

    def render(self, indent=0):
        lines = [f"{' ' * indent}{self.name}/ ({self.size})"]
        lines += [child.render(indent + 2) for child in self.children]
        return "\n".join(lines)

    def walk(self):                               # iterate only the files, at any depth
        for child in self.children:
            if isinstance(child, Directory):
                yield from child.walk()
            else:
                yield child

project = Directory("project", [
    Directory("src", [File("main.py", 100), File("util.py", 50)]),
    File("README.md", 20),
])
print(project.render())
print("total:", project.size)
print([f.name for f in project.walk()])
```

**Output:**

```text
project/ (170)
  src/ (150)
    main.py (100)
    util.py (50)
  README.md (20)
total: 170
['main.py', 'util.py', 'README.md']
```

Client code calls `size` or `render` on *anything* in the tree without checking whether it is a file or a folder.

**Pitfalls:** deep trees can hit Python's recursion limit (use an explicit stack for very deep ones), and it can be unclear whether leaves should raise or silently ignore `add()`.

---

## 13. Decorator (the Structural Pattern)

**Intent:** attach additional responsibilities to an object dynamically by wrapping it in another object with the same interface. It is a flexible alternative to subclassing.

**Use when:** you want to add behavior to individual objects at runtime, stack several behaviors in any order, or avoid an explosion of subclasses.

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    @abstractmethod
    def read(self) -> str: ...

class InMemorySource(DataSource):
    def __init__(self, text):
        self.text = text

    def read(self):
        return self.text

class SourceDecorator(DataSource):                # same interface, holds another DataSource
    def __init__(self, inner: DataSource):
        self._inner = inner

    def read(self):
        return self._inner.read()

class Strip(SourceDecorator):
    def read(self):
        return super().read().strip()

class Upper(SourceDecorator):
    def read(self):
        return super().read().upper()

class Logging(SourceDecorator):
    def read(self):
        data = super().read()
        print(f"read {len(data)} chars")
        return data

source = Logging(Upper(Strip(InMemorySource("  hello world  "))))
print(repr(source.read()))
```

**Output:**

```text
read 11 chars
'HELLO WORLD'
```

Each wrapper adds one behavior and the order is chosen at runtime: here the data is stripped first, then upper-cased, then logged.

### The same idea for functions: `@decorator`

Python's decorator *syntax* applies this pattern to functions. The wrapper has the same call signature as the function it wraps (see `09-Decorators.md` for the full story):

```python
import functools

def logged(func):
    @functools.wraps(func)                        # keep the original name and docstring
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}{args}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result!r}")
        return result
    return wrapper

@logged
def add(a, b):
    return a + b

add(2, 3)
```

**Output:**

```text
calling add(2, 3)
add returned 5
```

**Decorator vs. Proxy:** both wrap an object with the same interface. A Decorator *adds behavior*; a Proxy *controls access* to the object (laziness, permissions, caching).

**Pitfalls:** a long stack of wrappers is hard to debug; the wrapper must forward the whole interface; identity checks (`is`) see the wrapper, not the wrapped object.

---

## 14. Facade

**Intent:** provide a simple, unified interface to a set of interfaces in a subsystem, so the subsystem is easier to use.

**Use when:** a workflow involves several classes in a fixed order and most callers just want "do the whole thing".

```python
class Inventory:
    def __init__(self):
        self.stock = {"book": 3}

    def reserve(self, sku, qty):
        if self.stock.get(sku, 0) < qty:
            raise ValueError(f"not enough {sku}")
        self.stock[sku] -= qty
        print(f"reserved {qty} x {sku}")

class Payment:
    def charge(self, card, amount):
        print(f"charged {amount} to {card}")
        return "txn-1"

class Shipping:
    def schedule(self, sku, qty):
        print(f"shipping {qty} x {sku}")
        return "track-1"

class OrderFacade:                                # one simple entry point
    def __init__(self):
        self.inventory = Inventory()
        self.payment = Payment()
        self.shipping = Shipping()

    def place_order(self, sku, qty, card, unit_price):
        self.inventory.reserve(sku, qty)
        transaction = self.payment.charge(card, qty * unit_price)
        tracking = self.shipping.schedule(sku, qty)
        return {"transaction": transaction, "tracking": tracking}

shop = OrderFacade()
print(shop.place_order("book", 2, "4111-xxxx", 12.5))
try:
    shop.place_order("book", 5, "4111-xxxx", 12.5)
except ValueError as e:
    print("failed:", e)
```

**Output:**

```text
reserved 2 x book
charged 25.0 to 4111-xxxx
shipping 2 x book
{'transaction': 'txn-1', 'tracking': 'track-1'}
failed: not enough book
```

Callers never learn that three subsystems were involved, but the subsystems are still available directly for the rare caller who needs them.

**Pitfalls:** a facade doesn't make a workflow transactional (if `charge` failed after `reserve`, the stock would stay reserved; add compensation or a Unit of Work for that), and it shouldn't grow into a "god object" that does everything. A Python *module* with a few well-named functions is often the lightest facade.

---

## 15. Flyweight

**Intent:** save memory by sharing the parts of objects that are identical, instead of storing a copy in each.

**Use when:** you have a huge number of similar objects and much of their state is the same (characters in a document, trees in a game forest, map tiles).

Split each object's state into **intrinsic** (shared, immutable, kept in the flyweight) and **extrinsic** (unique, passed in or stored per object):

```python
class TreeType:                                   # the flyweight: shared, immutable data
    __slots__ = ("name", "color", "texture")

    def __init__(self, name, color, texture):
        self.name, self.color, self.texture = name, color, texture

class TreeTypeFactory:
    _types = {}

    @classmethod
    def get(cls, name, color, texture):
        key = (name, color, texture)
        if key not in cls._types:                 # create each distinct type once
            cls._types[key] = TreeType(name, color, texture)
        return cls._types[key]

    @classmethod
    def count(cls):
        return len(cls._types)

class Tree:                                       # unique state: position, plus a reference to the shared type
    __slots__ = ("x", "y", "type")

    def __init__(self, x, y, tree_type):
        self.x, self.y, self.type = x, y, tree_type

kinds = [("oak", "green", "rough"), ("pine", "dark green", "needles"), ("birch", "white", "smooth")]
forest = [Tree(i, i * 2, TreeTypeFactory.get(*kinds[i % 3])) for i in range(30_000)]

print(len(forest), "trees share", TreeTypeFactory.count(), "tree types")
print(forest[0].type is forest[3].type)           # same shared object
```

**Output:**

```text
30000 trees share 3 tree types
True
```

Python already uses flyweights: small integers and interned strings are shared (see `17-Internals.md`), and `__slots__` shrinks per-object memory by removing the instance `__dict__`.

**Pitfalls:** flyweights must be **immutable**, or a change to one tree's type would change every tree; the complexity is only worth it when you measure a real memory problem.

---

## 16. Proxy

**Intent:** provide a stand-in for another object to control access to it.

**Use when:** you need lazy creation (*virtual proxy*), permission checks (*protection proxy*), caching, logging, or a local representative of a remote object (*remote proxy*).

### Virtual proxy: lazy loading

```python
class HeavyReport:
    def __init__(self, name):
        print(f"loading {name} from disk...")
        self.data = [1, 2, 3]

    def total(self):
        return sum(self.data)

class LazyReportProxy:                            # same interface as HeavyReport
    def __init__(self, name):
        self._name = name
        self._real = None

    def total(self):
        if self._real is None:                    # create the real object only on first use
            self._real = HeavyReport(self._name)
        return self._real.total()

report = LazyReportProxy("sales")
print("proxy created, nothing loaded yet")
print(report.total())
print(report.total())                             # no second load
```

**Output:**

```text
proxy created, nothing loaded yet
loading sales from disk...
6
6
```

### Protection proxy: access control

```python
class Document:
    def __init__(self, text):
        self.text = text

    def read(self):
        return self.text

    def write(self, text):
        self.text = text

class DocumentProxy:
    def __init__(self, document, user, can_write):
        self._document = document
        self._user = user
        self._can_write = can_write

    def read(self):
        return self._document.read()

    def write(self, text):
        if not self._can_write:
            raise PermissionError(f"{self._user} may not write")
        self._document.write(text)

doc = Document("v1")
viewer = DocumentProxy(doc, "ana", can_write=False)
editor = DocumentProxy(doc, "raj", can_write=True)

editor.write("v2")
print(viewer.read())
try:
    viewer.write("hacked")
except PermissionError as e:
    print("blocked:", e)
```

**Output:**

```text
v2
blocked: ana may not write
```

### A generic forwarding proxy with `__getattr__`

Python lets you write one proxy for *any* class by forwarding unknown attribute lookups:

```python
class LoggingProxy:
    def __init__(self, target):
        self._target = target

    def __getattr__(self, name):                  # called only for attributes the proxy doesn't have
        attr = getattr(self._target, name)
        if not callable(attr):
            return attr

        def wrapper(*args, **kwargs):
            print(f"-> {name}{args}")
            result = attr(*args, **kwargs)
            print(f"<- {result!r}")
            return result
        return wrapper

class Calculator:
    def add(self, a, b):
        return a + b

calc = LoggingProxy(Calculator())
calc.add(2, 3)

proxy_list = LoggingProxy([1, 2, 3])
try:
    len(proxy_list)                               # special methods are looked up on the TYPE, not via __getattr__
except TypeError as e:
    print("TypeError:", e)
```

**Output:**

```text
-> add(2, 3)
<- 5
TypeError: object of type 'LoggingProxy' has no len()
```

`__getattr__` is only consulted for ordinary attribute access. Operations like `len()`, `+` and iteration look up special methods on the *type*, so a generic proxy must define those explicitly. The standard library's `weakref.proxy` is a ready-made proxy that handles this for you.

**Pitfalls:** a proxy adds a layer of indirection (and a small cost), and `type(proxy)` and `isinstance` see the proxy class, not the target's.

---

# Part III: Behavioral Patterns

Behavioral patterns are about *communication*: who does what, and how objects cooperate without being tangled together.

---

## 17. Chain of Responsibility

**Intent:** pass a request along a chain of handlers. Each handler decides either to handle it or to pass it to the next one.

**Use when:** more than one object may handle a request and the handler isn't known in advance (approval workflows, request middleware, validation pipelines, event bubbling).

```python
class Approver:
    def __init__(self, name, limit, successor=None):
        self.name = name
        self.limit = limit
        self.successor = successor

    def handle(self, amount):
        if amount <= self.limit:
            return f"{self.name} approved {amount}"
        if self.successor:
            return self.successor.handle(amount)            # pass it along the chain
        return f"{amount} rejected: no one can approve it"

ceo = Approver("CEO", 100_000)
director = Approver("Director", 10_000, ceo)
manager = Approver("Manager", 1_000, director)

for amount in (500, 7_500, 50_000, 500_000):
    print(manager.handle(amount))
```

**Output:**

```text
Manager approved 500
Director approved 7500
CEO approved 50000
500000 rejected: no one can approve it
```

The sender only knows the first handler. Adding a "VP" level means inserting one object into the chain.

### The Pythonic version: a list of functions (middleware)

Web frameworks use this shape everywhere. Each handler either returns a response (stopping the chain) or returns `None` (continue):

```python
def authenticated(request):
    if "user" not in request:
        return "401 unauthenticated"

def rate_limited(request):
    if request.get("calls", 0) > 100:
        return "429 too many requests"

def valid_payload(request):
    if not request.get("body"):
        return "400 empty body"

def run_chain(handlers, request):
    for handler in handlers:
        response = handler(request)
        if response is not None:                     # a handler answered: stop here
            return response
    return "200 OK"

chain = [authenticated, rate_limited, valid_payload]
print(run_chain(chain, {"body": "x"}))
print(run_chain(chain, {"user": "ana", "calls": 500, "body": "x"}))
print(run_chain(chain, {"user": "ana", "body": ""}))
print(run_chain(chain, {"user": "ana", "body": "data"}))
```

**Output:**

```text
401 unauthenticated
429 too many requests
400 empty body
200 OK
```

Python itself uses this idea: a log record propagates up the logger hierarchy through each parent's handlers.

**Pitfalls:** a request can fall off the end unhandled (decide what that means), long chains are hard to debug, and the order of handlers matters.

---

## 18. Command

**Intent:** turn a request into an object, so you can store it, queue it, log it, pass it around, and undo it.

**Use when:** you need undo/redo, a task queue, macro recording, scheduling, or to decouple *who asks* from *who does*.

```python
from abc import ABC, abstractmethod

class Command(ABC):
    @abstractmethod
    def execute(self): ...

    @abstractmethod
    def undo(self): ...

class Editor:                                        # the receiver
    def __init__(self):
        self.text = ""

class Insert(Command):
    def __init__(self, editor, s):
        self.editor, self.s = editor, s

    def execute(self):
        self.editor.text += self.s

    def undo(self):
        self.editor.text = self.editor.text[:-len(self.s)]

class DeleteLast(Command):
    def __init__(self, editor, n):
        self.editor, self.n, self._removed = editor, n, ""

    def execute(self):
        self._removed = self.editor.text[-self.n:]       # remember what we remove so undo can restore it
        self.editor.text = self.editor.text[:-self.n]

    def undo(self):
        self.editor.text += self._removed

class History:                                       # the invoker
    def __init__(self):
        self._done, self._undone = [], []

    def run(self, command):
        command.execute()
        self._done.append(command)
        self._undone.clear()                             # a new action invalidates the redo stack

    def undo(self):
        if self._done:
            command = self._done.pop()
            command.undo()
            self._undone.append(command)

    def redo(self):
        if self._undone:
            command = self._undone.pop()
            command.execute()
            self._done.append(command)

editor, history = Editor(), History()
history.run(Insert(editor, "Hello"))
history.run(Insert(editor, " world"))
history.run(DeleteLast(editor, 3))
print(editor.text)
history.undo();  print(editor.text)
history.undo();  print(editor.text)
history.redo();  print(editor.text)
history.run(Insert(editor, "!"))
history.redo()                                       # nothing to redo any more
print(editor.text)
```

**Output:**

```text
Hello wo
Hello world
Hello
Hello world
Hello world!
```

### The Pythonic version: callables

If you only need to *queue and run* actions (no undo), a command is just a callable, and `functools.partial` bundles the arguments:

```python
from collections import deque
from functools import partial

def send_email(to, subject):
    print(f"email to {to}: {subject}")

def resize(image, width):
    print(f"resizing {image} to {width}px")

tasks = deque([
    partial(send_email, "ana@example.com", "Welcome"),
    partial(resize, "logo.png", 128),
])
while tasks:
    tasks.popleft()()                                # each task is a ready-to-run command
```

**Output:**

```text
email to ana@example.com: Welcome
resizing logo.png to 128px
```

**Pitfalls:** every undoable command must capture enough state to reverse itself; undo for destructive actions can be impossible; unbounded history grows memory.

---

## 19. Iterator

**Intent:** access the elements of a collection sequentially without exposing how the collection is built.

**Use when:** you want one traversal interface for different structures (lists, trees, database cursors, files, infinite streams).

Python builds this pattern into the language (`__iter__`, `__next__`, `for`, generators; see `10-Iterators-Generators.md`). The easiest custom iterator is a generator method:

```python
class Tree:
    def __init__(self, value, children=()):
        self.value = value
        self.children = list(children)

    def __iter__(self):                              # a generator: each loop gets a fresh iterator
        yield self.value
        for child in self.children:
            yield from child                         # recursion through the children's __iter__

tree = Tree(1, [Tree(2, [Tree(4)]), Tree(3)])
print(list(tree))
print(sum(tree))
print(list(tree))                                    # can be iterated again
```

**Output:**

```text
[1, 2, 4, 3]
10
[1, 2, 4, 3]
```

When you need a real iterator object (to keep traversal state separate from the collection), return a separate class from `__iter__`:

```python
class Countdown:
    def __init__(self, start):
        self.start = start

    def __iter__(self):
        return CountdownIterator(self.start)          # a fresh iterator each time

class CountdownIterator:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        return self

    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n + 1

counter = Countdown(3)
print(list(counter), list(counter))
```

**Output:**

```text
[3, 2, 1] [3, 2, 1]
```

**Pitfalls:** an iterator is single-use, so don't hand the same iterator to two consumers; modifying a collection while iterating over it gives surprising results.

---

## 20. Mediator

**Intent:** define an object that encapsulates how a set of objects interact, so they don't refer to each other directly.

**Use when:** many objects talk to many others and the web of references is becoming unmanageable (chat rooms, UI dialogs, air-traffic control).

```python
class ChatRoom:                                       # the mediator
    def __init__(self):
        self._users = {}

    def register(self, user):
        self._users[user.name] = user
        user.room = self

    def send(self, sender, message, to=None):
        if to:
            targets = [self._users[to]]
        else:
            targets = [u for name, u in self._users.items() if name != sender.name]
        for user in targets:
            user.receive(sender.name, message)

class User:
    def __init__(self, name):
        self.name = name
        self.room = None

    def send(self, message, to=None):
        self.room.send(self, message, to)             # talks to the room, never to other users

    def receive(self, sender, message):
        print(f"{self.name} got from {sender}: {message}")

room = ChatRoom()
ana, raj, mei = User("ana"), User("raj"), User("mei")
for user in (ana, raj, mei):
    room.register(user)

ana.send("hello all")
raj.send("psst", to="mei")
```

**Output:**

```text
raj got from ana: hello all
mei got from ana: hello all
mei got from raj: psst
```

With N users and no mediator you could have N x (N - 1) links; with a mediator there are N.

**Mediator vs. Observer:** Observer is one-to-many broadcasting from a subject; a Mediator coordinates two-way conversations among peers and may apply rules. **Pitfall:** the mediator tends to absorb all the logic and become a god object.

---

## 21. Memento

**Intent:** capture an object's internal state in a snapshot, without breaking encapsulation, so it can be restored later.

**Use when:** you need undo, checkpoints, transactions that can roll back, or save games.

```python
from dataclasses import dataclass

@dataclass(frozen=True)                               # the memento: immutable
class Snapshot:
    text: str
    cursor: int

class Editor:                                          # the originator
    def __init__(self):
        self.text, self.cursor = "", 0

    def type(self, s):
        self.text = self.text[:self.cursor] + s + self.text[self.cursor:]
        self.cursor += len(s)

    def move(self, position):
        self.cursor = max(0, min(position, len(self.text)))

    def save(self):
        return Snapshot(self.text, self.cursor)

    def restore(self, snapshot):
        self.text, self.cursor = snapshot.text, snapshot.cursor

    def __repr__(self):
        return f"Editor(text={self.text!r}, cursor={self.cursor})"

editor = Editor()
history = []                                           # the caretaker: just keeps the snapshots

editor.type("Hello")
history.append(editor.save())
editor.type(" world")
history.append(editor.save())

editor.move(0)
editor.type(">> ")
print(editor)

editor.restore(history[-1])
print(editor)
editor.restore(history[0])
print(editor)
```

**Output:**

```text
Editor(text='>> Hello world', cursor=3)
Editor(text='Hello world', cursor=11)
Editor(text='Hello', cursor=5)
```

**Memento vs. Command:** a Memento stores *state* and restores it wholesale; a Command stores *the operation* and reverses it. Mementos are simpler but use more memory for large state (consider storing diffs). `pickle` and `copy.deepcopy` are quick ways to snapshot arbitrary objects.

---

## 22. Observer

**Intent:** define a one-to-many dependency, so that when one object (the *subject*) changes, all its dependents (*observers*) are notified automatically.

**Use when:** a change in one place should trigger updates elsewhere without the subject knowing who is listening (events, GUIs, model/view, change notifications).

In Python the observers are usually just callables. The version below also handles two real-world problems: observers that should be able to **unsubscribe**, and observers whose objects get **garbage collected** (a strong reference would keep them alive forever, which is a classic memory leak):

```python
import gc
import inspect
import weakref

class Subject:
    def __init__(self):
        self._observers = []

    def subscribe(self, callback):
        if inspect.ismethod(callback):                # bound method: hold the object WEAKLY
            ref = weakref.WeakMethod(callback)
        else:                                         # plain function or lambda: hold it normally
            ref = lambda: callback
        self._observers.append(ref)
        return lambda: self._observers.remove(ref)    # calling this unsubscribes

    def notify(self, *args, **kwargs):
        for ref in list(self._observers):             # iterate a copy: callbacks may unsubscribe
            callback = ref()
            if callback is None:                      # the observer object was garbage collected
                self._observers.remove(ref)
                continue
            callback(*args, **kwargs)

    @property
    def count(self):
        return len(self._observers)

class Logger:
    def on_event(self, name):
        print(f"Logger saw {name}")

button = Subject()
logger = Logger()
unsubscribe = button.subscribe(lambda name: print(f"lambda saw {name}"))
button.subscribe(logger.on_event)

button.notify("click")
print("observers:", button.count)

del logger                                            # the Logger object goes away
gc.collect()
button.notify("hover")                                # the dead observer is skipped and removed
print("observers:", button.count)

unsubscribe()
print("observers:", button.count)
```

**Output:**

```text
lambda saw click
Logger saw click
observers: 2
lambda saw hover
observers: 1
observers: 0
```

**Pitfalls:**
- **Memory leaks:** a subject holding strong references to observers keeps them alive; use weak references or explicit unsubscribe.
- **Exceptions:** one failing observer shouldn't stop the others (consider catching and logging per observer).
- **Order and re-entrancy:** don't rely on notification order, and expect callbacks to subscribe or unsubscribe during a notification (hence iterating over a copy).
- **Threads:** add locking if subscribing and notifying can happen on different threads.

A topic-based variant of this idea (an event bus) is in section 36.

---

## 23. State

**Intent:** let an object change its behavior when its internal state changes, so it appears to change its class.

**Use when:** an object behaves differently in different states and you are drowning in `if state == ...` chains (order lifecycles, connections, document workflows, vending machines).

```python
class InvalidTransition(Exception):
    pass

class State:                                          # default: every action is illegal
    name = "state"

    def pay(self, order):
        raise InvalidTransition(f"cannot pay: order is {self.name}")

    def ship(self, order):
        raise InvalidTransition(f"cannot ship: order is {self.name}")

    def deliver(self, order):
        raise InvalidTransition(f"cannot deliver: order is {self.name}")

class Draft(State):
    name = "draft"
    def pay(self, order):
        order.state = Paid()

class Paid(State):
    name = "paid"
    def ship(self, order):
        order.state = Shipped()

class Shipped(State):
    name = "shipped"
    def deliver(self, order):
        order.state = Delivered()

class Delivered(State):
    name = "delivered"

class Order:                                          # the context: delegates to the current state
    def __init__(self):
        self.state = Draft()

    def pay(self):
        self.state.pay(self)

    def ship(self):
        self.state.ship(self)

    def deliver(self):
        self.state.deliver(self)

order = Order()
print(order.state.name)
order.pay()
order.ship()
print(order.state.name)
try:
    order.pay()                                       # already shipped
except InvalidTransition as e:
    print("error:", e)
order.deliver()
print(order.state.name)
```

**Output:**

```text
draft
shipped
error: cannot pay: order is shipped
delivered
```

Each state class holds only the behavior that is legal in that state; illegal actions fail in one place instead of in a tangle of conditionals.

### The Pythonic version: an enum and a transition table

When states differ only in *which transitions are allowed* (not in rich behavior), data beats classes:

```python
from enum import Enum

class Status(Enum):
    DRAFT = "draft"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"

TRANSITIONS = {
    Status.DRAFT: {"pay": Status.PAID},
    Status.PAID: {"ship": Status.SHIPPED},
    Status.SHIPPED: {"deliver": Status.DELIVERED},
    Status.DELIVERED: {},
}

class Order:
    def __init__(self):
        self.status = Status.DRAFT

    def apply(self, action):
        try:
            self.status = TRANSITIONS[self.status][action]
        except KeyError:
            raise ValueError(f"cannot {action} when {self.status.value}") from None

order = Order()
for action in ("pay", "ship", "pay"):
    try:
        order.apply(action)
        print(action, "->", order.status.value)
    except ValueError as e:
        print("error:", e)
```

**Output:**

```text
pay -> paid
ship -> shipped
error: cannot pay when shipped
```

**State vs. Strategy:** the structure is nearly identical (a context delegating to an interchangeable object), but in State the objects *change themselves* as the context moves between states, while in Strategy the *client* picks one and it stays put.

---

## 24. Strategy

**Intent:** define a family of algorithms, encapsulate each one, and make them interchangeable, so the algorithm can vary independently of the code that uses it.

**Use when:** you have several ways to do the same job (pricing rules, sorting, compression, routing, validation) and want to pick at runtime without `if/elif` ladders.

In Python a strategy is usually just a function:

```python
from typing import Callable

Discount = Callable[[float], float]

def no_discount(total):
    return total

def percent_off(percent):
    return lambda total: total * (1 - percent / 100)

def flat_off(amount):
    return lambda total: max(total - amount, 0)

class Cart:
    def __init__(self, prices, discount: Discount = no_discount):
        self.prices = prices
        self.discount = discount                      # the strategy, injected

    def total(self):
        return round(self.discount(sum(self.prices)), 2)

prices = [40.0, 60.0, 25.5]
for label, strategy in [("none", no_discount), ("10% off", percent_off(10)),
                        ("flat 30", flat_off(30)), ("flat 500", flat_off(500))]:
    print(f"{label:9} -> {Cart(prices, strategy).total()}")
```

**Output:**

```text
none      -> 125.5
10% off   -> 112.95
flat 30   -> 95.5
flat 500  -> 0
```

You already use Strategy every time you pass `key=` to a built-in:

```python
users = [{"name": "Raj", "age": 28}, {"name": "ana", "age": 31}, {"name": "Mei", "age": 40}]

print([u["name"] for u in sorted(users, key=lambda u: u["name"].lower())])
print([u["name"] for u in sorted(users, key=lambda u: u["age"], reverse=True)])
```

**Output:**

```text
['ana', 'Mei', 'Raj']
['Mei', 'ana', 'Raj']
```

### Class-based strategies when they carry configuration

When a strategy needs state or several methods, use a small class that satisfies a `Protocol` (see `15-Protocals.md`):

```python
import bz2
import zlib
from typing import Protocol

class Compressor(Protocol):
    def compress(self, data: bytes) -> bytes: ...

class ZlibCompressor:
    def __init__(self, level=6):
        self.level = level

    def compress(self, data):
        return zlib.compress(data, self.level)

class Bz2Compressor:
    def compress(self, data):
        return bz2.compress(data)

def archive(data: bytes, compressor: Compressor) -> int:
    return len(compressor.compress(data))

payload = b"abc" * 1000
for compressor in (ZlibCompressor(9), Bz2Compressor()):
    print(type(compressor).__name__, "shrank the data:", archive(payload, compressor) < len(payload))
```

**Output:**

```text
ZlibCompressor shrank the data: True
Bz2Compressor shrank the data: True
```

**Pitfalls:** don't create a class hierarchy for what a function can express; clients must know enough to choose a sensible strategy; and don't confuse it with State (see above).

---

## 25. Template Method

**Intent:** define the skeleton of an algorithm in a base class, deferring some steps to subclasses, so subclasses can redefine certain steps without changing the algorithm's structure.

**Use when:** several classes follow the same overall procedure but differ in details (ETL jobs, report generators, test fixtures, game turns).

```python
from abc import ABC, abstractmethod

class ETLJob(ABC):
    def run(self):                                    # the TEMPLATE METHOD: a fixed order of steps
        self.before_run()
        data = self.extract()
        data = self.transform(data)
        self.load(data)
        self.after_run()

    def before_run(self):                             # hook with a default implementation
        print(f"[{type(self).__name__}] starting")

    def after_run(self):                              # hook that does nothing by default
        pass

    @abstractmethod
    def extract(self): ...                            # steps every subclass MUST provide

    def transform(self, data):                        # optional step: default is "no change"
        return data

    @abstractmethod
    def load(self, data): ...

class SalesJob(ETLJob):
    def __init__(self):
        self.warehouse = []

    def extract(self):
        return [("a", 10), ("b", 20)]

    def transform(self, data):
        return [(key.upper(), value * 2) for key, value in data]

    def load(self, data):
        self.warehouse.extend(data)
        print(f"loaded {len(data)} rows")

    def after_run(self):
        print("done")

job = SalesJob()
job.run()
print(job.warehouse)
```

**Output:**

```text
[SalesJob] starting
loaded 2 rows
done
[('A', 20), ('B', 40)]
```

The base class controls *when* each step runs ("don't call us, we'll call you"), so no subclass can forget a step or run them in the wrong order. Python's own `unittest.TestCase` (`setUp` / test / `tearDown`) works this way.

**Pythonic alternative:** if the variation is small, pass the steps in as functions (a Strategy) instead of subclassing. **Pitfall:** deep inheritance chains of templates are hard to follow; keep the number of hooks small and document which ones are required.

---

## 26. Visitor

**Intent:** represent an operation to be performed on the elements of an object structure, so you can define new operations without changing the classes of the elements.

**Use when:** you have a stable set of element classes (like the nodes of a syntax tree) and keep adding new operations over them (evaluate, print, optimize, type-check).

### The classic form (double dispatch)

```python
class Num:
    def __init__(self, value):
        self.value = value
    def accept(self, visitor):
        return visitor.visit_num(self)

class Add:
    def __init__(self, left, right):
        self.left, self.right = left, right
    def accept(self, visitor):
        return visitor.visit_add(self)

class Mul:
    def __init__(self, left, right):
        self.left, self.right = left, right
    def accept(self, visitor):
        return visitor.visit_mul(self)

class Evaluator:                                      # one operation = one visitor class
    def visit_num(self, node):
        return node.value
    def visit_add(self, node):
        return node.left.accept(self) + node.right.accept(self)
    def visit_mul(self, node):
        return node.left.accept(self) * node.right.accept(self)

class Printer:
    def visit_num(self, node):
        return str(node.value)
    def visit_add(self, node):
        return f"({node.left.accept(self)} + {node.right.accept(self)})"
    def visit_mul(self, node):
        return f"({node.left.accept(self)} * {node.right.accept(self)})"

tree = Add(Num(2), Mul(Num(3), Num(4)))
print(tree.accept(Printer()), "=", tree.accept(Evaluator()))
```

**Output:**

```text
(2 + (3 * 4)) = 14
```

Adding a new operation means adding a visitor class; no element class changes.

### The Pythonic form: `functools.singledispatch`

Python can dispatch on the type of the first argument, which removes the `accept` boilerplate:

```python
from dataclasses import dataclass
from functools import singledispatch

@dataclass
class Num:
    value: float

@dataclass
class Add:
    left: object
    right: object

@dataclass
class Mul:
    left: object
    right: object

@singledispatch
def evaluate(node):
    raise TypeError(f"cannot evaluate {type(node).__name__}")

@evaluate.register
def _(node: Num):
    return node.value

@evaluate.register
def _(node: Add):
    return evaluate(node.left) + evaluate(node.right)

@evaluate.register
def _(node: Mul):
    return evaluate(node.left) * evaluate(node.right)

@singledispatch
def show(node):
    raise TypeError(f"cannot show {type(node).__name__}")

@show.register
def _(node: Num):
    return str(node.value)

@show.register
def _(node: Add):
    return f"({show(node.left)} + {show(node.right)})"

@show.register
def _(node: Mul):
    return f"({show(node.left)} * {show(node.right)})"

tree = Add(Num(2), Mul(Num(3), Num(4)))
print(show(tree), "=", evaluate(tree))
```

**Output:**

```text
(2 + (3 * 4)) = 14
```

The standard library's `ast.NodeVisitor` is this pattern for Python's own syntax trees.

**Pitfalls:** the classic Visitor makes adding a new *element* class painful (every visitor must change), so it only pays off when the element set is stable. The `singledispatch` form fixes much of that.

---

## 27. Interpreter

**Intent:** given a language, define a representation of its grammar together with an interpreter that uses the representation to evaluate sentences in the language.

**Use when:** you have a small, stable mini-language (filter rules, search queries, simple config expressions) that you need to evaluate repeatedly. For anything large, use a real parser library.

Here is a tiny rule language such as `age >= 30 and (city == 'Pune' or vip == 1)`. Each grammar rule becomes a class with an `interpret` method:

```python
import operator
import re
from abc import ABC, abstractmethod

class Expr(ABC):
    @abstractmethod
    def interpret(self, context: dict) -> bool: ...

class Compare(Expr):                                  # terminal expression
    OPS = {"==": operator.eq, "!=": operator.ne, ">": operator.gt,
           "<": operator.lt, ">=": operator.ge, "<=": operator.le}

    def __init__(self, name, op, value):
        self.name, self.op, self.value = name, op, value

    def interpret(self, context):
        if self.name not in context:
            return False
        return self.OPS[self.op](context[self.name], self.value)

class And(Expr):                                      # non-terminal expressions
    def __init__(self, left, right):
        self.left, self.right = left, right
    def interpret(self, context):
        return self.left.interpret(context) and self.right.interpret(context)

class Or(Expr):
    def __init__(self, left, right):
        self.left, self.right = left, right
    def interpret(self, context):
        return self.left.interpret(context) or self.right.interpret(context)

class Not(Expr):
    def __init__(self, operand):
        self.operand = operand
    def interpret(self, context):
        return not self.operand.interpret(context)

TOKEN = re.compile(r"""\s*(?:(?P<num>\d+(?:\.\d+)?)|(?P<str>'[^']*'|"[^"]*")|(?P<op>==|!=|>=|<=|>|<)|(?P<paren>[()])|(?P<word>[A-Za-z_]\w*))""")

def tokenize(text):
    tokens, pos, text = [], 0, text.rstrip()
    while pos < len(text):
        match = TOKEN.match(text, pos)
        if not match:
            raise SyntaxError(f"unexpected text at position {pos}: {text[pos:]!r}")
        kind = match.lastgroup
        tokens.append((kind, match.group(kind)))
        pos = match.end()
    return tokens

class Parser:                                         # builds the expression tree from tokens
    def __init__(self, tokens):
        self.tokens, self.i = tokens, 0

    def peek(self):
        return self.tokens[self.i] if self.i < len(self.tokens) else (None, None)

    def take(self):
        token = self.peek()
        self.i += 1
        return token

    def parse(self):
        expr = self.or_expr()
        if self.i != len(self.tokens):
            raise SyntaxError(f"unexpected token {self.peek()[1]!r}")
        return expr

    def or_expr(self):
        left = self.and_expr()
        while self.peek() == ("word", "or"):
            self.take()
            left = Or(left, self.and_expr())
        return left

    def and_expr(self):
        left = self.factor()
        while self.peek() == ("word", "and"):
            self.take()
            left = And(left, self.factor())
        return left

    def factor(self):
        if self.peek() == ("word", "not"):
            self.take()
            return Not(self.factor())
        if self.peek() == ("paren", "("):
            self.take()
            expr = self.or_expr()
            if self.take() != ("paren", ")"):
                raise SyntaxError("missing closing parenthesis")
            return expr
        return self.comparison()

    def comparison(self):
        kind, name = self.take()
        if kind != "word":
            raise SyntaxError(f"expected a field name, got {name!r}")
        kind, op = self.take()
        if kind != "op":
            raise SyntaxError(f"expected a comparison operator after {name!r}")
        kind, raw = self.take()
        if kind == "num":
            value = float(raw) if "." in raw else int(raw)
        elif kind == "str":
            value = raw[1:-1]
        else:
            raise SyntaxError(f"expected a number or string after {op!r}")
        return Compare(name, op, value)

def parse(text):
    return Parser(tokenize(text)).parse()

rule = parse("age >= 30 and (city == 'Pune' or vip == 1) and not banned == 1")

people = [
    {"name": "Ana", "age": 31, "city": "Pune"},
    {"name": "Raj", "age": 45, "city": "Delhi", "vip": 1},
    {"name": "Mei", "age": 29, "city": "Pune"},
    {"name": "Kim", "age": 50, "city": "Pune", "banned": 1},
]
print([p["name"] for p in people if rule.interpret(p)])

try:
    parse("age >")
except SyntaxError as e:
    print("SyntaxError:", e)
```

**Output:**

```text
['Ana', 'Raj']
SyntaxError: expected a number or string after '>'
```

The expression tree is built once by the parser and can then be evaluated against any number of records. Python's `re` module and `ast.literal_eval` are everyday interpreters of small languages.

**Pitfalls:** every grammar rule adds a class, so it doesn't scale to large languages. Never `eval()` untrusted text as a shortcut; that is exactly the hole a purpose-built interpreter closes.

---

# Part IV: Patterns Python and Modern Back-End Code Lean On

These didn't come from the original 23, but they show up constantly in real Python code and in interviews.

---

## 28. Dependency Injection

**Intent:** give an object the things it depends on (database, clock, notifier) from the outside, instead of letting it create or look them up itself.

**Use when:** you want code that is easy to test, easy to reconfigure, and free of hidden globals. It is the practical form of the Dependency Inversion principle.

```python
class SqlOrderRepository:                             # "production" dependency
    def totals(self):
        return [100, 250]

class FakeOrderRepository:                            # test double
    def __init__(self, totals):
        self._totals = totals

    def totals(self):
        return self._totals

class ConsoleNotifier:
    def send(self, text):
        print(f"notify: {text}")

class RecordingNotifier:
    def __init__(self):
        self.sent = []

    def send(self, text):
        self.sent.append(text)

class ReportService:
    def __init__(self, repository, notifier):         # constructor injection
        self.repository = repository
        self.notifier = notifier

    def publish_total(self):
        total = sum(self.repository.totals())
        self.notifier.send(f"total sales: {total}")
        return total

# Production wiring
ReportService(SqlOrderRepository(), ConsoleNotifier()).publish_total()

# Test wiring: no database, no console, and the result can be inspected
notifier = RecordingNotifier()
assert ReportService(FakeOrderRepository([1, 2, 3]), notifier).publish_total() == 6
print(notifier.sent)

# A tiny container ("composition root") that wires things once, lazily
class Container:
    def __init__(self):
        self._factories, self._instances = {}, {}

    def register(self, name, factory):
        self._factories[name] = factory

    def get(self, name):
        if name not in self._instances:
            self._instances[name] = self._factories[name](self)
        return self._instances[name]

container = Container()
container.register("repository", lambda c: SqlOrderRepository())
container.register("notifier", lambda c: ConsoleNotifier())
container.register("service", lambda c: ReportService(c.get("repository"), c.get("notifier")))

container.get("service").publish_total()
print(container.get("service") is container.get("service"))
```

**Output:**

```text
notify: total sales: 350
['total sales: 6']
notify: total sales: 350
True
```

Most Python projects don't need a container library: plain constructor arguments and one place near program start that wires everything (the *composition root*) go a long way. Dependency Injection is also the healthy replacement for a Singleton, because the shared object is passed in rather than reached for.

**Pitfalls:** injecting everything into everything creates huge constructors (a sign the class does too much), and a service locator ("ask the container for it from anywhere") brings the hidden-global problem back.

---

## 29. Registry / Plugin

**Intent:** let new implementations announce themselves to a central lookup table, so code can find them by name without knowing them in advance.

**Use when:** you want an extensible system: exporters, parsers, commands, handlers, database backends loaded by name from configuration.

Python's `__init_subclass__` hook (see `11-Object-Oriented-Program.md`) makes registration automatic: just defining a subclass adds it:

```python
import json

class Exporter:
    registry = {}

    def __init_subclass__(cls, format=None, **kwargs):
        super().__init_subclass__(**kwargs)
        if format:
            Exporter.registry[format] = cls          # registered the moment the class is defined

    @classmethod
    def create(cls, format):
        try:
            return cls.registry[format]()
        except KeyError:
            raise ValueError(f"no exporter for {format!r}; known: {sorted(cls.registry)}") from None

class CsvExporter(Exporter, format="csv"):
    def export(self, rows):
        return "\n".join(",".join(map(str, row)) for row in rows)

class JsonExporter(Exporter, format="json"):
    def export(self, rows):
        return json.dumps(rows)

rows = [[1, "Ana"], [2, "Raj"]]
print(Exporter.create("csv").export(rows))
print(Exporter.create("json").export(rows))
try:
    Exporter.create("xml")
except ValueError as e:
    print(e)
```

**Output:**

```text
1,Ana
2,Raj
[[1, "Ana"], [2, "Raj"]]
no exporter for 'xml'; known: ['csv', 'json']
```

Third-party packages can also register plugins through *entry points* in their packaging metadata. The decorator-based registry from section 5 is the other common form.

**Pitfall:** a class is only registered once its module is imported, so make sure plugin modules actually get imported.

---

## 30. Null Object

**Intent:** provide an object with neutral "do nothing" behavior in place of `None`, so callers don't need to check.

**Use when:** a collaborator is optional (logging, metrics, notifications) and you are tired of `if logger is not None:` scattered everywhere.

```python
class Logger:
    def info(self, message):
        print(f"INFO {message}")

class NullLogger:                                     # same interface, does nothing
    def info(self, message):
        pass

def process(items, logger=None):
    logger = NullLogger() if logger is None else logger
    for item in items:
        logger.info(f"processing {item}")             # no None check needed
    return len(items)

print(process([1, 2]))                                # silent
print(process([1, 2], Logger()))                      # logs
```

**Output:**

```text
2
INFO processing 1
INFO processing 2
2
```

The standard library has several: `contextlib.nullcontext()` is a context manager that does nothing, and `os.devnull` is a file that discards writes. **Pitfall:** a Null Object can hide a mistake (you meant to configure a real logger and silently didn't), so use it only where "nothing" is genuinely a valid behavior.

---

## 31. Repository and Unit of Work

**Intent:** a **Repository** hides how data is stored behind a collection-like interface; a **Unit of Work** groups changes so they are applied together or not at all.

**Use when:** you want business logic that doesn't know about the database, can be tested with an in-memory store, and keeps related changes consistent.

```python
from abc import ABC, abstractmethod

class UserRepository(ABC):                            # what the business logic sees
    @abstractmethod
    def add(self, user): ...

    @abstractmethod
    def get(self, user_id): ...

    @abstractmethod
    def list(self): ...

class InMemoryUserRepository(UserRepository):        # swap for a SQL one in production
    def __init__(self):
        self._users = {}

    def add(self, user):
        self._users[user["id"]] = user

    def get(self, user_id):
        return self._users.get(user_id)

    def list(self):
        return list(self._users.values())

class UnitOfWork:
    def __init__(self, repository):
        self.repository = repository
        self._pending = []

    def add(self, user):
        self._pending.append(user)                    # staged, not yet stored

    def commit(self):
        for user in self._pending:
            self.repository.add(user)
        self._pending.clear()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc, traceback):
        if exc_type is None:
            self.commit()                             # success: apply everything
        else:
            self._pending.clear()                     # failure: roll back by discarding
        return False                                  # never swallow the exception

repo = InMemoryUserRepository()

with UnitOfWork(repo) as uow:
    uow.add({"id": 1, "name": "Ana"})
    uow.add({"id": 2, "name": "Raj"})
print([u["name"] for u in repo.list()])

try:
    with UnitOfWork(repo) as uow:
        uow.add({"id": 3, "name": "Mei"})
        raise RuntimeError("payment failed")
except RuntimeError as e:
    print("rolled back:", e)
print([u["name"] for u in repo.list()])
```

**Output:**

```text
['Ana', 'Raj']
rolled back: payment failed
['Ana', 'Raj']
```

The second block's user "Mei" was never stored, because the error discarded the staged changes. With a real database the unit of work wraps a transaction (`commit` / `rollback`) in the same shape; see `13-Context-Manager.md` for the context-manager mechanics.

---

## 32. Specification

**Intent:** express business rules as small, named, combinable objects.

**Use when:** filtering or validation rules get combined in many ways and `if a and not b or c` conditions are multiplying.

```python
class Spec:
    def __init__(self, predicate):
        self._predicate = predicate

    def __call__(self, item):
        return self._predicate(item)

    def __and__(self, other):
        return Spec(lambda item: self(item) and other(item))

    def __or__(self, other):
        return Spec(lambda item: self(item) or other(item))

    def __invert__(self):
        return Spec(lambda item: not self(item))

adult = Spec(lambda user: user["age"] >= 18)
in_pune = Spec(lambda user: user["city"] == "Pune")
premium = Spec(lambda user: user.get("premium", False))

users = [
    {"name": "Ana", "age": 31, "city": "Pune", "premium": True},
    {"name": "Raj", "age": 17, "city": "Pune"},
    {"name": "Mei", "age": 40, "city": "Delhi"},
    {"name": "Kim", "age": 25, "city": "Pune"},
]

eligible = adult & (in_pune | premium)               # rules read like the business language
outside_pune = adult & ~in_pune

print([u["name"] for u in users if eligible(u)])
print([u["name"] for u in users if outside_pune(u)])
```

**Output:**

```text
['Ana', 'Kim']
['Mei']
```

Operator overloading (`__and__`, `__or__`, `__invert__`) lets the combined rule stay readable. For SQL-backed data, the same idea builds query filters instead of Python predicates.

---

## 33. Pipeline (Pipes and Filters)

**Intent:** process data through a sequence of independent stages, where each stage's output feeds the next.

**Use when:** you transform streams of records (ETL, log processing, text processing) and want small testable steps that don't hold the whole data set in memory.

Generators make this natural, because each stage pulls one item at a time from the previous one (see `10-Iterators-Generators.md`):

```python
from functools import reduce

def read_lines(text):
    for line in text.splitlines():
        yield line

def parse(lines):
    for line in lines:
        name, qty = line.split(",")
        yield {"name": name.strip(), "qty": int(qty)}

def only_positive(rows):
    return (row for row in rows if row["qty"] > 0)

def totals(rows):
    result = {}
    for row in rows:
        result[row["name"]] = result.get(row["name"], 0) + row["qty"]
    return result

text = "apple, 3\npear, 0\napple, 4\nfig, 2"
print(totals(only_positive(parse(read_lines(text)))))

def pipeline(*stages):                               # compose stages left to right
    return lambda source: reduce(lambda data, stage: stage(data), stages, source)

process = pipeline(read_lines, parse, only_positive, totals)
print(process(text))
```

**Output:**

```text
{'apple': 7, 'fig': 2}
{'apple': 7, 'fig': 2}
```

Each stage can be tested on its own, stages can be reordered or replaced, and nothing is read until the last stage asks for it.

---

## 34. Retry

**Intent:** automatically repeat an operation that may fail temporarily (network calls, locked resources), usually with growing delays (*exponential backoff*).

**Use when:** failures are often transient and the operation is safe to repeat (**idempotent**).

```python
import functools
import time

def retry(times=3, delay=0.5, backoff=2.0, exceptions=(Exception,), sleep=None):
    sleep = sleep or time.sleep

    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            wait = delay
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    if attempt == times:
                        raise                          # out of attempts: let the error escape
                    print(f"attempt {attempt} failed ({exc}); retrying in {wait}s")
                    sleep(wait)
                    wait *= backoff
        return wrapper
    return decorator

calls = {"n": 0}

@retry(times=4, delay=1.0, sleep=lambda seconds: None, exceptions=(ConnectionError,))
def flaky():
    calls["n"] += 1
    if calls["n"] < 3:
        raise ConnectionError("timeout")
    return f"ok after {calls['n']} calls"

print(flaky())

@retry(times=2, delay=1.0, sleep=lambda seconds: None)
def always_fails():
    raise RuntimeError("service down")

try:
    always_fails()
except RuntimeError as e:
    print("gave up:", e)
```

**Output:**

```text
attempt 1 failed (timeout); retrying in 1.0s
attempt 2 failed (timeout); retrying in 2.0s
ok after 3 calls
attempt 1 failed (service down); retrying in 1.0s
gave up: service down
```

(The `sleep` parameter is injected so this demo doesn't really wait: dependency injection again.)

**Pitfalls:** retry only *transient* errors (not bugs or bad input), only idempotent operations, add random **jitter** to the delays so many clients don't retry in lockstep, and set an overall limit so retries can't pile up. Pair it with a Circuit Breaker.

---

## 35. Circuit Breaker

**Intent:** stop calling a failing dependency for a while, failing fast instead, so a struggling service isn't hammered and callers don't pile up waiting on timeouts.

**States:** **closed** (normal), **open** (failing fast), **half-open** (after a cool-down, let one trial call through: success closes it, failure re-opens it).

```python
import time

class CircuitOpen(Exception):
    pass

class CircuitBreaker:
    def __init__(self, failure_threshold=3, reset_timeout=30.0, clock=time.monotonic):
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.clock = clock
        self.failures = 0
        self.opened_at = None
        self.state = "closed"

    def call(self, func, *args, **kwargs):
        if self.state == "open":
            if self.clock() - self.opened_at >= self.reset_timeout:
                self.state = "half-open"              # cool-down over: allow one trial call
            else:
                raise CircuitOpen("circuit is open; failing fast")
        try:
            result = func(*args, **kwargs)
        except Exception:
            self._record_failure()
            raise
        self._record_success()
        return result

    def _record_failure(self):
        self.failures += 1
        if self.state == "half-open" or self.failures >= self.failure_threshold:
            self.state = "open"
            self.opened_at = self.clock()

    def _record_success(self):
        self.failures = 0
        self.state = "closed"

now = [0.0]                                           # a fake clock so the demo is instant
breaker = CircuitBreaker(failure_threshold=2, reset_timeout=10, clock=lambda: now[0])

def service(ok):
    if not ok:
        raise ConnectionError("service down")
    return "response"

def attempt(ok):
    try:
        result = breaker.call(service, ok)
        print(f"state={breaker.state:9} ok: {result}")
    except (ConnectionError, CircuitOpen) as e:
        print(f"state={breaker.state:9} {type(e).__name__}: {e}")

attempt(False)
attempt(False)                                        # second failure opens the circuit
attempt(True)                                         # fails fast: the service is not even called
now[0] = 11                                           # time passes beyond reset_timeout
attempt(True)                                         # half-open trial succeeds, circuit closes
attempt(True)
```

**Output:**

```text
state=closed    ConnectionError: service down
state=open      ConnectionError: service down
state=open      CircuitOpen: circuit is open; failing fast
state=closed    ok: response
state=closed    ok: response
```

Injecting the `clock` made the time-dependent logic testable without sleeping. In production use a maintained library, and add locking if the breaker is shared between threads.

---

## 36. Event Bus (Publish/Subscribe)

**Intent:** decouple publishers from subscribers completely, using named topics instead of direct references.

**Use when:** many unrelated parts of a system react to events ("order created", "file uploaded") and the publisher shouldn't know who listens.

```python
from collections import defaultdict

class EventBus:
    def __init__(self):
        self._handlers = defaultdict(list)

    def subscribe(self, topic, handler):
        self._handlers[topic].append(handler)

    def publish(self, topic, **payload):
        delivered = 0
        for handler in list(self._handlers[topic]):
            try:
                handler(**payload)
                delivered += 1
            except Exception as exc:                  # one bad subscriber must not break the rest
                print(f"handler {handler.__name__} failed: {exc}")
        return delivered

bus = EventBus()

def send_email(order_id, total):
    print(f"email: order {order_id} confirmed")

def update_stock(order_id, total):
    print(f"stock: reserved items for order {order_id}")

def broken_handler(order_id, total):
    raise RuntimeError("analytics offline")

for handler in (send_email, update_stock, broken_handler):
    bus.subscribe("order.created", handler)

print("delivered to", bus.publish("order.created", order_id=42, total=99.5), "handlers")
print("delivered to", bus.publish("order.cancelled", order_id=42, total=0), "handlers")
```

**Output:**

```text
email: order 42 confirmed
stock: reserved items for order 42
handler broken_handler failed: analytics offline
delivered to 2 handlers
delivered to 0 handlers
```

**Observer vs. Event Bus:** an Observer is attached to one specific subject; an Event Bus is a shared middleman addressed by topic. For cross-process messaging, the same shape is provided by message brokers and queues. **Pitfalls:** it becomes hard to trace "what happens when X is published", and a synchronous bus makes publishers wait for every subscriber.

---

# Part V: Putting It All Together

---

## 37. Telling Similar Patterns Apart

Many patterns have a similar *structure* but a different *intent*. The intent is what you name in a code review.

### Wrappers: Adapter, Decorator, Proxy, Facade

| Pattern | Wraps | Interface | Purpose |
|---|---|---|---|
| Adapter | one object | **changes** it to a different interface | make incompatible things fit |
| Decorator | one object | **same** interface | add behavior |
| Proxy | one object | **same** interface | control access (lazy, permissions, cache) |
| Facade | many objects | a **new, simpler** interface | hide complexity |

### Creating objects

| Pattern | Creates | How |
|---|---|---|
| Factory Method | one product | a method that subclasses override |
| Abstract Factory | a family of products | a factory object passed in |
| Builder | one complex object | step by step, then `build()` |
| Prototype | a copy | clone an existing object |
| Singleton | the only instance | cached, returned every time |

### Swappable behavior: Strategy, State, Template Method, Command

| Pattern | Who decides what runs | Typical shape |
|---|---|---|
| Strategy | the **client** picks an algorithm | a function or small object passed in |
| State | the **object itself** as it changes state | state objects that swap themselves |
| Template Method | the **base class** fixes the order; subclasses fill in steps | inheritance |
| Command | whoever **holds** the command decides when to run it | an object (or callable) wrapping an action |

### Communication: Observer, Mediator, Event Bus, Chain of Responsibility

| Pattern | Shape |
|---|---|
| Observer | one subject notifies its many observers |
| Mediator | peers talk through one coordinator, with two-way logic |
| Event Bus | publishers and subscribers meet on named topics |
| Chain of Responsibility | one request travels until someone handles it |

---

## 38. Combining Patterns

Real designs mix patterns. Some natural pairings:

- **Factory + Strategy:** a factory chooses which strategy object to build from configuration.
- **Command + Memento:** commands record the snapshots they need to undo themselves.
- **Composite + Visitor:** a visitor performs operations over every node of a composite tree.
- **Composite + Iterator:** a generator walks the tree (as `walk()` did in section 12).
- **Decorator + Strategy:** wrappers add logging or caching around an interchangeable algorithm.
- **Observer + Mediator / Event Bus:** change notifications flow through a central coordinator.
- **Repository + Unit of Work + Dependency Injection:** the business layer receives all of them and can be tested with in-memory fakes.
- **Retry + Circuit Breaker:** retry absorbs brief blips; the breaker stops retries from hammering a service that is really down.

---

## 39. Anti-Patterns and When *Not* to Use a Pattern

| Anti-pattern | Why it hurts | Better |
|---|---|---|
| **Pattern fever** (patterns for their own sake) | More classes, more indirection, no benefit | Start simple; add a pattern when a real change forces it |
| **Singleton as a global** | Hidden dependencies, test leakage | Dependency injection, a module-level object |
| **Java in Python** (a class for everything, getters and setters) | Verbose, un-Pythonic | Functions, `dataclass`, properties |
| **Deep inheritance trees** | Fragile, hard to follow | Composition, small protocols |
| **God object / god mediator** | One class knows and does everything | Split by responsibility |
| **Service locator** | A hidden global in disguise | Explicit constructor arguments |
| **Premature abstraction** (an interface with one implementation, "just in case") | Cost now for a benefit that may never come | Wait for the second use case |
| **Mutable flyweights / shared state** | A change in one place breaks others | Immutable shared objects |
| **Visitor on a changing hierarchy** | Every new element class breaks every visitor | `singledispatch`, or plain methods |

**A useful test before adding a pattern:** what concrete change am I making easier? If you can't name one, don't add it. And the "Rule of Three": tolerate duplication until the third occurrence, then abstract.

---

## 40. Patterns You Already Use From the Standard Library

| Pattern | Where Python gives it to you |
|---|---|
| Iterator | the `for` loop, generators, `iter()` / `next()` |
| Decorator | `functools.wraps`, `functools.lru_cache`, wrapper streams in `io` |
| Adapter | `io.TextIOWrapper` (adapts a bytes stream to a text interface) |
| Singleton / Registry | `logging.getLogger("name")` returns the same logger every time; modules via `sys.modules` |
| Flyweight | `sys.intern`, the small-integer cache |
| Prototype | `copy.copy`, `copy.deepcopy` |
| Strategy | `sorted(key=...)`, `max(key=...)` |
| Command | `functools.partial`, `threading.Thread(target=...)` |
| Template Method | `unittest.TestCase`, subclassing `json.JSONEncoder` and overriding `default` |
| Visitor | `ast.NodeVisitor` |
| Interpreter | `re` (a pattern language), `ast.literal_eval` |
| Composite | `xml.etree.ElementTree` elements, the `ast` tree |
| Null Object | `contextlib.nullcontext`, `os.devnull` |
| Proxy | `weakref.proxy` |
| Facade | `shutil` over file operations, `subprocess.run` over `Popen` |
| Object Pool | `concurrent.futures.ThreadPoolExecutor`, `multiprocessing.Pool` |
| Memento | `pickle` (snapshot and restore object state) |
| Chain of Responsibility | log records propagating up the `logging` logger hierarchy |

A few of these are easy to see in action:

```python
import ast
import copy
import io
import logging

print(logging.getLogger("app") is logging.getLogger("app"))          # registry: same object per name

print(io.TextIOWrapper(io.BytesIO(b"hello")).read())                  # adapter: bytes stream read as text

class NameCounter(ast.NodeVisitor):                                   # visitor over Python's own syntax tree
    def __init__(self):
        self.count = 0

    def visit_Name(self, node):
        self.count += 1

visitor = NameCounter()
visitor.visit(ast.parse("a = b + c"))
print("names in 'a = b + c':", visitor.count)

original = [[1, 2], [3]]
print(copy.deepcopy(original) == original, copy.deepcopy(original) is original)   # prototype: equal but separate
```

**Output:**

```text
True
hello
names in 'a = b + c': 3
True False
```

---

## 41. Interview Questions and Answers

**What is a design pattern, and why use one?**
A named, reusable solution to a recurring design problem. The value is a shared vocabulary ("use a Strategy here") and proven structures that keep code flexible, not code you copy.

**What are the three categories of patterns?**
Creational (object creation), Structural (composing objects), Behavioral (communication and responsibility). See the table in section 3 for matching a problem to a pattern.

**Why are some patterns "invisible" in Python?**
Python has first-class functions, duck typing, modules, generators, decorators and context managers, so Strategy, Command, Iterator, Singleton and others collapse into a function, a generator or a module-level object.

**How do you implement a Singleton in Python, and what are its dangers?**
The simplest way is a module-level instance. Other ways are `__new__` (remember `__init__` runs on every call), a metaclass, or `functools.cache` on a getter. The danger is that it is hidden global state that makes testing hard; prefer dependency injection.

**Factory Method vs. Abstract Factory vs. Builder?**
Factory Method creates one product through an overridable method. Abstract Factory creates a *family* of matching products. Builder assembles one complex object *step by step*.

**Strategy vs. State?**
Same structure, different intent. In Strategy the client chooses an interchangeable algorithm. In State the object changes its own behavior object as it moves between states.

**Decorator vs. Proxy vs. Adapter?**
All wrap an object. Adapter changes the interface, Decorator adds behavior behind the same interface, Proxy controls access behind the same interface.

**How is the Decorator *pattern* related to Python's `@decorator` syntax?**
`@decorator` applies the same idea to functions (a wrapper with the same call signature). The Decorator pattern wraps *objects* that share an interface. See `09-Decorators.md`.

**How does Observer cause memory leaks, and how do you avoid it?**
A subject holding strong references keeps observers alive forever. Use `weakref.WeakMethod`/`WeakSet`, or require explicit unsubscribe.

**What is double dispatch, and how does Visitor use it?**
Choosing the method by the types of *two* objects: first `element.accept(visitor)` dispatches on the element, then the visitor's `visit_x` dispatches on the visitor. In Python `functools.singledispatch` gives you much of this directly.

**What problem does Flyweight solve, and what must be true of the shared object?**
Memory duplication across many similar objects. The shared (intrinsic) state must be immutable.

**Shallow copy vs. deep copy, and which pattern relies on it?**
A shallow copy shares nested mutable objects; a deep copy recursively copies them. Prototype relies on copying (`copy.copy` / `copy.deepcopy`).

**What is dependency injection, and how does it relate to patterns?**
Passing collaborators in rather than creating them inside. It implements Dependency Inversion, makes testing easy with fakes, and is the clean alternative to Singleton and global lookups.

**When would you use a Circuit Breaker together with Retry?**
Retry handles brief transient failures; the breaker stops repeated retries from overwhelming a dependency that is genuinely down, letting callers fail fast.

**How do you decide whether to use a pattern at all?**
Name the specific change it makes easier. If you can't, don't add it. Prefer the simple Pythonic form (function, module, dataclass) first.

---

## 42. Quick Reference

| # | Pattern | One-line intent | Pythonic form |
|---|---|---|---|
| 4 | Singleton | One shared instance | Module-level object, `functools.cache` |
| 5 | Factory Method | Subclass or config chooses the class to create | Function, `dict` registry, `@classmethod` |
| 6 | Abstract Factory | Create matching families of objects | `dict` of classes, factory object passed in |
| 7 | Builder | Build a complex object step by step | Keyword args, `dataclass` + `replace` |
| 8 | Prototype | Create by copying | `copy.copy` / `copy.deepcopy` |
| 9 | Object Pool | Reuse expensive objects | `queue.Queue` + context manager |
| 10 | Adapter | Make an interface fit | Wrapper class, `functools.partial` |
| 11 | Bridge | Two independent hierarchies joined by composition | Pass the implementation in |
| 12 | Composite | Treat a tree and its parts uniformly | Recursive classes, generators |
| 13 | Decorator | Add behavior by wrapping | `@decorator`, wrapper classes |
| 14 | Facade | A simple front for a complex subsystem | A module or small class |
| 15 | Flyweight | Share identical state | `__slots__`, caching, interning |
| 16 | Proxy | Control access to an object | `__getattr__`, `weakref.proxy` |
| 17 | Chain of Responsibility | Pass a request along until handled | List of functions (middleware) |
| 18 | Command | Make an action an object | Callable, `functools.partial` |
| 19 | Iterator | Traverse without exposing structure | `__iter__`, generators |
| 20 | Mediator | Peers communicate through one coordinator | A coordinator class |
| 21 | Memento | Save and restore state | Frozen `dataclass`, `copy`, `pickle` |
| 22 | Observer | Notify dependents of changes | Callbacks, weak references |
| 23 | State | Behavior changes with internal state | State classes, or enum + transition table |
| 24 | Strategy | Interchangeable algorithms | Pass a function |
| 25 | Template Method | Fixed skeleton, variable steps | Abstract base class with hooks |
| 26 | Visitor | New operations without editing classes | `functools.singledispatch` |
| 27 | Interpreter | Evaluate a small language | Expression-tree classes + a parser |
| 28 | Dependency Injection | Supply dependencies from outside | Constructor arguments |
| 29 | Registry / Plugin | Find implementations by name | Decorator, `__init_subclass__` |
| 30 | Null Object | A do-nothing stand-in for `None` | Small class with no-op methods |
| 31 | Repository / Unit of Work | Hide storage; group changes atomically | Abstract class + context manager |
| 32 | Specification | Combinable business rules | Callables with `&`, `\|`, `~` |
| 33 | Pipeline | Stages feeding stages | Chained generators |
| 34 | Retry | Repeat transient failures with backoff | Decorator |
| 35 | Circuit Breaker | Fail fast when a dependency is down | State machine class |
| 36 | Event Bus | Publish/subscribe by topic | Dict of topic to handlers |

**The core habits:** program to an interface (a `Protocol` or duck typing), favor composition over inheritance, prefer the simple Pythonic form, and add a pattern only when you can name the change it makes easier.

---

## 43. Practice Exercises

1. Turn the `Order` State example into one where `cancel()` is allowed from `draft` and `paid` but not once shipped.
2. Add a `PushChannel` to the Bridge example and an `Urgent` notification type; count how many classes you changed.
3. Give the Command editor a `Replace` command with undo and redo support.
4. Extend the Interpreter to support `in` (for example `city in ('Pune', 'Delhi')`).
5. Rewrite the Template Method ETL job as a function that takes `extract`, `transform` and `load` callables, and compare the two designs.
6. Add topic wildcards (such as `order.*`) to the Event Bus.
7. Write a Registry-based plugin loader that discovers classes from a list of module names with `importlib.import_module`.
8. Make the Circuit Breaker thread-safe with a lock, and write a test using a fake clock.

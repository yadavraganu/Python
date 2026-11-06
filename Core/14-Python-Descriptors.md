## 1. What Is a Descriptor?

A **descriptor** is any object whose class implements one or more of `__get__`, `__set__`, or `__delete__`. When such an object is stored as a **class attribute**, Python routes normal attribute access (`instance.attr`) through those methods instead of simply returning the stored value.

```python
class Ten:
    def __get__(self, obj, owner):
        return 10

class A:
    x = Ten()          # x is a descriptor, stored on the class

a = A()
print(a.x)              # 10 — not the Ten instance itself, but __get__'s return value
```

This is the foundation for `@property`, bound methods, `@classmethod`, `@staticmethod`, and `__slots__` — they're all implemented as descriptors under the hood.

## 2. The Descriptor Protocol

| Method | Signature | Triggered by |
| --- | --- | --- |
| `__get__` | `__get__(self, obj, objtype=None)` | `instance.attr` or `ClassName.attr` |
| `__set__` | `__set__(self, obj, value)` | `instance.attr = value` |
| `__delete__` | `__delete__(self, obj)` | `del instance.attr` |
| `__set_name__` | `__set_name__(self, owner, name)` | Automatically, when the class body finishes executing |

A descriptor doesn't need to implement all of them, but **which methods it defines changes how it behaves** (see Section 3). Here is a descriptor with only `__get__`:

```python
class ReadOnly:
    def __init__(self, value):
        self.value = value

    def __get__(self, obj, objtype=None):
        return self.value

    # no __set__ defined

class Config:
    version = ReadOnly("1.0")

c = Config()
print(c.version)         # 1.0
c.version = "2.0"        # no error! (explained below)
print(c.version)         # 2.0 — the assignment created an instance attribute that now shadows the descriptor
print(Config.version)    # 1.0 — the class-level descriptor is untouched
```

With only `__get__` it is a **non-data descriptor**, so assignment doesn't go through it at all: Python just stores `"2.0"` in the instance `__dict__`, which wins on later lookups. To make an attribute genuinely read-only, define `__set__`, even if it only raises:

```python
class ReadOnly:
    def __init__(self, value):
        self.value = value

    def __get__(self, obj, objtype=None):
        return self.value

    def __set__(self, obj, value):
        raise AttributeError("read-only attribute")

class Config:
    version = ReadOnly("1.0")

Config().version = "2.0"     # AttributeError: read-only attribute
```

Note `obj` is `None` when the descriptor is accessed via the class itself rather than an instance:

```python
class Descriptor:
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self          # convention: return the descriptor itself
        return "instance value"

class A:
    x = Descriptor()

print(A.x)          # <Descriptor object> — accessed via the class
print(A().x)         # "instance value" — accessed via an instance
```

### Functions Are Descriptors: Where Bound Methods Come From

Every function defined in a class body is a (non-data) descriptor, and this is the mechanism that turns `obj.method` into a bound method:

```python
class F:
    def f(self):
        return "hi"

obj = F()
bound = F.f.__get__(obj, F)                          # what `obj.f` does behind the scenes
print(bound())                                        # hi
print(bound == obj.f)                                 # True
print(obj.f.__self__ is obj, obj.f.__func__ is F.f)   # True True
print(hasattr(F.f, "__get__"), hasattr(type(F.f), "__set__"))   # True False — has __get__, no __set__
```
`staticmethod` and `classmethod` are descriptors too: `staticmethod.__get__` hands back the plain function, while `classmethod.__get__` binds the class instead of the instance.

## 3. Data vs. Non-Data Descriptors

This distinction controls **priority** during attribute lookup.

- **Data descriptor** — defines `__set__` and/or `__delete__` (with or without `__get__`). **Always wins** over an entry in the instance's `__dict__`.
- **Non-data descriptor** — defines only `__get__`. **Loses** to an entry in the instance's `__dict__` if one exists.

```python
class NonDataDescriptor:
    def __get__(self, obj, objtype=None):
        return "from descriptor"

class DataDescriptor:
    def __get__(self, obj, objtype=None):
        return "from descriptor"
    def __set__(self, obj, value):
        pass   # even a no-op __set__ makes it a data descriptor

class A:
    non_data = NonDataDescriptor()
    data = DataDescriptor()

a = A()
a.__dict__['non_data'] = "from instance dict"
a.__dict__['data'] = "from instance dict"

print(a.non_data)   # "from instance dict"  — instance dict wins
print(a.data)         # "from descriptor"     — data descriptor wins regardless
```

This is why `@property` (a data descriptor, since it defines `__set__` even if only to raise `AttributeError`) can't be silently shadowed by `self.attr = value` inside `__init__` — the property intercepts it.

## 4. How Attribute Lookup Actually Works

`instance.attr` is handled by `object.__getattribute__`, which roughly follows this algorithm:

```python
def object_getattribute(obj, name):
    cls = type(obj)
    meta_attr = find_in_mro(cls, name)

    # 1. Data descriptor on the class/MRO wins outright
    if meta_attr is not None and is_data_descriptor(meta_attr):
        return meta_attr.__get__(obj, cls)

    # 2. Instance __dict__ next
    if name in obj.__dict__:
        return obj.__dict__[name]

    # 3. Non-data descriptor or plain class attribute
    if meta_attr is not None:
        if is_non_data_descriptor(meta_attr):
            return meta_attr.__get__(obj, cls)
        return meta_attr          # plain class attribute, not a descriptor

    # 4. Fall back to __getattr__ if defined, else AttributeError
    raise AttributeError(name)
```

(This is a simplified illustration of the real C implementation, but it captures the priority order correctly.)

### Lookup on the Class Itself

Reading an attribute through the class (`A.x`) goes through `type.__getattribute__`, which searches the class and its bases and, if the attribute it finds has `__get__`, calls `__get__(None, A)`. That is why descriptors are expected to handle `obj is None`.

## 5. `__set_name__` — Knowing Your Own Name

Added in Python 3.6, `__set_name__` is called automatically once, right after the class body is executed, telling the descriptor what attribute name it was assigned to — so it doesn't need to be passed manually.

```python
class LoggedAttribute:
    def __set_name__(self, owner, name):
        self.public_name = name
        self.private_name = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name)

    def __set__(self, obj, value):
        print(f"Setting {self.public_name} to {value!r}")
        setattr(obj, self.private_name, value)

class Person:
    name = LoggedAttribute()
    age = LoggedAttribute()

    def __init__(self, name, age):
        self.name = name
        self.age = age

p = Person("Ana", 30)
# Setting name to 'Ana'
# Setting age to 30
```

Without `__set_name__`, you'd have to pass the attribute name redundantly to every descriptor instance (`name = LoggedAttribute("name")`), which is error-prone.

### `__set_name__` Caveats

- It is called once per descriptor during **class creation**. A descriptor attached afterwards (`H.y = Desc()`) never receives its name.
- If one descriptor instance is assigned to two attribute names, `__set_name__` is called for each, which is how `functools.cached_property` detects and rejects that.
- In Python 3.12 an exception raised inside `__set_name__` propagates unchanged, with a note such as `Error calling __set_name__ on 'Bad' instance 'b' in 'Z'`. Before 3.12 it was wrapped in a `RuntimeError`.

## 6. Storage Strategies (and a Classic Bug)

### The bug: storing state directly on the descriptor

```python
class Broken:
    def __init__(self):
        self.value = None            # WRONG: one Broken instance is shared by ALL objects

    def __get__(self, obj, objtype=None):
        return self.value

    def __set__(self, obj, value):
        self.value = value           # overwrites the SAME storage for every instance

class A:
    x = Broken()

a1, a2 = A(), A()
a1.x = "first"
a2.x = "second"
print(a1.x)     # "second" — BUG: a1 and a2 share the same descriptor instance
```

Because the descriptor object itself is a **class attribute**, there is only one instance of it shared by every `A()`. Storing the value directly on `self` inside the descriptor leaks state across all instances.

### Fix 1 — store on the instance's `__dict__` (most common)

```python
class Fixed:
    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        obj.__dict__[self.name] = value    # stored per-instance, keyed by the owning object

class A:
    x = Fixed()

a1, a2 = A(), A()
a1.x, a2.x = "first", "second"
print(a1.x, a2.x)      # "first second" — correct, independent storage
```

### Fix 2 — a `WeakKeyDictionary` (when instances have no `__dict__`)

```python
import weakref

class SlottedDescriptor:
    def __init__(self):
        self.data = weakref.WeakKeyDictionary()

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return self.data.get(obj)

    def __set__(self, obj, value):
        self.data[obj] = value

class A:
    __slots__ = ("__weakref__",)    # no per-instance __dict__, but instances can still be weakly referenced
    x = SlottedDescriptor()
```

`WeakKeyDictionary` avoids keeping instances alive forever just because the descriptor references them. Two caveats:

- Instances must support weak references. With a plain `__slots__ = ()`, setting the attribute fails with `TypeError: cannot create weak reference to 'A' object`; add `"__weakref__"` to `__slots__` as above.
- The instances are used as dictionary keys, so they must be hashable, and two instances that compare equal and hash equal would share one entry.

### Fix 3 — store the value in a slot named by `__set_name__` (best fit for `__slots__`)

```python
class SlotStore:
    def __set_name__(self, owner, name):
        self.slot = "_" + name

    def __get__(self, obj, objtype=None):
        return self if obj is None else getattr(obj, self.slot)

    def __set__(self, obj, value):
        setattr(obj, self.slot, value)

class B:
    __slots__ = ("_x",)            # the descriptor stores its value in this slot
    x = SlotStore()

b = B()
b.x = 9
print(b.x)    # 9
```

## 7. Building `@property` From Scratch

`property` is literally a built-in data descriptor. Here's a functioning re-implementation:

```python
class MyProperty:
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def setter(self, fset):             # supports the @x.setter syntax
        return MyProperty(self.fget, fset, self.fdel)

class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @MyProperty
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        self._celsius = value

t = Temperature(20)
print(t.celsius)       # 20
t.celsius = 25
print(t.celsius)       # 25
```

## 8. Real-World Descriptor Patterns

### Type-checked / validated attributes

```python
class Typed:
    def __init__(self, expected_type):
        self.expected_type = expected_type

    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(f"{self.name[1:]} must be a {self.expected_type.__name__}")
        obj.__dict__[self.name] = value

class Employee:
    name = Typed(str)
    salary = Typed(int)

    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

Employee("Sam", "not a number")   # raises TypeError
```

### A single reusable `Positive` validator across many attributes

```python
class Positive:
    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self                 # `Product.price` returns the descriptor instead of crashing
        return obj.__dict__[self.name]

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name[1:]} must be positive")
        obj.__dict__[self.name] = value

class Product:
    price = Positive()
    quantity = Positive()

    def __init__(self, price, quantity):
        self.price = price
        self.quantity = quantity
```

### Lazy, computed-once attributes

```python
class LazyProperty:
    def __init__(self, func):
        self.func = func
        self.name = func.__name__

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        value = self.func(obj)
        obj.__dict__[self.name] = value   # cache: overrides the descriptor next time
        return value

class Report:
    def __init__(self, rows):
        self.rows = rows

    @LazyProperty
    def total(self):
        print("Computing total...")
        return sum(self.rows)

r = Report([1, 2, 3])
print(r.total)     # "Computing total..." then 6
print(r.total)     # 6 — no recomputation; found directly in r.__dict__ this time
```

This works because `total` is a **non-data descriptor** here (no `__set__`), so once the computed value is written into `obj.__dict__['total']`, the instance dict takes priority on future lookups and the function never runs again.

## 9. Descriptors and Inheritance

Descriptors defined on a base class work automatically for subclasses, since class-attribute lookup follows the MRO:

```python
class Base:
    x = Positive()

class Sub(Base):
    pass

s = Sub()
s.x = 5          # works — Positive() is found via Sub's MRO
```

Descriptors can also be overridden per-subclass by simply reassigning the attribute name to a new descriptor instance in the subclass body.

## 10. `functools.cached_property`

The standard library provides a ready-made version of the lazy-attribute pattern from Section 8:

```python
from functools import cached_property

class Report:
    def __init__(self, rows):
        self.rows = rows

    @cached_property
    def total(self):
        print("Computing total...")
        return sum(self.rows)

r = Report([1, 2, 3])
print(r.total)     # computes, caches
print(r.total)     # cached, no recomputation
```

Unlike `@property`, `cached_property` is a **non-data descriptor** by design — exactly so the cached value in `obj.__dict__` can take over on subsequent lookups, same as the hand-rolled `LazyProperty` above.

Behaviors worth knowing:

- **Clearing the cache:** `del r.total` removes the cached value, so the next access recomputes it.
- **It needs an instance `__dict__`.** On a class with `__slots__ = ()` it raises `TypeError: No '__dict__' attribute on 'CPS' instance to cache 'v' property.`
- **One name only.** Assigning the same `cached_property` to two names raises `TypeError: Cannot assign the same cached_property to two different names`.
- **Threads:** since Python 3.12 `cached_property` takes no lock, so in multi-threaded code the getter can run more than once for the same instance. Before 3.12 an internal lock serialized calls but caused contention between unrelated instances.

## 11. Descriptors vs. `__getattr__`/`__getattribute__`

These solve related but distinct problems:

| Mechanism | Defined on | Scope | Typical use |
| --- | --- | --- | --- |
| Descriptor (`__get__`/`__set__`) | The attribute's own class | One specific named attribute | Validation, computed/cached values, type checking |
| `__getattr__` | The owning class | Any attribute not found normally | Fallback/proxy behavior, dynamic attributes |
| `__getattribute__` | The owning class | **Every** attribute access | Rarely overridden directly; this is what implements descriptor lookup in the first place |

## 12. Performance Notes

- Descriptor `__get__`/`__set__` calls add function-call overhead compared to a plain attribute — usually negligible, but avoid heavy descriptors in extremely hot loops.
- `__slots__` (itself descriptor-based) removes the per-instance `__dict__`, saving memory at scale — useful for classes with millions of instances.
- `cached_property` trades a small amount of memory (the cached value in `__dict__`) for avoiding repeated computation — a good default for expensive derived values that don't change.

## 13. Common Pitfalls

1. **Storing state on `self` inside the descriptor** instead of on the instance — causes state to leak across all instances sharing that descriptor (see Section 6).
2. **Forgetting `if obj is None: return self`** in `__get__` — breaks introspection like `ClassName.attribute`.
3. **Defining `__set__` unintentionally** (even as a no-op) — turns a non-data descriptor into a data descriptor, changing priority versus the instance `__dict__`.
4. **Using `__slots__` alongside a descriptor that assumes `obj.__dict__` exists** — it will fail; store the value in a named slot (Fix 3 in Section 6), or use a `WeakKeyDictionary` and include `"__weakref__"` in `__slots__`.
5. **Not handling `__set_name__` when the same descriptor class backs several attributes** — without it, each attribute needs its storage key passed explicitly, which is easy to get wrong.
6. **Expecting a descriptor without `__set__` to be read-only** — assignment silently shadows it in the instance `__dict__` (Section 2). Define `__set__` and raise.
7. **Assigning to the attribute name on the *class*** (`W.x = 5`) replaces the descriptor entirely; `__set__` only intercepts assignment through an instance.
8. **Attaching a descriptor after class creation** — `__set_name__` is never called for it.
9. **Built-in `property` error wording:** a property without a setter raises `AttributeError: property 'x' of 'T' object has no setter` (3.12 wording; older versions said `can't set attribute`).

## 14. Practice Exercises

1. Write a `Choices` descriptor that only allows a value from a fixed set (e.g., `status = Choices("pending", "active", "closed")`).
2. Build a `TypedList` descriptor that enforces every item appended to a list attribute matches a given type.
3. Implement your own minimal `cached_property` from scratch, verifying it only computes the wrapped function once per instance.
4. Create a `Range` descriptor that validates a numeric value falls between a min and max, raising `ValueError` otherwise.
5. Combine two descriptors (e.g., `Positive` and `Typed`) on the same class and confirm both validations run when the attribute is set.

## 15. Quick Reference

| Concept | Detail |
| --- | --- |
| Descriptor | Object whose class defines `__get__`, `__set__`, and/or `__delete__` |
| Data descriptor | Has `__set__` and/or `__delete__` — always wins over instance `__dict__` |
| Non-data descriptor | Has only `__get__` — loses to instance `__dict__` |
| `__set_name__` | Auto-called with `(owner, name)` when the class body finishes |
| `obj is None` in `__get__` | Means access happened via the class, not an instance |
| Storage rule | Store per-instance state in `obj.__dict__`, not `self`, inside the descriptor |
| Built on descriptors | `@property`, methods, `@classmethod`, `@staticmethod`, `__slots__`, `functools.cached_property` |

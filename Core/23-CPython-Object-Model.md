# CPython Object Model & Memory Internals

This covers what actually happens in memory beneath Python's high-level syntax — the object model, the custom small-object allocator, and how `dict`, `list`, and `str` are represented internally.

## 1. The CPython Object Model — `PyObject`

Every single Python object — an `int`, a `list`, a custom class instance, even a function is, in C, a `PyObject*` (a pointer to a `PyObject` struct). Every such struct shares a common header:

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;   // reference count
    PyTypeObject *ob_type;   // pointer to the object's type
} PyObject;
```

- **`ob_refcnt`** — The reference count. This is the mechanism `sys.getrefcount(obj)` reads, and it's the foundation of CPython's **primary** garbage-collection strategy: when it hits zero, the object is immediately deallocated.
- **`ob_type`** — A pointer to the object's type object, which is itself a `PyObject` (specifically a `PyTypeObject`) describing the class: its methods, its `__dict__`, its MRO. This is what `type(obj)` actually returns, and why "everything is an object, including classes" is literally true at the C level — classes are instances of `type`.

```python
import sys

x = []
print(sys.getrefcount(x))      # ref count (includes the temporary ref from the call itself)
print(type(x) is list)         # True — ob_type points at the `list` type object
print(type(list) is type)      # True — even `list` itself is an instance of `type`
```

## 2. Memory Allocation — `pymalloc`

Typical Python programs allocate huge numbers of small, short-lived objects (every `int`, every small `list`, every stack frame). Calling the OS-level `malloc()` for each one would be far too slow and fragment memory badly. CPython instead uses a custom small-object allocator, **`pymalloc`**, layered in three tiers:

- **Arenas** — Large chunks (256 KB) requested directly from the OS.
- **Pools** — Each arena is divided into pools (4 KB each), and each pool is dedicated to allocations of one specific *size class*.
- **Blocks** — Each pool is divided into fixed-size blocks; a request for, say, a 32-byte object is served from the pool whose blocks are sized for that class.

`pymalloc` handles objects **under ~512 bytes**; larger allocations fall through to the system allocator directly. This tiered design means small-object allocation and deallocation are extremely cheap — mostly bookkeeping, no syscalls — which is a big part of why Python's object churn (creating/discarding tons of small objects) doesn't tank performance as badly as it otherwise would.

This also connects to why CPython rarely returns memory to the OS eagerly — arenas are only freed when *completely* empty, so a program that briefly allocates a huge number of small objects can keep that memory reserved (though not leaked) for the rest of the process's life.

## 3. `dict` Internals

Since Python 3.6 (officially guaranteed from 3.7), dicts are both **fast** and **insertion-ordered**, via a "compact dict" design:

- A dict stores **two structures**: a sparse hash table of indices, and a dense array of the actual key/value/hash entries in **insertion order**. Iterating a dict walks the dense array directly — which is *why* insertion order is preserved essentially for free, rather than as extra bookkeeping.
- **Key-sharing dicts (PEP 412):** when many instances of the same class have identical attribute names (the normal case), their per-instance `__dict__`s can share one common "keys" table, storing only the *values* separately per instance. This significantly cuts memory use for typical object-heavy programs.
- Average-case `O(1)` lookup, insert, and delete, same as any hash table — collisions are handled via open addressing with a specific perturbation scheme, not chaining.

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p1, p2 = Point(1, 2), Point(3, 4)
# p1.__dict__ and p2.__dict__ have identical *keys* ('x', 'y') —
# CPython can share that keys-table between them, storing only the values separately.
```

## 4. `list` Internals — Over-Allocation

A Python `list` is a dynamic array (a contiguous block of pointers to `PyObject`s), not a linked list. Appending would be `O(n)` every time if it resized to the exact new length on each `append()` — instead, CPython **over-allocates**, growing the underlying buffer by roughly ~12.5% extra (a specific growth pattern, not just "double") whenever it needs to resize.

This is why:
- `list.append()` is **amortized O(1)** — most calls just write into already-reserved space; resizes happen infrequently and get amortized across many cheap appends.
- `list.insert(0, x)` is **O(n)** — inserting at the front requires shifting every existing element's pointer over by one slot, unlike `append`.
- `len(a_list)` is `O(1)` — CPython stores the current length directly (`ob_size`) rather than counting elements, unlike some naive implementations.

## 5. String Representation — PEP 393 (Flexible String Representation)

Before Python 3.3, every `str` internally used a fixed width per character (effectively always reserving room for the widest possible Unicode codepoint), wasting memory for strings that were pure ASCII or Latin-1. **PEP 393** made CPython choose the **narrowest sufficient representation** per string, based on the largest codepoint actually present:

| Representation | Used when the string contains... | Bytes per char |
|---|---|---|
| Latin-1 (`PyASCIIObject`/compact) | Only codepoints ≤ 0xFF | 1 |
| UCS-2 | At least one codepoint > 0xFF but ≤ 0xFFFF | 2 |
| UCS-4 | At least one codepoint > 0xFFFF (e.g., many emoji) | 4 |

```python
import sys
print(sys.getsizeof("hello"))     # smaller — pure ASCII/Latin-1 storage
print(sys.getsizeof("héllo"))      # larger — needs at least Latin-1/UCS-2 range... 
print(sys.getsizeof("👋"))          # larger still — needs UCS-4 for a codepoint this high
```

This is a nice concrete answer if asked "how does Python store Unicode strings internally" — it's not one-size-fits-all, and the choice is made per-string automatically at creation time, invisible at the Python level except through memory-size differences.

## Notes & Gotchas Worth Knowing for Interviews

- **`type` is itself an instance of `type`** — the "everything is an object" claim bottoms out at `type(type) is type`, a fun one-liner to demonstrate real understanding.
- **Small-object allocation is cheap by design (`pymalloc`)** — this is part of why "just create more small objects" is a less costly habit in Python than in a language without a tiered small-object allocator, though it's still not free.
- **Key-sharing dicts** are why instances of a simple class are more memory-efficient than a raw per-instance dict with no shared structure would suggest.
- **`list.append` is amortized O(1); `list.insert(0, ...)` is O(n)`** — a very common Big-O gotcha; use `collections.deque` if you need efficient operations at both ends.
- **Not all strings cost the same per character** — PEP 393 means `sys.getsizeof` on two equal-length strings can differ if their codepoints differ, which surprises people who assume `str` is a fixed-width type internally.

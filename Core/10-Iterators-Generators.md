## 1. Iterables vs. Iterators — The Core Distinction

These two terms get used interchangeably in casual conversation but mean precisely different things — a very common opening interview question.

- **Iterable:** anything you can call `iter()` on to get an iterator. It implements `__iter__`. Examples: `list`, `tuple`, `str`, `dict`, `set`.
- **Iterator:** an object that produces values one at a time via `__next__`, and remembers its position between calls. It implements **both** `__iter__` (returning itself) and `__next__`. Examples: what `iter([1,2,3])` returns; a generator object.

```python
nums = [1, 2, 3]          # nums is ITERABLE (has __iter__)
it = iter(nums)             # it is an ITERATOR (has __iter__ AND __next__)

print(next(it))   # 1
print(next(it))   # 2
print(next(it))   # 3
print(next(it))   # raises StopIteration — exhausted
```

**Key distinction to state clearly if asked:** every iterator is an iterable (its `__iter__` just returns `self`), but not every iterable is an iterator — a `list` is iterable but has no `__next__`, so `next(nums)` raises `TypeError`. This asymmetry is exactly why you can loop over the same list twice but can't loop over the same *exhausted* iterator once.

### The Iterator Protocol

```python
class Countdown:
    """A custom iterator counting down to zero."""
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self   # an iterator's __iter__ returns itself

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for n in Countdown(3):
    print(n)   # 3, 2, 1
```

`for` loops, `list()`, `sum()`, unpacking, and every other iteration construct in Python all work by calling `iter()` on the object once, then repeatedly calling `next()` until `StopIteration` is raised — that exception is how the protocol signals "done," not an error condition to be alarmed by.

### `iter()` with a Sentinel — the Lesser-Known Two-Argument Form

`iter(callable, sentinel)` repeatedly calls `callable()` until it returns `sentinel`, then stops — useful for reading until a marker value:

```python
with open("file.txt") as f:
    for line in iter(f.readline, ''):      # stops at the first '' (end of file)
        print(line, end='')

# Reading a binary file in fixed-size blocks, a classic use of the same form:
with open("data.bin", "rb") as f:
    sizes = [len(block) for block in iter(lambda: f.read(4096), b'')]
print(sizes)   # e.g. [4096, 4096, 1808] for a 10,000-byte file
```

### Which Objects Are Iterables vs. Iterators?

| Iterator (has `__next__`; one pass) | Iterable but **not** an iterator (re-iterable) |
|---|---|
| generator objects, generator expressions | `list`, `tuple`, `str`, `dict`, `set` |
| file objects (`iter(f) is f`) | `range` (`next(range(3))` raises `TypeError`) |
| `map`, `filter`, `zip`, `enumerate` (all lazy in Python 3) | `dict.keys()`, `.values()`, `.items()` views |
| what `iter(x)` returns, and `itertools` objects | most user-defined container classes |

### Making Your Own Iterable

The easiest way to write a re-iterable class is to make `__iter__` a generator method. Each `for` loop then gets a fresh generator:

```python
class Bag:
    def __init__(self, items):
        self.items = items

    def __iter__(self):
        for item in self.items:
            yield item

b = Bag([1, 2])
print(list(b), list(b))   # [1, 2] [1, 2] — can be looped over repeatedly
```
Contrast this with `Countdown` above, whose `__iter__` returns `self`: it's a one-shot iterator. As a last-resort fallback, Python also supports the **old sequence protocol**: an object with only `__getitem__` (taking 0, 1, 2, ... and raising `IndexError` at the end) can be iterated with `for` and `iter()`.

## 2. Generators — Functions That Yield

A **generator function** is any function containing `yield`. Calling it doesn't run the body — it immediately returns a **generator object**, which is an iterator that lazily runs the function body, pausing at each `yield` and resuming right where it left off on the next `next()` call.

```python
def count_up_to(n):
    i = 1
    while i <= n:
        yield i
        i += 1

gen = count_up_to(3)
print(type(gen))    # <class 'generator'>
print(next(gen))     # 1
print(next(gen))     # 2
print(next(gen))     # 3
print(next(gen))     # raises StopIteration
```

**`yield` vs. `return`:**
- `return` exits a function permanently, discarding all local state.
- `yield` **suspends** the function, preserving all local variables, the instruction pointer, and the call stack frame — exactly the frame-object mechanism described in `17-Internals.md`. The function resumes exactly where it left off on the next `next()` call, rather than restarting.

A generator function can still use a bare `return` to stop early — this simply raises `StopIteration` immediately (optionally carrying a value, accessible via the exception's `.value` in advanced use, e.g., inside `yield from`).

### Inspecting a Generator's State

```python
import inspect

def gen():
    yield 1

g = gen()
print(inspect.getgeneratorstate(g))   # GEN_CREATED  — not started yet
next(g)
print(inspect.getgeneratorstate(g))   # GEN_SUSPENDED — paused at a yield
try:
    next(g)
except StopIteration:
    pass
print(inspect.getgeneratorstate(g))   # GEN_CLOSED   — finished (a generator can also be GEN_RUNNING while executing)
```

### Why Generators Matter: Lazy Evaluation and Memory

A generator produces values **on demand**, one at a time, instead of building an entire collection in memory upfront.

```python
def squares_list(n):
    return [i * i for i in range(n)]     # builds the WHOLE list in memory

def squares_gen(n):
    for i in range(n):
        yield i * i                        # produces ONE value at a time

# For n = 10_000_000, squares_list allocates a huge list immediately;
# squares_gen uses effectively constant memory regardless of n.
total = sum(squares_gen(10_000_000))   # never holds more than one value at once
```
Measured on CPython 3.12: the generator object is about 192 bytes no matter how large `n` is, while a list of a million items needs roughly 8 MB for its pointers alone (the `int` objects are extra).

This is the standard answer to "why use a generator instead of a list" — constant memory usage, and the ability to represent **infinite** sequences, which a list fundamentally cannot do.

### Generator Expressions vs. List Comprehensions

Same syntax family, different evaluation strategy — a very frequently asked comparison:

```python
list_comp = [x * x for x in range(5)]     # eager — built immediately, a real list
gen_expr = (x * x for x in range(5))       # lazy — a generator, values computed on demand

print(list_comp)    # [0, 1, 4, 9, 16]
print(gen_expr)      # <generator object <genexpr> at 0x...>
print(list(gen_expr))  # [0, 1, 4, 9, 16] — must consume it to see values
```

**When to prefer which:**
- Use a **generator expression** when you'll only iterate once, don't need indexing/`len()`, and want to avoid building the full structure (e.g., feeding straight into `sum()`, `any()`, `join()`).
- Use a **list comprehension** when you need to iterate multiple times, need random access, need `len()`, or the data is small enough that eagerness doesn't matter.

### Generator Expressions: Only the First Iterable Is Evaluated Immediately

```python
try:
    g = (x for x in undefined_name)    # NameError right here, at creation: the outermost iterable is evaluated eagerly
except NameError:
    print("raised immediately")

y = 0
g = (x for x in range(3) if y == 0)    # the condition is evaluated lazily, per item
y = 1
print(list(g))                          # [] — by the time it runs, y is already 1
```

### Generator Pipelines

Chaining generators processes data one item at a time, so even a huge file never has to fit in memory:

```python
def read_lines(path):
    with open(path) as f:
        for line in f:
            yield line.rstrip("\n")

def non_empty(lines):
    return (line for line in lines if line)

def shout(lines):
    return (line.upper() for line in lines)

for line in shout(non_empty(read_lines("file.txt"))):
    print(line)
```
Each stage pulls from the one before it on demand; nothing is read until the final `for` loop asks for the next item.

## 3. Two-Way Communication: `send()`, `throw()`, `close()`

Generators aren't just one-way value producers — you can send values *into* a paused generator, inject exceptions, or terminate it early.

### `send()` — Pass a Value Into the Generator

```python
def echo():
    while True:
        received = yield
        print(f"Received: {received}")

gen = echo()
next(gen)          # "priming" the generator — must advance to the first yield before sending
gen.send("hello")   # Received: hello
gen.send("world")   # Received: world
```

`yield` used as an expression (`received = yield`) receives whatever value `.send()` passes in — the generator must already be paused at a `yield` for this to work, which is why the initial `next(gen)` "priming" call is required before the first `.send()`.

### `throw()` — Raise an Exception Inside the Generator

```python
def resilient_gen():
    try:
        while True:
            yield "running"
    except ValueError:
        yield "caught the ValueError, recovering"

gen = resilient_gen()
print(next(gen))              # "running"
print(gen.throw(ValueError))   # "caught the ValueError, recovering"
```

### `close()` — Terminate a Generator Early

```python
gen = resilient_gen()
next(gen)
gen.close()   # raises GeneratorExit inside the generator at the suspended yield point
```
`close()` raises `GeneratorExit` at the paused point and then marks the generator closed (`inspect.getgeneratorstate` reports `GEN_CLOSED`). The generator can use a `finally` block (or catch `GeneratorExit`) to clean up:

```python
def cleanup_gen():
    try:
        yield 1
    finally:
        print("finally ran on close()")

g = cleanup_gen()
next(g)
g.close()          # prints: finally ran on close()
```
What it must **not** do is `yield` again while handling `GeneratorExit`; that raises `RuntimeError: generator ignored GeneratorExit`. Letting `GeneratorExit` propagate, or simply returning, is correct. Generators are also closed automatically when garbage-collected.

## 4. `yield from` — Delegating to a Sub-Generator

`yield from` delegates iteration to another iterable/generator, automatically forwarding values out and `send()`/`throw()` calls in — without this, manually delegating would require an explicit loop re-implementing all of that plumbing.

```python
def inner():
    yield 1
    yield 2
    return "inner done"

def outer():
    result = yield from inner()   # forwards 1, 2 out; captures inner's return value
    print(f"inner returned: {result}")
    yield 3

for val in outer():
    print(val)
# 1
# 2
# inner returned: inner done
# 3
```

This is also the standard tool for flattening nested generators cleanly:
```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item

list(flatten([1, [2, 3, [4, 5]], 6]))   # [1, 2, 3, 4, 5, 6]
```

## 5. The `itertools` Module — Generator Building Blocks

The standard library's toolbox for constructing and combining lazy iterators without ever materializing a full list:

```python
import itertools

# Infinite iterators — only safe because generators are lazy
itertools.count(10)                     # 10, 11, 12, ... forever
itertools.cycle([1, 2, 3])               # 1, 2, 3, 1, 2, 3, ... forever
itertools.repeat("x", times=3)            # "x", "x", "x"

# Combinatoric generators
list(itertools.combinations([1, 2, 3], 2))   # [(1,2), (1,3), (2,3)]
list(itertools.permutations([1, 2], 2))       # [(1,2), (2,1)]
list(itertools.product([0, 1], repeat=2))      # [(0,0),(0,1),(1,0),(1,1)]

# Practical everyday tools
list(itertools.chain([1, 2], [3, 4]))          # [1, 2, 3, 4] — lazily concatenates
list(itertools.islice(itertools.count(), 5))    # [0, 1, 2, 3, 4] — slice an infinite generator
```

`itertools.count()`/`cycle()` are only usable at all *because* generators are lazy — a truly infinite list could never be built, but a truly infinite generator costs no memory since it only ever holds its current state.

A few more worth knowing:

```python
import itertools

list(itertools.pairwise([1, 2, 3]))          # [(1, 2), (2, 3)] — 3.10+
list(itertools.batched(range(5), 2))         # [(0, 1), (2, 3), (4,)] — 3.12+
a, b = itertools.tee([1, 2, 3])              # two independent iterators over one source
list(a), list(b)                             # ([1, 2, 3], [1, 2, 3])
```
`tee` buffers items for whichever copy lags behind, so don't keep using the original iterator after splitting it, and avoid it when one copy will run far ahead of the other.

## 6. Common Pitfalls & Gotchas

### Iterators/Generators Are Exhausted After One Full Pass
```python
gen = (x for x in range(3))
print(list(gen))   # [0, 1, 2]
print(list(gen))   # [] — already exhausted, cannot be restarted
```
Unlike a `list`, which you can iterate over repeatedly, a generator (and any iterator) is single-use. If you need to iterate multiple times, either keep the original iterable (e.g., a list) around, or call the generator **function** again to get a fresh generator object.

### `StopIteration` Inside a Generator (PEP 479)
Before PEP 479, an unhandled `StopIteration` raised *inside* a generator body would silently propagate and prematurely terminate the enclosing `for` loop — a confusing, hard-to-debug behavior. **PEP 479** changed this: an unhandled `StopIteration` escaping a generator's body is now converted into a `RuntimeError`, forcing the bug to surface loudly instead of silently truncating iteration. It was opt-in in Python 3.5 and 3.6 (via `from __future__ import generator_stop`) and became the default in 3.7.

```python
def broken_gen():
    yield 1
    raise StopIteration   # don't do this — raises RuntimeError instead, by design (PEP 479)
    yield 2
```

### Forgetting to "Prime" a Generator Before `.send()`
Calling `.send(value)` on a generator that hasn't been advanced to its first `yield` yet raises `TypeError: can't send non-None value to a just-started generator`. Always call `next(gen)` (or `gen.send(None)`, equivalent) once before the first real `.send()`.

### Checking "Is This an Iterator?" Correctly
```python
from collections.abc import Iterator, Iterable

x = [1, 2, 3]
print(isinstance(x, Iterable))   # True — has __iter__
print(isinstance(x, Iterator))    # False — no __next__

it = iter(x)
print(isinstance(it, Iterator))   # True
```
Using `collections.abc.Iterable`/`Iterator` for `isinstance` checks is more robust and idiomatic than manually checking for `__iter__`/`__next__` with `hasattr`.

## Notes & Gotchas

- **"Every iterator is iterable, not every iterable is an iterator"** — state this precisely; it's the single most-tested definitional question on this topic.
- **`StopIteration` is the normal termination signal for the protocol**, not an error to catch defensively everywhere — `for` loops handle it automatically.
- **Generators trade eagerness for memory** — the standard justification for choosing them, and the reason infinite sequences (`itertools.count`) are even possible.
- **Generators are single-use** — a very common "why did my second loop print nothing" bug.
- **`yield` preserves the entire function's local state via its frame object**, unlike `return`, which discards it — ties directly into how CPython frames work (see `17-Internals.md`).
- **PEP 479 turns a `StopIteration` raised inside a generator into a `RuntimeError`** — know this if asked why manually raising `StopIteration` inside a generator body is discouraged/broken in modern Python.
- **`yield from` forwards `send()`/`throw()`/`close()` automatically** — reimplementing that delegation manually with a plain loop is significantly more code and easy to get subtly wrong.
- **A generator expression inside a single function call doesn't need extra parentheses**: `sum(x*x for x in range(10))` is valid — the call's own parentheses double as the generator expression's parentheses.
- **`range`, dict views and containers are iterable but not iterators; file objects, generators, `map`, `filter` and `zip` are iterators** — only the latter are exhausted after one pass.
- **A class whose `__iter__` is a generator method is re-iterable**; one whose `__iter__` returns `self` is a one-shot iterator.
- **`iter(callable, sentinel)` doesn't need `partial`**: `iter(f.readline, '')` is enough, because `f.readline` is already a callable.
- **A generator expression evaluates its outermost iterable immediately** but everything else lazily.
- **`gen.close()` runs `finally` blocks and marks the generator closed; yielding inside `except GeneratorExit` raises `RuntimeError`.**
- **`async def` containing `yield` is an *async* generator** (iterated with `async for`), a separate protocol covered in `20-Async-Programming.md`.

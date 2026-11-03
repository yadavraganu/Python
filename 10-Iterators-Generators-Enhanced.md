## 1. Iterables vs. Iterators — The Core Distinction

These terms are often used interchangeably in casual conversation, but they describe different protocols. This is a common opening interview question.

- **Iterable:** an object from which an iterator can be obtained by calling `iter(obj)`. Most iterables implement `__iter__`. Examples include `list`, `tuple`, `str`, `dict`, `set`, and `range`.
- **Iterator:** an object that produces values one at a time and maintains iteration state. It implements both `__iter__` (returning itself) and `__next__` (returning the next value or raising `StopIteration`).
- **Generator:** a convenient kind of iterator created by a generator function or generator expression. Generators are covered in Section 2.

```python
nums = [1, 2, 3]          # nums is iterable
it = iter(nums)           # it is an iterator

print(next(it))           # 1
print(next(it))           # 2
print(next(it))           # 3
print(next(it))           # raises StopIteration
```

**Key interview statement:** every iterator is iterable, but not every iterable is an iterator. A list is iterable but does not implement `__next__`, so `next(nums)` raises `TypeError`. Calling `iter(nums)` creates an iterator; calling `iter(it)` on an iterator returns that same iterator.

This distinction explains why a list can be traversed repeatedly while an exhausted iterator cannot be restarted.

### The iterator protocol

```python
class Countdown:
    """A one-shot iterator that counts down to zero."""

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

for number in Countdown(3):
    print(number)         # 3, 2, 1
```

A `for` loop obtains an iterator once, then repeatedly requests the next item until `StopIteration` is raised. `list()`, `sum()`, unpacking, and many other iteration constructs use the iteration protocol too. `StopIteration` is the normal completion signal, not an application error in this context.

### `iter()` with a sentinel — the two-argument form

`iter(callable, sentinel)` creates an iterator that calls the function with no arguments until the returned value compares equal to the sentinel. The sentinel value itself is not yielded.

```python
with open("file.txt", encoding="utf-8") as file:
    for line in iter(file.readline, ""):
        print(line, end="")

# Read a binary file in fixed-size blocks.
with open("data.bin", "rb") as file:
    sizes = [len(block) for block in iter(lambda: file.read(4096), b"")]

print(sizes)  # e.g. [4096, 4096, 1808] for a 10,000-byte file
```

This is useful when an API exposes a callable that returns a sentinel at end-of-input.

### Which objects are iterable vs. iterator?

| Iterator (has `__next__`; stateful, normally one-pass) | Iterable but not an iterator (can usually create fresh iterators) |
|---|---|
| Generator objects and generator expressions | `list`, `tuple`, `str`, `dict`, `set` |
| File objects (`iter(file_obj) is file_obj`) | `range` (`next(range(3))` raises `TypeError`) |
| `map`, `filter`, `zip`, `enumerate` | `dict.keys()`, `.values()`, `.items()` views |
| Many `itertools` objects | Many user-defined container classes |

“Can usually create fresh iterators” is deliberate: an iterable can wrap a one-shot source, so repeatability depends on its implementation and underlying data.

### Making your own re-iterable class

A convenient way to make a container re-iterable is to implement `__iter__` as a generator method. Each call to `iter(container)` creates a fresh generator.

```python
class Bag:
    def __init__(self, items):
        self.items = items

    def __iter__(self):
        for item in self.items:
            yield item

bag = Bag([1, 2])
print(list(bag), list(bag))  # [1, 2] [1, 2]
```

This is repeatable here because `self.items` is a list. If `self.items` were itself an already-created iterator, the second traversal would still be exhausted.

Contrast `Bag` with `Countdown`, whose `__iter__` returns `self`: `Countdown` is an iterator and is one-shot. Python also supports the **old sequence protocol** as a fallback: an object with `__getitem__` accepting indices `0, 1, 2, ...` and raising `IndexError` at the end can be iterated even if it does not define `__iter__`.

## 2. Generators — Functions That Yield

A **generator function** is a function whose body contains `yield`. Calling it returns a generator object; the function body does not start running until the generator is advanced. A generator object is an iterator that runs lazily, suspends at each `yield`, and resumes from that point on the next request.

```python
def count_up_to(n):
    i = 1
    while i <= n:
        yield i
        i += 1

gen = count_up_to(3)
print(type(gen))           # <class 'generator'>
print(next(gen))           # 1
print(next(gen))           # 2
print(next(gen))           # 3
print(next(gen))           # raises StopIteration
```

### `yield` vs. `return`

- `return` exits a function. In a generator, `return` ends iteration; `return value` places that value on the terminating `StopIteration` exception (and can be captured by `yield from`).
- `yield` suspends a generator, preserving its local variables and execution position so it can resume later.

A generator's suspended execution state is represented by its runtime frame/state. The exact internal implementation is interpreter-specific; the important language-level behavior is that local state is preserved across a suspension.

### Inspecting a generator's state

```python
import inspect

def one_value():
    yield 1

g = one_value()
print(inspect.getgeneratorstate(g))  # GEN_CREATED

next(g)
print(inspect.getgeneratorstate(g))  # GEN_SUSPENDED

try:
    next(g)
except StopIteration:
    pass

print(inspect.getgeneratorstate(g))  # GEN_CLOSED
```

The standard states are `GEN_CREATED`, `GEN_RUNNING`, `GEN_SUSPENDED`, and `GEN_CLOSED`.

### Why generators matter: lazy evaluation and memory

A generator produces values on demand rather than building the entire output collection up front.

```python
def squares_list(n):
    return [i * i for i in range(n)]  # creates the complete list


def squares_gen(n):
    for i in range(n):
        yield i * i                   # creates each value on demand

# The sum consumes values incrementally; it does not retain the full output list.
total = sum(squares_gen(10_000_000))
```

For this particular `squares_gen` implementation, additional memory is effectively constant with respect to `n`, excluding the input range object, the current value, and the consumer's own state. Do **not** conclude that every generator uses constant memory:

```python
def generator_with_cache(n):
    cache = []
    for i in range(n):
        cache.append(i)   # retained state grows with n
        yield i
```

A more accurate interview explanation is:

> Generators avoid eagerly materializing the complete output. Their memory use depends on the state they retain, the source iterator, buffering in upstream stages, and what the consumer stores.

A generator object is often small, but its referenced objects can be large. Exact object sizes also vary by Python implementation and version, so avoid memorizing a fixed byte count as a general rule.

Generators can represent unbounded sequences because they need not store every future value. The consumer must still stop an infinite iterator deliberately.

### Generator expressions vs. list comprehensions

Both use a compact comprehension syntax, but their evaluation strategies differ.

```python
list_comp = [x * x for x in range(5)]  # eager: creates a list now
gen_expr = (x * x for x in range(5))  # lazy: creates a generator now

print(list_comp)       # [0, 1, 4, 9, 16]
print(gen_expr)        # <generator object ...>
print(list(gen_expr))  # [0, 1, 4, 9, 16]
```

**When to choose which:**

- Use a **generator expression** when you want incremental, usually one-pass consumption and do not need indexing, repeated traversal, or the complete length.
- Use a **list comprehension** when you need the results more than once, random access, `len()`, or a materialized snapshot—and the memory cost is acceptable.

Generator expressions work particularly well with consumers such as `sum()`, `any()`, and `all()`. These can consume incrementally; `any()` and `all()` can also stop as soon as the result is known. By contrast, `str.join()` must construct the complete output string, so it should not be presented as a way to avoid materializing the final result.

### Generator expressions: the outermost iterable is evaluated immediately

The expression for the leftmost `for` clause's iterable is evaluated when the generator expression is created. The rest of the work (filters, later clauses, and element expressions) happens as values are requested.

```python
try:
    g = (x for x in undefined_name)
except NameError:
    print("outermost iterable evaluated at creation")

y = 0
g = (x for x in range(3) if y == 0)
y = 1
print(list(g))  # []: the condition sees y when values are requested
```

### Generator pipelines

Chaining generators lets each stage pull one item from the previous stage as needed.

```python
def read_lines(path):
    with open(path, encoding="utf-8") as file:
        for line in file:
            yield line.rstrip("\n")


def non_empty(lines):
    return (line for line in lines if line)


def shout(lines):
    return (line.upper() for line in lines)

for line in shout(non_empty(read_lines("file.txt"))):
    print(line)
```

Nothing is read until the final loop requests an item. The pipeline is lazy, but it is still synchronous and runs in one process; generators do not automatically provide parallelism, retries, checkpointing, persistence, or distributed execution.

## 3. Two-Way Communication: `send()`, `throw()`, `close()`

Generators are not only value producers. A caller can send a value into a suspended generator, inject an exception at its suspension point, or close it early.

### `send()` — pass a value into the generator

```python
def echo():
    while True:
        received = yield
        print(f"Received: {received}")

gen = echo()
next(gen)             # prime: advance to the first yield

gen.send("hello")    # Received: hello
gen.send("world")    # Received: world
```

When `yield` is used as an expression (`received = yield`), the expression evaluates to the value passed by `.send(value)`. A newly created generator must first be advanced with `next(gen)` or `gen.send(None)`; sending a non-`None` value before the first suspension raises `TypeError`.

### `throw()` — raise an exception inside the generator

```python
def resilient_gen():
    try:
        while True:
            yield "running"
    except ValueError:
        yield "caught the ValueError, recovering"

gen = resilient_gen()
print(next(gen))                 # running
print(gen.throw(ValueError))     # caught the ValueError, recovering
```

`throw()` raises the supplied exception at the generator's current suspension point. The generator may handle it, yield another value, or let it propagate to the caller.

### `close()` — terminate a generator early

```python
def cleanup_gen():
    try:
        yield 1
    finally:
        print("cleanup ran")

g = cleanup_gen()
next(g)
g.close()                       # prints: cleanup ran
```

`close()` injects `GeneratorExit` at the suspension point. A `finally` block can release resources. The generator should not yield another value while handling `GeneratorExit`; doing so raises `RuntimeError: generator ignored GeneratorExit`. It should allow `GeneratorExit` to propagate or return.

When a generator is finalized, Python attempts to close it, but the timing of finalization is not a portable resource-management strategy. Use explicit cleanup or a context manager when deterministic cleanup matters; see Section 7.

## 4. `yield from` — Delegating to a Sub-Generator

`yield from` delegates iteration to another iterable. With a generator delegate, it also forwards generator operations such as `send()` and `throw()`, and handles close propagation. It is more complete than simply writing a `for` loop around the delegate.

```python
def inner():
    yield 1
    yield 2
    return "inner done"


def outer():
    result = yield from inner()  # yields 1 and 2; receives the subgenerator's return value
    print(f"inner returned: {result}")
    yield 3

for value in outer():
    print(value)
# 1
# 2
# inner returned: inner done
# 3
```

The subgenerator's `return` value becomes the value of the `yield from` expression. This is useful when composing generators.

### Flattening nested lists

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item

print(list(flatten([1, [2, 3, [4, 5]], 6])))
# [1, 2, 3, 4, 5, 6]
```

This example intentionally treats only `list` objects as nested containers. A production implementation should define which container types count as nested and how it should treat strings, mappings, and cyclic structures.

## 5. The `itertools` Module — Iterator Building Blocks

`itertools` provides efficient tools for constructing and combining iterators without requiring a complete output list. Many functions return iterators, but their memory behavior is not identical.

```python
import itertools

# Infinite or repeatable iterators
itertools.count(10)                    # 10, 11, 12, ...
itertools.cycle([1, 2, 3])              # 1, 2, 3, 1, 2, 3, ...
itertools.repeat("x", times=3)         # x, x, x

# Combinatoric iterators (list() materializes the results here)
list(itertools.combinations([1, 2, 3], 2))
# [(1, 2), (1, 3), (2, 3)]

list(itertools.permutations([1, 2], 2))
# [(1, 2), (2, 1)]

list(itertools.product([0, 1], repeat=2))
# [(0, 0), (0, 1), (1, 0), (1, 1)]

# Everyday tools
list(itertools.chain([1, 2], [3, 4]))
# [1, 2, 3, 4]

list(itertools.islice(itertools.count(), 5))
# [0, 1, 2, 3, 4]
```

**Important memory nuance:**

- `itertools.count()` generates values incrementally and does not cache all previous values.
- `itertools.cycle(iterable)` saves the elements it has seen so it can replay them. Its cache grows with the size of the input. Cycling an unbounded source can therefore consume unbounded memory.
- `itertools.repeat(value)` repeats a reference to the value; it does not create a new copy of the object on each iteration.
- Wrapping an iterator in `list()` eagerly materializes its output and can remove the memory benefit of lazy iteration.

### More useful tools

```python
import itertools

list(itertools.pairwise([1, 2, 3]))       # [(1, 2), (2, 3)] — Python 3.10+
list(itertools.batched(range(5), 2))      # [(0, 1), (2, 3), (4,)] — Python 3.12+

left, right = itertools.tee([1, 2, 3])
print(list(left), list(right))            # [1, 2, 3] [1, 2, 3]
```

`tee()` creates independent iterators from one input iterator. It buffers values when one branch gets ahead of another. The buffer can become large if the branches advance at very different speeds. After splitting, do not also consume the original source elsewhere. `tee()` iterators are not intended for concurrent use by multiple threads.

`chain.from_iterable()` is particularly useful when flattening one level of batches:

```python
batches = [[1, 2], [3, 4], [5]]
print(list(itertools.chain.from_iterable(batches)))  # [1, 2, 3, 4, 5]
```

## 6. Common Pitfalls & Gotchas

### Iterators and generators are exhausted after one full pass

```python
gen = (x for x in range(3))
print(list(gen))  # [0, 1, 2]
print(list(gen))  # [] — already exhausted
```

An iterator is not restarted by calling `iter()` on it. Keep a re-iterable source (such as a list) or call the generator function again to obtain a fresh generator.

### `StopIteration` inside a generator — PEP 479

An unhandled `StopIteration` escaping from a generator body is converted into `RuntimeError`. This prevents a bug from silently looking like normal generator completion. PEP 479 became the default in Python 3.7; before that it could be enabled with `from __future__ import generator_stop` in the relevant Python versions.

```python
def broken_gen():
    yield 1
    raise StopIteration  # becomes RuntimeError when it escapes the generator body
```

Use `return` to end a generator. Do not manually raise `StopIteration` to signal its completion from inside the generator body.

### Forgetting to prime before `.send()`

Calling `gen.send("value")` before the generator reaches its first `yield` raises `TypeError`. Advance it with `next(gen)` or `gen.send(None)` first.

### Checking whether an object is an iterator

```python
from collections.abc import Iterable, Iterator

values = [1, 2, 3]
print(isinstance(values, Iterable))  # True
print(isinstance(values, Iterator))  # False

it = iter(values)
print(isinstance(it, Iterator))      # True
```

`collections.abc` checks are generally preferable to manually checking for `__iter__` and `__next__` with `hasattr()`. For custom classes, note that structural behavior and ABC registration can differ; the most reliable practical test is whether `iter(obj)` and `next(iterator)` behave as expected.

### Accidentally consuming an iterator

Functions such as `sum()`, `max()`, `any()`, `all()`, `list()`, and `tuple()` consume iterators. If the same iterator is passed to one of these and then reused, it may already be exhausted.

```python
numbers = (n for n in range(5))
print(sum(numbers))   # 10; consumes the generator
print(list(numbers))  # []
```

### Assuming a generator makes the whole pipeline memory efficient

A generator only controls how its own values are produced. An upstream API may buffer all records, a downstream consumer may append everything to a list, or a stage may retain a cache. Check every stage, not only the stage that contains `yield`.

## 7. Generators in Data Engineering

Generators are useful in application-side data processing when records can be read, validated, transformed, and written incrementally. They are not a replacement for distributed processing frameworks or durable streaming systems.

### A. Stream a large CSV file

```python
import csv


def read_csv_rows(path):
    """Yield one parsed row at a time; the generator owns the file handle."""
    with open(path, newline="", encoding="utf-8") as file:
        yield from csv.DictReader(file)


for row in read_csv_rows("orders.csv"):
    process(row)
```

The file is opened when iteration starts, not when `read_csv_rows()` is called. It closes when the generator finishes normally or is explicitly closed. If a consumer stops early while keeping the generator alive, the file may remain open. Use deterministic cleanup when early termination is possible:

```python
from contextlib import closing

with closing(read_csv_rows("orders.csv")) as rows:
    for row in rows:
        process(row)
        if should_stop(row):
            break
# closing() calls rows.close(), which exits the generator's with block.
```

An alternative ownership model is to open the file in the caller and pass the file object into a parsing function. The caller then controls the file's lifetime directly.

### B. Build a lazy ETL pipeline

```python
from decimal import Decimal


def extract(records):
    for record in records:
        yield record


def validate(records):
    for record in records:
        if record.get("order_id") and record.get("amount") is not None:
            yield record


def transform(records):
    for record in records:
        yield {
            "order_id": record["order_id"],
            "amount": Decimal(record["amount"]),
        }


def load(records):
    for record in records:
        write_record(record)  # replace with a real sink / database operation


source = [
    {"order_id": "A-1", "amount": "100.50"},
    {"order_id": None, "amount": "25.00"},
    {"order_id": "A-3", "amount": "75.25"},
]

pipeline = transform(validate(extract(source)))
load(pipeline)
```

Each stage processes an item only when the next stage asks for it. This pattern limits intermediate materialization, but the example's `source` is already a list. A real large-file pipeline should feed the pipeline from a streaming source such as `read_csv_rows()`.

In production, also define what happens to malformed records: fail the task, send records to a quarantine/dead-letter destination, or record a structured validation error. Avoid silently dropping bad data unless that is an explicit business rule.

### C. Process records in batches

A generator can produce records incrementally while a sink writes batches. Batching limits per-call overhead and can match database or API bulk-write interfaces.

**Python 3.12+:**

```python
from itertools import batched

for batch in batched(read_csv_rows("orders.csv"), 1_000):
    write_batch(batch)
```

**Compatible alternative for earlier Python versions:**

```python
from itertools import islice


def batches(iterable, batch_size):
    if batch_size <= 0:
        raise ValueError("batch_size must be greater than zero")

    iterator = iter(iterable)
    while batch := list(islice(iterator, batch_size)):
        yield batch


for batch in batches(read_csv_rows("orders.csv"), 1_000):
    write_batch(batch)
```

The batch itself is materialized as a list, so memory is proportional to the batch size plus the state of the source and processing stages. Choose the batch size based on record size, sink limits, latency, and available memory.

### D. Iterate over database results

Many database drivers expose cursor iteration. Some drivers or cursor configurations buffer results client-side, while others support server-side or streaming cursors. Do not assume that wrapping a query in a generator guarantees that the database driver is fetching incrementally.

A `fetchmany()` wrapper makes the application-side batching explicit:

```python
def iter_cursor_rows(cursor, batch_size=1_000):
    if batch_size <= 0:
        raise ValueError("batch_size must be greater than zero")

    while rows := cursor.fetchmany(batch_size):
        yield from rows


# Connection, transaction, cursor type, and cursor lifetime are driver-specific.
# Keep the cursor/connection open for the full duration of iteration.
for row in iter_cursor_rows(cursor, batch_size=1_000):
    process(row)
```

In an actual application, use the database driver's documented cursor and connection context managers, and confirm whether server-side cursors, streaming results, or fetch-size settings are required.

### E. What generators do not provide automatically

Generators are a language-level mechanism for incremental iteration. They do **not** automatically provide:

- Parallelism or distributed execution.
- Durable storage, offsets, checkpoints, or exactly-once processing.
- Retries, dead-letter handling, or transactional writes.
- Backpressure across arbitrary APIs or queues.
- A guarantee that the upstream library or database driver does not buffer data.

For a Senior Data Engineer interview, distinguish **in-process lazy iteration** from **distributed data processing** and **event-streaming platforms**. Explain which layer is responsible for buffering, concurrency, recovery, delivery guarantees, and state management.

## 8. Async Generators — A Separate Iteration Protocol

An **asynchronous generator function** is an `async def` function that contains `yield`. It produces an asynchronous iterator, consumed with `async for` rather than a normal `for` loop.

```python
import asyncio


async def fetch_records():
    for record_id in range(3):
        await asyncio.sleep(0)  # represents an asynchronous operation
        yield {"id": record_id}


async def main():
    async for record in fetch_records():
        print(record)


asyncio.run(main())
```

Key differences:

| Synchronous generator | Asynchronous generator |
|---|---|
| `def` containing `yield` | `async def` containing `yield` |
| Implements the synchronous iterator protocol | Implements the asynchronous iterator protocol |
| Consumed with `for` | Consumed with `async for` |
| Advanced with `next()` | Advanced with `anext()` / asynchronous iteration |
| Useful for synchronous incremental processing | Useful when producing values around `await`-based I/O |

An async generator is not simply a faster generator. It supports cooperative asynchronous execution when used with an event loop and async-aware operations. It does not make CPU-bound work parallel by itself.

For early termination, asynchronous generators also support asynchronous cleanup through `aclose()`; use the async context-management patterns documented by the libraries involved when deterministic cleanup is required.

## 9. Senior Data Engineer Interview Questions

Use these as practice prompts. For each answer, explain the concept, show a short example, and describe a real trade-off.

| # | Question | What a strong answer should cover |
|---|---|---|
| 1 | Iterable vs. iterator vs. generator? | Protocols, `__iter__`, `__next__`, generator as a specialized iterator |
| 2 | What happens when a `for` loop runs? | `iter()`, repeated `next()`, `StopIteration` |
| 3 | How does `yield` differ from `return`? | Suspension, preserved local state, generator termination |
| 4 | Why can generators reduce memory usage? | Lazy production and the importance of retained state and buffering |
| 5 | Can a generator be reused? | Exhaustion and creating a new generator object |
| 6 | Generator expression vs. list comprehension? | Lazy vs. eager evaluation, access patterns, memory trade-offs |
| 7 | What is PEP 479? | Conversion of escaping `StopIteration` into `RuntimeError` |
| 8 | Explain `send()`, `throw()`, and `close()`. | Two-way communication, exception injection, cleanup |
| 9 | What does `yield from` do? | Delegation, forwarding, subgenerator return value |
| 10 | How would you process a 10 GB CSV file? | Streaming reads, incremental transforms, bounded batches, cleanup, malformed-row policy |
| 11 | How would you build a lazy ETL pipeline? | Composable stages, source/sink boundaries, observability, error handling |
| 12 | Do generators guarantee low memory use with a database cursor? | Driver buffering, cursor configuration, fetch size, connection lifetime |
| 13 | How would you add batch writes? | Bounded batches, batch-size trade-offs, retries and idempotency at the sink |
| 14 | What happens if a consumer stops early? | Generator lifetime, `close()`, `finally`, context managers |
| 15 | Generator vs. async generator? | Sync vs. async protocol, event-loop I/O, `for` vs. `async for` |
| 16 | Are generators a replacement for Spark, Kafka, or distributed processing? | Scope limits: process-local iteration vs. distributed execution or durable streaming |

### Practice problems

1. Implement a custom iterator that reads a sequence in fixed-size chunks.
2. Write a generator that yields only valid JSON records from a line-delimited file and reports malformed lines.
3. Build an ETL pipeline with extract, validate, transform, batch, and load stages.
4. Merge two already-sorted input iterators without converting either input into a list.
5. Explain and demonstrate how an early `break` affects a generator that owns a file handle.
6. Compare memory behavior for a generator pipeline, a list-based pipeline, and a pipeline whose `tee()` branches advance at different speeds.

## 10. Quick Revision — Notes & Gotchas

- **Every iterator is iterable, but not every iterable is an iterator.** Be able to demonstrate this with a list and `iter(list)`.
- **`StopIteration` is the normal end-of-iteration signal.** A `for` loop handles it for you.
- **Generators defer production; they do not guarantee constant memory.** Retained state, caches, source buffering, and consumer behavior all matter.
- **Generators and iterators are normally one-pass.** Create a new generator by calling the generator function again when another pass is needed.
- **A generator function call returns a generator object without executing the body.** The body starts when the generator is advanced.
- **`yield` suspends and preserves execution state; `return` ends the generator.** Use `return`, not `raise StopIteration`, to finish a generator.
- **PEP 479 converts an escaping `StopIteration` from a generator body into `RuntimeError`.**
- **`yield from` delegates iteration and generator operations.** It also captures a subgenerator's return value.
- **`send()` requires priming before sending a non-`None` value.**
- **`close()` injects `GeneratorExit`.** Use `finally` for cleanup and do not yield while handling `GeneratorExit`.
- **`iter(callable, sentinel)` stops before yielding the sentinel.**
- **A generator expression evaluates its outermost iterable immediately; the remaining work is lazy.**
- **`itertools.cycle()` caches its input.** Do not assume all infinite iterators have constant memory use.
- **`itertools.tee()` buffers for lagging branches.** Large divergence can mean large memory usage.
- **A generator-based `__iter__` can make a container re-iterable only if its underlying source is itself re-iterable.**
- **Resource lifetime matters.** Close generators or use context managers when iteration may stop before exhaustion.
- **Generators do not add distributed execution, retries, checkpoints, or delivery guarantees.** Those are separate system-design concerns.
- **`async def` containing `yield` defines an async generator.** Consume it using `async for` and use async cleanup where required.

## Further reading

- [Python documentation: Iterator Types](https://docs.python.org/3/library/stdtypes.html#iterator-types)
- [Python documentation: Generator Types](https://docs.python.org/3/reference/datamodel.html#the-standard-type-hierarchy)
- [Python documentation: Expressions — Generator expressions](https://docs.python.org/3/reference/expressions.html#generator-expressions)
- [Python documentation: `itertools`](https://docs.python.org/3/library/itertools.html)
- [Python documentation: `contextlib`](https://docs.python.org/3/library/contextlib.html)
- [PEP 479 — Change StopIteration handling inside generators](https://peps.python.org/pep-0479/)
- [Python documentation: Asynchronous generator functions](https://docs.python.org/3/reference/expressions.html#asynchronous-generator-functions)

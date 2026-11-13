# Python Standard-Library Toolkit

A practical guide to standard-library tools that repeatedly appear in coding rounds, code reviews, and senior Python interviews. The goal is not to memorize every module; it is to recognize the right tool and understand its complexity and semantics.

> **Version baseline:** examples use Python 3.10+ unless a later version is called out. Standard-library APIs can evolve, so check the version deployed by your application.

## 1. `collections`: specialized containers

The dedicated [collection guide](06-Choose-Collection.md) covers the common container choices in depth, including `Counter`, `defaultdict`, `deque`, `OrderedDict`, and `namedtuple`. Use this quick map when choosing a standard-library tool:

| Tool | Use it for | Important caveat |
|---|---|---|
| `Counter` | Frequency maps and multiset arithmetic | `c1 - c2` drops zero/negative results; `subtract()` preserves them |
| `defaultdict` | Grouping or accumulating under missing keys | `d[key]` invokes the factory and inserts; `get()` does not |
| `deque` | Fast appends/pops at both ends | Not a blocking producer-consumer queue; use `queue.Queue` for that |
| `ChainMap` | Layered configuration lookup | A live view; writes affect only the first mapping |
| `namedtuple` | Compact immutable record-like tuple | Use a dataclass when validation, defaults, mutability, or richer methods are needed |

### `ChainMap` — layered configuration lookup

```python
from collections import ChainMap

defaults = {"timeout": 10, "retries": 2}
environment = {"timeout": 20}
command_line = {"retries": 5}

config = ChainMap(command_line, environment, defaults)
assert config["timeout"] == 20
assert config["retries"] == 5
```

Lookup searches mappings from first to last. Writes go only to the first mapping. It is a view, not a merged copy.

## 2. `itertools`: memory-efficient iteration

`itertools` functions generally consume iterables lazily. They are useful for streaming data and avoiding intermediate lists.

```python
from itertools import chain, islice, product

assert list(chain([1, 2], [3, 4])) == [1, 2, 3, 4]
assert list(islice(range(1_000_000), 3, 6)) == [3, 4, 5]
assert list(product("AB", [1, 2])) == [
    ("A", 1), ("A", 2), ("B", 1), ("B", 2)
]
```

Frequently used tools:

| Tool | Behavior | Typical use |
|---|---|---|
| `chain(a, b, ...)` | Concatenate iterables lazily | Stream multiple sources |
| `islice(iterable, start, stop, step)` | Slice an iterator | Read a window without materializing |
| `pairwise(iterable)` | Adjacent pairs | Compare neighboring values |
| `batched(iterable, n)` | Fixed-size batches | Chunk an input stream (Python 3.12+) |
| `product(a, b)` | Cartesian product | Generate combinations |
| `combinations(items, r)` | Unordered selections | Combination search |
| `permutations(items, r)` | Ordered selections | Permutation search |
| `groupby(items, key=...)` | Consecutive equal-key groups | Run-length grouping |
| `accumulate(items, func=...)` | Running reduction | Prefix sums/products |
| `zip_longest(a, b, fillvalue=...)` | Zip to longest input | Unequal streams |

### `groupby` does not globally group

```python
from itertools import groupby

rows = [("a", 1), ("b", 2), ("a", 3)]
groups = [(key, list(group)) for key, group in groupby(rows, key=lambda row: row[0])]

assert [key for key, _ in groups] == ["a", "b", "a"]
```

Sort by the same key first when you need all equal keys in one group. Each group iterator shares the underlying source; materialize a group before advancing if it must be retained.

### `tee` caveat

`itertools.tee(iterable, n)` creates independent logical iterators, but it buffers values when consumers advance at different rates. Large gaps between consumers can require substantial memory. It also should not be used to duplicate a single iterator across threads.

## 3. `functools`: function composition and caching

### `lru_cache` and `cache`

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

assert fibonacci(10) == 55
print(fibonacci.cache_info())
```

- Arguments must be hashable.
- Cached results are retained strongly until eviction or `cache_clear()`.
- Cache only functions whose result is determined by their arguments. Avoid caching time-dependent, random, user-specific, or side-effecting behavior.
- A cache is not a substitute for bounding memory or handling invalidation.
- `functools.cache` is an unbounded cache; `lru_cache(maxsize=None)` is equivalent.

### `partial`, `reduce`, `wraps`, and `singledispatch`

```python
from functools import partial, reduce, singledispatch, wraps

def multiply(a: int, b: int) -> int:
    return a * b

double = partial(multiply, 2)
assert double(5) == 10

assert reduce(lambda total, value: total + value, [1, 2, 3], 0) == 6

@singledispatch
def render(value):
    return str(value)

@render.register
def _(value: list):
    return ", ".join(map(str, value))

assert render([1, 2]) == "1, 2"
```

Prefer the built-in `sum()` for simple numeric sums; `reduce()` is useful when the reduction operation is not naturally expressed by a clearer built-in.

Use `@wraps(func)` in decorators so the wrapper preserves metadata such as `__name__`, `__doc__`, and `__wrapped__`.

## 4. `operator`: avoid unnecessary lambdas

```python
from operator import itemgetter, attrgetter

rows = [{"name": "Mina", "score": 91}, {"name": "Arun", "score": 84}]
by_score = sorted(rows, key=itemgetter("score"))
assert by_score[0]["name"] == "Arun"

class Job:
    def __init__(self, priority):
        self.priority = priority

jobs = [Job(3), Job(1), Job(2)]
assert [job.priority for job in sorted(jobs, key=attrgetter("priority"))] == [1, 2, 3]
```

`itemgetter`, `attrgetter`, and `methodcaller` express common operations and can be clearer than repeated anonymous lambdas.

## 5. Dates and times: `datetime`, `zoneinfo`, and UTC

A timestamp represents an instant. A local wall-clock time is a civil-time representation that depends on a timezone and daylight-saving rules. Do not treat them as interchangeable.

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

instant = datetime.now(timezone.utc)
india_time = instant.astimezone(ZoneInfo("Asia/Kolkata"))
assert india_time.tzinfo is not None
```

Rules for reliable application code:

- Store and exchange instants in UTC, typically as timezone-aware datetimes.
- Convert to a named IANA timezone for display or business rules.
- Prefer `zoneinfo.ZoneInfo("Area/City")` over fixed offsets when daylight-saving transitions matter.
- Do not use naive `datetime` objects to represent globally meaningful instants.
- Be explicit about ambiguous/nonexistent local times around daylight-saving transitions.
- Use `date` for calendar dates and `timedelta` for durations.

`datetime.utcnow()` returns a naive datetime and is discouraged for new code; use `datetime.now(timezone.utc)`.

## 6. Exact numeric domains: `decimal` and `fractions`

Binary floating-point cannot represent many decimal fractions exactly.

```python
from decimal import Decimal, ROUND_HALF_UP

price = Decimal("19.99")
tax = Decimal("0.075")
total = (price * (1 + tax)).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
assert total == Decimal("21.49")
```

Construct `Decimal` from strings or integers when decimal semantics matter. `Decimal(0.1)` captures the binary float approximation, not the exact decimal 0.1.

Use `fractions.Fraction` for exact rational arithmetic:

```python
from fractions import Fraction

assert Fraction(1, 3) + Fraction(1, 6) == Fraction(1, 2)
```

## 7. `contextvars`: context-local state

`contextvars` provides context-local values that work with asynchronous tasks, unlike thread-local storage which is scoped to an OS thread.

```python
from contextvars import ContextVar

request_id = ContextVar("request_id", default=None)

token = request_id.set("req-123")
try:
    assert request_id.get() == "req-123"
finally:
    request_id.reset(token)
```

Use it for correlation IDs or request context when explicit parameter passing would be excessively invasive. Do not use it as a replacement for clear dependencies or pass arbitrary mutable request state through a global context.

## 8. Other high-value modules

| Module | What it solves | Interview/use-case note |
|---|---|---|
| `pathlib` | Filesystem paths | Prefer `Path` over manual path concatenation |
| `argparse` | CLI arguments | Standard CLI parser; validates flags and types |
| `logging` | Application diagnostics | Use the logging guide in this repository |
| `inspect` | Runtime introspection | Signatures, annotations, stack information; source may be unavailable |
| `traceback` | Format/report exceptions | Preserve exception context; avoid swallowing tracebacks |
| `statistics` | Basic statistics | Mean, median, quantiles; not a substitute for NumPy |
| `heapq` | Min-heap | Priority queues and top-k; see the collections guide |
| `bisect` | Binary search/insertion | Search O(log n), list insertion still O(n) |
| `enum` | Named constant sets | Prefer `Enum`/`StrEnum` over magic strings |
| `secrets` | Cryptographically secure tokens | Use for passwords/reset tokens, not `random` |
| `hashlib` | Cryptographic digests | Hashing is not encryption or password hashing |
| `hmac` | Message authentication | Use `compare_digest` for MAC/token comparisons |
| `subprocess` | External processes | Use argument lists and avoid `shell=True` for untrusted input |
| `concurrent.futures` | Thread/process pools | Higher-level task execution API |

## 9. Common interview questions

**When would you choose `Counter` over a dictionary?**  
When the problem is fundamentally a multiset/frequency count; `Counter` supplies `most_common`, arithmetic, and zero for missing counts. A normal dict is better when values are not counts or custom invariants are needed.

**Why is `itertools.groupby` often misunderstood?**  
It groups consecutive runs, not all equal keys globally. Sort by the same key first when global grouping is required.

**When is `lru_cache` unsafe or counterproductive?**  
When function outputs depend on hidden state, arguments are mutable/unhashable, cached values become stale, or result retention grows memory unexpectedly.

**Why use `Decimal` for currency?**  
It models base-10 decimal arithmetic and explicit rounding. It does not automatically define a currency's rounding/business rules.

**Why use `contextvars` instead of `threading.local` in async code?**  
Multiple async tasks can interleave on the same OS thread. Context variables are propagated per execution context rather than shared only by thread.

## 10. Practical checklist

- [ ] Know `Counter`, `defaultdict`, `deque`, and `ChainMap`.
- [ ] Understand lazy iteration and the `groupby`/`tee` pitfalls.
- [ ] Use `lru_cache` only for pure, bounded, cacheable functions.
- [ ] Use aware UTC datetimes and named timezones for real instants.
- [ ] Choose `Decimal` only when decimal arithmetic is a domain requirement.
- [ ] Know `secrets`, `hashlib`, `hmac`, `subprocess`, `argparse`, and `pathlib`.

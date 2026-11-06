## Core Built-In Types

| Type         | Mutability | Order | Duplicates | Lookup/Indexing | Insertion/Append  | Memory Footprint | Use Case Highlights |
|--------------|------------|-------|------------|-----------------|-------------------|------------------|---------------------|
| **list**     | Mutable    | Yes   | Yes        | O(1) for index, O(n) for membership test | O(1) amortized append  | Moderate         | Ordered, resizable sequence with duplicates |
| **tuple**    | Immutable  | Yes   | Yes        | O(1) for index, O(n) for membership test | N/A               | Low              | Fixed-size records; hashable keys for dicts |
| **set**      | Mutable    | No    | No         | O(1) membership/test     | O(1)               | Moderate-High    | Unique-item collection; fast de-dup and lookups |
| **frozenset**| Immutable  | No    | No         | O(1) membership/test     | N/A               | Moderate-High    | Hashable set for use as dict keys or in other sets |
| **dict**     | Mutable    | Yes*  | N/A        | O(1) key lookup          | O(1)               | High             | Key→value mapping; fast lookups and inserts |

\*Insertion-ordered since Python 3.7

### list
- **When to Use:**
  - You need an ordered, growable collection that may include duplicates.
  - Frequent random access by index.
  - Sequential iteration with modifications (via loops).
- **When to Avoid:**
  - Large datasets requiring many membership tests (`x in lst`)—those run in O(n).
  - Repeated insertions/removals at front or middle (shifts cost O(n)).
  - Use as dictionary keys (unhashable).
- **Performance Notes:**
  - `append()`: O(1) amortized (occasional resize costs more).
  - `pop()` from end: O(1); from front or middle: O(n).
  - `insert(i, x)`: O(n) due to element shifting.
  - `remove(x)`, `index(x)`: O(n). `sort()` / `sorted()`: O(n log n) (Timsort, stable).
  - `del lst[i]`: O(n) for front/middle, O(1) for end.
  - Slicing `lst[a:b]`: O(b-a) creates new list.
  - Iteration: O(n). Each slot stores an 8-byte pointer (plus the object itself), and the buffer over-allocates so that `append` is amortized O(1).
  - **Thread-safe for atomic operations only** (append, pop); not safe for multi-step sequences.

### tuple
- **When to Use:**
  - Fixed collections of items (e.g., coordinates, record rows, function return values).
  - Use as keys in dicts or elements in sets (hashable).
  - When immutability guarantees data integrity.
- **When to Avoid:**
  - You need to modify, append, or reorder elements.
  - Very large homogeneous sequences (consider NumPy arrays).
- **Performance Notes:**
  - Iteration O(n), the same as a list.
  - Lower memory than a list because there is no over-allocation buffer: `sys.getsizeof((1, 2, 3))` is 64 bytes versus 88 for `[1, 2, 3]` (CPython 3.12, 64-bit).
  - Hashable only if **all** its elements are: `hash((1, 2))` works, `hash((1, [2]))` raises `TypeError`.
  - Unpacking: O(n); equality checking: O(n) until first difference.
  - Single element tuple requires trailing comma: `(x,)`.

### set
- **When to Use:**
  - You need unique elements and O(1) membership tests.
  - Performing set algebra: union, intersection, difference.
  - Removing duplicates from a collection.
  - Checking exclusivity/containment efficiently.
- **When to Avoid:**
  - Order matters for your logic or output.
  - You require indexed access or slicing.
  - Storing unhashable items (lists, dicts, sets).
- **Performance Notes:**
  - `add()`, `remove()`, `discard()`, `in`: O(1) average; worst-case O(n) if many hash collisions.
  - Set operations: `union()` O(n+m), `intersection()` O(min(n,m)), `difference()` O(n).
  - Memory: hash-table overhead. An empty set is 216 bytes and a large one costs about 42 bytes per element, versus 8 bytes per list slot (measured on CPython 3.12, container only, excluding the elements themselves).
  - Iteration: O(n), order not guaranteed.

### frozenset
- **When to Use:**
  - All the benefits of `set` plus immutability.
  - Use as dict keys or inside other sets.
  - Function parameters where mutation must be prevented.
- **When to Avoid:**
  - Any scenario where you must update the collection (use `set` instead).
  - Building the collection incrementally: a `frozenset` has no `add()`, so build a `set` first and convert once at the end.
- **Performance Notes:**
  - Same time complexities as `set`.
  - Same memory per element as `set` (measured: about 42 bytes each).
  - Hashable, so it can be a dict key or an element of another set; equal frozensets hash equal regardless of element order.

### dict
- **When to Use:**
  - Mapping unique keys to values with constant-time lookups.
  - Configs, lookup tables, JSON-like structures, caching.
  - Counting occurrences (use alongside or instead of `Counter`).
- **When to Avoid:**
  - When insertion order pre-3.7 mattered and you needed extra ordering features (use `OrderedDict`).
  - Keys must be unhashable; use nested structures or custom key transforms instead.
  - When you need ordered iteration in Python <3.7 (use `OrderedDict`).
- **Performance Notes:**
  - `__getitem__`, `__setitem__`, `in`: O(1) average, O(n) worst-case (hash collisions).
  - `keys()`, `values()`, `items()`: O(1) to create (they are live views, not copies) and O(n) to iterate. `keys()` and `items()` views also support set operations, e.g. `d.keys() & other`.
  - `pop(key)`, `get(key)`: O(1) average.
  - Higher memory than a list: an empty dict is 64 bytes and a populated one costs about 52 bytes per item (CPython 3.12, measured; the key and value objects are extra).
  - Iteration follows insertion order (guaranteed by the language since 3.7); only avoid relying on it if you must support Python older than 3.7.
  - **Subclassing gotcha:** overriding `__setitem__` in a `dict` subclass does *not* affect `update()` or the constructor (they bypass it). Subclass `collections.UserDict` when you need every write to go through your override.

## Specialized Collections (`collections` Module)

| Type            | Mutability | Order | Lookup/Indexing | Insertion/Append | Best For                        | Avoid When                |
|-----------------|------------|-------|-----------------|------------------|----------------------------------|----------------------------|
| **deque**       | Mutable    | Yes   | O(n)            | O(1) at both ends| Fast queues/stacks, sliding windows | Random access/slicing    |
| **defaultdict** | Mutable    | Yes   | O(1)            | O(1)             | Grouping/counting w/o key checks   | Strict missing-key control |
| **OrderedDict** | Mutable    | Yes   | O(1)            | O(1)             | LRU caches, reordering on updates   | Python ≥3.7 plain dicts  |
| **Counter**     | Mutable    | Yes   | O(1)            | O(1)             | Frequency tallying                  | Values that aren't counts |
| **ChainMap**    | Mutable    | Yes   | O(k) for k maps | O(1)             | Layered configs, nested scopes      | Needing single underlying dict |
| **namedtuple**  | Immutable  | Yes   | O(1) via name   | N/A              | Lightweight record objects          | Behavior-rich classes    |
| **UserDict**, **UserList**, **UserString** | Mutable | Yes | Respective type | Respective type | Customizing behavior of built-ins | Direct subclassing built-ins |

### deque
- **When to Use:**
  - Real-time FIFO/LIFO queues, sliding window algorithms, BFS/DFS.
  - Efficient rotation of collections (`rotate(n)`).
  - Buffer with fast append/pop on both ends.
- **When to Avoid:**
  - Random-access lookups or slicing—those are O(n).
  - Frequent index-based updates; use `list` instead.
  - Memory-constrained scenarios with many tiny deques: an empty `deque` is 760 bytes (it allocates a 64-slot block up front), although a populated one costs only about 8 to 9 bytes per item (CPython 3.12, measured).
- **Performance:**
  - `append()`, `appendleft()`, `pop()`, `popleft()`: O(1)
  - `rotate(n)`: O(n)
  - Iteration: O(n). Indexing is fast near either end but O(n) toward the middle.
  - `deque(maxlen=n)` automatically discards from the opposite end when full, which gives you a ready-made "last n items" buffer.
  - **Thread-safe for individual atomic operations** (append/pop from either end are safe from race conditions), but this isn't the same as `queue.Queue` — see the dedicated comparison below.

### `collections.deque` vs. `queue.Queue` — Don't Confuse These

Both support fast append/pop, but they solve **different problems**:

| | `collections.deque` | `queue.Queue` / `LifoQueue` / `PriorityQueue` |
|---|---|---|
| Purpose | General-purpose double-ended sequence | Purpose-built for **producer-consumer** threading |
| Thread safety | Individual ops are atomic (GIL-protected); no blocking support | Fully thread-safe, with **blocking** `put()`/`get()` and internal locking |
| Blocking behavior | None — `popleft()` on empty raises `IndexError` immediately | `get()` can block until an item is available; supports timeouts |
| Typical use | Sliding windows, BFS, general fast queue/stack in single-threaded code | Coordinating work between threads (e.g., a thread pool's task queue) |

**Rule of thumb:** use `deque` for algorithmic/general-purpose queueing; use `queue.Queue` specifically when multiple threads need to safely hand off work items to each other with blocking semantics.

### defaultdict
- **When to Use:**
  - Auto-initialize missing keys without KeyError (e.g., grouping lists, counting).
  - Simplifies code: no need for `if key in dict` checks.
  - Building accumulator structures (e.g., `defaultdict(list)` or `defaultdict(int)`).
- **When to Avoid:**
  - You need to strictly detect missing keys (avoiding silent defaults).
  - Silent initialization masks logic errors.
  - Pickling with a lambda factory: `defaultdict(lambda: 0)` can't be pickled, while `defaultdict(int)` can. (`json.dumps` handles a defaultdict fine, since it is a `dict` subclass.)
  - Lookup side effect: `d[missing]` **inserts** the default, so merely reading can grow the dict; `k in d` and `d.get(k)` do not insert.
- **Performance:**
  - Same as dict: O(1) for get/set, with default factory invoked on miss.
  - Factory function called only for missing keys.
  - No performance penalty over `dict` for existing keys.

### OrderedDict
- **When to Use:**
  - Pre-3.7: preserve insertion order plus support methods like `move_to_end()`.
  - LRU cache implementations (reorder on access via `move_to_end()`).
  - Explicit ordering guarantees for backward compatibility.
  - Order-sensitive equality: two `OrderedDict`s with the same items in a different order are **not** equal (`OrderedDict(a=1, b=2) == OrderedDict(b=2, a=1)` is `False`), unlike plain dicts. Also offers `popitem(last=False)`.
- **When to Avoid:**
  - Python 3.7+ when plain insertion order is all you need: a plain `dict` is faster and uses about half the memory (measured: roughly 105 versus 52 bytes per item).
- **Performance:**
  - Extra bookkeeping (a doubly linked list) makes it slower and larger than a plain `dict`.
  - `move_to_end(key)`: O(1) operation.
  - Iteration: O(n), guaranteed insertion order.

### Counter
- **When to Use:**
  - Counting frequencies of hashable items in one pass.
  - Finding most common elements via `.most_common(n)`.
  - Set-like operations on tallies (add/subtract counts).
  - It is a `dict` subclass, so it keeps insertion order (3.7+), and a missing key reads as `0` without being inserted.
- **When to Avoid:**
  - When you need custom ordering beyond counts.
  - You need negative or zero counts to survive arithmetic: `a - b` and `a + b` silently drop results that are zero or negative (use `a.subtract(b)` to keep negatives).
  - Per-item data that isn't a count (use a plain `dict`).
- **Performance:**
  - `update()` and lookups: O(1) per element; uses `dict` underneath.
  - `.most_common(n)`: O(k log n) for k unique elements (it uses `heapq.nlargest`); `.most_common()` with no argument sorts everything, O(k log k).
  - Subtraction/intersection: O(n) where n is total keys.
  - Memory: the same as a `dict` (the counts are just the values), about 52 bytes per item measured.

### ChainMap
- **When to Use:**
  - Merging multiple dict contexts (e.g., multiple config layers, environment + defaults).
  - Scoping/namespace simulation without copying dicts.
  - Fallback chains: primary config → secondary config → defaults.
- **When to Avoid:**
  - You must update all underlying maps at once (only updates first map).
  - Needing a single unified dict (use `.copy()` or dict merge instead).
  - Performance-critical lookups across many layers.
- **Performance:**
  - Lookup: checks each mapping in turn, O(k) for k maps; returns first match.
  - Insertion/deletion: only affects first map, O(1) per operation.
  - Iteration yields each key **once** (no duplicates). `cm[key]` returns the value from the first map containing it, but the iteration order starts from the *last* map, so don't rely on it.
  - Memory: O(k) overhead for k mappings; no data duplication.

### namedtuple
- **When to Use:**
  - Lightweight record types with named fields and tuple semantics.
  - Function return values with named components (`return Point(x=1, y=2)`).
  - Keys in dicts or set members (hashable, lightweight).
- **When to Avoid:**
  - When you need methods or mutable attributes—use `@dataclass` or classic class.
  - High inheritance complexity; use classes instead.
  - Large datasets (use NumPy structured arrays for better performance).
- **Performance:**
  - Attribute access is O(1): a named lookup that maps onto a tuple index.
  - Memory is identical to a plain tuple of the same length (64 bytes for 3 fields). The field names live on the class, and instances have no `__dict__`.
  - Creation: `_make()` O(n); unpacking O(n).
  - Iteration: O(n), same as tuple.

## Priority Queues and Sorted Sequences — `heapq` and `bisect`

These two standard-library tools solve two very commonly asked "which collection would you use" interview scenarios and are missing from a plain list/dict/set comparison.

### `heapq` — Priority Queue via a Binary Min-Heap

`heapq` doesn't give you a new collection type — it turns a plain `list` into a binary min-heap in place, giving O(log n) push/pop with the smallest element always at index 0.

- **When to Use:**
  - You repeatedly need the smallest (or largest, with negation) item from a changing collection — task schedulers, Dijkstra's algorithm, merge-k-sorted-lists, top-k streaming problems.
  - You want O(log n) insert/extract instead of O(n log n) re-sorting on every change.
- **When to Avoid:**
  - You need to inspect anything other than the minimum efficiently — arbitrary-position lookup is still O(n).
  - You need a **max-heap** directly — `heapq` only implements a min-heap; negate values (or wrap in a tuple with a negated priority) to simulate one.
- **Performance:**
  - `heappush()`, `heappop()`: O(log n)
  - `heapify(list)`: O(n) — converts an existing list in place, faster than pushing n items one at a time
  - `heapq.nlargest(k, iterable)` / `nsmallest(k, iterable)`: O(n log k) — better than a full `sorted()` when k is small relative to n

```python
import heapq

tasks = []
heapq.heappush(tasks, (2, "medium priority"))
heapq.heappush(tasks, (1, "high priority"))
heapq.heappush(tasks, (3, "low priority"))

heapq.heappop(tasks)   # (1, "high priority") — smallest tuple wins
```

**Tie-break gotcha:** when two entries have the same priority, `heapq` compares the *next* tuple element, and if that isn't comparable you get an error:

```python
import heapq

h = []
heapq.heappush(h, (1, {"job": "a"}))
heapq.heappush(h, (1, {"job": "b"}))   # TypeError: '<' not supported between instances of 'dict' and 'dict'
```
The standard fix is a unique, increasing counter as a tie-breaker, which also makes equal priorities pop in insertion order:
```python
import heapq, itertools

h = []
counter = itertools.count()
heapq.heappush(h, (1, next(counter), {"job": "a"}))
heapq.heappush(h, (1, next(counter), {"job": "b"}))   # never compares the dicts
```

### `bisect` — Keeping a List Sorted Without Re-sorting

`bisect` provides binary-search insertion into an already-sorted list, avoiding an O(n log n) full re-sort every time you add one item.

- **When to Use:**
  - You maintain a sorted list that grows incrementally (leaderboard, sorted timestamps, grading-curve cutoffs) and need fast insert-in-place or "find where this value would fit."
  - You need fast range queries on sorted data (e.g., "how many values are ≤ x").
- **When to Avoid:**
  - The underlying list isn't already sorted — `bisect` assumes sortedness and won't detect or fix violations, silently giving wrong results.
  - You need frequent insertions **and** frequent removals from the middle — insertion into a `list` is still O(n) to shift elements, even though *finding* the position is O(log n); a `SortedList` from the third-party `sortedcontainers` library is a better fit at scale.
- **Performance:**
  - `bisect_left()`/`bisect_right()`: O(log n) to find the insertion point
  - `insort()`: O(log n) to find the position + O(n) to shift elements — the search is fast, but the actual list insertion isn't

```python
import bisect

scores = [10, 25, 40, 60]
bisect.insort(scores, 35)
print(scores)   # [10, 25, 35, 40, 60] — inserted in the correct sorted position
```

**Duplicates and keys:** `bisect_left` returns the position *before* any equal items and `bisect_right` (also called `bisect`) the position *after* them. Since Python 3.10 both accept a `key=` function, applied to the list elements only:
```python
import bisect

L = [1, 2, 2, 2, 3]
bisect.bisect_left(L, 2)    # 1
bisect.bisect_right(L, 2)   # 4

records = [("a", 1), ("b", 5), ("c", 9)]
bisect.bisect(records, 5, key=lambda r: r[1])   # 2 — the search value is compared against the key of each element
```

## Also Worth Knowing

- **`array.array('i', ...)`** stores raw C integers compactly (4 bytes per item), where a list pays an 8-byte pointer *plus* a 28-byte `int` object for each value. Handy for large numeric sequences when NumPy isn't available.
- **`range`** is a lazy sequence, not a list: it is always 48 bytes, and `len`, indexing and `in` on integers are O(1) arithmetic (`999_999_999 in range(10**9)` is instant).
- **Dict views behave like sets:** `d.keys() & other_keys`, `d1.keys() - d2.keys()` and `d.items() & other.items()` work without building intermediate sets.

## Common Pitfalls & Best Practices

### Mutability & Hashing
- **Never use mutable objects (list, dict, set) as dict keys or set members**—they're unhashable.
- **Mutable default arguments** (`def f(x, bucket=[])`) share one object across every call; default to `None` and create the container inside the function (see `02-Functions.md`).

### Iteration Safety
- **Don't modify a list while iterating over it** (causes skipped/repeated elements). Use list comprehension or `list.copy()` instead.
- **Set/dict mutation during iteration** raises `RuntimeError` when the *size* changes (adding or removing keys). Overwriting the value of an existing dict key is fine. Iterate over `.copy()` or a list of the items when you need to add or remove.

### Performance Traps
- **Checking `x in list` is O(n)**; use `set` for large datasets with many lookups.
- **String concatenation in loops**: use `''.join([...])` or `io.StringIO()`, not `+=`.
- **Deepcopy is slow**: verify you need it; often a reference or shallow copy suffices.
- **Re-sorting a growing list on every insert is O(n log n) each time**; use `bisect.insort` (O(n) but with an O(log n) search) or a `heapq` if you only ever need the extreme value.

### Memory Awareness
- **Empty containers still consume memory** (CPython 3.12, 64-bit, via `sys.getsizeof`): `[]` is 56 bytes, `()` 40, `{}` 64, `set()` 216, `deque()` 760.
- **Slicing creates a new list**: `lst[1:1000]` copies 999 references (not the objects themselves); use `itertools.islice` to iterate over a window without copying.

---

## Choosing the Right Collection: Decision Tree

1. **Need mutability?**
   - **Yes:** `list`, `dict`, `set`, `deque`, `defaultdict`, `Counter`, `OrderedDict`
   - **No:** `tuple`, `frozenset`, `namedtuple`

2. **Need ordering?**
   - **Yes:** `list`, `tuple`, `dict` (3.7+) and its subclasses `defaultdict`, `Counter` and `OrderedDict`, plus `deque` and `namedtuple`
   - **No:** `set`, `frozenset`

3. **Need O(1) random access by index?**
   - **Yes:** `list`, `tuple`, `namedtuple`
   - **No:** `set`, `frozenset`, `dict`, `deque`

4. **Need fast membership tests (`x in collection`)?**
   - **O(1):** `set`, `frozenset`, `dict`, `defaultdict`, `Counter`
   - **O(n):** `list`, `tuple`, `deque`

5. **Need to use as dict key or set member (hashable)?**
   - **Use:** `tuple`, `frozenset`, `namedtuple`, or scalar types (`int`, `str`, etc.)
   - **Avoid:** `list`, `dict`, `set`

6. **Frequent insertions/deletions at both ends?**
   - **Yes:** `deque` (O(1) for both ends)
   - **No:** `list` (O(n) for front/middle)

7. **Need frequency counts or auto-key defaults?**
   - **Frequency counts:** `Counter`
   - **Auto-initialize missing keys:** `defaultdict(list)` or `defaultdict(int)`

8. **Need lightweight records with named attributes?**
   - **Yes:** `namedtuple` (immutable) or `@dataclass` (mutable, more features)
   - **No:** use dict or class instance

9. **Need the smallest/largest item repeatedly from a changing collection, or a sorted list you insert into incrementally?**
   - **Repeated min/max extraction:** `heapq`
   - **Maintain sorted order while inserting:** `bisect.insort`
   - **Thread-safe producer-consumer handoff:** `queue.Queue` (not `deque`)

## Quick Reference: Memory & Performance

| Collection  | Empty size | Per-item cost* | Insertion at end | Lookup | Membership | Notes |
|-------------|-----------|----------------|------------------|--------|------------|-------|
| `list`      | 56 B | 8 B (a pointer) | O(1) amortized | O(1) by index | O(n) | Over-allocates so appends are amortized O(1) |
| `tuple`     | 40 B | 8 B | N/A | O(1) by index | O(n) | Immutable; hashable if all elements are |
| `dict`      | 64 B | ~52 B | O(1) | O(1) by key | O(1) | Insertion-ordered (3.7+) |
| `set`       | 216 B | ~42 B | O(1) | N/A | O(1) | Unordered, unique, hashable items |
| `frozenset` | 216 B | ~42 B | N/A | N/A | O(1) | Immutable and hashable |
| `deque`     | 760 B | ~8-9 B | O(1) both ends | O(n) toward the middle | O(n) | Stored in blocks of 64 items |
| `Counter`   | 80 B | ~52 B | O(1) | O(1) | O(1) | `dict` subclass; ordered; missing key reads as 0 |
| `OrderedDict` | 128 B | ~105 B | O(1) | O(1) | O(1) | Extra linked-list bookkeeping |
| `defaultdict` | 72 B | ~52 B | O(1) | O(1) | O(1) | `d[missing]` inserts the default |
| `namedtuple` | tuple-sized (64 B for 3 fields) | 8 B per field | N/A | O(1) by name or index | O(n) | No per-instance `__dict__` |
| `heapq` (on a list) | 56 B | 8 B | O(log n) push | O(1) min only | O(n) | In-place min-heap over a list |

\*Container overhead only, excluding the key and value objects themselves (a small `int` object is 28 bytes). Figures come from `sys.getsizeof` on 64-bit CPython 3.12 with 100,000 items and will vary by version and platform; use them for relative comparison, not exact budgeting.

## Interview Gotchas

- **"How would you implement a priority queue in Python?"** — `heapq`, not manually re-sorting a list on every insert.
- **"How do you keep a list sorted as you insert into it?"** — `bisect.insort`; know that the search is O(log n) but the actual shift is still O(n).
- **`deque` is not `queue.Queue`** — one is a general-purpose double-ended container, the other is a blocking, thread-safe handoff primitive for producer-consumer patterns.
- **`heapq` is a min-heap only** — simulate a max-heap by negating values or using `(-priority, item)` tuples.
- **`bisect` assumes the list is already sorted** — it won't warn you if it isn't; it'll just silently return a meaningless insertion point.
- **`Counter` keeps insertion order and returns 0 for a missing key without inserting it**, but `a - b` and `a + b` drop non-positive counts (`subtract()` keeps them).
- **`dict.keys()`, `.values()` and `.items()` are O(1) live views**, not copies, and `keys()`/`items()` support set operators.
- **Subclass `UserDict`, not `dict`, when you override `__setitem__`**: `dict.update()` and the constructor bypass the override.
- **`heapq` with equal priorities needs a tie-breaker** like `(priority, next(counter), item)`, or comparing the payloads can raise `TypeError`.
- **A `namedtuple` is exactly as small as a tuple**; its field names are stored once on the class.

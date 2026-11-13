# Python Performance and Profiling

Senior interviews expect more than "Python is slow" or premature use of threads. A strong answer identifies the bottleneck, measures it, chooses the right optimization level, and explains the trade-off.

## 1. Start with the right cost model

Big-O describes how work grows as input grows. It does not capture constants, allocations, cache locality, I/O latency, or Python interpreter overhead.

| Complexity | Typical examples | Growth intuition |
|---|---|---|
| O(1) | Dict/set average lookup; list index | Does not grow with input size |
| O(log n) | Binary search; heap push/pop | Input can grow exponentially for a linear increase in steps |
| O(n) | Linear scan; copying a list | Doubles when input doubles |
| O(n log n) | Comparison sorting | Common efficient sorting bound |
| O(n²) | Nested scan over all pairs | Four times the work when input doubles |
| O(2ⁿ) | Exhaustive subset search | Quickly infeasible |

Complexity should be stated with assumptions. Dict/set operations are average O(1), but worst-case collision behavior can be worse. List membership is O(n); list indexing is O(1).

## 2. A disciplined optimization workflow

1. **Define the objective.** Latency, throughput, memory, startup time, cost, or tail latency?
2. **Create a representative workload.** Include realistic input sizes and distributions.
3. **Establish a baseline.** Record wall-clock time, CPU time, memory, and environment.
4. **Profile before changing code.** Determine whether cost is CPU, I/O, allocations, lock contention, or an external dependency.
5. **Make one change.** Keep behavior and correctness tests fixed.
6. **Re-measure.** Compare multiple runs and include variance.
7. **Check the trade-off.** Faster code may use more memory or be harder to maintain.

Do not optimize a microbenchmark that does not represent the application's workload.

## 3. Benchmarking with `timeit`

```python
from timeit import repeat

def build_with_append(n: int) -> list[int]:
    result = []
    for value in range(n):
        result.append(value)
    return result

def build_with_comprehension(n: int) -> list[int]:
    return [value for value in range(n)]

append_times = repeat(
    stmt="build_with_append(10_000)",
    globals=globals(),
    repeat=5,
    number=100,
)
comprehension_times = repeat(
    stmt="build_with_comprehension(10_000)",
    globals=globals(),
    repeat=5,
    number=100,
)

print(min(append_times), min(comprehension_times))
```

Use the minimum of several repeats as a rough estimate of best uncontended execution, but inspect distributions when the workload is noisy. `timeit` disables garbage collection by default; enable it when GC cost is part of the behavior you intend to measure. Benchmark realistic inputs, not only tiny values.

For command-line use:

```bash
python -m timeit -s "data = list(range(1000))" "sum(data)"
```

## 4. Profiling CPU time with `cProfile`

```python
import cProfile
import pstats

def workload() -> int:
    return sum(value * value for value in range(100_000))

profiler = cProfile.Profile()
profiler.enable()
workload()
profiler.disable()

pstats.Stats(profiler).strip_dirs().sort_stats("cumtime").print_stats(20)
```

Interpretation:

- **`tottime`**: time spent in a function excluding subcalls.
- **`cumtime`**: time in the function plus its subcalls.
- Many calls to a cheap function can dominate total time.
- High cumulative time in a wrapper may mean the actual cost is in a child call.

Use sampling profilers for lower-overhead production profiling when appropriate. Profilers perturb execution; validate improvements with an unprofiled benchmark.

## 5. Memory profiling

```python
import tracemalloc

tracemalloc.start()
data = [bytearray(1024) for _ in range(10_000)]
current, peak = tracemalloc.get_traced_memory()
print(f"current={current:,} peak={peak:,}")
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics("lineno")[:5]:
    print(stat)
tracemalloc.stop()
```

`tracemalloc` tracks Python memory allocations made after tracing starts. It does not necessarily include all native allocations made by extension modules or the process's total resident memory.

`sys.getsizeof(obj)` reports the shallow size of one object; it does not recursively include objects referenced by it:

```python
import sys

values = [object() for _ in range(100)]
print(sys.getsizeof(values))  # list container and its pointer array, not each object
```

For process-level memory, use operating-system metrics or a process-memory tool. Distinguish allocated Python objects, allocator arenas, and RSS.

## 6. Common optimization levers

### Improve the algorithm first

Replacing O(n²) with O(n log n) or O(n) usually matters more than changing syntax. Use a set for repeated membership tests, a heap for repeated min/max extraction, and `bisect` only when the list is already sorted and insertion costs are acceptable.

### Reduce unnecessary work and allocation

- Stream records rather than materializing the whole input when possible.
- Avoid repeated parsing, sorting, conversions, and database/network calls.
- Use `join()` for building strings from many fragments.
- Avoid copying large nested structures unless ownership requires it.
- Use generators for lazy pipelines, but do not assume they are always faster than lists.

### Choose the right data structure

The collection guide explains list/dict/set/deque trade-offs. The right structure often dominates micro-optimizations.

### Use caching carefully

Caching can remove repeated expensive pure computation. Define a maximum size, invalidation strategy, and correctness behavior when inputs or external state change.

### Consider `__slots__` only after measurement

`__slots__` can reduce per-instance memory by avoiding an instance `__dict__` when the class design allows it. It changes dynamic attribute behavior and inheritance details; it is not a universal speed switch.

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
```

Subclasses, weak references, descriptors, pickling, and debugging tools may require additional care.

## 7. CPU-bound vs I/O-bound

- **CPU-bound:** time is spent performing computations. Consider a better algorithm, vectorized native libraries, process pools, free-threaded Python when the ecosystem supports it, or native extensions.
- **I/O-bound:** time is spent waiting on files, sockets, databases, or other services. Consider asynchronous I/O or thread pools, with connection limits and timeouts.
- **Mixed workloads:** profile each stage. Moving all work to a process pool can make serialization and startup overhead worse.

For normal GIL-enabled CPython, threads are useful for overlapping I/O but generally do not execute Python bytecode in parallel across cores. This is not a blanket statement for free-threaded builds, C extensions that release the GIL, or other Python implementations.

## 8. Benchmark pitfalls

- Comparing different Python versions, builds, machine loads, or input data.
- Timing one run and drawing a conclusion.
- Including setup in one benchmark but not the other.
- Forgetting that `timeit` disables GC by default.
- Measuring a tiny function whose execution is dominated by timer overhead.
- Optimizing average latency while ignoring p95/p99.
- Using profiling results as absolute timing results.
- Assuming CPython implementation details apply to PyPy or other interpreters.

## 9. Senior-level discussion prompts

**How would you investigate a Python service with rising p99 latency?**  
Segment latency by dependency and endpoint, inspect traces and histograms, compare p50/p95/p99, check saturation/queueing/retries/timeouts, profile representative slow requests, and identify whether the issue is CPU, lock contention, memory pressure, or downstream I/O.

**Why might a generator reduce memory but not improve latency?**  
It avoids eager materialization, but adds iteration overhead and may prevent vectorized/batched operations. The objective may be peak memory rather than throughput.

**Why is `sys.getsizeof()` not a complete memory measurement?**  
It is shallow and does not include referents, all native allocations, allocator fragmentation, or process-wide resident memory.

**When is multiprocessing slower than threading?**  
For small tasks, large serialized arguments/results, frequent inter-process communication, high startup cost, or workloads already dominated by I/O.

## 10. Checklist

- [ ] State the objective and workload before optimizing.
- [ ] Know `timeit`, `cProfile`, `pstats`, and `tracemalloc`.
- [ ] Distinguish shallow size from total/process memory.
- [ ] Explain CPU-bound and I/O-bound workloads accurately.
- [ ] Optimize algorithms and data access before syntax.
- [ ] Re-test correctness and benchmark after every material change.

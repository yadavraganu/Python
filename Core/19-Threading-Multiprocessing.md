Python multithreading lets you run multiple threads (smaller units of a process) concurrently, ideal for tasks that involve *waiting* — network requests, file I/O (**I/O-bound tasks**). Due to Python's **Global Interpreter Lock (GIL)**, though, it does not provide true parallelism for **CPU-bound** tasks within a single CPython process — that's what **multiprocessing** is for.

# Threading

## What is a Thread?

Think of a program as a cook in a kitchen (a **process**), following a recipe.

* **Single-threading:** the cook does one step at a time — chop, then boil, then stir. If boiling takes 5 minutes, the cook just stands there waiting.
* **Multi-threading:** the cook starts boiling water (one **thread**), and while it heats, starts chopping vegetables (a second **thread**). Waiting time is used productively.

A thread is a separate flow of execution. Threads within the same process **share the same memory space** — lightweight, but requiring careful management to avoid conflicts.

#### The Global Interpreter Lock (GIL)

The **GIL** is a mutex in CPython allowing only one thread to execute Python bytecode at a time within a single process. This is why multithreading isn't effective for CPU-bound tasks. However, when a thread is waiting on an external operation (file read, network response), it **releases the GIL**, letting other threads run — perfect for **I/O-bound** work.

## Creating Your First Thread

```python
import threading
import time

def worker(name, duration):
    print(f"Thread {name}: Starting...")
    time.sleep(duration)
    print(f"Thread {name}: Finishing.")

thread1 = threading.Thread(target=worker, args=("Alice", 2))
thread2 = threading.Thread(target=worker, args=("Bob", 3))

print("Main: Before running threads.")
thread1.start()
thread2.start()
print("Main: All threads are running. Now waiting for them to finish.")
thread1.join()
thread2.join()
print("Main: All done.")
```

**Output:**
```
Main: Before running threads.
Thread Alice: Starting...
Thread Bob: Starting...
Main: All threads are running. Now waiting for them to finish.
Thread Alice: Finishing.
Thread Bob: Finishing.
Main: All done.
```

Key methods:
* `start()` — begins the thread's activity.
* `join()` — blocks the caller until the thread finishes.

## Sharing Data and Synchronization

### The Problem: Race Conditions

```python
import threading

shared_counter = 0

def increment_counter():
    global shared_counter
    for _ in range(1000000):
        shared_counter += 1

t1 = threading.Thread(target=increment_counter)
t2 = threading.Thread(target=increment_counter)
t1.start(); t2.start()
t1.join(); t2.join()

print(f"Final counter value: {shared_counter}")
# Expected: 2000000. Actual: a smaller, unpredictable number.
```

`shared_counter += 1` isn't **atomic** — it's read, add, write. Two threads can read the same value before either writes back, losing updates.

### The Solution: `Lock`

```python
import threading

shared_counter = 0
lock = threading.Lock()

def increment_with_lock():
    global shared_counter
    for _ in range(1000000):
        lock.acquire()
        try:
            shared_counter += 1
        finally:
            lock.release()
```

Cleaner with `with`:
```python
def increment_with_lock_cleaner():
    global shared_counter
    for _ in range(1000000):
        with lock:
            shared_counter += 1
```

### `RLock` — Reentrant Lock

A plain `Lock` **cannot** be acquired twice by the same thread — attempting to do so deadlocks the thread against itself. `threading.RLock` (reentrant lock) tracks *which thread* holds it and how many times, allowing the same thread to acquire it repeatedly (each `acquire()` needs a matching `release()`):

```python
lock = threading.RLock()

def outer():
    with lock:
        inner()   # same thread re-acquiring the same lock — fine with RLock, deadlocks with plain Lock

def inner():
    with lock:
        pass
```

Useful when a locked method might call another locked method on the same object, from the same thread (common in recursive or nested-call code protecting shared state).

### Deadlocks

A **deadlock** occurs when two or more threads each hold a lock the other needs, and neither can proceed:

```python
lock_a = threading.Lock()
lock_b = threading.Lock()

def thread_1():
    with lock_a:
        with lock_b:   # waits for lock_b
            pass

def thread_2():
    with lock_b:
        with lock_a:   # waits for lock_a — deadlock if thread_1 already holds lock_a
            pass
```

**Common fixes:** always acquire multiple locks in the same global order across all threads; use `lock.acquire(timeout=...)` to fail instead of hanging forever; or restructure to avoid needing more than one lock at a time.

## Thread-Safe Communication: The `queue` Module

Explicit locks get complex fast. The thread-safe `queue.Queue` handles locking internally — safely put items in from one thread, get them out from another.

* `q.put(item)` — add an item.
* `q.get()` — remove and return an item; **blocks** if the queue is empty.

### Producer-Consumer Example

```python
import threading, queue, time, random

def producer(q):
    for i in range(5):
        item = f"item-{i}"
        time.sleep(random.uniform(0.1, 0.5))
        q.put(item)
        print(f"Producer produced {item}")
    q.put(None)   # sentinel value to signal the end

def consumer(q):
    while True:
        item = q.get()
        if item is None:
            break
        print(f"Consumer consumed {item}")
        q.task_done()

work_queue = queue.Queue()
producer_thread = threading.Thread(target=producer, args=(work_queue,))
consumer_thread = threading.Thread(target=consumer, args=(work_queue,))
producer_thread.start(); consumer_thread.start()
producer_thread.join(); consumer_thread.join()
print("All work is done.")
```

## ThreadPoolExecutor

`concurrent.futures.ThreadPoolExecutor` gives a modern, high-level interface — submit tasks, get back `Future` objects instead of managing `start()`/`join()` yourself.

```python
import concurrent.futures
import requests

URLS = ['https://www.google.com', 'https://www.python.org',
        'https://www.github.com', 'https://www.microsoft.com']

def fetch_url(url):
    try:
        response = requests.get(url, timeout=5)
        return f"{url}: {response.status_code}"
    except requests.exceptions.RequestException as e:
        return f"{url}: Error ({e})"

with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(fetch_url, url) for url in URLS]
    for future in concurrent.futures.as_completed(futures):
        print(future.result())
```

**Benefits:** simpler API (no manual `start()`/`join()`), easy result retrieval, exceptions re-raised on `future.result()`, automatic worker lifecycle management.

### Other Threading Primitives

* **Daemon Threads** — `thread.daemon = True` marks a background thread that won't block program exit.
* **`threading.Event`** — simple signal: one thread sets it (`event.set()`), others wait (`event.wait()`).
* **`threading.Semaphore`** — like a lock, but allows N threads to hold it simultaneously; useful for capping concurrent access to a limited resource (e.g., a pool of DB connections).
* **`threading.Condition`** — combines a lock with the ability to wait for/notify about a specific condition being met (`wait()`, `notify()`, `notify_all()`) — the building block behind more complex producer-consumer patterns than a plain `Queue` covers.

# Multiprocessing

Since threads share one GIL-limited interpreter, they can't achieve real parallelism for CPU-bound work (tight loops, number crunching, image processing). The `multiprocessing` module sidesteps the GIL entirely by running code in **separate processes** — each with its **own Python interpreter and own memory space** — achieving genuine parallel execution across CPU cores.

## Process vs. Thread — The Core Trade-off

| | Threading | Multiprocessing |
|---|---|---|
| GIL impact | Limited by GIL — no true parallel *CPU* execution | Each process has its own GIL — real parallelism |
| Memory | Shared between threads (lightweight, but risk of races) | Separate per process (safer, but must explicitly share data) |
| Best for | I/O-bound work (network, disk, waiting) | CPU-bound work (computation, data crunching) |
| Startup cost | Cheap | More expensive (spinning up a new interpreter) |
| Communication | Shared variables + locks | `Queue`, `Pipe`, or shared memory objects — explicit IPC |
| Data passed to worker | Any object (shared memory) | Must be **picklable** — serialized to cross process boundaries |

## Creating Your First Process

The API deliberately mirrors `threading` closely:

```python
import multiprocessing
import time

def worker(name, duration):
    print(f"Process {name}: Starting...")
    time.sleep(duration)
    print(f"Process {name}: Finishing.")

if __name__ == "__main__":     # REQUIRED on Windows/macOS (spawn start method) — see note below
    p1 = multiprocessing.Process(target=worker, args=("Alice", 2))
    p2 = multiprocessing.Process(target=worker, args=("Bob", 3))

    p1.start()
    p2.start()
    p1.join()
    p2.join()
    print("Main: All done.")
```

**Why the `if __name__ == "__main__":` guard is mandatory here (not just good practice):** on platforms using the `spawn` start method (the default on Windows and macOS), each child process **re-imports** the main module to set itself up. Without the guard, that re-import would recursively spawn more processes — an infinite loop of process creation. Linux's default `fork` start method copies the existing process instead, so this bites people less often there, but the guard is required for portable code.

## `Pool` — Distributing Work Across Processes

For simple "run this function over a bunch of inputs" parallelism, `multiprocessing.Pool` is far less code than managing individual `Process` objects:

```python
import multiprocessing

def square(n):
    return n * n

if __name__ == "__main__":
    with multiprocessing.Pool(processes=4) as pool:
        results = pool.map(square, range(10))
    print(results)   # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

* `pool.map(func, iterable)` — like the built-in `map`, but distributed across worker processes, blocking until all results are ready.
* `pool.apply_async(func, args)` — non-blocking; returns a result object you `.get()` later.
* `pool.starmap(func, iterable_of_tuples)` — like `map`, but unpacks each tuple as multiple arguments.

## Sharing State Between Processes

Since processes **don't share memory** by default, mutating a plain global variable across processes silently does nothing useful — each process gets its own copy. Explicit tools are required:

### `Value` and `Array` — Shared Memory Primitives

```python
import multiprocessing

def increment(shared_val, lock):
    for _ in range(100000):
        with lock:
            shared_val.value += 1

if __name__ == "__main__":
    counter = multiprocessing.Value('i', 0)   # 'i' = C int, shared across processes
    lock = multiprocessing.Lock()

    processes = [multiprocessing.Process(target=increment, args=(counter, lock)) for _ in range(4)]
    for p in processes: p.start()
    for p in processes: p.join()

    print(counter.value)   # 400000 — correct, thanks to the shared Value + Lock
```

`Value` and `Array` allocate memory from a shared block that both the parent and child processes map into their own address space — a low-level but efficient way to share small amounts of data, still needing a `Lock` for the same race-condition reasons as threading.

### `Manager` — Shared, Higher-Level Objects

For shared `dict`s, `list`s, or more complex structures, `multiprocessing.Manager()` runs a separate server process that holds the real object; other processes get proxies that transparently forward operations to it (slower than `Value`/`Array`, but far more flexible):

```python
import multiprocessing

def worker(shared_dict, key, value):
    shared_dict[key] = value

if __name__ == "__main__":
    with multiprocessing.Manager() as manager:
        shared_dict = manager.dict()
        processes = [multiprocessing.Process(target=worker, args=(shared_dict, i, i * i)) for i in range(5)]
        for p in processes: p.start()
        for p in processes: p.join()
        print(dict(shared_dict))   # {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

### `Queue` and `Pipe` — Message Passing Between Processes

Mirrors `threading`'s producer-consumer pattern, but `multiprocessing.Queue` handles the pickling/unpickling needed to move data across process boundaries:

```python
import multiprocessing

def producer(q):
    for i in range(5):
        q.put(f"item-{i}")
    q.put(None)

def consumer(q):
    while True:
        item = q.get()
        if item is None:
            break
        print(f"Consumed {item}")

if __name__ == "__main__":
    q = multiprocessing.Queue()
    p1 = multiprocessing.Process(target=producer, args=(q,))
    p2 = multiprocessing.Process(target=consumer, args=(q,))
    p1.start(); p2.start()
    p1.join(); p2.join()
```

`multiprocessing.Pipe()` offers a lower-level, two-endpoint alternative for direct one-to-one process communication, faster than a `Queue` for that specific case but less flexible for many-to-many patterns.

## Expert Level: `ProcessPoolExecutor`

The `concurrent.futures` module offers a process-based pool with the **same interface** as `ThreadPoolExecutor` — swapping between them for CPU-bound vs. I/O-bound work is often a one-line change:

```python
import concurrent.futures

def cpu_heavy(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as executor:
        futures = [executor.submit(cpu_heavy, n) for n in [10_000_000] * 4]
        for future in concurrent.futures.as_completed(futures):
            print(future.result())
```

## Common Multiprocessing Pitfalls

- **Everything passed to a child process must be picklable** (functions defined at module level, not lambdas or local closures, and picklable arguments/return values) — a very common "why did my multiprocessing code fail with `PicklingError`" bug.
- **Process creation overhead is real** — spawning a process is far more expensive than a thread; for very short tasks, the overhead can exceed the benefit of parallelism.
- **`fork` vs. `spawn` vs. `forkserver` start methods** behave differently across platforms (Linux defaults to `fork`, Windows/macOS to `spawn`) — code relying on inherited state via `fork` can silently break on a `spawn`-only platform.
- **Debugging is harder** — print statements and tracebacks from child processes can interleave unpredictably or fail to surface cleanly; logging with process-aware formatting helps.

# Threading vs. Multiprocessing vs. Asyncio — Choosing the Right Tool

| Scenario | Best Tool | Why |
|---|---|---|
| Waiting on network requests, file I/O, DB queries | `threading` or `asyncio` | I/O releases the GIL; threads/coroutines can overlap the waiting |
| Heavy CPU computation (math, image/data processing) | `multiprocessing` | Bypasses the GIL entirely via separate interpreters/cores |
| Thousands of concurrent I/O-bound connections (e.g., a web server) | `asyncio` | Single-threaded event loop avoids per-thread memory/context-switch overhead at scale |
| A handful of I/O-bound tasks, simple to reason about | `threading` (`ThreadPoolExecutor`) | Simpler mental model than async/await for small-scale concurrency |
| Need true parallel speedup on multi-core hardware | `multiprocessing` | Only multiprocessing achieves genuine parallel *execution*, not just concurrency |

**The one-line interview answer:** *threading and asyncio give you concurrency (interleaved progress, good for waiting); multiprocessing gives you parallelism (simultaneous execution, good for computing) — and the GIL is exactly the reason Python needs multiprocessing at all for CPU-bound speedups.*

## Notes & Gotchas

- **"Concurrency vs. parallelism"** is the single most important distinction in this whole topic — threading/asyncio interleave work; only multiprocessing runs Python code truly simultaneously.
- **The GIL is per-process** — this is *why* multiprocessing works around it: each process gets its own interpreter and its own GIL, so they don't contend with each other.
- **`if __name__ == "__main__":` is not optional for multiprocessing** on `spawn`-based platforms — omitting it can cause runaway recursive process creation.
- **Data shared across processes must be explicitly shared** (`Value`, `Array`, `Manager`, or message-passed via `Queue`/`Pipe`) — unlike threads, which share memory by default.
- **`RLock` vs. `Lock`**: use `RLock` when the same thread might need to re-acquire a lock it already holds (e.g., recursive calls into locked methods); a plain `Lock` would deadlock in that case.
- **Deadlocks come from inconsistent lock-acquisition order** across threads — the standard fix is a globally consistent ordering, not more locks.
- **Multiprocessing has real overhead** — it's a poor fit for many small, fast tasks where process startup cost dwarfs the actual work.

# Python Asynchronous Programming (`asyncio`): Beginner to Advanced Guide

> **Version baseline:** Examples target Python 3.11+ unless a different version is stated. `asyncio.TaskGroup` and `asyncio.timeout()` require Python 3.11+. The asynchronous-iterator form of `asyncio.as_completed()` is available from Python 3.13.

Asynchronous programming lets a program make progress on other work while an operation is waiting for I/O. Python's main standard-library toolkit for this is **`asyncio`**.

`asyncio` is especially useful for I/O-bound workloads such as calling APIs, handling many network connections, and coordinating asynchronous database or messaging clients. It does not automatically make code faster, and it is not a substitute for parallel execution of CPU-heavy Python code.

## 1. The Mental Model: Event Loop and Cooperative Multitasking

An event loop runs callbacks and asyncio tasks. In the usual `asyncio` model, a given event loop runs in **one OS thread** and executes **one task at a time** in that thread. When a task reaches an `await` that suspends, the event loop can run another ready task. When the awaited operation becomes ready, the loop can resume the original task.

Think of it as one worker coordinating many jobs:

1. Start a task.
2. The task begins an operation, such as sending a network request.
3. If the operation is not ready, the task suspends at `await`.
4. The event loop runs another ready task instead of leaving the thread idle.
5. When the operation completes, the original task becomes ready to resume.

### Key terms

- **Concurrency:** multiple operations make progress over overlapping periods of time. They do not necessarily execute at the exact same instant.
- **Parallelism:** multiple operations execute at the same instant, usually on different CPU cores.
- **Cooperative multitasking:** an asyncio task gives the event loop opportunities to run other work by suspending at await points. The event loop does not normally interrupt a task in the middle of ordinary Python code.
- **Blocking operation:** a call that holds up the event-loop thread instead of yielding control while it waits.

### Important consequences

- Between suspension points, a task's synchronous Python code runs without another task on that same event loop interleaving with it.
- Logical race conditions are still possible when a task reads or changes shared state across an `await`. Another task may run during that suspension and change the state.
- A CPU-heavy loop or blocking call such as `time.sleep()` can block the event loop and delay every task using it.
- A single event loop does not provide CPU parallelism. Use threads or processes when the workload requires them.
- `await` does **not** always cause a task switch. If the awaited operation is already complete, the coroutine may continue without suspending.

> **Interview point:** `asyncio` primarily improves I/O concurrency by overlapping waiting. It does not make a CPU-bound algorithm faster merely because the function is declared with `async def`.

## 2. Core Syntax and Program Entry Point

- **`async def`** declares a coroutine function. Calling it creates a coroutine object; it does not execute the function body immediately.
- **`await`** waits for an awaitable (for example, a coroutine, Task, or Future). If waiting suspends, control returns to the event loop so other work can run.
- **`asyncio.run()`** is the normal top-level entry point for an asyncio program. It creates and manages an event loop, runs the awaitable to completion, finalizes asynchronous generators, shuts down the default executor, and closes the loop.

```python
import asyncio


async def say_hello():
    print("Hello...")
    await asyncio.sleep(1)  # Non-blocking delay for this task
    print("...World!")


if __name__ == "__main__":
    asyncio.run(say_hello())
```

`asyncio.run()` cannot be called while another event loop is already running in the **same thread**. In Jupyter notebooks and many async frameworks, use `await say_hello()` at the appropriate async entry point instead.

### The classic bug: forgetting `await`

```python
import asyncio


async def main():
    asyncio.sleep(1)        # BUG: creates a coroutine object but never awaits it
    await asyncio.sleep(1)  # Correct: waits for the sleep operation


asyncio.run(main())
```

The first call creates a coroutine object that is never awaited or scheduled. It does not perform the sleep and typically causes a `RuntimeWarning` such as `coroutine 'sleep' was never awaited` when the object is discarded.

The same rule explains this common behavior:

```python
result = some_async_function()
```

`result` is a coroutine object, not the function's eventual return value. Await it or schedule it as a Task.

## 3. Coroutines vs. Tasks vs. Futures

| Concept | Meaning | Typical use |
|---|---|---|
| **Coroutine object** | Created by calling an `async def` function. It describes asynchronous work but is not automatically scheduled. | `coro = fetch_data()` |
| **Task** | Wraps and schedules a coroutine to run on an event loop. A Task is awaitable and exposes its result when complete. | `task = asyncio.create_task(fetch_data())` |
| **Future** | A lower-level object representing a result that will become available later. `Task` is a subclass of `Future`. | Usually created and managed by asyncio libraries, not application code. |

### Sequential awaiting vs. concurrent scheduling

Awaiting independent coroutines one after another does not schedule those operations together. They run as a sequence from the caller's perspective. Each one may still suspend and allow *other already-scheduled tasks* to run.

```python
import asyncio


async def work(name):
    await asyncio.sleep(1)
    return name


async def main():
    # Sequential: these two operations take about 2 seconds in total.
    a = await work("a")
    b = await work("b")

    # Concurrent: schedule both before awaiting their results.
    task_a = asyncio.create_task(work("a"))
    task_b = asyncio.create_task(work("b"))

    a = await task_a
    b = await task_b
    print(a, b)


asyncio.run(main())
```

The two scheduled tasks can overlap their one-second waits, so the concurrent section takes about one second rather than two, excluding overhead and scheduling variation.

> **Important distinction:** `await` means “wait for this result.” It does not, by itself, mean “run this operation concurrently with the next line.” Schedule independent work with `create_task()`, `gather()`, or `TaskGroup` when appropriate.

## 4. Running and Managing Multiple Tasks

### 4.1 `asyncio.gather()`

`asyncio.gather(*aws)` schedules the supplied awaitables concurrently and returns their results in **the same order as the inputs**, not completion order.

```python
import asyncio


async def process_data(item_id):
    await asyncio.sleep(1)
    return f"Data {item_id} processed"


async def main():
    results = await asyncio.gather(
        process_data(1),
        process_data(2),
        process_data(3),
    )
    print(results)
    # ['Data 1 processed', 'Data 2 processed', 'Data 3 processed']


asyncio.run(main())
```

**Exception behavior:** by default, when one awaitable raises an exception, `gather()` propagates that exception to its caller. The other awaitables are **not automatically cancelled** and may continue running. With `return_exceptions=True`, exceptions are returned as result-list items instead of being propagated immediately.

```python
async def main():
    results = await asyncio.gather(
        good_operation(),
        operation_that_may_fail(),
        return_exceptions=True,
    )

    for result in results:
        if isinstance(result, BaseException):
            print("Operation failed:", repr(result))
        else:
            print("Operation succeeded:", result)
```

Use `return_exceptions=True` deliberately: it changes failures into result values, so callers must inspect the returned items. Also note that once `gather()` has propagated an exception and is itself marked done, calling `gather.cancel()` afterward will not cancel the other awaitables that are still running.

### 4.2 `asyncio.TaskGroup` — structured concurrency (Python 3.11+)

A `TaskGroup` gives a lexical scope to related tasks. Exiting the `async with` block waits for all tasks in the group to finish. If a child task raises an exception other than `CancelledError`, the group cancels the remaining tasks and reports failures as an `ExceptionGroup` (or `BaseExceptionGroup` when appropriate).

```python
import asyncio


async def process_data(item_id):
    await asyncio.sleep(1)
    return f"Data {item_id} processed"


async def main():
    async with asyncio.TaskGroup() as group:
        task_a = group.create_task(process_data(1))
        task_b = group.create_task(process_data(2))

    # The tasks are complete once the TaskGroup block exits successfully.
    print(task_a.result(), task_b.result())


asyncio.run(main())
```

If a task fails, the group normally raises an exception group when it exits. Handle selected exception types with `except*`:

```python
async def main():
    try:
        async with asyncio.TaskGroup() as group:
            group.create_task(operation_a())
            group.create_task(operation_b())
    except* ValueError as errors:
        print("One or more tasks raised ValueError:", errors)
```

A child ending with `CancelledError` is treated as cancellation, not as an ordinary failure that automatically cancels all siblings. `TaskGroup` and `gather()` have different failure semantics; neither should be described as a drop-in replacement for the other.

**Rule of thumb:** prefer `TaskGroup` for related child tasks when fail-fast cancellation and clear task ownership are desirable. Use `gather()` when its ordered result-list behavior or its exception-handling semantics suit the use case.

### 4.3 `asyncio.as_completed()`

`as_completed()` is useful when you want to handle whichever operation finishes first rather than waiting to process results in input order.

Compatible form for Python versions before 3.13 and after:

```python
async def main():
    tasks = [
        asyncio.create_task(fetch_one(url))
        for url in urls
    ]

    for next_completed in asyncio.as_completed(tasks):
        result = await next_completed
        print("Completed:", result)
```

From Python 3.13, `as_completed()` can also be used as an asynchronous iterator. When the inputs are Tasks or Futures, this form yields the original objects:

```python
async def main():
    async for completed_task in asyncio.as_completed(tasks):
        print(completed_task.result())
```

A timeout from `as_completed()` does not automatically cancel unfinished tasks. Keep track of those tasks and decide whether to cancel or continue them.

### 4.4 `asyncio.wait()`

`asyncio.wait()` returns two sets: `done` and `pending`. It is useful when you need lower-level control over completion conditions.

```python
async def main():
    tasks = {
        asyncio.create_task(fetch_one(url))
        for url in urls
    }

    done, pending = await asyncio.wait(
        tasks,
        timeout=5,
        return_when=asyncio.FIRST_COMPLETED,
    )

    for task in done:
        print("Finished:", task.result())

    print("Still pending:", len(pending))

    # Decide explicitly what should happen to pending tasks.
    for task in pending:
        task.cancel()
    await asyncio.gather(*pending, return_exceptions=True)
```

Important details:

- `asyncio.FIRST_COMPLETED` returns when at least one Future or Task has completed.
- `asyncio.ALL_COMPLETED` waits for all supplied Futures or Tasks.
- `asyncio.FIRST_EXCEPTION` returns when a Future finishes by raising an exception; if none raises, it behaves like `ALL_COMPLETED`.
- `wait()` does **not** cancel pending tasks when its timeout expires.
- Since Python 3.11, pass Tasks or Futures to `wait()`; passing coroutine objects directly is forbidden. Create Tasks first.

## 5. Real-World Example: Concurrent HTTP Requests

A normal synchronous HTTP call, such as `requests.get()`, blocks the thread while it waits for the response. Calling it directly inside an asyncio task blocks the event loop too. Use an asynchronous HTTP client such as `aiohttp` or the async client provided by `httpx`.

Install the example dependency with `pip install aiohttp`.

```python
import asyncio
import time

import aiohttp


async def fetch_status(session, url):
    async with session.get(url) as response:
        response.raise_for_status()
        return url, response.status


async def main():
    urls = [
        "https://www.python.org/",
        "https://www.github.com/",
        "https://www.example.com/",
    ]

    timeout = aiohttp.ClientTimeout(total=10)
    connector = aiohttp.TCPConnector(limit=10)

    async with aiohttp.ClientSession(
        timeout=timeout,
        connector=connector,
    ) as session:
        results = await asyncio.gather(
            *(fetch_status(session, url) for url in urls)
        )

    for url, status in results:
        print(url, status)


if __name__ == "__main__":
    start = time.perf_counter()
    asyncio.run(main())
    print(f"Total time: {time.perf_counter() - start:.2f} seconds")
```

Why this example is better suited to production than an unbounded demo:

- It reuses one `ClientSession`, which manages connection pooling.
- It sets a total request timeout.
- It configures a connection limit.
- `raise_for_status()` makes unsuccessful HTTP status codes raise an exception instead of silently treating them as success.
- The `async with` block closes the session even when an error occurs.

For a very large input, do not blindly create one Task for every URL. A connection limit controls active connections, but a list of millions of Tasks can still consume substantial memory. Use bounded batches or a bounded `asyncio.Queue` with a fixed number of workers (see Section 7).

## 6. Timeouts and Cancellation

Timeouts and cancellation are closely related: asyncio commonly enforces timeouts by cancelling a task or the operation it is waiting for.

### 6.1 `asyncio.wait_for()`

`wait_for()` waits for an awaitable up to the specified timeout. On timeout it cancels the awaited operation and raises `TimeoutError` (called `asyncio.TimeoutError` in older Python versions; since Python 3.11, `asyncio.TimeoutError` is an alias of the built-in `TimeoutError`).

```python
import asyncio


async def main():
    try:
        result = await asyncio.wait_for(fetch_data(), timeout=5)
    except TimeoutError:
        print("The operation timed out")


asyncio.run(main())
```

`wait_for()` waits for cancellation of the underlying operation to finish. Therefore, total elapsed time can exceed the timeout when cancellation cleanup takes time. To avoid cancelling the underlying Task, `asyncio.shield()` can be used when that behavior is genuinely required.

### 6.2 `asyncio.timeout()` (Python 3.11+)

The context-manager form is often convenient when one timeout should cover a block containing several awaits:

```python
import asyncio


async def main():
    try:
        async with asyncio.timeout(5):
            first = await fetch_first()
            second = await fetch_second(first)
    except TimeoutError:
        print("The overall operation timed out")


asyncio.run(main())
```

Catch `TimeoutError` **outside** the `async with` block. The timeout context uses cancellation internally and converts its own cancellation into `TimeoutError` when the block exits.

### 6.3 Cancellation

Calling `task.cancel()` requests cancellation. `asyncio.CancelledError` is injected into the coroutine at the next opportunity the task gets to run; cancellation is cooperative, not a forced termination of arbitrary Python code.

Use `finally` for cleanup. When explicitly catching `CancelledError`, normally re-raise it after cleanup so cancellation can propagate to the caller and structured-concurrency features can work correctly.

```python
import asyncio


async def worker():
    try:
        await asyncio.sleep(30)
    except asyncio.CancelledError:
        print("Cleaning up before exit")
        # Release resources here if needed.
        raise  # Preserve cancellation


async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(1)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Worker was cancelled")


asyncio.run(main())
```

`asyncio.CancelledError` has directly inherited from `BaseException` since Python 3.8, not from `Exception`. This prevents many broad `except Exception:` handlers from accidentally swallowing cancellation. Do not routinely suppress `CancelledError`; `TaskGroup` and `asyncio.timeout()` rely on cancellation internally.

## 7. Synchronization, Backpressure, and Throttling

Launching thousands of operations at once can overwhelm remote services, exhaust sockets or file descriptors, create memory pressure, or trigger rate limits. Limit concurrency and apply backpressure.

### 7.1 `asyncio.Semaphore`: limit in-flight work

A semaphore allows at most a configured number of tasks into a critical section at once.

```python
import asyncio


async def limited_fetch(session, url, semaphore):
    async with semaphore:
        return await fetch_status(session, url)


async def main():
    semaphore = asyncio.Semaphore(10)  # At most 10 calls inside the section

    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(
            *(
                limited_fetch(session, url, semaphore)
                for url in urls
            )
        )
```

The semaphore limits the number of operations inside the `async with` block. It does **not** prevent the program from creating a very large number of waiting Tasks. For huge inputs, a bounded queue and fixed worker pool can control both active work and the amount of queued work.

### 7.2 `asyncio.Lock`: protect shared state

A Lock ensures only one asyncio task at a time enters a protected section. It is useful when a logical operation must remain exclusive across one or more `await` points.

```python
import asyncio

shared_value = 0


async def update_shared_value(lock, persist_value):
    global shared_value
    async with lock:
        current = shared_value
        await persist_value(current + 1)
        shared_value = current + 1


async def main(persist_value):
    lock = asyncio.Lock()
    await update_shared_value(lock, persist_value)
```

If the protected code contains no `await`, tasks on the same event loop generally cannot interleave in the middle of that synchronous section. A lock is not a general-purpose fix for every shared variable, and `asyncio.Lock` is **not thread-safe**. Use `threading.Lock` or another thread-safe primitive when coordinating OS threads.

### 7.3 `asyncio.Queue`: producer-consumer with backpressure

An `asyncio.Queue(maxsize=N)` is useful for distributing work to a fixed number of consumers. When the queue is full, `await q.put(item)` waits until space becomes available. This naturally slows the producer and provides backpressure.

The following Python 3.11+ example uses sentinel values to tell consumers when to stop:

```python
import asyncio


async def producer(queue, worker_count):
    for item in range(5):
        await queue.put(item)

    # One stop signal for each consumer.
    for _ in range(worker_count):
        await queue.put(None)


async def consumer(queue):
    while True:
        item = await queue.get()
        try:
            if item is None:
                return
            print(f"Consumed {item}")
            await asyncio.sleep(0.1)  # Represents asynchronous processing
        finally:
            queue.task_done()


async def main():
    worker_count = 2
    queue = asyncio.Queue(maxsize=3)

    async with asyncio.TaskGroup() as group:
        for _ in range(worker_count):
            group.create_task(consumer(queue))

        await producer(queue, worker_count)
        await queue.join()  # Includes data items and stop signals


asyncio.run(main())
```

Call `task_done()` exactly once for every successful `get()`. `queue.join()` waits until the unfinished-item count returns to zero; it does not independently prove that business processing succeeded. Handle and report processing failures separately.

`asyncio.Queue` is designed for tasks on an event loop and is **not thread-safe**. Use `queue.Queue` for communication between ordinary OS threads, or use thread-safe event-loop APIs such as `asyncio.run_coroutine_threadsafe()` or `loop.call_soon_threadsafe()` when a thread needs to submit work to an event loop.

## 8. Async Iterators, Generators, and Context Managers

Async syntax also supports objects whose iteration or resource-management steps may require waiting.

- **`async with`** uses `__aenter__()` and `__aexit__()`, which may perform asynchronous setup and cleanup (for example, acquiring or closing a connection).
- **`async for`** consumes an asynchronous iterator through `__aiter__()` and `__anext__()`.
- An **async generator** is an `async def` function that contains `yield`. It can produce values over time and suspend between them.

```python
import asyncio


async def ticker(count):
    for value in range(count):
        await asyncio.sleep(0.1)
        yield value


async def main():
    async for value in ticker(3):
        print(value)  # 0, 1, 2


asyncio.run(main())
```

Use `async with` for asynchronous resources that provide an async context-manager interface. This makes cleanup reliable on normal exit and exceptional exit.

## 9. Dealing with Blocking and CPU-Bound Code

A blocking function called directly from an asyncio task blocks the event-loop thread. Move blocking work to an executor when you cannot replace it with a native asynchronous API.

### 9.1 Blocking I/O: `asyncio.to_thread()` (Python 3.9+)

`to_thread()` runs a normal synchronous function in a worker thread and lets the event loop continue while that function runs.

```python
import asyncio
import time


def blocking_io():
    time.sleep(2)
    return "done"


async def main():
    result = await asyncio.to_thread(blocking_io)
    print(result)


asyncio.run(main())
```

This is useful for blocking I/O libraries that do not offer an async API. `asyncio.to_thread()` uses the default thread pool. It does not turn a blocking function into a truly asynchronous implementation. Cancelling the coroutine that is awaiting `to_thread()` does not forcibly stop a synchronous function that is already running in its worker thread; design long-running functions with their own cooperative stop mechanism when necessary.

### 9.2 `run_in_executor()`

`loop.run_in_executor()` can use the default thread pool or an explicitly supplied executor.

```python
import asyncio
import time


def blocking_io():
    time.sleep(2)
    return "done"


async def main():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, blocking_io)
    print(result)


asyncio.run(main())
```

Passing `None` selects the loop's default thread-pool executor. It is not a process pool.

### 9.3 CPU-bound Python work: `ProcessPoolExecutor`

For CPU-heavy pure-Python work, a process pool can use multiple processes and avoid the usual CPython GIL limitation on parallel execution of Python bytecode.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor


def cpu_heavy(number):
    return sum(i * i for i in range(number))


async def main():
    loop = asyncio.get_running_loop()

    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 1_000_000)
        print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

For process pools, the submitted function and its arguments generally need to be picklable. On platforms that use process spawning, keep process creation behind the `if __name__ == "__main__":` guard. Processes add startup, memory, and data-serialization overhead, so benchmark before choosing them.

**GIL nuance:** threads are still useful for I/O-bound work. In standard GIL-enabled CPython, threads usually do not execute CPU-bound Python bytecode in parallel. Some native extensions release the GIL while doing their work, and free-threaded Python builds have different behavior; do not treat “threads can never run CPU work in parallel” as a universal rule.

## 10. Async vs. Synchronous, Threading, and Multiprocessing

| Model | Best use case | Main trade-off |
|---|---|---|
| **Synchronous** | Simple scripts, sequential workflows, or low-concurrency tasks. | A thread waits while a blocking operation runs. |
| **Threading** | Concurrent blocking I/O when libraries are synchronous; integrating with thread-based APIs. | Thread coordination and memory overhead; GIL limits CPU-bound Python parallelism in standard CPython. |
| **Multiprocessing** | CPU-bound Python work that benefits from multiple processes. | Process startup, memory use, serialization, and inter-process communication costs. |
| **`asyncio`** | Many concurrent I/O operations when the libraries support async I/O. | Requires async-compatible APIs; CPU-bound or blocking code can stall the event loop. |

### Function coloring (“async is viral”)

Async code tends to spread through a call chain: an async caller that needs the result of an async operation must `await` it or schedule it. This often means that database, HTTP, and messaging libraries on a performance-sensitive path need async-compatible interfaces.

The statement “an `async def` can only be called from another `async def`” is **not correct**. A synchronous entry point can call `asyncio.run()`, and frameworks or event loops can schedule coroutines. The restriction is that the `await` expression itself is normally used inside an async function (or an environment that supports top-level await).

## 11. Production Best Practices

1. **Do not block the event loop.** Avoid `time.sleep()`, synchronous HTTP/database calls, and long CPU loops inside async tasks. Prefer async libraries, `asyncio.to_thread()`, or a process pool where appropriate.
2. **Set timeouts.** Bound network calls and other operations that can wait indefinitely. Choose timeout values based on the operation and service-level requirements.
3. **Bound concurrency and queue size.** Use semaphores for in-flight limits and bounded queues or batches for very large workloads.
4. **Reuse clients and connections.** Keep HTTP sessions or database connection pools alive for the intended scope instead of repeatedly opening new connections.
5. **Use structured concurrency where practical.** `TaskGroup` makes the lifetime and failure handling of related tasks explicit.
6. **Keep references to background Tasks.** The event loop keeps weak references to Tasks; retain strong references to fire-and-forget tasks and arrange to inspect their results or exceptions. Prefer `TaskGroup` when tasks belong to a lexical scope.
7. **Handle cancellation correctly.** Use `try/finally` for cleanup. When catching `CancelledError`, normally re-raise it after cleanup.
8. **Use async synchronization primitives only in their intended context.** `asyncio.Lock`, `Semaphore`, and `Queue` are not thread-safe.
9. **Do not call `asyncio.run()` inside a running event loop.** In Jupyter or an async framework, await the coroutine from the existing loop.
10. **Measure before optimizing.** Async code has scheduling and coordination overhead. It helps most when many operations spend time waiting on I/O.

## 12. Common Mistakes and Interview Gotchas

| Mistake or misconception | Correct understanding |
|---|---|
| Calling an async function runs it immediately. | Calling an `async def` function creates a coroutine object; it must be awaited or scheduled. |
| `await` always makes work concurrent. | `await` waits for an awaitable. Concurrency requires other work to be scheduled and able to run while the current task is suspended. |
| `asyncio` makes CPU-bound code faster. | A single event loop does not provide CPU parallelism. Use an appropriate thread/process/native-code approach. |
| `gather()` cancels every sibling if one fails. | By default it propagates the first exception but does not automatically cancel the other awaitables. |
| `TaskGroup` and `gather()` have identical failure behavior. | `TaskGroup` cancels remaining children on an ordinary child failure and reports failures as an exception group. |
| `wait()` cancels pending tasks after its timeout. | It returns pending tasks without cancelling them. Cancel and await them explicitly when needed. |
| A semaphore prevents creation of too many Tasks. | It limits entry into the protected section, not the number of waiting Tasks already created. Use a bounded queue or batches for huge inputs. |
| Cancellation instantly kills a coroutine. | Cancellation is delivered cooperatively at the next opportunity. The coroutine can run cleanup and can even suppress cancellation (usually a mistake). |
| `asyncio.Queue` is thread-safe. | It is intended for communication among tasks on an event loop, not ordinary OS threads. |
| Async tasks are automatically retained forever. | Keep strong references to detached Tasks and observe their outcomes. |

## 13. Quick API Reference

| API | Purpose | Important note |
|---|---|---|
| `asyncio.run(main())` | Run the top-level async program. | Cannot be called inside a running loop in the same thread. |
| `asyncio.create_task(coro)` | Schedule a coroutine as a Task. | Retain a reference unless task lifetime is managed by a `TaskGroup`. |
| `asyncio.gather(*aws)` | Run awaitables concurrently and collect ordered results. | Default exception behavior does not cancel siblings. |
| `asyncio.TaskGroup()` | Manage a related set of child Tasks. | Python 3.11+; ordinary child failure cancels remaining children. |
| `asyncio.as_completed(aws)` | Process awaitables in completion order. | Async-iterator form is Python 3.13+; regular iterator form works on earlier versions. |
| `asyncio.wait(tasks)` | Wait for configurable completion conditions. | Python 3.11+ requires Tasks/Futures, not raw coroutines. |
| `asyncio.wait_for(aw, timeout)` | Apply a timeout to one awaitable. | Cancels the awaited operation on timeout unless shielded. |
| `asyncio.timeout(seconds)` | Apply a timeout to a block. | Python 3.11+; catch `TimeoutError` outside the context. |
| `asyncio.Semaphore(n)` | Limit concurrent entry to a section. | Does not limit how many Tasks have been created. |
| `asyncio.Lock()` | Protect shared state across await points. | Not thread-safe. |
| `asyncio.Queue(maxsize=n)` | Coordinate producer-consumer work. | A bounded queue applies backpressure; not thread-safe. |
| `asyncio.to_thread(func, ...)` | Run a synchronous function in a worker thread. | Python 3.9+; primarily useful for blocking I/O. |
| `loop.run_in_executor(executor, func, ...)` | Run synchronous work in an executor. | `None` uses the default thread pool; pass a process pool explicitly for CPU-bound work. |

## 14. Official References

- [Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html)
- [Runners](https://docs.python.org/3/library/asyncio-runner.html)
- [Synchronization Primitives](https://docs.python.org/3/library/asyncio-sync.html)
- [Queues](https://docs.python.org/3/library/asyncio-queue.html)
- [Developing with asyncio](https://docs.python.org/3/library/asyncio-dev.html)

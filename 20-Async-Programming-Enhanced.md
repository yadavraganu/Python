# Part I — Build the mental model

## 1. What problem does asynchronous programming solve?

Imagine a program has to call 100 web APIs. A typical request spends most of its time waiting for a remote server. During that wait, the program is not doing useful CPU work for that request.

There are three broad ways to handle this:

| Approach | How it works | Good fit | Main cost / limitation |
|---|---|---|---|
| Synchronous code | Perform one operation, wait for it, then start the next. | Small scripts and simple sequential work. | Waiting operations happen one after another. |
| Threads | Run multiple paths of execution in OS threads. | Concurrent use of blocking I/O libraries. | Thread coordination, memory overhead, and shared-state complexity. |
| `asyncio` | One event loop coordinates many tasks; tasks yield while waiting. | Many concurrent network or other asynchronous I/O operations. | Requires non-blocking/async-compatible operations; CPU work can stall the loop. |
| Processes | Run work in separate OS processes. | CPU-heavy work that benefits from multiple CPU cores. | Startup, memory, serialization, and communication overhead. |

**I/O-bound work** spends much of its time waiting for input/output: network requests, database responses, sockets, or asynchronous file/network services.

**CPU-bound work** spends most of its time calculating: image transformations, compression, large mathematical loops, or model inference implemented in CPU code.

`asyncio` is primarily a tool for I/O concurrency. Declaring a CPU-heavy function with `async def` does not make its calculations run in parallel.

## 2. The event loop: the central coordinator

An **event loop** is the scheduler that runs asyncio tasks and callbacks and coordinates non-blocking I/O. A loop runs in one OS thread. In that thread, it runs one task at a time. When that task suspends at an `await`, the loop can run another ready task.

A useful mental picture:

```text
Task A: start request ---- waiting for network -------- resume ---- finish
Task B:       start timer ---- waiting ---- resume ---- finish
Task C:             start DB query -------- waiting -------- resume
                         ^                 ^
                         |                 |
                  event loop runs another ready task
```

The tasks are **concurrent** because their work overlaps in time. They are not necessarily executing Python instructions at the exact same instant.

### What does `await` really do?

`await` means: “I need the result of this awaitable before continuing.” If the awaited operation is not ready, the current coroutine suspends and gives the event loop an opportunity to run other work.

Important: `await` does **not** guarantee a context switch every time. If the awaited object is already complete, execution may continue without suspending.

### Cooperative multitasking

Asyncio uses cooperative scheduling:

- A task runs ordinary Python code until it completes or suspends.
- The event loop does not normally interrupt a task halfway through a Python statement.
- `await` points are opportunities for the task to suspend.
- A long CPU loop or blocking function can prevent every other task on that event loop from running.

This is why `time.sleep(2)` is dangerous inside an async task: it blocks the event-loop thread. Use `await asyncio.sleep(2)` for an asynchronous delay.

## 3. The four words you must understand: coroutine function, coroutine, Task, Future

These concepts are related but are not interchangeable.

| Term | What it is | Example |
|---|---|---|
| Coroutine function | A function declared using `async def`. | `async def fetch(): ...` |
| Coroutine object | The object created when you call a coroutine function. It has not necessarily started running. | `coro = fetch()` |
| Task | An asyncio object that schedules a coroutine to run on an event loop and tracks its result/state. | `task = asyncio.create_task(fetch())` |
| Future | A low-level placeholder for a result that will be supplied later. Tasks are Future-like and `asyncio.Task` is a subclass of `asyncio.Future`. | `future = loop.create_future()` |

A **coroutine object is not a Task**. Calling an `async def` function does not automatically schedule its body.

### Example 1 — Create a coroutine and run it

```python
import asyncio


async def greet():
    print("Hello")
    await asyncio.sleep(0.2)
    print("Goodbye")


async def main():
    coroutine_object = greet()             # 1. Create a coroutine object.
    print(type(coroutine_object).__name__) # 2. Show its type; greet() has not run yet.
    await coroutine_object                # 3. Run it here and wait for it to finish.


if __name__ == "__main__":
    asyncio.run(main())                    # 4. Start and manage the event loop.
```

**Line-by-line explanation**

1. `greet()` creates a coroutine object. It does not print `Hello` yet.
2. `type(...).__name__` displays the object's type, usually `coroutine`.
3. `await coroutine_object` starts/continues that coroutine as part of `main` and waits for its completion. While `asyncio.sleep()` is pending, the event loop can run other tasks.
4. `asyncio.run(main())` creates an event loop, runs `main()` to completion, and cleans up the loop.

**Expected output** (the type line may be displayed exactly as `coroutine`):

```text
coroutine
Hello
Goodbye
```

### Example 2 — The classic missing-`await` bug

```python
import asyncio


async def save_data():
    print("Saving data")
    await asyncio.sleep(0.1)
    print("Data saved")


async def main():
    save_data()          # BUG: creates a coroutine object but does not run it.
    await save_data()    # Correct: await the coroutine.


asyncio.run(main())
```

The first call to `save_data()` creates a coroutine object and discards it. It does not print `Saving data` or perform the sleep. Python will normally emit a `RuntimeWarning` similar to `coroutine 'save_data' was never awaited` when the un-awaited object is finalized. The second call is awaited and does run.

**Rule:** an async operation must be awaited, scheduled as a Task, or deliberately passed to an API that accepts an awaitable. Do not create a coroutine and then forget it.

## 4. The basic syntax

| Syntax / function | Meaning |
|---|---|
| `async def` | Declares a coroutine function. |
| `await` | Waits for an awaitable from inside async code. |
| `asyncio.run(coro)` | Runs the top-level coroutine and manages the event loop. |
| `asyncio.sleep(seconds)` | Asynchronous timer; yields while waiting. |
| `async with` | Uses an asynchronous context manager. |
| `async for` | Iterates over an asynchronous iterator. |

### Example 3 — A minimal asyncio program

```python
import asyncio


async def main():
    print("Step 1: before waiting")
    await asyncio.sleep(1)  # Pause this task, not the whole event-loop thread.
    print("Step 2: after waiting")


asyncio.run(main())
```

- `import asyncio` loads the standard-library async framework.
- `async def main()` declares the entry-point coroutine.
- The first `print` runs immediately when `main` starts.
- `await asyncio.sleep(1)` suspends this task for about one second. The event loop remains free to run other tasks.
- The final `print` runs when the task resumes.
- `asyncio.run(main())` owns the event loop lifecycle for this simple program.

**Important:** do not call `asyncio.run()` from a thread that already has a running event loop. In Jupyter notebooks, many web frameworks, and async application code, use `await main()` from the existing async context instead.

# Part II — Scheduling and running tasks

## 5. Sequential awaiting versus concurrent scheduling

This distinction is the foundation of `asyncio`.

- **Sequential:** await operation A; after A finishes, await operation B.
- **Concurrent:** schedule A and B before waiting for their results. The event loop can let both make progress while they wait.

### Example 4 — Same work, sequential then concurrent

```python
import asyncio
import time


async def work(name):
    print(f"{name}: started")
    await asyncio.sleep(1)
    print(f"{name}: finished")
    return name


async def main():
    start = time.perf_counter()

    first = await work("A")  # A must finish before the next line begins.
    second = await work("B") # B starts only after A has finished.
    print("Sequential results:", first, second)
    print("Sequential elapsed:", round(time.perf_counter() - start, 2))

    start = time.perf_counter()
    task_a = asyncio.create_task(work("C")) # Schedule C as a Task.
    task_b = asyncio.create_task(work("D")) # Schedule D before awaiting either.
    result_c = await task_a                  # Wait for C's result.
    result_d = await task_b                  # D may already have finished.
    print("Concurrent results:", result_c, result_d)
    print("Concurrent elapsed:", round(time.perf_counter() - start, 2))


asyncio.run(main())
```

**What each important line does**

- `time.perf_counter()` reads a high-resolution monotonic clock for measuring elapsed time.
- `await work("A")` runs A as part of the current call chain. It must complete before the next statement in `main` executes.
- The same is true for B, so the first section takes roughly two seconds.
- `asyncio.create_task(...)` wraps the coroutine in a Task and schedules it on the running loop.
- Both Tasks are created before we await either result. At the next opportunity for the event loop to run them, both begin and can overlap their one-second waits.
- Awaiting `task_a` does not mean `task_b` is paused. It is independently scheduled and can finish while `main` is waiting for A.
- The concurrent section takes roughly one second, not two.

**Do not misread the last two `await` statements:** they are written one after another, but the Tasks were already scheduled together. That is why their waiting periods overlap.

## 6. `asyncio.create_task()` — schedule a coroutine

`asyncio.create_task(coro, name=...)` creates and schedules a Task on the currently running event loop. It returns the Task object so you can later await it, cancel it, inspect it, or get its result.

A Task is useful when you want work to begin before you are ready to wait for its result.

### Example 5 — Observe task scheduling

```python
import asyncio


async def child():
    print("child: running")
    await asyncio.sleep(0.1)
    print("child: finished")


async def main():
    print("main: before create_task")
    task = asyncio.create_task(child(), name="child-task")
    print("main: after create_task")
    await asyncio.sleep(0)  # Yield once so the event loop can run ready tasks.
    print("main: after yielding")
    await task              # Ensure the child is finished before main exits.


asyncio.run(main())
```

**Typical output** with the default task factory:

```text
main: before create_task
main: after create_task
child: running
main: after yielding
child: finished
```

- The first print happens before task creation.
- `create_task()` schedules the child; under the normal task factory, the child does not run in the middle of the current synchronous block.
- The next print therefore appears before `child: running`.
- `await asyncio.sleep(0)` suspends `main` and gives the event loop a chance to run the child.
- The child prints `running`, then suspends for 0.1 seconds.
- `main` resumes and prints `after yielding`.
- `await task` waits until the child finishes.

This example demonstrates the default scheduling behavior. Advanced custom task factories can alter when a coroutine begins, so do not use print ordering as a substitute for proper task coordination.

### Keep a reference to detached Tasks

If you create a Task and do not await it, retain a reference and arrange to handle its exception. Prefer `TaskGroup` for related work. A detached “fire-and-forget” Task without an owner can finish with an exception that nobody retrieves.

## 7. Task state and useful Task methods

A Task has a lifecycle:

```text
created/scheduled -> running -> waiting -> running -> finished
                                                   \-> cancelled
                                                   \-> failed
```

Common methods and properties:

| Method / property | Meaning |
|---|---|
| `task.done()` | `True` when the Task finished, whether by returning, raising, or cancellation. |
| `task.result()` | Return the result; raises `InvalidStateError` if not done, re-raises the task exception if failed, and raises `CancelledError` if cancelled. |
| `task.exception()` | Return the exception, or `None` if successful; raises if the task is not done, and raises `CancelledError` if cancelled. |
| `task.cancel()` | Request cancellation. It does not forcibly kill Python code. |
| `task.cancelled()` | `True` if the Task finished due to cancellation. |
| `task.get_name()` / `task.set_name()` | Read or change the diagnostic task name. |

### Example 6 — Inspect a Task

```python
import asyncio


async def calculate():
    await asyncio.sleep(0.05)
    return 42


async def main():
    task = asyncio.create_task(calculate(), name="calculate-42")
    print("Name:", task.get_name())
    print("Done immediately:", task.done())

    result = await task
    print("Done after await:", task.done())
    print("Result:", task.result())
    print("Exception:", task.exception())


asyncio.run(main())
```

- `create_task` starts the calculation as a scheduled Task.
- `get_name()` returns the diagnostic name.
- `done()` is normally `False` immediately after creation because the short sleep has not completed.
- `await task` waits for completion and returns the coroutine's result.
- `task.result()` is now safe because the Task is complete.
- `task.exception()` returns `None` because no exception occurred.

Do not call `task.result()` on a pending Task. Await it first or confirm that it is done.

## 8. `asyncio.gather()` — run a set of awaitables together

`asyncio.gather(*aws)` schedules the supplied coroutine objects/awaitables concurrently and returns a list of results in **input order**, not completion order.

Use it when you naturally want one result list for a known set of operations.

### Example 7 — Basic `gather()`

```python
import asyncio


async def fetch_record(record_id, delay):
    await asyncio.sleep(delay)
    return {"id": record_id, "status": "ok"}


async def main():
    results = await asyncio.gather(
        fetch_record(1, 0.3),
        fetch_record(2, 0.1),
        fetch_record(3, 0.2),
    )
    print(results)


asyncio.run(main())
```

The second operation finishes first because it sleeps for only 0.1 seconds. However, the output list is still ordered by the inputs: record 1, then 2, then 3.

- Each `fetch_record(...)` call creates a coroutine object.
- `gather` schedules the awaitables concurrently.
- `await asyncio.gather(...)` waits until all supplied awaitables have completed successfully, or until its default exception behavior propagates a failure.

### `gather()` exception behavior

By default, when one awaitable raises an exception, `gather()` propagates that exception to the caller. It does **not automatically cancel the other submitted awaitables**; the others may continue running.

### Example 8 — One `gather()` operation fails, another continues

```python
import asyncio


async def slow_job():
    try:
        await asyncio.sleep(0.2)
        print("slow job completed")
        return "slow result"
    except asyncio.CancelledError:
        print("slow job was cancelled")
        raise


async def failing_job():
    await asyncio.sleep(0.02)
    raise ValueError("database unavailable")


async def main():
    slow_task = asyncio.create_task(slow_job())
    try:
        await asyncio.gather(slow_task, failing_job())
    except ValueError as error:
        print("gather propagated:", error)

    # The slow task is still running; we own it and wait for it explicitly.
    slow_result = await slow_task
    print("remaining task result:", slow_result)


asyncio.run(main())
```

- The slow Task is explicitly created and retained in `slow_task`.
- `gather` also schedules `failing_job()`.
- The failing job raises `ValueError` first, so the `await gather(...)` expression raises it into the `except` block.
- The slow Task was not automatically cancelled. We still own it, so we await it and observe its result.

**Important:** after `gather()` has already propagated an exception and become done, calling `gather.cancel()` is not a reliable way to cancel remaining child work. Retain the child Tasks if you need explicit control over them.

Two additional cancellation rules matter:

- If the **`gather()` call itself** is cancelled, unfinished awaitables submitted to it are cancelled.
- If **one child Task/Future** supplied to `gather()` is cancelled, `gather()` treats that child as having raised `CancelledError`; it does not automatically cancel the other children.

### `return_exceptions=True`

With `return_exceptions=True`, exceptions are returned as items in the results list instead of being immediately propagated by `gather`.

```python
import asyncio


async def succeeds():
    return "success"


async def fails():
    raise ValueError("bad input")


async def main():
    results = await asyncio.gather(
        succeeds(),
        fails(),
        return_exceptions=True,
    )

    for index, result in enumerate(results):
        if isinstance(result, BaseException):
            print(index, "failed:", repr(result))
        else:
            print(index, "succeeded:", result)


asyncio.run(main())
```

Use this option only when you intend to inspect each result. An exception in the list is a value that your code must handle; it is not a successful business result.

## 9. `asyncio.TaskGroup` — structured concurrency (Python 3.11+)

A `TaskGroup` manages a group of related child Tasks inside an `async with` block.

Its key guarantees:

- Exiting the block waits for the child Tasks to finish.
- If a child raises an exception other than `CancelledError`, the group requests cancellation of the remaining child Tasks.
- Failures are normally raised together as an `ExceptionGroup` or `BaseExceptionGroup` when the group exits.
- A child ending in `CancelledError` is treated as cancellation, not as an ordinary child failure that automatically cancels siblings.

This is called **structured concurrency**: the lifetime of child work is bounded by a clear parent scope.

### Example 9 — Successful `TaskGroup`

```python
import asyncio


async def process(item):
    await asyncio.sleep(0.1)
    return f"processed-{item}"


async def main():
    async with asyncio.TaskGroup() as group:
        task_a = group.create_task(process("A"))
        task_b = group.create_task(process("B"))
        task_c = group.create_task(process("C"))

    # Reaching this line means every child in the group has finished.
    print(task_a.result(), task_b.result(), task_c.result())


asyncio.run(main())
```

- `async with asyncio.TaskGroup()` creates a task scope.
- `group.create_task(...)` schedules a child Task owned by the group.
- We store the Task references so we can inspect their results after the group exits.
- Exiting the `async with` block waits for all children.
- `task.result()` is safe after successful group exit because the Task is complete.

### Example 10 — A TaskGroup failure cancels siblings

```python
import asyncio


async def worker(name, delay, should_fail=False):
    try:
        print(name, "started")
        await asyncio.sleep(delay)
        if should_fail:
            raise ValueError(f"{name} failed")
        print(name, "finished")
        return name
    except asyncio.CancelledError:
        print(name, "received cancellation; cleaning up")
        raise


async def main():
    try:
        async with asyncio.TaskGroup() as group:
            group.create_task(worker("slow-A", 1.0))
            group.create_task(worker("fast-B", 0.05, should_fail=True))
            group.create_task(worker("slow-C", 1.0))
    except* ValueError as errors:
        print("Handled ValueError group:", errors)


asyncio.run(main())
```

- All three child Tasks are scheduled by the group.
- `fast-B` raises `ValueError` after 0.05 seconds.
- The TaskGroup requests cancellation of the unfinished siblings.
- The siblings receive `CancelledError` at their next suspension point, execute their cleanup path, then re-raise cancellation.
- When the group exits, it raises an exception group containing the non-cancellation failure(s).
- `except* ValueError` handles the `ValueError` portion of an exception group. Any unmatched exception types would continue propagating.

**Which should you use?** Prefer `TaskGroup` for related child work whose lifetimes and failure handling should be owned by one scope. Use `gather` when the ordered results list or its particular failure behavior is what you need. They are not exact replacements for each other.

## 10. `asyncio.as_completed()` — process work in completion order

Use this when the fastest completed operation is useful before slower operations finish.

### Example 11 — Print each result as soon as it completes

```python
import asyncio


async def request(name, delay):
    await asyncio.sleep(delay)
    return f"response-{name}"


async def main():
    tasks = [
        asyncio.create_task(request("A", 0.3)),
        asyncio.create_task(request("B", 0.1)),
        asyncio.create_task(request("C", 0.2)),
    ]

    for completed in asyncio.as_completed(tasks):
        result = await completed
        print(result)


asyncio.run(main())
```

The expected completion order is B, C, A because their delays are 0.1, 0.2, and 0.3 seconds.

- We create and retain Tasks explicitly.
- `as_completed(tasks)` provides awaitables in the order each input completes.
- `await completed` obtains that operation's result.

**Version note:** the regular `for` form shown here works on Python 3.11+. Starting in Python 3.13, `as_completed()` can also be used with `async for`; when its inputs are Tasks/Futures, asynchronous iteration yields the original Task/Future objects. Use the regular form when writing code that must also support 3.11 and 3.12.

## 11. `asyncio.wait()` — control over done and pending Tasks

`asyncio.wait()` returns two sets:

- `done`: Tasks/Futures that have completed by the return condition.
- `pending`: Tasks/Futures that have not completed yet.

The `return_when` argument controls the condition:

| Constant | Meaning |
|---|---|
| `asyncio.FIRST_COMPLETED` | Return when at least one item completes. |
| `asyncio.FIRST_EXCEPTION` | Return when a Task/Future raises; if none raises, wait until all complete. |
| `asyncio.ALL_COMPLETED` | Wait until all complete. This is the default. |

### Example 12 — Wait for the first result, then cancel the rest

```python
import asyncio


async def job(name, delay):
    await asyncio.sleep(delay)
    return f"{name} done"


async def main():
    tasks = {
        asyncio.create_task(job("A", 0.1)),
        asyncio.create_task(job("B", 0.5)),
        asyncio.create_task(job("C", 0.8)),
    }

    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED,
    )

    for task in done:
        print("First completed:", task.result())

    for task in pending:
        task.cancel()  # Request cancellation of work we no longer need.

    await asyncio.gather(*pending, return_exceptions=True)  # Wait for cleanup.


asyncio.run(main())
```

- A set of Tasks is passed to `wait`. Python 3.11+ does not permit raw coroutine objects here; create Tasks first.
- `FIRST_COMPLETED` returns after at least one Task completes.
- `task.result()` reads a completed Task's result. If the completed Task failed, this line would re-raise its exception, so production code may need `try/except` around it.
- The Tasks in `pending` are still running. `wait()` does not cancel them automatically.
- `task.cancel()` requests cancellation.
- `gather(..., return_exceptions=True)` waits for the cancelled Tasks to finish their cancellation cleanup without letting their `CancelledError` results escape as an exception from this cleanup call.

## 12. Choosing the right task coordination API

| API | Use it when | Result / behavior |
|---|---|---|
| `create_task()` | You want to schedule one coroutine and manage it directly. | One Task. |
| `gather()` | You want multiple operations and an input-ordered result list. | One list of results; default first-exception propagation does not cancel siblings. |
| `TaskGroup` | Related tasks should have one owner and fail together. | Waits for all; cancels remaining children on ordinary child failure; may raise an exception group. |
| `as_completed()` | You want to process results in completion order. | Iteration over completed awaitables. |
| `wait()` | You want the `done` and `pending` sets and custom return conditions. | `(done, pending)` sets; pending work is not automatically cancelled. |

# Part III — Failures, timeouts, and cancellation

## 13. Exceptions in asynchronous code

An exception raised inside a coroutine behaves like an exception in ordinary Python: if it is not handled inside the coroutine, it propagates to the code that awaits it.

### Example 13 — Catch an exception from an awaited coroutine

```python
import asyncio


async def read_user(user_id):
    await asyncio.sleep(0.05)
    if user_id < 0:
        raise ValueError("user_id must be non-negative")
    return {"id": user_id}


async def main():
    try:
        user = await read_user(-1)
        print(user)
    except ValueError as error:
        print("Could not read user:", error)


asyncio.run(main())
```

- `read_user(-1)` creates the coroutine.
- `await` runs it and waits for it.
- The coroutine raises `ValueError` after the sleep.
- The exception travels through the `await` expression to the `try/except` block in `main`.

## 14. `asyncio.CancelledError` — how cancellation works

Cancellation is a **request**, not a forceful kill.

Calling `task.cancel()` arranges for `asyncio.CancelledError` to be thrown into the coroutine when it next gets an opportunity to run. The coroutine may perform cleanup. In almost all cases, code that catches `CancelledError` should re-raise it after cleanup.

`asyncio.CancelledError` has inherited from `BaseException` (not `Exception`) since Python 3.8. A normal `except Exception:` handler therefore does not catch it.

### Example 14 — Cancel a running Task safely

```python
import asyncio


async def worker():
    try:
        print("worker: started")
        await asyncio.sleep(10)
        print("worker: finished normally")
    except asyncio.CancelledError:
        print("worker: cancellation received")
        raise  # Do not swallow cancellation.
    finally:
        print("worker: cleanup runs")


async def main():
    task = asyncio.create_task(worker(), name="background-worker")
    await asyncio.sleep(0.05)  # Let the worker start and reach its sleep.
    task.cancel()              # Request cancellation.

    try:
        await task             # Observe its final cancelled state.
    except asyncio.CancelledError:
        print("main: worker was cancelled")


asyncio.run(main())
```

- The worker begins and suspends in `asyncio.sleep(10)`.
- `main` sleeps briefly to let the worker start.
- `task.cancel()` requests cancellation.
- The worker receives `CancelledError` at its suspension point.
- The `except` block performs any cancellation-specific cleanup and re-raises.
- The `finally` block runs whether the worker succeeds, fails, or is cancelled.
- Awaiting the cancelled Task in `main` raises `CancelledError`, which `main` catches.

**Prefer `finally` for resource cleanup** when the cleanup should run regardless of success, failure, or cancellation. If cleanup itself requires awaiting, it can be done in `finally`, but design it carefully so cancellation does not leave resources half-closed.

## 15. `asyncio.wait_for()` — timeout one awaitable

`wait_for(aw, timeout=seconds)` waits up to the timeout. If the timeout expires, it cancels the awaited operation and raises `TimeoutError`.

### Example 15 — Timeout with `wait_for`

```python
import asyncio


async def slow_operation():
    await asyncio.sleep(2)
    return "finished"


async def main():
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=0.1)
        print(result)
    except TimeoutError:
        print("The operation exceeded 0.1 seconds")


asyncio.run(main())
```

- `slow_operation()` creates the coroutine to be controlled by the timeout.
- `wait_for` starts/waits for it and sets a deadline of 0.1 seconds.
- The operation is still sleeping when the deadline arrives.
- `wait_for` requests cancellation and raises `TimeoutError` after the cancellation process completes.

**Subtle point:** elapsed time can be longer than the specified timeout if the awaited operation takes time to handle cancellation and finish cleanup. A timeout is not a hard operating-system kill.

## 16. `asyncio.timeout()` — timeout a block (Python 3.11+)

Use `asyncio.timeout()` when one deadline should apply to several awaits in a block.

### Example 16 — A single time budget for multiple calls

```python
import asyncio


async def get_profile():
    await asyncio.sleep(0.04)
    return {"name": "Ada"}


async def get_orders():
    await asyncio.sleep(0.2)
    return [101, 102]


async def main():
    try:
        async with asyncio.timeout(0.1):
            profile = await get_profile()
            orders = await get_orders()
            print(profile, orders)
    except TimeoutError:
        print("The combined operation exceeded its time budget")


asyncio.run(main())
```

- `asyncio.timeout(0.1)` creates a timeout context for the block.
- `get_profile()` takes about 0.04 seconds.
- The remaining time budget is about 0.06 seconds when `get_orders()` begins.
- The timeout expires while waiting for orders, cancelling the current Task internally.
- The timeout context converts its own cancellation into `TimeoutError` as it exits.
- Catch `TimeoutError` **outside** the `async with` block.

For a nested operation with an absolute deadline, `asyncio.timeout_at(deadline)` is also available. Use `loop.time()` to calculate monotonic deadlines rather than wall-clock timestamps.

## 17. `asyncio.shield()` — protect an operation from caller cancellation

`asyncio.shield(aw)` prevents cancellation of the *outer waiter* from automatically cancelling the inner awaitable. It does not make the inner Task immune to direct cancellation, and it does not stop the outer caller from receiving cancellation.

Use shielding only when the inner operation genuinely must continue independently. Keep a strong reference to the inner Task and later await it or otherwise retrieve its result/exception.

### Example 17 — Timeout the wait but allow the Task to continue

```python
import asyncio


async def important_write():
    await asyncio.sleep(0.15)
    return "write completed"


async def main():
    write_task = asyncio.create_task(important_write())

    try:
        await asyncio.wait_for(asyncio.shield(write_task), timeout=0.03)
    except TimeoutError:
        print("Stopped waiting; the write Task was not cancelled")

    result = await write_task  # Still own it and retrieve its result.
    print(result)


asyncio.run(main())
```

- `important_write()` is scheduled as its own Task.
- `shield(write_task)` creates a protected awaitable around that Task.
- `wait_for` times out the shielded wait. It cancels the shield wrapper, not the underlying `write_task`.
- The original Task continues running.
- We later await `write_task` so its completion and result are observed.

Do not use `shield()` as a way to ignore cancellation during shutdown. It is a specific control over cancellation propagation, not a general reliability mechanism.

# Part IV — Synchronization, throttling, and work distribution

## 18. Why do we need synchronization in asyncio?

A single event loop runs one Task at a time, but tasks can still interleave at `await` points. Consider a read-modify-write sequence:

1. Task A reads a shared value.
2. Task A awaits a slow write.
3. Task B runs and updates the same value.
4. Task A resumes using stale information.

This is a logical race condition. Asyncio synchronization primitives coordinate Tasks, but they are **not thread-safe**. Use `threading` primitives for ordinary OS-thread synchronization.

The synchronization primitives are in the `asyncio` module:

- `Lock`
- `Event`
- `Condition`
- `Semaphore`
- `BoundedSemaphore`
- `Barrier`

Their methods generally do not accept a `timeout=` argument. Wrap an awaited method with `asyncio.wait_for()` or use a timeout context when needed.

## 19. `asyncio.Lock` — one Task at a time

A Lock provides exclusive access to a section of code. The preferred form is `async with lock:` because it releases the lock even when an exception occurs.

### Example 18 — Protect a shared counter

```python
import asyncio


async def increment(counter, lock):
    async with lock:                         # Acquire the lock; wait if another task owns it.
        old_value = counter["value"]         # Read the shared value while holding the lock.
        await asyncio.sleep(0)               # Simulate a suspension while holding the lock.
        counter["value"] = old_value + 1     # Update without another lock user interleaving.


async def main():
    counter = {"value": 0}                   # Mutable shared state.
    lock = asyncio.Lock()                    # One lock shared by all increment tasks.
    await asyncio.gather(*(increment(counter, lock) for _ in range(100)))
    print(counter["value"])                  # Always 100 in this example.


asyncio.run(main())
```

- `counter` is a mutable dictionary used as shared state.
- `asyncio.Lock()` creates one lock for the event loop.
- Every call to `increment` uses the same lock object.
- `async with lock` waits until the lock is available and acquires it.
- The read and update are protected by the same lock.
- The artificial `sleep(0)` demonstrates why the lock matters: it lets another Task become ready while this Task is inside the critical section.
- When the `async with` block ends, the lock is released automatically.
- `gather` schedules 100 increments concurrently and waits for all of them.

Do not hold a lock while doing slow network operations unless exclusivity truly needs to cover that wait. Keep critical sections as small as the correctness requirement allows.

## 20. `asyncio.Event` — broadcast that something is ready

An Event is a flag shared by Tasks:

- Initially it is unset (`False`).
- `await event.wait()` pauses until it is set.
- `event.set()` sets the flag and wakes waiting Tasks.
- `event.clear()` resets it.

### Example 19 — Wait until initialization is complete

```python
import asyncio


async def service(event):
    print("service: waiting for initialization")
    await event.wait()                       # Pause until another Task calls set().
    print("service: initialization complete")


async def main():
    initialized = asyncio.Event()             # Initially unset.
    service_task = asyncio.create_task(service(initialized))
    await asyncio.sleep(0.05)                 # Give service time to start waiting.
    print("main: initializing")
    initialized.set()                         # Wake every Task waiting on this Event.
    await service_task


asyncio.run(main())
```

This is useful for “wait until ready” signals. An Event does not count how many notifications occurred; it represents a set/unset condition.

## 21. `asyncio.Semaphore` — limit concurrent operations

A Semaphore has a counter. `async with semaphore` acquires one permit; leaving the block releases it. If no permit is available, the Task waits.

Use it to cap the number of in-flight operations, for example, at most five calls to a remote service at once.

### Example 20 — Allow at most two workers inside a section

```python
import asyncio


async def request(number, semaphore):
    async with semaphore:                    # Acquire one of the two permits.
        print("started", number)
        await asyncio.sleep(0.1)             # Simulate an in-flight network request.
        print("finished", number)
        return number


async def main():
    semaphore = asyncio.Semaphore(2)          # At most two Tasks enter the block at once.
    results = await asyncio.gather(
        *(request(number, semaphore) for number in range(5))
    )
    print("results:", results)


asyncio.run(main())
```

The five Tasks are all created/scheduled by `gather`, but only two can be inside the semaphore-protected section at once. The others wait for permits.

**Important distinction:** a semaphore limits active entries, not Task creation. If you create one million Tasks, a semaphore can still leave a very large number of Tasks waiting in memory. For very large input collections, use a bounded queue and a fixed number of workers (see Section 24).

## 22. `asyncio.BoundedSemaphore`

`BoundedSemaphore` behaves like `Semaphore`, but it raises `ValueError` if code releases more permits than were initially acquired. This helps detect accidental over-release bugs.

Use `Semaphore` when you need a simple concurrency limit. Use `BoundedSemaphore` when catching incorrect release counts is useful. Both are not thread-safe.

## 23. `asyncio.Condition` — wait for a state change while coordinating access

A Condition combines a Lock with the ability to wait until a predicate becomes true. The waiting Task releases the underlying lock while waiting and reacquires it before returning from `wait()`.

### Example 21 — Consumer waits until an item is available

```python
import asyncio


async def consumer(items, condition):
    async with condition:                       # Acquire the condition's lock.
        await condition.wait_for(lambda: bool(items))  # Release lock while waiting; re-check predicate.
        item = items.pop(0)                      # Predicate is true while lock is held again.
        print("consumed:", item)


async def producer(items, condition):
    await asyncio.sleep(0.05)                    # Simulate work before producing.
    async with condition:                        # Acquire the same condition lock.
        items.append("message-1")                # Change the shared state.
        condition.notify_all()                   # Wake Tasks waiting on the condition.


async def main():
    items = []
    condition = asyncio.Condition()
    await asyncio.gather(consumer(items, condition), producer(items, condition))


asyncio.run(main())
```

- The consumer enters the Condition and acquires its lock.
- `wait_for(predicate)` checks whether the list is non-empty.
- If the predicate is false, the condition releases the lock and suspends the consumer.
- The producer can now acquire the same lock, append an item, and notify waiters.
- The consumer wakes, reacquires the lock, re-checks the predicate, and then removes the item.

Always re-check the condition/predicate. Notifications indicate that state may have changed; they do not themselves prove the condition is true.

## 24. `asyncio.Barrier` — wait until a group has arrived

A Barrier releases a fixed number of participating Tasks once all have called `wait()`. It is useful when several concurrent jobs must reach the same phase before any proceed.

### Example 22 — Three Tasks meet at a barrier

```python
import asyncio


async def participant(name, barrier):
    print(name, "working")
    await asyncio.sleep(0.05)
    print(name, "waiting at barrier")
    await barrier.wait()                       # Wait until all three participants arrive.
    print(name, "passed barrier")


async def main():
    barrier = asyncio.Barrier(3)                # Exactly three parties are required.
    await asyncio.gather(
        participant("A", barrier),
        participant("B", barrier),
        participant("C", barrier),
    )


asyncio.run(main())
```

The order of the three `passed barrier` messages is not a business guarantee. The important guarantee is that no participant passes the barrier until all three have arrived.

## 25. `asyncio.Queue` — producer-consumer pattern and backpressure

A Queue transfers items between Tasks. It is often a better design than creating a Task for every item.

Useful methods:

| Method | Meaning |
|---|---|
| `await queue.put(item)` | Add an item; if a bounded queue is full, wait for space. |
| `await queue.get()` | Get the next item; if empty, wait for an item. |
| `queue.task_done()` | Tell the queue that one item returned by `get()` has been processed. |
| `await queue.join()` | Wait until every enqueued item has a matching `task_done()`. |
| `queue.qsize()` | Current number of items waiting in the queue. |
| `queue.get_nowait()` / `put_nowait()` | Non-waiting versions; raise `QueueEmpty` / `QueueFull` when unavailable. |

A queue with `maxsize=10` stores at most ten waiting items. When full, `put()` suspends the producer. This is **backpressure**: a slow consumer naturally slows the producer.

### Example 23 — Complete bounded worker-pool example

```python
import asyncio


async def producer(queue, worker_count):
    for item in range(6):
        await queue.put(item)                   # Wait if the bounded queue is full.
        print("produced:", item)

    for _ in range(worker_count):
        await queue.put(None)                   # One sentinel tells each worker to stop.


async def consumer(worker_name, queue):
    while True:
        item = await queue.get()                # Wait until an item is available.
        try:
            if item is None:                    # None is our agreed stop signal.
                print(worker_name, "stopping")
                return
            print(worker_name, "processing", item)
            await asyncio.sleep(0.05)            # Simulate asynchronous work.
        finally:
            queue.task_done()                   # Exactly once for every successful get().


async def main():
    worker_count = 2
    queue = asyncio.Queue(maxsize=2)             # At most two items wait in the queue.

    async with asyncio.TaskGroup() as group:
        for worker_number in range(worker_count):
            group.create_task(consumer(f"worker-{worker_number}", queue))

        await producer(queue, worker_count)      # Add jobs and then stop signals.
        await queue.join()                       # Wait until all jobs/signals are acknowledged.


asyncio.run(main())
```

**Line-by-line behavior**

- `worker_count = 2` means two consumers will process items.
- `Queue(maxsize=2)` allows only two waiting items. The producer is slowed when both slots are occupied.
- `TaskGroup` owns both consumer Tasks and will wait for them on exit.
- Each consumer waits at `queue.get()` when there is no item.
- `None` is a sentinel. It is not a special asyncio value; it is simply a convention chosen by this program.
- `try/finally` ensures `task_done()` is called even when a consumer returns for a sentinel or processing raises an exception.
- `task_done()` must be called exactly once for every successful `get()`.
- The producer adds six real items, then one sentinel per worker so both workers can stop.
- `queue.join()` waits until every item—including stop signals—has been acknowledged with `task_done()`.
- Exiting the TaskGroup waits for both workers to finish.

**Production note:** `queue.join()` proves that every dequeued item was acknowledged; it does not prove business processing succeeded. Decide how to record/retry/report failed jobs. In this example, an unhandled worker exception will cause the TaskGroup to cancel sibling tasks and propagate the failure.

**Python 3.13+ note:** `asyncio.Queue.shutdown()` and `asyncio.QueueShutDown` provide an explicit queue-shutdown mechanism. The sentinel pattern above remains useful for Python 3.11/3.12 compatibility and for understanding producer-consumer coordination.

### Example 23B — Queue shutdown without sentinels (Python 3.13+)

```python
import asyncio


async def consumer(queue):
    while True:
        try:
            item = await queue.get()
        except asyncio.QueueShutDown:
            return
        try:
            print("processed:", item)
        finally:
            queue.task_done()


async def main():
    queue = asyncio.Queue()
    async with asyncio.TaskGroup() as group:
        group.create_task(consumer(queue))
        group.create_task(consumer(queue))

        for item in range(4):
            await queue.put(item)

        queue.shutdown()  # Reject new items, but allow queued items to drain.
        await queue.join()  # Wait for every queued item to receive task_done().


asyncio.run(main())
```

- The consumers loop until `queue.get()` raises `QueueShutDown`.
- `queue.shutdown()` prevents additional items from being put into the queue. With the default `immediate=False`, already queued items remain available to consumers.
- Each successful `get()` receives exactly one `task_done()` call in `finally`.
- When the queue is empty after shutdown, waiting/future `get()` calls raise `QueueShutDown`, allowing consumers to exit without sentinel values.
- This API was added in Python 3.13, so do not use this example unchanged on Python 3.11 or 3.12.

Avoid `queue.shutdown(immediate=True)` unless you specifically want to discard queued work. Immediate shutdown changes the normal `join()` invariant: join can unblock even though those discarded items were not processed.

# Part V — Async iteration and resource management

## 26. `async for` and asynchronous iterators

An asynchronous iterator supplies values where getting the next value may require waiting. The protocol uses:

- `__aiter__()` to obtain an asynchronous iterator.
- `__anext__()` to obtain the next awaitable value.
- `StopAsyncIteration` to signal the end.

You usually use `async for` rather than implementing these methods manually.

## 27. Async generators

An async generator is declared with `async def` and contains `yield`. It can produce values over time and suspend between values.

### Example 24 — Generate values asynchronously

```python
import asyncio


async def ticker(count):
    for number in range(count):
        await asyncio.sleep(0.05)               # Wait before producing each value.
        yield number                            # Yield one value to the async iterator.


async def main():
    async for value in ticker(3):               # Request values one at a time.
        print(value)


asyncio.run(main())
```

- `ticker(3)` creates an asynchronous generator object.
- `async for` requests the next value from it.
- Each `await asyncio.sleep(...)` allows other Tasks to run before the next value is yielded.
- `yield` sends a value to the consumer without ending the generator.
- After three values, the generator stops and the loop exits.

## 28. `async with` and asynchronous context managers

An asynchronous context manager implements `__aenter__()` and `__aexit__()`. These methods can await, so they can acquire or release resources asynchronously.

### Example 25 — Create an async context manager

```python
import asyncio
from contextlib import asynccontextmanager


@asynccontextmanager
async def managed_resource():
    print("opening resource")
    resource = {"connected": True}
    try:
        yield resource                        # Value provided to the `as resource` variable.
    finally:
        resource["connected"] = False
        print("closing resource")


async def main():
    async with managed_resource() as resource:
        print("using resource:", resource["connected"])
        await asyncio.sleep(0.05)


asyncio.run(main())
```

- `@asynccontextmanager` turns an async generator into an async context manager.
- Setup runs before `yield`.
- The object yielded becomes the value after `as`.
- The body inside `async with` uses the resource.
- The `finally` block runs on normal exit, exception, or cancellation, making it the right place for cleanup.

In real applications, database clients and HTTP clients often provide their own `async with` interface. Use it so connections are released reliably.

# Part VI — Blocking work, threads, and processes

## 29. Never block the event loop accidentally

These calls block the event-loop thread when called directly inside an async Task:

- `time.sleep(2)`
- `requests.get(url)`
- synchronous database client calls
- long CPU loops

Prefer a native asynchronous library. If that is unavailable, move blocking work to a thread. For CPU-heavy Python work, consider a process pool.

## 30. `asyncio.to_thread()` — run blocking I/O in a thread (Python 3.9+)

### Example 26 — Offload a synchronous function

```python
import asyncio
import time


def blocking_lookup(key):
    time.sleep(0.2)                             # Simulates a blocking I/O call.
    return f"value-for-{key}"


async def main():
    result = await asyncio.to_thread(blocking_lookup, "customer-7")
    print(result)


asyncio.run(main())
```

- `blocking_lookup` is an ordinary synchronous function. It does not use `async def`.
- `time.sleep` is intentionally blocking, but it runs in a worker thread rather than the event-loop thread.
- `asyncio.to_thread(function, *args)` submits the function to the default thread pool.
- `await` waits for the thread's result without blocking the event-loop thread.

`to_thread()` is primarily for blocking I/O. In standard GIL-enabled CPython, it normally does not provide parallel speedup for CPU-heavy pure-Python bytecode. Also, cancelling the Task awaiting `to_thread()` does not forcibly stop a synchronous function that is already running in its thread.

## 31. `run_in_executor()` — select an executor

`loop.run_in_executor(executor, function, *args)` schedules a synchronous function in an executor and returns an awaitable Future.

- Pass `None` to use the event loop's default thread pool.
- Pass a `ThreadPoolExecutor` to use a specific thread pool.
- Pass a `ProcessPoolExecutor` for process-based execution.

### Example 27 — Use the default thread pool

```python
import asyncio
import time


def blocking_work():
    time.sleep(0.1)
    return "done in a worker thread"


async def main():
    loop = asyncio.get_running_loop()            # Get the loop currently running this Task.
    result = await loop.run_in_executor(None, blocking_work)
    print(result)


asyncio.run(main())
```

`asyncio.get_running_loop()` is the preferred way to get the current loop from inside a coroutine. Avoid manually creating an event loop for ordinary application code.

## 32. `ProcessPoolExecutor` — CPU-bound work

For CPU-heavy pure-Python functions, a process pool can run work in separate processes and use multiple CPU cores. Submitted functions and arguments generally need to be picklable. Keep process-pool startup behind the `if __name__ == "__main__"` guard, especially on platforms using process spawning.

### Example 28 — Run CPU work in a process pool

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor


def sum_of_squares(limit):
    return sum(number * number for number in range(limit))


async def main():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, sum_of_squares, 1_000_000)
        print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

- `sum_of_squares` is a normal synchronous function suitable for a worker process.
- `ProcessPoolExecutor()` creates a pool of processes.
- `run_in_executor(pool, ...)` submits the function and argument to that pool.
- `await` lets the event loop continue scheduling other Tasks while the process computes.
- The `with` block shuts down the pool after the result is received.

Processes have startup and serialization costs. Use them when the work is substantial enough to justify those costs, and benchmark with realistic workloads.

## 33. Calling asyncio safely from another OS thread

Asyncio objects such as `asyncio.Event`, `asyncio.Queue`, and Tasks are not general-purpose thread-safe objects. When another OS thread needs to notify the loop, use:

- `loop.call_soon_threadsafe(callback, *args)` to schedule a normal callback on the loop thread.
- `asyncio.run_coroutine_threadsafe(coro, loop)` to submit a coroutine from another thread. It returns a `concurrent.futures.Future`; call `.result()` from the external thread, not from the event-loop thread.

### Example 29 — Notify an asyncio Future from a worker thread

```python
import asyncio
import threading


def notify_loop(loop, future):
    loop.call_soon_threadsafe(future.set_result, "message from thread")


async def main():
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    thread = threading.Thread(target=notify_loop, args=(loop, future))
    thread.start()

    result = await future
    await asyncio.to_thread(thread.join)         # Join without blocking the event-loop thread.
    print(result)


asyncio.run(main())
```

- The main coroutine creates a Future that belongs to the running loop.
- A normal OS thread is started.
- That thread calls `call_soon_threadsafe`, asking the event loop to run `future.set_result(...)` on the loop's own thread.
- The `await future` expression suspends until the callback sets the Future result.
- `thread.join()` is also blocking, so it is offloaded with `asyncio.to_thread`.

### Additional API — `asyncio.run_coroutine_threadsafe()`

Use this only when an external OS thread needs to submit a coroutine to an event loop that is already running. The example runs the blocking `.result()` call inside a worker thread, never on the event-loop thread.

```python
import asyncio


async def async_operation():
    await asyncio.sleep(0.05)
    return "result from event loop"


def submit_from_thread(loop):
    future = asyncio.run_coroutine_threadsafe(async_operation(), loop)
    return future.result(timeout=2)              # This function runs in a worker thread.


async def main():
    loop = asyncio.get_running_loop()
    result = await asyncio.to_thread(submit_from_thread, loop)
    print(result)


asyncio.run(main())
```

- `async_operation()` is the coroutine to be submitted.
- `submit_from_thread` is synchronous code that is executed by `asyncio.to_thread` in a worker thread.
- `run_coroutine_threadsafe` schedules the coroutine on the already-running event loop and returns a thread-safe `concurrent.futures.Future`.
- `future.result(timeout=2)` blocks the worker thread until the coroutine completes. It must not be called from the event-loop thread, because that would block the loop that needs to run the coroutine.
- `await asyncio.to_thread(...)` keeps the event loop responsive while the worker thread waits for the result.

# Part VII — Networking with asyncio

## 34. Does asyncio include an HTTP client?

No. The `asyncio` module provides low-level networking support and high-level stream APIs, but it is not a complete HTTP client library. For HTTP requests, use an async-capable third-party client such as `aiohttp` or `httpx`'s `AsyncClient`.

The standard library does provide `asyncio` **streams** for TCP connections. A stream gives you:

- `StreamReader` for receiving data (`read`, `readline`, `readexactly`).
- `StreamWriter` for sending data (`write`, `drain`, `close`, `wait_closed`).
- `asyncio.open_connection()` to connect to a server.
- `asyncio.start_server()` to start a TCP server.

## 35. `asyncio` streams — a complete local TCP echo example

This example starts a TCP server on the local machine, connects a client to it, sends one line, receives the uppercase response, and shuts down. It does not require internet access.

### Example 30 — Local TCP echo server and client

```python
import asyncio


async def handle_client(reader, writer):
    data = await reader.readline()                # Read bytes until a newline arrives.
    writer.write(data.upper())                    # Queue the uppercase response for sending.
    await writer.drain()                          # Wait until the write buffer has room.
    writer.close()                                # Begin closing this connection.
    await writer.wait_closed()                    # Wait for the connection to close.


async def main():
    server = await asyncio.start_server(handle_client, "127.0.0.1", 0)
    address = server.sockets[0].getsockname()
    port = address[1]                             # Port 0 asks the OS to choose a free port.

    try:
        reader, writer = await asyncio.open_connection("127.0.0.1", port)
        writer.write(b"hello asyncio\n")         # StreamWriter.write expects bytes.
        await writer.drain()                      # Flush buffered output to the transport.

        response = await reader.readline()        # Read the server's newline-terminated response.
        print(response.decode().strip())          # Convert bytes to text for display.

        writer.close()
        await writer.wait_closed()
    finally:
        server.close()                            # Stop accepting new connections.
        await server.wait_closed()                # Wait for the listening socket to close.


asyncio.run(main())
```

**Line-by-line explanation**

- `handle_client(reader, writer)` is invoked for each accepted client connection. Because it is async, asyncio schedules it as a Task.
- `reader.readline()` waits asynchronously for a newline-terminated byte sequence.
- `writer.write(...)` queues bytes for sending; it does not itself wait for the buffer to drain.
- `await writer.drain()` applies flow control and waits when the write buffer is full.
- `writer.close()` begins closing the stream; `wait_closed()` waits for completion.
- `start_server(..., "127.0.0.1", 0)` starts a server on loopback. Port `0` means the OS selects an available port.
- `server.sockets[0].getsockname()` returns the bound socket address; element 1 is the selected TCP port.
- `open_connection` connects the local client to that port.
- The client writes bytes and reads the server response.
- The `finally` block shuts down the server even if the client code raises an exception.

Expected output:

```text
HELLO ASYNCIO
```

**Production considerations:** define a protocol, validate incoming data, handle disconnects and connection errors, set appropriate limits/timeouts, and close writers in `finally` blocks. This tiny echo server is a teaching example, not a hardened production server.

# Part VIII — Event loop and lower-level APIs

## 36. High-level versus low-level APIs

Application code should usually use high-level APIs:

- `asyncio.run()`
- `asyncio.create_task()`
- `asyncio.TaskGroup`
- `asyncio.gather()`
- `asyncio.timeout()`
- `asyncio.Queue`, `Lock`, `Semaphore`
- `asyncio.open_connection()` / `start_server()`

Framework authors and advanced integrations may use lower-level event-loop APIs. You should understand what they do, but you usually do not need to manually create and close an event loop.

## 37. `asyncio.get_running_loop()`

Returns the event loop running in the current OS thread. Call it from inside a coroutine or callback. It raises `RuntimeError` if no loop is running in that thread.

Prefer it over `asyncio.get_event_loop()` inside coroutines. The behavior of `get_event_loop()` has changed across Python versions and the older policy system is deprecated in modern Python.

## 38. Futures — the low-level bridge for callback-based APIs

A Future represents a result that will be supplied later. Most application code should await library-provided Futures rather than create them manually. When you need to bridge a callback API into async code, use `loop.create_future()`.

### Example 31 — Turn a scheduled callback into an awaitable result

```python
import asyncio


async def main():
    loop = asyncio.get_running_loop()
    future = loop.create_future()                # Create a Future attached to this loop.

    loop.call_later(0.05, future.set_result, "ready")
    result = await future                        # Wait until the callback sets its result.
    print(result)


asyncio.run(main())
```

- `loop.create_future()` creates a Future associated with the current loop.
- `loop.call_later(0.05, ...)` schedules a callback roughly 0.05 seconds in the future.
- The callback invokes `future.set_result("ready")`.
- `await future` suspends until that result is available.

A Future can be completed once. Calling `set_result()` or `set_exception()` after it is already done raises `InvalidStateError`. Prefer `set_result`/`set_exception` only in the code responsible for completing that Future.

## 39. Scheduling callbacks: `call_soon`, `call_later`, `call_at`

These are lower-level APIs that schedule ordinary synchronous functions (callbacks), not coroutine functions.

| API | Meaning |
|---|---|
| `loop.call_soon(callback, *args)` | Run callback on the next event-loop iteration. |
| `loop.call_later(delay, callback, *args)` | Run callback after a delay. |
| `loop.call_at(when, callback, *args)` | Run callback at an absolute time measured using the loop's monotonic clock. |
| `loop.time()` | Read the loop's monotonic clock. |
| `loop.call_soon_threadsafe(...)` | Thread-safe scheduling from another OS thread. |

### Example 32 — Schedule callbacks

```python
import asyncio


def show_message(message):
    print(message)


async def main():
    loop = asyncio.get_running_loop()

    loop.call_soon(show_message, "scheduled soon")
    loop.call_later(0.05, show_message, "scheduled after a delay")

    deadline = loop.time() + 0.1
    loop.call_at(deadline, show_message, "scheduled at a loop-time deadline")

    await asyncio.sleep(0.15)                    # Keep main alive until callbacks have run.


asyncio.run(main())
```

- `show_message` is a normal synchronous function.
- `call_soon` schedules it for the next loop iteration.
- `call_later` schedules a delayed callback.
- `loop.time()` uses a monotonic clock, appropriate for elapsed-time deadlines.
- `call_at` takes an absolute timestamp from that same loop clock.
- `asyncio.sleep(0.15)` keeps the loop running long enough for all callbacks to execute.

Do not use `call_soon` from a different OS thread. Use `call_soon_threadsafe` for that case.

## 40. `asyncio.Runner` — multiple top-level async calls on one loop (Python 3.11+)

`asyncio.run()` is usually the right choice for one top-level entry point. `asyncio.Runner` is useful when a synchronous program needs to make multiple top-level async calls using the same event loop and context.

```python
import asyncio


async def show_number(number):
    await asyncio.sleep(0.05)
    print(number)


with asyncio.Runner() as runner:
    runner.run(show_number(1))
    runner.run(show_number(2))
```

- `Runner()` owns the loop lifecycle.
- Each `runner.run(...)` runs one coroutine on the same managed loop.
- Leaving the `with` block shuts the runner down.

Most beginners do not need `Runner`; use `asyncio.run(main())` unless you have the specific multiple-entry-point requirement.

# Part IX — Additional useful asyncio APIs

## 41. Introspection and diagnostics

Useful functions when debugging:

| API | Purpose |
|---|---|
| `asyncio.current_task()` | Return the Task currently running in this loop/thread, or `None` if not in a Task. |
| `asyncio.all_tasks()` | Return unfinished Tasks for the specified/current loop. |
| `asyncio.Task.get_name()` | Read a Task's name. |
| `asyncio.Task.print_stack()` | Print the stack of a suspended Task. |
| `loop.set_debug(True)` / `asyncio.run(main(), debug=True)` | Enable asyncio debug checks and diagnostics. |

### Example 33 — Print the current Task name

```python
import asyncio


async def worker():
    task = asyncio.current_task()
    print("current task:", task.get_name())
    await asyncio.sleep(0.05)


async def main():
    task = asyncio.create_task(worker(), name="diagnostic-worker")
    await task
    print("unfinished tasks:", len(asyncio.all_tasks()))


asyncio.run(main(), debug=True)
```

The `main` Task itself is still running when `all_tasks()` is called, so the set normally includes `main`. Introspection is useful for debugging; do not use task enumeration as a replacement for explicit task ownership.

## 42. Async subprocesses

`asyncio.create_subprocess_exec()` and `asyncio.create_subprocess_shell()` let a program wait asynchronously for an external process.

Prefer `create_subprocess_exec()` when you can pass the executable and arguments separately. Shell command strings need careful quoting and may create command-injection risks if built from untrusted input.

### Example 34 — Run a child Python process

```python
import asyncio
import sys


async def main():
    process = await asyncio.create_subprocess_exec(
        sys.executable,
        "-c",
        "print('hello from child process')",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )

    stdout, stderr = await process.communicate()  # Read output and wait for process exit.
    print("return code:", process.returncode)
    print("stdout:", stdout.decode().strip())
    print("stderr:", stderr.decode().strip())


asyncio.run(main())
```

- `sys.executable` runs the same Python interpreter as the current program.
- `-c` tells Python to execute the following string as code.
- `stdout=PIPE` and `stderr=PIPE` capture the child's output.
- `communicate()` reads the pipes and waits for process completion, avoiding the common deadlock risk of waiting without draining full pipes.
- `returncode` is the child's exit status.

# Part X — Errors, anti-patterns, and production design

## 43. Common errors and what they mean

| Error / warning | Usual reason | Fix |
|---|---|---|
| `RuntimeWarning: coroutine was never awaited` | Called an async function but neither awaited nor scheduled its coroutine. | Use `await`, `create_task`, or an appropriate asyncio API. |
| `RuntimeError: asyncio.run() cannot be called from a running event loop` | Tried to start a second loop in the same thread. | Await the coroutine from the existing loop. |
| `RuntimeError: no running event loop` | Called `get_running_loop()` when not inside a running loop/callback. | Call from async code or use the appropriate top-level runner. |
| `asyncio.CancelledError` | The Task received a cancellation request. | Clean up and normally re-raise. |
| `TimeoutError` | A timeout deadline expired. | Handle timeout, retry if safe, or propagate. |
| `ExceptionGroup` | Multiple errors were collected, commonly by `TaskGroup`. | Handle relevant types with `except*`. |
| `asyncio.QueueEmpty` | `get_nowait()` found no item. | Use `await queue.get()` or handle the empty case. |
| `asyncio.QueueFull` | `put_nowait()` found no capacity. | Use `await queue.put()` or handle the full case. |
| `asyncio.InvalidStateError` | Tried to read/complete a pending or already completed Future/Task incorrectly. | Check state and assign one owner to Future completion. |

## 44. Common anti-patterns

### Anti-pattern 1: blocking the event loop

Bad inside async code:

```python
import time

# time.sleep(2)  # Blocks the event-loop thread.
```

Correct asynchronous delay:

```python
import asyncio


async def main():
    await asyncio.sleep(2)


asyncio.run(main())
```

The snippets above are deliberately tiny illustrations: the first call blocks its thread; the second suspends the current Task. `await asyncio.sleep(2)` belongs inside an `async def` function.

### Anti-pattern 2: creating Tasks without owning them

Bad design: create a Task and immediately discard the reference. The Task may outlive the code that created it, and nobody may retrieve its exception.

Better: use `TaskGroup`, or store the Task in a collection and have a clear shutdown/result/error-handling strategy.

### Anti-pattern 3: creating too many Tasks

A semaphore caps active work but does not cap how many waiting Tasks are created. For millions of jobs, use a bounded queue and a fixed number of workers.

### Anti-pattern 4: swallowing cancellation

Do not write `except asyncio.CancelledError: pass` unless you have a very deliberate reason and fully understand cancellation state. In normal application code, perform cleanup and re-raise.

### Anti-pattern 5: confusing `gather` with `TaskGroup`

They have different ownership and failure behavior. Choose intentionally; do not assume both cancel siblings on failure.

### Anti-pattern 6: using asyncio primitives across threads

`asyncio.Lock`, `asyncio.Event`, `asyncio.Queue`, and related primitives are not thread-safe. Use `threading` primitives or the event-loop thread-safe scheduling methods for cross-thread communication.

## 45. Debug mode and development checks

Enable debug mode while developing:

```python
import asyncio


async def main():
    await asyncio.sleep(0.01)
    print("debug run complete")


asyncio.run(main(), debug=True)
```

Debug mode helps identify some wrong-thread calls, slow callbacks, and resource problems. It does not prove that your concurrency logic is correct. Also enable useful warnings during development, and treat “coroutine was never awaited” and unclosed-resource warnings seriously.

## 46. Unit testing async code

Python's `unittest` has `IsolatedAsyncioTestCase` for async test methods. Each test gets an isolated event loop.

### Example 35 — Test an async function

```python
import asyncio
import unittest


async def add_after_delay(a, b):
    await asyncio.sleep(0.01)
    return a + b


class AsyncMathTests(unittest.IsolatedAsyncioTestCase):
    async def test_add_after_delay(self):
        result = await add_after_delay(2, 3)
        self.assertEqual(result, 5)


if __name__ == "__main__":
    unittest.main()
```

- `IsolatedAsyncioTestCase` is the standard-library test base class for `async def` test methods.
- Test methods can use `await` directly.
- `assertEqual` validates the behavior.
- Run this file with `python filename.py` or discovery with `python -m unittest`.

When testing concurrency, avoid assertions that depend on incidental print order. Assert on results, limits, cancellation outcomes, or events that represent the behavior you actually need.

# Part XI — Master the API surface

## 47. Most-used `asyncio` functions

| API | What it does | Typical situation |
|---|---|---|
| `asyncio.run()` | Owns the top-level event loop. | Start a standalone async application. |
| `asyncio.create_task()` | Schedules a coroutine as a Task. | Start work concurrently and keep a handle. |
| `asyncio.sleep()` | Suspends a Task for a time. | Timers, retry delays, yielding in tests. |
| `asyncio.gather()` | Runs multiple awaitables and collects ordered results. | Fan out to a fixed set of independent operations. |
| `asyncio.TaskGroup()` | Manages related child Tasks in one scope. | Structured concurrency and fail-fast task ownership. |
| `asyncio.as_completed()` | Yields completions in completion order. | Process fast results first. |
| `asyncio.wait()` | Returns `done` and `pending` sets. | Custom completion conditions. |
| `asyncio.wait_for()` | Time-limits one awaitable. | A timeout for one operation. |
| `asyncio.timeout()` | Time-limits a block. | One deadline for multiple awaits. |
| `asyncio.shield()` | Stops outer cancellation from propagating into an inner awaitable. | Preserve a specific operation while caller wait is cancelled. |
| `asyncio.to_thread()` | Runs synchronous work in the default thread pool. | Blocking I/O library with no async API. |
| `asyncio.current_task()` | Finds the current Task. | Diagnostics. |
| `asyncio.all_tasks()` | Lists unfinished Tasks on a loop. | Diagnostics and shutdown investigation. |
| `asyncio.open_connection()` | Opens a TCP stream connection. | Async TCP client. |
| `asyncio.start_server()` | Starts a TCP server. | Async TCP server. |
| `asyncio.create_subprocess_exec()` | Starts a subprocess asynchronously. | Run external commands without blocking the loop. |

# Part XII — Choosing an approach and building good systems

## 48. Asyncio versus threading versus multiprocessing

| Workload | Recommended starting point | Why |
|---|---|---|
| A few sequential API calls | Synchronous or async, whichever fits the application. | Concurrency may not be needed. |
| Hundreds/thousands of concurrent network calls | `asyncio` with an async HTTP/database client. | Waiting can overlap without one thread per request. |
| Blocking I/O library that has no async API | `asyncio.to_thread()` or a thread pool. | Keep the event loop responsive while the blocking call runs elsewhere. |
| CPU-heavy pure-Python calculation | `ProcessPoolExecutor` or multiprocessing. | Separate processes can use multiple CPU cores. |
| Native computation that releases the GIL | Benchmark threads or the library's own parallel API. | Native code may run concurrently even in a GIL-enabled build. |
| Extremely low-concurrency script | Synchronous code. | Simplicity can be better than async complexity. |

The GIL story depends on the Python implementation/build and whether native extensions release it. The rule of thumb above describes standard GIL-enabled CPython, not every possible Python runtime.

# Part XIII — Final revision sheet

## 49. Ten rules to remember

1. Calling an `async def` function creates a coroutine object; it does not automatically schedule it.
2. `await` waits for an awaitable; it does not automatically make the next operation concurrent.
3. `create_task`, `gather`, and `TaskGroup` are common ways to schedule concurrent work.
4. `gather` returns results in input order; `as_completed` lets you process completion order.
5. `TaskGroup` owns child Tasks and cancels remaining children when a child fails with an ordinary exception.
6. `task.cancel()` requests cancellation; it does not forcibly terminate arbitrary code.
7. Use `try/finally` for cleanup and normally re-raise `CancelledError`.
8. Use `asyncio.timeout` or `wait_for` to bound waits.
9. Use semaphores to limit active work and bounded queues to control waiting work/backpressure.
10. Never block the event-loop thread; move blocking I/O to threads and consider processes for CPU-heavy Python work.

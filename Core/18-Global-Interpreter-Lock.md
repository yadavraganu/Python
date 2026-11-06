# The Global Interpreter Lock (GIL)

The GIL is a mutex (lock) in CPython that protects access to Python objects, preventing multiple threads from executing Python bytecode at the same time. Even on a 16-core machine, only one thread in a given process is running Python bytecode at any instant.

**Scope:** the GIL is **per-process** and an implementation detail of **CPython** (and PyPy), not part of the Python language. Jython and IronPython have no GIL because they rely on their host VM's concurrency model.

## 1. Why Does the GIL Exist?

It looks like a bottleneck, but it was a pragmatic design choice, mainly about **memory management**.

* **Reference counting:** CPython tracks object lifetime with a reference count on every object (`ob_refcnt`, see `CPython-Object-Model.md`). If two threads incremented or decremented the same count simultaneously, the count could be corrupted, causing memory leaks or, worse, freeing an object that's still in use. One global lock makes every refcount update safe without per-object locking.
* **Simplicity for C extensions:** the early Python ecosystem grew around C extensions. The GIL let extension authors assume the interpreter state wouldn't change under them, without reasoning about fine-grained thread safety.
* **Single-threaded speed:** one coarse lock is cheap. Fine-grained per-object locking adds overhead to *every* operation, which would have slowed the (far more common) single-threaded case.

## 2. How the GIL Actually Works

A thread must hold the GIL to run bytecode. It gives the GIL up in two ways:

1. **Voluntarily, around blocking operations.** Before waiting on I/O (file reads, sockets), `time.sleep()`, or lock waits, the thread releases the GIL so another thread can run. This is why threads help I/O-bound work.
2. **Forced, on a timer.** If a thread keeps running Python code, other threads waiting for the GIL request it, and after a **switch interval** (default 5 ms) the running thread is made to drop it at the next opportunity.

```python
import sys
print(sys.getswitchinterval())   # 0.005 (seconds)
sys.setswitchinterval(0.01)       # tunable, though rarely worth changing
```

Which waiting thread gets the GIL next is up to the OS scheduler, not Python, so thread switching is **non-deterministic**.

## 3. The GIL Does NOT Make Your Code Thread-Safe

A common misconception: "I have the GIL, so I don't need locks." The GIL protects the **interpreter's internals**, not your program's logic. A thread switch can happen **between bytecode instructions**, and one line of Python is often several instructions:

```python
import dis

counter = 0

def inc():
    global counter
    counter += 1

dis.dis(inc)
# LOAD_GLOBAL counter
# LOAD_CONST 1
# BINARY_OP +=
# STORE_GLOBAL counter
```

If a switch lands after the load but before the store, two threads can read the same value and one update is lost. That's the race condition from `Threading-Multiprocessing.md`, and it happens even with the GIL. Use `threading.Lock` (or `queue.Queue`) for shared mutable state.

Some single operations are effectively atomic in CPython because they compile to one bytecode (e.g. `list.append(x)`, `d[key] = value`), but treating that as a guarantee is fragile. It's an implementation detail, and free-threaded builds (below) make it a poor assumption.

## 4. When Does the GIL Matter?

| Task Type | Impact of GIL | Reality |
| :--- | :--- | :--- |
| **CPU-bound** (heavy math, image processing, pure-Python loops) | **High** | Threads take turns rather than running in parallel; overhead from contending for the lock can make multi-threaded code *slower* than sequential. |
| **I/O-bound** (downloads, database calls, file reads) | **Low** | A thread waiting on I/O releases the GIL, so other threads make progress. |

A quick way to see it:

```python
import time
from threading import Thread

def count(n):
    while n > 0:
        n -= 1

N = 50_000_000

start = time.perf_counter()
count(N); count(N)
print(f"Sequential: {time.perf_counter() - start:.2f}s")

start = time.perf_counter()
t1 = Thread(target=count, args=(N,))
t2 = Thread(target=count, args=(N,))
t1.start(); t2.start(); t1.join(); t2.join()
print(f"2 threads:  {time.perf_counter() - start:.2f}s")   # about the same, or slower, on a GIL build
```

Swap `Thread` for `multiprocessing.Process` and the time drops by roughly half on a multi-core machine, since each process has its own interpreter and its own GIL.

### What Releases the GIL?
- Blocking I/O (files, sockets, subprocess waits)
- `time.sleep()` and waiting on locks/queues
- Many C extensions during heavy computation (NumPy, parts of pandas, `hashlib` on large inputs, `zlib`), which explicitly release the GIL while working outside Python objects

## 5. Ways Around the GIL

1. **Multiprocessing:** separate processes, each with its own interpreter, memory, and GIL. Real parallelism, at the cost of process startup, memory, and pickling of data between processes.
2. **C extensions / native libraries:** NumPy, pandas, and similar libraries do heavy work in C/C++/Rust and release the GIL while they do, so threads calling them can genuinely run in parallel.
3. **`asyncio`:** doesn't bypass the GIL, but for I/O-bound work it gets high concurrency on one thread, so GIL contention never comes up.
4. **Sub-interpreters:** since Python 3.12 (PEP 684), each sub-interpreter can have its **own GIL**, so multiple interpreters in one process can run in parallel. Python 3.14 exposes this through the standard library `concurrent.interpreters` module (PEP 734).
5. **Free-threaded Python:** an optional build that removes the GIL entirely (next section).

## 6. Free-Threaded Python (No-GIL)

**PEP 703** made the GIL optional in CPython, rolled out in phases:

- **Python 3.13 (Phase I):** free-threaded build available as an **experimental**, opt-in variant (often installed as `python3.13t`).
- **Python 3.14 (Phase II):** the free-threaded build is **officially supported** (PEP 779), but still a separate, optional build. The standard build still has the GIL, and making no-GIL the default is a later, undecided step (Phase III).

```python
import sys
print(sys._is_gil_enabled())   # False on a free-threaded build (3.13+), True otherwise
```

You can also control it at runtime with the `PYTHON_GIL` environment variable or the `-X gil` flag. A free-threaded interpreter will re-enable the GIL if you import a C extension that hasn't declared itself free-threading compatible.

**Why it took so long, and what it costs:**
- Removing the lock means reference counting and internal data structures need thread-safe replacements (PEP 703 uses techniques like biased reference counting), which adds some **single-threaded overhead** compared with the regular build.
- The enormous **C-extension ecosystem** was written assuming the GIL, so each extension needs auditing and updates before it's safe without it.
- Earlier attempts (such as "Gilectomy") were abandoned because they made single-threaded code noticeably slower.

**Consequence for your code:** code that relied on the GIL to hide races becomes visibly buggy. Proper locking (Section 3) matters *more*, not less.

## Notes & Gotchas

- **The GIL is a CPython implementation detail**, not a Python language rule. Jython and IronPython don't have one.
- **The GIL exists to protect reference counts** (memory management), which is the answer to "why does it exist?"
- **Threads still help I/O-bound work** because blocking operations release the GIL. For CPU-bound work, use `multiprocessing`, native libraries, or a free-threaded build.
- **The GIL doesn't make your code thread-safe.** `counter += 1` is multiple bytecodes, and a thread switch between them loses updates. You still need locks.
- **The switch interval is 5 ms** by default (`sys.getswitchinterval()`); a running thread is asked to release the GIL after that, and the OS decides who gets it next.
- **CPU-bound threads can be slower than sequential code** because of contention for the GIL, a classic demo interviewers like.
- **Each process has its own GIL**, which is exactly why multiprocessing achieves real parallelism.
- **Free-threaded Python is optional and still evolving:** experimental in 3.13, officially supported but non-default in 3.14. Don't say "the GIL has been removed"; say "there's an optional no-GIL build."

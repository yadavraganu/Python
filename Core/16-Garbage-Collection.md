# Python Garbage Collection

Python uses a private heap space where all objects and data structures reside. This heap is managed by the Python memory manager; programmers don't access it directly.

Within this private heap, Python's small-object allocator (`pymalloc`) handles the huge number of small, short-lived objects typical code creates, via an arena/pool/block system. On top of that memory layer, Python primarily uses a hybrid approach to decide **when an object's memory can be reclaimed**, combining:

- **Reference counting** (primary mechanism)
- **Generational (cyclic) garbage collection** (a backstop for reference cycles)

## 1. Reference Counting

Every object has a reference count — an integer tracking how many references point to it (`ob_refcnt` in the `PyObject` header).

**Incrementing** happens when:
- A new variable refers to the object.
- The object is passed as a function argument.
- The object is placed in a container (list, tuple, dict, ...).

**Decrementing** happens when:
- A variable referencing it goes out of scope.
- A reference is explicitly deleted (`del`).
- The object is removed from a container.
- A variable is reassigned to something else.

**Deallocation:** when the count hits zero, no part of the program can reach the object anymore, and Python deallocates it **immediately** — not on some later sweep.

**Advantages:** simple to implement, and memory is reclaimed the instant it's no longer needed, avoiding the pause-the-world behavior some other languages' collectors have.

### Inspecting Reference Counts — `sys.getrefcount()`

```python
import sys

x = []
print(sys.getrefcount(x))   # 2, not 1!
```
**The off-by-one gotcha, confirmed by running it:** `getrefcount` itself takes `x` as an argument, creating a temporary extra reference for the duration of the call — so the reported count is always **one higher** than you'd naively expect. Binding a second name to the same object and checking again confirms the pattern:
```python
y = x
print(sys.getrefcount(x))   # 3 — x, y, and the temporary argument reference
del y
print(sys.getrefcount(x))   # back to 2
```

### The Limitation: Cyclic References

Pure reference counting cannot handle **cycles** — two or more objects referencing each other in a loop, even with no external references pointing in:

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

node1 = Node(1)
node2 = Node(2)
node1.next = node2
node2.next = node1   # cyclic reference

# Even after node1 and node2 go out of scope, each still holds a reference
# to the other, so neither's refcount ever reaches zero through refcounting alone.
```
**Confirmed empirically:** with the cycle collector disabled (`gc.disable()`), deleting both names leaves the objects un-finalized — nothing happens until `gc.collect()` is called explicitly, at which point they're correctly found and freed. Reference counting genuinely cannot reclaim this memory on its own; something else has to notice the cycle.

## 2. Generational Garbage Collection (the Cycle Detector)

Reference counting frees most objects the moment they are no longer used, but it can never free a **cycle**. The cycle detector (the `gc` module) is the second mechanism, and its only job is to find and free cycles. This section explains three things using one small example: **what a generation is, when the detector runs, and what it does when it runs.**

### 2.1 The Example We Will Use

It is the same `Node` class from Section 1, plus a function that creates a cycle and then throws it away:

```python
import gc

def make_cycle():
    a, b = Node(1), Node(2)
    a.next, b.next = b, a      # a and b point at each other
```

When `make_cycle()` returns, the names `a` and `b` disappear, but each Node is still referenced by the other one, so each reference count is 1 and refcounting cannot free them. They are **garbage that only the cycle detector can remove.**

### 2.2 What Is a Generation?

A generation is simply **a list of objects grouped by age.** Python keeps three of them:

| Generation | What is in it | How often it is checked |
|---|---|---|
| 0 | brand-new objects | very often |
| 1 | objects that survived one generation-0 check | less often |
| 2 | objects that survived a generation-1 check | rarely |

Every new object starts in generation 0. Each time an object **survives** a check of its generation, it moves up one level. There is no generation 3: generation 2 is where long-lived objects stay.

**Why bother?** Because most objects die young. A function creates temporary objects and drops them straight away. By checking the newest objects often and the old ones rarely, Python finds most garbage cheaply and avoids repeatedly rescanning the long-lived objects your program keeps for its whole run.

You can watch one object make the journey:

```python
gc.collect()                  # start clean
gc.disable()                  # no automatic runs while we watch, so nothing surprises us

keep = Node(99)               # a long-lived object

def where(obj):               # which generation list is this object in?
    for g in range(3):
        if any(o is obj for o in gc.get_objects(generation=g)):
            return g

print(where(keep))            # 0  - new objects start in generation 0
gc.collect(0)                 # check generation 0 only: keep survives
print(where(keep))            # 1  - promoted
gc.collect(1)                 # check generations 0 and 1
print(where(keep))            # 2  - promoted again
gc.collect()                  # a full check
print(where(keep))            # 2  - it stays in the oldest generation

gc.enable()
```

### 2.3 When Does It Run? (The Trigger)

Python does not check on a timer. It counts. Each generation has a **counter** and a **threshold**:

```python
print(gc.get_threshold())     # (700, 10, 10)
print(gc.get_count())         # the three counters right now, for example (31, 2, 0)
```

What each counter measures:

| Counter | Goes up when | Goes down when |
|---|---|---|
| `count0` | a new container object is created | one is freed (for example by reference counting) |
| `count1` | a generation-0 check finishes | (reset when generation 1 is checked) |
| `count2` | a generation-1 check finishes | (reset when generation 2 is checked) |

So `count0` is roughly "new objects created minus objects freed since the last check." Watch it with our example. Objects that refcounting frees cancel out, but cycle objects stay counted:

```python
gc.collect()
start = gc.get_count()[0]

keep = [Node(i) for i in range(100)]      # 100 Nodes + 1 list = 101 new objects
after_creating = gc.get_count()[0]
del keep                                   # reference counting frees all 101 right away
after_freeing = gc.get_count()[0]
print(start, after_creating, after_freeing)     # for example: 0 102 2

gc.collect()
start = gc.get_count()[0]
for _ in range(50):
    make_cycle()                           # 100 Nodes that refcounting cannot free
print(start, gc.get_count()[0])            # for example: 0 102
```
(Each reading can be off by one or two because the call that reads the counter creates a small object itself.) Freed objects cancel out, but the cycle objects pile up, and that pile is what eventually triggers the detector.

**The rule:** after a new object is created, Python looks at the generations from oldest to youngest, and the first one whose counter is **greater than** its threshold gets checked. Checking generation N also checks every younger generation.

1. `count0` goes above **700**: check generation 0.
2. `count1` goes above **10** (that is, after 11 generation-0 checks): check generation 1 as well.
3. `count2` goes above **10** (after 11 generation-1 checks) **and** the 25% rule below allows it: check generation 2, which means everything.

Now let the detector run on its own and watch it. `gc.callbacks` lets us print a line when each check starts and stops:

```python
def watch(phase, info):
    if phase == "start":
        print(f"generation {info['generation']} check starting, counters = {gc.get_count()}")
    else:
        print(f"   done, freed {info['collected']} objects")

gc.collect()                  # reset the counters to (0, 0, 0)
gc.callbacks.append(watch)

for _ in range(800):          # 800 calls create 1,600 Node objects
    make_cycle()

gc.callbacks.remove(watch)
```
Typical output (the exact numbers can differ by a few):
```
generation 0 check starting, counters = (702, 0, 0)
   done, freed 698 objects
generation 0 check starting, counters = (701, 1, 0)
   done, freed 700 objects
```
Read it like this:

- Each call adds 2 objects, so after about 350 calls `count0` passes 700 (it shows **701 or 702**) and generation 0 is checked.
- That check finds that nearly all of them (about 700) are unreachable cycles and frees them. `count0` drops back to about 0.
- Because a generation-0 check just finished, `count1` is now 1 (see the second line). Generation 1 will be checked on the 11th trigger, not before.

Over a longer run you can count how often each generation is checked. In one run of 20,000 `make_cycle()` calls, generation 0 was checked 53 times and generation 1 was checked 4 times: roughly one trigger in every eleven escalates to generation 1.

**What about generation 2?** A full check has to look at *every* tracked object in the program, which gets expensive as programs grow. So CPython adds an extra rule on top of the threshold: it only runs a full check when the number of objects that have survived since the last full check is more than **25%** of the number that survived the last one. You can see the rule at work:

```python
# 100,000 calls where every object is garbage that dies young:      generation 2 was checked 0 times
# 100,000 calls where we also keep one long-lived object each time: generation 2 was checked 3 times
```
When everything dies young there is nothing old to re-examine, so full checks are skipped. They only happen once enough objects are *surviving* into the oldest generation.

You can also trigger checks yourself: `gc.collect()` runs a full check, and `gc.collect(0)` or `gc.collect(1)` check just the younger generations.

### 2.4 What Happens During a Check? (The Algorithm, Step by Step)

Here is what the detector does to the generation it is checking. Python does **not** start from your global variables and walk outwards. It asks a simpler question: *"Is this object referenced from outside this group of objects?"* If not, only other garbage is holding it.

It uses a scratch number called `gc_refs` for each object, and works in five steps. Follow our two-node cycle through them.

**Case 1: both names `a` and `b` are gone** (what `make_cycle()` leaves behind):

| Step | What the detector does | Node `a` | Node `b` |
|---|---|---|---|
| 1. Copy | `gc_refs` = the object's reference count | 1 | 1 |
| 2. Subtract internal references | for every reference *between objects in this group*, take 1 off the target | `b` points at `a`: 1 - 1 = **0** | `a` points at `b`: 1 - 1 = **0** |
| 3. Find what is still used | anything with `gc_refs` above 0 is referenced from outside, so it is alive (and so is everything it reaches) | 0: not used | 0: not used |
| 4. Free the rest | run `__del__` and weakref callbacks, then clear the unreachable objects' references so their counts drop to 0 and they are deallocated | freed | freed |
| 5. Promote survivors | anything that was kept moves up one generation | n/a | n/a |

Both objects ended with `gc_refs` of 0 in step 2: all of their references came from each other, nothing outside the group uses them, so they are garbage.

**Case 2: you still hold the name `a`**, so one object is used from outside:

| Step | Node `a` | Node `b` |
|---|---|---|
| 1. Copy | reference count 2 (the name `a`, plus `b.next`) | 1 (just `a.next`) |
| 2. Subtract internal references | 2 - 1 (from `b`) = **1** | 1 - 1 (from `a`) = **0** |
| 3. Find what is still used | 1 is above 0: referenced from outside, so **alive** | reachable from `a`, so it is **rescued** too |
| 4. Free the rest | nothing to free | nothing to free |

You can run both cases and see it:

```python
gc.collect()
gc.disable()

a, b = Node(1), Node(2)
a.next, b.next = b, a
import weakref
wa, wb = weakref.ref(a), weakref.ref(b)        # weak references let us check if they still exist

del b                                           # only the NAME b is gone; the name a remains
print(gc.collect(), wa() is not None, wb() is not None)    # 0 True True   - nothing freed, both alive

del a                                           # now nothing outside refers to the cycle
print(gc.collect(), wa() is not None, wb() is not None)    # 2 False False - both freed

gc.enable()
```
One outside reference was enough to keep the whole cycle alive, and removing it let the detector free both objects on the next check.

### 2.5 The Whole Story in One Timeline

1. `make_cycle()` runs. Two new Nodes are born in **generation 0** and `count0` goes up by 2.
2. The function returns. The names vanish, but each Node still has a reference count of 1, so refcounting leaves them alone. They are garbage sitting in generation 0.
3. After enough calls, `count0` passes **700**. The detector checks generation 0 using the five steps above, finds the garbage cycles, and frees them. Anything that was still in use moves to generation 1. `count0` resets and `count1` goes up by 1.
4. This repeats. On the 11th trigger `count1` passes **10**, so generation 1 is checked as well.
5. Generation 2 is only checked when `count2` passes **10** *and* enough objects have survived into it (the 25% rule), so long-lived objects are rescanned rarely.

### 2.6 Not Every Object Is Tracked

Only objects that **can** hold references to other objects ("container" types) are tracked by the cycle detector, and CPython trims the set further as it goes:

```python
import gc

gc.is_tracked(5)           # False — an int can't reference anything else
gc.is_tracked("hello")     # False — same for str
gc.is_tracked([1, 2, 3])   # True  — a list could hold a reference back to itself

d = {}
gc.is_tracked(d)           # False — an empty dict starts out untracked...
d["k"] = [1]
gc.is_tracked(d)           # True  — ...and becomes tracked once it holds a container

t = tuple([1, 2, 3])
gc.is_tracked(t)           # True  — tuples are tracked when created
gc.collect()
gc.is_tracked(t)           # False — a collection found nothing inside that could form a cycle
```
Tuples are untracked **lazily**, the first time a collection examines them and finds only untrackable contents, while a dict starts untracked and only becomes tracked when a trackable value is stored in it. (A tuple written as a literal constant like `(1, 2, 3)` is typically already untracked by the first collection after the code is compiled, which is why a quick check can seem to give inconsistent answers.) A tracked dict whose contents later become untrackable is untracked by a full collection. This trimming keeps the collector's workload well below "every live object."

## 3. `__del__` and Cycles: An Outdated Worry

In older Python an object with a `__del__` method caught in a reference cycle is **permanently uncollectable**, leaking into `gc.garbage` forever — true before Python 3.4, but **no longer accurate**. **PEP 442** gave the cycle collector a way to determine a safe finalization order and call `__del__` on cyclic objects properly:

```python
import gc

class WithDel:
    def __init__(self, name): self.name = name
    def __del__(self): print(f"__del__ called for {self.name}")

a = WithDel("a")
b = WithDel("b")
a.other = b
b.other = a
del a, b

print("gc.garbage before collect:", gc.garbage)   # []
n = gc.collect()
# __del__ called for a
# __del__ called for b
print("collected:", n)                              # 2
print("gc.garbage after collect:", gc.garbage)       # still [] — properly finalized and freed
```
Modern `gc.garbage` is **normally empty**. It can still end up non-empty in rare cases — most notably if an object's own `__del__` *resurrects* it (creates a new strong reference to `self` during finalization), or with certain legacy C-extension types that don't fully support the cycle collector's finalization protocol — but "cycle + `__del__` = permanent leak" is no longer the correct mental model.

## 4. `weakref` — Intentionally Breaking Cycles

Rather than relying on the cycle collector to clean up after the fact, you can avoid creating a true cycle in the first place with the `weakref` module. A **weak reference** points to an object **without** incrementing its reference count, so it doesn't keep the object alive on its own.

```python
import weakref

class Big:
    pass

obj = Big()
r = weakref.ref(obj)
print(r() is obj)   # True — call it to get the actual object back
del obj
print(r())            # None — the weak reference doesn't block collection
```
With an optional callback, you can even be notified exactly when the referent is collected:
```python
def on_collected(weak_ref):
    print("the object was just garbage collected")

obj2 = Big()
r2 = weakref.ref(obj2, on_collected)
del obj2   # prints the callback message immediately — pure refcounting handles it,
            # since there's no real cycle to wait for the tracing collector on
```

**The classic use case — parent/child back-references:** a `Parent` holding a `Child`, and the `Child` holding a reference back to its `Parent`, is exactly the cyclic shape from Section 1. Making the child's back-reference a `weakref.ref` instead of a normal attribute breaks the cycle entirely — confirmed empirically: with the back-reference as a weak reference, `gc.collect()` after deleting both finds **0** unreachable objects, because plain reference counting already reclaimed them with no cycle to worry about.

Other common tools in the module:
- **`weakref.WeakValueDictionary`** / **`WeakKeyDictionary`** — dict variants whose entries disappear automatically once the referenced object is collected elsewhere. Ideal for caches that shouldn't themselves keep an object alive:
  ```python
  cache = weakref.WeakValueDictionary()
  val = Big()
  cache["key"] = val
  print("key" in cache)   # True
  del val
  print("key" in cache)    # False — the entry vanished on its own once val was collected
  ```
- **`weakref.proxy(obj)`** — like `ref()`, but behaves transparently like the object itself (attribute access forwards through), raising `ReferenceError` if used after the referent is gone, instead of requiring you to call it like `ref()` does.

## 5. The `gc` Module — Reference

| Function | What it does |
|---|---|
| `gc.enable()` | Enables automatic collection (on by default) |
| `gc.disable()` | Disables automatic collection — use with caution; cycles will only go away via a manual `gc.collect()` |
| `gc.collect(generation=None)` | Forces a collection (full, by default). Returns **the number of unreachable objects found** (and collected) — not "uncollectable objects," a common mis-description |
| `gc.get_count()` | Current `(count0, count1, count2)` allocation counters per generation |
| `gc.get_threshold()` / `gc.set_threshold(...)` | Reads/sets the `(threshold0, threshold1, threshold2)` trigger points |
| `gc.get_objects()` | All objects currently tracked by the collector — useful for memory debugging |
| `gc.get_referrers(obj)` | What refers *to* `obj` |
| `gc.get_referents(obj)` | What `obj` refers *to* |
| `gc.is_tracked(obj)` | Whether the collector tracks this object at all (see Section 2) |
| `gc.garbage` | Objects the collector found uncollectable — normally empty in modern Python (see Section 3) |
| `gc.freeze()` / `gc.unfreeze()` / `gc.get_freeze_count()` | Move all currently-tracked objects into a special permanent generation the collector skips, confirmed to work as described — used by frameworks that `fork()` worker processes (e.g., pre-loading app state in Gunicorn) to avoid the copy-on-write memory duplication a collection sweep would otherwise trigger across forked children |
| `gc.set_debug(flags)` | Enables collector debug output (e.g., `gc.DEBUG_STATS`), for diagnosing GC behavior directly |

## Notes & Gotchas

- **Reference counting is immediate; the generational collector is a periodic backstop specifically for cycles** — a strong answer distinguishes these two mechanisms rather than treating "Python's GC" as one monolithic thing.
- **`sys.getrefcount(x)` always reports one more than you'd expect**, because the call itself holds a temporary reference — confirmed above, and a common off-by-one trap when debugging reference leaks.
- **"A `__del__` method inside a cycle means it can never be collected" is outdated** — true before Python 3.4, fixed by PEP 442. Stating the *old* belief confidently in an interview is itself a minor red flag; know the modern behavior.
- **`gc.collect()`'s return value is unreachable objects found, not "uncollectable" ones** — the two are easy to conflate, and the actual docstring says "unreachable."
- **Not everything is GC-tracked** — scalar types that can't reference other objects (`int`, `str`, ...) are skipped by `gc.is_tracked()`, so the collector's real workload is smaller than "every live object."
- **`weakref` prevents a cycle from forming in the first place**, rather than relying on the tracing collector to clean one up after the fact — the standard fix for parent/child or observer/subject back-references, and a common follow-up question after discussing cycles.
- **There is no Generation 3** — Gen 2 is the ceiling; survivors just stay there and get rechecked on future full collections.
- **Disabling `gc` entirely can boost throughput for short-lived scripts that create no cycles**, but is risky in long-running processes, since any cycle created will leak for the process's lifetime without a manual `gc.collect()`.
- **CPython's cycle detector isn't a textbook mark-and-sweep from the roots.** It subtracts references that come from inside the generation and keeps what is still referenced from outside; knowing this explains why one outside reference rescues a whole cycle.
- **Tracking is lazy and selective:** tuples are tracked at creation and untracked by the first collection that finds only untrackable contents; dicts start untracked and become tracked when they hold a container.

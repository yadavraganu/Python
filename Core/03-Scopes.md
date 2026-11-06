## Scopes in Python

In Python, **scope** refers to the region of a program where a variable is **defined** and **accessible**. It determines the **visibility** and **lifetime** of a name. Python resolves variable names using a well-defined rule called the **LEGB Rule**.

**Scope vs. namespace — a distinction worth stating precisely:** a **namespace** is the actual mapping from names to objects (implemented as a dict); a **scope** is the textual region of code where a particular namespace is directly accessible without a prefix. `globals()` returns *a* namespace; LEGB is the *rule* for which namespaces get searched, in what order.

### The LEGB Rule

| Scope Level | Description |
|---|---|
| **L**ocal | Names defined inside the current function (including parameters) |
| **E**nclosing | Names in an enclosing (outer) function, for nested functions/closures |
| **G**lobal | Names defined at the top level of the current module |
| **B**uilt-in | Names preloaded by Python (`len`, `print`, `sum`, ...), living in the `builtins` module |

Python searches in this order: **Local → Enclosing → Global → Built-in**, stopping at the first match.

### Examples of Each Scope

#### 1. Local Scope
```python
def greet():
    name = "Anurag"  # Local to greet()
    print(name)
```

#### 2. Enclosing Scope
```python
def outer():
    msg = "Hello"  # Enclosing scope for inner()
    def inner():
        print(msg)
    inner()
```

#### 3. Global Scope
```python
greeting = "Hi"  # Global variable — global to THIS module only, see note below

def say():
    print(greeting)
```

**Important nuance:** "global" in Python means **module-global**, not program-global. Each module has its own separate global namespace; `greeting` here lives in this module's namespace and isn't automatically visible in another module unless explicitly imported.

#### 4. Built-in Scope
```python
print(len("Python"))  # 'len' is a built-in function, found in the `builtins` module
```

---

## Static (Lexical) Scoping and `UnboundLocalError`

Python uses **static/lexical scoping**: which scope a name belongs to is decided by the **compiler**, by scanning the function body ahead of time — not determined dynamically at each line as the function runs.

The practical consequence: if a name is assigned **anywhere** in a function body, Python treats it as **local for the entire function**, even on lines before that assignment.

```python
x = 10

def broken():
    print(x)   # you might expect this to print the global x=10...
    x = 20      # ...but this assignment makes x local for the WHOLE function
    print(x)

broken()   # UnboundLocalError — the wording varies by version (3.12: "cannot access local variable 'x' where it is not associated with a value"; older: "local variable 'x' referenced before assignment")
```

This is one of the most frequently asked "why does this raise an error" interview questions. The fix, if you actually want to modify the module-global `x`, is the `global` keyword (see below) — merely reading a global without assigning to it never requires `global`.

---

## Class Bodies Are NOT Part of the LEGB Chain

A common surprise: the namespace created by a `class` body is **not** treated as an enclosing scope for methods defined inside it. Methods only see Local → Enclosing (from an outer function, if any) → Global → Built-in — the class body is skipped entirely.

```python
class Config:
    timeout = 30

    def show(self):
        print(timeout)     # NameError! class scope isn't searched here

    def show_fixed(self):
        print(Config.timeout)   # must go through the class (or self/cls)
        print(self.timeout)      # or through the instance
```

This is why every method needs `self.attr` or `ClassName.attr` — class-level names are **not** implicitly visible inside method bodies the way enclosing-function variables are inside nested functions. It's a frequently tested distinction between "class scope" and genuine LEGB "enclosing scope."

---

## Comprehension Scoping (Python 3+)

Since Python 3, list/dict/set comprehensions and generator expressions have their **own local scope** — the loop variable does not leak into the surrounding scope (a deliberate change from Python 2's leaky behavior).

```python
squares = [i * i for i in range(5)]
print(i)   # NameError: name 'i' is not defined — 'i' never existed outside the comprehension
```

Note this only applies to comprehensions/generator expressions — a regular `for` loop's variable **does** leak into the enclosing scope and remains accessible afterward:
```python
for j in range(5):
    pass
print(j)   # 4 — plain for-loops don't get their own scope
```

### Comprehensions Inside a Class Body

Combining the two rules above produces a well-known trap: a comprehension has its own scope, and a class body is not a scope that nested scopes can see. So inside a comprehension in a class body, only the **outermost iterable** (which is evaluated in the class scope) can see class variables:

```python
class A:
    xs = [1, 2, 3]
    k = 2
    ys = [v * 2 for v in xs]    # works: `xs` is the outermost iterable, evaluated in the class scope
    zs = [v * k for v in xs]    # NameError: name 'k' is not defined — `k` is looked up inside the comprehension's own scope
```
Workarounds: build the value with a plain `for` loop in the class body, compute it at module level, or do it in `__init__` or a classmethod.

---

## Accessing and Modifying Scope

### Modifying Global Variables — `global`
```python
counter = 0

def increment():
    global counter
    counter += 1
```
`global` can also be used to **create** a module-level name from inside a function, if it doesn't already exist — the name is added to the module's namespace the first time it's assigned.

### Modifying Enclosing Variables — `nonlocal`
```python
def outer():
    count = 0
    def inner():
        nonlocal count
        count += 1
        print(count)
    inner()
```
`nonlocal` looks specifically for the nearest **enclosing function** scope — it explicitly skips the global scope, and raises `SyntaxError` if no matching enclosing binding exists (unlike `global`, which will happily create one at module level).

---

## More Scope Gotchas

### `global` / `nonlocal` Must Come Before Use
Declaring a name `global` after it has already been used or assigned in the function is a `SyntaxError` (`name 'q' is used prior to global declaration`, or `... is assigned to before global declaration`). Put the declaration at the top of the function.

### Shadowing Built-ins
Because Built-in is the **last** scope searched, any assignment in an earlier scope hides the built-in of the same name:
```python
list = [1, 2]         # this module-level name now shadows the built-in `list`
list("abc")           # TypeError: 'list' object is not callable
del list              # removing the shadowing name makes the built-in visible again
list("abc")           # ['a', 'b', 'c']
```
Common victims: `list`, `dict`, `str`, `id`, `max`, `sum`, `input`. Linters flag this.

### `except ... as name` Un-binds the Name
The variable in `except E as err:` is deleted when the block ends, so using `err` afterwards raises `NameError` (`UnboundLocalError` inside a function). Copy it to another name if you need it later (see `04-Exceptions.md`).

### Closures Capture Variables, Not Values
```python
fs = [lambda: n for n in range(3)]
print([f() for f in fs])    # [2, 2, 2] — every lambda looks up `n` when called, after the loop has finished
```
The fixes (a default argument or a factory function) are covered in `08-Closures.md`.

---

## Inspecting Scope Programmatically

### `globals()` and `locals()`
```python
x = 10

def test():
    y = 20
    print("Local:", locals())
    print("Global:", globals())

test()
```
- `globals()` returns the **actual, live** dictionary backing the module's global namespace — mutating it (`globals()['x'] = 99`) really does change the global variable.
- `locals()` inside a function returns a **snapshot copy** of the local namespace at that point, not a live view. In CPython, local variables are actually stored in a fast array-like structure (not a dict) for performance, so `locals()` builds a dict on demand — **mutating the dict `locals()` returns does not reliably affect the actual local variables.** (At module or class-body level, where the namespace genuinely is dict-based, `locals()` behaves more like a live view — the function-scope restriction is the special case worth remembering.) Python 3.13 formalized this with PEP 667: each `locals()` call inside a function returns an independent snapshot, while `frame.f_locals` became a write-through view.

### Using `dir()`
```python
print(dir())   # Lists names in the current scope
```

---

## Summary

| Scope Type | Keyword to Modify | Access Method | Notes |
|---|---|---|---|
| Local | — | `locals()` (snapshot inside functions) | Assignment anywhere in the function makes the whole function treat the name as local |
| Enclosing | `nonlocal` | Closure/nested function | Skips global scope entirely; errors if no enclosing binding exists |
| Global | `global` | `globals()` (live dict) | Actually means "module-global," not program-wide |
| Built-in | — | `import builtins` (not `__builtins__`) | Last resort in LEGB lookup. `__builtins__` is a CPython detail: a module in `__main__` but a dict in imported modules |
| *(Class body)* | — | `ClassName.attr` / `self.attr` | **Not** part of LEGB — must be accessed explicitly, never implicitly from inside a method |

## Notes & Gotchas Worth Knowing for Interviews

- **Scoping is resolved at compile time (static/lexical), not at runtime** — this is the root cause of `UnboundLocalError`, and a stronger answer than just memorizing "it happens when you assign after reading."
- **Class bodies are excluded from LEGB for methods** — a very commonly tested distinction from "enclosing scope," which only refers to enclosing *functions*.
- **Comprehensions get their own scope in Python 3**; plain `for`/`while` loops do not.
- **`nonlocal` vs `global`**: `nonlocal` never touches module-global scope and errors without an enclosing binding; `global` can silently create a new module-level name.
- **`globals()` is live; `locals()` inside a function is a snapshot** — a subtle but real difference that occasionally comes up when someone tries to "hack" a local variable via the `locals()` dict and it silently doesn't work.
- **Class-body comprehensions can't see class variables** (except in the outermost iterable) — two separate rules combining into a classic trick question.
- **Built-ins are the last scope searched**, so assigning to `list`, `id`, `max`, etc. shadows them for the rest of that scope.
- **`global`/`nonlocal` declarations must precede any use of the name**, or Python raises a `SyntaxError`.

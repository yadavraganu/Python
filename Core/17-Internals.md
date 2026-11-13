# Python Internals

## 1. String Interning

String interning is an optimization technique used by interpreters and compilers to save memory and speed up string comparisons. For identical immutable string values, only **one copy** is stored in memory — when a new string literal with the same value appears, the system reuses the reference to the existing interned string instead of allocating a new object.

CPython (the standard, most common implementation) performs automatic interning for certain strings and also exposes a manual interning mechanism. **This is a CPython implementation detail, not a language guarantee** — the Python language spec never promises interning behavior, so relying on `is` for string equality is technically implementation-dependent, not portable behavior.

### When Automatic Interning Typically Happens

- **String literals that look like identifiers, and short strings:**
  Strings resembling Python identifiers (alphanumeric + underscore, no spaces, not starting with a digit) — variable names, function names, keywords — are often interned. Short literal strings (roughly up to ~20 characters, implementation/version-dependent) without spaces or special characters are good candidates.
  ```python
  a = "hello"
  b = "hello"
  print(a is b)   # True (likely interned)

  x = "my_variable_name"
  y = "my_variable_name"
  print(x is y)   # True (likely interned)
  ```

- **Compile-time constants:**
  String literals appearing directly in your code (in function bodies or at module scope) are often interned during compilation.
  ```python
  def greet():
      return "welcome"

  s1 = greet()
  s2 = "welcome"
  print(s1 is s2)   # True (often interned, since "welcome" is a literal)
  ```

- **Literal concatenation (constant folding):**
  If you concatenate string *literals* directly in source code, the compiler's peephole optimizer may pre-compute the result at compile time and intern it if it's short and "internable."
  ```python
  s = "hello" + "world"
  t = "helloworld"
  print(s is t)   # True (folded and interned at compile time)
  ```

### When Automatic Interning Typically Does NOT Happen

- **Dynamically generated strings** — built at runtime via variable concatenation, f-strings, or `.format()`/`%` — are generally *not* automatically interned (no compile-time constant to fold).
- **Strings with spaces or special characters** are less likely to be interned unless very short.

```python
name = "John"
s1 = "hello " + name          # built at runtime — not a compile-time constant
s2 = "hello John"
print(s1 is s2)                # False — not interned, even though values are equal
print(s1 == s2)                 # True — value equality still holds
```

### Manual Interning with `sys.intern()`

For cases where automatic interning doesn't apply but you want it anyway — e.g., many identical long strings parsed from a file or a network stream — force it explicitly:

```python
import sys

long_string_1 = "this is a very very very very long string that might not be automatically interned"
long_string_2 = "this is a very very very very long string that might not be automatically interned"
print(long_string_1 is long_string_2)   # False — not automatically interned

interned_1 = sys.intern(long_string_1)
interned_2 = sys.intern(long_string_2)
print(interned_1 is interned_2)          # True — forced interning
```

**Practical use case:** parsers/tokenizers that see the same repeated tokens (keys, tags, field names) millions of times use `sys.intern()` to cut memory usage and make dict lookups/comparisons faster, since comparing interned strings can short-circuit to a pointer (`is`) comparison in the implementation before falling back to full value comparison.

**Related concept — small integer caching:** CPython applies a similar idea to integers, pre-allocating and reusing `int` objects from `-5` to `256`. `a = 100; b = 100; a is b` is `True`, but the same pattern with `1000` is generally `False`. Same underlying idea (object reuse for memory/speed), different mechanism, and a frequent companion interview question to string interning.

## 2. Python's Module Loading Mechanism

Every `import` statement triggers a sophisticated system of **finders** and **loaders** to locate, prepare, and execute a module.

### Step 1 — `sys.modules` Cache Check
Python first checks `sys.modules`, a dict of every previously loaded module. If present, the cached module object is reused — the file is **not** re-executed.

### Step 2 — The Import Protocol (Finders and Loaders)
If the name is not cached, Python consults the ordered `sys.meta_path` finder list. The default list includes finders for **built-in** modules, **frozen** modules, and path-based imports. A finder that succeeds returns a `ModuleSpec`, which describes the module and the loader responsible for loading it.

### Step 3 — `sys.path` Search
For ordinary filesystem imports, the path-based finder searches the locations in `sys.path`, which commonly include:
- The script directory (or the current directory in interactive use).
- Directories from the `PYTHONPATH` environment variable.
- Standard-library and installed-package directories determined by the interpreter/environment.

Custom meta-path finders can use other sources, so not every import is a filesystem search.

### Step 4 — Create the Module and Cache It Early
The import machinery creates the module object from its `ModuleSpec` (the loader may provide a custom object). It inserts that object into `sys.modules` **before executing the module body**. This prevents unbounded recursion during circular imports, but another module in a circular-import chain can observe a partially initialized module.

### Step 5 — Compile and Cache Source Bytecode
For source modules, the loader compiles source into version-specific bytecode. When bytecode caching is enabled, Python can store it under `__pycache__` as a `.pyc` file. The cache is validated using source metadata (or a hash-based mode); `-B` / `PYTHONDONTWRITEBYTECODE` disables writing bytecode cache files. Bytecode caches are an optimization, not a portable distribution format.

### Step 6 — Execute the Module
The loader executes the module code in its namespace (`module.__dict__`), populating it with the functions, classes, and variables it defines. This is why **top-level code has side effects on import** — anything not guarded by `if __name__ == "__main__":` runs immediately. If execution fails, the failing module's cache entry is normally removed; successfully imported dependencies remain cached.

### Extensibility
The whole system is pluggable: you can register a custom object on `sys.meta_path` (a custom finder/loader pair) to import modules from unusual sources — databases, zip archives (`zipimport` does exactly this), network locations, or in-memory bytecode.

```python
import sys
print(sys.meta_path)   # the ordered list of finders Python checks, in order
```

## 3. Python Code Execution

Python code passes through a compiler and an interpreter (the **Python Virtual Machine**, PVM) that bridge human-readable source and machine-executable instructions. The stages, in the standard **CPython** implementation:

1. **Source Code Creation** — a programmer writes code and saves it in a `.py` file.
2. **Compilation to Bytecode** — when run, the interpreter's compiler translates source into an intermediate, platform-independent instruction format called **bytecode**, performing lexical and syntax analysis (and raising `SyntaxError` here if something's malformed).
3. **Caching** — the bytecode is saved as a `.pyc` file inside a `__pycache__` directory. If the source hasn't changed, Python loads the cached `.pyc` next time and skips recompilation.
4. **Execution by the interpreter** — bytecode is executed by the CPython evaluation loop. The exact instruction set changes between Python releases. Execution state is represented by interpreter frames; Python-visible frame objects may be materialized when introspection/debugging needs them.
5. **Machine Code Generation** — under the hood, each bytecode instruction is ultimately carried out via C code in the CPython interpreter that performs the corresponding low-level operation on the CPU.
6. **Output** — the CPU executes that underlying machine code and produces the program's results.

This is why Python is called an "interpreted language" even though it does involve an initial compile-to-bytecode step: the bytecode itself is still interpreted at runtime rather than compiled ahead-of-time to native machine code, unlike a fully compiled language such as C. The bytecode/PVM design is also what gives CPython its portability across operating systems — the same `.pyc` bytecode format runs anywhere the PVM does.

### Inspecting Bytecode Directly

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# Opcode names and specialization details vary by Python release and build.
```
Being able to say "I'd check this with `dis.dis()`" is a good way to show real familiarity with internals in an interview, not just memorized theory.

### CPython Is Not the Only Implementation

Worth knowing for a well-rounded answer:
- **CPython** — the reference implementation (what "Python" usually means); compiles to its own bytecode, interpreted by the PVM.
- **PyPy** — uses a JIT (Just-In-Time) compiler, often much faster for long-running programs, at the cost of slower startup and some C-extension compatibility gaps.
- **Jython** — compiles to JVM bytecode, runs on the Java Virtual Machine.
- **IronPython** — targets the .NET runtime.

The **GIL** (Global Interpreter Lock) is specifically a **CPython** (and PyPy) implementation detail of how the PVM's execution loop is protected — not a property of "Python" as a language, another common point of confusion.

## 4. Attribute Lookup — How `obj.attr` Actually Resolves

When you write `obj.attr`, the default `object.__getattribute__` implementation follows a defined descriptor/instance/class lookup order. A custom `__getattribute__` can override this behavior:

1. **Look up `attr` on `type(obj)` and its MRO** (the class hierarchy, via C3 linearization). If found and it is a **data descriptor** (defines `__set__` or `__delete__`, usually alongside `__get__`, e.g. a `property`), it wins immediately — data descriptors take priority over instance attributes.
2. **Look up `attr` in `obj.__dict__`** (the instance's own attributes). If found here, use it — *unless* step 1 already matched a data descriptor.
3. **Fall back to the class-level result from step 1** if it was a plain attribute or a **non-data descriptor** (defines `__get__` but neither `__set__` nor `__delete__`, e.g. a plain function/method).
4. **Raise `AttributeError`** if nothing matched, unless the class defines `__getattr__`, which is called as a last resort.

```python
class Demo:
    class_attr = "class"

    @property
    def prop(self):        # data descriptor — always wins over instance dict
        return "from property"

d = Demo()
d.__dict__["prop"] = "instance"   # even forcing it into the instance dict...
print(d.prop)                       # ...still prints "from property"
```

This ordering — **data descriptor > instance `__dict__` > non-data descriptor/class attribute > `__getattr__`** — is exactly why `@property` can't be silently overridden by setting `self.x = value` from inside `__init__`, and it's the unifying mechanism behind descriptors, methods (which are non-data descriptors), and `__slots__`.

## 5. Frame Objects and the Call Stack

A function call has an **execution frame** — the interpreter's record of that call's execution state. In CPython 3.11+, the internal interpreter frame is distinct from the Python-level `frame` object exposed through introspection; the latter can be materialized lazily:

- **Local variables** (the frame's own namespace).
- **The instruction pointer** (which bytecode instruction is next).
- **The value/evaluation stack** used to execute that frame's bytecode.
- **A reference to the calling frame** (`f_back`), which is what forms the call stack.

```python
import sys

def inner():
    frame = sys._getframe()
    print(frame.f_code.co_name)     # 'inner'
    print(frame.f_back.f_code.co_name)  # the caller's function name
    print(frame.f_locals)             # this frame's local variables

def outer():
    inner()

outer()
```

This is exactly what the `traceback` module walks to print a stack trace, what debuggers (`pdb`) inspect to let you step through code, and what generators rely on to suspend/resume — a generator object keeps its frame alive between `yield` statements instead of discarding it like a normal function return would.

## Notes & Gotchas Worth Knowing for Interviews

- **Interning is a CPython implementation detail, not a language guarantee** — never rely on `is` for string value equality in real code; always use `==`.
- **`sys.modules` caches modules by import name** — repeated imports under the same name normally reuse the cached object rather than execute the source again. Explicit reloads, cache deletion, failed imports, or importing one file under multiple names can cause execution again.
- **`.pyc` invalidation is based on the source's mtime/hash, not content diffing** — touching a file's timestamp without changing content can still trigger recompilation.
- **Bytecode ≠ machine code** — bytecode is still interpreted by the PVM at runtime; this distinguishes Python from ahead-of-time compiled languages, and is the precise answer to "is Python compiled or interpreted?" (both, in a sense — compiled to bytecode, then interpreted).
- **The GIL belongs to the CPython/PyPy implementations**, not to the Python language itself — Jython and IronPython don't have one, since they rely on their host VM's own concurrency model.
- **`sys.meta_path` order determines resolution priority** — built-ins and frozen modules are checked before the filesystem-based path finder, which is part of why you can't accidentally shadow a truly built-in module (like `sys` itself) by naming a local file the same thing, unlike pure-Python standard-library modules (like `math` before Python 3's C implementation nuances, or `json`).
- **Data descriptors always beat instance `__dict__`** — this single rule explains why `@property` setters can't be bypassed by direct attribute assignment, and it's the same mechanism `__slots__` uses to block a `__dict__` from being created at all.
- **A generator's frame stays alive between `yield`s** — this is the concrete internals answer to "how does a generator remember where it left off," tying frames directly to how `yield` works under the hood.
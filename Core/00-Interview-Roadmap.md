# Python Core Interview Roadmap

This repository is a practical, code-first guide to Python from language fundamentals through production engineering. It is designed for interviews ranging from junior developer to senior/lead engineer and Python-focused architect.

## Version policy

- The examples target **Python 3.12+** unless a section explicitly marks a different minimum version.
- All Python code fences were syntax-checked with **CPython 3.13.5**. The targeted behavior suite (16 tests) was executed with **CPython 3.13.5 and pytest 9.0.2**. This is not a claim that every contextual fragment was executed as a standalone program.
- As of **13 November 2026**, Python 3.15 is the latest stable feature release. Do not confuse the latest release with the version deployed by a company; interview answers should state the version assumption when behavior is version-dependent.
- Python 3.14+/3.15-only behavior is explicitly version-labelled and was not runtime-tested on the 3.13.5 interpreter. Current-release notes were checked against the relevant documentation.
- Most language-level concepts apply across implementations. Sections on bytecode, reference counts, memory allocation, string interning, and the GIL are explicitly **CPython implementation details**.

## How to use the repository

1. Read the fundamentals in order. Do not memorize isolated definitions; run the examples and predict their output first.
2. For each topic, be able to explain the concept, show a small implementation, name the trade-off, and identify a common failure mode.
3. Work through the exercises without looking at the solution. Then explain why the selected approach is appropriate.
4. For senior interviews, connect language features to production concerns: correctness, complexity, memory, concurrency, testing, observability, security, deployability, and maintainability.
5. Verify version-sensitive claims against the Python and tool versions used by the target team.

## Coverage map

| Area | Main guide(s) | Interview depth |
|---|---|---|
| Values, identity, mutability, copying, hashability | `01-Data-Types.md` | Junior → Senior |
| Expressions, control flow, comprehensions, unpacking, pattern matching | `25-Language-Syntax-And-Control-Flow.md` | Junior → Senior |
| Functions, arguments, callables, recursion, introspection | `02-Functions.md` | Junior → Senior |
| LEGB, namespaces, `global`, `nonlocal`, closures | `03-Scopes.md`, `08-Closures.md` | Junior → Senior |
| Exceptions, chaining, cleanup, exception groups | `04-Exceptions.md` | Junior → Senior |
| Built-in and specialized collections, algorithmic choices | `06-Choose-Collection.md` | Junior → Senior |
| Strings, Unicode, encoding, formatting, regular expressions | `26-Strings-Unicode-And-Regex.md` | Junior → Senior |
| Files, paths, text/binary I/O, JSON/CSV, serialization | `27-File-IO-And-Serialization.md` | Junior → Architect |
| Modules, imports, packaging, environments, publishing, plugins | `07-Modules-And-Packages.md` | Junior → Architect |
| Type annotations, generics, static checking, typing APIs | `28-Typing-And-Static-Analysis.md`, `15-Protocols.md` | Mid → Architect |
| Classes, inheritance, composition, dataclasses, SOLID, metaclasses | `11-Object-Oriented-Program.md`, `29-Dataclasses.md` | Junior → Architect |
| Dunder methods, descriptors, context managers, protocols | `12-Dunder-Methods.md`, `13-Context-Manager.md`, `14-Python-Descriptors.md`, `15-Protocols.md` | Mid → Architect |
| Iteration, generators, lazy evaluation | `10-Iterators-Generators.md` | Junior → Senior |
| Decorators and higher-order functions | `09-Decorators.md` | Mid → Senior |
| Memory management, garbage collection, CPython object layout | `16-Garbage-Collection.md`, `17-Internals.md`, `23-CPython-Object-Model.md` | Senior → Architect |
| GIL, threads, processes, asyncio | `18-Global-Interpreter-Lock.md`, `19-Threading-Multiprocessing.md`, `20-Async-Programming.md` | Mid → Architect |
| Logging and automated testing | `21-Python-Logger.md`, `22-Pytest.md` | Junior → Architect |
| Standard-library problem solving | `30-Standard-Library-Toolkit.md` | Junior → Senior |
| Performance and profiling | `31-Performance-And-Profiling.md` | Mid → Architect |
| Security and safe boundaries | `32-Security-And-Safe-Python.md` | Mid → Architect |
| Design patterns and application-level design | `24-Python-Design-Patterns.md`, `33-Python-Architecture-And-Design.md` | Senior → Architect |
| Compact idioms and revision | `05-OneLiners.md` | Junior → Mid |

## Interview-readiness levels

### Junior

You should be able to:
- Explain Python's built-in types, mutability, identity versus equality, truthiness, and basic complexity.
- Write functions with correct parameter handling and return values.
- Use comprehensions, loops, exceptions, files, modules, and common collections.
- Write classes, use inheritance and composition appropriately, and write basic pytest tests.
- Explain common traps such as mutable defaults, late-bound closures, mutable class attributes, and `is` versus `==`.

### Mid-level

You should also be able to:
- Design clean APIs with positional-only and keyword-only parameters, decorators, context managers, iterators, and generators.
- Use type hints, protocols, dataclasses, dependency injection, and testing doubles.
- Choose between lists, sets, dictionaries, deques, heaps, and iterators using complexity and workload.
- Select threading, multiprocessing, or asyncio based on workload and resource constraints.
- Package and install an application correctly; configure logging and meaningful tests.

### Senior

You should also be able to:
- Explain object lifetime, reference counting, cyclic GC, descriptors, MRO, the data model, and relevant CPython implementation details.
- Diagnose performance and memory issues with measurement rather than intuition.
- Design concurrency boundaries, cancellation behavior, backpressure, timeouts, failure handling, and resource cleanup.
- Establish reliable test strategies, observability, dependency boundaries, version compatibility, and secure serialization.
- Discuss the cost of abstractions and choose the simplest design that satisfies the requirements.

### Architect / Staff

You should also be able to:
- Set conventions for package boundaries, API stability, type checking, error taxonomy, configuration, and dependency management.
- Define concurrency and execution models for services, batch jobs, and libraries, including shutdown and overload behavior.
- Make explicit trade-offs involving latency, throughput, memory, CPU, reliability, operability, security, and maintainability.
- Design for deployment: reproducible builds, virtual environments, package metadata, CI, test isolation, logging, metrics, and operational diagnostics.
- Explain which conclusions are Python language guarantees and which are CPython-, OS-, or library-version-specific.

## A reusable answer structure

For a design or behavior question, answer in this order:

1. **Requirement:** What problem and constraints are we solving?
2. **Mechanism:** What Python feature or runtime behavior is involved?
3. **Implementation:** Show the smallest correct example.
4. **Trade-off:** Explain complexity, allocation, concurrency, API, or operational cost.
5. **Failure mode:** Name the likely bug and how to test or prevent it.
6. **Version / implementation caveat:** State it only when it changes the answer.

## Important distinction: language guarantees vs implementation details

| Topic | Safe interview phrasing |
|---|---|
| `dict` insertion order | Guaranteed by the language from Python 3.7 onward. |
| Small integer reuse, string interning | CPython implementation details; never use `is` for value comparison. |
| Reference counting and `pymalloc` | CPython memory-management details, not language-wide guarantees. |
| GIL | A CPython build/runtime property; free-threaded CPython builds are available. |
| Bytecode opcodes and object sizes | Vary by Python release, build, architecture, and implementation. |
| Thread/process start method | Depends on OS and Python release; select explicitly when behavior matters. |
| Type annotations | Primarily metadata for tools; Python does not automatically enforce them at runtime. |

## Practical completion checklist

- [ ] I can explain each topic without relying on memorized one-line definitions.
- [ ] I can run and modify the examples.
- [ ] I can state the time and space complexity of common operations.
- [ ] I know which objects are mutable, hashable, iterable, and lazy.
- [ ] I can explain exceptions, resource cleanup, and cancellation.
- [ ] I can design a package and install it outside the source tree.
- [ ] I can add types and tests without coupling code to a concrete implementation.
- [ ] I can choose a concurrency model and explain its trade-offs.
- [ ] I can profile a bottleneck and validate an optimization.
- [ ] I can identify unsafe operations at input, filesystem, process, and serialization boundaries.
- [ ] I can describe system boundaries, dependency direction, idempotency, backpressure, observability, and architecture trade-offs.
- [ ] I can distinguish portable Python behavior from CPython-specific behavior.

The repository is a study guide, not a substitute for reading the release notes of the exact Python, pytest, and dependency versions used in a production environment.

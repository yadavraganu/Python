# Python Core — Interview and Engineering Guide

A structured, code-first repository for mastering Python from fundamentals to senior-engineer and architect-level interviews. The guides emphasize how Python behaves, when to use a feature, complexity and performance trade-offs, production failure modes, and the reasoning expected in interviews.

## Start here

1. Read **[00-Interview-Roadmap](Core/00-Interview-Roadmap.md)** for the learning sequence, coverage map, and interview-readiness checklist.
2. Follow the topic guides in order for a first pass.
3. Revisit the advanced sections using the interview questions and practical checklists.
4. Run examples in the Python version used by your target environment. Version-specific syntax and CPython-only details are labelled.

## Guide index

### Foundations and language

- [01 — Data Types](Core/01-Data-Types.md) — values, mutability, identity, hashability, copying, and built-in types.
- [02 — Functions](Core/02-Functions.md) — parameters, arguments, return values, callables, recursion, and function design.
- [03 — Scopes](Core/03-Scopes.md) — namespaces, LEGB, closures' scope rules, `global`, and `nonlocal`.
- [04 — Exceptions](Core/04-Exceptions.md) — exception flow, custom exceptions, chaining, cleanup, and exception groups.
- [05 — One-Liners](Core/05-OneLiners.md) — idiomatic compact Python and when clarity should win over brevity.
- [25 — Language Syntax and Control Flow](Core/25-Language-Syntax-And-Control-Flow.md) — expressions, truthiness, loops, comprehensions, unpacking, pattern matching, and syntax evolution.
- [26 — Strings, Unicode, and Regex](Core/26-Strings-Unicode-And-Regex.md) — text model, encoding, formatting, normalization, and regular expressions.

### Collections, files, and standard library

- [06 — Choosing Collections](Core/06-Choose-Collection.md) — list, tuple, dict, set, deque, heap, bisect, complexity, and memory trade-offs.
- [27 — File I/O and Serialization](Core/27-File-IO-And-Serialization.md) — `pathlib`, text/binary I/O, atomic writes, JSON, CSV, and pickle risks.
- [30 — Standard-Library Toolkit](Core/30-Standard-Library-Toolkit.md) — `collections`, `itertools`, `functools`, dates/timezones, `decimal`, and useful modules.

### Modules and functional programming

- [07 — Modules and Packages](Core/07-Modules-And-Packages.md) — import system, package design, environments, distribution, and plugin entry points.
- [08 — Closures](Core/08-Closures.md) — lexical closures, captured state, and practical patterns.
- [09 — Decorators](Core/09-Decorators.md) — function/class decorators, metadata, composition, and reusable cross-cutting behavior.
- [10 — Iterators and Generators](Core/10-Iterators-Generators.md) — iteration protocol, generator lifecycle, lazy pipelines, and async iteration.

### Object model and design

- [11 — Object-Oriented Programming](Core/11-Object-Oriented-Program.md) — classes, inheritance, composition, SOLID, and object-oriented design.
- [12 — Dunder Methods](Core/12-Dunder-Methods.md) — Python data model, operators, containers, and special-method protocols.
- [13 — Context Managers](Core/13-Context-Manager.md) — deterministic resource cleanup and `with` semantics.
- [14 — Descriptors](Core/14-Python-Descriptors.md) — descriptor protocol, properties, binding, and attribute lookup precedence.
- [15 — Protocols](Core/15-Protocols.md) — duck typing, structural subtyping, and runtime/static protocols.
- [24 — Python Design Patterns](Core/24-Python-Design-Patterns.md) — idiomatic implementations and trade-offs for common patterns.
- [28 — Typing and Static Analysis](Core/28-Typing-And-Static-Analysis.md) — annotations, generics, protocols, `TypedDict`, overloads, and type-checker workflows.
- [29 — Dataclasses](Core/29-Dataclasses.md) — generated methods, mutability, defaults, inheritance, slots, and value objects.

### Runtime, memory, and concurrency

- [16 — Garbage Collection](Core/16-Garbage-Collection.md) — reference counting, cyclic GC, weak references, and object lifetime.
- [17 — Python Internals](Core/17-Internals.md) — import lifecycle, bytecode, frames, attribute lookup, and interpreter execution.
- [18 — Global Interpreter Lock](Core/18-Global-Interpreter-Lock.md) — GIL behavior, free-threaded builds, sub-interpreters, and parallelism.
- [19 — Threading and Multiprocessing](Core/19-Threading-Multiprocessing.md) — synchronization, pools, IPC, start methods, and concurrency choices.
- [20 — Async Programming](Core/20-Async-Programming.md) — event loops, coroutines, tasks, cancellation, structured concurrency, and async I/O.
- [23 — CPython Object Model](Core/23-CPython-Object-Model.md) — CPython object layouts, allocator, dictionaries, lists, and Unicode internals.

### Engineering practice and architecture

- [21 — Python Logging](Core/21-Python-Logger.md) — logger hierarchy, handlers, formatters, structured logging, and operational practice.
- [22 — pytest](Core/22-Pytest.md) — fixtures, parametrization, mocking, plugins, coverage, and CI.
- [31 — Performance and Profiling](Core/31-Performance-And-Profiling.md) — Big-O, benchmarking, CPU profiling, memory profiling, and optimization workflow.
- [32 — Security and Safe Python](Core/32-Security-And-Safe-Python.md) — safe parsing, serialization, subprocesses, secrets, paths, and dependency hygiene.
- [33 — Python Architecture and Design](Core/33-Python-Architecture-And-Design.md) — dependency direction, boundaries, reliability, observability, testing, and senior design interviews.

## Scope and interview depth

| Level | Expected capability |
|---|---|
| Junior | Explain built-in types, functions, control flow, collections, exceptions, modules, basic OOP, and write correct readable code. |
| Mid-level | Apply iterators/generators, decorators, context managers, descriptors, typing, dataclasses, testing, and standard-library tools. |
| Senior | Explain runtime behavior, memory, concurrency, async cancellation, profiling, security, reliability, and maintainable package design. |
| Lead / architect | Evaluate system boundaries, dependency direction, data contracts, concurrency models, backpressure, operability, deployment, and trade-offs. |

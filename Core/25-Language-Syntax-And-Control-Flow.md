# Python Language Syntax and Control Flow

This guide covers language constructs that appear in everyday code and interview questions. It complements data types (`01-Data-Types.md`), functions (`02-Functions.md`), and scope (`03-Scopes.md`).

**Version baseline:** examples use syntax supported by Python 3.12+, unless a version note says otherwise.

## 1. Statements, expressions, names, and assignment

An **expression** produces a value; a **statement** performs an action. A function call can be part of an expression, while `if`, `for`, `class`, and `def` are statements.

Python variables are **names bound to objects**, not boxes that contain values. Assignment binds or rebinds a name; it does not copy an object.

```python
first = [1, 2]
second = first          # both names refer to the same list
second.append(3)
print(first)            # [1, 2, 3]

second = [9]            # rebind second; first is still bound to the old list
print(first, second)    # [1, 2, 3] [9]
```

- `a = b = []` binds both names to one list.
- `a, b = b, a` swaps references using tuple unpacking; no temporary variable is needed.
- `+=` may mutate an object in place when the type implements in-place addition. For immutable types such as `int` and `str`, it creates/rebinds to a new value.
- Assignment does not imply copying. See the aliasing and copy discussion in `01-Data-Types.md`.

## 2. Truthiness and boolean operators

The following values are false in a Boolean context:

- `False`, `None`
- Numeric zero: `0`, `0.0`, `0j`
- Empty strings and bytes: `""`, `b""`
- Empty containers: `[]`, `()`, `{}`, `set()`, `frozenset()`, `range(0)`
- Instances whose `__bool__()` returns `False` or whose `__len__()` returns zero

Most other objects are true. `and` and `or` return one of their operands, not necessarily a `bool`; they short-circuit.

```python
name = ""
display_name = name or "Anonymous"
print(display_name)                 # Anonymous

items = [1, 2, 3]
result = items and items[0]
print(result)                       # 1

def fallback():
    print("fallback evaluated")
    return "default"

value = "configured" or fallback()  # fallback() is not called
print(value)                        # configured
```

Use `is None` to test the `None` singleton. Do not use truthiness when `0`, `False`, `""`, or an empty collection is a valid value:

```python
count = 0
if count is not None:
    print("A count was supplied")   # this branch runs
```

`bool` is a subclass of `int`, so `True == 1` and `False == 0`; this matters when using booleans as dictionary keys.

## 3. Equality, identity, and membership

| Operator | Meaning | Typical use |
|---|---|---|
| `==`, `!=` | Value equality | Compare values. |
| `is`, `is not` | Object identity | Compare with singletons such as `None`. |
| `in`, `not in` | Membership | Test whether an item belongs to a container. |

```python
a = [1, 2]
b = [1, 2]
print(a == b)       # True: equal contents
print(a is b)       # False: distinct list objects
print(2 in a)       # True
print(None is None) # True
```

Never use `is` to compare strings, numbers, or other values. Interning and caching can make identity appear to work accidentally; it is not the value-comparison contract.

## 4. Conditional expressions

The conditional expression has the form `value_if_true if condition else value_if_false`.

```python
age = 20
category = "adult" if age >= 18 else "minor"
```

Use it for a small expression. Nested conditional expressions are usually harder to read than a regular `if` statement.

## 5. Loops and their `else` clauses

A `for` loop iterates over an iterable. A `while` loop repeats while its condition is true. A loop's `else` block runs only when the loop finishes normally, not when it exits through `break`.

```python
def find_first_even(numbers):
    for number in numbers:
        if number % 2 == 0:
            return number
    else:
        print("No even number found")
    return None

print(find_first_even([1, 3, 5]))   # prints the message, then None
print(find_first_even([1, 4, 5]))   # 4
```

The `else` on a loop is most useful for search logic: it means "the loop did not find an item and was not interrupted." Do not confuse it with `if/else`.

- `break` exits the nearest loop and skips that loop's `else`.
- `continue` skips to the next iteration; it does not skip the loop's `else` if the loop eventually finishes normally.
- `pass` is a no-op placeholder.
- `return` exits the current function, including any loop inside it.

## 6. Unpacking and the `*` / `**` operators

Iterable unpacking assigns elements to names. A starred target captures the remaining values in a list.

```python
first, second = (10, 20)
head, *middle, tail = range(6)
print(head, middle, tail)           # 0 [1, 2, 3, 4] 5

a, b = b"OK"                        # bytes unpack to integers
print(a, b)                         # 79 75

def build_user(name, *, active=True, **extra):
    return {"name": name, "active": active, **extra}

defaults = {"active": False, "role": "reader"}
overrides = {"active": True, "region": "west"}
merged = {**defaults, **overrides}  # later values win
print(merged)
# {'active': True, 'role': 'reader', 'region': 'west'}
```

In calls:
- `func(*items)` expands an iterable into positional arguments.
- `func(**mapping)` expands string-keyed mapping entries into keyword arguments.
- Duplicate keyword names supplied by multiple sources raise `TypeError`.

In assignments:
- `*` may appear once in a target list.
- The starred target always receives a list, even when it captures one item.
- The number of non-starred targets must match the corresponding number of values.

## 7. Comprehensions and generator expressions

Comprehensions construct collections from iterables, optionally filtering each item.

```python
numbers = [1, 2, 3, 4, 5]
squares = [n * n for n in numbers]
even_squares = [n * n for n in numbers if n % 2 == 0]
lookup = {n: n * n for n in numbers}
unique_remainders = {n % 3 for n in numbers}

print(squares)             # [1, 4, 9, 16, 25]
print(even_squares)        # [4, 16]
print(lookup[4])           # 16
print(unique_remainders)   # {0, 1, 2}
```

A generator expression uses parentheses and produces values lazily:

```python
total = sum(n * n for n in range(1_000_000))
```

Choose based on the consumer:
- List/set/dict comprehension: materialize a collection now.
- Generator expression: produce values as the consumer asks for them.
- Avoid multiple nested loops or complex side effects inside a comprehension; use a normal loop when it is clearer.

Since Python 3, comprehension loop variables do not leak into the containing scope. A regular `for` loop variable does. See `03-Scopes.md` for the class-body comprehension edge case.

**Python 3.15 note — PEP 798:** list, set, dict, and generator comprehensions can unpack each produced iterable or mapping. For example, `[*row for row in rows]` flattens one level, `{**part for part in mappings}` merges mapping entries, and `(*row for row in rows)` lazily yields the flattened values. These forms are a concise alternative to nested comprehensions or `itertools.chain`. Do not use them in code that must run on Python 3.14 or earlier.

## 8. Assignment expressions (`:=`)

The walrus operator assigns a value as part of an expression. It is useful when a computed value is needed both for a condition and later use.

```python
records = ["ana", "", "raj"]
non_empty = [value for record in records if (value := record.strip())]
print(non_empty)           # ['ana', 'raj']

if (length := len("python")) > 5:
    print(f"Long word: {length}")
```

Use it sparingly. A regular assignment on its own line is often easier to understand. The operator has restrictions in comprehensions and cannot be used in every expression position.

## 9. Structural pattern matching (`match` / `case`)

Structural pattern matching was introduced in Python 3.10. It is not merely a C-style `switch`: patterns can match literals, sequences, mappings, classes, and capture names.

```python
def describe(value):
    match value:
        case 0:
            return "zero"
        case [first, *rest]:
            return f"sequence starts with {first}; {len(rest)} more"
        case {"kind": "event", "id": event_id}:
            return f"event {event_id}"
        case _:
            return "other"

print(describe(0))                         # zero
print(describe([10, 20, 30]))              # sequence starts with 10; 2 more
print(describe({"kind": "event", "id": 7})) # event 7
```

Important rules:
- A bare name such as `case status:` is a **capture pattern**; it binds the value and normally matches anything. Use `case Status.READY:` or a qualified constant to compare against a constant.
- `_` is the wildcard and does not bind a value.
- A guard (`case pattern if condition:`) adds a Boolean condition after the pattern matches.
- The first matching case wins.
- A failed pattern does not guarantee rollback of every name binding in all circumstances; do not rely on side effects from partial matches.
- Use `match` when the shape of the data is central to the branching. For simple value selection, `if` / `elif` may be clearer.

## 10. `enumerate`, `zip`, `any`, `all`, and `sorted`

These built-ins make loops more expressive without manual indexing.

```python
names = ["Ana", "Raj", "Mei"]
for index, name in enumerate(names, start=1):
    print(index, name)

ids = [1, 2, 3]
scores = [80, 90, 85]
for record_id, score in zip(ids, scores, strict=True):
    print(record_id, score)

print(any(score >= 90 for score in scores))  # True
print(all(score >= 0 for score in scores))  # True
print(sorted(names, key=str.casefold))
```

`zip()` stops at the shortest iterable by default. `strict=True` (Python 3.10+) raises `ValueError` when the inputs have different lengths; use it when unequal lengths indicate a data-integrity problem. `any()` and `all()` short-circuit.

## 11. Common operator and syntax traps

- `and` / `or` return operands; use `bool(...)` only when an actual Boolean is required.
- `==` compares values; `is` compares identity.
- `a = b = []` aliases one mutable object.
- `items.sort()` mutates a list and returns `None`; `sorted(items)` returns a new list.
- `list * n` repeats references, not nested objects: `[[]] * 3` contains three references to the same inner list.
- `dict_a | dict_b` returns a merged dictionary (Python 3.9+); the right-hand value wins on duplicate keys.
- `for ... else` executes only when the loop did not `break`.
- A `match` capture name is not a constant comparison.
- Comprehensions have their own scope; regular loop targets do not.
- A generator expression is lazy, but the iterable in its leftmost `for` expression is evaluated immediately.

## 12. Interview questions to practise

1. What is the difference between `is`, `==`, and `in`?
2. Why does `and` return a list or string sometimes instead of `True` / `False`?
3. What is the difference between `sorted(x)` and `x.sort()`?
4. Why does `[[]] * 3` behave unexpectedly when an inner list is modified?
5. What does a loop `else` block mean, and when is it skipped?
6. When should a generator expression be preferred to a list comprehension?
7. How does pattern matching differ from a switch statement?
8. Why might `zip(..., strict=True)` be safer in a data pipeline?

The best answer includes a tiny example, the output or exception, and the practical reason to choose the construct.

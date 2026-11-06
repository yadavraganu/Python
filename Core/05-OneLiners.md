# Python One-Liners Cheat Sheet

<br></br>

## 1. Basics and Quick Expressions

- **Print formatted string (f-string)**
```python
name, n = "Anurag", 3
print(f"user {name} has {n} items")
```
- **Ternary conditional**
```python
res = "OK" if x > 0 else "NOT OK"
```
- **Swap two variables**
```python
a, b = b, a
```
- **Inline loop with index**
```python
for i, v in enumerate(["a", "b", "c"]): print(i, v)
```
- **Enumerate with a custom start**
```python
for i, v in enumerate(["a", "b", "c"], start=1): print(i, v)
```
- **List of squares, with and without a filter**
```python
squares = [i * i for i in range(10)]
even_squares = [i * i for i in range(10) if i % 2 == 0]
```
- **Dict and set comprehensions**
```python
square_map = {i: i * i for i in range(5)}
unique_lengths = {len(w) for w in ["a", "bb", "cc"]}
```
- **Generator (lazy) and `next` with a default**
```python
g = (x * x for x in range(5)); first = next(g, None)
```
- **Repeat a string**
```python
dash = "-" * 40
```
- **Check membership**
```python
found = item in container
```
- **Truthy test, compact**
```python
if items: print("has items")
```
- **Walrus operator: assign inside an expression (3.8+)**
```python
if (n := len(data)) > 10:
    print(f"too long: {n}")
```
- **Chained comparison**
```python
is_mid = 0 < x < 100
```
- **`any` / `all`**
```python
has_neg = any(x < 0 for x in L)
all_pos = all(x > 0 for x in L)
```
- **Star-unpacking**
```python
first, *middle, last = [1, 2, 3, 4, 5]
```
- **One-liner lambda**
```python
square = lambda x: x * x
```

<br></br>

## 2. Lists, Tuples, Sets, Dicts

- **Reverse a list**
```python
rev = L[::-1]
```
- **Flatten one level of nesting**
```python
flat = [y for x in nested for y in x]
```
```python
import itertools
flat = list(itertools.chain.from_iterable(nested))   # same result; flattens exactly ONE level
```
- **Flatten arbitrarily deep nesting (a small recursive generator)**
```python
def flatten(items):
    for item in items:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item

list(flatten([1, [2, [3, [4]]], 5]))   # [1, 2, 3, 4, 5]
```
- **Unique, preserving order**
```python
uniq = list(dict.fromkeys(L))              # relies on dicts keeping insertion order (3.7+)
```
```python
seen = set(); uniq = [x for x in L if not (x in seen or seen.add(x))]   # works on any version
```
- **Chunk a list into n-sized pieces**
```python
chunks = [L[i:i + n] for i in range(0, len(L), n)]
```
```python
import itertools
chunks = list(itertools.batched(L, n))     # 3.12+; yields tuples rather than lists
```
- **Sort by a custom key**
```python
by_len = sorted(words, key=len)
by_second_desc = sorted(pairs, key=lambda p: p[1], reverse=True)
```
- **Sort a dict by its values**
```python
sorted_items = sorted(d.items(), key=lambda kv: kv[1], reverse=True)
```
- **Largest / smallest by a key**
```python
longest = max(words, key=len)
```
- **Top N with `heapq`**
```python
import heapq; top3 = heapq.nlargest(3, L)
```
- **Dict from two lists**
```python
d = dict(zip(keys, values))
```
- **Invert a dict (values must be unique)**
```python
inv = {v: k for k, v in d.items()}
```
- **Merge dicts**
```python
merged = d1 | d2            # 3.9+
merged = {**d1, **d2}       # any 3.5+; on duplicate keys the right-hand side wins in both forms
```
- **Count frequencies**
```python
from collections import Counter
cnt = Counter(L)
top2 = cnt.most_common(2)
```
- **Group values by key (`defaultdict`)**
```python
from collections import defaultdict
dd = defaultdict(list)
for k, v in pairs: dd[k].append(v)
```
- **Set operations**
```python
union = a | b
intersection = a & b
difference = a - b
symmetric_diff = a ^ b
```
- **Group *adjacent* equal items (`itertools.groupby`: sort first for a full grouping)**
```python
import itertools
grouped = {k: list(v) for k, v in itertools.groupby(sorted(L))}
```
- **All combinations / permutations**
```python
import itertools
combos = list(itertools.combinations(L, 2))
perms = list(itertools.permutations(L, 2))
```
- **Rotate a deque**
```python
from collections import deque
dq = deque(L); dq.rotate(1)     # rotate right by 1
```

<br></br>

## 3. Strings, Text and Regex

- **Reverse a string**
```python
s[::-1]
```
- **Split and strip into words**
```python
words = [w.strip() for w in s.split(",")]
```
- **Join a list into a string**
```python
out = " ".join(words)
```
- **Collapse repeated whitespace**
```python
clean = " ".join(text.split())                       # simplest: split() with no argument handles any whitespace
```
```python
import re; clean = re.sub(r"\s+", " ", text).strip()   # regex equivalent
```
- **Find all words with a regex**
```python
import re; words = re.findall(r"\w+", text)
```
- **Extract the first capture safely**
```python
import re
m = re.search(r"id:(\d+)", s)
id_ = m.group(1) if m else None        # `id_` avoids shadowing the built-in `id`
```
- **Check a palindrome (ignoring case and non-alphanumerics)**
```python
import re
t = "".join(re.findall(r"[a-z0-9]", s.lower()))
is_pal = t == t[::-1]
```
- **Format with padding and separators**
```python
f"{value:08d}"       # zero-pad an integer to width 8
f"{value:,.2f}"      # thousands separator, 2 decimals
f"{value:>10}"       # right-align in width 10
```
- **Case and prefix checks**
```python
s.title()
s.startswith("pre"); s.endswith("fix")
```
- **Remove specific characters (`translate` is fast on large strings)**
```python
clean = s.translate(str.maketrans("", "", ",.!?"))
```

<br></br>

## 4. File, OS, Shell and Quick Scripting

- **Read an entire file**
```python
text = open("file.txt").read()
```
- **Read lines, stripped**
```python
lines = [l.rstrip("\n") for l in open("file.txt")]
```
- **Write to a file**
```python
open("out.txt", "w").write(data)
```
- **Count lines in a file**
```python
n_lines = sum(1 for _ in open("file.txt"))
```
- **Tail the last N lines**
```python
from collections import deque
last10 = deque(open("file.txt"), maxlen=10)
```
- **Modern path handling with `pathlib`**
```python
from pathlib import Path
text = Path("file.txt").read_text()        # opens and closes the file for you
Path("out.txt").write_text(data)
py_files = list(Path(".").rglob("*.py"))
```
- **Walk a directory tree**
```python
import os
for root, dirs, files in os.walk("."): print(root, files)
```
- **Glob matching**
```python
import glob; matches = glob.glob("*.csv")
```
- **Read an environment variable with a default**
```python
import os; token = os.environ.get("API_TOKEN", "")
```
- **Run a shell command and capture its output**
```python
import subprocess
out = subprocess.run(["ls", "-1"], capture_output=True, text=True, check=True).stdout
```
- **One-liner HTTP server (shell)**
```bash
python -m http.server 8000
```
- **Read stdin integers and sum them (shell pipe)**
```bash
cat file | python -c "import sys; print(sum(int(x) for x in sys.stdin))"
```

<br></br>

## 5. Networking, Web Requests, JSON, CSV

- **Simple `urllib` GET**
```python
import urllib.request
data = urllib.request.urlopen(url).read().decode()
```
- **`requests` GET (if installed)**
```python
import requests
text = requests.get(url, params={"q": "x"}).text
```
- **Post JSON with `requests`**
```python
import requests
requests.post(url, json={"k": "v"}).status_code
```
- **Reuse a connection with a `Session` (faster for many requests)**
```python
import requests
with requests.Session() as s:
    r = s.get(url)
```
- **Load JSON from a file**
```python
import json; obj = json.load(open("data.json"))
```
- **Dump JSON, pretty-printed**
```python
import json
json.dumps(obj, indent=2, ensure_ascii=False)
```
- **Read CSV as dicts**
```python
import csv; rows = list(csv.DictReader(open("file.csv")))
```
- **Write CSV from a list of dicts**
```python
import csv
with open("out.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=rows[0].keys())
    writer.writeheader(); writer.writerows(rows)
```

<br></br>

## 6. Data Science, Numeric and Small Algorithms

- **Sum, min, max, mean**
```python
import statistics
total = sum(L); mn = min(L); mx = max(L); mean = statistics.mean(L)
```
- **Product of elements**
```python
import math; math.prod(L)
```
- **Running totals**
```python
import itertools; list(itertools.accumulate(L))
```
- **Transpose a matrix**
```python
list(map(list, zip(*matrix)))
```
- **GCD of a list**
```python
import math; math.gcd(*L)            # 3.9+ accepts many arguments
```
```python
import math; from functools import reduce; reduce(math.gcd, L)   # works on older versions too
```
- **Simple prime check**
```python
is_prime = lambda n: n > 1 and all(n % i for i in range(2, int(n ** 0.5) + 1))
```
- **Fibonacci, first n terms (generator)**
```python
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

list(fib(8))   # [0, 1, 1, 2, 3, 5, 8, 13]
```
- **Sum of digits**
```python
digit_sum = sum(map(int, str(n)))
```
- **Apply a function across a list of dicts**
```python
vals = [d["col"] * 2 for d in rows]
```
- **Pandas quick load and preview**
```python
import pandas as pd
df = pd.read_csv("file.csv"); df.head()
```
- **NumPy quick stats**
```python
import numpy as np
arr = np.array(L); arr.mean(), arr.std()
```
- **Normalize a list to 0-1 (guarding the all-equal case)**
```python
mn, mx = min(L), max(L)
norm = [(x - mn) / (mx - mn) if mx > mn else 0.0 for x in L]   # without the guard, [5, 5, 5] raises ZeroDivisionError
```

<br></br>

## 7. Concurrency, Timing, Debugging, Introspection

- **Run a function in a thread**
```python
import threading
threading.Thread(target=fn, args=(a,)).start()
```
- **Thread pool for many small I/O tasks**
```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=4) as ex:
    results = list(ex.map(fn, items))
```
- **Run an async entry point (3.7+)**
```python
import asyncio
asyncio.run(main())
```
- **Measure elapsed time**
```python
import time
t = time.perf_counter(); f(); print(time.perf_counter() - t)
```
- **Micro-benchmark a snippet**
```python
import timeit
timeit.timeit("sum(range(100))", number=10000)
```
- **Cache expensive calls (memoization from the standard library)**
```python
from functools import lru_cache

@lru_cache(maxsize=None)
def slow_fn(n):
    return n * n
```
- **Pretty-print a structure**
```python
import pprint; pprint.pprint(obj)
```
- **List the callable members of an object**
```python
[k for k in dir(obj) if callable(getattr(obj, k))]
```
- **Dynamic attribute set / get**
```python
setattr(obj, "name", value)
getattr(obj, "name", default)
```
- **Instantiate with a kwargs dict**
```python
obj = MyClass(**kwargs)
```
- **Quick logging instead of `print`**
```python
import logging
logging.basicConfig(level=logging.INFO)
logging.info("started with n=%s", n)
```
- **`assert` for a quick sanity check**
```python
assert len(L) > 0, "L must not be empty"      # stripped entirely under `python -O`, so never use it for input validation
```

<br></br>

## Favorite Gotchas to Remember

- `chain.from_iterable` flattens **exactly one level**. For arbitrary depth you need recursion (the `flatten` generator above).
- `sorted(d)` sorts only the **keys**; use `sorted(d.items(), key=...)` to sort by value.
- `dict.fromkeys(L)` deduplicates in order only because dicts preserve insertion order (guaranteed since 3.7).
- `itertools.groupby` groups only **adjacent** equal keys: on unsorted `[3, 1, 2, 3, 1]` it yields five groups, not three. Sort first.
- `heapq.nlargest(k, L)` is O(n log k), better than `sorted(L)[-k:]` when `k` is small relative to `n`.
- `lru_cache` needs **hashable** arguments; passing a list raises `TypeError`.
- `statistics.mean([])` raises `StatisticsError`, and min-max normalization divides by zero when all values are equal, so guard both.
- A palindrome check that filters with `[a-z]` silently drops digits; use `[a-z0-9]` if digits count.
- `assert` is for invariants you believe are always true, not for validating input: it disappears under `python -O`.

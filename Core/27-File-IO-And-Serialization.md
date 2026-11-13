# File I/O, Paths, and Serialization

Reliable file handling requires explicit encoding, deterministic cleanup, clear failure behavior, and a serialization format chosen for the data. This guide focuses on the standard library.

## 1. Use `pathlib.Path` for filesystem paths

`Path` represents a filesystem path and provides readable, cross-platform operations. It does not make filesystem access immune to permission errors, races, or untrusted paths.

```python
from pathlib import Path

project = Path("project")
data_dir = project / "data"
data_dir.mkdir(parents=True, exist_ok=True)

file_path = data_dir / "users.txt"
print(file_path.name)       # users.txt
print(file_path.suffix)     # .txt
print(file_path.parent)     # project/data
print(file_path.exists())   # False until created
```

Useful operations:
- `Path.cwd()` and `Path.home()`
- `Path.resolve()` for an absolute, normalized path (resolution may access the filesystem).
- `Path.mkdir(parents=True, exist_ok=True)`
- `Path.iterdir()`, `Path.glob("*.csv")`, `Path.rglob("*.py")`
- `Path.read_text()` / `Path.write_text()` for small text files.
- `Path.read_bytes()` / `Path.write_bytes()` for bytes.
- `Path.unlink(missing_ok=True)` and `Path.rename()`.

Use `Path` for path composition, but use a context manager with `open()` when you need streaming, explicit buffering, newline handling, or precise control.

## 2. `open()` and file modes

Prefer the explicit form:

```python
from pathlib import Path

path = Path("notes.txt")
with path.open("w", encoding="utf-8", newline="\n") as file:
    file.write("Python\n")
    file.write("file I/O\n")

with path.open("r", encoding="utf-8") as file:
    contents = file.read()

print(contents)
```

Common modes:

| Mode | Meaning |
|---|---|
| `r` | Read text; file must exist. |
| `w` | Write text; truncate an existing file or create a new one. |
| `a` | Append text; create if absent. |
| `x` | Exclusive create; fail if the path already exists. |
| `b` | Binary mode, e.g. `rb`, `wb`. No text encoding/decoding. |
| `t` | Text mode (default), e.g. `rt`, `wt`. |
| `+` | Update mode (read and write), e.g. `r+`, `w+`. |

Important:
- Text mode translates bytes to/from `str` using an encoding and newline policy.
- Binary mode reads/writes `bytes` and does not accept `encoding` or `newline`.
- `w` truncates. Use `x` when overwriting an existing file must be prevented.
- A file object maintains a cursor; `seek()` moves it and `tell()` reports its position.
- `read()` with no size reads the rest of the file into memory. For large files, iterate line-by-line or in chunks.
- A `with` statement closes the file even when the block raises.

## 3. Streaming large files

Avoid loading a huge file into memory if it can be processed incrementally.

```python
from pathlib import Path

def count_non_empty_lines(path: Path) -> int:
    count = 0
    with path.open("r", encoding="utf-8") as file:
        for line in file:
            if line.strip():
                count += 1
    return count
```

For fixed-size binary chunks:

```python
def copy_in_chunks(source, destination, chunk_size=1024 * 1024):
    while chunk := source.read(chunk_size):
        destination.write(chunk)
```

The walrus operator requires Python 3.8+. Chunk size should be selected by measurement and workload rather than treated as a universal constant.

## 4. JSON: interoperable text data

JSON is a good default for interoperable data, configuration, and APIs. It supports a limited set of value types: objects, arrays, strings, numbers, booleans, and `null`. Python-specific objects such as `datetime`, `Decimal`, sets, or custom classes need an explicit conversion policy.

```python
import json

record = {"id": 7, "name": "Ana", "active": True, "tags": ["python", "data"]}
encoded = json.dumps(record, ensure_ascii=False, indent=2)
decoded = json.loads(encoded)

print(decoded == record)        # True
```

File operations:

```python
import json
from pathlib import Path

path = Path("config.json")
config = {"retries": 3, "timeout_seconds": 5}

with path.open("w", encoding="utf-8") as file:
    json.dump(config, file, indent=2)

with path.open("r", encoding="utf-8") as file:
    loaded = json.load(file)
```

Useful options:
- `ensure_ascii=False` writes readable non-ASCII characters.
- `indent=2` formats for humans.
- `sort_keys=True` can make output deterministic for diffs.
- `allow_nan=False` rejects non-standard NaN/Infinity tokens.
- `object_hook` / `parse_float` can customize decoding.

For untrusted input, catch `json.JSONDecodeError` and validate the decoded structure, field types, ranges, and size. Valid JSON is not automatically valid application data.

## 5. CSV: tabular text

Use the `csv` module instead of manually splitting on commas. CSV supports quoting, embedded delimiters, and escaped fields.

```python
import csv
from pathlib import Path

path = Path("users.csv")
rows = [
    {"id": 1, "name": "Ana, Marie"},
    {"id": 2, "name": "Raj"},
]

with path.open("w", encoding="utf-8", newline="") as file:
    writer = csv.DictWriter(file, fieldnames=["id", "name"])
    writer.writeheader()
    writer.writerows(rows)

with path.open("r", encoding="utf-8", newline="") as file:
    reader = csv.DictReader(file)
    loaded = list(reader)

print(loaded)
# [{'id': '1', 'name': 'Ana, Marie'}, {'id': '2', 'name': 'Raj'}]
```

CSV fields are text. Convert values explicitly and validate them at the boundary. `newline=""` is the recommended way to open CSV files so the CSV module controls newline handling.

## 6. Binary files and the buffer protocol

Binary I/O returns bytes and is suitable for images, compressed data, protocol payloads, and arbitrary byte streams.

```python
payload = b"\x00\x01\x02"
with open("payload.bin", "wb") as file:
    file.write(payload)

with open("payload.bin", "rb") as file:
    restored = file.read()

assert restored == payload
```

`memoryview` provides a view over a buffer-protocol object without copying the underlying bytes for every slice. It is useful for performance-sensitive binary parsing; understand the lifetime and mutability of the underlying buffer before using it.

## 7. Temporary files and directories

Use `tempfile` rather than guessing a temporary filename. The module creates unique names safely and provides cleanup helpers.

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as temp_dir:
    workdir = Path(temp_dir)
    output = workdir / "result.txt"
    output.write_text("temporary", encoding="utf-8")
    assert output.read_text(encoding="utf-8") == "temporary"
# The temporary directory is removed here.
```

For a temporary file that must be atomically moved into place, create it in the target directory (same filesystem), close it, and then use `os.replace()`.

## 8. Atomic file replacement

Writing directly to a destination can leave a partially written file if the process crashes. A safer pattern is to write a temporary file in the same directory and replace the destination after the write completes.

```python
import os
import tempfile
from pathlib import Path

def atomic_write_text(destination: Path, content: str) -> None:
    destination = Path(destination)
    destination.parent.mkdir(parents=True, exist_ok=True)

    fd, temporary_name = tempfile.mkstemp(
        dir=destination.parent,
        prefix=f".{destination.name}.",
        suffix=".tmp",
    )
    try:
        with os.fdopen(fd, "w", encoding="utf-8", newline="\n") as file:
            file.write(content)
            file.flush()
            os.fsync(file.fileno())
        os.replace(temporary_name, destination)
    except BaseException:
        try:
            os.unlink(temporary_name)
        except FileNotFoundError:
            pass
        raise
```

`os.replace()` is atomic when the source and destination are on the same filesystem and the platform supports atomic replacement. This provides atomic visibility to readers; it is not a complete durability protocol for every filesystem or sudden power-loss scenario. Production systems may also need to sync the containing directory and define permissions/ownership.

## 9. Pickle: Python object serialization

`pickle` can serialize many Python-specific objects and is convenient for trusted internal data or Python process communication. It is **not safe for untrusted input**: unpickling can execute arbitrary code.

```python
import pickle

payload = pickle.dumps({"count": 3})
restored = pickle.loads(payload)   # only unpickle data from a trusted source
```

Rules:
- Never unpickle user uploads, internet downloads, untrusted caches, or files from an unknown source.
- A signature or checksum alone does not make an untrusted pickle safe unless authenticity and key handling are correctly designed.
- Pickle is Python-specific and generally unsuitable as a long-term, cross-language interchange format.
- Pickled classes are coupled to importable module paths and compatible class definitions.
- For interoperable data, prefer JSON, CSV, or a documented schema format such as Protocol Buffers or Avro where appropriate.

## 10. Common exceptions and robust handling

| Exception | Typical cause |
|---|---|
| `FileNotFoundError` | Source path or parent directory does not exist. |
| `PermissionError` | Current user/process lacks permission. |
| `IsADirectoryError` / `NotADirectoryError` | A path component has the wrong kind. |
| `FileExistsError` | Exclusive creation or directory creation collided with an existing path. |
| `UnicodeDecodeError` | Bytes are invalid for the chosen text encoding. |
| `UnicodeEncodeError` | Text cannot be encoded with the chosen codec/error policy. |
| `json.JSONDecodeError` | Input is not valid JSON. |
| `csv.Error` | CSV parser/writer reports malformed input or output failure. |
| `OSError` | General operating-system I/O failure. |

Catch the narrowest useful exception and preserve context. Do not catch `OSError` and silently continue unless the failure is explicitly safe to ignore.

## 11. Common traps

- Omitting `encoding=` makes behavior environment-dependent on Python versions before 3.15 and when UTF-8 mode is disabled.
- `w` truncates an existing file.
- `read()` can exhaust memory on large files.
- CSV is not safely parsed by `line.split(",")`.
- JSON decoding does not validate application-level types or business rules.
- `pickle.loads()` on untrusted bytes can execute code.
- `Path.resolve()` is not a complete defense against filesystem race conditions or symlink replacement.
- A temporary file should be created in the target directory when relying on same-filesystem atomic replace.
- Always use context managers for resource ownership.

## 12. Practice questions

1. What is the difference between text mode and binary mode?
2. Why should a text file's encoding be explicit?
3. Why does the CSV module require `newline=""`?
4. How do you process a very large file without loading it all into memory?
5. When should JSON be preferred to pickle?
6. Why is unpickling untrusted data dangerous?
7. What does atomic file replacement guarantee—and what does it not guarantee?
8. Which exceptions would you catch when reading a configuration file?

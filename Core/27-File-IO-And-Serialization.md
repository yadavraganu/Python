# File I/O, Paths, And Serialization

Reliable file handling is more than opening a file and reading its contents. Production-quality code must choose the right path API, understand text and binary modes, manage resources, handle large inputs without exhausting memory, validate serialized data, and define what should happen when an operation fails halfway through.

## 1. Understand Filesystem Paths

A path identifies a location in a filesystem. It is not the file itself, and constructing a path does not guarantee that the target exists or that the current process can access it.

### 1.1 Prefer `pathlib.Path`

`pathlib.Path` provides an object-oriented, cross-platform way to compose and inspect paths.

```python
# Demonstrate: Prefer pathlib.Path.
from pathlib import Path

project_dir = Path("project")
data_dir = project_dir / "data"
data_dir.mkdir(parents=True, exist_ok=True)

file_path = data_dir / "users.txt"

print(file_path.name)       # users.txt
print(file_path.stem)       # users
print(file_path.suffix)     # .txt
print(file_path.parent)     # project/data
print(file_path.exists())   # False until created
```

The `/` operator joins path components. It does not perform string concatenation and handles platform-specific separators.

Useful properties and methods:

| API | Purpose |
|---|---|
| `Path.cwd()` | Current working directory |
| `Path.home()` | Current user's home directory |
| `Path.name` | Final path component |
| `Path.stem` | Filename without its final suffix |
| `Path.suffix` | Final suffix, including the dot |
| `Path.suffixes` | All suffixes, e.g. `[".tar", ".gz"]` |
| `Path.parent` | Parent path |
| `Path.parts` | Components of the path |
| `Path.exists()` | Whether the path exists |
| `Path.is_file()` / `is_dir()` | Whether it refers to a file or directory |
| `Path.mkdir()` | Create a directory |
| `Path.iterdir()` | Iterate over immediate children |
| `Path.glob()` / `rglob()` | Match paths using patterns |
| `Path.read_text()` / `write_text()` | Convenient whole-file text operations |
| `Path.read_bytes()` / `write_bytes()` | Convenient whole-file binary operations |
| `Path.unlink()` | Remove a file or symbolic link |
| `Path.rename()` / `replace()` | Rename or replace a path |
| `Path.stat()` | Retrieve filesystem metadata |
| `Path.resolve()` | Resolve to an absolute path, including symlinks where possible |

Use `Path` for path composition and filesystem operations. Use `Path.open()` or built-in `open()` when you need streaming, explicit buffering, newline control, or fine-grained file handling.

### 1.2 Relative And Absolute Paths

A relative path is interpreted relative to the process's current working directory, not necessarily the directory containing the Python script.

```python
# Demonstrate: Relative And Absolute Paths.
from pathlib import Path

relative = Path("data/input.csv")
absolute = relative.resolve()

print("Working directory:", Path.cwd())
print("Resolved path:", absolute)
```

This distinction matters when code is launched from an IDE, scheduler, service manager, notebook, or a different directory.

For application resources, construct paths from a known project or module location rather than assuming the current working directory.

```python
# Demonstrate: Relative And Absolute Paths.
from pathlib import Path

MODULE_DIR = Path(__file__).resolve().parent
CONFIG_PATH = MODULE_DIR / "config" / "settings.json"
```

`__file__` is not defined in every execution environment, such as some interactive notebooks. Choose the base directory deliberately for the environment in which the code runs.

### 1.3 Create Directories Safely

```python
# Demonstrate: Create Directories Safely.
from pathlib import Path

output_dir = Path("output")
output_dir.mkdir(parents=True, exist_ok=True)
```

- `parents=True` creates missing parent directories.
- `exist_ok=True` avoids an error when the directory already exists.
- Without `exist_ok=True`, an existing directory normally causes `FileExistsError`.
- If a regular file exists at the requested directory path, creation still fails.

### 1.4 Inspect And Enumerate Paths

```python
# Demonstrate: Inspect And Enumerate Paths.
from pathlib import Path

data_dir = Path("data")

for path in data_dir.iterdir():
    if path.is_file():
        print(path.name, path.stat().st_size)
```

Pattern matching:

```python
# Demonstrate: Inspect And Enumerate Paths.
for csv_path in data_dir.glob("*.csv"):
    print(csv_path)

for python_file in Path(".").rglob("*.py"):
    print(python_file)
```

For very large directories, `os.scandir()` can be more efficient than repeatedly asking for file metadata because directory entries may expose cached metadata.

```python
# Demonstrate: Inspect And Enumerate Paths.
import os

with os.scandir("data") as entries:
    for entry in entries:
        if entry.is_file():
            print(entry.name, entry.stat().st_size)
```

Do not assume directory iteration order is stable. Sort results when deterministic processing is required.

### 1.5 Paths Are Not Security Boundaries

Joining an untrusted path fragment to a trusted directory does not automatically keep the result inside that directory.

```python
# Demonstrate: Paths Are Not Security Boundaries.
from pathlib import Path

base = Path("/srv/uploads")
candidate = base / user_supplied_name
```

A value such as `../../etc/passwd` may escape the intended directory. Resolve and validate paths when accepting untrusted path components, and consider symlink and race-condition risks. A simple string-prefix check is not a safe containment check.

```python
# Demonstrate: Paths Are Not Security Boundaries.
base = Path("/srv/uploads").resolve()
candidate = (base / user_supplied_name).resolve()

try:
    candidate.relative_to(base)
except ValueError:
    raise ValueError("Path escapes the upload directory")
```

This check is useful but does not eliminate time-of-check/time-of-use races if an attacker can modify directories or symlinks concurrently. High-security file handling may require directory-descriptor-based operations and operating-system-specific protections.

## 2. Open Files And Understand File Modes

`open()` returns a file object. The object manages a stream and maintains state such as the current cursor position and buffering.

### 2.1 Basic File Handling

```python
# Demonstrate: Basic File Handling.
from pathlib import Path

path = Path("notes.txt")

with path.open("w", encoding="utf-8", newline="\n") as file:
    file.write("Python\n")
    file.write("file I/O\n")

with path.open("r", encoding="utf-8") as file:
    contents = file.read()

print(contents)
```

The `with` statement uses the file object's context-manager protocol. The file is closed when execution leaves the block, including when an exception is raised.

Closing a file releases its associated operating-system resource and flushes Python's buffered output as appropriate. Do not rely on garbage collection to close files promptly.

### 2.2 File Modes

| Mode | Meaning |
|---|---|
| `r` | Read; file must exist |
| `w` | Write; create or truncate the file |
| `a` | Append; create if absent, writes go to the end |
| `x` | Exclusive creation; fail if the path already exists |
| `b` | Binary mode modifier |
| `t` | Text mode modifier; default |
| `+` | Enable both reading and writing |

Common combinations:

| Mode | Behavior |
|---|---|
| `rb` | Read bytes |
| `wb` | Write bytes and truncate/create |
| `ab` | Append bytes |
| `xb` | Exclusively create a binary file |
| `r+` | Read and write without initially truncating |
| `w+` | Read and write, truncating first |
| `a+` | Read and append; writes go to the end |

Important distinctions:

- `w` destroys the previous contents immediately when the file is opened successfully.
- `x` is useful when overwriting an existing file must be prevented.
- `a` is intended for append-style writes. The stream's write position is moved to the end for writes.
- `r+` requires the file to exist.
- `w+` truncates the file.
- `b` and `t` are mutually exclusive.
- `encoding` and `newline` are text-mode options; they must not be supplied for binary mode.

### 2.3 Text Mode Versus Binary Mode

Text mode converts between bytes on disk and Python `str` objects using an encoding and newline policy.

```python
# Demonstrate: Text Mode Versus Binary Mode.
with open("message.txt", "w", encoding="utf-8") as file:
    file.write("Hello, नमस्ते")
```

Binary mode reads and writes `bytes` without text decoding or newline translation.

```python
# Demonstrate: Text Mode Versus Binary Mode.
with open("image.bin", "rb") as file:
    payload = file.read()
```

Use text mode for textual formats such as JSON, CSV, and ordinary logs. Use binary mode for images, compressed files, executable files, network payloads, and any format whose bytes must be preserved exactly.

### 2.4 The File Cursor

A file object tracks a current position, often called the cursor or file pointer.

```python
# Demonstrate: The File Cursor.
with open("sample.txt", "w+", encoding="utf-8") as file:
    file.write("abcdef")
    print(file.tell())  # Current position after writing

    file.seek(0)
    print(file.read(3))  # abc
    print(file.tell())   # 3
```

- `tell()` reports the current stream position.
- `seek(offset, whence)` changes the position.
- `whence=0` means relative to the beginning.
- `whence=1` means relative to the current position.
- `whence=2` means relative to the end.

In binary files, seeking by byte offset is generally straightforward. In text files, the position returned by `tell()` is an opaque cookie intended to be passed back to `seek()`; it should not generally be treated as a byte offset because text decoding and newline translation complicate the mapping.

## 3. Text Encoding And Newline Handling

Encoding errors are a frequent source of production bugs, especially when files move between operating systems or systems with different locale settings.

### 3.1 Use Explicit Encodings

```python
# Demonstrate: Use Explicit Encodings.
with open("input.txt", "r", encoding="utf-8") as file:
    text = file.read()
```

UTF-8 is a strong default for newly created text files and interoperable data. Omitting `encoding` can make behavior dependent on the environment and Python's UTF-8 mode. Python 3.15 changes the default text encoding to UTF-8; code that supports multiple Python versions or environments should still specify the encoding explicitly.

Common encodings include:

- `utf-8`: common interoperable Unicode encoding.
- `utf-8-sig`: UTF-8 with a byte-order mark handled specially when reading and written when encoding.
- `utf-16`: Unicode encoding with byte-order handling.
- `latin-1`: maps every byte value to a code point, but does not mean arbitrary bytes are correctly interpreted as the intended text.

Do not choose an encoding merely because it suppresses an exception. Identify the source encoding whenever possible.

### 3.2 Encoding And Decoding Errors

```python
# Demonstrate: Encoding And Decoding Errors.
raw = b"\xff\xfe"

try:
    text = raw.decode("utf-8")
except UnicodeDecodeError as error:
    print("Invalid UTF-8:", error)
```

The `errors` parameter controls how encoding/decoding errors are handled:

- `strict` (default): raise an exception.
- `ignore`: discard problematic data; this can cause silent data loss.
- `replace`: substitute a replacement character.
- `backslashreplace`: represent problematic characters using escape sequences during encoding.
- `surrogateescape`: preserve undecodable bytes in a reversible way for certain low-level workflows.

For business data, `strict` is usually preferable unless there is a documented recovery policy.

### 3.3 Newline Translation

Text mode can translate newline sequences. On reading, the default `newline=None` enables universal newline handling, recognizing `\n`, `\r\n`, and `\r` and translating them to `\n`.

On writing with `newline=None`, Python translates `\n` to the platform's default line separator. Use `newline="\n"` when a format requires a specific line ending.

CSV is a special case: open CSV files with `newline=""` so the `csv` module controls newline handling.

```python
# Demonstrate: Newline Translation.
with open("report.txt", "w", encoding="utf-8", newline="\n") as file:
    file.write("first line\nsecond line\n")
```

## 4. Read And Write Efficiently

The right reading strategy depends on file size, access pattern, and whether the full content is required at once.

### 4.1 Read The Whole File

```python
# Demonstrate: Read The Whole File.
from pathlib import Path

text = Path("small.txt").read_text(encoding="utf-8")
```

This is concise and appropriate for small files. It loads the entire file into memory, so it is not the default choice for very large inputs.

### 4.2 Read Line By Line

```python
# Demonstrate: Read Line By Line.
def count_non_empty_lines(path):
    count = 0

    with open(path, "r", encoding="utf-8") as file:
        for line in file:
            if line.strip():
                count += 1

    return count
```

Iteration over a text file is memory-efficient because it reads incrementally. The current line is held in memory rather than the entire file.

Avoid `readlines()` for large files because it creates a list containing every line.

### 4.3 Read In Fixed-Size Chunks

Chunked reading is useful for large files, streaming transformations, and binary data.

```python
# Demonstrate: Read In Fixed-Size Chunks.
def copy_in_chunks(source, destination, chunk_size=1024 * 1024):
    while True:
        chunk = source.read(chunk_size)
        if not chunk:
            break
        destination.write(chunk)
```

The walrus-operator version requires Python 3.8+:

```python
# Demonstrate: Read In Fixed-Size Chunks.
def copy_in_chunks(source, destination, chunk_size=1024 * 1024):
    while chunk := source.read(chunk_size):
        destination.write(chunk)
```

Choose chunk sizes through measurement. A larger chunk can reduce Python-level loop overhead but increases per-operation memory use. The best value depends on storage, workload, and buffering.

### 4.4 Copy Files With `shutil`

For ordinary file copying, use the standard library instead of writing a custom copy loop.

```python
# Demonstrate: Copy Files With shutil.
import shutil

shutil.copyfile("source.bin", "destination.bin")
shutil.copy("source.bin", "backup/source.bin")
shutil.copy2("source.bin", "backup/source.bin")
```

- `copyfile()` copies file contents.
- `copy()` copies contents and permission mode.
- `copy2()` attempts to preserve additional metadata such as timestamps.

For directories, use `shutil.copytree()`. For moving paths, use `shutil.move()`. Metadata preservation varies by platform and filesystem.

### 4.5 Buffered I/O

Python file objects are generally buffered by default. Buffering reduces the number of lower-level I/O operations by collecting reads or writes.

The `buffering` argument to `open()` controls buffering:

- `-1`: default buffering.
- `0`: unbuffered binary I/O only.
- `1`: line buffering in text mode.
- A positive integer: use a buffer of approximately that size.

Most application code should use the defaults. Disable or tune buffering only when the access pattern and measurements justify it.

### 4.6 Flush Versus Close

- `flush()` pushes Python's buffered data to the underlying stream.
- `close()` flushes and closes the stream.
- `os.fsync()` requests that the operating system flush file data to stable storage, subject to filesystem and hardware guarantees.

```python
# Demonstrate: Flush Versus Close.
import os

with open("output.txt", "w", encoding="utf-8") as file:
    file.write("important data")
    file.flush()
    os.fsync(file.fileno())
```

`flush()` alone does not guarantee that data has reached durable storage. `fsync()` is more demanding but still does not provide an absolute guarantee against every hardware or filesystem failure.

## 5. Binary I/O And The Buffer Protocol

Binary I/O is used whenever data should be treated as bytes rather than decoded text.

```python
# Demonstrate: Binary I/O And The Buffer Protocol.
payload = b"\x00\x01\x02"

with open("payload.bin", "wb") as file:
    file.write(payload)

with open("payload.bin", "rb") as file:
    restored = file.read()

assert restored == payload
```

### 5.1 `bytes`, `bytearray`, And `memoryview`

- `bytes` is immutable.
- `bytearray` is mutable.
- `memoryview` exposes a view over an object's buffer without copying the underlying data for each slice.

```python
# Demonstrate: bytes, bytearray, And memoryview.
data = bytearray(b"abcdef")
view = memoryview(data)

print(view[1:4].tobytes())  # b'bcd'
view[0] = ord("X")

print(data)                 # bytearray(b'Xbcdef')
```

A `memoryview` can be useful for performance-sensitive binary parsing. Be aware that a view may keep the underlying buffer alive, and writable views can mutate the original object.

### 5.2 Memory-Mapped Files

`mmap` maps a file into a process's address space. It can be useful for random access to large files without explicitly reading the entire file into a Python bytes object.

```python
# Demonstrate: Memory-Mapped Files.
import mmap

with open("large.bin", "r+b") as file:
    with mmap.mmap(file.fileno(), length=0) as mapped:
        first_bytes = mapped[:16]
        print(first_bytes)
```

Memory mapping is not automatically faster. Performance depends on access patterns, operating-system paging, file size, and storage. It also introduces considerations around file lifetime, concurrent modification, and platform-specific behavior.

## 6. Temporary Files And Directories

Use `tempfile` rather than constructing a temporary filename yourself. Guessable names can introduce collisions and security vulnerabilities.

### 6.1 Temporary Directory

```python
# Demonstrate: Temporary Directory.
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as temp_dir:
    work_dir = Path(temp_dir)
    output = work_dir / "result.txt"
    output.write_text("temporary", encoding="utf-8")
    print(output.read_text(encoding="utf-8"))

# The Temporary Directory And Its Contents Are Cleaned Up Here.
```

### 6.2 Temporary File

```python
# Demonstrate: Temporary File.
from tempfile import TemporaryFile

with TemporaryFile(mode="w+t", encoding="utf-8") as file:
    file.write("temporary content")
    file.seek(0)
    print(file.read())
```

Other useful APIs include:

- `NamedTemporaryFile()`: creates a temporary file with a visible filesystem name.
- `TemporaryDirectory()`: creates a temporary directory and cleans it up.
- `mkstemp()`: low-level creation that returns an open file descriptor and a path.
- `mkdtemp()`: low-level temporary-directory creation.

On some platforms, reopening a `NamedTemporaryFile` by name while it remains open has different behavior. Account for platform semantics when sharing the temporary path with another process or library.

## 7. Atomic File Replacement

Writing directly to a destination can leave a partially written file if a process crashes or an exception occurs. A common safer pattern is:

1. Create a temporary file in the destination directory.
2. Write the complete new content.
3. Flush and, when required, sync the temporary file.
4. Close it.
5. Replace the destination using `os.replace()`.

```python
# Demonstrate: Atomic File Replacement.
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

Why create the temporary file in the same directory?

- Replacement is generally atomic only when source and destination are on the same filesystem and the platform supports the operation.
- A temporary file in a system-wide temporary directory may be on another filesystem.
- `os.replace()` replaces an existing destination where permitted; it is not the same as exclusive creation.

Atomic replacement provides atomic visibility to readers: they should see the old file or the new file, rather than a partially written intermediate file. It is not a complete durability protocol for every filesystem or power-loss scenario. Some applications also need to sync the containing directory after replacement and define ownership and permissions.

Do not catch `BaseException` in ordinary application error handling. The example uses it only to attempt temporary-file cleanup before re-raising every interruption or failure.

## 8. JSON Serialization

JSON is a text-based, language-independent format widely used for APIs, configuration, and data exchange.

### 8.1 Supported JSON Values

JSON directly represents:

- Object: Python `dict` with supported key types.
- Array: Python `list`.
- String: Python `str`.
- Number: Python `int` and `float`.
- Boolean: Python `True` and `False`.
- Null: Python `None`.

Python-specific types such as `datetime`, `Decimal`, `set`, `bytes`, and custom class instances need an explicit conversion policy.

### 8.2 Serialize And Deserialize

```python
# Demonstrate: Serialize And Deserialize.
import json

record = {
    "id": 7,
    "name": "Ana",
    "active": True,
    "tags": ["python", "data"],
}

encoded = json.dumps(record, ensure_ascii=False, indent=2)
decoded = json.loads(encoded)

assert decoded == record
```

- `json.dumps()` converts a Python object to a JSON string.
- `json.loads()` parses a JSON string into Python objects.
- `json.dump()` writes JSON to a file-like object.
- `json.load()` reads JSON from a file-like object.

```python
# Demonstrate: Serialize And Deserialize.
import json

config = {"retries": 3, "timeout_seconds": 5}

with open("config.json", "w", encoding="utf-8") as file:
    json.dump(config, file, indent=2)

with open("config.json", "r", encoding="utf-8") as file:
    loaded = json.load(file)
```

### 8.3 Important JSON Options

| Option | Purpose |
|---|---|
| `ensure_ascii=False` | Write non-ASCII characters directly instead of escaping them |
| `indent=2` | Pretty-print with indentation |
| `sort_keys=True` | Sort object keys for more stable output |
| `allow_nan=False` | Reject `NaN` and infinity values that are outside strict JSON |
| `default=` | Convert otherwise unsupported Python objects |
| `object_hook=` | Transform decoded dictionaries |
| `parse_float=` | Customize parsing of JSON floating-point values |
| `parse_int=` | Customize parsing of JSON integers |
| `parse_constant=` | Customize handling of non-standard constants |

### 8.4 Serialize Custom Python Types

A `default` function can convert unsupported values into JSON-compatible values.

```python
# Demonstrate: Serialize Custom Python Types.
import json
from datetime import datetime, timezone


def json_default(value):
    if isinstance(value, datetime):
        return value.isoformat()
    raise TypeError(f"Unsupported type: {type(value).__name__}")


record = {
    "created_at": datetime.now(timezone.utc),
}

encoded = json.dumps(record, default=json_default)
print(encoded)
```

A conversion function should have a documented policy. For example, converting a `Decimal` to `float` may lose precision. Converting a `set` to a list may lose ordering. A decoder also needs a deliberate strategy if the original Python type must be reconstructed.

### 8.5 JSON Lines (NDJSON)

A JSON document normally contains one JSON value. JSON Lines stores one independent JSON value per line and is useful for logs and streaming data pipelines.

```python
# Demonstrate: JSON Lines (NDJSON).
import json

records = [
    {"id": 1, "event": "created"},
    {"id": 2, "event": "updated"},
]

with open("events.jsonl", "w", encoding="utf-8") as file:
    for record in records:
        file.write(json.dumps(record) + "\n")

with open("events.jsonl", "r", encoding="utf-8") as file:
    for line in file:
        if line.strip():
            record = json.loads(line)
            print(record)
```

JSON Lines supports incremental processing because each record can be parsed independently. A malformed line can be isolated, but the application must define whether to stop, skip, quarantine, or report it.

### 8.6 Validate Decoded JSON

Successful JSON parsing only proves that the input is syntactically valid JSON. It does not prove that the decoded value has the expected structure or valid business values.

```python
# Demonstrate: Validate Decoded JSON.
import json

with open("config.json", encoding="utf-8") as file:
    config = json.load(file)

if not isinstance(config, dict):
    raise ValueError("Configuration must be a JSON object")

retries = config.get("retries")

if not isinstance(retries, int) or retries < 0:
    raise ValueError("'retries' must be a non-negative integer")
```

For untrusted input, also consider maximum input size, nesting depth, allowed fields, numeric ranges, and resource consumption.

### 8.7 JSON Limitations

- JSON does not preserve arbitrary Python types.
- Standard JSON has no native date, decimal, binary, or set type.
- JSON object keys are strings in the format; Python's encoder accepts certain non-string key types and converts them, which can create key collisions.
- JSON is not a framed protocol. Repeatedly calling `json.dump()` on the same stream does not create one valid JSON document containing multiple values. Use an array or JSON Lines.
- JSON parsing loads a complete JSON value into memory with the standard `json` API. For very large data, use a streaming format or process line-delimited records.

## 9. CSV Serialization

CSV is a text format for tabular data. Use the `csv` module rather than splitting lines on commas because fields can contain delimiters, quotes, and embedded newlines.

### 9.1 Write And Read Dictionaries

```python
# Demonstrate: Write And Read Dictionaries.
import csv

rows = [
    {"id": 1, "name": "Ana, Marie"},
    {"id": 2, "name": "Raj"},
]

with open("users.csv", "w", encoding="utf-8", newline="") as file:
    writer = csv.DictWriter(file, fieldnames=["id", "name"])
    writer.writeheader()
    writer.writerows(rows)

with open("users.csv", "r", encoding="utf-8", newline="") as file:
    reader = csv.DictReader(file)
    loaded = list(reader)

print(loaded)
# [{'Id': '1', 'Name': 'Ana, Marie'}, {'Id': '2', 'Name': 'Raj'}]
```

### 9.2 `reader` And `DictReader`

- `csv.reader()` returns each row as a list of strings.
- `csv.DictReader()` maps each row to a dictionary using the header.
- `csv.writer()` writes rows from sequences.
- `csv.DictWriter()` writes rows from dictionaries.

CSV does not inherently preserve Python data types. Values are read as strings and must be converted explicitly.

```python
# Demonstrate: reader And DictReader.
with open("users.csv", encoding="utf-8", newline="") as file:
    for row in csv.DictReader(file):
        user_id = int(row["id"])
        name = row["name"]
```

Validate missing values, malformed numbers, duplicate headers, unexpected columns, and inconsistent row lengths according to the application contract.

### 9.3 Dialects And Formatting

Important options include:

- `delimiter`: field separator, commonly `,` or `\t`.
- `quotechar`: character used to quote fields.
- `quoting`: controls when fields are quoted.
- `escapechar`: escape character.
- `doublequote`: whether quotes inside quoted fields are doubled.
- `lineterminator`: line ending used by the writer.
- `skipinitialspace`: whether spaces immediately after delimiters are skipped by the reader.

Use `csv.Sniffer` only as a heuristic. Automatic dialect detection can be wrong, especially with short or irregular samples. Prefer a known format contract when possible.

### 9.4 CSV Formula Injection

When CSV is opened in spreadsheet software, values beginning with characters such as `=`, `+`, `-`, or `@` may be interpreted as formulas. If exporting untrusted text for spreadsheet users, define and apply a formula-injection mitigation policy. The correct mitigation depends on the target spreadsheet application and downstream use; quoting a field alone is not a universal defense.

### 9.5 CSV Versus JSON

| CSV | JSON |
|---|---|
| Best suited to rows and columns | Best suited to nested objects and arrays |
| Values are text | Has strings, numbers, booleans, arrays, objects, and null |
| Compact and widely supported by spreadsheet tools | Common in APIs and configuration |
| Schema is usually external or implied by headers | Structure is represented in the document |
| Nested data is awkward | Nested data is natural |

## 10. Pickle Serialization

`pickle` serializes Python objects into a Python-specific binary representation. It can preserve many Python-specific object structures and is useful for trusted internal workflows.

```python
# Demonstrate: Pickle Serialization.
import pickle

record = {"count": 3, "items": [10, 20]}

payload = pickle.dumps(record)
restored = pickle.loads(payload)

assert restored == record
```

File-based usage:

```python
# Demonstrate: Pickle Serialization.
import pickle

with open("state.pkl", "wb") as file:
    pickle.dump(record, file)

with open("state.pkl", "rb") as file:
    restored = pickle.load(file)
```

### 10.1 Pickle Security

**Never unpickle data from an untrusted or unauthenticated source.** Unpickling can execute arbitrary code.

Do not unpickle:

- User-uploaded files.
- Files downloaded from the internet.
- Data from an untrusted cache or shared directory.
- Artifacts whose origin and integrity cannot be established.

A checksum detects accidental corruption, not malicious tampering. A cryptographic signature or message authentication code can establish authenticity only when key management, verification, and trust boundaries are correctly designed. Even then, use pickle only when the source is genuinely trusted.

### 10.2 Pickle Compatibility And Limitations

- Pickle is Python-specific and is not a general cross-language interchange format.
- Pickled instances can depend on importable module paths and compatible class definitions.
- Refactoring or moving classes can make old pickles difficult to load.
- Protocol versions affect compatibility with Python versions.
- Not every object can be pickled, especially objects tied to active operating-system resources or runtime state.

Specify a protocol deliberately when compatibility requirements matter. The highest available protocol is not necessarily appropriate if older Python versions must read the data.

Use JSON, CSV, or a documented schema-based format for long-lived or cross-language data exchange.

## 11. Other Standard-Library Data Formats

Choose a format based on the shape of the data, interoperability requirements, size, and compatibility needs.

### 11.1 `configparser` For INI-Style Configuration

```python
# Demonstrate: configparser For INI-Style Configuration.
import configparser

config = configparser.ConfigParser()
config["database"] = {
    "host": "localhost",
    "port": "5432",
}

with open("settings.ini", "w", encoding="utf-8") as file:
    config.write(file)

loaded = configparser.ConfigParser()
loaded.read("settings.ini", encoding="utf-8")

host = loaded["database"]["host"]
port = loaded.getint("database", "port")
```

`ConfigParser` is useful for section-based configuration. Values are read as strings unless typed getter methods such as `getint()`, `getfloat()`, or `getboolean()` are used. Define how missing sections, missing keys, interpolation, and defaults should behave.

### 11.2 `tomllib` For TOML Reading

Python 3.11+ includes `tomllib` for reading TOML. It does not provide a standard-library TOML writer.

```python
# Demonstrate: tomllib For TOML Reading.
import tomllib

with open("pyproject.toml", "rb") as file:
    config = tomllib.load(file)
```

TOML is commonly used for human-edited configuration. `tomllib.load()` expects a binary file object.

### 11.3 `xml.etree.ElementTree` For XML

```python
# Demonstrate: xml.etree.ElementTree For XML.
import xml.etree.ElementTree as ET

root = ET.fromstring("<users><user id='1'>Ana</user></users>")

for user in root.findall("user"):
    print(user.get("id"), user.text)
```

XML supports hierarchical, tagged data and namespaces. For untrusted XML, understand the security properties of the parser and the Python version in use; do not assume every XML-processing library has identical protections. For very large XML documents, incremental parsing with `iterparse()` can reduce memory usage.

### 11.4 `shelve` And `dbm`

`shelve` provides a dictionary-like interface backed by `dbm` databases.

```python
# Demonstrate: shelve And dbm.
import shelve

with shelve.open("application_state") as db:
    db["user:1"] = {"name": "Ana"}
    print(db["user:1"])
```

Important considerations:

- `shelve` uses pickle internally, so never open an untrusted shelf.
- Concurrent read/write access and portability depend on the underlying `dbm` implementation.
- It is not a replacement for a transactional database.
- Changes to mutable objects retrieved from a shelf may need to be assigned back, or `writeback=True` must be used with its memory/performance trade-offs.

### 11.5 Compression Formats

The standard library includes modules such as:

- `gzip`: gzip-compressed files.
- `bz2`: bzip2 compression.
- `lzma`: LZMA/XZ compression.
- `zipfile`: ZIP archives.
- `tarfile`: tar archives, optionally compressed.

Example with gzip:

```python
# Demonstrate: Compression Formats.
import gzip

with gzip.open("events.txt.gz", "wt", encoding="utf-8") as file:
    file.write("event one\n")

with gzip.open("events.txt.gz", "rt", encoding="utf-8") as file:
    print(file.read())
```

When extracting archives from untrusted sources, validate member paths and extraction targets. Archive entries can contain path traversal attempts. Also consider decompression bombs, unexpected file counts, and excessive extracted sizes.

## 12. Serialization Format Selection

Serialization converts in-memory data into a representation that can be stored or transmitted. Deserialization reconstructs data from that representation.

Before choosing a format, evaluate:

1. **Interoperability:** Must another language or system read it?
2. **Schema:** Is the structure simple, nested, or strongly typed?
3. **Compatibility:** Must older application versions read newer or older data?
4. **Size and speed:** Are encoding and decoding costs important?
5. **Streaming:** Can records be processed independently?
6. **Security:** Can the input be untrusted?
7. **Human readability:** Must people inspect or edit the data?
8. **Evolution:** How will fields be added, removed, or reinterpreted?

| Format | Good fit | Main trade-off |
|---|---|---|
| JSON | APIs, configuration, nested interoperable data | Verbose; custom types need conversion |
| JSON Lines | Logs and record-oriented streaming | Each line must be handled as a separate record |
| CSV | Flat tabular data and spreadsheet exchange | Weak type and nested-data support |
| Pickle | Trusted Python-only object persistence | Unsafe for untrusted input; Python-specific |
| TOML | Human-edited configuration | Standard library reads but does not write TOML |
| INI | Simple section-based configuration | Limited data model |
| XML | Structured document interchange | Verbose; parser/security considerations |
| `shelve` | Small Python-local key/value persistence | Pickle security and database implementation limitations |
| Protocol Buffers / Avro | Schema-based, cross-language systems | Requires external libraries and schema/tooling |

For production data pipelines, the file extension alone is not a schema. Define field names, types, nullability, encoding, compression, versioning, and error-handling rules.

## 13. File Metadata, Permissions, And Filesystem Operations

### 13.1 Inspect Metadata

```python
# Demonstrate: Inspect Metadata.
from pathlib import Path

path = Path("report.csv")
info = path.stat()

print(info.st_size)   # Size in bytes
print(info.st_mtime)  # Modification time (Unix timestamp)
print(info.st_mode)   # Mode bits
```

`stat()` can fail if the path does not exist or cannot be accessed. `Path.stat()` follows symbolic links by default; use `Path.lstat()` when link metadata itself is needed.

### 13.2 Use `os` For Lower-Level Operations

`pathlib` is convenient for common path work. The `os` module exposes lower-level operating-system operations, including:

- `os.getcwd()` and `os.chdir()`
- `os.mkdir()` and `os.makedirs()`
- `os.listdir()` and `os.scandir()`
- `os.remove()` and `os.unlink()`
- `os.rename()` and `os.replace()`
- `os.stat()` and `os.chmod()`
- `os.fsync()` and `os.ftruncate()`

Avoid changing the process-wide current working directory inside reusable library code. It affects every code path in the process and can create concurrency bugs.

### 13.3 Symbolic Links And Hard Links

A symbolic link points to another path. A hard link is another directory entry for the same underlying file object on filesystems that support it.

- `Path.is_symlink()` checks whether the path itself is a symbolic link.
- `Path.resolve()` resolves symlinks where possible.
- `Path.lstat()` inspects the link rather than following it.
- `Path.hardlink_to()` creates a hard link where supported.

Do not assume links behave identically across operating systems or filesystems. Security-sensitive code must account for link traversal and replacement races.

# Strings, Unicode, Formatting, and Regular Expressions

Text handling is more than calling `split()` and `replace()`. Production Python code must distinguish Unicode text from encoded bytes, handle normalization and case folding, format values safely, and use regular expressions without creating performance or security problems.

## 1. `str` is Unicode text

A Python `str` is an immutable sequence of Unicode code points. It is not a sequence of UTF-8 bytes and it is not necessarily a sequence of user-perceived characters (a visible grapheme can contain multiple code points).

```python
word = "café"
print(len(word))                  # 4 Unicode code points
print(word[3])                    # é

emoji = "👩‍💻"                     # woman + ZWJ + laptop
print(len(emoji))                 # multiple code points, usually one displayed grapheme
```

Consequences:
- Indexing and slicing operate on code points, not grapheme clusters.
- `len(text)` is not necessarily the number of visible symbols.
- `str` is immutable. Methods such as `replace`, `strip`, and `lower` return new strings.
- String equality compares code-point sequences; visually identical text can have different underlying sequences.

## 2. Unicode normalization and case handling

Some characters can be represented by a single code point or by a base character followed by a combining mark. Normalize when comparing or storing text where canonical equivalence matters.

```python
import unicodedata

composed = "é"
decomposed = "e\u0301"

print(composed == decomposed)  # False: different code-point sequences
print(unicodedata.normalize("NFC", composed) ==
      unicodedata.normalize("NFC", decomposed))  # True
```

Use:
- `NFC` for canonical composition in many normal text-processing workflows.
- `NFD` for canonical decomposition.
- `NFKC` / `NFKD` when compatibility characters should also be normalized. This can change distinctions and should not be applied blindly to identifiers, passwords, or security-sensitive values.
- `casefold()` for caseless Unicode matching. It is more aggressive and more suitable than `lower()` for case-insensitive comparisons.

```python
print("Straße".lower())       # straße
print("Straße".casefold())    # strasse
```

Normalization is a domain decision: define it at the system boundary and apply it consistently rather than normalizing arbitrary values in unrelated layers.

## 3. Text, bytes, and encodings

- `str` represents text.
- `bytes` represents an immutable sequence of byte values from 0 through 255.
- Encoding converts text to bytes; decoding converts bytes to text.

```python
text = "café"
raw = text.encode("utf-8")

print(raw)                         # b'caf\xc3\xa9'
print(raw.decode("utf-8"))         # café
print(len(text), len(raw))         # 4 5
```

Always know which encoding a protocol or file format specifies. UTF-8 is the usual default for modern application data. Python 3.15 makes UTF-8 the default text encoding (PEP 686); code that must be predictable across environments and older versions should still specify `encoding="utf-8"` explicitly.

Error handling:
- The default `errors="strict"` raises `UnicodeDecodeError` or `UnicodeEncodeError` instead of silently losing data.
- `errors="replace"` substitutes replacement characters; useful for display-only recovery, but it is lossy.
- `errors="ignore"` silently drops invalid data and is usually inappropriate for data ingestion.

## 4. Common string operations

```python
message = "  Python, Python, Python  "

print(message.strip())                  # remove leading/trailing whitespace
print(message.lower())
print(message.upper())
print(message.casefold())
print(message.count("Python"))          # 3
print(message.replace("Python", "Py"))  # replace all occurrences
print("a,b,c".split(","))               # ['a', 'b', 'c']
print(",".join(["a", "b", "c"]))        # a,b,c
print("file.csv".removesuffix(".csv"))  # file
print("report.final.csv".partition(".")) # ('report', '.', 'final.csv')
```

Important details:
- `strip(chars)` removes any matching characters from both ends; it does **not** remove an exact substring. Use `removeprefix()` / `removesuffix()` for exact prefixes/suffixes.
- `split()` with no separator collapses runs of whitespace and removes leading/trailing whitespace.
- `splitlines()` handles line boundaries without keeping newline characters by default.
- `join()` is called on the separator and consumes an iterable of strings.
- `find()` returns `-1` when absent; `index()` raises `ValueError`.
- `startswith()` / `endswith()` can accept a tuple of alternatives.

## 5. Formatting: f-strings and `format()`

Prefer f-strings for readable interpolation. Use the format mini-language for alignment, precision, widths, and numeric presentation.

```python
name = "Ana"
amount = 1234.5678

print(f"Hello, {name}!")
print(f"Amount: {amount:,.2f}")       # Amount: 1,234.57
print(f"{name:>10}")                  # right aligned in width 10
print(f"{name:<10}")                  # left aligned
print(f"{amount:.1%}")                # 123456.78% (format as a percentage)
print(f"{amount=:.2f}")               # amount=1234.57 (debug format)
```

Common format specifiers:
- `:.2f` — fixed-point with two decimal places.
- `:,.2f` — thousands separators and two decimals.
- `:>12`, `:<12`, `:^12` — right, left, and centered alignment.
- `:.3g` — three significant digits.
- `:08d` — zero-padded integer width.
- `:.1%` — multiply by 100 and append `%`.

For logging, use the logging module's parameterized message form (`logger.info("user=%s", user_id)`) rather than eagerly building an f-string when the log level may be disabled. See `21-Python-Logger.md`.

**Security:** never treat untrusted input as a format template for a custom object or interpolate it into SQL or shell commands. Formatting is not parameterization.

## 6. `str` methods vs regular expressions

Use ordinary string methods for literal operations:
- `in`, `startswith`, `endswith`, `split`, `partition`, `replace`, `removeprefix`, `removesuffix`.
- Reach for `re` when the rule is genuinely a pattern, not merely because regex looks compact.

## 7. Regular expressions with `re`

Raw string literals (`r"..."`) reduce escaping because the Python string parser does not consume most backslashes. They do not change regex syntax.

```python
import re

text = "Order 418 shipped; order 529 is pending."
pattern = re.compile(r"order\s+(?P<order_id>\d+)", re.IGNORECASE)

for match in pattern.finditer(text):
    print(match.group("order_id"))
# 418
# 529
```

Core functions:
- `re.search(pattern, text)` — first match anywhere.
- `re.match(pattern, text)` — match only at the beginning of the string.
- `re.fullmatch(pattern, text)` — the entire string must match.
- `re.findall(...)` — return captured values or matched strings.
- `re.finditer(...)` — lazy iterator of `Match` objects.
- `re.sub(...)` / `re.subn(...)` — replace matches; the latter also returns a replacement count.
- `re.compile(...)` — compile once when reusing the same pattern.

```python
import re

email_like = re.compile(r"(?P<user>[\w.+-]+)@(?P<domain>[\w.-]+\.[A-Za-z]{2,})")
match = email_like.fullmatch("ana@example.com")

if match:
    print(match.group("user"))    # ana
    print(match.groupdict())      # {'user': 'ana', 'domain': 'example.com'}

print(re.sub(r"\s+", " ", "too   many\nspaces").strip())
```

This example is a simple syntax demonstration, **not** a complete email-address validator. Real validation depends on the application's accepted address rules.

### Important regex operators

| Pattern | Meaning |
|---|---|
| `.` | Any character except newline by default |
| `^`, `$` | Start and end of string/line (depending on flags) |
| `\d`, `\w`, `\s` | Digit, word character, whitespace |
| `*`, `+`, `?` | Zero or more, one or more, zero or one |
| `{m,n}` | Between m and n repetitions |
| `[...]`, `[^...]` | Character class / negated character class |
| `(...)` | Capturing group |
| `(?:...)` | Non-capturing group |
| `(?P<name>...)` | Named capture group |
| `\b` | Word boundary |
| `|` | Alternation |

Greediness: `.*` is greedy; `.*?` is lazy. Prefer precise character classes over broad `.*` patterns when possible.

### ReDoS and catastrophic backtracking

Some patterns can take exponentially long on carefully chosen input, especially nested ambiguous quantifiers such as `(a+)+$`. This can create a Regular Expression Denial of Service (ReDoS) risk when matching attacker-controlled strings.

Mitigations:
- Keep patterns simple and bounded.
- Avoid nested ambiguous quantifiers.
- Bound input length before matching.
- Test long near-miss strings, not only successful examples.
- Use a regex engine with timeout support when the application requires hard execution limits; Python's standard `re` API does not expose a per-match timeout.

## 8. Common interview traps

- `strip("abc")` removes any of the characters `a`, `b`, or `c` from both ends; it does not remove the substring `"abc"`.
- `lower()` is not the same as Unicode-aware caseless matching with `casefold()`.
- A visible grapheme may have more than one code point, so `len()` is not a display-character count.
- `str` and `bytes` are different types; compare or concatenate them only after choosing an explicit encoding/decoding boundary.
- `re.match()` and `re.search()` differ: start-only versus anywhere.
- `re.fullmatch()` is the right primitive when the entire input must satisfy the pattern.
- A raw regex string helps with escaping but does not make a pattern safe from catastrophic backtracking.
- Prefer literal string methods when regex is unnecessary.

## 9. Practice questions

1. Why can two visually identical Unicode strings compare unequal?
2. Why is `casefold()` preferable to `lower()` for some caseless comparisons?
3. What is the difference between encoding and decoding?
4. What is the difference between `strip()` and `removesuffix()`?
5. Compare `re.match`, `re.search`, and `re.fullmatch`.
6. What is a named capture group, and when is it useful?
7. How can a regular expression become a denial-of-service vulnerability?

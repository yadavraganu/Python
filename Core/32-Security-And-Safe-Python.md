# Security and Safe Python

Python does not make untrusted input safe automatically. Security depends on trust boundaries, validation, safe APIs, permissions, dependency hygiene, and operational controls.

This guide focuses on core Python interfaces and common interview scenarios. It is not a substitute for a threat model or security review.

## 1. Treat input as untrusted

Validate data at system boundaries:

- Check type, shape, length, ranges, allowed values, and encoding.
- Reject unexpected fields when the data contract requires a closed schema.
- Normalize data deliberately; do not silently repair values in ways that change meaning.
- Separate validation from authorization. A syntactically valid user ID does not mean the caller may access that user.
- Enforce limits on request size, recursion depth, decompressed size, and processing time.

Prefer allow-lists for structured inputs. Avoid building code, SQL, shell commands, or filesystem paths by concatenating untrusted strings.

## 2. Never use `eval` or `exec` on untrusted input

```python
# Unsafe: user-controlled_text can execute arbitrary Python code.
# result = eval(user_controlled_text)

# Safer: parse a literal-only Python data structure if this exact format is needed.
import ast

value = ast.literal_eval("[1, 2, {'enabled': True}]")
assert value == [1, 2, {"enabled": True}]
```

`ast.literal_eval()` accepts only Python literals and container displays, not arbitrary expressions or calls. It can still consume excessive resources with maliciously large/deep input, so enforce size/resource limits. For external interchange, JSON is usually preferable.

## 3. Deserialization: pickle is executable

`pickle.load()` and `pickle.loads()` can execute arbitrary code during unpickling. Never unpickle data supplied by an untrusted party, including uploaded files, messages, or cache entries whose integrity is not guaranteed.

```python
import json

payload = json.dumps({"user_id": 42, "active": True})
decoded = json.loads(payload)
assert decoded["user_id"] == 42
```

JSON is data-only relative to pickle, but still validate schemas and limits. Other serializers and object-mapping libraries need their own security review.

## 4. SQL injection: parameterize values

Never interpolate untrusted values into SQL text.

```python
# Unsafe:
# cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# Safe parameter binding (placeholder syntax depends on the DB driver):
cursor.execute("SELECT * FROM users WHERE email = ?", (email,))
```

Placeholders bind values, not table or column identifiers. For dynamic identifiers, choose from a strict allow-list and use the driver's identifier-quoting facilities. Parameterization does not replace authorization or least-privilege database accounts.

## 5. Shell commands and subprocess

Prefer a list of arguments with `shell=False` (the default), set timeouts, and validate what the command is allowed to do.

```python
import subprocess

completed = subprocess.run(
    ["python", "--version"],
    check=True,
    capture_output=True,
    text=True,
    timeout=5,
)
print(completed.stdout or completed.stderr)
```

Avoid `shell=True` with data that can be influenced by a user. Do not assume quoting untrusted strings makes shell execution safe. Also consider executable lookup, working directory, environment variables, permissions, output size, and process cleanup.

## 6. Secrets and cryptography

### Generate secrets with `secrets`, not `random`

```python
import secrets

reset_token = secrets.token_urlsafe(32)
```

`random` is designed for simulation and general-purpose pseudo-randomness, not security tokens.

### Hashes are not encryption or password hashing

- `hashlib.sha256()` is a cryptographic digest; it is useful for integrity fingerprints but is too fast to store passwords safely.
- Passwords should be hashed with a dedicated password-hashing scheme such as Argon2id, scrypt, or bcrypt, using a unique salt and work factor managed by a vetted library.
- Do not invent a password-hashing algorithm.
- Encryption is reversible with a key; hashing is not.
- Use HMAC or a digital signature when authenticity/integrity against an attacker matters.

For comparison of secret MACs/tokens, use `hmac.compare_digest()` rather than a normal equality comparison where timing differences could leak information.

```python
import hmac

assert hmac.compare_digest(b"expected", b"expected")
```

Keep secrets out of source control, logs, exception messages, URLs, and test fixtures. Rotate exposed credentials and scope them to the minimum permissions.

## 7. Filesystem paths and path traversal

A path supplied by a user may contain `..`, absolute paths, symlinks, or alternate separators. Resolve it and verify it remains inside the intended root.

```python
from pathlib import Path

def safe_child(root: Path, user_name: str) -> Path:
    base = root.resolve()
    candidate = (base / user_name).resolve()
    if not candidate.is_relative_to(base):
        raise ValueError("path escapes the permitted directory")
    return candidate
```

This is a useful containment check, not a complete defense against every race condition. In a hostile concurrent filesystem, an attacker may replace a path or symlink between validation and use. Stronger designs use directory file descriptors and OS-level no-follow/open flags where available, restrict write permissions, and avoid attacker-controlled directory structures.

Use `tempfile` for temporary files/directories; do not create predictable temporary names yourself.

## 8. Resource exhaustion and denial of service

Untrusted input can be syntactically valid but expensive.

Examples:

- Huge JSON arrays or deeply nested structures.
- Regex patterns with catastrophic backtracking.
- Very large compressed payloads (decompression bombs).
- Expensive sorting, parsing, or cryptographic work repeated by many requests.
- Unbounded caches, queues, thread pools, or task creation.
- Image/archive/document inputs with unexpectedly large expansion ratios.

Mitigations include input-size limits, timeouts, concurrency limits, bounded queues/caches, streaming parsers, safe regex design, and per-user quotas. A timeout in one layer may not stop work in another; design cancellation and cleanup explicitly.

## 9. Logging and error handling

- Do not log passwords, access tokens, session identifiers, private keys, or full sensitive request bodies.
- Avoid returning stack traces and internal paths to untrusted clients.
- Keep enough structured context to investigate failures without leaking secrets.
- Do not silently catch broad exceptions and continue in a corrupted state.
- Sanitize control characters in values that may be rendered in terminals or log viewers.
- Treat logs as sensitive data with retention and access controls.

See the logging and exception-handling guides for mechanics.

## 10. Dependencies and supply chain

For production Python systems:

- Pin or constrain dependencies deliberately and use a lockfile for reproducible deployments.
- Prefer maintained packages from trusted sources; review ownership, release cadence, and transitive dependencies.
- Use hashes or trusted package-manager verification where appropriate.
- Avoid dependency confusion by controlling package indexes and names.
- Keep Python and dependencies patched; remove unused dependencies.
- Build in controlled environments and scan artifacts/dependencies as part of CI.
- Do not treat a version pin as proof that a package is safe.

## 11. Security review checklist

- [ ] Identify trust boundaries and classify sensitive data.
- [ ] Validate schema, size, ranges, and authorization at the boundary.
- [ ] Never pass untrusted input to `eval`, `exec`, or `pickle`.
- [ ] Parameterize SQL values; allow-list dynamic identifiers.
- [ ] Use argument-list subprocess calls, timeouts, and least privilege.
- [ ] Use `secrets` and vetted password-hashing/cryptography libraries.
- [ ] Constrain paths and protect against symlink/race attacks where relevant.
- [ ] Bound time, memory, concurrency, and queue growth.
- [ ] Redact secrets from logs and errors.
- [ ] Review dependency provenance and update cadence.

## 12. Interview scenarios

**A cache stores objects with pickle. Is it safe?**  
Only if the data and cache integrity are fully trusted. A compromised writer or shared cache can turn unpickling into code execution. Consider JSON or another constrained schema for untrusted/shared data.

**Why doesn't `ast.literal_eval` solve every input problem?**  
It does not execute arbitrary Python code, but deeply nested or enormous literal inputs can still consume excessive memory or CPU. It is not a general external data format.

**Why is `Path.resolve()` not a complete sandbox?**  
Filesystem state can change between validation and use; symlinks and race conditions matter. Security also depends on directory permissions and OS-level open semantics.

**Is SHA-256 suitable for password storage?**  
No. General-purpose hashes are intentionally fast. Use a password-specific KDF such as Argon2id, scrypt, or bcrypt with a unique salt and appropriate work factor.

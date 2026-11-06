## 1. Why Not Just Use `print()`

`print()` has no levels, no timestamps, no destinations besides stdout, and no way to turn it off in production without deleting code. The built-in `logging` module solves all of this: it lets you tag messages by severity, route them to files/consoles/network sockets, format them consistently, and control verbosity per-module without touching your source code.

## 2. The Five Log Levels

| Level | Value | When to use |
| --- | --- | --- |
| `DEBUG` | 10 | Detailed diagnostic info, useful only when tracing a problem |
| `INFO` | 20 | Confirmation that things are working as expected |
| `WARNING` | 30 | Something unexpected happened, but the program continues |
| `ERROR` | 40 | A serious problem; some functionality failed |
| `CRITICAL` | 50 | The program itself may be unable to continue |

A logger/handler only emits messages **at or above** its configured level.

```python
import logging

logging.debug("Debug message")
logging.info("Info message")
logging.warning("Warning message")
logging.error("Error message")
logging.critical("Critical message")
```

By default, the root logger is set to `WARNING`, so running the above only prints the last three lines.

## 3. The Core Building Blocks

Logging has four main components that work together:

- **Logger** – the entry point your code calls (`logger.info(...)`).
- **Handler** – decides *where* a log record goes (console, file, network, email...).
- **Formatter** – decides *how* a log record looks as text.
- **Filter** – fine-grained control over which records get through.

```
Your code → Logger → Filter(s) → Handler(s) → Formatter → Destination
```

## 4. Quick Start (Single Script)

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

logger = logging.getLogger(__name__)

logger.debug("Starting process")
logger.info("Process running")
logger.warning("Disk space low")
logger.error("Process failed")
```

`basicConfig()` is convenient for small scripts, but it only works the **first time** it's called in a process (subsequent calls are no-ops unless you pass `force=True`). For anything beyond a single script, prefer the manual setup below.

## 5. The Recommended Pattern: `getLogger(__name__)`

In every module, create a logger named after the module:

```python
import logging

logger = logging.getLogger(__name__)
```

This creates a hierarchy that mirrors your package structure (`myapp.db`, `myapp.api`, `myapp.api.routes`, ...). Because loggers propagate messages up to their parents, you can configure logging **once**, at the application's entry point, and every module's logger inherits that configuration automatically. Never call `basicConfig()` or attach handlers inside library code — only the final application should configure handlers.

## 6. Manual Setup: Logger, Handler, Formatter

```python
import logging

# 1. Create/get a logger
logger = logging.getLogger("myapp")
logger.setLevel(logging.DEBUG)  # lowest level this logger will process

# 2. Create a handler (console)
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)  # only INFO+ goes to console

# 3. Create a formatter and attach it to the handler
formatter = logging.Formatter(
    "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
console_handler.setFormatter(formatter)

# 4. Attach the handler to the logger
logger.addHandler(console_handler)

logger.debug("This won't show on console (below INFO)")
logger.info("This will show")
```

Note the two-tier filtering: the **logger's** level is the first gate, and each **handler's** level is a second, independent gate. A message must clear both to be emitted by that handler.

## 7. Logging to a File

```python
import logging

logger = logging.getLogger("myapp")
logger.setLevel(logging.DEBUG)

file_handler = logging.FileHandler("app.log", encoding="utf-8")
file_handler.setLevel(logging.DEBUG)
file_handler.setFormatter(
    logging.Formatter("%(asctime)s %(levelname)s %(name)s: %(message)s")
)

logger.addHandler(file_handler)
logger.info("Application started")
```

### Rotating log files

Plain `FileHandler` grows forever. Use rotation in real applications:

```python
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler

# Rotate when the file hits 5 MB, keep 3 old copies
size_handler = RotatingFileHandler(
    "app.log", maxBytes=5 * 1024 * 1024, backupCount=3
)

# Rotate at midnight, keep 7 days of logs
time_handler = TimedRotatingFileHandler(
    "app.log", when="midnight", backupCount=7
)
```

## 8. Multiple Handlers, Different Levels

A single logger can fan out to several destinations at different verbosities — a very common production pattern: everything to a file, only warnings+ to the console.

```python
import logging

logger = logging.getLogger("myapp")
logger.setLevel(logging.DEBUG)

console = logging.StreamHandler()
console.setLevel(logging.WARNING)
console.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))

file = logging.FileHandler("app.log")
file.setLevel(logging.DEBUG)
file.setFormatter(
    logging.Formatter("%(asctime)s %(levelname)s %(name)s: %(message)s")
)

logger.addHandler(console)
logger.addHandler(file)

logger.debug("Only in file")
logger.warning("In file AND console")
```

## 9. Logging Exceptions

Use `logger.exception()` inside an `except` block — it logs at `ERROR` level and automatically includes the traceback:

```python
logger = logging.getLogger(__name__)

try:
    1 / 0
except ZeroDivisionError:
    logger.exception("Division failed")
```

Output includes the full traceback. Equivalent to `logger.error("...", exc_info=True)`, which you can use if you want a different level:

```python
logger.critical("Unrecoverable error", exc_info=True)
```

## 10. Useful Format Attributes

| Attribute | Meaning |
| --- | --- |
| `%(asctime)s` | Human-readable timestamp |
| `%(name)s` | Logger name |
| `%(levelname)s` | DEBUG / INFO / WARNING / ERROR / CRITICAL |
| `%(message)s` | The log message |
| `%(filename)s` | Source filename |
| `%(funcName)s` | Function the call came from |
| `%(lineno)d` | Line number |
| `%(process)d` | Process ID |
| `%(threadName)s` | Thread name |

Example:

```python
formatter = logging.Formatter(
    "%(asctime)s [%(levelname)s] %(filename)s:%(lineno)d %(funcName)s(): %(message)s"
)
```

## 11. Passing Variables Safely (Lazy Formatting)

Prefer `%`-style lazy formatting over f-strings for log calls:

```python
# Good — string is only formatted if the log level is enabled
logger.debug("User %s logged in from %s", user_id, ip_address)

# Avoid — the f-string is always built, even if DEBUG is disabled
logger.debug(f"User {user_id} logged in from {ip_address}")
```

This matters for performance in hot code paths, since the interpolation is skipped entirely when the message wouldn't be emitted.

## 12. Structured / Extra Data

Attach extra fields to a record with the `extra` argument, then reference them in your formatter:

```python
formatter = logging.Formatter(
    "%(asctime)s %(levelname)s [request_id=%(request_id)s] %(message)s"
)
handler.setFormatter(formatter)

logger.info("Handling request", extra={"request_id": "abc-123"})
```

For JSON logs (common in production/observability pipelines), write a custom `Formatter`:

```python
import json
import logging

class JSONFormatter(logging.Formatter):
    def format(self, record):
        payload = {
            "time": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        if record.exc_info:
            payload["exception"] = self.formatException(record.exc_info)
        return json.dumps(payload)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)
```

## 13. Configuring via Dictionary (Recommended for Real Apps)

`logging.config.dictConfig` lets you define your entire logging setup declaratively — easy to load from YAML/JSON config files and easy to reason about.

```python
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "level": "INFO",
            "formatter": "standard",
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "DEBUG",
            "formatter": "standard",
            "filename": "app.log",
            "maxBytes": 5_000_000,
            "backupCount": 3,
        },
    },
    "loggers": {
        "myapp": {
            "handlers": ["console", "file"],
            "level": "DEBUG",
            "propagate": False,
        },
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
logger = logging.getLogger("myapp")
logger.info("Configured via dictConfig")
```

## 14. Common Pitfalls

- **Attaching handlers more than once.** If a module is imported repeatedly (e.g., in tests or reloaded), `addHandler` can stack duplicate handlers, causing duplicate log lines. Guard with `if not logger.handlers:` or configure logging only once at startup.
- **Configuring logging inside library code.** Libraries should only call `getLogger(__name__)` and log messages — never call `basicConfig` or attach handlers. Let the *application* decide where logs go.
- **Forgetting `propagate`.** By default, child loggers propagate to the root logger, which can cause duplicate output if both the child and root have handlers. Set `logger.propagate = False` when a logger has its own handlers and shouldn't also go through the root's.
- **Using f-strings in log calls.** As noted above, this always formats the string, even if the message won't be logged.
- **Logging sensitive data.** Passwords, tokens, and PII should never end up in logs — scrub or mask before logging.

## 15. Minimal Reusable Setup for a Project

A pattern you can drop into most projects — one `logging_config.py`, called once from `main.py`:

```python
# logging_config.py
import logging
import logging.config

def setup_logging(level=logging.INFO, log_file="app.log"):
    config = {
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "default": {
                "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
                "datefmt": "%Y-%m-%d %H:%M:%S",
            },
        },
        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "formatter": "default",
                "level": level,
            },
            "file": {
                "class": "logging.handlers.RotatingFileHandler",
                "formatter": "default",
                "filename": log_file,
                "maxBytes": 5_000_000,
                "backupCount": 3,
                "level": "DEBUG",
            },
        },
        "root": {
            "handlers": ["console", "file"],
            "level": "DEBUG",
        },
    }
    logging.config.dictConfig(config)
```

```python
# main.py
from logging_config import setup_logging
import logging

setup_logging()
logger = logging.getLogger(__name__)
logger.info("App started")
```

## 16. Quick Reference Cheat Sheet

```python
import logging

logger = logging.getLogger(__name__)          # get a module-level logger
logger.setLevel(logging.DEBUG)                 # set threshold

handler = logging.StreamHandler()              # or FileHandler, RotatingFileHandler...
handler.setLevel(logging.INFO)
handler.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(message)s"))
logger.addHandler(handler)

logger.debug("...")
logger.info("...")
logger.warning("...")
logger.error("...")
logger.critical("...")
logger.exception("...")   # inside except block, includes traceback
```

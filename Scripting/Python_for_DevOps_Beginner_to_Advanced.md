# Python for DevOps Engineers: Zero to Production Automation

**A practical, self-contained handbook | Beginner → Intermediate → Advanced**  
**Audience:** DevOps engineers, Linux administrators, SREs, CI/CD engineers, and cloud engineers who already know some Bash and infrastructure but may know **zero Python**.  
**Recommended runtime:** Python 3.11 or newer (most examples work on 3.10+).  
**Primary environment:** Linux or Ubuntu on Windows Subsystem for Linux (WSL).  
**Learning style:** Explain → compare with Bash → run code → inspect output → complete a lab.

> **Safety defaults:** All initial labs run locally. Cloud and Kubernetes examples are **read-only** unless explicitly labeled otherwise. Never run production-modifying commands while learning. Do not print API tokens, commit secrets, disable TLS verification, or copy untrusted code into privileged CI jobs.

---

## Navigation

- [Part I — Python from zero](#part-i): variables, strings, collections, conditions, loops, functions and errors.
- [Part II — Practical Python automation](#part-ii): files, JSON/YAML, regex, subprocess, APIs, CLI tools and release hashes.
- [Part III — Python in real DevOps platforms](#part-iii): Git, GitHub Actions, Docker, Kubernetes, AWS, tests, typing and security.
- [Part IV — Guided labs](#part-iv): fourteen step-by-step labs, including [the complete local GitOps capstone](#capstone).
- [Part V — Revision and assessment](#part-v): exercises, Bash comparison, debugging, study plan and references.
- [Appendix — Terraform, Helm, Ansible and reusable CI orchestration](#appendix).

---

## How to use this book

1. Work through **Part I** in order, even if individual topics seem obvious. A DevOps script is just small Python concepts composed together.
2. Type and run examples; don't only read them. Use a virtual environment.
3. Do the **guided labs in Part IV**. Each lab has a goal, code, execution instructions, and checks.
4. Use **Part V** as a reference when reviewing CI/CD pipelines and GitOps control code.
5. For sections using AWS, Kubernetes, GitHub, or Docker, having credentials/daemons is optional; first understand the code. Start with read-only access.

### Roadmap and approximate effort

| Level | Sections | Outcome | Suggested practice |
|---|---|---|---|
| Foundations | 1–12 | Read and write ordinary Python | 8–12 hours |
| Core automation | 13–23 | Build safe CLI tools, handle JSON/YAML and APIs | 12–20 hours |
| Platform automation | 24–30 | Integrate GitHub, Docker, Kubernetes, AWS | 12–20 hours |
| Advanced engineering | 31–38 | Test, type, secure, package and operate tooling | 12–20 hours |
| Hands-on | 39–52 | Build portfolio-grade scripts and a capstone | 15–25 hours |

**Reference map:** `def`, loops and `dict` → sections 5–11; `subprocess` → section 18; `argparse` → section 20; JSON/YAML → section 16; release integrity → sections 30 and 45; GitHub Actions outputs → section 26; polling/attestations → sections 32 and 46.

---

<a id="part-i"></a>

# Part I — Python from zero

## 1. What Python is and when to use it instead of Bash

Bash excels at composing command-line programs. Python excels at structured data, APIs, reusable validation, complex branching, maintainability and tests.

**Bash:**

```bash
component="frontend"
echo "Deploying $component"
```

**Python (`hello.py`):**

```python
component = "frontend"
print(f"Deploying {component}")
```

Run it:

```bash
python3 hello.py
# Deploying frontend
```

The `f` creates an **f-string**: expressions inside `{}` are evaluated. In Python, indentation is syntax; you do not put `fi`, `done`, or `{}` around control blocks.

**Use Bash** for short, simple command sequences. **Use Python** for JSON parsing, complicated input checking, APIs, testing and stateful workflows. Often the best automation combines both.

## 2. Environment setup: Linux/WSL, venv, editor and REPL

Check installation:

```bash
python3 --version
python3 -m pip --version
```

On Debian/Ubuntu (only if missing):

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip
```

Create an isolated project:

```bash
mkdir -p ~/python-devops
cd ~/python-devops
python3 -m venv .venv
source .venv/bin/activate
python -V
python -m pip --version
```

A **virtual environment** gives each project its own Python packages; avoid `sudo pip install`. Exit with `deactivate`. On native Windows PowerShell, activation is `.venv\\Scripts\\Activate.ps1`; on WSL use the Linux commands above.

Try the Python REPL:

```bash
python
```

```python
>>> 2 + 3
5
>>> "prod".upper()
'PROD'
```

Exit with `exit()` or Ctrl-D. The `>>>` prompt is not part of your source file.

**Recommended editor:** VS Code with Python and Pylance extensions, or any text editor. Add `.venv/`, `__pycache__/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/` and `.env` to `.gitignore`.

## 3. Script structure, comments, indentation and execution

```python
#!/usr/bin/env python3
"""A one-line module docstring explains what this script does."""

# Comments explain why, not the obvious.

def main():
    environment = "dev"
    print(f"Environment: {environment}")


if __name__ == "__main__":
    main()
```

- `#!` is the shebang, used when the script is executable on Unix.
- A **docstring** documents a module, function, or class.
- `def` declares a function. Its body is indented (normally 4 spaces).
- `if __name__ == "__main__"` runs `main()` only when the file is executed directly, not when imported.
- Python executes ordinary top-level statements during import; keep side effects inside functions.

Both work:

```bash
python3 script.py
chmod +x script.py && ./script.py
```

A frequent error is `IndentationError` after mixing tabs and spaces.

## 4. Variables, basic types and conversions

```python
component = "orders"        # str
replicas = 3               # int
cpu_limit = 0.75           # float
enabled = True             # bool
previous_release = None   # NoneType

print(type(replicas))      # <class 'int'>
print(replicas + 2)        # 5
print(str(replicas))       # '3'
print(int("8080"))         # 8080
print(float("1.5"))        # 1.5
```

`=` means **assignment**. `==` means **comparison**. Python distinguishes `True` from `"True"`: the second is a nonempty string.

```python
value = "False"
print(bool(value))         # True! Any nonempty string is truthy.
print(value.lower() == "true")  # False
```

**Never** parse an environment-variable Boolean with `bool(os.environ["FLAG"])`.

### Expressions and operators

```python
print(7 / 2)     # 3.5, true division
print(7 // 2)    # 3, floor division
print(7 % 2)     # 1, remainder
print(2 ** 4)    # 16, exponentiation
print(3 == 3)    # True
print(3 != 4)    # True
print(3 >= 2)    # True
```

Use `is None` or `is not None` for checking `None`, not `== None`. Use `is` to compare object identity, not ordinary strings or numbers.

### Exercise

Create variables `namespace`, `replicas`, and `enabled`. Print a sentence using all three. Change `replicas` from `"3"` to the integer `3` before doing arithmetic.

## 5. Strings and f-strings

```python
image = "ghcr.io/example/orders"
digest = "sha256:abc123"
reference = f"{image}@{digest}"
print(reference)
print(image.upper())
print(image.startswith("ghcr.io/"))   # True
print(image.removeprefix("ghcr.io/"))  # example/orders
print(" dev,qa,prod ".strip().split(","))  # ['dev', 'qa', 'prod']
```

Useful methods: `.strip()`, `.lower()`, `.upper()`, `.split()`, `.join()`, `.replace()`, `.startswith()`, `.endswith()`, `.removeprefix()`.

```python
components = ["frontend", "catalogue", "orders"]
print(", ".join(components))  # frontend, catalogue, orders
print("release-abcdef"[:7])   # release
print("abcdef"[-2:])         # ef
```

**Indexing begins at zero**. Slices use `[start:stop]` where `stop` is excluded. `text[:8]` means its first eight characters.

Beware: shell quoting and Python string quoting are different. Do not interpolate untrusted input into shell command strings.

## 6. Collections: lists, tuples, sets and dictionaries

### Lists: ordered, mutable, duplicates allowed

```python
servers = ["web01", "web02", "web03"]
print(servers[0])   # web01
servers.append("web04")
servers.remove("web02")
print(len(servers)) # 3
for server in servers:
    print(server)
```

### Tuples: ordered, immutable

```python
environments = ("dev", "qa", "production")
print(environments[2])
# environments[0] = "test"  # TypeError
```

A tuple is useful for a fixed collection of allowed choices. `tuple` immutability is *shallow*: mutable objects inside one may still change.

### Sets: unique, unordered

```python
found = {"frontend", "orders", "frontend"}
required = {"frontend", "orders", "catalogue"}
print(found)                     # unique values, order not guaranteed
print(required - found)          # {'catalogue'}
print(found <= required)         # True: subset
print("orders" in found)         # True
```

An empty set is `set()`, not `{}`. `{}` makes an empty dictionary.

### Dictionaries: key/value records (like JSON objects)

```python
release = {
    "component": "orders",
    "runId": 123,
    "ready": True,
}
print(release["component"])           # orders
print(release.get("missing"))          # None, no KeyError
print(release.get("runId", 0))         # 123
release["digest"] = "sha256:example" # add a key
for key, value in release.items():
    print(key, value)
```

Useful methods:

| Expression | Meaning |
|---|---|
| `d[k]` | value for key `k`; raises `KeyError` if missing |
| `d.get(k, default)` | return default if key missing |
| `d.keys()` | keys |
| `d.values()` | values |
| `d.items()` | `(key, value)` pairs |
| `k in d` | whether key exists |
| `d.update(other)` | merge/update entries in place |

`dict` values can be nested:

```python
state = {"release": {"id": "abc", "components": {"orders": {"runId": 42}}}}
print(state["release"]["components"]["orders"]["runId"])  # 42
```

**DevOps translation:** YAML/JSON configuration usually becomes nested `dict` and `list` objects in Python.

## 7. Conditions, booleans and truthiness

```python
environment = "qa"
ready = True

if environment == "production":
    print("Require human review")
elif environment == "qa" and ready:
    print("Run quality gates")
else:
    print("No deployment")
```

Operators: `==`, `!=`, `<`, `>`, `<=`, `>=`, `and`, `or`, `not`, `in`, `not in`.

Falsy values include `False`, `None`, `0`, `""`, `[]`, `{}`, `set()`. Nonempty strings (even `"false"`) are truthy.

```python
status = "failure"
if status in {"failure", "cancelled"}:
    print("Stop promotion")
```

**Security tip:** Validate exact allowed values, rather than assuming every unexpected value means success.

## 8. Loops: for, while, range, enumerate, zip

```python
for number in range(3):
    print(number)  # 0, 1, 2

for index, env in enumerate(["dev", "qa"], start=1):
    print(index, env)

for env, approval in zip(["dev", "production"], [False, True]):
    print(env, approval)
```

`while` repeats until a condition becomes false. `break` exits the loop, `continue` skips the rest of an iteration.

```python
attempt = 0
while attempt < 3:
    attempt += 1
    print(f"Attempt {attempt}")
    if attempt == 2:
        break
```

Avoid infinite loops in automation: implement a maximum attempt count or deadline.

## 9. Functions, parameters, return values and scope

```python
def image_reference(repository, digest):
    """Create a content-addressed image reference."""
    return f"{repository}@{digest}"

result = image_reference("ghcr.io/example/orders", "sha256:abc")
print(result)
```

A function is **defined** with `def`; its body runs only when **called**. `return` passes a value to the caller; `print` only displays it.

### Default and keyword parameters

```python
def deploy(name, environment="dev"):
    return f"{name} -> {environment}"

print(deploy("orders"))
print(deploy("orders", environment="qa"))
```

### `*args` versus `**kwargs`

```python
def command(*args):
    print(args)  # tuple of positional arguments

command("git", "rev-parse", "HEAD")
# ('git', 'rev-parse', 'HEAD')

def describe(**kwargs):
    print(kwargs)  # dict of keyword arguments

describe(environment="qa", ready=True)
# {'environment': 'qa', 'ready': True}
```

The same star syntax can *unpack* values:

```python
parts = ["git", "status", "--short"]
command(*parts)   # equivalent to command('git', 'status', '--short')
```

### Scope and return semantics

```python
counter = 10

def calculate():
    counter = 2       # local variable, not the global one
    return counter * 3

print(calculate())     # 6
print(counter)         # 10
```

A function without an explicit `return` returns `None`.

**Mutable default pitfall:** Don't write `def add(item, results=[]):`—the same list is reused across calls. Prefer:

```python
def add(item, results=None):
    if results is None:
        results = []
    results.append(item)
    return results
```

## 10. Comprehensions, unpacking, generators and `yield`

A **list comprehension** is a compact loop that creates a list:

```python
names = ["FRONTEND", "ORDERS"]
lowercase = [name.lower() for name in names]
print(lowercase)  # ['frontend', 'orders']
```

Filter while creating:

```python
numbers = [1, 2, 3, 4, 5]
even = [n for n in numbers if n % 2 == 0]  # [2, 4]
```

**Dictionary comprehension:**

```python
fact = {"repository": "org/repo", "commit": "abcd", "token": "secret"}
allowed = {k: fact[k] for k in ("repository", "commit")}
print(allowed)  # excludes token
```

**Set comprehension:**

```python
extensions = {p.rsplit(".", 1)[-1] for p in ["a.json", "b.json", "c.yaml"]}
```

**Generator expression:** lazy values instead of immediately building a list:

```python
print(any(n > 10 for n in [3, 12, 7]))   # True
print(all(n > 0 for n in [3, 12, 7]))    # True
```

`yield` creates a generator function:

```python
def batches(items, size):
    for i in range(0, len(items), size):
        yield items[i:i + size]

for batch in batches([1, 2, 3, 4, 5], 2):
    print(batch)  # [1,2], [3,4], [5]
```

### `next()` and iterator patterns

```python
runs = [{"status": "failed"}, {"status": "success"}]
first_success = next((r for r in runs if r["status"] == "success"), None)
print(first_success)
```

`next(generator, None)` returns `None` when no item matches instead of raising `StopIteration`.

## 11. Error handling and validation

```python
def require(condition, message):
    if not condition:
        raise ValueError(message)


def validate_environment(env):
    require(env in {"dev", "qa", "production"}, "invalid environment")
    return env

try:
    validate_environment("demo")
except ValueError as exc:
    print(f"Validation failed: {exc}")
```

- `raise` creates/propagates an exception.
- `try` marks code that might fail.
- `except` handles only specified exceptions.
- `else` (optional) runs when no exception happens.
- `finally` (optional) always runs, useful for cleanup.

```python
try:
    number = int("4")
except ValueError:
    print("Bad integer")
else:
    print("Parsed:", number)
finally:
    print("Finished")
```

Avoid `except Exception: pass`: it can hide failures. Catch the errors you can handle, log context without exposing secrets, and let unexpected failures stop the CI job.

## 12. Modules, imports, Python standard library and names

A **module** is a `.py` file; a **package** groups modules. Imports make code reusable.

```python
import json
import os
from pathlib import Path
from datetime import datetime, timezone

print(Path.cwd())
print(os.environ.get("HOME"))
print(datetime.now(timezone.utc))
```

Standard-library modules worth learning: `pathlib`, `os`, `sys`, `json`, `csv`, `argparse`, `subprocess`, `logging`, `hashlib`, `base64`, `re`, `datetime`, `time`, `tempfile`, `shutil`, `concurrent.futures`, `unittest.mock`.

### Create your own module

`validators.py`:

```python
ALLOWED = {"dev", "qa", "production"}

def validate_env(env):
    if env not in ALLOWED:
        raise ValueError(f"unknown environment: {env}")
    return env
```

`main.py`:

```python
from validators import validate_env
print(validate_env("qa"))
```

Run `python main.py` from the directory containing both files. Don't name your own files `json.py`, `logging.py`, `yaml.py`, or `subprocess.py`; that can shadow libraries.

---

<a id="part-ii"></a>

# Part II — Practical Python automation

## 13. Working with paths and files: `pathlib`

Prefer `pathlib.Path` over manually joining strings with `/`.

```python
from pathlib import Path

root = Path.home() / "python-devops"
root.mkdir(parents=True, exist_ok=True)
file = root / "health.txt"
file.write_text("healthy\n", encoding="utf-8")
print(file.read_text(encoding="utf-8").strip())
print(file.exists())
for path in root.glob("*.txt"):
    print(path.name)
```

Useful: `.exists()`, `.is_file()`, `.iterdir()`, `.glob()`, `.rglob()`, `.resolve()`, `.parent`, `.name`, `.suffix`, `.mkdir()`.

A path's existence can change between checking and opening (TOCTOU). For privileged tools, constrain allowed directories and guard against traversal/symlinks when accepting untrusted paths.

### `with` and context managers

```python
with open("status.txt", "w", encoding="utf-8") as target:
    target.write("OK\n")
```

`with` closes the file even if an error occurs. `Path.write_text()` is convenient for simple files, but not automatically atomic.

### Atomic file replacement

For local files, write to a temporary file in the **same directory** and then replace it:

```python
import os
import tempfile
from pathlib import Path


def atomic_text_write(destination: Path, text: str) -> None:
    destination.parent.mkdir(parents=True, exist_ok=True)
    temp_name = None
    try:
        with tempfile.NamedTemporaryFile(
            mode="w", encoding="utf-8", dir=destination.parent,
            prefix=".tmp-", delete=False
        ) as handle:
            temp_name = handle.name
            handle.write(text)
            handle.flush()
            os.fsync(handle.fileno())
        os.replace(temp_name, destination)
        temp_name = None
    finally:
        if temp_name is not None:
            Path(temp_name).unlink(missing_ok=True)
```

`os.replace()` provides atomic replacement on typical local filesystems when source/destination are on the same filesystem. Strong crash durability may additionally require syncing the directory.

## 14. Filesystem and Linux automation

```python
import shutil
from pathlib import Path

usage = shutil.disk_usage("/")
free_gib = usage.free / (1024 ** 3)
print(f"Free space: {free_gib:.2f} GiB")

for log in Path("/var/log").glob("*.log"):
    try:
        print(log.name, log.stat().st_size)
    except PermissionError:
        print("No permission for", log)
```

Do not parse `df -h` if `shutil.disk_usage()` is enough. Prefer standard-library APIs when available; commands remain useful for `systemctl`, `git`, `kubectl`, etc.

**Idempotency:** An operation can run twice without unintentionally multiplying its effect. `Path(...).mkdir(exist_ok=True)` is commonly idempotent; blindly appending the same line to a config file is not.

## 15. Reading environment variables and handling secrets

```python
import os

region = os.getenv("AWS_REGION", "us-east-1")
enabled = os.getenv("DELIVERY_ENABLED", "false").lower() == "true"
token = os.getenv("GITHUB_TOKEN")
if not token:
    raise RuntimeError("GITHUB_TOKEN is required")
# Use token only in an authenticated client, never print it.
```

Environment variables are always **strings**. Convert numbers explicitly:

```python
raw = os.getenv("TIMEOUT_SECONDS", "120")
try:
    timeout = int(raw)
except ValueError as exc:
    raise ValueError("TIMEOUT_SECONDS must be an integer") from exc
if not 1 <= timeout <= 3600:
    raise ValueError("TIMEOUT_SECONDS out of allowed range")
```

Use short-lived GitHub OIDC/AWS role credentials where supported; do not embed static access keys in source code, workflow logs, JSON fixtures, or copied terminal commands.

## 16. JSON, YAML, CSV and TOML

### JSON

```python
import json
from pathlib import Path

record = {"component": "orders", "runId": 42, "ready": True}
Path("release.json").write_text(json.dumps(record, indent=2) + "\n", encoding="utf-8")
loaded = json.loads(Path("release.json").read_text(encoding="utf-8"))
print(loaded["component"])
```

- `json.dumps(obj)` → Python object to JSON **string**.
- `json.loads(text)` → JSON **string** to Python object.
- `json.dump(obj, file)` / `json.load(file)` work with open file handles.
- JSON booleans `true/false` become Python `True/False`; JSON `null` becomes `None`.

### YAML

Install an external package inside your venv:

```bash
python -m pip install PyYAML
```

```python
import yaml

manifest = """
apiVersion: v1
kind: ConfigMap
metadata:
  name: example
"""
obj = yaml.safe_load(manifest)
print(obj["metadata"]["name"])
```

Use `yaml.safe_load()` or `yaml.safe_load_all()` for untrusted input. Avoid `yaml.load()` with unsafe loaders. For multi-document Kubernetes YAML:

```python
documents = list(yaml.safe_load_all("kind: Service\n---\nkind: Deployment\n"))
```

**Warning:** Parsing safely does **not** mean a Kubernetes manifest is approved or safe to apply. Validate its policy, resource types, target namespace and provenance separately.

### CSV

```python
import csv
with open("hosts.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["host", "role"])
    writer.writeheader()
    writer.writerow({"host": "server01", "role": "jenkins"})
```

### TOML (Python 3.11+)

```python
import tomllib
from pathlib import Path
settings = tomllib.loads('environment = "dev"\nreplicas = 2\n')
print(settings["replicas"])
```

`tomllib` reads TOML but does not write it. `pyproject.toml` is a standard place for Python project configuration.

## 17. Regular expressions and basic identity validation

`re.fullmatch` requires the **entire** input to match:

```python
import re

sha = "a" * 40
assert re.fullmatch(r"[a-f0-9]{40}", sha)

image_digest = "sha256:" + "b" * 64
assert re.fullmatch(r"sha256:[a-f0-9]{64}", image_digest)
```

Useful patterns: `\d` digit; `\w` word character; `{40}` exact length; `+` one or more; `*` zero or more; `^` start; `$` end. Prefix pattern strings with `r` to avoid Python interpreting backslashes.

Use regex for **format**, not for authenticating provenance. A syntactically valid digest could still point to an untrusted image.

## 18. `subprocess`: invoke Git, `kubectl`, `gh`, `docker` safely

```python
import subprocess

result = subprocess.run(
    ["git", "rev-parse", "--show-toplevel"],
    check=True,
    capture_output=True,
    text=True,
    timeout=15,
)
print(result.stdout.strip())
```

- First argument is a **list of arguments**, not a shell script.
- `check=True`: raise `CalledProcessError` on nonzero status.
- `capture_output=True`: capture stdout/stderr.
- `text=True`: decode bytes into text.
- `timeout`: avoid hanging indefinitely.
- `.returncode`, `.stdout` and `.stderr` are on the result.

### `subprocess.run()` vs `check_output()`

```python
output = subprocess.check_output(
    ["git", "rev-parse", "HEAD"], text=True, timeout=15
).strip()
```

`check_output()` directly returns stdout and raises on nonzero exit. To pass stdin, `subprocess.run(..., input="text", text=True)` is convenient.

### Never do this with user-controlled input

```python
# BAD: command injection risk when interpolating untrusted strings.
# subprocess.run(f"git show {untrusted_revision}", shell=True)
```

Safer:

```python
revision = "HEAD"  # validate input if it comes from a user or PR
subprocess.run(["git", "show", revision], check=True, timeout=15)
```

**Nuance:** An argument list prevents **shell** injection, but doesn't prevent malicious CLI flags. Validate untrusted arguments and use `--` to terminate options where the command supports it. Never execute code or scripts from an untrusted PR with powerful credentials.

### Errors and retries

```python
try:
    result = subprocess.run(["git", "status", "--short"], check=True,
                            text=True, capture_output=True, timeout=20)
except subprocess.TimeoutExpired:
    print("Git command timed out")
except subprocess.CalledProcessError as exc:
    print(f"Git failed: exit {exc.returncode}")
```

Only retry operations that are **safe to repeat**. Retries of writes, releases, signatures and merges need explicit idempotency safeguards.

## 19. Dates, timers, retries and exponential backoff

```python
import time
from datetime import datetime, timezone, timedelta

now = datetime.now(timezone.utc)
print(now.isoformat())
expires = now + timedelta(minutes=15)
```

Use timezone-aware datetimes for timestamps. For elapsed time or polling deadlines, use `time.monotonic()` instead of wall time:

```python
import time

def wait_until(predicate, timeout=10, interval=1):
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        if predicate():
            return True
        time.sleep(interval)
    return False
```

Exponential backoff is commonly `min(base * 2 ** attempt, cap)` plus **jitter**. Respect server `Retry-After` for rate limits. Never turn a permanent authorization or validation failure into an infinite retry.

## 20. Command-line tools with `argparse`

Create `release_cli.py`:

```python
import argparse


def main():
    parser = argparse.ArgumentParser(description="Validate a release destination")
    parser.add_argument("environment", choices=["dev", "qa", "production"])
    parser.add_argument("--timeout", type=int, default=120)
    parser.add_argument("--dry-run", action="store_true")
    args = parser.parse_args()

    if not 1 <= args.timeout <= 3600:
        parser.error("--timeout must be between 1 and 3600")

    print(f"env={args.environment}, timeout={args.timeout}, dry_run={args.dry_run}")


if __name__ == "__main__":
    main()
```

Run:

```bash
python release_cli.py qa --timeout 60 --dry-run
python release_cli.py --help
python release_cli.py unknown   # argparse rejects it
```

`action="store_true"` makes a Boolean flag. Positional arguments are required unless `nargs` makes them optional. `parser.error()` prints an error and exits nonzero.

## 21. Logging, exit codes and production diagnostics

```python
import logging
import sys

logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger(__name__)


def main() -> int:
    logger.info("Checking release readiness")
    try:
        raise ValueError("Missing component")
    except ValueError as exc:
        logger.error("Validation rejected: %s", exc)
        return 2


if __name__ == "__main__":
    sys.exit(main())
```

- Exit `0` = success. Any nonzero exit usually fails a CI step.
- Use `logger.info`, `.warning`, `.error`, `.exception` instead of indiscriminate `print`.
- `logger.exception()` inside `except` emits a traceback; avoid it when the traceback could expose secrets.
- Separate user-facing messages from machine-readable output; structured JSON logs are helpful at scale.

## 22. HTTP APIs using `requests`

```bash
python -m pip install requests
```

```python
import requests

url = "https://api.github.com/repos/python/cpython"
response = requests.get(url, timeout=(5, 20), headers={"Accept": "application/vnd.github+json"})
response.raise_for_status()
data = response.json()
print(data["full_name"])
```

- `timeout=(connect_timeout, read_timeout)` prevents endless waits.
- `raise_for_status()` raises for HTTP 4xx/5xx responses.
- `.json()` parses the response body, but it can fail for invalid JSON.
- For authenticated GitHub calls, use `Authorization: Bearer ...` and avoid logging headers.

### Pagination and limits

An API returning the first 100 entries may not contain the entire result. Follow provider pagination (GitHub's `Link` header or documented pagination endpoints), cap total pages, honor rate-limit headers, and validate schemas. Not every endpoint has the same pagination behavior.

### Read-only API wrapper

```python
import requests


def fetch_json(url: str, token: str | None = None) -> dict:
    headers = {"Accept": "application/vnd.github+json"}
    if token:
        headers["Authorization"] = f"Bearer {token}"
    response = requests.get(url, headers=headers, timeout=(5, 20))
    response.raise_for_status()
    value = response.json()
    if not isinstance(value, dict):
        raise ValueError("Expected a JSON object")
    return value
```

Avoid arbitrary caller-provided URLs in privileged code (server-side request forgery); use an allowlisted API origin or a fixed base URL.

## 23. Hashing, encoding and what they do *not* prove

```python
import base64
import hashlib
import json

fact = {"component": "orders", "runId": 42}
canonical = json.dumps(fact, sort_keys=True, separators=(",", ":")).encode("utf-8")
release_id = hashlib.sha256(canonical).hexdigest()
print(release_id)

encoded = base64.b64encode(b"hello").decode("ascii")
assert base64.b64decode(encoded) == b"hello"
```

- **SHA-256 hash:** stable content fingerprint, not identity verification.
- **Git commit SHA:** identifies a specific Git object/history commit.
- **Container image digest:** identifies content in a registry (subject to the referenced content and digest algorithm).
- **Base64:** reversible encoding, **not encryption**.
- **Digital signature/attestation:** can provide cryptographically verifiable provenance when trusted identities, subjects, issuers and payload are checked.

Don't confuse “matches a 64-digit digest format” with “this image is trusted.”

---
<a id="part-iii"></a>

# Part III — Python in real DevOps platforms

## 24. Python project layout, dependencies and packaging

A one-file script works for experiments, but shared tooling needs an intentional structure:

```text
release-audit/
├── pyproject.toml
├── README.md
├── .gitignore
├── src/
│   └── release_audit/
│       ├── __init__.py
│       ├── cli.py
│       ├── models.py
│       ├── checks.py
│       └── github.py
└── tests/
    ├── test_checks.py
    └── test_cli.py
```

Example `pyproject.toml` (install this only after creating the files):

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "release-audit"
version = "0.1.0"
description = "Read-only GitOps release audit CLI"
requires-python = ">=3.11"
dependencies = ["requests>=2.31,<3", "PyYAML>=6,<7"]

[project.optional-dependencies]
dev = ["pytest>=8,<10", "ruff>=0.8,<1", "mypy>=1.10,<2"]

[project.scripts]
release-audit = "release_audit.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.ruff]
line-length = 100

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Install editable (changes in `src/` are reflected without reinstalling):

```bash
python -m pip install -e '.[dev]'
python -m pytest -q
python -m ruff check .
```

Packaging terms:

- **`pip`**: installs Python distributions.
- **`venv`**: isolates project packages.
- **`pyproject.toml`**: build metadata, dependencies, tool configuration.
- **wheel**: built installation artifact.
- **editable install**: import source code from your working directory.
- **`requirements.txt`**: simple requirements input; for applications and security-sensitive CI, use a reviewed lock strategy or pinned, hashed dependencies. A broad version range alone is not reproducible.

To build a wheel: `python -m pip install build && python -m build`. Pin **GitHub Actions** by verified full commit SHAs in critical production workflows; pin Python package dependencies using your team's approved reproducible dependency workflow.

## 25. Practical Git automation with Python

Some Git operations are easier through its CLI than an external Python Git library. Check you're inside the intended repository before running commands.

```python
import re
import subprocess


def git(*args: str) -> str:
    result = subprocess.run(
        ["git", *args], text=True, capture_output=True, check=True, timeout=30
    )
    return result.stdout.strip()


def current_commit() -> str:
    value = git("rev-parse", "HEAD")
    if not re.fullmatch(r"[a-f0-9]{40}", value):
        raise ValueError("Unexpected Git object ID format")
    return value


print(current_commit())
```

**Git SHA note:** The 40-hex-character check assumes SHA-1-format Git repositories. Git can also support different object formats. For a fixed enterprise workflow, validate the repository's expected object format explicitly.

Useful Git CLI commands:

```bash
git rev-parse HEAD
git rev-parse --show-toplevel
git show HEAD:environments/dev/release.json
git diff --name-only BASE HEAD --
git ls-tree --name-only HEAD -- charts/
```

Security considerations: untrusted refnames/path arguments need validation; prefer `--` before pathspecs, and don't run unchecked Git hooks or repository-provided executable files with privileged credentials.

## 26. Python inside GitHub Actions

A workflow can run your Python script and use its exit status to pass/fail a job.

```yaml
name: Python release validation
on:
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Validate
        run: |
          python -m pip install -e '.[dev]'
          python -m ruff check .
          python -m pytest -q
```

The action version tags above make a learning example easier to read; production security-sensitive workflows should pin actions to **reviewed full commit SHAs** and update them through a controlled process. Do not make a PR workflow privileged merely to retrieve information.

### GitHub Actions environment files

```python
import os
from pathlib import Path

output_path = os.getenv("GITHUB_OUTPUT")
if output_path:
    with Path(output_path).open("a", encoding="utf-8") as f:
        f.write("release_id=abc123\n")

summary_path = os.getenv("GITHUB_STEP_SUMMARY")
if summary_path:
    with Path(summary_path).open("a", encoding="utf-8") as f:
        f.write("### Release validation\n\nPassed.\n")
```

`GITHUB_OUTPUT` and `GITHUB_STEP_SUMMARY` are file paths provided by the runner. They are not normal environment-variable values to overwrite. For multi-line output, follow GitHub's documented delimiter format. Treat untrusted PR text as untrusted—don't let a newline inject extra output keys or summary content containing secrets.

### Job outputs vs Python `return`

A Python function `return` goes back to its caller **within the Python process**. A GitHub Actions step output is persisted by the runner for later steps/jobs. To expose it to another job, declare the job output in workflow YAML.

## 27. GitHub REST API: workflow runs, releases and PRs

Using `requests` with a bearer token:

```python
import os
import requests

BASE = "https://api.github.com"
TOKEN = os.environ.get("GITHUB_TOKEN")


def github_get(path: str):
    if not path.startswith("/") or path.startswith("//"):
        raise ValueError("Path must be an absolute API path")
    headers = {"Accept": "application/vnd.github+json", "X-GitHub-Api-Version": "2022-11-28"}
    if TOKEN:
        headers["Authorization"] = f"Bearer {TOKEN}"
    response = requests.get(BASE + path, headers=headers, timeout=(5, 20))
    response.raise_for_status()
    return response.json()


repo = github_get("/repos/python/cpython")
print(repo["full_name"])
```

**Important:** In production, avoid reading a token into a global variable at import time. Inject an authenticated client or look up the token when the command runs. Use a `requests.Session` for connection reuse and test it with mocks.

For workflow run identity, inspect relevant API fields—`conclusion`, `event`, `head_branch`, `head_sha`, `path`, and repository identity. A successful run **alone** is not sufficient: verify it belongs to the correct workflow, repository, branch, and commit.

For a PR, check author identity **and** actual modified files, head repository, target branch, commits and required checks. No single field should be trusted in isolation.

**Do not automatically approve production deployments** from the same process that constructs the release. Separate review and merge privileges.

## 28. Docker-related Python automation

### Read-only Docker CLI inspection

```python
import json
import subprocess

result = subprocess.run(
    ["docker", "ps", "--format", "{{json .}}"],
    capture_output=True, text=True, check=True, timeout=15,
)
containers = [json.loads(line) for line in result.stdout.splitlines() if line.strip()]
for item in containers:
    print(item["Names"], item["Status"])
```

Requires Docker installed and permission to access the daemon. **Docker socket access is effectively highly privileged** on a typical rootful host; never expose it to untrusted PR workloads or casually mount `/var/run/docker.sock` into containers.

### Alternative: Docker SDK

```bash
python -m pip install docker
```

```python
import docker

client = docker.from_env()
try:
    for container in client.containers.list():
        print(container.name, container.status)
finally:
    client.close()
```

**Practice:** Add an allowlist of expected container names and report missing/running/stopped containers, but don't restart production containers until you have authorization, tests and a recovery plan.

## 29. Kubernetes using the Python client

```bash
python -m pip install kubernetes
```

```python
from kubernetes import client, config

config.load_kube_config()  # local kubeconfig; requires a reachable cluster
api = client.CoreV1Api()
result = api.list_namespaced_pod(namespace="default", _request_timeout=20)
for pod in result.items:
    phase = pod.status.phase
    print(pod.metadata.name, phase)
```

When running **inside** a cluster, use `config.load_incluster_config()` with a narrowly scoped service account. RBAC should grant only the resources and verbs required.

### Detect non-running pods

```python
unhealthy = [pod.metadata.name for pod in result.items if pod.status.phase != "Running"]
if unhealthy:
    raise RuntimeError(f"Non-running pods: {', '.join(unhealthy)}")
```

**Caveat:** A pod in `Running` phase is not necessarily *ready*. A robust health check also examines container readiness, deployment status, desired vs available replicas, and application-level probes. `Succeeded` is normal for some Job pods. Make the checks workload-specific.

**Kind lab:** Point `KUBECONFIG` at your local cluster. Read namespace/pod/deployment status only. No cloud spend required.

## 30. AWS automation with Boto3 (read-only first)

```bash
python -m pip install boto3
aws sts get-caller-identity
```

Boto3 uses the standard AWS credential/provider chain. Prefer temporary assumed-role credentials, SSO or OIDC rather than hardcoded keys.

### Verify identity and account

```python
import boto3

sts = boto3.client("sts")
identity = sts.get_caller_identity()
print(identity["Account"], identity["Arn"])
```

### List EC2 instances using a paginator

```python
import boto3

client = boto3.client("ec2", region_name="us-east-1")
pages = client.get_paginator("describe_instances").paginate()
for page in pages:
    for reservation in page.get("Reservations", []):
        for instance in reservation.get("Instances", []):
            print(instance["InstanceId"], instance["State"]["Name"])
```

**Why pagination?** API responses may be truncated. Paginators follow service-specific continuation tokens and are preferable to silently inspecting only the first page.

### S3 object inventory

```python
import boto3

s3 = boto3.client("s3")
bucket = "REPLACE_WITH_YOUR_EXISTING_BUCKET"
for page in s3.get_paginator("list_objects_v2").paginate(Bucket=bucket, Prefix="logs/"):
    for obj in page.get("Contents", []):
        print(obj["Key"], obj["Size"])
```

Only run against an existing bucket you have permission to inspect; the example doesn't create a bucket. Do not rely on S3 listing as a constant-time or free operation for millions of objects—review request/transfer costs.

### AWS reliability and safety checklist

- Confirm the account and region before operating (`sts.get_caller_identity()`).
- Use least-privilege roles and separate read-only from write permissions.
- Use service-specific paginators, waiters and retry configuration.
- Prefer explicit tags/allowlists over “all resources”.
- For mutation tools, implement `--dry-run`, approval, concurrency/race controls and audit logs. Some AWS `DryRun` flags only test permissions, not simulate changes.
- Free-tier assumptions change: verify current pricing for actions you actually run.

## 31. Type hints, dataclasses and object-oriented Python

Type hints make scripts understandable and help static analyzers, but **do not enforce types at runtime**.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ReleaseFact:
    component: str
    repository: str
    commit: str
    run_id: int


def describe(fact: ReleaseFact) -> str:
    return f"{fact.component} at {fact.commit[:12]}"

fact = ReleaseFact("orders", "org/orders", "a" * 40, 123)
print(describe(fact))
```

A **class** groups data and behavior; `self` is a reference to the instance. A dataclass auto-generates methods such as `__init__` and `__repr__`. `frozen=True` blocks normal reassignment of its fields but does not recursively freeze mutable field contents.

### Basic custom class

```python
class DeploymentGate:
    def __init__(self, allowed: set[str]):
        self.allowed = allowed

    def check(self, environment: str) -> bool:
        return environment in self.allowed


gate = DeploymentGate({"dev", "qa"})
print(gate.check("production"))  # False
```

Use functions and dictionaries for small scripts; introduce classes when they simplify shared state, dependencies or interfaces. Avoid unnecessary inheritance.

### Type annotations you will see

```python
def check(name: str, retries: int = 3) -> bool:
    return bool(name) and retries > 0

images: dict[str, str] = {"orders": "sha256:..."}
names: list[str] = ["orders", "cart"]
commit: str | None = None   # union: string or None
```

Run `python -m mypy src` after installing `mypy`. Consider `TypedDict` for a predictable JSON dictionary schema and runtime validation for external API payloads.

## 32. Polling, concurrency, threads, processes and asyncio

### Thread pools for I/O-bound operations

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time


def check_host(name: str) -> str:
    time.sleep(0.1)   # stand-in for a network request
    return f"{name}: checked"

hosts = ["web01", "web02", "web03"]
with ThreadPoolExecutor(max_workers=3) as pool:
    futures = {pool.submit(check_host, h): h for h in hosts}
    for future in as_completed(futures):
        print(future.result())  # completion order may differ
```

Threads are useful for I/O waits; CPU-heavy work often benefits from processes or native libraries, depending on runtime and workload. Always limit concurrency, propagate errors, and respect API quotas.

### Basic `asyncio`

```python
import asyncio

async def inspect(name: str) -> str:
    await asyncio.sleep(0.1)
    return f"{name}: checked"

async def main():
    results = await asyncio.gather(inspect("api"), inspect("worker"))
    print(results)

asyncio.run(main())
```

- `async def` defines a coroutine.
- `await` suspends until an awaitable completes.
- `asyncio.gather` combines several awaitables.
- Plain blocking functions (such as `requests.get()` or `time.sleep()`) still block the event loop; use async-capable libraries or offload blocking work.

**Do not** parallelize multiple mutating production deployments just because concurrency is possible. Coordination and idempotency matter more than speed.

## 33. Iterators, decorators and context managers (advanced syntax)

### Decorator

```python
from functools import wraps
import time


def timed(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.monotonic()
        try:
            return func(*args, **kwargs)
        finally:
            print(f"{func.__name__}: {time.monotonic() - start:.3f}s")
    return wrapper


@timed
def validate_release():
    return True

validate_release()
```

`@timed` is a concise way of writing `validate_release = timed(validate_release)`. `@wraps` preserves the original function's metadata. Decorators are used for tracing, retrying, caching and authorization—but indiscriminate retry decorators can repeat unsafe writes.

### Custom context manager

```python
from contextlib import contextmanager

@contextmanager
def task(name: str):
    print("start", name)
    try:
        yield
    finally:
        print("finish", name)

with task("validate"):
    print("working")
```

This demonstrates a generator-backed context manager. It runs cleanup even when the body raises an exception.

## 34. Testing with pytest and mocking external systems

Install:

```bash
python -m pip install pytest
```

`validator.py`:

```python
ALLOWED = {"dev", "qa", "production"}


def check_environment(name: str) -> str:
    if name not in ALLOWED:
        raise ValueError("unknown environment")
    return name
```

`test_validator.py`:

```python
import pytest
from validator import check_environment


def test_qa_accepted():
    assert check_environment("qa") == "qa"


@pytest.mark.parametrize("value", ["Prod", "staging", "", None])
def test_invalid_rejected(value):
    with pytest.raises(ValueError, match="unknown environment"):
        check_environment(value)
```

Run `python -m pytest -q`. `assert` checks an expectation in test code. For critical production input validation, use explicit `if/raise` or `require`, **not** Python `assert`, because `python -O` can remove assert statements.

### Mock an external command without running it

`status.py`:

```python
import subprocess


def git_branch():
    return subprocess.run(["git", "branch", "--show-current"],
                          capture_output=True, text=True, check=True,
                          timeout=10).stdout.strip()
```

`test_status.py`:

```python
from unittest.mock import patch
from subprocess import CompletedProcess
from status import git_branch


def test_git_branch():
    fake = CompletedProcess(["git"], 0, stdout="main\n", stderr="")
    with patch("status.subprocess.run", return_value=fake) as mocked:
        assert git_branch() == "main"
        mocked.assert_called_once()
```

The patch target is where the function **looks up** the dependency (`status.subprocess.run`), not necessarily where it was originally defined.

### Test categories

| Test | What it answers | External dependencies? |
|---|---|---|
| Unit | Is one function correct? | Usually mocked |
| Integration | Do components work together? | Sometimes |
| Contract | Does payload match expected schema? | Can use fixtures |
| End-to-end | Does the actual delivery path work? | Usually yes; use disposable environment |
| Security/negative | Does invalid/untrusted input fail safely? | Often unit + integration |

For DevOps tooling, negative-path tests (wrong repo, wrong SHA, failed CI, invalid signature, stale status, timeout) are as important as successful cases.

## 35. Linting, formatting, type checks and CI quality gates

```bash
python -m pip install ruff mypy pytest
python -m ruff check .
python -m ruff format --check .
python -m mypy src
python -m pytest -q
```

`ruff` catches style and many correctness problems; `mypy` examines type annotations; `pytest` verifies behavior. None individually proves security or correctness. Add dependency checks (for example, `pip-audit`), SBOM generation and policy verification as appropriate.

Typical pipeline order:

```text
checkout → setup Python → install locked dependencies
→ lint/format → type checks → tests → dependency/security checks
→ build/package → publish immutable artifact → attest/sign
```

A failure must exit nonzero and stop publication. Upload test evidence only from trusted paths and avoid leaking tokens into artifacts.

## 36. Secure coding and reliability checklist

### Input trust boundaries

Treat as untrusted: PR body, branch names, path strings, workflow dispatch payload, external API response, YAML in a candidate branch, registry metadata and environment variables supplied by callers.

Validate **shape**, **types**, **allowlisted values**, **identity** and **relationship to trusted evidence**. Example:

```python
import re


def valid_release_fact(fact: object) -> dict:
    if not isinstance(fact, dict):
        raise ValueError("release fact must be an object")
    expected_keys = {"repository", "commit", "runId"}
    if set(fact) != expected_keys:
        raise ValueError("unexpected fields")
    if fact["repository"] not in {"org/orders", "org/frontend"}:
        raise ValueError("untrusted repository")
    if not isinstance(fact["commit"], str) or not re.fullmatch(r"[a-f0-9]{40}", fact["commit"]):
        raise ValueError("invalid commit")
    if type(fact["runId"]) is not int or fact["runId"] <= 0:
        raise ValueError("invalid workflow run id")
    return fact
```

Why `type(x) is int`? In Python, `bool` is a subclass of `int`:

```python
print(isinstance(True, int))  # True
print(type(True) is int)      # False
```

For a run ID, `True` should **not** pass integer validation.

### Operational rules

1. **Least privilege:** read-only by default; isolate code that writes/merges/deploys.
2. **Validate before acting:** fail closed on missing evidence.
3. **No shell injection:** argument arrays; additionally guard CLI flags and paths.
4. **No code execution from untrusted branches** in credential-bearing workflows.
5. **Timeouts everywhere:** APIs, subprocesses, polling, deployment gates.
6. **Retry only transient, idempotent requests** with exponential backoff/jitter and a cap.
7. **Avoid stale evidence:** bind statuses/attestations to the exact run, commit, image digest, configuration revision and expected bot identity.
8. **Protect secrets:** use trusted secret stores, OIDC where possible, redact logs.
9. **Dry-run + explicit approval** for dangerous mutations.
10. **Test negative paths and races:** superseded PR head, rejected approval, closed PR, conflicting release, delayed event, failed rollout.
11. **Log safe context:** environment, release ID prefix, run ID and trace/correlation ID; not authorization headers.
12. **Review dependencies and supply chain:** pin or lock versions; verify action/container identities.

### Common mistakes

| Mistake | Better approach |
|---|---|
| `except: pass` | Catch known exceptions and log/propagate |
| `bool("false")` | Parse explicit accepted values |
| `os.system(f"...")` | `subprocess.run([arg, ...], check=True, timeout=...)` |
| `yaml.load(untrusted)` | `yaml.safe_load()` + schema/policy validation |
| `requests.get(url)` with no timeout | Set connect/read timeouts |
| Print secrets for debugging | Redact; log metadata instead |
| `assert` for security checks | `if not condition: raise ValueError(...)` |
| Recreate resources on every run | Read first; converge toward desired state |
| Trust an image because digest format matches | Verify cryptographic provenance and policy |
| Assume a CI run proves deployment health | Check trusted deployment/health evidence |
| Retry any failed API call | Retry only safe, transient operations |

## 37. Observability: logs, metrics, tracing and structured events

In a pipeline spanning repositories, logs should correlate operations using shared IDs.

```python
import json
import logging

logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger("delivery")


def event(name: str, *, environment: str, release_id: str, status: str) -> None:
    logger.info(json.dumps({
        "event": name,
        "environment": environment,
        "release_id": release_id,
        "status": status,
    }, sort_keys=True))


event("promotion_checked", environment="qa", release_id="abc123", status="accepted")
```

Avoid logging entire webhook payloads; they might contain sensitive data. Persist structured audit events with appropriate retention. For metrics, track request/error rates, gate duration, retries, and time-to-deploy; don't attach unbounded labels such as a unique commit SHA to every Prometheus metric.

Across GitHub Actions, prefer a central workflow summary and evidence links rather than relying on job names to reconstruct the delivery graph.

## 38. How to read a large production Python script

A reliable technique when facing a 600-line release controller:

1. **Find `if __name__ == '__main__'` and `main()`.** Map command-line operations.
2. **Find global constants** (`COMPONENTS`, `ENVIRONMENTS`, trusted builder IDs). Identify policy boundaries.
3. **Inspect I/O wrappers** (`run`, `api`, `read`, `write`). These are its external effects.
4. **Read validators** before workflows. Understand what inputs must be true.
5. **Trace one happy path**: CLI `stage dev` → select facts → validate → open PR → wait for evidence.
6. **Trace one failure path**: invalid source CI → verification failure → nonzero CI step.
7. **Trace rollback** separately from promotion, including approval and current trusted health.
8. **List all permissions** required by each path and who can supply input.
9. **Map loops/retries** and prove they have bounds.
10. **Write a call graph** to identify dependencies and mockable API boundaries.

### Syntax decoder for patterns found in advanced delivery scripts

```python
# 1. *args: take a variable number of positional arguments.
def run(*args):
    return args

# 2. .get(): safe dictionary lookup.
state = {"status": "success"}
print(state.get("conclusion"))    # None

# 3. dict comprehension: select trusted fields.
fact = {"component": "orders", "runId": 3, "extra": "ignored"}
selected = {key: fact[key] for key in ("component", "runId")}

# 4. Generator expression with next(): find first matching item.
checks = [{"state": "pending"}, {"state": "success"}]
match = next((x for x in checks if x["state"] == "success"), None)

# 5. Conditional expression (ternary).
env = "prod"
timeout = 3600 if env == "prod" else 600

# 6. f-string expressions.
path = f"environments/{env}/release.json"

# 7. Unpacking a tuple into variables.
repository, commit = ("org/orders", "a" * 40)

# 8. Unpack tuple/list arguments into a call.
args = ("git", "status")
print(*args)  # git status

# 9. Nested for comprehension.
pages = [[1, 2], [3]]
records = [record for page in pages for record in page]

# 10. Explicit object/type checks.
if type(fact["runId"]) is not int:
    raise ValueError("run ID must be a real integer")
```

### Release workflow model

```text
Application CI success on protected main
        |
        v
Publish immutable image + source-bound attestations
        |
        v
Python intake validates facts, workflow identity and signatures
        |
        v
Generate canonical release bundle and stable release ID
        |
        v
GitOps PR for dev → required checks/merge → observer health status
        |
        v
Promote identical bundle to QA → verify health
        |
        v
Production PR → independent code-owner review/merge
        |
        v
Observe production rollout + PostSync/smoke evidence
        |
        +----failure----> verify previous healthy state → rollback PR
```

This is a **conceptual architecture**, not a claim that every GitOps project implements every guard. The exact guarantees depend on your workflow permissions, policy files, signature verification settings and who can write deployment statuses.

---
<a id="part-iv"></a>

# Part IV — Guided hands-on labs (complete exercises)

The labs use the same working directory: `~/python-devops`. Start each session by activating `.venv` and running `pwd`. All labs labeled **local** work without a GitHub token, Kubernetes cluster, AWS account or Docker daemon. Each lab uses a distinct filename so earlier labs are not overwritten.

## 39. Lab 01 — Your first operational Python script (local)

**Goal:** Read inputs, compute a result, print an actionable status, return an exit code.

Create `lab01_capacity.py`:

```python
import sys


def capacity_status(used: int, total: int, threshold: float = 80.0) -> str:
    if total <= 0 or used < 0 or used > total:
        raise ValueError("Invalid capacity values")
    percentage = 100 * used / total
    return "WARNING" if percentage >= threshold else "OK"


def main() -> int:
    used, total = 85, 100
    state = capacity_status(used, total)
    print(f"Storage: {used}/{total} GiB => {state}")
    return 1 if state == "WARNING" else 0


if __name__ == "__main__":
    sys.exit(main())
```

Run:

```bash
python lab01_capacity.py
# Storage: 85/100 GiB => WARNING
echo $?   # 1 (on Linux/WSL)
```

**Inspect:** Change `used` to `75` and rerun; exit should become `0`. Try `total=0` and explain the exception.

**Challenge:** Add an `CRITICAL` status at 95% and return exit `2`. Bonus: make the threshold an `argparse` option.

## 40. Lab 02 — Host inventory processing with lists/dicts (local)

**Goal:** Iterate nested data, filter, count and summarize.

`lab02_inventory.py`:

```python
hosts = [
    {"name": "server01", "role": "jenkins", "environment": "dev", "healthy": True},
    {"name": "server02", "role": "worker", "environment": "qa", "healthy": False},
    {"name": "server03", "role": "worker", "environment": "dev", "healthy": True},
]

for host in hosts:
    print(f"{host['name']:<10} {host['role']:<8} {host['environment']}")

unhealthy = [h["name"] for h in hosts if not h["healthy"]]
counts = {env: sum(h["environment"] == env for h in hosts)
          for env in {h["environment"] for h in hosts}}
print("Unhealthy:", unhealthy)
print("Environment counts:", counts)
```

Run: `python lab02_inventory.py`.

Expected facts: exactly one unhealthy host (`server02`); two `dev` hosts; one `qa` host. Dictionary ordering in printed output should not be treated as meaningful here because the outer comprehension iterates a set.

**Challenge:** Build `{role: [hostnames...]}`. Reject records without a `name` key instead of silently ignoring them.

## 41. Lab 03 — Validate a release JSON schema (local)

**Goal:** Translate a CI/CD release contract into defensive Python.

Create `lab03_release.json`:

```json
{
  "component": "orders",
  "repository": "example/orders",
  "commit": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "runId": 123,
  "digest": "sha256:bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
}
```

Create `lab03_validate.py`:

```python
import json
import re
import sys
from pathlib import Path

EXPECTED = {"component", "repository", "commit", "runId", "digest"}


def validate_fact(value: object) -> dict:
    if not isinstance(value, dict) or set(value) != EXPECTED:
        raise ValueError("Release must be an object with exactly the expected keys")
    if value["component"] not in {"orders", "frontend"}:
        raise ValueError("Unknown component")
    if value["repository"] != f"example/{value['component']}":
        raise ValueError("Repository/component mismatch")
    if not isinstance(value["commit"], str) or not re.fullmatch(r"[a-f0-9]{40}", value["commit"]):
        raise ValueError("Commit must be 40 lowercase hex characters")
    if type(value["runId"]) is not int or value["runId"] <= 0:
        raise ValueError("Run ID must be a positive integer")
    if not isinstance(value["digest"], str) or not re.fullmatch(r"sha256:[a-f0-9]{64}", value["digest"]):
        raise ValueError("Invalid image digest")
    return value


def main() -> int:
    try:
        data = json.loads(Path("lab03_release.json").read_text(encoding="utf-8"))
        fact = validate_fact(data)
        print(f"Valid fact: {fact['component']} @ {fact['commit'][:12]}")
        return 0
    except (OSError, json.JSONDecodeError, ValueError) as exc:
        print(f"Rejected: {exc}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    sys.exit(main())
```

Run:

```bash
python lab03_validate.py
# Valid fact: orders @ aaaaaaaaaaaa
```

**Negative tests:** Change the run ID to `true`, the digest to a tag, or the repository to `example/frontend`. Each should exit nonzero. Restore the original JSON afterward.

**What this does not do:** Cryptographically verify the digest or confirm a GitHub Actions run. You will add those controls later.

## 42. Lab 04 — Inspect Kubernetes YAML safely (local)

**Goal:** Parse a multi-document manifest and report workload resource names.

Install: `python -m pip install PyYAML`.

`lab04_manifests.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
  namespace: dev
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  namespace: dev
spec:
  replicas: 2
```

`lab04_inspect.py`:

```python
from pathlib import Path
import yaml

text = Path("lab04_manifests.yaml").read_text(encoding="utf-8")
for index, obj in enumerate(yaml.safe_load_all(text), start=1):
    if obj is None:
        continue
    if not isinstance(obj, dict) or not isinstance(obj.get("metadata"), dict):
        raise ValueError(f"Document {index} is not a resource")
    kind = obj.get("kind")
    name = obj["metadata"].get("name")
    namespace = obj["metadata"].get("namespace", "default")
    if not isinstance(kind, str) or not isinstance(name, str):
        raise ValueError(f"Document {index} lacks kind/name")
    print(f"{namespace}/{kind}/{name}")
```

Run: `python lab04_inspect.py`.

Expected:

```text
dev/ConfigMap/app-settings
dev/Deployment/orders
```

**Challenge:** Reject any `Secret` manifest. Add an allowlist for `Deployment`, `ConfigMap`, and `Service`. Explain why `safe_load_all` does not replace admission-policy checks.

## 43. Lab 05 — Safe local backup with `pathlib` (local)

**Goal:** Write a local backup without duplication and without touching system files.

`lab05_backup.py`:

```python
from pathlib import Path
import hashlib
import shutil

source = Path("lab05_data")
backup = Path("lab05_backup")
source.mkdir(exist_ok=True)
backup.mkdir(exist_ok=True)
(source / "demo.txt").write_text("important demo configuration\n", encoding="utf-8")

for path in source.glob("*.txt"):
    destination = backup / path.name
    source_hash = hashlib.sha256(path.read_bytes()).hexdigest()
    if destination.exists() and hashlib.sha256(destination.read_bytes()).hexdigest() == source_hash:
        print(f"Unchanged: {path.name}")
        continue
    shutil.copy2(path, destination)
    print(f"Copied: {path.name}")
```

Run twice:

```bash
python lab05_backup.py
# Copied: demo.txt
python lab05_backup.py
# Unchanged: demo.txt
```

**Challenge:** Copy to a timestamped subdirectory, ignore symlinks, and add a `--dry-run` option. Discuss backup integrity vs full disaster-recovery testing.

## 44. Lab 06 — Run Git and handle failure (local, requires Git)

**Goal:** Use `subprocess` with argument lists and clear error handling.

Create a throwaway repository:

```bash
mkdir -p lab06_repo
cd lab06_repo
git init
git -c user.name=Lab -c user.email=lab@example.invalid commit --allow-empty -m 'initial lab'
cd ..
```

`lab06_git.py`:

```python
from pathlib import Path
import subprocess


def git(*args: str) -> str:
    completed = subprocess.run(
        ["git", "-C", str(Path("lab06_repo").resolve()), *args],
        capture_output=True, text=True, check=True, timeout=15,
    )
    return completed.stdout.strip()


try:
    print("HEAD:", git("rev-parse", "HEAD"))
    print("Branch:", git("branch", "--show-current"))
except FileNotFoundError:
    print("Git is not installed")
except subprocess.CalledProcessError as exc:
    print("Git returned a nonzero code:", exc.returncode)
except subprocess.TimeoutExpired:
    print("Git did not respond in time")
```

Run `python lab06_git.py`.

**Challenge:** Add an argument to print `git log -1 --format=%s`. What happens if you pass a nonexistent revision to `git show`?

## 45. Lab 07 — Generate a canonical release identifier (local)

**Goal:** Learn `json.dumps`, stable representation, SHA-256, deterministic outputs.

`lab07_id.py`:

```python
import hashlib
import json


def release_id(components: dict) -> str:
    canonical = json.dumps(components, sort_keys=True, separators=(",", ":"))
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()


one = {"orders": "sha256:bbb", "frontend": "sha256:aaa"}
two = {"frontend": "sha256:aaa", "orders": "sha256:bbb"}
assert release_id(one) == release_id(two)
print(release_id(one))

three = {**one, "orders": "sha256:ccc"}
assert release_id(three) != release_id(one)
```

`**one` expands dictionary entries into a new dictionary; the later `"orders"` entry overrides the original.

**Challenge:** Canonicalize complete structured release facts, not merely image names. Explain why a stable hash does **not** establish the source's authenticity.

## 46. Lab 08 — Wait for an event with a deadline (local)

**Goal:** Practice bounded polling and avoid infinite waits.

`lab08_poll.py`:

```python
import time


def wait_for_ready(check, timeout: float, interval: float = 0.1) -> bool:
    if timeout <= 0 or interval <= 0:
        raise ValueError("Timeout and interval must be positive")
    deadline = time.monotonic() + timeout
    while True:
        if check():
            return True
        remaining = deadline - time.monotonic()
        if remaining <= 0:
            return False
        time.sleep(min(interval, remaining))


attempts = 0

def fake_check():
    global attempts
    attempts += 1
    return attempts >= 3

print(wait_for_ready(fake_check, timeout=1))  # True
print("Checks:", attempts)                    # 3
```

**Challenge:** Replace `global attempts` with a callable class or closure. Add a `max_attempts` guard. In production, a remote “failure” status should raise immediately instead of being polled until timeout.

## 47. Lab 09 — Inspect GitHub repo metadata (optional internet)

**Goal:** Use real HTTP JSON, validate the payload and handle errors.

Install: `python -m pip install requests`.

`lab09_github.py`:

```python
import requests


def get_public_repo(full_name: str) -> dict:
    allowed = {"python/cpython", "kubernetes/kubernetes"}
    if full_name not in allowed:
        raise ValueError("Repository not on this lab's allowlist")
    url = f"https://api.github.com/repos/{full_name}"
    response = requests.get(url, timeout=(5, 20),
                            headers={"Accept": "application/vnd.github+json"})
    response.raise_for_status()
    data = response.json()
    if not isinstance(data, dict) or data.get("full_name") != full_name:
        raise ValueError("Unexpected API response")
    return data


if __name__ == "__main__":
    try:
        repo = get_public_repo("python/cpython")
        print(repo["full_name"], repo["default_branch"])
    except (requests.RequestException, ValueError, KeyError) as exc:
        print("Request failed:", type(exc).__name__, str(exc))
        raise SystemExit(1) from exc
```

This only sends a **read-only, public** API request. Authentication is not required for this example but anonymous rate limits apply.

**Challenge:** Extract `open_issues_count` and explain why it may not exactly equal the number of issues (GitHub historically includes PRs in that field).

## 48. Lab 10 — Audit a local simulated CI status (local)

**Goal:** Validate a combination of attributes rather than trusting one success flag.

`lab10_ci.py`:

```python

def trusted_run(run: dict, expected_repo: str, expected_sha: str) -> bool:
    return (
        run.get("conclusion") == "success"
        and run.get("event") == "push"
        and run.get("head_branch") == "main"
        and run.get("head_sha") == expected_sha
        and run.get("path") == ".github/workflows/ci.yml"
        and run.get("repository") == expected_repo
    )


run = {
    "conclusion": "success", "event": "push", "head_branch": "main",
    "head_sha": "a" * 40, "path": ".github/workflows/ci.yml",
    "repository": "example/orders",
}
print(trusted_run(run, "example/orders", "a" * 40))  # True
run["event"] = "pull_request"
print(trusted_run(run, "example/orders", "a" * 40))  # False
```

**Challenge:** Require an exact workflow run ID. Add a test for a successful workflow from a fork. Later connect this function to a **trusted** GitHub API client.

## 49. Lab 11 — Read-only Docker inventory (optional Docker)

**Goal:** Integrate Python with the Docker CLI without using `shell=True`.

Create `lab11_docker.py`:

```python
import json
import subprocess


def containers() -> list[dict]:
    result = subprocess.run(
        ["docker", "ps", "--all", "--format", "{{json .}}"],
        capture_output=True, text=True, check=True, timeout=20,
    )
    return [json.loads(line) for line in result.stdout.splitlines() if line]


if __name__ == "__main__":
    try:
        items = containers()
        for item in items:
            print(f"{item['Names']}: {item['Status']}")
        print(f"Total: {len(items)}")
    except (FileNotFoundError, subprocess.CalledProcessError) as exc:
        print("Docker not available or permission denied:", exc)
```

This doesn't create, start, stop or delete containers. If Docker isn't installed, read the function and compare it to the `subprocess` lab.

**Challenge:** Report containers whose name begins with `server` and display their state. Do not automatically restart anything.

## 50. Lab 12 — Read-only Kubernetes health snapshot (optional kind cluster)

**Goal:** Query your local cluster from Python.

Create `lab12_k8s.py`:

```python
from kubernetes import client, config
from kubernetes.client.exceptions import ApiException


def list_pod_phases(namespace: str) -> list[tuple[str, str]]:
    config.load_kube_config()
    api = client.CoreV1Api()
    result = api.list_namespaced_pod(namespace=namespace, _request_timeout=20)
    return [(pod.metadata.name, pod.status.phase) for pod in result.items]


if __name__ == "__main__":
    try:
        for name, phase in list_pod_phases("default"):
            print(name, phase)
    except (ApiException, OSError) as exc:
        print("Kubernetes API unavailable:", exc)
        raise SystemExit(1) from exc
```

Prerequisites: `python -m pip install kubernetes`; a working `kubectl get pods -n default` using your current kubeconfig. Kubernetes configuration errors can involve exception types beyond the two handled here; investigate rather than treating them as a healthy empty cluster.

**Challenge:** Count the number of pods per phase. Add deployment ready/desired replica checks; distinguish completed Jobs from long-running services.

## 51. Lab 13 — AWS EC2 inventory without changing resources (optional AWS)

**Goal:** Understand boto3 clients, regions, paginators and AWS identity.

Create `lab13_aws.py`:

```python
import boto3
from botocore.exceptions import BotoCoreError, ClientError, NoCredentialsError


def main():
    session = boto3.Session(region_name="us-east-1")
    identity = session.client("sts").get_caller_identity()
    print("Account:", identity["Account"])
    ec2 = session.client("ec2")
    count = 0
    for page in ec2.get_paginator("describe_instances").paginate():
        for reservation in page.get("Reservations", []):
            for instance in reservation.get("Instances", []):
                print(instance["InstanceId"], instance["State"]["Name"])
                count += 1
    print("Instances:", count)


if __name__ == "__main__":
    try:
        main()
    except (BotoCoreError, ClientError, NoCredentialsError) as exc:
        print("AWS read failed:", type(exc).__name__)
        raise SystemExit(1) from exc
```

Prerequisites: `python -m pip install boto3`, authenticated AWS profile or role and `sts:GetCallerIdentity`/`ec2:DescribeInstances` permissions. This code **does not create, delete, stop or start** resources. API usage may incur charges depending on service and account; inspect current pricing before large-scale inventories.

**Challenge:** Filter instances with a specific tag, include instance type, and export CSV. Test against mocked AWS responses before accessing additional accounts.

<a id="capstone"></a>

## 52. Lab 14 — Complete capstone: a testable GitOps release audit CLI (local)

**Goal:** Combine JSON, YAML-like policy concepts, validation, hashes, argparse, timeouts, logging, testing and CLI exit codes. This initial version is **offline/read-only** and doesn't claim to verify real signatures or remote deployment status.

Create a new directory and files:

```bash
mkdir -p capstone/tests
cd capstone
```

### File A — `release_audit.py`

```python
#!/usr/bin/env python3
"""Offline GitOps release validation: no network, no deployment, no merge."""

import argparse
import hashlib
import json
import logging
import re
import sys
from pathlib import Path

LOG = logging.getLogger("release_audit")
COMPONENTS = {"frontend", "catalogue", "orders", "cart"}
ENVIRONMENTS = ("dev", "qa", "production")
FACT_FIELDS = {"component", "repository", "commit", "runId", "image", "digest"}


def require(condition: bool, message: str) -> None:
    if not condition:
        raise ValueError(message)


def read_json(path: Path) -> dict:
    try:
        value = json.loads(path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError) as exc:
        raise ValueError(f"Cannot read JSON: {path.name}") from exc
    require(isinstance(value, dict), "Expected JSON object")
    return value


def validate_fact(fact: object, name: str) -> dict:
    require(isinstance(fact, dict), f"{name}: fact is not an object")
    require(set(fact) == FACT_FIELDS, f"{name}: unexpected fields")
    require(fact["component"] == name, f"{name}: wrong component")
    require(fact["repository"] == f"example/{name}", f"{name}: wrong source repo")
    require(isinstance(fact["commit"], str) and
            re.fullmatch(r"[a-f0-9]{40}", fact["commit"]), f"{name}: invalid commit")
    require(type(fact["runId"]) is int and fact["runId"] > 0,
            f"{name}: invalid run ID")
    require(isinstance(fact["digest"], str) and
            re.fullmatch(r"sha256:[a-f0-9]{64}", fact["digest"]),
            f"{name}: invalid digest")
    require(fact["image"] == f"ghcr.io/example/{name}",
            f"{name}: unexpected image name")
    return fact


def compute_release_id(components: dict) -> str:
    canonical = json.dumps(components, sort_keys=True, separators=(",", ":"))
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()


def validate_bundle(bundle: object) -> str:
    require(isinstance(bundle, dict) and set(bundle) == {"release", "components"},
            "Unexpected bundle fields")
    meta = bundle["release"]
    components = bundle["components"]
    require(isinstance(meta, dict) and set(meta) == {"id", "ready"},
            "Invalid metadata")
    require(meta["ready"] is True, "Release is not ready")
    require(isinstance(components, dict) and set(components) == COMPONENTS,
            "Exactly four components required")
    for name, fact in components.items():
        validate_fact(fact, name)
    expected = compute_release_id(components)
    require(meta["id"] == expected, "Release ID does not match component facts")
    return expected


def validate_promotion(upstream: dict, candidate: dict) -> str:
    upstream_id = validate_bundle(upstream)
    candidate_id = validate_bundle(candidate)
    require(upstream_id == candidate_id, "Different release ID: rebuild not permitted")
    require(upstream["components"] == candidate["components"],
            "Promotion must preserve exact component facts")
    return candidate_id


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("bundle", type=Path, help="Candidate release JSON path")
    parser.add_argument("--environment", choices=ENVIRONMENTS, required=True)
    parser.add_argument("--upstream", type=Path, help="Upstream release JSON for QA/production")
    parser.add_argument("--verbose", action="store_true")
    args = parser.parse_args(argv)
    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.INFO,
                        format="%(levelname)s: %(message)s")
    try:
        candidate = read_json(args.bundle)
        if args.environment == "dev":
            release_id = validate_bundle(candidate)
        else:
            require(args.upstream is not None, "QA/production require --upstream")
            release_id = validate_promotion(read_json(args.upstream), candidate)
        LOG.info("Validated %s release %s", args.environment, release_id[:16])
        return 0
    except ValueError as exc:
        LOG.error("Release rejected: %s", exc)
        return 2


if __name__ == "__main__":
    sys.exit(main())
```

### File B — `make_fixture.py`

This generates safe synthetic example data, no credentials and no network:

```python
import json
from pathlib import Path
from release_audit import COMPONENTS, compute_release_id


def main():
    components = {}
    for name in sorted(COMPONENTS):
        components[name] = {
            "component": name,
            "repository": f"example/{name}",
            "commit": "a" * 40,
            "runId": 100,
            "image": f"ghcr.io/example/{name}",
            "digest": "sha256:" + "b" * 64,
        }
    release = {"release": {"ready": True, "id": compute_release_id(components)},
               "components": components}
    Path("sample-release.json").write_text(json.dumps(release, indent=2) + "\n",
                                            encoding="utf-8")
    print("Created sample-release.json")


if __name__ == "__main__":
    main()
```

### File C — `tests/test_release_audit.py`

```python
import copy
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
from release_audit import compute_release_id, validate_bundle, validate_promotion


def sample():
    components = {}
    for name in ("frontend", "catalogue", "orders", "cart"):
        components[name] = {
            "component": name, "repository": f"example/{name}",
            "commit": "a" * 40, "runId": 1,
            "image": f"ghcr.io/example/{name}", "digest": "sha256:" + "b" * 64,
        }
    return {"release": {"ready": True, "id": compute_release_id(components)},
            "components": components}


def test_valid_bundle():
    assert validate_bundle(sample()) == sample()["release"]["id"]


def test_promotion_exact_bundle():
    s = sample()
    assert validate_promotion(s, copy.deepcopy(s)) == s["release"]["id"]


@pytest.mark.parametrize("field,bad", [("commit", "abc"), ("runId", True),
                                     ("digest", "latest"), ("repository", "evil/repo")])
def test_invalid_fact_rejected(field, bad):
    s = sample()
    s["components"]["orders"][field] = bad
    with pytest.raises(ValueError):
        validate_bundle(s)


def test_missing_component_rejected():
    s = sample()
    s["components"].pop("cart")
    with pytest.raises(ValueError, match="four components"):
        validate_bundle(s)


def test_different_promotion_rejected():
    upstream = sample()
    candidate = copy.deepcopy(upstream)
    candidate["components"]["orders"]["commit"] = "c" * 40
    candidate["release"]["id"] = compute_release_id(candidate["components"])
    with pytest.raises(ValueError, match="Different release ID"):
        validate_promotion(upstream, candidate)
```

### Run the capstone

```bash
# From the capstone/ directory, with your venv active:
python -m pip install pytest
python make_fixture.py
python release_audit.py sample-release.json --environment dev
# INFO: Validated dev release <16-hex-character-prefix>

python release_audit.py sample-release.json --environment qa --upstream sample-release.json
# INFO: Validated qa release <same-prefix>

python release_audit.py sample-release.json --environment production
# ERROR: Release rejected: QA/production require --upstream
# exit code 2

python -m pytest -q
# Expected: 8 passed
```

**Why it's useful:** This is real offline contract enforcement, not a fake “deployment”. The controller refuses malformed source facts, mismatched release IDs, missing components and modified promotions. Your tests prove both allowed and rejected paths.

### Upgrade tasks (advanced)

1. Put logic into `src/release_audit/` and package with `pyproject.toml`.
2. Add `--json` output for machine consumers while routing logs to stderr.
3. Add `GITHUB_OUTPUT` and `GITHUB_STEP_SUMMARY` writing, with sanitized values.
4. Add GitHub API read-only checks for successful CI on protected `main` and exact commit/run association.
5. Add a verifier interface and mock tests for Cosign results (don't invent cryptographic assurances).
6. Add a bounded deployment-health observer that rejects trusted failure immediately.
7. Add a policy requiring QA facts to match dev and production facts to match QA.
8. Add safe retry behavior for read-only transient API failures only.
9. Add CI tests with an unprivileged token. Do not enable PR merge/write permissions until independently reviewed.
10. Add fault-injection tests for stale events, rate limits, malformed API JSON and superseded releases.

### Productionization gap

The offline capstone **cannot** verify signatures, source CI, GitHub App identity, deployment health, review approval or rollback eligibility. Those are distinct trust boundaries requiring authentic remote evidence and correctly scoped permissions. Do **not** treat a locally valid JSON file as authorization to deploy.

---

<a id="part-v"></a>

# Part V — Exercises, solutions, interview preparation, reference

## 53. Fifteen short exercises with answer guidance

Try each without looking at the hint, then implement it in a separate file.

| # | Task | Concept | Check / hint |
|---|---|---|---|
| 1 | Extract `orders` from `"dev/orders"` | `.split()` | `text.split("/")[1]` |
| 2 | Return the first eight chars of a Git SHA | slicing | `sha[:8]` |
| 3 | Check environment is in dev/qa/prod | set membership | `env in {"dev", "qa", "prod"}` |
| 4 | Count nonempty entries in a list | comprehension | `sum(bool(x) for x in items)` |
| 5 | Find first failed status in a list | generator + `next` | `next((s for s in statuses if ...), None)` |
| 6 | Build a dict of pod-name to phase | dict comprehension | `{p["name"]: p["phase"] for p in pods}` |
| 7 | Require an integer run ID, not bool | type checking | `type(value) is int` |
| 8 | Validate a 40-hex Git commit | regex | `re.fullmatch(r"[a-f0-9]{40}", sha)` |
| 9 | Read a local JSON file | `pathlib`, `json` | `json.loads(Path(path).read_text())` |
| 10 | Run `git status` without a shell | subprocess | `subprocess.run(["git", "status"], check=True)` |
| 11 | Add `--environment` to a script | argparse | `parser.add_argument("--environment", required=True)` |
| 12 | Wait no more than 30 seconds for health | time | `time.monotonic() + 30` |
| 13 | Emit a GitHub step output | environment files | append `key=value\n` to `GITHUB_OUTPUT` |
| 14 | Write unit tests for invalid digest | pytest | `with pytest.raises(ValueError): ...` |
| 15 | Sort JSON keys before SHA-256 | deterministic encoding | `json.dumps(..., sort_keys=True, separators=(",", ":"))` |

**Solutions should be reasoning-first:** For each exercise, explain what kind of input should be rejected, which API can fail, and how the process signals failure to CI.

## 54. Python-to-Bash quick reference

| Task | Bash | Python |
|---|---|---|
| Print | `echo "$name"` | `print(name)` |
| Read env var | `$HOME` | `os.environ["HOME"]` |
| Default env | `${VAR:-dev}` | `os.getenv("VAR", "dev")` |
| String equality | `[[ "$a" == "$b" ]]` | `a == b` |
| If | `if ...; then ... fi` | `if condition: ...` |
| Loop | `for x in ...; do ... done` | `for x in items:` |
| Function | `fn() { ...; }` | `def fn(...):` |
| Function result | stdout / exit status | `return value` / exception |
| Run command | `git rev-parse HEAD` | `subprocess.run(["git", "rev-parse", "HEAD"])` |
| Parse JSON | `jq` | `json.loads()` and indexing |
| List files | `find` / `ls` | `Path.glob()` / `rglob()` |
| Exit status | `exit 1` | `sys.exit(1)` |
| Error | stderr + exit | `raise ValueError(...)` |
| Current time | `date -u` | `datetime.now(timezone.utc)` |

**Decision rule:** If your Bash script starts handling multi-layer JSON, retries, several service APIs and complicated validation, migrating its logic to Python can improve readability and testability.

## 55. Important built-in functions and idioms

| Name / syntax | Purpose | DevOps example |
|---|---|---|
| `len(x)` | count | number of changed PR files |
| `type(x)` | exact class | exclude Boolean from run ID |
| `isinstance(x, dict)` | type family | validate parsed JSON object |
| `str(x)`, `int(x)` | conversion | parse env var |
| `bool(x)` | truthiness | *not* string-to-Boolean parser |
| `sum(...)` | total | count healthy pods |
| `any(...)` | any true | find a failure status |
| `all(...)` | all true | require all digests well-formed |
| `sorted(x)` | order | canonical reports |
| `enumerate(x)` | index + value | numbered audit results |
| `zip(a,b)` | pair iterables | align names with values; beware truncation |
| `set(x)` | unique items | allowed changed paths |
| `next(it, default)` | first item | newest matching status |
| `iter(x)` | iterator | stream through results |
| `range(n)` | integer sequence | bounded retries |
| `min`, `max` | limit values | backoff cap |
| `open(...)` | open a file | read manifest |
| `print(..., file=sys.stderr)` | stderr | CLI errors |
| `getattr(x, "field", default)` | object attribute lookup | API/error attributes |
| `hasattr(x, "field")` | attribute exists | compatibility checks |
| `object.method()` | invoke object method | `.items()`, `.strip()` |
| `f"{value}"` | format string | release PR title |
| `obj or fallback` | truthy fallback | default optional value, watch `0` |
| `value if cond else other` | conditional expression | timeout by environment |

### `and`, `or`, and short-circuit evaluation

```python
value = None
if value is not None and value > 0:
    print("positive")

name = "" or "default"
print(name)  # default
```

`and` and `or` do not always return Boolean: they can return an operand. `"" or "default"` returns the string `"default"`. Use this intentionally, particularly in GitHub Actions-style fallback logic.

### `None` versus empty structures

```python
x = None        # no value was supplied
x = []          # a list exists but is empty
x = {}          # a dict exists but is empty
```

These states may have different meanings for deployment contracts. Validate exact types and required fields instead of relying only on truthiness.

## 56. Debugging playbook

1. **Read the exception type** (`KeyError`, `TypeError`, `ValueError`, `CalledProcessError`, `TimeoutExpired`).
2. **Find the traceback's first relevant file/line** inside your own code.
3. **Inspect data shape safely**: `print(type(data), list(data.keys()))` rather than printing tokens or entire API responses.
4. **Reproduce the smallest failing input** as a unit test.
5. **Use the debugger**: `breakpoint()` in trusted local scripts; `python -m pdb script.py` when needed.
6. **Turn on structured debug logs**, with redaction.
7. **Check environmental differences**: interpreter/venv, working directory, PATH, token permissions, branch, Git repository, proxy and hostname.
8. **Check clock/deadline assumptions** and whether a retry loop can hang.
9. **Check encoding and line endings** when parsing files from Linux/Windows.
10. **Distinguish validation failure from transport failure**. Both should fail closed, but require different remediation.

### Common errors

```text
ModuleNotFoundError: No module named 'yaml'
→ Activate the correct venv and run: python -m pip install PyYAML

KeyError: 'commit'
→ Dictionary missing a required key; validate the response schema.

TypeError: string indices must be integers
→ You indexed a string as though it were a dict; check json.loads() and nesting.

subprocess.CalledProcessError
→ A command returned nonzero; inspect sanitized stderr/returncode.

subprocess.TimeoutExpired
→ External command exceeded its timeout; distinguish slow from stuck.

requests.exceptions.HTTPError: 403
→ Permission, rate-limit or policy issue; inspect status and headers safely.

IndentationError
→ Use consistent four-space blocks; avoid mixing tabs and spaces.
```

## 57. Four-week focused learning plan

| Week | Learn | Build | Completion test |
|---|---|---|---|
| 1 | Sections 1–12 | Labs 01–03 | Explain lists, dictionaries, functions, `return`, loops, exceptions without notes |
| 2 | Sections 13–23 | Labs 04–08 | Build a JSON/YAML validator CLI with no external service |
| 3 | Sections 24–30 | Labs 09–13 (optional services) | Read a GitHub API result; understand Docker/Kubernetes/Boto3 clients and least privilege |
| 4 | Sections 31–38 | Lab 14 + tests | Deliver a tested release audit CLI with negative tests, logs, help text and nonzero failure status |

For short study sessions, follow **30 minutes reading + 45 minutes coding + 15 minutes debugging/notes**. Your target is not memorization: you should be able to look up unfamiliar functions and trace a safe execution path.

## 58. Final assessment: can you read a release controller?

After this book you should be able to answer these questions about a Python release-management script:

- Which CLI operation is executed from `main()` for `stage qa`?
- What do `run(*args)` and `subprocess.check_output(...)` do?
- Why is `json.loads()` used after `gh api`?
- Why is `type(run_id) is int` stricter than `isinstance(run_id, int)`?
- What does `{k: fact[k] for k in keys}` construct?
- What does `next((x for x in items if condition), None)` return?
- Why compare `set(candidate)` to an exact expected key set?
- Why generate a SHA-256 release ID from canonical JSON?
- Why check a workflow's head SHA, branch and repository after seeing `conclusion=success`?
- Why does verifying an image digest's syntax not verify signature or provenance?
- Why should read-only transient API requests be retried differently from release writes?
- What does `time.monotonic()` prevent in polling loops?
- What do `GITHUB_OUTPUT` and `GITHUB_STEP_SUMMARY` do?
- What prevents an old failure status from triggering an inappropriate rollback?
- What is the difference between a release being in Git and it being healthy in Kubernetes?
- Why are production PR creation, approval, merge and observation separate authorities?

If you can explain these in plain language and write a few tests, you're ready to work through the real script **function by function**, not merely copy its code.

## 59. Official documentation and further reading

Use official docs to confirm behavior in your installed versions:

- [Python tutorial](https://docs.python.org/3/tutorial/) — beginner language guide.
- [Python standard library](https://docs.python.org/3/library/) — `pathlib`, `json`, `argparse`, `subprocess`, `logging`, `time`, etc.
- [Virtual environments](https://docs.python.org/3/library/venv.html) — isolated Python environments.
- [Packaging Python projects](https://packaging.python.org/en/latest/tutorials/packaging-projects/) — `pyproject.toml`, wheels, publishing.
- [pytest](https://docs.pytest.org/en/stable/) — testing and fixtures.
- [Ruff](https://docs.astral.sh/ruff/) — linting and formatting.
- [Mypy](https://mypy.readthedocs.io/en/stable/) — static type checks.
- [Requests](https://requests.readthedocs.io/en/latest/) — HTTP API client.
- [PyYAML](https://pyyaml.org/wiki/PyYAMLDocumentation) — YAML parsing.
- [GitHub workflow commands](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands) — outputs and summaries.
- [GitHub REST API](https://docs.github.com/en/rest) — workflows, PRs, releases and status APIs.
- [Boto3 SDK](https://docs.aws.amazon.com/boto3/latest/) and [paginators](https://docs.aws.amazon.com/boto3/latest/guide/paginators.html) — AWS SDK.
- [Kubernetes Python client](https://github.com/kubernetes-client/python) — cluster API access.
- [Docker SDK for Python](https://docker-py.readthedocs.io/en/stable/) — Docker API usage.
- [Sigstore Cosign](https://docs.sigstore.dev/cosign/) — image signatures and attestations.

### Suggested additional advanced topics

Once the capstone is comfortable: Python `protocol` interfaces and dependency injection; JSON Schema/Pydantic validation; async HTTP clients; bounded queue workers; idempotency keys; circuit breakers; file locks; transactions; secure temporary files; OpenTelemetry; package supply-chain controls; reproducible builds; GitHub App installation tokens; Kubernetes watchers/informers; AWS STS assume-role; event-driven orchestration and disaster-recovery drills.

---

## Quick printable checklist

- [ ] I can run a `.py` file inside a venv.
- [ ] I understand `str`, `int`, `bool`, `None`, `list`, `tuple`, `set`, and `dict`.
- [ ] I can explain `if`, loops, functions, `return`, `*args`, `**kwargs` and comprehensions.
- [ ] I can parse JSON/YAML safely and validate exact fields.
- [ ] I can read/write files with `pathlib` and use context managers.
- [ ] I can run Git commands with `subprocess` argument lists and timeouts.
- [ ] I can build a CLI with `argparse` and sensible exit codes.
- [ ] I can perform authenticated read-only API requests without logging secrets.
- [ ] I know how to inspect Docker, Kubernetes and AWS safely.
- [ ] I can use `pytest`, mocks, linting and type checks.
- [ ] I can explain hashes vs signatures, CI status vs deployment health.
- [ ] I can trace a GitOps promotion and rollback state machine.
- [ ] I have run the capstone's **positive and negative tests**.

**Finish line:** Build small, safe Python tools before attempting a privileged release orchestrator. A few well-tested functions with narrow permissions are more valuable than a large unreviewed automation script.

<a id="appendix"></a>

# Appendix — Infrastructure-as-Code and multi-repository CI/CD

## 60. Python with Terraform, Helm and Ansible

As a DevOps engineer, Python often acts as a **controller or validation layer** around infrastructure tooling. You do not need to replace declarative IaC with handwritten API calls.

### Inspect a Terraform plan without applying it

A Terraform plan can be converted to JSON for machine-readable inspection:

```bash
# In a disposable sandbox with initialized Terraform configuration:
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
```

**Security:** Plan files and JSON output can contain sensitive values. Never publish them as unprotected CI artifacts or log them in full. For privileged CI, use an approved policy engine and handle state/plan secrets correctly.

Read only the resource change summary:

```python
import json
from pathlib import Path

plan = json.loads(Path("tfplan.json").read_text(encoding="utf-8"))
for change in plan.get("resource_changes", []):
    address = change.get("address", "unknown")
    actions = change.get("change", {}).get("actions", [])
    print(address, ",".join(actions))
```

**Exercise:** Fail the job if any resource action contains `delete`; explain why this is a limited guard rather than a complete approval policy.

### Check Helm chart rendering before deployment

Use `helm template` to render locally without deploying:

```python
import subprocess
import yaml

result = subprocess.run(
    ["helm", "template", "orders", "./chart", "--namespace", "dev"],
    check=True, capture_output=True, text=True, timeout=45,
)
resources = [doc for doc in yaml.safe_load_all(result.stdout) if doc is not None]
for resource in resources:
    print(resource.get("kind"), resource.get("metadata", {}).get("name"))
```

**Prerequisite:** `./chart` must be a local, reviewed Helm chart with its dependencies already available. `helm template` executes template rendering and can involve plugins or chart functions; don't give untrusted chart code privileged runner access. Before real deployment, also check policies and cluster-specific validation.

### Consume Ansible JSON output

Python can validate inventory files or analyze Ansible's machine-readable outputs. For a simple inventory in JSON format:

```python
import json
from pathlib import Path

inventory = json.loads(Path("inventory.json").read_text(encoding="utf-8"))
if not isinstance(inventory, dict):
    raise ValueError("Inventory must be a JSON object")
print("Top-level groups:", list(inventory))
```

Automation rule: **Prefer an Ansible module or Terraform resource** to a handwritten Python script for tasks those tools already model reliably. Use Python where it adds validation, orchestration, reports, custom integrations or tests.

## 61. Multi-repository GitHub Actions and Python verification gates

A large DevOps platform may have application repositories, a central CI repository, a GitOps repository, workflow callers and reusable workflows. Python is often the shared validation engine.

### Caller and reusable workflow concept

Application repository caller (illustrative):

```yaml
name: Application checks
on:
  pull_request:

permissions:
  contents: read

jobs:
  verify:
    uses: example-org/platform-ci/.github/workflows/reusable-verify.yml@main
    with:
      environment: dev
```

Reusable workflow in `platform-ci` (illustrative):

```yaml
name: Reusable verification
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

permissions:
  contents: read

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: python scripts/verify.py --environment "${{ inputs.environment }}"
```

**Important:** These YAML snippets illustrate a structure, not a complete secure production pipeline. The caller must make the intended trusted script available; `actions/checkout` normally checks out the caller repository, **not automatically the repository holding the reusable workflow definition**. If the Python verifier lives in a separate repository, fetch its exact reviewed commit securely (or package it as a versioned artifact) rather than executing an arbitrary branch. In production, pin the reusable workflow and referenced actions to reviewed immutable SHAs and validate untrusted string inputs before using them in shell commands. Passing `${{ inputs.environment }}` directly into shell source may permit injection when the input is untrusted; a safer pattern passes it through `env:` and reads the environment variable inside Python.

Safer input pattern in a workflow step:

```yaml
- name: Verify
  env:
    TARGET_ENVIRONMENT: ${{ inputs.environment }}
  run: python scripts/verify.py --environment "$TARGET_ENVIRONMENT"
```

The shell sees the variable as data, and Python's `argparse` choice validation can reject values outside dev/qa/production.

### Why separate source, release and deployment evidence?

A robust gate asks three distinct questions:

1. **Source trust:** Did the expected protected-main workflow build the expected commit?
2. **Artifact trust:** Is the exact digest supported by appropriately verified provenance/signatures and a valid release contract?
3. **Deployment trust:** Did the target configuration revision actually deploy and pass independent health/smoke checks?

These checks belong to different systems and identities. A single `success` string from an untrusted payload must never replace them.

### Recommended CI/CD Python module boundaries

```text
release_control/
  cli.py                # argparse, output formatting, exit status
  config.py             # allowed environments, policy loading
  schema.py             # exact input validation
  github_read.py        # read-only GitHub client with timeouts/pagination
  provenance.py         # attestation verification adapter
  bundle.py             # canonical image bundle and release IDs
  gitops.py             # read-only Git history/configuration analysis
  promotion.py          # pure promotion/rollback decisions
  writer.py             # carefully scoped PR creation (separate permissions)
  observer.py           # polling trusted health evidence
  outputs.py            # sanitized GitHub workflow outputs/summaries
  tests/                # pure-unit, mocked API, negative-path tests
```

Start by testing **pure functions** (`schema.py`, `bundle.py`, `promotion.py`) without tokens or network. Mock boundary clients, then exercise real API integrations in a sandbox. Isolate PR writing behind a separate audited identity; only trusted workflow events should invoke it.

---

**End of handbook.** Your best next exercise is to explain the script's `run()`, `api()`, `require()`, `facts_valid()` and `bundle()` functions in plain language, then reproduce their simplest behaviors in a local lab. That is a realistic bridge from Python basics to production GitOps control code.

# Python Production-Grade Cheatsheet

A comprehensive, production-ready reference guide for Python developers, covering environment setup, dependency management, packaging, typing, testing, CI/CD, tooling, concurrency, profiling, and core language features.

---

## Table of Contents

1. [Prerequisites & Environment](#prerequisites--environment)
2. [Installation & Dependency Management](#installation--dependency-management)
3. [Packaging & Distribution](#packaging--distribution)
4. [Type Hints & Static Checking](#type-hints--static-checking)
5. [Testing & CI](#testing--ci)
6. [Tooling & Formatting](#tooling--formatting)
7. [Concurrency & Parallelism](#concurrency--parallelism)
8. [Profiling & Monitoring](#profiling--monitoring)
9. [Core Syntax & Features](#core-syntax--features)

   * [Basics](#basics)
   * [Data Structures](#data-structures)
   * [Control Flow](#control-flow)
   * [Functions & Modules](#functions--modules)
   * [OOP & Advanced Features](#oop--advanced-features)
10. [Standard Library Highlights](#standard-library-highlights)
11. [Quick-Reference Summary](#quick-reference-summary)

---

## Prerequisites & Environment

**Supported Python Versions:** 3.9+

1. **Create a virtual environment**:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate   # Unix/macOS
   .\.venv\\Scripts\\activate # Windows PowerShell
   ```

2. **Using Poetry**:

   ```bash
   curl -sSL https://install.python-poetry.org | python3 -
   poetry init --name my_project --python ">=3.9"
   poetry shell
   poetry add <packages>
   ```

3. **Using Pipenv**:

   ```bash
   pip install pipenv
   pipenv --python 3.9
   pipenv install <packages>
   pipenv shell
   ```

---

## Installation & Dependency Management

Consolidate all external libraries with pinned minimum versions.

**Using pip:**

```bash
pip install \
    requests>=2.28 \
    aiohttp>=3.8 \
    pytest>=7.0 \
    mypy>=1.0 \
    black>=23.1 \
    flake8>=5.0 \
    isort>=5.0 \
    prometheus-client>=0.15 \
    sentry-sdk>=1.0
```

**Using Poetry (`pyproject.toml` snippet):**

```toml
[tool.poetry.dependencies]
python = ">=3.9,<4.0"
requests = ">=2.28"
aiohttp = ">=3.8"

[tool.poetry.dev-dependencies]
pytest = ">=7.0"
mypy = ">=1.0"
black = ">=23.1"
flake8 = ">=5.0"
isort = ">=5.0"
prometheus-client = ">=0.15"
sentry-sdk = ">=1.0"
```

---

## Packaging & Distribution

**Minimal `pyproject.toml`:**

```toml
[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"

[tool.poetry]
name = "my_package"
version = "0.1.0"
description = "Short description"
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = ">=3.9,<4.0"
```

**Build wheel & source:**

```bash
python -m build    # Generates wheel and sdist
```

**Install Locally:**

```bash
pip install .
```

**Publish to PyPI (Poetry):**

```bash
poetry publish --build
```

Alternatively, `setup.py` example:

```python
from setuptools import setup, find_packages

setup(
    name="my_package",
    version="0.1.0",
    packages=find_packages(),
    install_requires=[
        "requests>=2.28",
        "aiohttp>=3.8",
    ],
)
```

---

## Type Hints & Static Checking

Leverage static typing for clarity and safety.

```python
from typing import List, Optional, Dict

def fetch_data(ids: List[int]) -> Dict[int, str]:
    results: Dict[int, str] = {}
    for _id in ids:
        results[_id] = _fetch(_id)
    return results

def parse_user(input_str: str) -> Optional[int]:
    try:
        return int(input_str)
    except ValueError:
        return None
```

**Run MyPy:**

```bash
mypy src/ tests/ --ignore-missing-imports
```

---

## Testing & CI

**Project Layout:**

```
my_project/
├── src/
│   └── my_package/
│       └── __init__.py
├── tests/
│   └── test_example.py
├── pyproject.toml
└── pytest.ini
```

**Sample Test (`tests/test_example.py`):**

```python
from my_package import add

def test_add():
    assert add(1, 2) == 3
```

**Run Tests & Coverage:**

```bash
coverage run -m pytest
coverage report -m
```

**GitHub Actions Workflow (`.github/workflows/ci.yml`):**

```yaml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      - name: Install dependencies
        run: |
          python -m venv .venv
          source .venv/bin/activate
          pip install -r requirements.txt  # or: poetry install
      - name: Run formatting & linting
        run: |
          black --check .
          isort --check-only .
          flake8
      - name: Run type checks
        run: mypy src/ --ignore-missing-imports
      - name: Run tests
        run: |
          coverage run -m pytest
          coverage report -m
```

---

## Tooling & Formatting

Configure formatting and linting:

**Black & isort (`pyproject.toml`):**

```toml
[tool.black]
line-length = 88

[tool.isort]
profile = "black"
```

**Flake8 (`.flake8`):**

```ini
[flake8]
max-line-length = 88
extend-ignore = E203
```

---

## Concurrency & Parallelism

**Asyncio with aiohttp (fixed import):**

```python
import asyncio
from typing import List
import aiohttp

async def fetch(url: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()

async def main(urls: List[str]):
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print(results)

if __name__ == "__main__":
    asyncio.run(main(["https://example.com"]))
```

**Thread vs Process Pool:**

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_bound(n: int) -> int:
    return sum(i*i for i in range(n))

def io_bound(path: str) -> str:
    with open(path) as f:
        return f.read()

# Thread pool for IO-bound tasks
together_paths = ["file1.txt", "file2.txt"]
with ThreadPoolExecutor(max_workers=5) as pool:
    data = list(pool.map(io_bound, together_paths))

# Process pool for CPU-bound tasks
with ProcessPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(cpu_bound, [10**6]*4))
```

---

## Profiling & Monitoring

**cProfile:**

```bash
python -m cProfile -s time my_module.py
```

**timeit:**

```python
import timeit
print(timeit.timeit("[x*x for x in range(1000)]", number=1000))
```

**Prometheus Client:**

```python
from prometheus_client import Counter, start_http_server

REQUESTS = Counter("http_requests_total", "Total HTTP requests counted")

if __name__ == '__main__':
    start_http_server(8000)
    REQUESTS.inc()
```

**Sentry SDK (load DSN from env):**

```python
import os
import sentry_sdk

sentry_dsn = os.getenv("SENTRY_DSN")
if sentry_dsn:
    sentry_sdk.init(
        dsn=sentry_dsn,
        traces_sample_rate=1.0
    )
```

---

## Core Syntax & Features

### Basics

```python
# Variables, literals, f-strings, unpacking
name: str = "Alice"
age: int = 30
scores = [100, 95, 80]
first, *rest = scores
print(f"{name} is {age} years old. First score: {first}")
```

### Data Structures

```python
# List, dict, set comprehensions & methods
nums = [1, 2, 3, 4]
squares = [x*x for x in nums]
even = {x for x in nums if x%2==0}
user_map = {u.id: u.name for u in users_list}
```

### Control Flow

```python
# If/elif/else, loops, comprehensions
for x in range(3):
    if x == 0:
        print("Zero")
    elif x == 1:
        print("One")
    else:
        print("Many")

# While loop
i = 0
while i < 3:
    print(i)
    i += 1
```

### Functions & Modules

```python
# Default args, *args/**kwargs, imports
from math import sqrt

def greet(name: str, greeting: str = "Hello") -> None:
    print(f"{greeting}, {name}!")

def combine(*args: str, **kwargs: str) -> str:
    return ",".join(args) + ";" + ";".join(f"{k}={v}" for k,v in kwargs.items())
```

### OOP & Advanced Features

```python
# Decorators
def timer(func):
    def wrapper(*args, **kwargs):
        import time
        start = time.time()
        result = func(*args, **kwargs)
        print(f"Elapsed: {time.time()-start}")
        return result
    return wrapper

@timer
def compute(n: int):
    return sum(range(n))

# Context manager
from contextlib import contextmanager

@contextmanager
def open_file(path):
    f = open(path)
    try:
        yield f
    finally:
        f.close()

# Generator
def counter(limit: int):
    for i in range(limit):
        yield i
```

---

## Standard Library Highlights

*As before*

---

## Quick-Reference Summary

| Category            | Command / Idiom                   |
| ------------------- | --------------------------------- |
| Virtualenv          | `python3 -m venv .venv`           |
| Install deps        | `pip install -r requirements.txt` |
| Run script          | `python -m my_module`             |
| Format code         | `black .`                         |
| Import sorting      | `isort .`                         |
| Lint                | `flake8`                          |
| Static type check   | `mypy src/`                       |
| Test                | `pytest`                          |
| Coverage            | `coverage run -m pytest`          |
| Build (wheel/sdist) | `python -m build`                 |
| Install package     | `pip install .`                   |
| Publish             | `poetry publish --build`          |
| Async entry         | `asyncio.run(main())`             |
| ThreadPool          | `ThreadPoolExecutor`              |
| Profiling           | `python -m cProfile`              |
| Monitoring counter  | `prometheus_client.Counter`       |

---

*Generated on May 25, 2025*

---
name: python-pro
description: Advanced Python patterns: async/await, metaclasses, decorators, type hints, packaging, virtual environments, and performance optimization.
---

# Python Pro

Advanced Python programming patterns and best practices for senior developers. Covers the modern Python ecosystem (3.11+) including the type system, async concurrency, metaprogramming, packaging, performance tuning, and testing strategies.

## Table of Contents

- [Advanced Type System](#advanced-type-system)
- [Async Programming](#async-programming)
- [Decorator Patterns](#decorator-patterns)
- [Metaclasses and Descriptors](#metaclasses-and-descriptors)
- [Packaging and Distribution](#packaging-and-distribution)
- [Performance Optimization](#performance-optimization)
- [Testing Patterns](#testing-patterns)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## Advanced Type System

Modern Python typing goes far beyond `int` and `str`. Use the full type system to catch bugs at static analysis time.

### Protocol (Structural Subtyping)

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Renderable(Protocol):
    def render(self) -> str: ...

class HtmlWidget:
    def render(self) -> str:
        return "<div>widget</div>"

def display(item: Renderable) -> None:
    print(item.render())

# HtmlWidget satisfies Renderable without inheriting from it
display(HtmlWidget())  # OK
isinstance(HtmlWidget(), Renderable)  # True at runtime
```

### TypeVar, ParamSpec, and TypeVarTuple

```python
from typing import TypeVar, ParamSpec, TypeVarTuple, Callable, Unpack

T = TypeVar("T")
P = ParamSpec("P")
Ts = TypeVarTuple("Ts")

# ParamSpec preserves the full signature of the wrapped function
def with_logging(fn: Callable[P, T]) -> Callable[P, T]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
        print(f"Calling {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper

# TypeVarTuple for variadic generics
class Pipeline[*Ts]:
    """Typed pipeline where each stage's output feeds the next."""
    def __init__(self, *stages: Unpack[Ts]) -> None:
        self.stages = stages
```

### overload and TypeGuard

```python
from typing import overload, TypeGuard

@overload
def parse(raw: str) -> dict[str, str]: ...
@overload
def parse(raw: bytes) -> dict[str, bytes]: ...

def parse(raw: str | bytes) -> dict[str, str] | dict[str, bytes]:
    if isinstance(raw, str):
        return {"value": raw}
    return {b"value": raw}  # type checker knows the return type per call site

# TypeGuard narrows types in conditional branches
def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

def process(items: list[object]) -> None:
    if is_string_list(items):
        # type checker now knows items is list[str]
        print(items[0].upper())
```

### dataclass_transform and Generic Classes

```python
from typing import dataclass_transform, Generic, TypeVar

T = TypeVar("T")

@dataclass_transform()
class ModelBase:
    """ORM-style base. Type checkers treat subclasses as dataclasses."""
    def __init_subclass__(cls, **kwargs):
        # Auto-generate __init__ from annotations
        import dataclasses
        dataclasses.dataclass(cls)

class User(ModelBase):
    name: str
    age: int

# Generic container with bound
class Repository(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def add(self, item: T) -> None:
        self._items.append(item)

    def get(self, index: int) -> T:
        return self._items[index]

repo: Repository[User] = Repository()
repo.add(User(name="Alice", age=30))
```

**Anti-patterns to avoid:**
- Using `Any` to silence type errors -- fix the type instead.
- Writing `isinstance` checks where a `Protocol` or `TypeGuard` would eliminate them.
- Ignoring `# type: ignore` comments -- each one should have a justification suffix: `# type: ignore[assignment]`.

---

## Async Programming

### Core asyncio Patterns

```python
import asyncio
from contextlib import asynccontextmanager

async def fetch(url: str) -> bytes:
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        resp.raise_for_status()
        return resp.content

# Task groups (Python 3.11+) -- structured concurrency
async def fetch_all(urls: list[str]) -> list[bytes]:
    results: list[bytes] = []
    async with asyncio.TaskGroup() as tg:
        for url in urls:
            tg.create_task(_fetch_and_collect(url, results))
    return results

async def _fetch_and_collect(url: str, out: list[bytes]) -> None:
    out.append(await fetch(url))
```

### Async Generators and Context Managers

```python
from typing import AsyncIterator

async def stream_lines(path: str) -> AsyncIterator[str]:
    """Async generator -- yields lines without loading the full file."""
    async with aiofiles.open(path, "r") as f:
        async for line in f:
            yield line.strip()

@asynccontextmanager
async def db_transaction(pool):
    conn = await pool.acquire()
    tx = conn.transaction()
    await tx.start()
    try:
        yield conn
        await tx.commit()
    except Exception:
        await tx.rollback()
        raise
    finally:
        await pool.release(conn)
```

### Semaphore-Based Rate Limiting

```python
import asyncio
import httpx

async def rate_limited_fetch(
    urls: list[str], max_concurrent: int = 10
) -> list[httpx.Response]:
    semaphore = asyncio.Semaphore(max_concurrent)
    async with httpx.AsyncClient(timeout=30.0) as client:
        async def _get(url: str) -> httpx.Response:
            async with semaphore:
                return await client.get(url)
        return await asyncio.gather(*[_get(u) for u in urls])
```

### httpx Patterns (Preferred Over aiohttp)

```python
import httpx

async def resilient_request(url: str, retries: int = 3) -> httpx.Response:
    transport = httpx.AsyncHTTPTransport(retries=retries)
    async with httpx.AsyncClient(transport=transport) as client:
        resp = await client.get(url)
        resp.raise_for_status()
        return resp
```

**Anti-patterns to avoid:**
- Calling `asyncio.run()` inside an already-running loop -- use `await` instead.
- Using bare `asyncio.gather()` without error handling -- one exception cancels nothing by default. Prefer `TaskGroup` which cancels siblings on failure.
- Blocking the event loop with synchronous I/O -- offload to `asyncio.to_thread()`.

---

## Decorator Patterns

### Decorator Factory (With and Without Arguments)

```python
import functools
from typing import Callable, TypeVar, ParamSpec, overload

P = ParamSpec("P")
T = TypeVar("T")

# Decorator that works both with and without arguments:
# @retry          (no parens)
# @retry(times=5) (with parens)
@overload
def retry(fn: Callable[P, T]) -> Callable[P, T]: ...
@overload
def retry(*, times: int = 3, delay: float = 1.0) -> Callable[[Callable[P, T]], Callable[P, T]]: ...

def retry(fn=None, *, times=3, delay=1.0):
    def decorator(fn: Callable[P, T]) -> Callable[P, T]:
        @functools.wraps(fn)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
            last_exc: Exception | None = None
            for attempt in range(times):
                try:
                    return fn(*args, **kwargs)
                except Exception as exc:
                    last_exc = exc
                    time.sleep(delay * (2 ** attempt))
            raise last_exc  # type: ignore[misc]
        return wrapper
    if fn is not None:
        return decorator(fn)
    return decorator
```

### Class Decorators

```python
import functools
import time

def singleton(cls):
    """Class decorator -- ensures only one instance exists."""
    instances: dict = {}
    @functools.wraps(cls, updated=[])
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Config:
    def __init__(self):
        self.values = {}
```

### Stacked Decorators and functools.wraps

```python
def timer(fn):
    @functools.wraps(fn)  # preserves __name__, __doc__, __module__
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = fn(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{fn.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

def validate_args(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        if any(a is None for a in args):
            raise ValueError("None arguments not allowed")
        return fn(*args, **kwargs)
    return wrapper

# Execution order: validate_args runs first, then timer
@timer
@validate_args
def compute(x: int, y: int) -> int:
    return x ** y
```

### contextmanager as Decorator

```python
from contextlib import contextmanager

@contextmanager
def temp_env(**env_vars):
    """Use as context manager or decorator."""
    import os
    old = {k: os.environ.get(k) for k in env_vars}
    os.environ.update(env_vars)
    try:
        yield
    finally:
        for k, v in old.items():
            if v is None:
                os.environ.pop(k, None)
            else:
                os.environ[k] = v

# As context manager
with temp_env(DEBUG="1"):
    run_app()

# As decorator
@temp_env(DEBUG="1")
def test_debug_mode():
    assert os.environ["DEBUG"] == "1"
```

**Anti-patterns to avoid:**
- Forgetting `@functools.wraps` -- breaks introspection, docstrings, and debugging.
- Mutable default arguments in decorator factories (`def retry(times, backoff=[])`).
- Decorators that swallow exceptions silently.

---

## Metaclasses and Descriptors

### When to Use Metaclasses vs `__init_subclass__`

Prefer `__init_subclass__` for simple registration and validation. Reserve metaclasses for when you need to control class creation itself (e.g., modifying `__dict__` before the class object exists).

```python
# __init_subclass__ -- lightweight, preferred approach
class Plugin:
    _registry: dict[str, type] = {}

    def __init_subclass__(cls, *, name: str = "", **kwargs):
        super().__init_subclass__(**kwargs)
        key = name or cls.__name__.lower()
        Plugin._registry[key] = cls

class AuthPlugin(Plugin, name="auth"):
    pass

class CachePlugin(Plugin, name="cache"):
    pass

# Plugin._registry == {"auth": AuthPlugin, "cache": CachePlugin}
```

### Metaclass Example

```python
class ValidatedMeta(type):
    """Metaclass that enforces required class attributes."""
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:  # skip the base class itself
            for attr in getattr(cls, "_required_attrs", []):
                if attr not in namespace:
                    raise TypeError(f"{name} must define '{attr}'")
        return cls

class Service(metaclass=ValidatedMeta):
    _required_attrs = ["name", "version"]

class MyService(Service):
    name = "my-service"
    version = "1.0.0"

# class BadService(Service):  # TypeError: BadService must define 'name'
#     version = "0.1"
```

### Descriptor Protocol

```python
class Validated:
    """Descriptor that validates on assignment."""
    def __init__(self, min_val: float, max_val: float):
        self.min_val = min_val
        self.max_val = max_val

    def __set_name__(self, owner, name):
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not self.min_val <= value <= self.max_val:
            raise ValueError(f"Expected {self.min_val}..{self.max_val}, got {value}")
        setattr(obj, self.private_name, value)

class Sensor:
    temperature = Validated(-40.0, 125.0)
    humidity = Validated(0.0, 100.0)

    def __init__(self, temp: float, hum: float):
        self.temperature = temp
        self.humidity = hum
```

### `__slots__` Optimization

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

# ~40% less memory per instance, faster attribute access
# Cannot add arbitrary attributes: Point().z = 1 raises AttributeError
```

### ABC Patterns

```python
from abc import ABC, abstractmethod

class Serializer(ABC):
    @abstractmethod
    def serialize(self, data: dict) -> bytes: ...

    @abstractmethod
    def deserialize(self, raw: bytes) -> dict: ...

    def round_trip(self, data: dict) -> dict:
        """Concrete method using abstract ones."""
        return self.deserialize(self.serialize(data))
```

**Anti-patterns to avoid:**
- Using a metaclass where `__init_subclass__` suffices -- metaclasses complicate MRO and debugging.
- Inheriting from multiple classes with different metaclasses -- leads to `TypeError`.
- Descriptors without `__set_name__` -- forces manual name tracking.

---

## Packaging and Distribution

### pyproject.toml (The Only Config File You Need)

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-library"
version = "1.2.0"
description = "A useful library"
requires-python = ">=3.11"
license = "MIT"
authors = [{ name = "Dev", email = "dev@example.com" }]
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "ruff", "mypy"]

[project.scripts]
my-cli = "my_library.cli:main"

[tool.ruff]
target-version = "py311"
line-length = 99

[tool.mypy]
strict = true
```

### src Layout (Recommended)

```
my-library/
  pyproject.toml
  src/
    my_library/
      __init__.py
      cli.py
      core.py
  tests/
    conftest.py
    test_core.py
```

The `src/` layout prevents accidental imports from the working directory during testing.

### Build Tool Comparison

| Feature | setuptools | Hatch | Poetry | uv |
|---|---|---|---|---|
| Config format | pyproject.toml | pyproject.toml | pyproject.toml | pyproject.toml |
| Lock file | No | No | poetry.lock | uv.lock |
| Venv management | No | Yes | Yes | Yes |
| Speed | Moderate | Fast | Moderate | Fastest |
| Monorepo support | Manual | Workspaces | No | Workspaces |
| Recommended for | Legacy projects | Libraries | Apps | New projects |

### Dependency Management with uv

```bash
# Create project with uv
uv init my-project
cd my-project

# Add dependencies
uv add httpx pydantic
uv add --dev pytest ruff mypy

# Sync environment from lock file
uv sync

# Run commands in the managed environment
uv run python -m pytest
uv run my-cli --help

# Build and publish
uv build
uv publish --token $PYPI_TOKEN
```

### Publishing to PyPI

```bash
# Build sdist and wheel
uv build  # or: python -m build

# Upload (use a PyPI API token, never your password)
uv publish --token $PYPI_TOKEN
# or: twine upload dist/*

# Test on TestPyPI first
uv publish --index-url https://test.pypi.org/simple/
```

**Anti-patterns to avoid:**
- Using `setup.py` for new projects -- `pyproject.toml` is the standard.
- Pinning exact versions in library dependencies (`httpx==0.27.0`) -- use ranges (`httpx>=0.27`).
- Committing `.venv/` or `__pycache__/` to version control.

---

## Performance Optimization

### Profiling with cProfile and py-spy

```bash
# cProfile -- built-in, function-level granularity
python -m cProfile -s cumulative my_script.py

# py-spy -- sampling profiler, attach to running process
py-spy top --pid 12345
py-spy record -o profile.svg -- python my_script.py
```

```python
# Targeted profiling in code
import cProfile
import pstats

def profile_block():
    profiler = cProfile.Profile()
    profiler.enable()

    # ... code to profile ...
    result = expensive_computation()

    profiler.disable()
    stats = pstats.Stats(profiler).sort_stats("cumulative")
    stats.print_stats(20)  # top 20 functions
```

### Memory Profiling

```bash
# tracemalloc -- built-in
python -c "
import tracemalloc
tracemalloc.start()

data = [dict(x=i) for i in range(1_000_000)]

snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:5]:
    print(stat)
"

# memory_profiler -- line-by-line memory usage
pip install memory-profiler
python -m memory_profiler my_script.py
```

### Generator Patterns for Memory Efficiency

```python
# BAD: loads everything into memory
def get_all_users(db) -> list[User]:
    return [User(**row) for row in db.execute("SELECT * FROM users")]

# GOOD: yields one at a time
def iter_users(db) -> Iterator[User]:
    cursor = db.execute("SELECT * FROM users")
    while batch := cursor.fetchmany(1000):
        yield from (User(**row) for row in batch)
```

### C Extensions with ctypes/cffi

```python
import ctypes

# Load a shared library
lib = ctypes.CDLL("./libfast.so")
lib.compute.argtypes = [ctypes.c_double, ctypes.c_int]
lib.compute.restype = ctypes.c_double

result = lib.compute(3.14, 100)

# cffi -- cleaner API
from cffi import FFI
ffi = FFI()
ffi.cdef("double compute(double x, int n);")
lib = ffi.dlopen("./libfast.so")
result = lib.compute(3.14, 100)
```

### multiprocessing vs threading

```python
import concurrent.futures

# CPU-bound: use ProcessPoolExecutor (bypasses the GIL)
def cpu_task(n: int) -> int:
    return sum(i * i for i in range(n))

with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(cpu_task, [10**7] * 8))

# I/O-bound: use ThreadPoolExecutor
def io_task(url: str) -> bytes:
    import urllib.request
    return urllib.request.urlopen(url).read()

with concurrent.futures.ThreadPoolExecutor(max_workers=20) as pool:
    futures = [pool.submit(io_task, url) for url in urls]
    results = [f.result() for f in concurrent.futures.as_completed(futures)]
```

**Anti-patterns to avoid:**
- Optimizing before profiling -- measure first, then target the hotspot.
- Using `multiprocessing` for I/O-bound work -- the process overhead is wasteful.
- Storing large datasets as `list[dict]` instead of using generators, `__slots__`, or columnar formats (polars/numpy).

---

## Testing Patterns

### pytest Fixtures and conftest

```python
# conftest.py -- shared fixtures, auto-discovered by pytest
import pytest
from myapp.db import Database

@pytest.fixture(scope="session")
def db():
    """One database connection for the entire test session."""
    database = Database(":memory:")
    database.migrate()
    yield database
    database.close()

@pytest.fixture
def client(db):
    """Fresh test client per test (function scope by default)."""
    from myapp import create_app
    app = create_app(db=db)
    return app.test_client()
```

### Parametrize

```python
import pytest

@pytest.mark.parametrize("input_val, expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
    ("", ""),
    ("123", "123"),
])
def test_upper(input_val: str, expected: str):
    assert input_val.upper() == expected

# Parametrize with IDs for readable output
@pytest.mark.parametrize("status, should_retry", [
    pytest.param(429, True, id="rate-limited"),
    pytest.param(500, True, id="server-error"),
    pytest.param(200, False, id="success"),
    pytest.param(404, False, id="not-found"),
])
def test_retry_logic(status: int, should_retry: bool):
    assert needs_retry(status) == should_retry
```

### Mocking Strategies

```python
from unittest.mock import patch, AsyncMock, MagicMock

# Patch where the name is used, not where it's defined
@patch("myapp.service.httpx.AsyncClient.get")
async def test_fetch_data(mock_get):
    mock_get.return_value = AsyncMock(
        json=lambda: {"key": "value"},
        status_code=200,
    )
    result = await fetch_data("/endpoint")
    assert result == {"key": "value"}
    mock_get.assert_called_once()

# Use spec to catch attribute errors in mocks
mock_db = MagicMock(spec=Database)
mock_db.query.return_value = [{"id": 1}]
# mock_db.nonexistent_method()  # raises AttributeError -- good
```

### Property-Based Testing with Hypothesis

```python
from hypothesis import given, strategies as st, settings

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs: list[int]):
    sorted_once = sorted(xs)
    sorted_twice = sorted(sorted_once)
    assert sorted_once == sorted_twice

@given(st.text(min_size=1), st.text(min_size=1))
def test_concat_length(a: str, b: str):
    assert len(a + b) == len(a) + len(b)

@settings(max_examples=500)
@given(st.dictionaries(st.text(), st.integers()))
def test_serialization_round_trip(data: dict[str, int]):
    import json
    assert json.loads(json.dumps(data)) == data
```

### Coverage Best Practices

```bash
# Run with coverage
pytest --cov=src --cov-report=term-missing --cov-branch

# Fail if coverage drops below threshold
pytest --cov=src --cov-fail-under=90
```

```ini
# pyproject.toml coverage config
[tool.coverage.run]
branch = true
source = ["src"]
omit = ["*/tests/*", "*/migrations/*"]

[tool.coverage.report]
show_missing = true
fail_under = 90
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

**Anti-patterns to avoid:**
- Fixtures with `scope="session"` that mutate shared state -- causes test pollution.
- Mocking return values without `spec` -- mistyped attributes silently pass.
- Targeting 100% coverage over meaningful assertions -- high coverage with weak asserts is a false signal.

---

## Common Patterns Quick Reference

| Pattern | When to Use | Key Module/Tool |
|---|---|---|
| `Protocol` | Structural typing without inheritance | `typing` |
| `TypeGuard` | Narrowing types in conditionals | `typing` |
| `ParamSpec` | Preserving decorated function signatures | `typing` |
| `TaskGroup` | Structured concurrent async tasks (3.11+) | `asyncio` |
| `Semaphore` | Rate-limiting concurrent operations | `asyncio` |
| `@functools.wraps` | Every decorator wrapper function | `functools` |
| `__init_subclass__` | Class registration, validation hooks | built-in |
| Descriptors | Reusable attribute validation logic | built-in |
| `__slots__` | Memory optimization for many instances | built-in |
| `pyproject.toml` | All project config (build, lint, test) | `hatchling`/`uv` |
| `src/` layout | Prevent accidental local imports | convention |
| `uv` | Fast dependency management and packaging | CLI tool |
| `cProfile` / `py-spy` | CPU profiling | stdlib / PyPI |
| `tracemalloc` | Memory profiling | stdlib |
| `ProcessPoolExecutor` | CPU-bound parallelism | `concurrent.futures` |
| `ThreadPoolExecutor` | I/O-bound parallelism | `concurrent.futures` |
| `pytest.fixture` | Shared setup/teardown in tests | `pytest` |
| `@pytest.mark.parametrize` | Data-driven test cases | `pytest` |
| `hypothesis` | Property-based / fuzz testing | `hypothesis` |
| `--cov-branch` | Branch-aware coverage measurement | `pytest-cov` |

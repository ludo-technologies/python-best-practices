---
name: coding-standards
description: Python coding standards and best practices for AI coding agents
globs:
  - "**/*.py"
  - "pyproject.toml"
  - "setup.py"
  - "requirements*.txt"
---

# Python Coding Standards

A comprehensive collection of Python coding standards and best practices. Designed for AI agents and LLMs to generate high-quality, performant, and maintainable Python code.

## Categories

### Performance Optimization [CRITICAL]
Apply Python optimization patterns to improve processing speed and memory efficiency.

| Rule | Description |
|------|-------------|
| [perf-list-comprehension](rules/perf-list-comprehension.md) | Prefer list comprehensions over loops (1.5-2x faster) |
| [perf-generator-expression](rules/perf-generator-expression.md) | Use generators for large datasets (O(1) memory) |
| [perf-dict-get](rules/perf-dict-get.md) | Use dict.get() for efficient default values |
| [perf-set-lookup](rules/perf-set-lookup.md) | Use set for fast lookups (O(1) vs O(n)) |
| [perf-str-join](rules/perf-str-join.md) | Use join for string concatenation (O(n) vs O(n²)) |

### Async Processing [HIGH]
Efficient asynchronous programming patterns using asyncio.

| Rule | Description |
|------|-------------|
| [async-gather](rules/async-gather.md) | Use asyncio.gather for independent tasks |
| [async-create-task](rules/async-create-task.md) | Proper background task creation |
| [async-context-manager](rules/async-context-manager.md) | Resource management with async with |
| [async-semaphore](rules/async-semaphore.md) | Limit concurrency with semaphores |

### Design Principles [HIGH]
Software design principles for maintainability and extensibility.

| Rule | Description |
|------|-------------|
| [design-philosophy](rules/design-philosophy.md) | DRY, YAGNI, KISS principles |
| [design-single-responsibility](rules/design-single-responsibility.md) | Single Responsibility Principle |
| [design-dependency-injection](rules/design-dependency-injection.md) | Loose coupling with dependency injection |
| [design-pure-functions](rules/design-pure-functions.md) | Prefer pure functions without side effects |
| [design-early-return](rules/design-early-return.md) | Reduce nesting with early returns |

### Documentation [HIGH]
Documentation standards for public API clarity and machine-checkable contracts.

| Rule | Description |
|------|-------------|
| [doc-docstring](rules/doc-docstring.md) | Document public APIs with Google style docstrings |
| [doc-type-hints](rules/doc-type-hints.md) | Require type hints for public APIs |

### Object-Oriented Programming [MEDIUM]
Best practices for Pythonic object-oriented programming.

| Rule | Description |
|------|-------------|
| [oop-composition-over-inheritance](rules/oop-composition-over-inheritance.md) | Prefer composition over inheritance |
| [oop-dataclass](rules/oop-dataclass.md) | Use dataclass for data containers |
| [oop-protocol](rules/oop-protocol.md) | Prefer Protocol over abstract base classes |
| [oop-property](rules/oop-property.md) | Use property instead of getters |

## Quick Reference

### Performance Patterns
```python
# List comprehension (not loops)
result = [x * 2 for x in items]

# Generator for large data
total = sum(x * x for x in range(1_000_000))

# dict.get() with default
value = config.get("key", default_value)

# Set for fast lookup
valid_ids: set[int] = {1, 2, 3}
if item_id in valid_ids: ...

# join for strings
result = ",".join(values)
```

### Async Patterns
```python
# Concurrent execution
results = await asyncio.gather(task1(), task2(), task3())

# Resource management
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.json()

# Concurrency limit
semaphore = asyncio.Semaphore(10)
async with semaphore:
    await do_work()
```

### Design Patterns
```python
# Dependency injection
class Service:
    def __init__(self, repository: Repository) -> None:
        self.repository = repository

# Early return
def process(data: Data | None) -> Result:
    if data is None:
        return Result.empty()
    # main logic here
```

### Documentation Patterns
```python
def create_user(email: str, name: str) -> User:
    """Create a user account.

    Args:
        email: Unique email address for the account.
        name: Display name for the user.

    Returns:
        The created user.
    """
    return user_repository.create(email=email, name=name)
```

### OOP Patterns
```python
# Dataclass
@dataclass
class User:
    name: str
    email: str

# Protocol for interfaces
class Repository(Protocol):
    def get(self, id: str) -> Entity: ...

# Property
@property
def full_name(self) -> str:
    return f"{self.first} {self.last}"
```

## See Also

- [Recommended Tooling](tooling.md) - Tools to enforce these standards automatically (ruff, mypy, pytest, pyscn, uv)

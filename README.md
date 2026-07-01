# python-best-practices

Python best practices skill for AI coding agents. Provides structured guidelines for writing high-quality, performant, and maintainable Python code.

## Installation

```bash
uvx add-skills ludo-technologies/python-best-practices
```

## Skills

### coding-standards

Python coding standards and best practices. 23 rules across 5 categories.

| Category | Impact | Rules |
|----------|--------|-------|
| **Performance Optimization** | CRITICAL | list comprehension, generator expression, dict.get(), set lookup, str.join() |
| **Async Processing** | HIGH | asyncio.gather, create_task, async context manager, semaphore |
| **Design Principles** | HIGH | DRY/YAGNI/KISS, single responsibility, dependency injection, OCP, LSP, ISP, pure functions, early return |
| **Documentation** | HIGH | Google style docstrings, type hints for public APIs |
| **Object-Oriented Programming** | MEDIUM | composition over inheritance, dataclass, Protocol, property |

### tooling

Python development tooling configuration. 7 rules across 6 categories.

| Category | Impact | Tools |
|----------|--------|-------|
| **Analysis** | HIGH | pyscn (dead code, clones, complexity) |
| **Linting** | CRITICAL | ruff |
| **Type Checking** | HIGH | mypy |
| **Formatting** | HIGH | ruff format |
| **Testing** | HIGH | pytest |
| **Package Management** | MEDIUM | uv, pyproject.toml |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new rules.

## License

MIT

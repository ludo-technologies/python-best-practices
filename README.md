# python-best-practices

Python best practices skill for AI coding agents. Provides structured guidelines for writing high-quality, performant, and maintainable Python code.

## Installation

```bash
# Recommended
uvx add-skills ludo-technologies/python-best-practices

# Alternative (if you use pipx)
pipx run add-skills ludo-technologies/python-best-practices
```

## Skills

### coding-standards

Python coding standards and best practices. 25 rules across 7 categories.

| Category | Impact | Rules |
|----------|--------|-------|
| **Error Handling** | CRITICAL | never swallow exceptions |
| **Performance Optimization** | CRITICAL | list comprehension, generator expression, dict.get(), set lookup, str.join() |
| **Async Processing** | HIGH | asyncio.gather, create_task, async context manager, semaphore |
| **Design Principles** | HIGH | DRY/YAGNI/KISS, single responsibility, dependency injection, OCP, LSP, ISP, pure functions, early return |
| **Documentation** | HIGH | Google style docstrings, type hints for public APIs |
| **Data Validation** | HIGH | Pydantic for boundary data validation |
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

### testing

Python test-writing best practices with pytest. 12 rules across 4 categories.

| Category | Impact | Rules |
|----------|--------|-------|
| **Mocking** | CRITICAL | mock boundaries only, autospec, monkeypatch |
| **Test Structure** | HIGH | Arrange-Act-Assert, behavior-based naming, one behavior per test, no logic in tests |
| **Fixtures** | HIGH | narrowest scope, conftest placement, factory fixtures |
| **Parametrization** | MEDIUM | parametrize, readable ids |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new rules.

## License

MIT

# python-best-practices

Python best practices skill for AI coding agents. Provides structured guidelines for writing high-quality, performant, and maintainable Python code.

## Installation

#### Recommended (No installation)
```bash
uvx add-skills ludo-technologies/python-best-practices
```
#### Alternative (if you use pipx)

```bash
pipx run add-skills ludo-technologies/python-best-practices
```

## Skills

### coding-standards

Python coding standards and best practices. 26 rules across 7 categories.

| Category | Impact | Rules |
|----------|--------|-------|
| **Error Handling** | CRITICAL | never swallow exceptions |
| **Performance Optimization** | CRITICAL | list comprehension, generator expression, dict.get(), set lookup, str.join() |
| **Async Processing** | HIGH | asyncio.gather, create_task, async context manager, semaphore |
| **Design Principles** | HIGH | DRY/YAGNI/KISS, single responsibility, dependency injection, no global singleton, OCP, LSP, ISP, pure functions, early return |
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

### data-science

NumPy and pandas best practices for data science and ML code. 13 rules across 8 categories.

| Category | Impact | Rules |
|----------|--------|-------|
| **Vectorization** | CRITICAL | no Python loops / iterrows / row-wise apply, build arrays and frames once |
| **Indexing & Mutation** | CRITICAL | .loc not chained indexing, no inplace=True |
| **Data Types** | HIGH | explicit dtypes at load, category dtype, explicit NumPy dtypes |
| **Schema Validation** | HIGH | pandera for DataFrame boundaries |
| **Reproducibility** | HIGH | np.random.default_rng, no global seed |
| **Transformation Style** | MEDIUM | method chaining with assign/pipe, named aggregation |
| **Typing** | MEDIUM | numpy.typing.NDArray, pandas-stubs |
| **Testing** | MEDIUM | assert_allclose, assert_frame_equal |

### web-application

FastAPI web application best practices with SQLAlchemy 2.0, Alembic, and pydantic-settings. 22 rules across 8 categories.

| Category | Impact | Rules |
|----------|--------|-------|
| **Security** | CRITICAL | Argon2 password hashing, secrets and compare_digest, JWT validation, object-level authorization, explicit CORS origins |
| **Database** | CRITICAL | bound parameters, session per request, one commit per use case, eager loading with lazy="raise", Alembic migrations, SQLAlchemy 2.0 style |
| **Async I/O** | CRITICAL | no blocking calls in async def, shared httpx.AsyncClient with timeouts |
| **API Design** | HIGH | separate Create/Update/Read models, exception handlers for domain errors, bounded keyset pagination |
| **Configuration** | HIGH | pydantic-settings with SecretStr, fail at startup |
| **App Structure** | HIGH | app factory with lifespan, thin routes with Annotated dependencies |
| **Observability** | MEDIUM | request ID logging |
| **Testing** | MEDIUM | dependency_overrides with ASGITransport, real database with per-test rollback |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new rules.

## License

MIT

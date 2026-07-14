---
title: Project Configuration with pyproject.toml
impact: MEDIUM
impactDescription: Modern unified configuration
tags: [pyproject.toml, packaging, pep621, dependencies]
---

# Project Configuration with pyproject.toml [MEDIUM]

## Description
`pyproject.toml` is the single configuration file for modern Python projects. It consolidates project metadata, dependencies, and all tool configurations in one place, following PEP 621.

## Complete Example

```toml
[project]
name = "myproject"
version = "1.0.0"
description = "A Python project"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.11"
authors = [
    {name = "Your Name", email = "you@example.com"},
]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]
dependencies = [
    "fastapi>=0.100.0",
    "pydantic>=2.0",
    "sqlalchemy>=2.0",
]

[project.optional-dependencies]
dev = [
    "ruff>=0.8.0",
    "mypy>=1.13.0",
]
test = [
    "pytest>=8.0",
    "pytest-cov>=4.0",
]
docs = [
    "mkdocs>=1.5",
]
all = [
    "myproject[dev,test,docs]",
]

[project.scripts]
myproject = "myproject.cli:main"

[project.urls]
Homepage = "https://github.com/you/myproject"
Documentation = "https://myproject.readthedocs.io"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# Tool configurations
[tool.ruff]
target-version = "py311"
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "W", "I", "UP", "B", "BLE", "SIM", "S110", "S112"]

[tool.ruff.lint.flake8-bandit]
check-typed-exception = true

[tool.mypy]
python_version = "3.11"
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = ["-v", "--tb=short"]

[tool.pyscn]
max_complexity = 15
```

## Dependency Groups

```toml
[project]
# Production only
dependencies = [
    "fastapi>=0.100.0",
    "pydantic>=2.0",
]

[project.optional-dependencies]
# Development tools
dev = ["ruff>=0.8.0", "mypy>=1.13.0"]

# Testing
test = ["pytest>=8.0", "pytest-cov>=4.0"]

# All development dependencies
all = ["myproject[dev,test]"]
```

```bash
# Install production only
uv pip install -e .

# Install with dev tools
uv pip install -e ".[dev]"

# Install everything
uv pip install -e ".[all]"
```

## Key Sections

| Section | Purpose |
|---------|---------|
| `[project]` | Package metadata (PEP 621) |
| `[project.optional-dependencies]` | Extra dependencies |
| `[project.scripts]` | CLI entry points |
| `[build-system]` | Build backend |
| `[tool.*]` | Tool configurations |

## Notes
- `[project]` follows PEP 621 standard
- Use `>=` for minimum versions, not exact pins
- Group related extras (dev, test, docs)
- All tool configs go in `[tool.xxx]` sections
- Build backends: hatchling, setuptools, flit

## References
- [PEP 621](https://peps.python.org/pep-0621/)
- [Python Packaging Guide](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/)

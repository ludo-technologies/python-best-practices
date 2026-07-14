---
title: Linting with Ruff
impact: CRITICAL
impactDescription: Fast, comprehensive linting
tags: [ruff, linting, pyproject.toml, configuration]
---

# Linting with Ruff [CRITICAL]

## Description
Ruff is an extremely fast Python linter written in Rust. It replaces flake8, isort, pyupgrade, and many other tools with a single, unified linter. Configure it in `pyproject.toml` for consistent code quality.

## Basic Configuration

```toml
# pyproject.toml
[tool.ruff]
target-version = "py311"
line-length = 88
exclude = [
    ".venv",
    "migrations",
    "__pycache__",
]

[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "F",      # Pyflakes
    "W",      # pycodestyle warnings
    "I",      # isort
    "UP",     # pyupgrade
    "B",      # flake8-bugbear
    "BLE",    # flake8-blind-except
    "SIM",    # flake8-simplify
    "PTH",    # flake8-use-pathlib
    "RUF",    # Ruff-specific rules
    "S110",   # Disallow try/except/pass
    "S112",   # Disallow try/except/continue
]
ignore = []

[tool.ruff.lint.flake8-bandit]
check-typed-exception = true

# For mature projects, consider adding:
# "C4"   - flake8-comprehensions
# "DTZ"  - flake8-datetimez
# "T20"  - flake8-print
# "ARG"  - flake8-unused-arguments
# "PL"   - Pylint
# "PERF" - Perflint
```

## Per-File Ignores

```toml
[tool.ruff.lint.per-file-ignores]
# Test files
"tests/**/*.py" = [
    "S101",    # assert is fine in tests
    "ARG001",  # unused fixtures
    "PLR2004", # magic values ok
]

# __init__.py
"__init__.py" = [
    "F401",    # unused imports (re-exports)
]

# Scripts
"scripts/**/*.py" = [
    "T201",    # print allowed
]
```

## Auto-Fix

```toml
[tool.ruff.lint]
fixable = ["ALL"]
unfixable = [
    "F841",   # unused variable - might be intentional
    "ERA001", # commented code - might be needed
]
```

## Commands

```bash
ruff check .                    # Check for issues
ruff check . --fix              # Auto-fix safe issues
ruff check . --fix --unsafe-fixes  # Include unsafe fixes
ruff check . --select=ALL       # Check all rules
ruff check . --statistics       # Show rule counts
```

## Rule Categories

| Prefix | Name | Purpose |
|--------|------|---------|
| E/W | pycodestyle | PEP 8 style |
| F | Pyflakes | Logic errors |
| I | isort | Import sorting |
| UP | pyupgrade | Modernize syntax |
| B | flake8-bugbear | Bug patterns |
| BLE | flake8-blind-except | Broad exception handlers |
| SIM | flake8-simplify | Simplifications |
| PTH | flake8-use-pathlib | Path handling |
| RUF | Ruff | Ruff-specific |
| S110 | flake8-bandit | Silent try/except/pass |
| S112 | flake8-bandit | Silent try/except/continue |

## Notes
- Always specify `target-version` to match your project's minimum Python
- Enable `BLE001`, `S110`, and `S112`, with `check-typed-exception = true`, to detect broad handlers and silent `pass` or `continue`
- Ruff permits broad handlers that call `logging.exception`; review them to ensure they are explicit application boundaries rather than swallowed failures
- Use `--fix` for safe auto-fixes, review `--unsafe-fixes` before applying
- Start with recommended rules, add more as codebase matures
- Use `per-file-ignores` instead of global `ignore` when possible

## References
- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Ruff Rules](https://docs.astral.sh/ruff/rules/)

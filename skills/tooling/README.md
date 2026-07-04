# Python Tooling

A comprehensive guide to Python development tools, designed for AI agents and LLMs to configure and use modern Python tooling effectively.

## Overview

This skill provides 7 rules across 6 categories:

| Category | Prefix | Impact | Rules |
|----------|--------|--------|-------|
| Analysis | `analysis-` | HIGH | 1 |
| Linting | `lint-` | CRITICAL | 1 |
| Type Checking | `type-` | HIGH | 1 |
| Formatting | `fmt-` | HIGH | 1 |
| Testing | `test-` | HIGH | 1 |
| Package Management | `pkg-` | MEDIUM | 2 |

## Structure

```
skills/tooling/
├── SKILL.md              # Skill overview with quick reference
├── metadata.json         # Metadata (version, description)
├── README.md             # This file
└── rules/
    ├── analysis-pyscn.md # Code quality analysis
    ├── lint-ruff.md      # Linting
    ├── type-mypy.md      # Type checking
    ├── fmt-ruff.md       # Formatting
    ├── test-pytest.md    # Testing
    ├── pkg-uv.md         # Package management
    └── pkg-pyproject.md  # Project configuration
```

## Rules

### Analysis (HIGH)
- `analysis-pyscn` - Dead code, clones, dependencies, complexity with pyscn

### Linting (CRITICAL)
- `lint-ruff` - Fast, comprehensive linting with ruff

### Type Checking (HIGH)
- `type-mypy` - Static type checking with mypy

### Formatting (HIGH)
- `fmt-ruff` - Code formatting and import sorting with ruff

### Testing (HIGH)
- `test-pytest` - Testing with pytest, fixtures, and coverage

See the [testing skill](../testing/README.md) for how to write the tests themselves (structure, fixtures, parametrization, mocking).

### Package Management (MEDIUM)
- `pkg-uv` - Fast package management with uv
- `pkg-pyproject` - Project configuration with pyproject.toml

## Tools Covered

| Tool | Purpose |
|------|---------|
| [pyscn](https://github.com/ludo-technologies/pyscn) | Code Quality Analysis |
| [ruff](https://docs.astral.sh/ruff/) | Linting & Formatting |
| [mypy](https://mypy.readthedocs.io/) | Type Checking |
| [pytest](https://docs.pytest.org/) | Testing |
| [uv](https://docs.astral.sh/uv/) | Package Management |

## Usage

This skill is automatically applied when working with Python files and configuration files (`pyproject.toml`, `mypy.ini`, `ruff.toml`).

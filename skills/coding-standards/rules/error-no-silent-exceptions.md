---
title: Never Swallow Exceptions
impact: CRITICAL
impactDescription: Prevent silent failures and hidden data loss
tags: [exceptions, errors, logging, reliability]
---

# Never Swallow Exceptions [CRITICAL]

## Description
Catch an exception only when the current layer can recover, add meaningful context, or handle it at an application boundary. Never hide an unexpected failure with `pass`, an unexplained default value, or a log message followed by normal execution. Silent failures make corrupted or incomplete results look successful and remove the traceback needed to diagnose the cause.

## Bad Example
```python
def load_settings(path: Path) -> Settings:
    try:
        return Settings.from_json(path.read_text())
    except Exception:
        pass

    return Settings()  # Invalid JSON and I/O failures look like valid defaults
```

## Good Example
```python
class SettingsError(Exception):
    """Raised when application settings cannot be loaded."""


def load_settings(path: Path) -> Settings:
    try:
        content = path.read_text()
        return Settings.from_json(content)
    except FileNotFoundError:
        return Settings()  # A missing optional file has a documented fallback
    except (OSError, ValueError) as exc:
        raise SettingsError(f"Failed to load settings from {path}") from exc
```

## Notes
- Catch the narrowest exception types you can handle. Avoid bare `except`, `except BaseException`, and broad `except Exception` in library or domain code.
- If a function cannot recover or add useful context, do not catch the exception; let it propagate with its original traceback.
- When translating an exception, use `raise NewError(...) from exc` to preserve the causal chain.
- Logging is not recovery. Log and suppress an exception only at an explicit boundary, such as a task supervisor or command entry point, where the failure is converted to a defined outcome.
- Never suppress cancellation or process-control exceptions. Let `asyncio.CancelledError`, `KeyboardInterrupt`, and `SystemExit` propagate unless the boundary performs cleanup and re-raises them.
- If degraded behavior is intentional, make it observable and document which failures trigger the fallback; do not return a plausible default for unexpected failures.
- Enable Ruff's `BLE001` to flag many broad exception handlers and `S110` with `check-typed-exception = true` to flag typed `try`/`except`/`pass` patterns. These checks supplement review; logging calls are allowed by the lint rules but can still swallow failures.

## References
- [Python Tutorial - Handling Exceptions](https://docs.python.org/3/tutorial/errors.html#handling-exceptions)
- [Python Documentation - Exception Chaining](https://docs.python.org/3/reference/simple_stmts.html#the-raise-statement)
- [Ruff BLE001](https://docs.astral.sh/ruff/rules/blind-except/)
- [Ruff S110](https://docs.astral.sh/ruff/rules/try-except-pass/)
- [Ruff flake8-bandit settings](https://docs.astral.sh/ruff/settings/#lint_flake8-bandit_check-typed-exception)

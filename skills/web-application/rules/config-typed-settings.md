---
title: Load Configuration into a Typed Settings Object at Startup
impact: HIGH
impactDescription: Bad configuration fails the deploy, not the first request
tags: [configuration, pydantic-settings, secrets, environment, fail-fast]
---

# Load Configuration into a Typed Settings Object at Startup [HIGH]

## Description
`os.getenv("X", "default")` scattered across modules postpones failures to runtime, turns a typo into a silent default, and invites insecure fallbacks such as `os.getenv("JWT_SECRET", "dev-secret")`. Declare all configuration in one `pydantic-settings` class: per-environment values have no default, secrets are `SecretStr`, and every value is validated. Construct it once in the entrypoint, pass it to the app factory, and read it through a dependency.

The `Settings` class below is the one every example in this skill uses.

## Bad Example
```python
# app/security.py
JWT_SECRET = os.getenv("JWT_SECRET", "dev-secret")  # production signs with "dev-secret"
TOKEN_TTL = int(os.getenv("TOKEN_TTL", "15"))  # "15m" crashes at import

# app/db.py
DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:///./dev.db")  # unset -> wrong db

# app/payments.py
print(f"Using payments key {os.environ['PAYMENTS_API_KEY']}")  # secret in the logs
```

## Good Example
```python
# app/config.py
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="APP_", env_file=".env", extra="ignore"
    )

    database_url: SecretStr  # required: no default
    jwt_secret: SecretStr = Field(min_length=32)
    jwt_issuer: str
    jwt_audience: str
    access_token_ttl_minutes: int = Field(default=15, gt=0, le=60)
    cors_origins: list[
        str
    ] = []  # JSON in env: APP_CORS_ORIGINS='["https://app.example.com"]'
    payments_base_url: str
    payments_api_key: SecretStr


# app/deps.py
def get_settings(request: Request) -> Settings:
    settings: Settings = request.app.state.settings
    return settings


SettingsDep = Annotated[Settings, Depends(get_settings)]


# app/asgi.py: the only module that reads the environment
app = create_app(Settings())  # ValidationError lists every missing or invalid field
```

## Notes
- `SecretStr` prints as `'**********'` in `repr`, logs, and tracebacks; call `get_secret_value()` only where the value is used (engine URL, JWT signing).
- Enable the Pydantic mypy plugin (`plugins = ["pydantic.mypy"]` under `[tool.mypy]`) so `Settings()` without arguments and plain strings for `SecretStr` fields type-check.
- Tests build `Settings(...)` with test values and pass it to `create_app`; no environment patching is needed (see `test-dependency-overrides`).
- The FastAPI docs show `@lru_cache` on a `get_settings()` function. It works with `dependency_overrides`, but the cached instance can be imported from anywhere; constructing settings once in the entrypoint keeps them injected (see `coding-standards/design-no-global-singleton`).
- `.env` is for local development only: keep it in `.gitignore` and commit a `.env.example` without values. In production, inject secrets as environment variables from the platform's secret store.
- Group related settings with nested models and `env_nested_delimiter="__"` (`APP_DB__POOL_SIZE=10`).

## References
- [Pydantic - Settings Management](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [FastAPI - Settings and Environment Variables](https://fastapi.tiangolo.com/advanced/settings/)
- [Pydantic - mypy plugin](https://docs.pydantic.dev/latest/integrations/mypy/)

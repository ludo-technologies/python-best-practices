---
title: Build the App in a Factory and Manage Resources in lifespan
impact: HIGH
impactDescription: No side effects at import time; resources open once, close on shutdown, and can be replaced in tests
tags: [app-structure, fastapi, lifespan, app-factory, resources]
---

# Build the App in a Factory and Manage Resources in lifespan [HIGH]

## Description
A module-level `engine = create_async_engine(...)`, `client = httpx.AsyncClient()`, and `app = FastAPI()` with `@app.on_event("startup")` hooks mean that importing a module reads the environment and opens resources, every test shares one global app, and clients are never closed. `on_event` is deprecated. Build the app in `create_app(settings)`, which registers middleware, exception handlers, and routers, and acquire long-lived resources such as the engine and HTTP clients in a `lifespan` async context manager that releases them after `yield`. Only the ASGI entrypoint module calls the factory.

## Bad Example
```python
# app/main.py
engine = create_async_engine(os.environ["DATABASE_URL"])  # reads env and builds a pool at import
http_client = httpx.AsyncClient()  # never closed

app = FastAPI()


@app.on_event("startup")  # deprecated
async def startup() -> None:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

## Good Example
```python
# app/main.py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

import httpx
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine


def create_app(settings: Settings) -> FastAPI:
    @asynccontextmanager
    async def lifespan(app: FastAPI) -> AsyncIterator[None]:
        engine = create_async_engine(settings.database_url.get_secret_value(), pool_pre_ping=True)
        try:
            async with httpx.AsyncClient(
                base_url=settings.payments_base_url, timeout=httpx.Timeout(10.0, connect=3.0)
            ) as payments_client:
                app.state.sessionmaker = async_sessionmaker(engine, expire_on_commit=False)
                app.state.payments_client = payments_client
                yield
        finally:
            await engine.dispose()

    app = FastAPI(lifespan=lifespan)
    app.state.settings = settings
    add_exception_handlers(app)
    app.include_router(auth.router)
    app.include_router(users.router)
    app.include_router(invoices.router)
    return app
```

```python
# app/asgi.py: entrypoint for `fastapi run app/asgi.py` or `uvicorn app.asgi:app`
from app.config import Settings
from app.main import create_app

app = create_app(Settings())
```

## Notes
- Read lifespan resources through dependencies (`get_session`, `get_payments_client`) rather than touching `app.state` in routes, so tests can override them (see `test-dependency-overrides`).
- With many resources, `contextlib.AsyncExitStack` keeps the lifespan flat.
- The lifespan runs once per worker process; with `--workers 4` each process has its own engine and pool.
- Keep `Base.metadata.create_all` out of startup (see `db-migrations-alembic`).
- Starlette can also take resources from a dict yielded by the lifespan (`yield {"sessionmaker": ...}`, read through `request.state`); choose one style per project.
- Put routers in feature modules (`app/routers/invoices.py`) with `APIRouter(prefix=..., tags=[...])`; the factory only includes them.

## References
- [FastAPI - Lifespan Events](https://fastapi.tiangolo.com/advanced/events/)
- [FastAPI - Bigger Applications - Multiple Files](https://fastapi.tiangolo.com/tutorial/bigger-applications/)
- [Starlette - Lifespan](https://www.starlette.io/lifespan/)

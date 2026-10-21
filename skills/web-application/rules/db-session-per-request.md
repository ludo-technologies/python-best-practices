---
title: Create the Engine Once and a Session per Request
impact: CRITICAL
impactDescription: No cross-request leaks, connection exhaustion, or concurrent-use errors
tags: [database, sqlalchemy, asyncsession, dependency-injection, connection-pool]
---

# Create the Engine Once and a Session per Request [CRITICAL]

## Description
The engine owns the connection pool and should exist once per process: create it in the lifespan and dispose of it on shutdown. A `Session` is a unit of work with its own identity map and transaction and must not be shared between concurrent requests. A module-level session mixes requests' transactions; an engine created per request opens a new pool every time. Hand out sessions through a yield dependency that opens a fresh one per request and always closes it.

## Bad Example
```python
import os

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

engine = create_async_engine(os.environ["DATABASE_URL"])  # created at import time
session = AsyncSession(engine)  # one session shared by every request

router = APIRouter(prefix="/users")


@router.get("/{user_id}")
async def read_user(user_id: int) -> UserRead:
    user = await session.get(User, user_id)  # sees other requests' pending changes
    return UserRead.model_validate(user)


@router.get("/{user_id}/report")
async def user_report(user_id: int) -> Response:
    # a new pool per request, never disposed
    report_engine = create_async_engine(os.environ["DATABASE_URL"])
    async with AsyncSession(report_engine) as report_session:
        ...
```

## Good Example
```python
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

# inside lifespan (see app-factory-lifespan): one engine per process
engine = create_async_engine(
    settings.database_url.get_secret_value(), pool_pre_ping=True
)
app.state.sessionmaker = async_sessionmaker(engine, expire_on_commit=False)
...
await engine.dispose()  # on shutdown


# app/deps.py
async def get_session(request: Request) -> AsyncIterator[AsyncSession]:
    async with request.app.state.sessionmaker() as session:
        yield session  # closed after the request; uncommitted work is rolled back


SessionDep = Annotated[AsyncSession, Depends(get_session)]


# app/routers/users.py
router = APIRouter(prefix="/users")


@router.get("/{user_id}")
async def read_user(user_id: int, session: SessionDep) -> UserRead:
    user = await session.get(User, user_id)
    if user is None:
        raise NotFoundError("User")
    return UserRead.model_validate(user)
```

## Notes
- Use `expire_on_commit=False` with `AsyncSession`. Otherwise every attribute is expired on commit, and reading one afterwards needs implicit I/O, which fails with `MissingGreenlet` in async code.
- Never use one `AsyncSession` from concurrent tasks (`asyncio.gather` over the same session raises errors). Give each task its own session from the sessionmaker, or run the queries one after another.
- FastAPI caches dependencies per request, so the endpoint and sub-dependencies such as `get_current_user` share the same session.
- The dependency only opens and closes the session; committing belongs to the use case (see `db-transaction-boundary`).
- Size the pool for the whole deployment: (`pool_size` + `max_overflow`) multiplied by the number of worker processes must stay below the database's connection limit.
- Background jobs and workers open their own `async with sessionmaker() as session:`; never keep using a request's session after the response.

## References
- [SQLAlchemy - Session Basics: When do I construct a Session](https://docs.sqlalchemy.org/en/20/orm/session_basics.html#when-do-i-construct-a-session-when-do-i-commit-it-and-when-do-i-close-it)
- [SQLAlchemy - Asynchronous I/O (asyncio)](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [FastAPI - Dependencies with yield](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/)

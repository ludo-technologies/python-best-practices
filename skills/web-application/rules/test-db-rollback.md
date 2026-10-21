---
title: Run Tests Against a Real Database in a Rolled-Back Transaction
impact: MEDIUM
impactDescription: Real SQL and constraint bugs caught, tests still isolated
tags: [testing, database, sqlalchemy, postgresql, fixtures]
---

# Run Tests Against a Real Database in a Rolled-Back Transaction [MEDIUM]

## Description
A mocked `AsyncSession` only proves that code calls `add()` and `commit()`, not that queries, constraints, and eager loading work, and SQLite standing in for PostgreSQL hides dialect differences. Run tests against the production database engine, create the schema once per session, and wrap each test in an outer transaction that is rolled back afterwards. With `join_transaction_mode="create_savepoint"`, code under test can `commit()` as usual: it acts on a SAVEPOINT, and the outer rollback discards everything.

## Bad Example
```python
from unittest.mock import AsyncMock


@pytest.fixture
def session() -> AsyncMock:
    return AsyncMock(spec=AsyncSession)  # every query "succeeds"; constraints never run


@pytest.fixture
async def sqlite_session() -> AsyncIterator[AsyncSession]:
    engine = create_async_engine("sqlite+aiosqlite:///test.db")  # not production
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with AsyncSession(engine) as session:
        yield session  # committed rows leak into the next test
```

## Good Example
```python
# tests/conftest.py
import os
from collections.abc import AsyncIterator, Awaitable, Callable

import pytest
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, create_async_engine

TEST_DATABASE_URL = os.environ["TEST_DATABASE_URL"]  # a disposable PostgreSQL database

MakeUser = Callable[..., Awaitable[User]]
MakeInvoice = Callable[..., Awaitable[Invoice]]


@pytest.fixture(scope="session")
def anyio_backend() -> str:
    return "asyncio"


@pytest.fixture(scope="session")
async def engine() -> AsyncIterator[AsyncEngine]:
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest.fixture
async def session(engine: AsyncEngine) -> AsyncIterator[AsyncSession]:
    async with engine.connect() as conn:
        transaction = await conn.begin()
        session = AsyncSession(
            bind=conn, join_transaction_mode="create_savepoint", expire_on_commit=False
        )
        try:
            yield session  # commit() and rollback() in app code act on a SAVEPOINT
        finally:
            await session.close()
            await transaction.rollback()  # nothing from this test survives


@pytest.fixture
def make_user(session: AsyncSession) -> MakeUser:
    async def _make(
        email: str = "alice@example.com", password: str = "correct horse battery"
    ) -> User:
        user = User(email=email, password_hash=await hash_password(password))
        session.add(user)
        await session.flush()  # assigns user.id without committing
        return user

    return _make


@pytest.fixture
def make_invoice(session: AsyncSession) -> MakeInvoice:
    async def _make(owner: User, total_cents: int = 1000) -> Invoice:
        invoice = Invoice(owner_id=owner.id, total_cents=total_cents, lines=[])
        session.add(invoice)
        await session.flush()
        return invoice

    return _make
```

## Notes
- The session-scoped `engine` fixture needs a session-scoped `anyio_backend` fixture, as shown, so all tests share one event loop. With pytest-asyncio, give the engine fixture and the tests the same `loop_scope="session"`.
- Start PostgreSQL in CI with a service container or `docker compose`, or from a session fixture with `testcontainers[postgres]`. Never point `TEST_DATABASE_URL` at a shared development or production database.
- `create_all` is fine for the test schema; CI should still run `alembic upgrade head` or `alembic check` so migrations match the models (see `db-migrations-alembic`).
- Code that opens its own sessions from `app.state.sessionmaker`, such as background jobs, bypasses the override; pass the session in, or bind a test sessionmaker to the same connection.
- Tests of real concurrency (row locks, isolation levels) cannot share one connection; run them against a separate database and clean up by truncating tables.
- Factory fixtures like `make_user` and `make_invoice` follow `testing/fixture-factory`. Pass relationships at construction (`lines=[...]`); a `lazy="raise"` collection cannot be read on a flushed object without an explicit load (see `db-eager-loading`).

## References
- [SQLAlchemy - Joining a Session into an External Transaction](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#joining-a-session-into-an-external-transaction-such-as-for-test-suites)
- [AnyIO - Testing with AnyIO](https://anyio.readthedocs.io/en/stable/testing.html)
- [Testcontainers for Python](https://testcontainers-python.readthedocs.io/)

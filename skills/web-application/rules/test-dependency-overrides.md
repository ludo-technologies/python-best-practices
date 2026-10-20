---
title: Test Endpoints with dependency_overrides, Not Patching
impact: MEDIUM
impactDescription: Real routing, validation, and authorization in every endpoint test
tags: [testing, fastapi, dependency-overrides, httpx, pytest]
---

# Test Endpoints with dependency_overrides, Not Patching [MEDIUM]

## Description
Patching module attributes breaks when code moves and often has no effect: `Annotated[User, Depends(get_current_user)]` holds a reference to the original function, so patching `app.security.get_current_user` changes nothing. Build a fresh app per test with `create_app(test_settings)`, override only the dependencies that leave the process (database session, HTTP clients) through `app.dependency_overrides`, and drive it with `httpx.AsyncClient` over `ASGITransport`. Keep authentication real by logging in with real credentials.

## Bad Example
```python
from unittest.mock import AsyncMock, patch

from fastapi.testclient import TestClient

from app.asgi import app  # global app built from the real environment


def test_read_invoice() -> None:
    with (
        patch(
            "app.routers.invoices.get_owned_invoice",
            AsyncMock(return_value=FAKE_INVOICE),
        ),
        patch(
            "app.security.get_current_user", AsyncMock(return_value=FAKE_USER)
        ),  # no effect
    ):
        response = TestClient(app).get("/invoices/1")
    assert response.status_code == 200  # fails with 401, or passes for the wrong reason
```

## Good Example
```python
# tests/conftest.py
from collections.abc import AsyncIterator

import httpx
import pytest
from pydantic import SecretStr


@pytest.fixture
def settings() -> Settings:
    return Settings(
        database_url=SecretStr(
            "postgresql+asyncpg://unused"
        ),  # session is overridden below
        jwt_secret=SecretStr("test-secret-test-secret-test-secret"),
        jwt_issuer="https://api.test",
        jwt_audience="api.test",
        payments_base_url="https://payments.test",
        payments_api_key=SecretStr("test-key"),
    )


@pytest.fixture
async def client(
    session: AsyncSession, settings: Settings
) -> AsyncIterator[httpx.AsyncClient]:
    app = create_app(settings)

    async def override_session() -> AsyncIterator[AsyncSession]:
        yield session  # per-test transaction, see test-db-rollback

    def fake_payments(request: httpx.Request) -> httpx.Response:
        return httpx.Response(200, json={"status": "paid"})

    payments = httpx.AsyncClient(
        base_url="https://payments.test", transport=httpx.MockTransport(fake_payments)
    )
    app.dependency_overrides[get_session] = override_session
    app.dependency_overrides[get_payments_client] = lambda: payments

    # ASGITransport skips the lifespan; every lifespan resource is overridden above
    transport = httpx.ASGITransport(app=app)
    async with (
        payments,
        httpx.AsyncClient(transport=transport, base_url="http://test") as client,
    ):
        yield client
```

```python
# tests/test_invoices.py
@pytest.mark.anyio
async def test_user_cannot_read_another_users_invoice(
    client: httpx.AsyncClient, make_user: MakeUser, make_invoice: MakeInvoice
) -> None:
    await make_user(email="alice@example.com", password="correct horse battery")
    bob = await make_user(email="bob@example.com")
    bobs_invoice = await make_invoice(owner=bob)
    login = await client.post(
        "/auth/token",
        data={"username": "alice@example.com", "password": "correct horse battery"},
    )

    response = await client.get(
        f"/invoices/{bobs_invoice.id}",
        headers={"Authorization": f"Bearer {login.json()['access_token']}"},
    )

    assert response.status_code == 404
```

## Notes
- Override the function passed to `Depends()` (`get_session`), not the `Annotated` alias.
- A fresh `create_app()` per test discards overrides automatically; if a shared app is unavoidable, call `app.dependency_overrides.clear()` in the fixture teardown.
- `httpx.ASGITransport` skips the lifespan. Override everything the lifespan provides, as above, or wrap the app in `asgi_lifespan.LifespanManager`. `TestClient` runs the lifespan when used as a context manager, but it runs the app in another thread and event loop, so it cannot share an async database connection with your fixtures.
- Overriding `get_current_user` with a fixed user is fine for tests that are not about auth; keep at least one test per protected router that uses a real token.
- Fake external HTTP at the transport (`httpx.MockTransport` or `respx`) instead of patching service functions (see `testing/mock-boundaries`).
- `@pytest.mark.anyio` uses the AnyIO plugin that ships with AnyIO, as in the FastAPI docs; with pytest-asyncio, use `@pytest.mark.asyncio`.

## References
- [FastAPI - Testing Dependencies with Overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/)
- [FastAPI - Async Tests](https://fastapi.tiangolo.com/advanced/async-tests/)
- [HTTPX - Mock Transports](https://www.python-httpx.org/advanced/transports/#mock-transports)

---
title: Share One httpx.AsyncClient with Explicit Timeouts
impact: CRITICAL
impactDescription: Reused connections and bounded waits instead of a TLS handshake per call and requests hanging on a slow upstream
tags: [async, httpx, http-client, timeouts, lifespan]
---

# Share One httpx.AsyncClient with Explicit Timeouts [CRITICAL]

## Description
Creating an `httpx.AsyncClient` for every call throws away connection pooling, paying a new TCP and TLS handshake each time, and leaks connections whenever the client is not closed. Create one client per upstream service in the lifespan, with a `base_url`, timeouts chosen for that upstream, and connection limits; close it on shutdown; and inject it through a dependency so tests can replace the transport. Treat upstream responses as untrusted: check the status code and translate failures into your own exception.

## Bad Example
```python
import httpx


@router.post("/orders/{order_id}/pay")
async def pay(order_id: int, settings: SettingsDep) -> PaymentRead:
    async with httpx.AsyncClient() as client:  # new pool and TLS handshake per request
        response = await client.post(
            f"{settings.payments_base_url}/charges", json={"order_id": order_id}
        )
    return PaymentRead(**response.json())  # a 503 error page is parsed as if it were success
```

## Good Example
```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import Annotated

import httpx
from fastapi import Depends, FastAPI, Request


def create_app(settings: Settings) -> FastAPI:
    @asynccontextmanager
    async def lifespan(app: FastAPI) -> AsyncIterator[None]:
        async with httpx.AsyncClient(
            base_url=settings.payments_base_url,
            timeout=httpx.Timeout(10.0, connect=3.0),
            limits=httpx.Limits(max_connections=50),
            headers={"Authorization": f"Bearer {settings.payments_api_key.get_secret_value()}"},
        ) as payments_client:
            app.state.payments_client = payments_client
            yield

    return FastAPI(lifespan=lifespan)


def get_payments_client(request: Request) -> httpx.AsyncClient:
    client: httpx.AsyncClient = request.app.state.payments_client
    return client


PaymentsClientDep = Annotated[httpx.AsyncClient, Depends(get_payments_client)]


class PaymentGatewayError(Exception):
    pass


async def charge(client: httpx.AsyncClient, order_id: int, amount_cents: int) -> PaymentRead:
    try:
        response = await client.post(
            "/charges", json={"order_id": order_id, "amount": amount_cents}
        )
        response.raise_for_status()
    except httpx.HTTPError as exc:  # timeouts, connection errors, 4xx and 5xx
        raise PaymentGatewayError(f"Charge failed for order {order_id}") from exc
    return PaymentRead.model_validate_json(response.content)
```

## Notes
- httpx applies a 5-second default to every timeout; set timeouts per upstream instead of inheriting it. `requests` has no default timeout at all and blocks the event loop (see `async-no-blocking-calls`).
- Validate upstream payloads with a Pydantic model (`model_validate_json`): another service's response is a trust boundary too.
- Map `PaymentGatewayError` to 502 or 503 in an exception handler without echoing the upstream body (see `api-error-responses`).
- Retry only idempotent requests, or send an idempotency key. `httpx.AsyncHTTPTransport(retries=2)` retries failed connections only; use a library such as `stamina` or `tenacity` for backoff on 429 and 5xx.
- In tests, override `get_payments_client` with a client built on `httpx.MockTransport` (see `test-dependency-overrides`).

## References
- [HTTPX - Clients](https://www.python-httpx.org/advanced/clients/)
- [HTTPX - Timeouts](https://www.python-httpx.org/advanced/timeouts/)
- [HTTPX - Resource Limits](https://www.python-httpx.org/advanced/resource-limits/)

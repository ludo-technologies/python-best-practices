---
title: Share One httpx.AsyncClient with Explicit Timeouts
impact: CRITICAL
impactDescription: Connection reuse and bounded waits per upstream
tags: [async, httpx, http-client, timeouts, lifespan]
---

# Share One httpx.AsyncClient with Explicit Timeouts [CRITICAL]

## Description
Creating an `httpx.AsyncClient` per call throws away connection pooling, paying a new TCP and TLS handshake each time, and leaks connections when the client is not closed. Create one client per upstream in the lifespan with a `base_url`, timeouts chosen for that upstream, and connection limits; close it on shutdown; and inject it through a dependency so tests can replace the transport. Treat upstream responses as untrusted: check the status and translate failures into your own exception.

## Bad Example
```python
import httpx

router = APIRouter(prefix="/invoices", dependencies=[Depends(get_current_user)])


@router.post("/{invoice_id}/pay")
async def pay(invoice_id: int, settings: SettingsDep) -> PaymentRead:
    async with httpx.AsyncClient() as client:  # new pool and TLS handshake per request
        response = await client.post(
            f"{settings.payments_base_url}/charges", json={"invoice_id": invoice_id}
        )
    # a 503 error page is parsed as if it were success
    return PaymentRead(**response.json())
```

## Good Example
```python
from typing import Annotated

import httpx
from fastapi import Depends, Request

# inside lifespan (see app-factory-lifespan): one client per upstream
async with httpx.AsyncClient(
    base_url=settings.payments_base_url,
    timeout=httpx.Timeout(10.0, connect=3.0),
    limits=httpx.Limits(max_connections=50),
    headers={"Authorization": f"Bearer {payments_api_key}"},
) as payments_client:
    app.state.payments_client = payments_client
    yield


# app/deps.py
def get_payments_client(request: Request) -> httpx.AsyncClient:
    client: httpx.AsyncClient = request.app.state.payments_client
    return client


PaymentsClientDep = Annotated[httpx.AsyncClient, Depends(get_payments_client)]


# app/services/payments.py
class PaymentGatewayError(Exception):
    pass


async def charge(
    client: httpx.AsyncClient,
    invoice_id: int,
    amount_cents: int,
    *,
    idempotency_key: str,
) -> PaymentRead:
    try:
        response = await client.post(
            "/charges",
            json={"invoice_id": invoice_id, "amount": amount_cents},
            headers={"Idempotency-Key": idempotency_key},  # a retry cannot charge twice
        )
        response.raise_for_status()
    except httpx.HTTPError as exc:  # timeouts, connection errors, 4xx and 5xx
        raise PaymentGatewayError(f"Charge failed for invoice {invoice_id}") from exc
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

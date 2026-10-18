---
title: Correlate Logs with a Request ID
impact: MEDIUM
impactDescription: Every log line of a failing request can be found from one ID in the response or error report
tags: [observability, logging, request-id, contextvars, middleware]
---

# Correlate Logs with a Request ID [MEDIUM]

## Description
Under concurrency, log lines from different requests interleave, and without a shared identifier you cannot reconstruct what happened in the one that failed. Assign a request ID in middleware (reuse a validated `X-Request-ID` from your proxy, or generate one), keep it in a `ContextVar`, attach it to every log record with a logging filter, and return it in the response header so users and support can quote it. Log through module loggers with structured `extra` fields, never `print`, and never log secrets, tokens, or whole request bodies.

## Bad Example
```python
@router.post("/orders")
async def create_order(body: OrderCreate, request: Request) -> OrderRead:
    print(f"creating order {body}")  # no level or logger; may print card data
    logging.info("auth header: %s", request.headers.get("Authorization"))  # bearer token in logs
    order = await place_order(body)
    logging.info("order created")  # which request? which order?
    return OrderRead.model_validate(order)
```

## Good Example
```python
# app/observability.py
import logging
import re
from collections.abc import Awaitable, Callable
from contextvars import ContextVar
from uuid import uuid4

from fastapi import FastAPI, Request, Response

request_id_var: ContextVar[str] = ContextVar("request_id", default="-")
REQUEST_ID_PATTERN = re.compile(r"[A-Za-z0-9-]{1,64}")


class RequestIdFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id_var.get()
        return True


def configure_logging(level: str) -> None:
    handler = logging.StreamHandler()
    handler.addFilter(RequestIdFilter())
    handler.setFormatter(
        logging.Formatter("%(asctime)s %(levelname)s [%(request_id)s] %(name)s: %(message)s")
    )
    logging.basicConfig(level=level, handlers=[handler], force=True)


def add_request_id_middleware(app: FastAPI) -> None:
    @app.middleware("http")
    async def request_id_middleware(
        request: Request, call_next: Callable[[Request], Awaitable[Response]]
    ) -> Response:
        incoming = request.headers.get("X-Request-ID", "")
        request_id = incoming if REQUEST_ID_PATTERN.fullmatch(incoming) else uuid4().hex
        token = request_id_var.set(request_id)
        try:
            response = await call_next(request)
        finally:
            request_id_var.reset(token)
        response.headers["X-Request-ID"] = request_id
        return response


# app/services/orders.py
logger = logging.getLogger(__name__)


async def place_order(session: AsyncSession, data: OrderCreate) -> Order:
    order = Order(**data.model_dump())
    session.add(order)
    await session.commit()
    logger.info("order placed", extra={"order_id": order.id, "total_cents": order.total_cents})
    return order
```

## Notes
- Validate incoming IDs (length and characters) before trusting them; they end up in log files and response headers.
- The stdlib `Formatter` ignores `extra` fields that are not in its format string. In production, use a JSON formatter (`python-json-logger`) or `structlog` so fields like `order_id` become searchable.
- Context variables are copied into tasks created during the request, including `asyncio.create_task` and `asyncio.to_thread`, so their logs carry the same ID.
- Unhandled exceptions are logged by the server after this middleware has reset the variable; to get the ID on those tracebacks, log with `logger.exception(...)` inside the middleware and re-raise.
- Never log passwords, tokens, `Authorization` or `Cookie` headers, full request bodies, or personal data; log IDs instead.
- Configure logging once in the entrypoint; everything else only calls `logging.getLogger(__name__)`.
- The `asgi-correlation-id` package provides the same behavior as pure ASGI middleware, and OpenTelemetry's FastAPI instrumentation adds trace IDs that work across services.

## References
- [Python Documentation - contextvars](https://docs.python.org/3/library/contextvars.html)
- [Python Logging Cookbook - Adding contextual information](https://docs.python.org/3/howto/logging-cookbook.html#using-filters-to-impart-contextual-information)
- [FastAPI - Middleware](https://fastapi.tiangolo.com/tutorial/middleware/)

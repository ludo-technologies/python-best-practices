---
title: Never Block the Event Loop in async def Endpoints
impact: CRITICAL
impactDescription: One blocking call no longer stalls every request on the worker
tags: [async, fastapi, event-loop, threadpool, performance]
---

# Never Block the Event Loop in async def Endpoints [CRITICAL]

## Description
FastAPI runs `async def` endpoints on the event loop. A blocking call inside one, such as `requests.get`, `time.sleep`, a sync database driver, or CPU-heavy work like password hashing or PDF rendering, freezes the loop: every other request on that worker waits. Inside `async def`, use awaitable libraries only and move unavoidable blocking or CPU-bound calls to a thread with `asyncio.to_thread`. When an endpoint calls nothing but blocking libraries, declare it with plain `def`; FastAPI runs those in a thread pool.

## Bad Example
```python
import time

import requests

router = APIRouter(prefix="/reports", dependencies=[Depends(get_current_user)])


@router.post("")
async def create_report(month: str, user: CurrentUser, session: SessionDep) -> Response:
    invoices = list(
        await session.scalars(select(Invoice).where(Invoice.owner_id == user.id))
    )
    rates = requests.get(
        "https://fx.example.com/rates", timeout=5
    ).json()  # blocks the loop
    time.sleep(1)  # pauses every request on this worker, not just this one
    pdf = render_pdf(invoices, rates)  # seconds of CPU on the event loop
    return Response(pdf, media_type="application/pdf")
```

## Good Example
```python
import asyncio
from pathlib import Path

from fastapi import Response

router = APIRouter(prefix="/reports", dependencies=[Depends(get_current_user)])
ARCHIVE_DIR = Path("/var/reports")


def render_pdf(
    invoices: list[Invoice], rates: dict[str, float]
) -> bytes: ...  # pure CPU work that takes seconds


@router.post("")
async def create_report(
    month: str, user: CurrentUser, session: SessionDep, payments: PaymentsClientDep
) -> Response:
    invoices = list(
        await session.scalars(select(Invoice).where(Invoice.owner_id == user.id))
    )
    response = await payments.get("/fx-rates")  # shared httpx.AsyncClient
    response.raise_for_status()
    # CPU work off the event loop
    pdf = await asyncio.to_thread(render_pdf, invoices, response.json())
    return Response(pdf, media_type="application/pdf")


# Only blocking calls inside: plain def runs in FastAPI's thread pool
@router.get("/{report_id}/archive")
def read_archived_report(report_id: int) -> Response:
    pdf = (
        ARCHIVE_DIR / f"{report_id}.pdf"
    ).read_bytes()  # blocking file I/O is fine here
    return Response(pdf, media_type="application/pdf")
```

## Notes
- Common blockers: `requests`, `urllib`, `boto3`, `psycopg2` or a sync `Session`, the sync `redis` client, `smtplib`, `time.sleep`, `subprocess.run`, file I/O on network storage, Argon2 or bcrypt, and pandas or Pillow processing.
- Async replacements: `httpx.AsyncClient`, `AsyncSession` with asyncpg or psycopg 3, `redis.asyncio`, `aiobotocore`, `anyio.Path`.
- Threads help with blocking I/O and with C extensions that release the GIL. Pure-Python CPU work still holds the GIL; use a `ProcessPoolExecutor` or a task queue, and move jobs that take minutes out of the request into a worker (Celery, arq, RQ).
- Both thread pools are bounded: sync endpoints run in AnyIO's pool (40 threads by default), and `asyncio.to_thread` uses the event loop's separate default executor (`min(32, cpu_count + 4)` threads). Many slow blocking calls can exhaust either.
- During development, `PYTHONASYNCIODEBUG=1` makes asyncio log callbacks that block for more than 100 ms.
- Decide per endpoint: inside `def`, do not call async functions; inside `async def`, do not call sync I/O.

## References
- [FastAPI - Concurrency and async / await](https://fastapi.tiangolo.com/async/)
- [Python Documentation - asyncio.to_thread](https://docs.python.org/3/library/asyncio-task.html#asyncio.to_thread)
- [Python Documentation - Developing with asyncio: debug mode](https://docs.python.org/3/library/asyncio-dev.html#debug-mode)

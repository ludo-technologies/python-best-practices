---
title: Keep Route Functions Thin and Inject Dependencies with Annotated
impact: HIGH
impactDescription: Business logic reusable and testable without HTTP
tags: [app-structure, fastapi, depends, annotated, service-layer]
---

# Keep Route Functions Thin and Inject Dependencies with Annotated [HIGH]

## Description
A route function is the HTTP adapter: it declares inputs, authentication, and the response model, calls a service function, and converts the result. When token parsing, queries, business rules, external calls, and commits pile up inside the route, the logic cannot be reused from a worker or CLI and tests need the full HTTP stack. Put use-case logic in plain async functions that receive their dependencies as parameters, and declare shared dependencies once as `Annotated` aliases.

## Bad Example
```python
router = APIRouter(prefix="/invoices")


@router.post("/{invoice_id}/pay")
async def pay_invoice(
    invoice_id: int, request: Request, token: str = Depends(oauth2_scheme)
) -> dict[str, Any]:
    claims = jwt.decode(token, os.environ["JWT_SECRET"], algorithms=["HS256"])
    async with request.app.state.sessionmaker() as session:
        invoice = await session.get(Invoice, invoice_id)
        if invoice is None or invoice.owner_id != int(claims["sub"]):
            raise HTTPException(status_code=404)
        if invoice.paid_at is not None:
            raise HTTPException(status_code=409, detail="already paid")
        response = await request.app.state.payments_client.post(
            "/charges", json={"amount": invoice.total_cents}
        )
        invoice.paid_at = datetime.now(UTC)
        await session.commit()
    return {"ok": True, "charge": response.json()}
```

## Good Example
```python
# app/deps.py: declared once, reused by every router
SessionDep = Annotated[AsyncSession, Depends(get_session)]
PaymentsClientDep = Annotated[httpx.AsyncClient, Depends(get_payments_client)]
CurrentUser = Annotated[User, Depends(get_current_user)]


# app/services/invoices.py: no FastAPI imports; callable from routes, workers, and tests
async def pay_invoice(
    session: AsyncSession,
    payments: httpx.AsyncClient,
    owner_id: int,
    invoice_id: int,
    now: datetime,
) -> Invoice:
    invoice = await get_owned_invoice(session, owner_id=owner_id, invoice_id=invoice_id)
    if invoice.paid_at is not None:
        raise ConflictError("Invoice already paid")
    await charge(
        payments,
        invoice.id,
        invoice.total_cents,
        idempotency_key=f"invoice-{invoice.id}",
    )
    invoice.paid_at = now
    await session.commit()
    return invoice


# app/routers/invoices.py: HTTP adapter only
router = APIRouter(prefix="/invoices", dependencies=[Depends(get_current_user)])


@router.post("/{invoice_id}/pay")
async def pay(
    invoice_id: int, user: CurrentUser, session: SessionDep, payments: PaymentsClientDep
) -> InvoiceRead:
    invoice = await pay_invoice(
        session, payments, user.id, invoice_id, now=datetime.now(UTC)
    )
    return InvoiceRead.model_validate(invoice)
```

## Notes
- `Annotated[T, Depends(...)]` aliases keep signatures short, are reused across routers, and give type checkers the real type; prefer them to `param: T = Depends(...)` defaults.
- Services take plain values (`owner_id`, `now`), not `Request`. Passing the time in keeps time-dependent logic testable (see `coding-standards/design-pure-functions`).
- Services raise domain errors and routes do not catch them; exception handlers translate them (see `api-error-responses`).
- When a use case calls an external service and then commits, send an idempotency key derived from the record (as `charge` does above) so a retry after a failed commit cannot charge twice.
- A one-line read can query directly in the route. Extract a service when there is logic, more than one step, or reuse; do not add pass-through layers or repository classes that only forward calls (see `coding-standards/design-philosophy`).

## References
- [FastAPI - Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [FastAPI - Bigger Applications - Multiple Files](https://fastapi.tiangolo.com/tutorial/bigger-applications/)

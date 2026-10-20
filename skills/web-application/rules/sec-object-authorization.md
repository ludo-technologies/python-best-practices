---
title: Authorize Access to Every Object, Not Just the Endpoint
impact: CRITICAL
impactDescription: Editing an ID in the URL cannot reach another user's data (IDOR)
tags: [security, authorization, idor, bola, multi-tenancy]
---

# Authorize Access to Every Object, Not Just the Endpoint [CRITICAL]

## Description
Authentication says who the caller is; authorization decides whether they may touch this record. Broken object level authorization, number one in the OWASP API Security Top 10, happens when `/invoices/{invoice_id}` loads the row by primary key and returns it to any logged-in user. Scope every query by owner or tenant in the statement that loads the object, answer 404 for objects the caller may not see, and require authentication at the router level so new endpoints are protected by default.

## Bad Example
```python
router = APIRouter(prefix="/invoices")


@router.get("/{invoice_id}")
async def read_invoice(
    invoice_id: int, user: CurrentUser, session: SessionDep
) -> InvoiceRead:
    invoice = await session.get(Invoice, invoice_id)  # any user can read any invoice
    if invoice is None:
        raise HTTPException(status_code=404)
    return InvoiceRead.model_validate(invoice)


@router.delete("/{invoice_id}", status_code=204)
async def delete_invoice(
    invoice_id: int, session: SessionDep
) -> None: ...  # forgot CurrentUser: anonymous callers can delete
```

## Good Example
```python
router = APIRouter(prefix="/invoices", dependencies=[Depends(get_current_user)])


async def get_owned_invoice(
    session: AsyncSession, owner_id: int, invoice_id: int
) -> Invoice:
    invoice = await session.scalar(
        select(Invoice)
        .where(Invoice.id == invoice_id, Invoice.owner_id == owner_id)
        .options(selectinload(Invoice.lines))
    )
    if invoice is None:
        raise NotFoundError("Invoice")  # same answer for "missing" and "not yours"
    return invoice


@router.get("/{invoice_id}")
async def read_invoice(
    invoice_id: int, user: CurrentUser, session: SessionDep
) -> InvoiceRead:
    invoice = await get_owned_invoice(session, owner_id=user.id, invoice_id=invoice_id)
    return InvoiceRead.model_validate(invoice)


@router.delete("/{invoice_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_invoice(
    invoice_id: int, user: CurrentUser, session: SessionDep
) -> None:
    invoice = await get_owned_invoice(session, owner_id=user.id, invoice_id=invoice_id)
    await session.delete(invoice)  # Invoice.lines cascades "all, delete-orphan"
    await session.commit()
```

## Notes
- `APIRouter(dependencies=[Depends(get_current_user)])` makes "authenticated" the default for the router. FastAPI caches a dependency within a request, so endpoints that also take `CurrentUser` do not decode the token twice.
- Express roles as dependencies (`require_admin(user: CurrentUser) -> User` raising 403) and still scope data queries by owner or tenant.
- In multi-tenant apps, take `tenant_id` from the authenticated principal, never from a header, query parameter, or request body.
- Unguessable IDs (UUIDs) are not access control: they leak through URLs, logs, and screenshots.
- Use 403 only when the caller is allowed to know the object exists, for example read access without write access.
- Write a test per endpoint asserting that another user's ID returns 404 (see `test-dependency-overrides`).

## References
- [OWASP API1:2023 - Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [FastAPI - Dependencies in path operation decorators](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-in-path-operation-decorators/)

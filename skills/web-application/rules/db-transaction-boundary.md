---
title: Commit Once per Use Case, Not in Helpers or Dependency Teardown
impact: CRITICAL
impactDescription: No partial writes, and a failed commit never returns 200
tags: [database, transactions, sqlalchemy, unit-of-work, fastapi]
---

# Commit Once per Use Case, Not in Helpers or Dependency Teardown [CRITICAL]

## Description
A request that changes data should succeed or fail as one transaction. Helpers that each call `commit()` leave the first half persisted when a later step fails. Committing after `yield` in the session dependency is worse: by default FastAPI runs that code after the response is sent, so a failed commit reaches the client as `200 OK`. Let helpers only add and modify objects, and commit once, explicitly, in the function that owns the use case.

## Bad Example
```python
async def get_session(request: Request) -> AsyncIterator[AsyncSession]:
    async with request.app.state.sessionmaker() as session:
        yield session
        await session.commit()  # runs after the response; failures are invisible


async def add_line(session: AsyncSession, invoice: Invoice, line: LineCreate) -> None:
    invoice.lines.append(InvoiceLine(**line.model_dump()))
    await session.commit()  # commits in the middle of the use case


async def add_lines(
    session: AsyncSession, invoice: Invoice, lines: list[LineCreate]
) -> None:
    for line in lines:
        await add_line(session, invoice, line)  # line 3 fails: lines 1 and 2 are saved
```

## Good Example
```python
async def get_session(request: Request) -> AsyncIterator[AsyncSession]:
    async with request.app.state.sessionmaker() as session:
        yield session  # no commit here; closing rolls back anything uncommitted


def add_line(invoice: Invoice, line: LineCreate) -> None:
    invoice.lines.append(InvoiceLine(**line.model_dump()))


async def add_lines(
    session: AsyncSession, owner_id: int, invoice_id: int, lines: list[LineCreate]
) -> Invoice:
    invoice = await get_owned_invoice(session, owner_id=owner_id, invoice_id=invoice_id)
    for line in lines:
        add_line(invoice, line)
    await session.commit()  # one commit; an exception before it rolls everything back
    return invoice


router = APIRouter(prefix="/invoices", dependencies=[Depends(get_current_user)])


@router.post("/{invoice_id}/lines", status_code=status.HTTP_201_CREATED)
async def create_lines(
    invoice_id: int, body: list[LineCreate], user: CurrentUser, session: SessionDep
) -> InvoiceRead:
    invoice = await add_lines(session, user.id, invoice_id, body)
    return InvoiceRead.model_validate(invoice)
```

## Notes
- If you want the dependency to own one transaction per request, open it with `async with sessionmaker.begin() as session: yield session` and declare `Depends(get_session, scope="function")` (FastAPI 0.121+), which runs the teardown before the response is sent so a failed commit becomes a 500. Change the `SessionDep` alias itself so every user shares the same scope and session; a `yield` dependency with the default scope cannot depend on a `scope="function"` dependency.
- `async with session.begin():` inside a service raises if the session already started a transaction, which happens as soon as a dependency such as `get_current_user` runs a query. A single explicit `await session.commit()` works in every case.
- Use `await session.flush()` when you need generated IDs or want constraint errors early; it sends SQL inside the open transaction without committing.
- After a failed flush or commit, call `await session.rollback()` before using the session again. Translate expected failures such as unique violations into domain errors at that point (see `api-error-responses`).
- Side effects that must only happen after the data is saved (emails, queue messages) go after `commit()`, or into an outbox table written in the same transaction.

## References
- [SQLAlchemy - Managing Transactions](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html)
- [FastAPI - Dependencies with yield: Early exit and scope](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/#early-exit-and-scope)

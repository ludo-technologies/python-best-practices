---
title: Load Relationships Explicitly and Make Lazy Loads Raise
impact: CRITICAL
impactDescription: Two queries per page instead of N+1 or MissingGreenlet
tags: [database, sqlalchemy, n-plus-one, selectinload, performance]
---

# Load Relationships Explicitly and Make Lazy Loads Raise [CRITICAL]

## Description
Reading a relationship that was not loaded triggers a lazy load. In sync code that silently issues one query per row: 50 invoices with their lines run 51 queries. In async code the implicit I/O fails with `MissingGreenlet`, often only on some code paths. Declare `lazy="raise"` on relationships so any implicit load fails loudly in tests, and load what each query needs with `selectinload` for collections and `joinedload` for many-to-one references.

## Bad Example
```python
class Invoice(Base):
    __tablename__ = "invoices"

    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    lines: Mapped[list["InvoiceLine"]] = relationship()  # default lazy="select"


async def list_invoices(session: AsyncSession, owner_id: int) -> list[InvoiceRead]:
    invoices = await session.scalars(
        select(Invoice).where(Invoice.owner_id == owner_id)
    )
    # Each .lines access is one more query (sync) or MissingGreenlet (async)
    return [InvoiceRead.model_validate(invoice) for invoice in invoices]
```

## Good Example
```python
from sqlalchemy.orm import joinedload, selectinload


class Invoice(Base):  # full model in db-sqlalchemy-2-style
    ...
    owner: Mapped[User] = relationship(lazy="raise")
    lines: Mapped[list["InvoiceLine"]] = relationship(
        lazy="raise", cascade="all, delete-orphan"
    )


async def list_invoices(session: AsyncSession, owner_id: int) -> list[InvoiceRead]:
    invoices = await session.scalars(
        select(Invoice)
        .where(Invoice.owner_id == owner_id)
        .options(
            selectinload(Invoice.lines),  # one extra query: WHERE invoice_id IN (...)
            joinedload(Invoice.owner),  # many-to-one: JOIN in the same query
        )
        .order_by(Invoice.id)
        .limit(50)
    )
    return [InvoiceRead.model_validate(invoice) for invoice in invoices]
```

## Notes
- `selectinload` suits one-to-many and many-to-many: one extra query per batch of up to 500 parent rows, so a bounded page takes two queries in total. `joinedload` suits many-to-one and one-to-one; on a collection it multiplies rows and requires `.unique()` on the result.
- Chain options for nested data: `selectinload(Invoice.lines).joinedload(InvoiceLine.product)`.
- `cascade="all, delete-orphan"` makes `session.delete(invoice)` delete its lines too. Without it SQLAlchemy sets the children's `invoice_id` to NULL, which fails on a NOT NULL column and leaves orphans otherwise. A database-level `ondelete="CASCADE"` on the foreign key works as well, paired with `passive_deletes=True`.
- On models you cannot change, add `raiseload("*")` as a query option to get the same protection per query.
- A response model that includes relationships (`InvoiceRead.lines`) defines what the query must load; keep the query and the response model it serves next to each other.
- Index foreign key columns used for filtering and loading (`index=True`); PostgreSQL does not index them automatically.
- To count queries in a test, listen to `before_cursor_execute` on `engine.sync_engine`; during development, `echo=True` prints every statement.

## References
- [SQLAlchemy - Relationship Loading Techniques](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html)
- [SQLAlchemy - Preventing Implicit IO when Using AsyncSession](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#preventing-implicit-io-when-using-asyncsession)

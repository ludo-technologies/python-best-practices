---
title: Use SQLAlchemy 2.0 Typed Models and select()
impact: CRITICAL
impactDescription: Type-checked models and queries that work on AsyncSession
tags: [database, sqlalchemy, orm, typing, mapped]
---

# Use SQLAlchemy 2.0 Typed Models and select() [CRITICAL]

## Description
Generated code often follows 1.x tutorials: `declarative_base()`, untyped `Column(...)`, and `session.query(...)`. The `Query` API is legacy, absent from `AsyncSession`, and invisible to type checkers. Declare models with `DeclarativeBase`, `Mapped[...]`, and `mapped_column()`, build statements with `select()`, and run them with `session.scalars()`, `session.scalar()`, `session.execute()`, or `session.get()`.

The models below are the ones every example in this skill uses.

## Bad Example
```python
from sqlalchemy import Boolean, Column, Integer, String
from sqlalchemy.orm import Session, declarative_base

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)  # nullable by default; mypy sees no types
    is_active = Column(Boolean)


def active_users(session: Session) -> list[User]:
    return session.query(User).filter(User.is_active == True).all()  # legacy
```

## Good Example
```python
# app/models.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, MetaData, String, func, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    type_annotation_map = {datetime: DateTime(timezone=True)}
    metadata = MetaData(naming_convention=NAMING_CONVENTION)  # db-migrations-alembic


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(320), unique=True)  # NOT NULL from str
    password_hash: Mapped[str]
    display_name: Mapped[str | None] = mapped_column(String(100))  # NULL from | None
    is_active: Mapped[bool] = mapped_column(default=True)
    reset_token_hash: Mapped[str | None]
    reset_token_expires_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    last_login_at: Mapped[datetime | None]


class Invoice(Base):
    __tablename__ = "invoices"

    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    total_cents: Mapped[int]
    paid_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    owner: Mapped[User] = relationship(lazy="raise")  # see db-eager-loading
    lines: Mapped[list["InvoiceLine"]] = relationship(
        lazy="raise", cascade="all, delete-orphan"
    )


class InvoiceLine(Base):
    __tablename__ = "invoice_lines"

    id: Mapped[int] = mapped_column(primary_key=True)
    invoice_id: Mapped[int] = mapped_column(ForeignKey("invoices.id"), index=True)
    description: Mapped[str] = mapped_column(String(200))
    amount_cents: Mapped[int]


async def active_users(session: AsyncSession, limit: int) -> list[User]:
    stmt = select(User).where(User.is_active).order_by(User.id).limit(limit)
    return list(await session.scalars(stmt))


async def find_by_email(session: AsyncSession, email: str) -> User | None:
    return await session.scalar(select(User).where(User.email == email))
```

## Notes
- `Mapped[...]` is the single source of truth for type and nullability: `Mapped[str]` is `NOT NULL`, `Mapped[str | None]` allows `NULL`. A bare annotation without `mapped_column()` is enough for a plain column.
- `type_annotation_map` makes every `datetime` column timezone-aware without repeating `DateTime(timezone=True)`.
- Use `session.get(User, user_id)` for primary key lookups; it checks the identity map before querying.
- `session.scalars()` returns ORM objects, `session.execute()` returns rows for multi-column selects such as `select(User.id, User.email)`, and `.scalar_one()` asserts exactly one row.
- Run bulk changes as statements: `await session.execute(update(User).where(...).values(is_active=False))`.
- `Mapped[]` annotations type-check without the SQLAlchemy mypy plugin, which was deprecated in 2.0 and removed in 2.1.

## References
- [SQLAlchemy - ORM Quick Start](https://docs.sqlalchemy.org/en/20/orm/quickstart.html)
- [SQLAlchemy - Migrating to 2.0](https://docs.sqlalchemy.org/en/20/changelog/migration_20.html)
- [SQLAlchemy - ORM Querying Guide](https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html)

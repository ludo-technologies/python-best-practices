---
title: Use SQLAlchemy 2.0 Typed Models and select()
impact: CRITICAL
impactDescription: Type-checked models and queries; the legacy Query API does not exist on AsyncSession
tags: [database, sqlalchemy, orm, typing, mapped]
---

# Use SQLAlchemy 2.0 Typed Models and select() [CRITICAL]

## Description
Generated SQLAlchemy code often still follows 1.x tutorials: `declarative_base()`, untyped `Column(Integer)`, and `session.query(...)`. The `Query` API is legacy in 2.0, absent from `AsyncSession`, and gives type checkers nothing to check. Declare models with `DeclarativeBase`, `Mapped[...]`, and `mapped_column()`, build statements with `select()`, and run them with `session.scalars()`, `session.scalar()`, `session.execute()`, or `session.get()`.

## Bad Example
```python
from sqlalchemy import Boolean, Column, Integer, String
from sqlalchemy.orm import Session, declarative_base

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)  # nullable by default; mypy sees no column types
    is_active = Column(Boolean)


def active_users(session: Session) -> list[User]:
    return session.query(User).filter(User.is_active == True).all()  # legacy, unbounded
```

## Good Example
```python
from sqlalchemy import String, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(320), unique=True)  # NOT NULL from Mapped[str]
    display_name: Mapped[str | None] = mapped_column(String(100))  # NULL from "| None"
    is_active: Mapped[bool] = mapped_column(default=True)


async def active_users(session: AsyncSession, limit: int) -> list[User]:
    result = await session.scalars(
        select(User).where(User.is_active).order_by(User.id).limit(limit)
    )
    return list(result)


async def find_by_email(session: AsyncSession, email: str) -> User | None:
    return await session.scalar(select(User).where(User.email == email))
```

## Notes
- The `Mapped[...]` annotation is the single source of truth for the Python type and nullability: `Mapped[str]` is `NOT NULL`, `Mapped[str | None]` allows `NULL`.
- Use `session.get(User, user_id)` for primary key lookups; it checks the identity map before querying.
- `session.scalars()` returns ORM objects, `session.execute()` returns rows for multi-column selects such as `select(User.id, User.email)`, and `.scalar_one()` asserts exactly one row.
- Run bulk changes as statements: `await session.execute(update(User).where(...).values(is_active=False))`.
- `Mapped[]` annotations type-check without the SQLAlchemy mypy plugin, which was deprecated in 2.0 and removed in 2.1.

## References
- [SQLAlchemy - ORM Quick Start](https://docs.sqlalchemy.org/en/20/orm/quickstart.html)
- [SQLAlchemy - Migrating to 2.0](https://docs.sqlalchemy.org/en/20/changelog/migration_20.html)
- [SQLAlchemy - ORM Querying Guide](https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html)

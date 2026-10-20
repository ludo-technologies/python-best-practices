---
title: Never Build SQL with String Formatting
impact: CRITICAL
impactDescription: SQL injection is impossible by construction
tags: [database, sql-injection, sqlalchemy, security, text]
---

# Never Build SQL with String Formatting [CRITICAL]

## Description
SQL built with f-strings, `%`, `+`, or `.format()` from request data is injectable, even for values that look harmless such as IDs, search terms, and sort fields; wrapping the string in `text()` does not help. Use SQLAlchemy expressions, which always bind values, or `text()` with named `:params` for hand-written SQL. Identifiers such as column names cannot be bound; map user input onto a fixed whitelist of columns.

## Bad Example
```python
from sqlalchemy import text

router = APIRouter(prefix="/users")


@router.get("/search")
async def search_users(q: str, sort: str, session: SessionDep) -> list[UserRead]:
    result = await session.execute(
        text(f"SELECT id, email FROM users WHERE email LIKE '%{q}%' ORDER BY {sort}")
    )
    return [UserRead.model_validate(row, from_attributes=True) for row in result]
```

## Good Example
```python
from enum import StrEnum
from typing import Annotated

from fastapi import Query
from sqlalchemy import select, text

router = APIRouter(prefix="/users")


class UserSort(StrEnum):
    EMAIL = "email"
    CREATED = "created_at"


SORT_COLUMNS = {UserSort.EMAIL: User.email, UserSort.CREATED: User.created_at}


@router.get("/search")
async def search_users(
    q: Annotated[str, Query(min_length=1, max_length=100)],
    session: SessionDep,
    sort: UserSort = UserSort.EMAIL,  # anything else is rejected with 422
) -> list[UserRead]:
    stmt = (
        select(User)
        .where(User.email.icontains(q, autoescape=True))  # bound; % _ literal
        .order_by(SORT_COLUMNS[sort])
        .limit(50)
    )
    return [UserRead.model_validate(user) for user in await session.scalars(stmt)]


# Hand-written SQL: bind values, never interpolate them
async def count_recent_logins(session: AsyncSession, since: datetime) -> int:
    result = await session.execute(
        text("SELECT count(*) FROM users WHERE last_login_at >= :since"),
        {"since": since},
    )
    return result.scalar_one()
```

## Notes
- Binding works for values only. `ORDER BY :column` sorts by a constant string; use an `Enum` plus a mapping to column objects for sortable fields.
- For `IN` lists use `User.id.in_(ids)`, or `bindparam("ids", expanding=True)` inside `text()`.
- Bound parameters stop injection, but `%` and `_` in a `LIKE` pattern are still wildcards; `contains`, `icontains`, `startswith`, and `endswith` accept `autoescape=True`.
- `literal_column()` and `text()` fragments inside ORM queries are inserted verbatim; never feed them request data.
- Ruff `S608` flags SQL built from strings.

## References
- [SQLAlchemy - Using Textual SQL](https://docs.sqlalchemy.org/en/20/tutorial/dbapi_transactions.html#sending-parameters)
- [SQLAlchemy - ColumnOperators.icontains](https://docs.sqlalchemy.org/en/20/core/sqlelement.html#sqlalchemy.sql.expression.ColumnOperators.icontains)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [Ruff S608](https://docs.astral.sh/ruff/rules/hardcoded-sql-expression/)

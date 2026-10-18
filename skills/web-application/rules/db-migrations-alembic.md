---
title: Change the Schema with Alembic Migrations
impact: CRITICAL
impactDescription: Reproducible, reviewable schema changes instead of silent drift between environments
tags: [database, alembic, migrations, schema, deployment]
---

# Change the Schema with Alembic Migrations [CRITICAL]

## Description
`Base.metadata.create_all()` only creates tables that are missing. It never adds a column, changes a type, or adds an index to an existing table, so development, CI, and production drift apart without any error. Calling it at startup also makes every worker race to change the schema. Manage the schema with Alembic: autogenerate a revision from model changes, review it, commit it with the code, and run `alembic upgrade head` as a deploy step before the new code starts.

## Bad Example
```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    engine = create_async_engine(settings.database_url.get_secret_value())
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)  # new columns never reach existing tables
    ...


# A one-off fix nobody can reproduce
async with engine.begin() as conn:
    await conn.execute(text("ALTER TABLE users ADD COLUMN last_login_at timestamptz"))
```

## Good Example
```python
# app/models.py: deterministic constraint names so later migrations can alter or drop them
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    metadata = MetaData(
        naming_convention={
            "ix": "ix_%(column_0_label)s",
            "uq": "uq_%(table_name)s_%(column_0_name)s",
            "ck": "ck_%(table_name)s_%(constraint_name)s",
            "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
            "pk": "pk_%(table_name)s",
        }
    )
```

```python
# migrations/env.py, created by `alembic init -t async migrations`
import os

from app.models import Base

# configparser treats % as interpolation, so escape it in URL-encoded passwords
config.set_main_option("sqlalchemy.url", os.environ["APP_DATABASE_URL"].replace("%", "%%"))
target_metadata = Base.metadata
```

```bash
alembic revision --autogenerate -m "add last_login_at to users"
# review migrations/versions/<rev>_add_last_login_at_to_users.py, then commit it
alembic upgrade head   # deploy step, before new app instances start
alembic check          # CI: fails when models have changes without a migration
```

## Notes
- Autogenerate produces a draft. It sees a renamed column as drop plus add (data loss) and misses some type and server default changes; read every revision before committing it.
- Keep migrations compatible with the code that is still running during a deploy: add a nullable column, backfill it, and add `NOT NULL` in a later release (expand, migrate, contract).
- On large PostgreSQL tables, create indexes with `op.create_index(..., postgresql_concurrently=True)` inside `with op.get_context().autocommit_block():` to avoid locking writes.
- Tests may build their schema with `create_all` on a fresh database for speed (see `test-db-rollback`), as long as CI also runs `alembic upgrade head` or `alembic check`.
- Do not let several app replicas run `upgrade` at startup; run it once per deploy as a separate job.

## References
- [Alembic - Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html)
- [Alembic - Auto Generating Migrations](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)
- [Alembic - The Importance of Naming Constraints](https://alembic.sqlalchemy.org/en/latest/naming.html)

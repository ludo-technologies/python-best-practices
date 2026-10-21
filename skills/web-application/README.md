# Python Web Application

FastAPI web application best practices, designed for AI agents and LLMs to build backends that are secure, correct under concurrency, and testable.

The rules target one stack: FastAPI, SQLAlchemy 2.0 (async), Alembic, Pydantic v2 with pydantic-settings, httpx, PyJWT, and pwdlib. Django and Flask are out of scope.

## Overview

This skill provides 22 rules across 8 categories:

| Category | Prefix | Impact | Rules |
|----------|--------|--------|-------|
| Security | `sec-` | CRITICAL | 5 |
| Database | `db-` | CRITICAL | 6 |
| Async I/O | `async-` | CRITICAL | 2 |
| API Design | `api-` | HIGH | 3 |
| Configuration | `config-` | HIGH | 1 |
| App Structure | `app-` | HIGH | 2 |
| Observability | `obs-` | MEDIUM | 1 |
| Testing | `test-` | MEDIUM | 2 |

## Structure

```
skills/web-application/
├── SKILL.md                          # Skill overview with quick reference
├── metadata.json                     # Metadata (version, description)
├── README.md                         # This file
└── rules/
    ├── _sections.md                  # Section definitions
    ├── _template.md                  # Rule template
    ├── sec-password-hashing.md       # Argon2 via pwdlib
    ├── sec-secure-tokens.md          # secrets, hashed tokens, compare_digest
    ├── sec-jwt-validation.md         # Pinned algorithm, required claims
    ├── sec-object-authorization.md   # Owner-scoped queries, 404 for others' data
    ├── sec-cors-origins.md           # Explicit origins, no "*" with credentials
    ├── db-parameterized-queries.md   # Bound parameters, whitelisted identifiers
    ├── db-session-per-request.md     # Engine in lifespan, session dependency
    ├── db-transaction-boundary.md    # One explicit commit per use case
    ├── db-eager-loading.md           # selectinload/joinedload, lazy="raise"
    ├── db-migrations-alembic.md      # Alembic instead of create_all
    ├── db-sqlalchemy-2-style.md      # Mapped[], mapped_column, select()
    ├── async-no-blocking-calls.md    # No blocking I/O or CPU on the event loop
    ├── async-shared-http-client.md   # One httpx.AsyncClient per upstream
    ├── api-separate-schemas.md       # Create/Update/Read models
    ├── api-error-responses.md        # Domain errors mapped by exception handlers
    ├── api-bounded-lists.md          # Max page size, keyset pagination
    ├── config-typed-settings.md      # pydantic-settings, SecretStr, fail fast
    ├── app-factory-lifespan.md       # create_app() and lifespan resources
    ├── app-thin-routes.md            # Services plus Annotated dependencies
    ├── obs-request-id-logging.md     # Request ID middleware and log filter
    ├── test-dependency-overrides.md  # dependency_overrides with ASGITransport
    └── test-db-rollback.md           # Real database, SAVEPOINT per test
```

## Shared Example

All examples share one invoice application. Models (`User`, `Invoice`, `InvoiceLine`) are defined in `db-sqlalchemy-2-style`, `Settings` in `config-typed-settings`, and `create_app` in `app-factory-lifespan`; other rules show only the part they discuss. When a rule adds a field or helper, update the canonical file too.

## Rules

### Security (CRITICAL)
- `sec-password-hashing` - Hash passwords with Argon2
- `sec-secure-tokens` - Generate tokens with secrets and compare in constant time
- `sec-jwt-validation` - Validate JWTs with a pinned algorithm and required claims
- `sec-object-authorization` - Authorize access to every object, not just the endpoint
- `sec-cors-origins` - List allowed CORS origins explicitly

### Database (CRITICAL)
- `db-parameterized-queries` - Never build SQL with string formatting
- `db-session-per-request` - Create the engine once and a session per request
- `db-transaction-boundary` - Commit once per use case, not in helpers or dependency teardown
- `db-eager-loading` - Load relationships explicitly and make lazy loads raise
- `db-migrations-alembic` - Change the schema with Alembic migrations
- `db-sqlalchemy-2-style` - Use SQLAlchemy 2.0 typed models and select()

### Async I/O (CRITICAL)
- `async-no-blocking-calls` - Never block the event loop in async def endpoints
- `async-shared-http-client` - Share one httpx.AsyncClient with explicit timeouts

### API Design (HIGH)
- `api-separate-schemas` - Separate request and response models from ORM models
- `api-error-responses` - Map domain errors to HTTP responses in exception handlers
- `api-bounded-lists` - Bound and paginate every list endpoint

### Configuration (HIGH)
- `config-typed-settings` - Load configuration into a typed settings object at startup

### App Structure (HIGH)
- `app-factory-lifespan` - Build the app in a factory and manage resources in lifespan
- `app-thin-routes` - Keep route functions thin and inject dependencies with Annotated

### Observability (MEDIUM)
- `obs-request-id-logging` - Correlate logs with a request ID

### Testing (MEDIUM)
- `test-dependency-overrides` - Test endpoints with dependency_overrides, not patching
- `test-db-rollback` - Run tests against a real database in a rolled-back transaction

## Related

- [coding-standards / validation-pydantic](../coding-standards/rules/validation-pydantic.md) - Boundary validation that `api-separate-schemas` and `config-typed-settings` build on
- [coding-standards / design-no-global-singleton](../coding-standards/rules/design-no-global-singleton.md) - Same principle behind `app-factory-lifespan` and `db-session-per-request`
- [coding-standards / error-no-silent-exceptions](../coding-standards/rules/error-no-silent-exceptions.md) - Exception handling behind `api-error-responses`
- [testing / mock-boundaries](../testing/rules/mock-boundaries.md) - Mocking only external boundaries, as in `test-dependency-overrides`
- [testing / fixture-factory](../testing/rules/fixture-factory.md) - Factory fixtures used in `test-db-rollback`

## Usage

This skill is automatically applied when working with Alembic files (`alembic.ini`, `**/alembic/**`, `**/migrations/**`) or router modules (`**/routers/**`, `**/api/**`), and whenever the task involves a FastAPI backend, REST endpoints, SQLAlchemy, or authentication. It is not meant for Django or Flask projects.

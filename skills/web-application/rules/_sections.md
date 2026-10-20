# Section Definitions

All examples share one invoice application: models in `db-sqlalchemy-2-style`, `Settings` in `config-typed-settings`, `create_app` in `app-factory-lifespan`. Other rules show only the part they discuss.

## Security (sec)
**Impact:** CRITICAL

Authenticate users safely and authorize every access to their data.
Covers password hashing, random tokens and constant-time comparison, JWT validation, object-level authorization, and CORS.

**Rules:**
- `sec-password-hashing` - Hash passwords with Argon2
- `sec-secure-tokens` - Generate tokens with secrets and compare in constant time
- `sec-jwt-validation` - Validate JWTs with a pinned algorithm and required claims
- `sec-object-authorization` - Authorize access to every object, not just the endpoint
- `sec-cors-origins` - List allowed CORS origins explicitly

## Database (db)
**Impact:** CRITICAL

Query, scope, and evolve the database safely with SQLAlchemy 2.0 and Alembic.
Covers bound parameters, session and engine lifetime, transaction boundaries, eager loading, migrations, and 2.0-style models.

**Rules:**
- `db-parameterized-queries` - Never build SQL with string formatting
- `db-session-per-request` - Create the engine once and a session per request
- `db-transaction-boundary` - Commit once per use case, not in helpers or dependency teardown
- `db-eager-loading` - Load relationships explicitly and make lazy loads raise
- `db-migrations-alembic` - Change the schema with Alembic migrations
- `db-sqlalchemy-2-style` - Use SQLAlchemy 2.0 typed models and select()

## Async I/O (async)
**Impact:** CRITICAL

Keep the event loop free so one request cannot stall the others.
Covers blocking calls in `async def`, thread offloading, and shared outbound HTTP clients.

**Rules:**
- `async-no-blocking-calls` - Never block the event loop in async def endpoints
- `async-shared-http-client` - Share one httpx.AsyncClient with explicit timeouts

## API Design (api)
**Impact:** HIGH

Shape what enters and leaves the API.
Covers request/response models, error mapping and status codes, and pagination.

**Rules:**
- `api-separate-schemas` - Separate request and response models from ORM models
- `api-error-responses` - Map domain errors to HTTP responses in exception handlers
- `api-bounded-lists` - Bound and paginate every list endpoint

## Configuration (config)
**Impact:** HIGH

Fail at startup, not at the first request, when configuration is wrong.
Covers pydantic-settings, `SecretStr`, and injecting settings.

**Rules:**
- `config-typed-settings` - Load configuration into a typed settings object at startup

## App Structure (app)
**Impact:** HIGH

Compose the app explicitly so resources and logic are replaceable and testable.
Covers the app factory, lifespan resources, `Annotated` dependencies, and thin routes.

**Rules:**
- `app-factory-lifespan` - Build the app in a factory and manage resources in lifespan
- `app-thin-routes` - Keep route functions thin and inject dependencies with Annotated

## Observability (obs)
**Impact:** MEDIUM

Make each request traceable through the logs.
Covers request IDs, context variables, and what never to log.

**Rules:**
- `obs-request-id-logging` - Correlate logs with a request ID

## Testing (test)
**Impact:** MEDIUM

Test through the real app and a real database.
Covers `dependency_overrides`, `httpx.ASGITransport`, and per-test transaction rollback.

**Rules:**
- `test-dependency-overrides` - Test endpoints with dependency_overrides, not patching
- `test-db-rollback` - Run tests against a real database in a rolled-back transaction

---
name: web-application
description: FastAPI web application best practices covering authentication and authorization, SQLAlchemy 2.0 async sessions, transactions, eager loading, and Alembic migrations, non-blocking I/O and shared HTTP clients, request/response schemas, error responses, pagination, typed settings, app factory and lifespan, request-ID logging, and endpoint testing. Use when building, reviewing, or refactoring a FastAPI backend or REST API, or code that touches SQLAlchemy, Alembic, authentication, or HTTP endpoints. Not for Django or Flask projects.
paths:
  - "alembic.ini"
  - "**/alembic/**"
  - "**/migrations/**"
  - "**/routers/**"
  - "**/api/**"
---

# Python Web Application

Best practices for FastAPI backends. Designed for AI agents and LLMs to build web applications that are secure, correct under concurrency, and testable.

The rules assume one stack: **FastAPI, SQLAlchemy 2.0 (async), Alembic, Pydantic v2 with pydantic-settings, httpx, PyJWT, and pwdlib**. Examples use the async stack; the same rules apply to sync `def` endpoints with `Session`. Django and Flask are out of scope. Where a rule overlaps a `coding-standards` or `testing` rule, this skill takes precedence for web application code.

All examples share one invoice application. Its models are defined in `db-sqlalchemy-2-style`, its `Settings` in `config-typed-settings`, and its `create_app` in `app-factory-lifespan`; other rules show only the part they discuss. Read the matching rule file in full before writing code in its area.

## Categories

### Security [CRITICAL]
Authenticate users safely and authorize every access to their data.

| Rule | Description |
|------|-------------|
| [sec-password-hashing](rules/sec-password-hashing.md) | Hash passwords with Argon2 |
| [sec-secure-tokens](rules/sec-secure-tokens.md) | Generate tokens with secrets and compare in constant time |
| [sec-jwt-validation](rules/sec-jwt-validation.md) | Validate JWTs with a pinned algorithm and required claims |
| [sec-object-authorization](rules/sec-object-authorization.md) | Authorize access to every object, not just the endpoint |
| [sec-cors-origins](rules/sec-cors-origins.md) | List allowed CORS origins explicitly |

### Database [CRITICAL]
Query, scope, and evolve the database safely with SQLAlchemy 2.0 and Alembic.

| Rule | Description |
|------|-------------|
| [db-parameterized-queries](rules/db-parameterized-queries.md) | Never build SQL with string formatting |
| [db-session-per-request](rules/db-session-per-request.md) | Create the engine once and a session per request |
| [db-transaction-boundary](rules/db-transaction-boundary.md) | Commit once per use case, not in helpers or dependency teardown |
| [db-eager-loading](rules/db-eager-loading.md) | Load relationships explicitly and make lazy loads raise |
| [db-migrations-alembic](rules/db-migrations-alembic.md) | Change the schema with Alembic migrations |
| [db-sqlalchemy-2-style](rules/db-sqlalchemy-2-style.md) | Use SQLAlchemy 2.0 typed models and select() |

### Async I/O [CRITICAL]
Keep the event loop free so one request cannot stall the others.

| Rule | Description |
|------|-------------|
| [async-no-blocking-calls](rules/async-no-blocking-calls.md) | Never block the event loop in async def endpoints |
| [async-shared-http-client](rules/async-shared-http-client.md) | Share one httpx.AsyncClient with explicit timeouts |

### API Design [HIGH]
Shape what enters and leaves the API.

| Rule | Description |
|------|-------------|
| [api-separate-schemas](rules/api-separate-schemas.md) | Separate request and response models from ORM models |
| [api-error-responses](rules/api-error-responses.md) | Map domain errors to HTTP responses in exception handlers |
| [api-bounded-lists](rules/api-bounded-lists.md) | Bound and paginate every list endpoint |

### Configuration [HIGH]
Fail at startup, not at the first request, when configuration is wrong.

| Rule | Description |
|------|-------------|
| [config-typed-settings](rules/config-typed-settings.md) | Load configuration into a typed settings object at startup |

### App Structure [HIGH]
Compose the app explicitly so resources and logic are replaceable and testable.

| Rule | Description |
|------|-------------|
| [app-factory-lifespan](rules/app-factory-lifespan.md) | Build the app in a factory and manage resources in lifespan |
| [app-thin-routes](rules/app-thin-routes.md) | Keep route functions thin and inject dependencies with Annotated |

### Observability [MEDIUM]
Make each request traceable through the logs.

| Rule | Description |
|------|-------------|
| [obs-request-id-logging](rules/obs-request-id-logging.md) | Correlate logs with a request ID |

### Testing [MEDIUM]
Test through the real app and a real database.

| Rule | Description |
|------|-------------|
| [test-dependency-overrides](rules/test-dependency-overrides.md) | Test endpoints with dependency_overrides, not patching |
| [test-db-rollback](rules/test-db-rollback.md) | Run tests against a real database in a rolled-back transaction |

## Quick Reference

### App Factory and Dependencies
```python
def create_app(settings: Settings) -> FastAPI:
    @asynccontextmanager
    async def lifespan(app: FastAPI) -> AsyncIterator[None]:
        engine = create_async_engine(
            settings.database_url.get_secret_value(), pool_pre_ping=True
        )
        try:
            async with httpx.AsyncClient(
                base_url=settings.payments_base_url,
                timeout=httpx.Timeout(10.0, connect=3.0),
            ) as payments_client:
                app.state.sessionmaker = async_sessionmaker(
                    engine, expire_on_commit=False
                )
                app.state.payments_client = payments_client
                yield
        finally:
            await engine.dispose()

    app = FastAPI(lifespan=lifespan)
    app.state.settings = settings
    add_exception_handlers(app)
    app.include_router(invoices.router)
    return app


async def get_session(request: Request) -> AsyncIterator[AsyncSession]:
    async with request.app.state.sessionmaker() as session:
        yield session  # no commit here: teardown runs after the response is sent


SessionDep = Annotated[AsyncSession, Depends(get_session)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

### Authorized, Eager-Loaded Query
```python
router = APIRouter(prefix="/invoices", dependencies=[Depends(get_current_user)])


@router.get("/{invoice_id}")
async def read_invoice(
    invoice_id: int, user: CurrentUser, session: SessionDep
) -> InvoiceRead:
    invoice = await session.scalar(
        select(Invoice)
        .where(Invoice.id == invoice_id, Invoice.owner_id == user.id)  # scoped by owner
        .options(selectinload(Invoice.lines))  # relationships declared lazy="raise"
    )
    if invoice is None:
        raise NotFoundError("Invoice")  # mapped to 404 by an exception handler
    return InvoiceRead.model_validate(invoice)
```

### Passwords and Tokens
```python
password_hash = PasswordHash.recommended()  # Argon2id via pwdlib
hashed = await asyncio.to_thread(password_hash.hash, password)

claims = jwt.decode(
    token,
    settings.jwt_secret.get_secret_value(),
    algorithms=["HS256"],
    audience=settings.jwt_audience,
    issuer=settings.jwt_issuer,
    options={"require": ["exp", "iat", "sub", "iss", "aud"]},
)

reset_token = secrets.token_urlsafe(32)  # store only its SHA-256 hash
hmac.compare_digest(expected_signature.encode(), presented_signature.encode())
```

### Settings and CORS
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_")

    database_url: SecretStr
    jwt_secret: SecretStr = Field(min_length=32)
    jwt_issuer: str
    jwt_audience: str
    access_token_ttl_minutes: int = Field(default=15, gt=0, le=60)
    cors_origins: list[str] = []
    payments_base_url: str
    payments_api_key: SecretStr


app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,  # never ["*"] with allow_credentials=True
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

### Testing
```python
@pytest.fixture
async def session(engine: AsyncEngine) -> AsyncIterator[AsyncSession]:
    async with engine.connect() as conn:
        transaction = await conn.begin()
        session = AsyncSession(
            bind=conn, join_transaction_mode="create_savepoint", expire_on_commit=False
        )
        try:
            yield session
        finally:
            await session.close()
            await transaction.rollback()


@pytest.fixture
async def client(
    session: AsyncSession, settings: Settings
) -> AsyncIterator[httpx.AsyncClient]:
    app = create_app(settings)

    async def override_session() -> AsyncIterator[AsyncSession]:
        yield session

    app.dependency_overrides[get_session] = override_session
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as client:
        yield client
```

## See Also

- [coding-standards](../coding-standards/SKILL.md) - General Python rules; `validation-pydantic`, `design-dependency-injection`, `design-no-global-singleton`, and `error-no-silent-exceptions` underlie several rules here
- [testing](../testing/SKILL.md) - pytest structure, fixtures, and mocking that endpoint tests still follow
- [tooling](../tooling/SKILL.md) - ruff, mypy, and pytest configuration; ruff's `S` rules catch several issues covered here

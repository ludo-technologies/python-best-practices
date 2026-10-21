---
title: Map Domain Errors to HTTP Responses in Exception Handlers
impact: HIGH
impactDescription: Correct status codes and one error shape, no leaked internals
tags: [api, errors, exception-handlers, status-codes, fastapi]
---

# Map Domain Errors to HTTP Responses in Exception Handlers [HIGH]

## Description
Services should raise domain exceptions (`NotFoundError`, `ConflictError`) and know nothing about HTTP; one place maps them to status codes. Common anti-patterns: returning `200` with `{"error": ...}`, so clients and monitoring see success; catching `Exception` and returning `str(exc)`, which leaks SQL and file paths; and raising `HTTPException` deep inside services. Register exception handlers for domain errors, use precise status codes, and let unexpected exceptions propagate to a logged 500.

## Bad Example
```python
router = APIRouter(prefix="/users")


@router.post("")
async def create_user(body: UserCreate, session: SessionDep) -> dict[str, Any]:
    try:
        user = await register_user(session, body)
        return {"success": True, "user": UserRead.model_validate(user)}
    except Exception as exc:
        return {
            "success": False,
            "error": str(exc),
        }  # 200 OK carrying the database error
```

## Good Example
```python
# app/errors.py: domain errors with client-safe messages
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse


class NotFoundError(Exception):
    def __init__(self, resource: str) -> None:
        super().__init__(f"{resource} not found")


class ConflictError(Exception):
    pass


def add_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(NotFoundError)
    async def handle_not_found(request: Request, exc: NotFoundError) -> JSONResponse:
        return JSONResponse(
            status_code=status.HTTP_404_NOT_FOUND, content={"detail": str(exc)}
        )

    @app.exception_handler(ConflictError)
    async def handle_conflict(request: Request, exc: ConflictError) -> JSONResponse:
        return JSONResponse(
            status_code=status.HTTP_409_CONFLICT, content={"detail": str(exc)}
        )


# app/services/users.py: no FastAPI imports
async def register_user(session: AsyncSession, data: UserCreate) -> User:
    hashed = await hash_password(data.password.get_secret_value())
    user = User(email=data.email, password_hash=hashed)
    session.add(user)
    try:
        await session.commit()
    except (
        IntegrityError
    ) as exc:  # uq_users_email is the only constraint this can violate
        await session.rollback()
        raise ConflictError("Email already registered") from exc
    return user


# app/routers/users.py
router = APIRouter(prefix="/users")


@router.post("", status_code=status.HTTP_201_CREATED)
async def create_user(body: UserCreate, session: SessionDep) -> UserRead:
    return UserRead.model_validate(await register_user(session, body))
```

## Notes
- Keep FastAPI's `{"detail": ...}` shape for every error so domain errors, `HTTPException`, and 422 validation errors look alike to clients. If you adopt RFC 9457 Problem Details, also override the handlers for `HTTPException` and `RequestValidationError`.
- `HTTPException` belongs in the HTTP layer: routes and dependencies such as `get_current_user` raising 401.
- Messages of domain exceptions are shown to clients; write them for clients and keep internal detail in the exception chain and logs.
- Do not add a catch-all handler that returns `str(exc)`. If you handle `Exception` at all, log it with `logger.exception` and return a generic message (see `coding-standards/error-no-silent-exceptions`). Never run production with `debug=True`, which returns tracebacks.
- Catch `IntegrityError` at the commit instead of checking for duplicates first: two concurrent requests can both pass the check.
- A 409 on sign-up tells the caller that the email has an account. That is usually an accepted trade-off for usability; where account existence must stay private, return the same response for new and existing emails and notify the owner by email instead (compare `sec-password-hashing`).
- Status codes: 201 created, 204 no content, 401 not authenticated (with `WWW-Authenticate`), 403 authenticated but not allowed, 404 missing or not visible (see `sec-object-authorization`), 409 conflict, 422 invalid input, 429 rate limited, 502/503 upstream failure.

## References
- [FastAPI - Handling Errors](https://fastapi.tiangolo.com/tutorial/handling-errors/)
- [RFC 9457 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)

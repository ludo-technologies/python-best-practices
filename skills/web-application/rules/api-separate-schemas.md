---
title: Separate Request and Response Models from ORM Models
impact: HIGH
impactDescription: No leaked password hashes, no mass assignment of is_admin
tags: [api, pydantic, schemas, mass-assignment, response-model]
---

# Separate Request and Response Models from ORM Models [HIGH]

## Description
One model for input, output, and storage couples the API to the tables. Returning an ORM object, or a schema that mirrors it, leaks fields such as `password_hash` or `is_admin`; accepting the same schema as input lets clients set fields they must never control (mass assignment: `{"is_admin": true}`). Define Pydantic models per direction: `XCreate` and `XUpdate` hold only client-writable fields and forbid unknown keys, and `XRead` lists exactly what clients may see.

## Bad Example
```python
router = APIRouter(prefix="/users")


class UserSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int | None = None
    email: str
    password_hash: str | None = None
    is_admin: bool = False


@router.post("")
async def create_user(body: UserSchema, session: SessionDep) -> UserSchema:
    user = User(**body.model_dump())  # client can send is_admin=true
    session.add(user)
    await session.commit()
    return UserSchema.model_validate(user)  # response includes password_hash
```

## Good Example
```python
from pydantic import BaseModel, ConfigDict, EmailStr, Field, SecretStr

router = APIRouter(prefix="/users")


class UserCreate(BaseModel):
    model_config = ConfigDict(extra="forbid")  # unknown keys such as is_admin -> 422

    email: EmailStr
    password: SecretStr = Field(min_length=12, max_length=128)


class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    display_name: str | None = Field(default=None, max_length=100)


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr
    display_name: str | None


@router.post("", status_code=status.HTTP_201_CREATED)
async def create_user(body: UserCreate, session: SessionDep) -> UserRead:
    # register_user hashes the password and never sets is_admin
    user = await register_user(session, body)
    return UserRead.model_validate(user)


@router.patch("/me")
async def update_me(
    body: UserUpdate, user: CurrentUser, session: SessionDep
) -> UserRead:
    for field, value in body.model_dump(exclude_unset=True).items():
        setattr(user, field, value)  # only fields the client actually sent
    await session.commit()
    return UserRead.model_validate(user)
```

## Notes
- Pydantic ignores unknown keys by default, which already drops `is_admin`; `extra="forbid"` turns it into a 422 so client typos and probing are visible instead of silently ignored.
- `model_dump(exclude_unset=True)` separates "not sent" from "sent as null" in PATCH handlers.
- Name models by purpose (`UserCreate`, `UserUpdate`, `UserRead`) and add a separate `UserAdminRead` for privileged views instead of conditionally hiding fields.
- Fields that need a workflow, such as changing an email address that must be re-verified, get their own endpoint instead of a field in `UserUpdate`.
- `SecretStr` keeps passwords out of `repr`, logs, and tracebacks (see `sec-password-hashing`).
- This complements `coding-standards/validation-pydantic`: validate at the boundary, and also shape what leaves it.

## References
- [FastAPI - Response Model](https://fastapi.tiangolo.com/tutorial/response-model/)
- [FastAPI - Extra Models](https://fastapi.tiangolo.com/tutorial/extra-models/)
- [OWASP API3:2023 - Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)

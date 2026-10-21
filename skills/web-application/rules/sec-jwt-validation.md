---
title: Validate JWTs with a Pinned Algorithm and Required Claims
impact: CRITICAL
impactDescription: Forged, expired, and wrong-audience tokens are rejected
tags: [security, jwt, pyjwt, authentication, oauth2]
---

# Validate JWTs with a Pinned Algorithm and Required Claims [CRITICAL]

## Description
`jwt.decode` checks only what you ask it to. A token without `exp` stays valid forever, a token issued for another service is accepted unless the audience is checked, and taking `algorithms` from the token header hands the choice to the attacker (`alg=none`, RS256/HS256 key confusion). Pin the algorithm, require `exp`, `iat`, `sub`, `iss`, and `aud`, keep access tokens short-lived, and turn every `InvalidTokenError` into a 401.

## Bad Example
```python
import jwt

SECRET_KEY = "secret"  # hard-coded and guessable


def decode_token(token: str) -> int:
    header = jwt.get_unverified_header(token)
    alg = header["alg"]  # the attacker picks the algorithm
    claims = jwt.decode(token, SECRET_KEY, algorithms=[alg])
    return int(claims["sub"])  # tokens without exp, aud, or iss are accepted


def peek_user_id(token: str) -> int:
    # Signature not checked at all
    return int(jwt.decode(token, options={"verify_signature": False})["sub"])
```

## Good Example
```python
from datetime import datetime, timedelta
from typing import Annotated

import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

ALGORITHM = "HS256"
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")


def create_access_token(user_id: int, settings: Settings, now: datetime) -> str:
    claims = {
        "sub": str(user_id),
        "iss": settings.jwt_issuer,
        "aud": settings.jwt_audience,
        "iat": now,
        "exp": now + timedelta(minutes=settings.access_token_ttl_minutes),
    }
    return jwt.encode(
        claims, settings.jwt_secret.get_secret_value(), algorithm=ALGORITHM
    )


def decode_access_token(token: str, settings: Settings) -> int:
    claims = jwt.decode(
        token,
        settings.jwt_secret.get_secret_value(),
        algorithms=[ALGORITHM],
        audience=settings.jwt_audience,
        issuer=settings.jwt_issuer,
        options={"require": ["exp", "iat", "sub", "iss", "aud"]},
    )
    return int(claims["sub"])


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    session: SessionDep,
    settings: SettingsDep,
) -> User:
    unauthorized = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        user_id = decode_access_token(token, settings)
    except (jwt.InvalidTokenError, ValueError) as exc:
        # expired, bad signature, wrong aud/iss, missing claim, non-numeric sub
        raise unauthorized from exc
    user = await session.get(User, user_id)
    if user is None or not user.is_active:  # deleted or deactivated since issue
        raise unauthorized
    return user


CurrentUser = Annotated[User, Depends(get_current_user)]
```

## Notes
- Load the HS256 secret from settings as a `SecretStr` with `min_length=32` (see `config-typed-settings`); generate it with `python -c "import secrets; print(secrets.token_urlsafe(32))"`.
- When other services verify your tokens, use RS256 or ES256 so they only hold the public key; pin `algorithms=["RS256"]` and fetch keys with `jwt.PyJWKClient` from the issuer's JWKS endpoint.
- JWTs cannot be revoked before `exp`. Keep access tokens short (5-15 minutes), load the user and check `is_active` on each request as above, and use refresh tokens or server-side sessions for long logins.
- PyJWT requires `sub` to be a string: encode `str(user_id)` and convert back on decode.
- Use PyJWT (`pip install pyjwt`); the FastAPI docs moved away from `python-jose`.
- If tokens travel in cookies instead of the `Authorization` header, set `HttpOnly`, `Secure`, and `SameSite`, and protect state-changing requests against CSRF.

## References
- [PyJWT - Usage Examples](https://pyjwt.readthedocs.io/en/stable/usage.html)
- [RFC 8725 - JSON Web Token Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [FastAPI - OAuth2 with Password (and hashing), Bearer with JWT tokens](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)

---
title: Hash Passwords with Argon2
impact: CRITICAL
impactDescription: A leaked users table cannot be cracked offline
tags: [security, passwords, argon2, pwdlib, authentication]
---

# Hash Passwords with Argon2 [CRITICAL]

## Description
Store passwords only as hashes from a slow, salted, memory-hard algorithm. `hashlib.sha256` and `md5` are built to be fast, so a leaked table is brute-forced at billions of guesses per second, and unsalted digests repeat for equal passwords. Use Argon2id through `pwdlib`, verify with the library instead of `==`, rehash when parameters change, and run the CPU-heavy work in a thread from `async def` code.

## Bad Example
```python
import hashlib


async def register(session: AsyncSession, email: str, password: str) -> User:
    digest = hashlib.sha256(password.encode()).hexdigest()  # fast and unsalted
    user = User(email=email, password_hash=digest)
    session.add(user)
    await session.commit()
    return user


async def authenticate(session: AsyncSession, email: str, password: str) -> User | None:
    user = await session.scalar(select(User).where(User.email == email))
    if user is None:
        return None  # returns instantly: response time reveals which emails exist
    if user.password_hash != hashlib.sha256(password.encode()).hexdigest():
        return None
    return user
```

## Good Example
```python
import asyncio

from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()  # Argon2id with current parameters
DUMMY_HASH = password_hash.hash("dummy-password-for-timing")


async def hash_password(password: str) -> str:
    # Argon2 takes tens of milliseconds of CPU; keep it off the event loop
    return await asyncio.to_thread(password_hash.hash, password)


async def authenticate(session: AsyncSession, email: str, password: str) -> User | None:
    user = await session.scalar(select(User).where(User.email == email))
    if user is None:
        # Spend the same time as a real check so timing does not reveal valid emails
        await asyncio.to_thread(password_hash.verify, password, DUMMY_HASH)
        return None
    valid, updated_hash = await asyncio.to_thread(
        password_hash.verify_and_update, password, user.password_hash
    )
    if not valid:
        return None
    if updated_hash is not None:  # stored hash used outdated parameters
        user.password_hash = updated_hash
        await session.commit()
    return user
```

## Notes
- Install with `pip install "pwdlib[argon2]"`. `passlib` is unmaintained; do not add it to new projects. bcrypt (`pwdlib[bcrypt]`) is acceptable for existing hashes, but it only accepts 72 bytes of input.
- Return the same error for "unknown email" and "wrong password" (`401 Incorrect email or password`) so responses do not reveal which accounts exist.
- Cap password length in the request model (`Field(max_length=128)`) so oversized input cannot be used to burn hashing CPU.
- Accept passwords as `SecretStr` so they stay out of `repr`, logs, and tracebacks; call `get_secret_value()` only where you hash or verify.
- Never log passwords or hashes, and never include `password_hash` in a response model (see `api-separate-schemas`).
- In a plain `def` endpoint FastAPI already runs the function in a thread pool, so call `password_hash.hash()` directly there (see `async-no-blocking-calls`).

## References
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [pwdlib Documentation](https://frankie567.github.io/pwdlib/)
- [FastAPI - OAuth2 with Password (and hashing), Bearer with JWT tokens](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)

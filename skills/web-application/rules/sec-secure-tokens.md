---
title: Generate Tokens with the secrets Module and Compare in Constant Time
impact: CRITICAL
impactDescription: Unguessable tokens, no plaintext at rest, no timing leaks
tags: [security, tokens, secrets, hmac, api-keys]
---

# Generate Tokens with the secrets Module and Compare in Constant Time [CRITICAL]

## Description
Reset tokens, API keys, invite codes, and webhook secrets must be unpredictable. `random` is a Mersenne Twister whose future output can be reconstructed from observed values; use `secrets`, which reads the OS CSPRNG. Store only a hash of each issued token, give it an expiry, and compare presented secrets with `hmac.compare_digest`, because `==` returns at the first differing byte and leaks how much of a guess was right.

## Bad Example
```python
import hashlib
import hmac
import random
import string


def create_reset_token(user: User) -> str:
    token = "".join(random.choices(string.ascii_letters + string.digits, k=20))
    user.reset_token = token  # predictable, stored in plaintext, never expires
    return token


def verify_webhook(body: bytes, signature: str, secret: bytes) -> bool:
    expected = hmac.new(secret, body, hashlib.sha256).hexdigest()
    return expected == signature  # timing leak
```

## Good Example
```python
import hashlib
import hmac
import secrets
from datetime import datetime, timedelta


def hash_token(token: str) -> str:
    # High-entropy random tokens need a one-way hash, not a slow password hash
    return hashlib.sha256(token.encode()).hexdigest()


def create_reset_token(user: User, now: datetime) -> str:
    token = secrets.token_urlsafe(32)  # 256 bits from the OS CSPRNG
    user.reset_token_hash = hash_token(token)
    user.reset_token_expires_at = now + timedelta(minutes=30)
    return token  # send to the user; never store it


async def consume_reset_token(
    session: AsyncSession, token: str, now: datetime
) -> User | None:
    # One atomic UPDATE: concurrent requests with the same token cannot both succeed
    return await session.scalar(
        update(User)
        .where(
            User.reset_token_hash == hash_token(token),
            User.reset_token_expires_at > now,
        )
        .values(reset_token_hash=None, reset_token_expires_at=None)
        .returning(User)
    )  # the caller commits


def verify_webhook(body: bytes, signature: str, secret: bytes) -> bool:
    expected = hmac.new(secret, body, hashlib.sha256).hexdigest()
    # Compare bytes: str arguments with non-ASCII characters raise TypeError
    return hmac.compare_digest(expected.encode(), signature.encode())
```

## Notes
- Looking a token up by its hash needs no `compare_digest`: the attacker cannot predict the hash being matched. Use `compare_digest` whenever Python compares a presented secret with an expected one (signatures, static API keys, CSRF tokens).
- Use at least 32 bytes: `secrets.token_urlsafe(32)` for URLs and headers, `secrets.token_hex(32)` where only hex is allowed.
- Make reset and invite tokens single-use and short-lived, and invalidate outstanding reset tokens when the password changes. Consume a token with one conditional `UPDATE ... RETURNING` (or `SELECT ... FOR UPDATE`); a plain select followed by an assignment lets two concurrent requests use the same token.
- `compare_digest` accepts `str` only when both values are ASCII, and attacker-supplied headers need not be; encode both sides to bytes.
- Ruff `S311` flags `random` used where security matters.
- Passwords are the exception to "SHA-256 is enough": they are low-entropy and need Argon2 (see `sec-password-hashing`).

## References
- [Python Documentation - secrets](https://docs.python.org/3/library/secrets.html)
- [Python Documentation - hmac.compare_digest](https://docs.python.org/3/library/hmac.html#hmac.compare_digest)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Ruff S311](https://docs.astral.sh/ruff/rules/suspicious-non-cryptographic-random-usage/)

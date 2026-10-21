---
title: List Allowed CORS Origins Explicitly
impact: CRITICAL
impactDescription: Other websites cannot read a user's authenticated responses
tags: [security, cors, middleware, starlette, cookies]
---

# List Allowed CORS Origins Explicitly [CRITICAL]

## Description
CORS tells browsers which other sites may read your API's responses. `allow_origins=["*"]` with `allow_credentials=True` does not mean "public API": Starlette echoes the caller's `Origin` with `Access-Control-Allow-Credentials: true`, so script on any website can send the user's cookies and read the result. Load the exact list of trusted origins from settings, default to none, and allow only the methods and headers the frontend uses.

## Bad Example
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,  # with "*", Starlette reflects any Origin
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Good Example
```python
# Settings.cors_origins: list[str] = []  (see config-typed-settings)
# APP_CORS_ORIGINS='["https://app.example.com"]'
from fastapi.middleware.cors import CORSMiddleware

# inside create_app(settings) (see app-factory-lifespan)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,  # empty means no cross-origin access
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

## Notes
- Origins match exactly: `https://app.example.com`, `https://www.example.com`, and `http://app.example.com` are three different origins. Do not add a trailing slash.
- `allow_origin_regex` is matched with `fullmatch`, so escape dots: `r"https://.*example.com"` also matches `https://evilexample.com`; use `r"https://[a-z0-9-]+\.example\.com"`.
- A public, cookie-less API may use `allow_origins=["*"]` with `allow_credentials=False`.
- CORS is neither authentication nor CSRF protection. It only controls what browsers expose to scripts; curl and servers ignore it. Cookie-based auth still needs `SameSite` cookies or CSRF tokens.
- When the frontend is served from the same origin (for example behind one reverse proxy), you need no `CORSMiddleware` at all.

## References
- [FastAPI - CORS](https://fastapi.tiangolo.com/tutorial/cors/)
- [Starlette - CORSMiddleware](https://www.starlette.io/middleware/#corsmiddleware)
- [MDN - Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

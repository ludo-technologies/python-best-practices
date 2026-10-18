---
title: List Allowed CORS Origins Explicitly
impact: CRITICAL
impactDescription: Prevents any website a user visits from reading their authenticated API responses
tags: [security, cors, middleware, starlette, cookies]
---

# List Allowed CORS Origins Explicitly [CRITICAL]

## Description
CORS tells browsers which other sites may read your API's responses. The most common generated configuration, `allow_origins=["*"]` together with `allow_credentials=True`, does not mean "public API": Starlette's `CORSMiddleware` answers such requests by echoing the caller's `Origin` back with `Access-Control-Allow-Credentials: true`, so script on any website can make requests with the user's cookies and read the results. Load the exact list of trusted origins from settings, default to none, and allow only the methods and headers the frontend uses.

## Bad Example
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,  # with "*", Starlette reflects any Origin and allows credentials
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Good Example
```python
from fastapi.middleware.cors import CORSMiddleware


class Settings(BaseSettings):
    cors_origins: list[str] = []  # APP_CORS_ORIGINS='["https://app.example.com"]'


def create_app(settings: Settings) -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,  # empty means no cross-origin access
        allow_credentials=True,
        allow_methods=["GET", "POST", "PATCH", "DELETE"],
        allow_headers=["Authorization", "Content-Type"],
    )
    return app
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

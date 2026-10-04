# ADR-0003: Authentication tokens & API routing
- Status: **Proposed — needs approval**
- Date: 2026-10-04

## Context
The brief requires an auth strategy (registration, login, logout, token lifecycle, refresh) for a Next.js frontend and a DRF backend, without specifying the mechanism.

## Decision (proposed)
- **JWT via `djangorestframework-simplejwt`** (single added backend dependency).
- **Access token:** short-lived (15 min), returned in the JSON body, held **in memory only** by the frontend, sent as `Authorization: Bearer`.
- **Refresh token:** 7 days, issued as an **`HttpOnly; Secure; SameSite=Lax` cookie** scoped to `/api/v1/auth/`; **rotated on every refresh**, old one blacklisted (SimpleJWT blacklist app).
- **Logout** blacklists the refresh token and clears the cookie.
- **Routing:** same site — Nginx serves the frontend at `/` and the API at `/api/v1/` on one origin (staging/prod), so no wide-open CORS and cookies work with SameSite=Lax. Dev uses a Next.js rewrite/proxy or an explicit CORS allow-list.
- CSRF: the refresh/logout cookie endpoints require a custom header (`X-Requested-With`) and/or Origin check; bearer-authenticated endpoints are not CSRF-exposed.
- **Amendment (TASK-001-AMENDMENT):** applies to *customer* authentication. The CMS uses `/api/v1/admin/auth/` with audience `cms` and shorter lifetimes ([ADR-0009](ADR-0009-cms-auth-rbac-audit.md)). "Same origin" holds **per host**: the storefront host and the CMS host (e.g. `admin.<domain>`) each route `/api/v1/` to the backend via Nginx. Refresh cookies are host-only and path-scoped, so storefront and CMS sessions are isolated.

## Alternatives Considered
1. Django session cookies + CSRF: simpler, secure, but ties clients to cookie-session semantics and is awkward if non-browser clients are expected.
2. Tokens in localStorage: rejected — XSS exposure.
3. Third-party auth (Auth0 etc.): extra dependency/cost; not requested.

## Consequences
Page reloads silently call refresh to recover the access token. Full flow in [security.md](../../security/security.md). If rejected, option 1 is the fallback; contract `securitySchemes` would change accordingly (no code exists yet).

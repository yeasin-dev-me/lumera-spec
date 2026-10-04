# Security Guidelines

## Principles
Never trust the client. Validate and authorize server-side. Least privilege. No secrets in source control. Fail closed.

## Authentication Design
*(Mechanism pending approval of [ADR-0003](../architecture/decisions/ADR-0003-auth-token-strategy.md). Nothing is implemented.)*

| Aspect | Design |
|---|---|
| Registration | `POST /auth/register/` — email (unique, case-insensitive), password validated by Django validators (min 10 chars, not common/numeric-only), name. No auto-privilege. Email verification: open question (recommended before first order; decide in the auth task). |
| Login | `POST /auth/login/` — email + password → access token (body) + refresh cookie. Generic failure message (no account enumeration). Throttled. |
| Access token | JWT, 15 min, in-memory on the client, `Authorization: Bearer`. |
| Refresh | `POST /auth/refresh/` using HttpOnly cookie; refresh token **rotates** each use; reuse of a blacklisted token revokes the family/forces re-login. Lifetime 7 days (absolute). |
| Logout | `POST /auth/logout/` — blacklist refresh token, clear cookie. Client discards the access token. |
| Password change/reset | Invalidates all refresh tokens of the user. Reset via emailed single-use, time-limited token. |
| Protected endpoints | DRF default `IsAuthenticated`; public endpoints explicitly `AllowAny`. |
| Passwords | Django's PBKDF2 (or Argon2 if approved); never logged or returned. |
| Admin/CMS access | **Django Admin is not used.** CMS users authenticate via `/api/v1/admin/auth/` ([ADR-0009](../architecture/decisions/ADR-0009-cms-auth-rbac-audit.md)): `is_staff` accounts only, token audience `cms` (customer tokens rejected on `/admin/` and vice versa), access 10 min / refresh 8 h absolute, stricter throttling and lockout. MFA (TOTP) recommended for all staff (open question). CMS host optionally behind Cloudflare Access / IP allow-list. |

## Roles & Permissions (RBAC)
Authorization is **permission-based**, enforced by the backend on every request; roles are data (named permission sets), permissions are code-defined `resource.action` strings. Catalog and rationale: [cms-architecture](../architecture/cms-architecture.md#5-cms-authorization).

| Subject | Definition | Can |
|---|---|---|
| Anonymous | no token | `/storefront/` read endpoints; register; login |
| Customer | authenticated customer token, `is_staff=false` | Own data under `/customer/`; no `/admin/` access |
| Staff (CMS user) | `is_staff=true` + assigned role(s) | Only the union of their roles' permissions under `/admin/` |
| `super_admin` | all permissions (the only seeded role) | Everything, incl. `staff.manage`, `integrations.manage`, `audit.read` |

Candidate role presets (**create only if the business needs them**): Admin, Manager, Operator, Content Manager, Order Manager — e.g. Content Manager = `content.*`, `products.read`, `categories.read`; Order Manager = `orders.*`, `customers.read`, `payments.read`.

- Every admin view declares its required permission(s); default deny; unknown permission strings fail closed.
- Permissions are resolved from the DB, never from client input or token claims (the token carries user id and audience only).
- Frontend/CMS route protection and hidden buttons are UX only.
- Object-level rules: customers see only their own resources (others → 404); staff scoping beyond permissions only if a requirement appears.
- Privilege escalation guards: only `staff.manage` can change roles; nobody can grant permissions they don't hold; the last `super_admin` cannot be removed.

## CMS-Specific Security
- **No secrets in CMS browser code**; no `NEXT_PUBLIC_*` secrets; the CMS has no provider SDKs and no third-party tracking scripts.
- **Secret configuration:** write-only through the API (`is_set` flag only), encrypted at rest with an application key from the environment, never logged, never in audit before/after values. Public config is a strict whitelist ([ADR-0010](../architecture/decisions/ADR-0010-integrations-secrets-tracking.md)). Infra secrets (DB credentials, Django secret key, encryption key) and, by default, payment/webhook secrets are environment-only.
- **Payments:** CMS → Backend → Provider only. Webhooks at `/internal/` verify provider signatures; idempotent.
- **Customer data minimization:** `customers.read` returns only what BR-23 allows; masked fields reveal via audited action; individual carts not exposed.
- **Audit:** append-only log of admin writes ([cms-architecture §9](../architecture/cms-architecture.md#9-audit-requirements)); entries redact secrets/PII; `audit.read` required; no update/delete API.
- **CMS hardening:** `noindex`, `frame-ancestors 'none'`, strict CSP, CSRF/Origin checks on cookie endpoints, optimistic concurrency (`If-Match`) against lost updates, confirmation for destructive actions, upload validation (type/size, content sniffing; SVG/HTML disallowed or sanitized), rich text sanitized server-side.
- **Tracking:** client-sent events are untrusted; server sends purchase events from verified order data; personal identifiers hashed server-side; consent respected.
- **Rate limits** on admin auth, exports, and bulk actions.

## Server-Side Validation Rules
- Recompute price, totals, discounts, shipping, stock availability, order status transitions, and ownership on the server.
- Serializers whitelist writable fields; reject/ignore server-owned fields.
- Payments verified via provider (signature-checked webhook / server API), idempotent.
- Quantity bounds, string lengths, file type/size limits for uploads (images only, content-type sniffed, size cap, re-encoded if feasible).

## Application Hardening
- Django `DEBUG=False` outside dev; `ALLOWED_HOSTS` explicit; `SECURE_*` settings (HSTS, SSL redirect, secure cookies) in staging/prod.
- CORS: explicit allow-list only, never `*` with credentials.
- CSRF protections for cookie-authenticated endpoints (refresh/logout) as per ADR-0003.
- Security headers (CSP, X-Content-Type-Options, Referrer-Policy, frame protections) via Nginx and/or Next.js config.
- Rate limiting: DRF throttles (auth, review creation, etc.) + Cloudflare rules.
- ORM only; no raw SQL with string formatting. No `dangerouslySetInnerHTML` with user content in the frontend.
- Error responses never leak stack traces, SQL, or internals.
- Logging: no passwords, tokens, card data, or full PII. Audit log for staff actions on orders/products (design in admin task).
- Dependencies: pinned, lockfiles committed, automated vulnerability scanning in CI (`pip-audit`, `npm audit`) — advisory initially.
- Uploads served from a path that cannot execute code.

## Secrets & Configuration
- No real secrets in git (including examples, tests, docs, Docker images). Commit `.env.example` with placeholders only.
- Secrets are injected per environment (CI secret store / server env / secrets manager). Different secrets per environment; rotation process documented by DevOps.
- Pre-commit/CI secret scanning (e.g. gitleaks) recommended.
- Leaked secret = rotate immediately, then record in `lessons.md`.

## Data Protection
- TLS everywhere (Cloudflare Full (strict) + Nginx cert). Database not publicly exposed.
- Card data never stored or processed by Lumera.
- Backups encrypted; restore tested. Personal data minimization; retention/erasure policy TBD with product owner.

## Review Checklist (use in PRs touching security-relevant code)
Authz checked? Ownership enforced? Client value trusted anywhere? Input validated? Secrets absent? Errors safe? Tests for unauthorized and tampered input?

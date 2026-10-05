# API Architecture

Normative conventions for all endpoints. The machine-readable contract is [api-contract.yaml](../contracts/api-contract.yaml).

## Basics
- Base path: `/api/v1/`. JSON only (`application/json`; multipart for image upload). UTF-8. Timestamps ISO-8601 UTC.
- Versioning: URL major version. Additive changes (new optional fields/endpoints) are non-breaking; removing/renaming/changing semantics requires `/api/v2/` or an approved deprecation plan.
- Contract-first: any endpoint used by more than one domain is added to the contract before implementation.

## Namespaces (audience separation — [ADR-0008](decisions/ADR-0008-api-namespaces.md), Proposed)
| Namespace | Audience | Auth |
|---|---|---|
| `/api/v1/storefront/` | Public catalog/content/config, read-mostly | none |
| `/api/v1/auth/` | Customer register/login/refresh/logout | credentials |
| `/api/v1/customer/` | Logged-in customer's own data (profile, addresses, cart, checkout, orders, review creation) | customer token |
| `/api/v1/admin/` | CMS operations (incl. `/admin/auth/`) | CMS token + per-view permission |
| `/api/v1/internal/` | Machine callers: payment webhooks, jobs | signature/shared secret; not browser-facing |
| `/api/v1/health/` | Monitoring | none |

- Admin endpoints are never exposed without authentication + permission; admin serializers are never reused in public/customer views.
- Each namespace is a separate URL module and a separate OpenAPI tag.
- **Django Admin is not used**; all back-office functions are admin-namespace APIs consumed by the custom CMS.

## URL Naming
- Lowercase, hyphenated, **plural nouns** for collections: `/storefront/products/`, `/admin/products/`, `/customer/orders/`. Trailing slash (Django convention) — consistent everywhere.
- Resource ids: `/storefront/products/{slug}/`, `/customer/orders/{public_id}/`, `/admin/orders/{public_id}/` (see ADR-0004).
- Nested only one level to express ownership: `/storefront/products/{slug}/reviews/`.
- Non-CRUD actions are verbs as sub-resources with POST: `/customer/orders/{id}/cancel/`, `/admin/orders/{id}/status/`.
- Current-user resources: `/customer/me/`, `/customer/me/addresses/`, `/customer/cart/`, `/customer/wishlist/`, `/customer/checkout/` (singletons per user). Their lines are addressed by product `public_id`: `/customer/cart/items/{product_id}/` ([ADR-0012](decisions/ADR-0012-cart-and-wishlist.md)). Orders and order placement: `/customer/orders/` ([ADR-0013](decisions/ADR-0013-checkout-and-orders.md)).
- Admin permission per endpoint is documented in the contract (`x-required-permission`, e.g. `orders.read`, `orders.update`).

## Methods
| Method | Use | Idempotent |
|---|---|---|
| GET | Read | yes |
| POST | Create / action | no |
| PUT | Full replace (rare) | yes |
| PATCH | Partial update (preferred over PUT) | yes |
| DELETE | Remove | yes |

## Status Codes
| Code | Meaning |
|---|---|
| 200 | OK (read/update/action with body) |
| 201 | Created (+ `Location` header where applicable) |
| 204 | Success, no body (delete, logout) |
| 400 | Validation error / malformed request |
| 401 | Missing/invalid/expired authentication |
| 403 | Authenticated but not permitted |
| 404 | Not found (also used for other users' private resources to avoid leakage) |
| 405 | Method not allowed |
| 409 | State conflict (e.g. insufficient stock, invalid status transition, duplicate) |
| 415 | Unsupported media type |
| 429 | Rate limited |
| 500 | Unexpected server error (generic message, details only in logs) |

## Error Format
All non-2xx responses (including DRF defaults, via a custom exception handler in `common`):
```json
{
  "error": {
    "code": "validation_error",
    "message": "One or more fields are invalid.",
    "details": {
      "email": ["Enter a valid email address."]
    }
  }
}
```
- `code`: stable machine-readable snake_case (`validation_error`, `authentication_failed`, `not_authenticated`, `permission_denied`, `not_found`, `conflict`, `insufficient_stock`, `product_unavailable`, `invalid_state_transition`, `throttled`, `server_error`). Frontend branches on `code`, never on `message`.
- `message`: human-readable, safe to display, no internals.
- `details`: optional; field errors as `{field: [messages]}`, non-field errors under `non_field_errors`.

## Pagination
Page-number pagination on all list endpoints (default `page_size=20`, max 100):
```json
{ "count": 125, "next": "https://…?page=3", "previous": "https://…?page=1", "results": [ ] }
```
Query: `?page=2&page_size=20`. Chosen for simplicity; cursor pagination only if a measured need arises.

## Filtering, Search, Sorting
- Filtering: explicit query params with field names, e.g. `?category=serums&min_price=10&max_price=50&in_stock=true`. Only documented params are accepted (via `django-filter`, decision on that dependency deferred to the products task).
- Search: `?search=<text>`.
- Sorting: `?ordering=price` / `?ordering=-price`; whitelist of fields documented per endpoint.
- Unknown params are ignored; invalid values → 400.

## Authentication & Authorization
- Customers: `Authorization: Bearer <access_token>` ([ADR-0003](decisions/ADR-0003-auth-token-strategy.md), Proposed). CMS: separate `/admin/auth/` endpoints, token audience `cms` ([ADR-0009](decisions/ADR-0009-cms-auth-rbac-audit.md), Proposed). Tokens are not interchangeable across namespaces.
- Default DRF permission: deny/`IsAuthenticated`; public endpoints opt in explicitly with `AllowAny`; every admin view declares `HasPermission("<resource>.<action>")`.
- Permissions are resolved server-side from the database. Role/permission model: see [cms-architecture](cms-architecture.md) and [security.md](../security/security.md).

## Admin (CMS) API Conventions
- List endpoints: pagination, `search`, `ordering`, explicit filters (e.g. orders by `status`, `payment_status`, date range, customer).
- Updates use PATCH; stale edits detected with `If-Match`/`updated_at` → `409 conflict`.
- State changes are action endpoints (`POST /admin/orders/{id}/status/`), enforcing allowed transitions server-side.
- Secret fields are **write-only**: responses return `{"is_set": true}`, never the value.
- Every admin write is audit-logged by the backend service layer.
- Exports are explicit endpoints guarded by their own permission.
- Dashboard/analytics endpoints return backend-computed aggregates; clients never compute authoritative metrics.

## Request/Response Rules
- Responses never contain secrets, password hashes, or internal-only fields.
- Write endpoints use dedicated input serializers; clients cannot set computed/server-owned fields (`price` on order items, `total`, `status`, `is_staff`, …) — unknown/read-only fields are ignored.
- Money is serialized as a **string** decimal (`"19.90"`) with a separate `currency`.
- Rate limiting (DRF throttling) on auth endpoints at minimum.
- Health: `GET /api/v1/health/` (unauthenticated, no sensitive info) for monitoring.

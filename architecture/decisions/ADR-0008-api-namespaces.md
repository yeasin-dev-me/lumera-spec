# ADR-0008: API namespaces
- Status: **Proposed — needs approval**
- Date: 2026-10-04

## Context
The API serves the storefront (anonymous and logged-in customers), the CMS, and server-to-server callers (payment webhooks). Mixing them risks exposing admin behavior publicly.

## Decision (proposed)
Under `/api/v1/`:

| Namespace | Audience | Auth | Examples |
|---|---|---|---|
| `/storefront/` | Public (anonymous OK), read-mostly | none (`AllowAny`) | `products/`, `categories/`, `content/…`, `config/` (public config only), `products/{slug}/reviews/` (approved), tracking event intake |
| `/auth/` | Customers | credentials → customer tokens | `register/ login/ refresh/ logout/` |
| `/customer/` | Authenticated customers (own data only) | customer token (`aud=storefront`) | `me/`, `me/addresses/`, `cart/`, `checkout/`, `orders/`, review creation |
| `/admin/` | CMS users | CMS token (`aud=cms`) + per-view permission | `auth/…`, `products/`, `orders/`, `customers/`, `content/`, `integrations/`, `audit/`, `dashboard/` |
| `/internal/` | Machine callers (payment webhooks, scheduled jobs) | provider signature / shared-secret; never browser-facing; not routed by Nginx unless needed (webhooks only) | `payments/webhook/<provider>/` |
| `/health/` | Monitoring | none | liveness/readiness |

Rules: the namespace is part of the URL **and** the permission class (default deny), enforced in routing (`urls.py` per namespace) so an admin view cannot be accidentally mounted publicly. Resource shapes may differ per namespace (admin serializers expose more); never reuse an admin serializer in a public view. Each namespace has its own tag in the OpenAPI contract.

This supersedes the earlier `/api/v1/products/`-style flat paths in the TASK-001 draft (never implemented).

## Alternatives Considered
Flat resource URLs with permission-based differences (mixing audiences; rejected); separate subdomain/service for admin API (heavier, no need now).

## Consequences
Slightly longer URLs; clearer security review, throttling, logging, and Nginx/Cloudflare rules per namespace.

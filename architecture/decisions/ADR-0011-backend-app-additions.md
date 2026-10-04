# ADR-0011: Backend app additions for CMS
- Status: **Proposed — needs approval**
- Date: 2026-10-04

## Context
ADR-0002 fixed the backend apps to `users, products, categories, cart, orders, payments, reviews, common`. CMS features need homes for content, promotions, integrations/tracking, audit, and aggregate metrics.

## Decision (proposed)
Add the following apps, each **created only in the task that first needs it**:

| App | Purpose |
|---|---|
| `audit` | Append-only audit log + helper |
| `content` | Structured storefront content (blocks, banners, pages, announcement bar, SEO metadata) |
| `promotions` | Coupons / discount rules / campaigns (only when the requirement exists; plain `discount_price` lives in `products`) |
| `integrations` | Integration settings, provider adapters, tracking dispatch |
| `analytics` | Read-only dashboard/metrics selectors (no models initially) |

Roles/permissions/staff management stay in `users`; customer management is `users` too. Store-wide settings (non-integration) are modeled when a requirement exists (likely in `content` or a minimal `settings` model — decide then).

## Alternatives Considered
Putting these into existing apps (blurs responsibilities); a single `cms` app (duplicates domain logic and couples everything).

## Consequences
Supersedes the app list in ADR-0002 once approved; `architecture.md` already reflects the proposed list marked as pending.

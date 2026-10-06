# Architecture Decision Records

ADRs record significant decisions. File name: `ADR-NNNN-short-title.md`. Statuses: **Proposed → Accepted / Rejected → Superseded**. Accepted ADRs are not edited; supersede them with a new one.

Agents must not implement a decision whose ADR is still **Proposed** without human approval.

## Index
| ADR | Title | Status |
|---|---|---|
| [0001](ADR-0001-stack-and-repositories.md) | Technology stack & four-domain repository model | Accepted (from project brief) |
| [0002](ADR-0002-modular-monolith.md) | Modular monolith backend | Accepted (from project brief) |
| [0003](ADR-0003-auth-token-strategy.md) | Authentication tokens & API routing | **Proposed** |
| [0004](ADR-0004-primary-keys.md) | Primary key & public identifier strategy | **Proposed** |
| [0005](ADR-0005-payment-provider.md) | Payment provider | **Superseded by ADR-0014** |
| [0006](ADR-0006-merge-and-branching.md) | Git workflow: trunk-based, squash merge | **Proposed** |
| [0007](ADR-0007-cms-application.md) | CMS as a separate Next.js application/repository | **Proposed** (custom Next.js CMS / no Django Admin: Accepted by amendment) |
| [0008](ADR-0008-api-namespaces.md) | API namespaces (`storefront`, `auth`, `customer`, `admin`, `internal`) | **Proposed** |
| [0009](ADR-0009-cms-auth-rbac-audit.md) | CMS authentication, permission-based RBAC, audit log | **Proposed** |
| [0010](ADR-0010-integrations-secrets-tracking.md) | Integrations, secret configuration, tracking architecture | **Proposed** |
| [0011](ADR-0011-backend-app-additions.md) | Backend app additions (`audit, content, promotions, integrations, analytics`) | **Proposed** |
| [0012](ADR-0012-cart-and-wishlist.md) | Customer cart & wishlist: dynamic pricing, stock validation, ownership, identifiers | **Proposed** (implemented in Step 5) |
| [0013](ADR-0013-checkout-and-orders.md) | Checkout, Orders, Price Snapshots, and Transactional Inventory Deduction | Accepted (Step 6) |
| [0014](ADR-0014-payment-architecture-and-gateway.md) | Payment Architecture, Gateway Abstraction, and Server-Side Verification | Accepted (Step 7) |
| [0015](ADR-0015-order-lifecycle-fulfillment-notifications.md) | Order Lifecycle, Fulfillment, and Customer Notifications | Accepted (Step 8) |

## Template
```markdown
# ADR-NNNN: Title
- Status: Proposed | Accepted | Rejected | Superseded by ADR-XXXX
- Date: YYYY-MM-DD
- Deciders: 

## Context
## Decision
## Alternatives Considered
## Consequences
```

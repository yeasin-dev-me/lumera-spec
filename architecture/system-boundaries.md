# System Boundaries

## Systems & Responsibilities
```text
FRONTEND     → Customer Storefront
CMS          → Business/Admin Operations UI
BACKEND      → API + Database + Business Logic + Security
INTEGRATION  → Frontend/CMS ↔ Backend + E2E
DEVOPS       → Infrastructure + CI/CD + Deployment + Monitoring
```
The CMS is a Next.js interface, but its **data and authority come from the Backend**.

| Domain / System | Owns | Does NOT own |
|---|---|---|
| **Frontend (Storefront)** | Customer UI/UX, routing, components, client state, API consumption, forms, SEO, accessibility, responsive behavior, storefront tracking module | Business rules, price/total/stock calc, authorization, persistence, any admin functionality |
| **CMS** | Admin/operator UI: dashboards, management screens, forms/tables, permission-aware UI | Business rules, authoritative pricing, authorization decisions, DB access, secrets, provider calls |
| **Backend** | Business logic, database, authentication, authorization (RBAC), validation, API (all namespaces), order calculations, inventory, payment verification, secret storage. **Implemented through Step 7:** core commerce, RBAC, checkout/orders, and payments. **Planned:** audit log and integration/tracking dispatch. | Presentation |
| **Integration** | FE/CMS ↔ BE compatibility, E2E workflows, contract validation (all namespaces), cross-system tests, mismatch detection. **Implemented through Step 7:** core commerce and payment vertical-slice coverage. | Feature implementation (reports mismatches to owners) |
| **DevOps** | Docker, CI/CD, environments, deployment of storefront/CMS/backend, VPS/cloud, Nginx, SSL, secrets/config, monitoring, backups, rollback | Application code |

## External Systems (outside Lumera, accessed **only by the backend**)
Payment providers, shipping providers, email/SMS providers, analytics/marketing platforms (Meta, Google, …). Browsers load only provider *public* scripts via the storefront tracking module; no browser (storefront or CMS) ever calls a provider with a secret.

## Interface Contracts
| Boundary | Contract |
|---|---|
| Storefront ⇄ BE | `contracts/api-contract.yaml` namespaces `/storefront/`, `/auth/`, `/customer/` |
| CMS ⇄ BE | same contract, namespace `/admin/` (and `/admin/auth/`) |
| Provider → BE | `/internal/` webhooks, signature-verified |
| BE → providers | Provider adapters in `integrations`/`payments` |
| App ⇄ DevOps | Env variable catalog in `devops/environments.md`; health: backend `GET /api/v1/health/`, storefront and CMS `GET /healthz` |

## Rules
1. No domain imports another's code; the contract is the only shared artifact.
2. Contract changes land **before** implementation; breaking changes need `/api/v2/` or an approved plan.
3. Storefront and CMS types are derived from / checked against the contract.
4. Each domain edits only its own repo plus relevant `lumera-spec` docs.
5. Cross-domain problems go to `known-issues.md`/task file — never silent workarounds.
6. Integration tests treat all apps as black boxes.
7. **The CMS never touches PostgreSQL, never bypasses backend permissions, never contains secrets, and never duplicates critical business logic.**
8. Admin behavior is reachable only through `/api/v1/admin/` with authentication + permission; public and customer namespaces never expose admin capabilities.
9. Feature tasks are **vertical slices** across backend, storefront, CMS, and integration ([roadmap](../tasks/roadmap.md)); CMS is not deferred to the end.

## Change Ownership
| Change | Owner | Also update |
|---|---|---|
| New/changed endpoint | Backend | contract, storefront/CMS types, integration tests |
| New storefront UI data need | Frontend | contract (via task) |
| New CMS screen | CMS | contract if endpoint missing; permissions catalog |
| New permission | Backend | `cms-architecture.md` catalog, CMS UI gating, permission-matrix tests |
| Env var / deployment | DevOps | environments.md |
| Architecture change | Anyone proposes → human approves | ADR, architecture docs |

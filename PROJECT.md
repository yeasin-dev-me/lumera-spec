# Penguin Mart (formerly Lumera) — PROJECT.md

Single entry point for humans and AI agents. `lumera-spec` is the **Single Source of Truth**; if code and spec disagree, raise it — don't silently pick one.

## Product
**Penguin Mart** (Skin Care & Beauty) is a modern skincare and beauty e-commerce platform: storefront, customer accounts, catalog, categories, cart, checkout, orders, payments, reviews, and a **custom CMS / operations system** for the business.

## Stack (fixed — changes need explicit approval)
| Layer | Technology |
|---|---|
| Frontend (Storefront) | Next.js, TypeScript, Tailwind CSS |
| CMS | Custom Next.js, TypeScript, Tailwind CSS — **Django Admin is NOT used** |
| Backend | Python, Django, Django REST Framework (headless API) |
| Database | PostgreSQL |
| Infra | Docker, CI/CD, VPS/cloud, Nginx, Cloudflare |

## Repositories / Domains
```text
LUMERA
├── lumera-spec   (Shared Specification — this repo)
├── frontend      (Customer Storefront — Next.js)
├── cms           (Business/Admin Operations UI — Next.js; separate repo pending ADR-0007)
├── backend       (API + Database + Business Logic + Security — Django/DRF)
├── integration   (contract + Storefront/CMS ↔ Backend + E2E tests)
└── devops        (Infrastructure, CI/CD, deployment, monitoring)
```
Storefront, CMS, and backend are independent, independently deployable applications. The CMS is a Next.js interface, but its data and authority come from the Backend. Django Admin is not used. See [system-boundaries](architecture/system-boundaries.md) and [cms-architecture](architecture/cms-architecture.md).

## Core Principles
1. Storefront, CMS, and backend deploy independently.
2. Backend is the source of truth for business logic.
3. Never trust client-provided prices, totals, stock, permissions, or other business-critical values — **including values sent by the CMS**.
4. Storefront and CMS talk to the backend only via documented REST APIs (`/api/v1/`), in separate audience namespaces (`storefront`, `customer`, `admin`, `internal`).
5. Cross-system contracts are defined **before** implementation.
6. Modular monolith backend; no unnecessary microservices.
7. Avoid unnecessary dependencies; prefer simple solutions over premature abstraction.
8. No architecture changes without explicit approval (write an ADR first).
9. Do not modify unrelated files or systems; reuse established patterns.
10. Security-sensitive validation is server-side.
11. Every completed feature has automated tests.
12. Production deployment is reproducible.
13. Important decisions are documented.
14. **The CMS is a first-class system:** it never accesses the database, never holds secrets, never bypasses backend permissions, and never duplicates critical business logic. Authorization is enforced by the backend (permission-based RBAC); Django Admin is not used.
15. Features are delivered as vertical slices (backend + storefront + CMS + integration tests); CMS work is not deferred to the end.
16. Secrets never reach browsers; third-party/provider integrations (payments, tracking, …) are centralized and server-side where they carry secrets.

## Document Map
| Topic | Document |
|---|---|
| Requirements / rules / stories | [requirements](requirements.md), [business-rules](business-rules.md), [user-stories](user-stories.md), [acceptance-criteria](acceptance-criteria.md) |
| Architecture | [architecture](architecture/architecture.md), [system-boundaries](architecture/system-boundaries.md), [cms-architecture](architecture/cms-architecture.md), [database](architecture/database.md), [decisions](architecture/decisions/README.md) |
| API | [api-architecture](architecture/api-architecture.md), [api-contract.yaml](contracts/api-contract.yaml) |
| Design | [design-system](design/design-system.md) |
| Security | [security](security/security.md) |
| Testing | [testing-strategy](testing/testing-strategy.md) |
| Workflow | [git-workflow](git-workflow.md) |
| Environments | [environments](devops/environments.md) |
| Tracking | [tasks/](tasks/), [roadmap](tasks/roadmap.md), [known-issues](known-issues.md), [lessons](lessons.md) |

## Agent Instruction Files
`frontend/AGENTS.md` (storefront), `cms/AGENTS.md`, `backend/AGENTS.md`, `integration/AGENTS.md`, `devops/AGENTS.md` — domain-specific rules only. Read this file first, then your domain file.

## Status
- TASK-001 (foundation & architecture) and TASK-001-AMENDMENT (custom CMS & operations architecture): documentation complete, **pending approval of Proposed ADRs** (see [decisions](architecture/decisions/README.md)).
- No business features are implemented. TASK-002 must not start until the amendment is reviewed.

## Task Process
1. Task file in `tasks/` states scope, affected domains, and contract changes.
2. If it touches more than one domain, update `contracts/api-contract.yaml` first.
3. Implement within domain boundaries; meet the Definition of Done in [testing-strategy](testing/testing-strategy.md).
4. Record surprises in `lessons.md`, unresolved bugs in `known-issues.md`.

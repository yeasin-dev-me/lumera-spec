# Testing Strategy

## Layers
| Domain | Layer | Tools (proposed) | Scope |
|---|---|---|---|
| Frontend (Storefront) | Unit / Component | Vitest (or Jest) + React Testing Library; optional MSW for API mocks | Components, hooks, utils, form behavior, tracking module (events mapped, no direct provider calls), a11y checks (`jest-axe`) |
| CMS | Unit / Component | Same toolchain as storefront | Tables, forms, permission-aware navigation/gating, validation-error mapping, secret fields (write-only UI), confirmation flows |
| Backend | Unit | pytest + pytest-django | Models/constraints, services (business rules), selectors, permissions |
| Backend | API | pytest + DRF `APIClient` | Endpoints per namespace: status codes, schema/format, auth, authorization, validation, tampering |
| Backend | **Permission matrix** | pytest parametrized | Every `/admin/` endpoint × {anonymous, customer token, staff without permission, staff with permission}: 401/401-or-403/403/2xx. Customer tokens rejected on `/admin/`; CMS tokens rejected on `/customer/` |
| Backend | **Audit & secrets** | pytest | Admin writes create audit entries; secrets never appear in any API response, log capture, or audit before/after; secret fields write-only |
| Backend | **Tracking/integrations** | pytest with provider stubs | Event dedup by `event_id`, purchase event sent from verified order data, failures don't block orders, public-config whitelist |
| Integration | Integration | Contract validation of **all namespaces** (backend responses vs `api-contract.yaml`, e.g. schemathesis/openapi validator) + storefront and CMS types vs contract | Detect Storefront/CMS ↔ Backend mismatches |
| System | E2E | Playwright (storefront and CMS projects) | Critical journeys against a full stack in Docker Compose, including CMS → storefront propagation |
| DevOps | Deployment / Health | Smoke script, `/api/v1/health/`, `/healthz` (storefront, CMS), container healthchecks | Post-deploy verification, rollback trigger |

Tool names are proposals; picking them is part of each domain's scaffolding task (no dependencies added in TASK-001).

## Principles
- Test behavior, not implementation. Fast deterministic tests; no real network/payment calls (provider stubbed/test mode).
- Business rules are tested at service level; API tests verify the wiring, permissions, and error format.
- Every security-relevant rule has a negative test (unauthenticated, wrong user, tampered price/total/stock/status).
- Bug fix = regression test first.
- Test data via factories (e.g. factory_boy); each test isolated; PostgreSQL used in CI (not SQLite) to match production behavior (constraints, locking).

## Critical E2E Journeys (when features exist)
Register → login → browse → add to cart → checkout → pay (test mode) → see order; operator creates product in CMS → visible in storefront; operator updates order status in CMS → customer sees it; content change in CMS → storefront shows it; price/discount edit in CMS → storefront and new orders reflect it (old orders unchanged); unauthorized access attempts (customer → `/admin/`, staff lacking permission) blocked; purchase tracking event emitted once.

## Coverage Guidance
No blanket percentage gate initially. Expectation: all service functions and permissions covered; coverage reported in CI; a threshold may be introduced via ADR once a baseline exists.

## CI Gates (per repo)
Lint + format check, type check (FE: `tsc`; BE: optionally mypy later), unit/API tests, build, migration check (`makemigrations --check`), dependency audit (advisory). Integration/E2E run on `main` and before releases (and on PRs touching contract).

## Definition of Done (every future task)
A task is done only when:
1. Scope in the task file is met; no unrelated changes.
2. Contract (`api-contract.yaml`) updated first and matches implementation, including `x-required-permission` for admin operations.
3. Server-side validation/authorization in place; business-critical values never trusted from client (including from the CMS).
4. Automated tests added at the appropriate layers, including negative/security cases (permission matrix for admin endpoints, audit, secret non-disclosure); all pass in CI.
4a. **Vertical slice:** if the feature has an operator capability, the CMS part ships in the same task (or the task documents a deferral approved by the owner); integration/E2E tests cover the slice.
5. Lint, format, type checks pass; no new warnings.
6. Migrations included, reversible-safe, and checked.
7. UI work: responsive, accessible (keyboard, labels, contrast), has loading/empty/error states, uses design tokens.
8. No new dependency without PR justification; no secrets committed.
9. Docs updated (spec, ADR if architectural, `.env.example`, known issues/lessons).
10. PR reviewed and approved; squash-merged.
11. If deployable: works in staging, health checks green.

# TASK-001-AMENDMENT — Custom CMS & Operations Architecture

- Status: **Done — pending human approval of Proposed ADRs.** TASK-002 must not start before review.
- Type: Documentation only. No CMS code, no Django Admin, no business feature implementation.

## What Changed
| Area | Change |
|---|---|
| Architecture | Custom Next.js **CMS** is a first-class system; **Django Admin excluded** (`django.contrib.admin` not installed). Responsibility model: Frontend = Storefront, CMS = Admin Operations UI, Backend = API + DB + logic + security, Integration = Storefront/CMS ↔ Backend + E2E, DevOps = infra/CI/CD/deploy/monitoring |
| New doc | `architecture/cms-architecture.md` (responsibility, communication, auth, authorization, modules, API boundaries, secrets, audit, deployment) |
| `architecture/architecture.md` | Rewritten for storefront + CMS + backend; audience-split serializers/URLs; shared services; proposed extra backend apps |
| `architecture/system-boundaries.md` | Rewritten with CMS domain, external systems, CMS hard rules |
| `architecture/api-architecture.md` | Namespaces (`storefront/auth/customer/admin/internal`), admin conventions |
| `architecture/database.md` | Product discount/status/featured/ordering, category image (flat), order history/notes, payment status, Role, AuditLog, ContentBlock, Coupon, IntegrationSetting, TrackingEventLog |
| `contracts/api-contract.yaml` | Namespace tags, `cmsBearerAuth`, `x-required-permission` convention, `SecretFieldStatus`, `PermissionList`, `TrackingEvent` |
| `security/security.md` | Permission-based RBAC, CMS auth, secrets, audit, customer-data minimization, CMS hardening |
| `testing/testing-strategy.md` | CMS layer, permission matrix, audit/secret tests, tracking tests, CMS E2E, vertical-slice DoD |
| `requirements.md`, `business-rules.md`, `user-stories.md`, `acceptance-criteria.md` | CMS requirements, BR-02a/b, revised BR-04/05/12/14/20, BR-21–29, US-11–23, AC-X9–X12 |
| `tasks/roadmap.md` | Vertical-slice plan; CMS included in every relevant feature |
| Agent files | New `cms/AGENTS.md`; updated frontend, backend, integration, devops |
| ADRs | 0007–0011 (Proposed); notes added to 0002 and 0003 |

## Decisions Requiring Approval
1. **ADR-0007** — CMS as a **separate repo/app** (recommended) vs. inside the storefront app vs. monorepo.
2. **ADR-0008** — API namespaces.
3. **ADR-0009** — CMS auth audience, permission-based RBAC (only `super_admin` seeded), audit log.
4. **ADR-0010** — Integration secrets write-only/encrypted; payment secrets env-only by default; centralized tracking with server-side purchase events.
5. **ADR-0011** — Added backend apps.
6. **BR-12** — Proposed order lifecycle (`pending → confirmed → processing → shipped → delivered`, cancellation; returns/refunds undefined). Needs business confirmation. This **changes** the earlier TASK-001 draft lifecycle (which had `paid` as an order status); payment status is now separate.
7. Open product questions 7–15 in `requirements.md`.

## Conflict Resolutions Made
- Earlier flat URLs (`/products/`, `/users/me/`) → namespaced (never implemented).
- Earlier "Django admin may cover back-office" and `is_superuser` admin role → removed.
- Earlier category `parent` → removed (flat until justified).
- Earlier `is_active` on Product/Category → `status` + `is_visible`.

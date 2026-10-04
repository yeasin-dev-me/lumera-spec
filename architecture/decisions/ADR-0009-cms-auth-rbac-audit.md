# ADR-0009: CMS authentication, RBAC, and audit logging
- Status: **Proposed — needs approval**
- Date: 2026-10-04

## Context
The CMS controls business operations; authorization must be enforced by the backend with granular permissions, and administrative actions must be traceable. Django Admin is excluded.

## Decision (proposed)
1. **Auth:** separate `/admin/auth/` endpoints, same token mechanism as ADR-0003, but audience `cms`, shorter lifetimes, stricter throttling; only `is_staff` accounts.
2. **RBAC:** permissions are code-defined strings `resource.action` (e.g. `orders.update`). Roles are DB records (named permission sets) assigned to staff users. Implemented with a small custom `Role`/permission mapping in `users` (Django's `auth` Permission/Group tables are acceptable as storage if convenient — an implementation detail, not Django Admin). `super_admin` has all permissions; only `super_admin` seeded. The permission is the unit of authorization — never the role name — so roles can change without code changes.
3. **Enforcement:** DRF permission class `HasPermission("orders.update")` declared on every admin view; default deny; permissions resolved from the DB per request (cacheable later), never from the token.
4. **Audit:** append-only `audit` log written by backend services for admin write actions (actor, time, action, resource, redacted before/after, request id, IP). No CMS-side logging for audit purposes. Minimal first (one table + helper); extended per feature.

## Alternatives Considered
Fixed hard-coded roles (inflexible, creates unneeded roles); Django Groups only without a permission catalog (no reviewable permission list); third-party RBAC/audit packages (extra dependencies; reconsider only if building proves costly).

## Consequences
Every admin feature task must define its permissions and audit actions, and add permission-matrix tests. Permission catalog lives in `cms-architecture.md` and is refined per task.

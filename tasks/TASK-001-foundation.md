# TASK-001 — Project Foundation & Architecture

- Status: **Done**
- Domains: Spec (all domains read)
- Type: Documentation only. No application code, dependencies, or infrastructure.

## Deliverables → Location
| # | Deliverable | File |
|---|---|---|
| 1 | Architecture | `architecture/architecture.md` |
| 2 | System boundaries | `architecture/system-boundaries.md` |
| 3 | Database / ERD | `architecture/database.md` |
| 4 | API architecture + OpenAPI skeleton | `architecture/api-architecture.md`, `contracts/api-contract.yaml` |
| 5 | Authentication design | `security/security.md`, ADR-0003 |
| 6 | Environment strategy | `devops/environments.md` |
| 7 | Design system | `design/design-system.md` |
| 8 | Testing strategy + DoD | `testing/testing-strategy.md` |
| 9 | Security guidelines | `security/security.md` |
| 10 | Git workflow | `git-workflow.md`, ADR-0006 |
| 11 | Agent instruction files | `PROJECT.md`, `../frontend|backend|integration|devops/AGENTS.md` |
| 12 | ADR structure | `architecture/decisions/` |
| 13 | Project docs | `PROJECT.md`, `requirements.md`, `business-rules.md`, `user-stories.md`, `acceptance-criteria.md` |

## Decisions Requiring Approval
1. ADR-0003 — JWT (access in memory + rotating HttpOnly refresh cookie), same-origin routing. The current
   vertical-slice implementation still uses client-managed refresh tokens; cookie migration remains a
   security hardening task.
2. ADR-0004 — BIGINT PK + `public_id` UUID on sensitive entities.
3. ADR-0014 — Payment architecture and gateway selection is accepted. SSLCommerz is the initial
   configurable gateway; the gateway abstraction is live in Step 7.
4. ADR-0006 — Trunk-based, squash merge.
5. Open product questions in `requirements.md` (currency, guest checkout, shipping/tax, review policy, email provider).

## Notes
Two small additions beyond the requested tree: `architecture/api-architecture.md` and `git-workflow.md` (both required deliverables with no listed home). `devops/environments.md` and `integration/` hold their domain docs.

## Suggested Next Tasks (not started)
Superseded by [roadmap.md](roadmap.md) (vertical slices incl. CMS). See [TASK-001-AMENDMENT-cms](TASK-001-AMENDMENT-cms.md) for the CMS amendment that updated this task's documents.

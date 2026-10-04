# ADR-0007: CMS as a separate Next.js application/repository
- Status: **Proposed — needs approval** (CMS being custom Next.js and Django Admin being excluded is **Accepted** per TASK-001-AMENDMENT)
- Date: 2026-10-04

## Context
The amendment requires a custom Next.js CMS and asks that its structure (separate repo vs. part of the frontend) be decided explicitly, consistent with the existing "independent applications/repositories per domain" strategy.

## Decision (proposed)
A **separate `cms` repository and separately deployed Next.js application**, a fifth execution domain next to `frontend` (storefront), `backend`, `integration`, `devops`.

## Alternatives Considered
1. **Same Next.js app as the storefront (route group `/cms`)**: one repo/deploy, easy code sharing. Rejected: admin code and its dependencies ship in the customer deployment; any storefront release redeploys the admin and vice versa; larger blast radius for XSS/supply-chain issues; harder to isolate behind Cloudflare Access/IP rules; different release cadence and security posture.
2. **Monorepo with two apps (`apps/storefront`, `apps/cms`) + shared packages**: good sharing, but changes the multi-repo strategy and adds tooling (workspaces/Turborepo). Not chosen now; can be revisited if duplication hurts.

## Consequences
+ Independent deploy, CSP, access control, dependencies, and release cadence; matches ADR-0001.
− Some duplication (design tokens, API-client generation from the contract). Mitigation: both generate types from `api-contract.yaml`; tokens are copied from `design-system.md`; extract a shared package only via a later ADR.
Requires `cms/AGENTS.md`, CMS CI/CD pipeline, and an extra container/host.

# ADR-0001: Technology stack & four-domain repository model
- Status: Accepted (specified by the project brief)
- Date: 2026-10-04

## Context
Lumera needs a storefront and a business backend maintained with clear ownership.

## Decision
- Frontend: Next.js + TypeScript + Tailwind CSS. Backend: Python + Django + DRF. DB: PostgreSQL. Infra: Docker, CI/CD, VPS/cloud, Nginx, Cloudflare.
- Separate repos/domains: `lumera-spec`, `frontend`, `backend`, `integration`, `devops`.
- Backend is the source of truth for business logic; communication via REST under `/api/v1/`.

## Alternatives Considered
Single monorepo with Next.js API routes as backend; rejected by the brief (independent deployability, backend authority).

## Consequences
Contract-first workflow is required; integration tests are a first-class domain. Stack changes require a new ADR with explicit approval.

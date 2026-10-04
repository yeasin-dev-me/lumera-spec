# ADR-0006: Git workflow — trunk-based, squash merge
- Status: **Proposed — needs approval**
- Date: 2026-10-04

## Decision (proposed)
Single long-lived branch `main` (always deployable), short-lived feature branches, PRs required, **squash merge**, linear history. Staging deploys from `main`; production deploys from version tags. Details in [git-workflow.md](../../git-workflow.md).

## Alternatives Considered
GitFlow (`develop`/`release` branches): rejected as unnecessary complexity for the team size.

## Consequences
Requires good CI and feature-incomplete code kept behind not-yet-routed paths; hotfixes branch from the production tag.

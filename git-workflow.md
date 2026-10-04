# Git & Development Workflow

Applies to every repo (`lumera-spec`, `frontend`, `cms`, `backend`, `integration`, `devops`). Proposed in [ADR-0006](architecture/decisions/ADR-0006-merge-and-branching.md).

## Branching
- `main`: protected, always deployable. No direct pushes.
- Short-lived branches from `main`, deleted after merge:
  `feat/<task-id>-<slug>`, `fix/<task-id>-<slug>`, `docs/<slug>`, `chore/<slug>`, `refactor/<slug>`, `test/<slug>`
  Example: `feat/TASK-012-product-list-api`.
- Hotfix: `fix/…` branched from the production tag, merged to `main` too.

## Commits (Conventional Commits)
`<type>(<scope>): <summary>` — types: `feat, fix, docs, style, refactor, test, chore, ci, build, perf`. Imperative, ≤72 chars, scope = app/area. Reference the task: `feat(products): add list endpoint (TASK-012)`. Breaking changes: `!` + `BREAKING CHANGE:` footer. Small, logically coherent commits.

## Pull Requests
- One task/concern per PR; keep small (aim < 400 changed lines excluding generated/lockfiles).
- Description: what/why, link to task, contract/ADR changes, how tested, screenshots for UI, new dependencies + justification.
- Checklist: Definition of Done met, tests added, docs updated, no secrets, no unrelated files touched.
- Draft PRs welcome for early feedback.

## Review
- At least 1 approval required (human). AI-authored changes are reviewed like any other.
- Reviewers check: correctness, boundary adherence (no business logic in FE; no trust of client values), security checklist, tests, contract consistency, simplicity.
- Author resolves all comments; re-request review after substantive changes.

## Merge
- **Squash merge** into `main`; squash commit message follows Conventional Commits. Linear history. No force-push to `main`; avoid rewriting shared branches.

## CI Requirements (must pass before merge)
Lint/format, type check, tests, build, migration check (backend), contract validity (`api-contract.yaml` lints), secret scan. Defined per testing-strategy; implemented by DevOps. Rebase/update branch on `main` before merge.

## Releases
Semantic version tags (`vX.Y.Z`) per deployable (frontend, backend). Production deploys from tags only. A release/compatibility note lives in `integration/`.

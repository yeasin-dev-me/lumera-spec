# AGENTS.md — lumera-spec

## Repository Responsibility
`lumera-spec` is the authoritative **Single Source of Truth** for the entire Lumera project. All architectural standards, API contracts, entity schemas, security policies, and task roadmaps must be recorded here.

## Allowed Technologies
* Markdown (GFM)
* Mermaid.js diagrams
* OpenAPI 3.1 YAML (`contracts/api-contract.yaml`)

## Directory Conventions
* `architecture/`: High-level system design, domain boundaries, data models, error formats, and ADRs (`architecture/decisions/`).
* `contracts/`: OpenAPI 3.1 contracts defining exact endpoints, request/response bodies, status codes, and error schemas.
* `tasks/`: Task specifications (`TASK-*.md`) and roadmap tracking (`roadmap.md`).
* `security/`: Threat modeling, authentication flow, and RBAC matrix.
* `testing/`: Test strategy, pyramid, DoD, and acceptance criteria.

## Coding Conventions
* Use clear, unambiguous technical writing.
* Fenced code blocks must specify syntax language (`markdown`, `yaml`, `mermaid`, `json`).
* Use RFC 2119 keywords (`MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHOULD`, `RECOMMENDED`, `MAY`) where defining constraints.

## Testing Requirements
* Verify internal markdown links.
* Ensure YAML syntax in `contracts/api-contract.yaml` is valid OpenAPI 3.1.

## Git Rules
* Branch naming: `feature/<topic>`, `doc/<topic>`, `adr/<decision-title>`.
* Commit format: Conventional Commits (`docs: ...`, `spec: ...`, `chore: ...`).
* Protected branch: `main`. All changes via Pull Request.

## Security Rules
* **NEVER** commit secrets, passwords, private keys, or API tokens into specification files.
* Use mock/placeholder data (`user@example.com`, `00000000-0000-0000-0000-000000000000`) in contract examples.

## What the Agent Must NOT Modify
* Do not introduce application source code, npm dependencies, or python packages into this repository.
* Do not alter accepted ADRs without recording a new superseding ADR.

## Where Shared Project Specification Lives
This repository (`lumera-spec`) IS the shared project specification root.

## How to Handle Architecture Changes
1. Propose an ADR under `architecture/decisions/ADR-XXXX-<title>.md` with status `Proposed`.
2. Update relevant domain specifications (e.g. `architecture/api-architecture.md`, `contracts/api-contract.yaml`).
3. Seek human approval before applying changes to downstream application repositories.

## Definition of Done
* Specification document is complete, formatted, internally linked, and aligned with `PROJECT.md`.
* No conflicting rules across documents.
* OpenAPI contracts validate without syntax errors.

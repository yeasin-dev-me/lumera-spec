# Lumera Specifications (`lumera-spec`)

## Purpose
`lumera-spec` is the **Single Source of Truth (SSoT)** for the entire Lumera e-commerce platform. It defines all system requirements, architectural specifications, domain boundaries, data models, API contracts, security standards, testing strategies, architectural decision records (ADRs), and vertical slice task definitions.

**Critical Rule:** This repository must **NEVER** contain application source code, runtime dependencies, or deployment credentials.

---

## Architecture Role
In the Lumera multi-repository ecosystem:
* All design decisions and domain contracts originate in `lumera-spec`.
* Changes to data models, API schemas, or cross-system workflows must be documented and approved in `lumera-spec` before implementation in application repositories.
* Application repositories (`lumera-frontend`, `lumera-cms`, `lumera-backend`, `lumera-integration`, `lumera-devops`) implement and adhere strictly to the contracts defined here.

```text
                        lumera-spec (SSoT)
                                │
       ┌────────────────┬───────┴────────┬───────────────┐
       ▼                ▼                ▼               ▼
lumera-backend  lumera-frontend     lumera-cms   lumera-integration
       │                │                │               │
       └────────────────┼────────────────┼───────────────┘
                        ▼
                  lumera-devops
```

---

## Technology Stack
* **Format:** GitHub Flavored Markdown (GFM)
* **Contracts:** OpenAPI 3.1 / YAML
* **Diagrams:** Mermaid.js

---

## Repository Structure
```text
lumera-spec/
├── PROJECT.md               # Master project overview and governance
├── requirements.md          # Functional & non-functional requirements
├── business-rules.md        # Authoritative domain & business rules
├── user-stories.md          # User & staff personas and journeys
├── acceptance-criteria.md   # System-wide acceptance criteria
├── git-workflow.md          # Branching, commits, and review flow
├── known-issues.md          # Ecosystem tracking of known issues
├── lessons.md               # Post-task retrospective lessons
├── architecture/            # Architecture specifications & system boundaries
│   ├── architecture.md      # Global architecture document
│   ├── system-boundaries.md # Domain ownership boundaries
│   ├── database.md          # PostgreSQL schemas, ERD, conventions
│   ├── api-architecture.md  # Multi-namespace REST standards & error schema
│   └── decisions/           # Architectural Decision Records (ADRs)
├── contracts/               # Machine-readable API contracts (OpenAPI)
├── design/                  # Design system tokens and UX foundations
├── security/                # Authentication, authorization, RBAC & threat model
├── testing/                 # Test levels, coverage rules, DoD
├── devops/                  # Environments and infrastructure specs
├── integration/             # Cross-repo contract rules
└── tasks/                   # Vertical slices, roadmaps, task specs
```

---

## Local Development
Since this repository consists purely of documentation and contract specifications:
1. Use any Markdown editor (VS Code, Obsidian, etc.) with Mermaid preview support.
2. Validate OpenAPI syntax with standard Swagger/OpenAPI validators.

---

## Environment Variables
* `lumera-spec` does **not** use runtime environment variables or `.env` files.
* Never commit real keys, tokens, or environment secrets to specification files.

---

## Running Tests
Markdown link checks and contract linting:
```bash
# Verify YAML syntax on contracts
npx @stoplight/spectral-cli lint contracts/api-contract.yaml
```

---

## Build
No build step is required for specification files.

---

## Related Repositories
* [lumera-backend](https://github.com/yeasin-dev-me/lumera-backend) — Headless Django REST API
* [lumera-frontend](https://github.com/yeasin-dev-me/lumera-frontend) — Customer-facing Next.js Storefront
* [lumera-cms](https://github.com/yeasin-dev-me/lumera-cms) — Operational Next.js CMS
* [lumera-integration](https://github.com/yeasin-dev-me/lumera-integration) — E2E and cross-system test harness
* [lumera-devops](https://github.com/yeasin-dev-me/lumera-devops) — Multi-container orchestration, Nginx, CI/CD

---

## Documentation
* Master Overview: [PROJECT.md](PROJECT.md)
* Architecture: [architecture/architecture.md](architecture/architecture.md)
* API Architecture: [architecture/api-architecture.md](architecture/api-architecture.md)
* API Contract: [contracts/api-contract.yaml](contracts/api-contract.yaml)
* Roadmap: [tasks/roadmap.md](tasks/roadmap.md)

---

## Contribution / Git Workflow
1. Create a branch: `feature/<spec-topic>` or `doc/<topic>`.
2. Propose changes or new ADRs in `architecture/decisions/`.
3. Submit a Pull Request for review.
4. Merge to `main` via squash-and-merge once approved.

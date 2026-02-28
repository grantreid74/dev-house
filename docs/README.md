# Dev-House Documentation Index

**Start here to find the right doc.** This is your fridge — check it first before searching code.

## Quick Navigation

| What You Need | Document | Purpose |
|---------------|----------|---------|
| **Project naming standards** | [naming-standards.md](naming-standards.md) | File naming patterns (sequential, snapshot, descriptive, code, config) with decision router |
| **Doc-code sync map** | [documentation-maintenance.md](documentation-maintenance.md) | When you change X, update Y docs (prevents documentation drift) |
| **Content review policy** | [review-policy.md](review-policy.md) | Self-review process: per-commit, per-session, quarterly deep reviews with staleness detection |
| **Document lifecycle** | [archive-structure.md](archive-structure.md) | Where docs go: transient (temp notes) → temporal (snapshots) → archived (historical) → removed |
| **GitHub as narrative** | [workflow/github-workflow.md](workflow/github-workflow.md) | Use GitHub issues/PRs as single source of truth for context and project evolution |
| **Edge cases & gotchas** | [gotchas.md](gotchas.md) | Things that don't work out of the box — captured as we discover them |
| **High-level architecture** | [architecture/overview.md](#) | System design, layers, components |
| **How it all connects** | [architecture/decisions.md](#) | Why we chose this architecture |
| **Claude Harness patterns** | [harness/orchestration.md](#) | Harness workflow engine |
| **Claude Codex integration** | [codex/generation.md](#) | Code generation patterns |
| **OpenClaw integration** | [openclaw/integration.md](#) | Infrastructure orchestration |
| **Deployment strategies** | [deployment/customer-deployment.md](#) | How customers deploy |
| **Design patterns** | [patterns/harness-patterns.md](#) | Reusable harness patterns |
| **API / Integration** | [harness/api.md](#) | Harness API reference |

---

## Architecture Documents

### Foundation
- **overview.md** — Layers (PRD input → Harness → Codex → Deployment)
- **decisions.md** — Architectural decisions and rationale
- **data-flow.md** — How data flows through the system
- **[CRITICAL-SEPARATIONS.md](#)** — Three fundamental separations: (1) Productionization vs Product, (2) Two execution streams, (3) Local-to-cloud parity
- **[anthropic-harness-pattern-extended.md](#)** — How Dev-House applies Anthropic's proven Harness pattern at three levels
- **[dev-house-operational-infrastructure.md](#)** — Where Dev-House itself executes (self-hosted vs cloud cost analysis)
- **[execution-streams-codex-vs-harness.md](#)** — Two separate streams: Harness orchestration vs Codex code generation (with separate worktrees)

### Getting Started
- **[GETTING-STARTED.md](#)** — 30-minute narrative explaining the complete Dev-House story from problem to solution

### Security (Critical)
- **[security/prompt-security.md](#)** — Three-phase security pipeline (discovery, dev, runtime)
- **[security/local-guard-implementation.md](#)** — Building a local LLM security guard

### Key Components
- **harness-architecture.md** — Harness orchestration engine design
- **codex-architecture.md** — Claude Codex integration points
- **deployment-architecture.md** — Customer deployment infrastructure

---

## Workflow Documents

### Process & Operations
- **[workflow/PROCESS-OVERVIEW.md](#)** — Complete cascade from discovery to delivery with inter-system sequence diagrams and intra-system process flows showing all 5 phases and 20+ breakdown points
- **[workflow/FAILURE-RECOVERY.md](#)** — Comprehensive failure handling across all phases with recovery procedures, git state recovery, and escalation matrix
- **[workflow/DAILY-OPERATIONS.md](#)** — Session-by-session operational procedures (Setup → Analysis → Codex → Features → Infrastructure → Delivery) with concrete steps and bash commands

---

## Harness Documents

### Core Concepts
- **orchestration.md** — Workflow engine, task scheduling, state management
- **api.md** — Harness API for customers
- **state-machine.md** — State transitions, error handling
- **[local-development-environment.md](#)** — Docker Compose parity (every production component has local equivalent)

### Patterns
- **retry-patterns.md** — Handling transient failures
- **caching-strategies.md** — Caching PRD analyses, generated code
- **validation-patterns.md** — Code validation before deployment

---

## Terraform Documents

### Multi-Provider Strategy
- **[terraform/multi-provider-strategy.md](#)** — AWS, Azure, GCP support with provider-specific module patterns
- **terraform/module-patterns.md** — Reusable module design principles
- **terraform/cost-optimization.md** — Multi-provider cost comparison

---

## Codex Documents

### Integration
- **generation.md** — How Claude Codex generates code from PRDs
- **templates.md** — Code templates for different infrastructure types
- **validation.md** — Validation pipelines

---

## OpenClaw Documents

### Integration
- **integration.md** — How OpenClaw orchestrates infrastructure deployment
- **workflows.md** — Workflow definitions and orchestration patterns
- **policies.md** — Policy enforcement and compliance
- **state-management.md** — State tracking and rollback

---

## Deployment Documents

### Architecture & Patterns
- **[deployment-patterns.md](#)** — Tier 1-4 patterns, pattern selection framework, decision matrix
- **[customer-repository-structure.md](#)** — PRD-driven decisions: separate repos vs monorepo, service organization
- **customer-deployment.md** — How customers deploy the harness

### Operations
- **[pattern-selection-workbook.md](#)** — **Practical guide for evaluating new PRDs and selecting deployment patterns** (10-min checklist, decision tree, weakness checking, template instantiation)
- **[custom-domain-provisioning.md](#)** — Custom domain setup, DNS, certificates, Keycloak, provisioning API
- **monitoring.md** — Monitoring, logging, observability
- **security.md** — Security best practices for customers
- **troubleshooting.md** — Common issues and fixes

---

## Patterns & Reusable Designs

### Harness Patterns
- **harness-patterns.md** — Reusable workflow patterns
- **task-templates.md** — Common task templates

### Codex Patterns
- **generation-patterns.md** — Common code generation patterns
- **error-handling.md** — Error recovery strategies

---

## How to Add New Docs

When you add a new document:
1. **Add a row to this index** (above)
2. **Link it from CLAUDE.local.md** if it's a major pattern
3. **Update the category section** (Architecture, Harness, etc.)

---

## Rule: Check Index Before Searching

Before `grep`-ing for something:
1. **Look in this index** — Is there already a doc for it?
2. **If yes** — Read the doc (fast, cached)
3. **If no** — Create a doc (don't just scatter knowledge)

This keeps knowledge centralized, discoverable, and session-cacheable.


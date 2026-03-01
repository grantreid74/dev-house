# Dev-House Documentation Index

**Start here to find the right doc.** This is your fridge — check it first before searching code.

---

## Quick Navigation

| What You Need | Document | Purpose |
|---------------|----------|---------|
| **Project overview** | [project-overview.md](project-overview.md) | 5-min intro: what Dev-House is and who it's for |
| **Project naming standards** | [naming-standards.md](naming-standards.md) | File naming patterns with decision router |
| **Doc-code sync map** | [documentation-maintenance.md](documentation-maintenance.md) | When you change X, update Y docs |
| **Content review policy** | [review-policy.md](review-policy.md) | Per-commit, per-session, quarterly review process |
| **Document lifecycle** | [archive-structure.md](archive-structure.md) | Transient → temporal → archived → removed |
| **GitHub as narrative** | [workflow/github-workflow.md](workflow/github-workflow.md) | Issues/PRs as single source of truth |
| **Edge cases & gotchas** | [gotchas.md](gotchas.md) | Things that don't work out of the box |
| **Ticket templates** | [Ticket Standards section ↓](#ticket-standards) | feature, bug, infra, refactor — selection guide |
| **Open questions** | [../CURRENT_REVIEW.md](../CURRENT_REVIEW.md) | Unresolved decisions and risks |
| **Deployment models** | [deployment/models/README.md](deployment/models/README.md) | G-series + D-series infrastructure selection guide |
| **Work dispatch architecture** | [architecture/work-dispatch-models.md](architecture/work-dispatch-models.md) | Coordinator-push vs worker-daemon pull — the key harness decision |
| **Model routing** | [architecture/model-router.md](architecture/model-router.md) | Which model (Haiku/Sonnet/Opus) handles which task type; subagent handover contract |

---

## Architecture Documents

### Foundation
- **[overview.md](architecture/overview.md)** — Layers (PRD input → Harness → Codex → Deployment): system components and how they connect
- **[decisions.md](architecture/decisions.md)** — Architectural decisions: D1 (Harness-first), D2 (local-to-cloud parity), and others with rationale
- **[critical-separations.md](architecture/critical-separations.md)** — Three fundamental separations: (1) Productionization vs Product, (2) Two execution streams, (3) Local-to-cloud parity
- **[execution-streams-codex-vs-harness.md](architecture/execution-streams-codex-vs-harness.md)** — Two separate streams: Harness orchestration vs Codex code generation (separate worktrees)

### Getting Started
- **[getting-started.md](architecture/getting-started.md)** — 30-minute narrative: complete Dev-House story from problem to solution

### Harness & Dispatch
- **[work-dispatch-models.md](architecture/work-dispatch-models.md)** — **Key decision**: Model A (coordinator-push) vs Model B (worker-daemon pull). Task schema, node capability config, queue options, harness implications.
- **[anthropic-harness-pattern-extended.md](architecture/anthropic-harness-pattern-extended.md)** — How Dev-House applies Anthropic's proven Harness pattern at three levels
- **[distributed-agent-deployment.md](architecture/distributed-agent-deployment.md)** — Multi-agent distributed execution vision: N "AI employee" agents working in parallel on isolated git branches
- **[model-router.md](architecture/model-router.md)** — Task type → model routing table (Haiku/Sonnet/Opus), subagent handover contract template, self-learning log

### Security
- **[security-integration.md](architecture/security-integration.md)** — How security integrates at every pipeline stage (not a separate layer)
- **[security/prompt-security.md](security/prompt-security.md)** — Three-phase security pipeline (discovery, dev, runtime)
- **[security/local-guard-implementation.md](security/local-guard-implementation.md)** — Building a local LLM security guard

### Operational Infrastructure
- **[dev-house-operational-infrastructure.md](architecture/dev-house-operational-infrastructure.md)** — Where Dev-House itself executes: self-hosted vs cloud cost analysis
- **[pattern-evolution-framework.md](architecture/pattern-evolution-framework.md)** — How to evaluate, strengthen, and grow the reference architecture as new PRDs arrive

---

## Deployment Documents

### Generation Infrastructure Models (G-series)

How Dev-House generates code — select based on hardware, GPU needs, and budget:

| Model | Doc | Summary |
|-------|-----|---------|
| G-S1 | [deployment/models/G-S1-standalone-pi.md](deployment/models/G-S1-standalone-pi.md) | Single Pi 5 16GB — Docker Compose, no cluster, pilot/entry (~$500 over 5yr) |
| G-L1 | [deployment/models/G-L1-pi-cluster.md](deployment/models/G-L1-pi-cluster.md) | Pi Cluster — k3s, no GPU, nomadic (~$2,000-2,400 over 5yr) |
| G-L2 | [deployment/models/G-L2-pi-mac-hybrid.md](deployment/models/G-L2-pi-mac-hybrid.md) | Pi + Mac Mini — optional Apple Silicon GPU |
| G-L3 | [deployment/models/G-L3-mac-mini-fleet.md](deployment/models/G-L3-mac-mini-fleet.md) | Mac Mini Fleet — GPU on all nodes, Docker-per-mini |
| G-C1 | [deployment/models/G-C1-cloud-burst.md](deployment/models/G-C1-cloud-burst.md) | Cloud Burst — pay-per-use, NVIDIA GPU, stateless workers |

See **[deployment/models/README.md](deployment/models/README.md)** for full selection dimensions and customer type mapping.

### G-L1 Extended Docs (Pi Cluster specific)

- **[deployment/models/G-L1/cluster-topology.md](deployment/models/G-L1/cluster-topology.md)** — Node roles, RAM budgets, NVMe storage, k3s layout, expansion options
- **[deployment/models/G-L1/testing-deployment-pattern.md](deployment/models/G-L1/testing-deployment-pattern.md)** — Cluster + desktop GPU testing solution: 3 docker-compose variants, NAS as transport, overnight batch pattern

### Customer Deployment Patterns (D-series)

What the customer's production system looks like:

- **[deployment/deployment-patterns.md](deployment/deployment-patterns.md)** — Tier 1-4 patterns, pattern selection framework, decision matrix
- **[deployment/customer-repository-structure.md](deployment/customer-repository-structure.md)** — PRD-driven decisions: separate repos vs monorepo, service organisation
- **[deployment/customer-deployment.md](deployment/customer-deployment.md)** — How customers deploy the harness
- **[deployment/pattern-selection-workbook.md](deployment/pattern-selection-workbook.md)** — 10-min checklist: evaluate new PRDs, select deployment pattern, check weaknesses
- **[deployment/custom-domain-provisioning.md](deployment/custom-domain-provisioning.md)** — Custom domain setup, DNS, certificates, Keycloak, provisioning API

---

## Workflow Documents

- **[workflow/01_process-overview.md](workflow/01_process-overview.md)** — Complete cascade from discovery to delivery: all 5 phases, 20+ breakdown points, inter-system sequence diagrams
- **[workflow/02_daily-operations.md](workflow/02_daily-operations.md)** — Session-by-session operational procedures with concrete steps and bash commands
- **[workflow/03_failure-recovery.md](workflow/03_failure-recovery.md)** — Failure handling across all phases: recovery procedures, git state recovery, escalation matrix
- **[workflow/github-workflow.md](workflow/github-workflow.md)** — GitHub as project narrative: issues for why, PRs for how

---

## Harness Documents

- **[harness/orchestration.md](harness/orchestration.md)** — Workflow engine, task scheduling, state management
- **[harness/local-development-environment.md](harness/local-development-environment.md)** — Docker Compose parity: every production component has a local equivalent
- **[patterns/harness-patterns.md](patterns/harness-patterns.md)** — Reusable harness workflow patterns

*Planned (not yet created)*: `harness/api.md`, `harness/state-machine.md`

---

## Terraform Documents

- **[terraform/multi-provider-strategy.md](terraform/multi-provider-strategy.md)** — AWS, Azure, GCP support with provider-specific module patterns

*Planned (not yet created)*: `terraform/module-patterns.md`, `terraform/cost-optimization.md`

---

## Codex Documents

- **[codex/generation.md](codex/generation.md)** — How Claude Codex generates code from PRDs

*Planned (not yet created)*: `codex/templates.md`, `codex/validation.md`

---

## OpenClaw Documents

> **Note**: "OpenClaw" in Dev-House refers to the internal infrastructure orchestration layer (Terraform execution, deployment automation, policy enforcement). This is unrelated to the Clawdbot consumer product (Anthropic TOS incident, Feb 2026). See CURRENT_REVIEW.md Section 8.

- **[openclaw/integration.md](openclaw/integration.md)** — How the OpenClaw layer orchestrates infrastructure deployment

*Planned (not yet created)*: `openclaw/workflows.md`, `openclaw/policies.md`, `openclaw/state-management.md`

---

## Research Documents

Point-in-time investigations. Frozen when written — never updated in place.

### Cross-Model Research

- **[research/20260301_OPUS_claude-flow-convergence.md](research/20260301_OPUS_claude-flow-convergence.md)** — claude-flow vs Claude Code native: ~70-80% convergence, v3 broken, skip it. Decision: skip.
- **[research/20260302_OPUS_cae-vs-kubernetes.md](research/20260302_OPUS_cae-vs-kubernetes.md)** — Serverless containers (ACA, Cloud Run, Fargate) vs managed K8s (AKS, EKS, GKE). Decision: serverless containers correct default for Claude API-bound workloads; K8s justified at ~15-20 services.
- **[research/G-L1/20260301_OPUS_harness-framework-comparison.md](research/G-L1/20260301_OPUS_harness-framework-comparison.md)** — Full AI orchestration framework evaluation: Anthropic, OpenAI, open-source, cloud. Recommended stack, master matrix, Claudbot incident.

### G-L1 Pi Cluster Research

- **[research/G-L1/20260228_OPUS_Pi5-Real-Data-Analysis.md](research/G-L1/20260228_OPUS_Pi5-Real-Data-Analysis.md)** — Pi 5 capacity analysis from real hardware data: per-node RAM, CPU load, API bottleneck confirmation
- **[research/G-L1/20260228_OPUS_portable-gpu-concurrency-analysis.md](research/G-L1/20260228_OPUS_portable-gpu-concurrency-analysis.md)** — GPU expansion options, cloud GPU costs (Opus — canonical version)
- **[research/G-L1/20260228_1430_local-hardware-cost-analysis.md](research/G-L1/20260228_1430_local-hardware-cost-analysis.md)** — Hardware cost breakdown analysis

*Also in G-L1/: Sonnet and Haiku variants of the GPU analysis — superseded by the Opus version above.*

---

## Ticket Standards

Templates for autonomous agent implementation. Each ticket must be self-contained — zero session context, full provenance. Version embedded in each file header; git recovers any past version.

### Template Selection

| Work type | Template | Version |
|-----------|----------|---------|
| New functionality, endpoint, service, integration | [standards/feature.md](standards/feature.md) | 1.0.0 |
| Incorrect behavior, regression, silent error | [standards/bug.md](standards/bug.md) | 1.0.0 |
| Terraform, Docker, K8s, CI/CD, cluster | [standards/infra.md](standards/infra.md) | 1.0.0 |
| Code improvement, DRY cleanup, rename | [standards/refactor.md](standards/refactor.md) | 1.0.0 |

### Provenance Tuple (in every ticket)

```
Template: standards/feature.md v1.0.0
Harness:  anthropic-harness-v1 | openai-agents-v1 | manual
PRD:      examples/<customer-id>.md § <section-name>
Created:  YYYY-MM-DD
```

### Versioning Rules
- Version lives in each file's header — templates evolve independently
- To update: edit file, bump semver, add changelog entry
- When a version bumps: review all open tickets using the previous version before implementing

---

## How to Add New Docs

1. Add a row to this index
2. Link from CLAUDE.local.md if it's a major pattern
3. Add to the relevant category section above

## Rule: Check Index Before Searching

Before `grep`-ing for something:
1. Look in this index — Is there already a doc for it?
2. If yes — read the doc
3. If no — create a doc (don't scatter knowledge)

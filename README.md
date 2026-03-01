# Dev-House

**AI automation framework that converts business requirements (PRDs) into running applications and infrastructure.**

You write what you want built. Dev-House figures out the architecture, generates the code, provisions the infrastructure, and hands you a running system.

---

## How to Read This Repo

There is a lot of documentation. Start here and follow the path — don't try to read everything.

### Step 1 — Understand what this is (5 min)

```
docs/project-overview.md          ← what Dev-House is and who it's for
```

**Then read `CURRENT_REVIEW.md` — but know where it sits:**

```
PRD → Architecture → Tickets → Generation → Validation → Deployment
                         ↑
               CURRENT_REVIEW.md lives here
```

CURRENT_REVIEW.md is not a random list of questions. It tracks the decisions that must be made *before the pipeline can be trusted*: what does the customer receive, how is a PRD broken into work, what does "production ready" mean, how are credentials handled. Until those are answered, the pipeline exists in design only. Reading it tells you exactly what is unbuilt and why.

### Step 2 — Understand how it works (30 min)

```
docs/architecture/getting-started.md          ← the full story, start to finish
docs/architecture/critical-separations.md     ← three things you must not conflate:
                                                  1. Dev-House infrastructure vs customer product
                                                  2. Harness stream vs Codex stream
                                                  3. Local Docker vs cloud deployment
docs/architecture/decisions.md                ← why it's built this way
```

### Step 3 — Understand the pipeline (PRD → running system)

```
Customer PRD (business requirements)
    │
    ▼
docs/architecture/anthropic-harness-pattern-extended.md   ← how Harness analyses the PRD
    │                                                          and decides what to build
    ▼
docs/standards/feature.md   ← tickets generated per requirement
docs/standards/infra.md        (each ticket is self-contained: what, why, acceptance criteria,
docs/standards/bug.md           docs to read, PRD provenance)
docs/standards/refactor.md
    │
    ▼
docs/architecture/work-dispatch-models.md    ← how tickets reach worker nodes
                                                (coordinator-push vs worker-daemon pull)
    │
    ▼
docs/deployment/models/README.md    ← which hardware runs the generation
    │
    ├── docs/deployment/models/G-S1-standalone-pi.md     (1 Pi, pilot)
    ├── docs/deployment/models/G-L1-pi-cluster.md        (Pi cluster, no GPU)
    ├── docs/deployment/models/G-L2-pi-mac-hybrid.md     (Pi + Mac Mini, optional GPU)
    ├── docs/deployment/models/G-L3-mac-mini-fleet.md    (Mac Mini fleet, GPU on all nodes)
    └── docs/deployment/models/G-C1-cloud-burst.md       (cloud, pay-per-use)
    │
    ▼
Generated code → GitHub → customer review
    │
    ▼
docs/deployment/deployment-patterns.md    ← how the customer's system is provisioned
                                              (Terraform, cloud infrastructure, isolation tiers)
```

### Step 4 — Understand the operational detail (as needed)

```
docs/workflow/01_process-overview.md       ← all 5 phases, end to end
docs/workflow/02_daily-operations.md       ← session-by-session procedures
docs/workflow/03_failure-recovery.md       ← when things go wrong

docs/security/prompt-security.md          ← three-phase security pipeline
docs/architecture/work-dispatch-models.md ← worker daemon architecture (Model B)
```

---

## Repository Structure

```
dev-house/
│
├── README.md                    ← you are here
├── CURRENT_REVIEW.md            ← what must be decided before the pipeline is trustworthy
│                                   (pre-flight checklist: product definition, harness design,
│                                    validation criteria, Terraform ownership, security gaps)
│
└── docs/
    ├── README.md                ← full documentation index (check before searching)
    │
    ├── project-overview.md      ← 5-min intro
    │
    ├── architecture/            ← how the system is designed
    │   ├── getting-started.md
    │   ├── critical-separations.md
    │   ├── decisions.md
    │   ├── work-dispatch-models.md     ← key harness architecture decision
    │   ├── anthropic-harness-pattern-extended.md
    │   ├── distributed-agent-deployment.md
    │   ├── execution-streams-codex-vs-harness.md
    │   ├── dev-house-operational-infrastructure.md
    │   ├── pattern-evolution-framework.md
    │   ├── security-integration.md
    │   └── overview.md
    │
    ├── deployment/
    │   ├── models/              ← pick your generation hardware (G-series)
    │   │   ├── README.md              ← selection guide (start here)
    │   │   ├── G-S1-standalone-pi.md
    │   │   ├── G-L1-pi-cluster.md
    │   │   │   └── G-L1/              ← extended Pi cluster docs
    │   │   │       ├── cluster-topology.md
    │   │   │       └── testing-deployment-pattern.md
    │   │   ├── G-L2-pi-mac-hybrid.md
    │   │   ├── G-L3-mac-mini-fleet.md
    │   │   └── G-C1-cloud-burst.md
    │   ├── deployment-patterns.md     ← customer system tiers (D-series)
    │   ├── pattern-selection-workbook.md
    │   └── customer-deployment.md
    │
    ├── standards/               ← ticket templates for agent implementation
    │   ├── feature.md
    │   ├── bug.md
    │   ├── infra.md
    │   └── refactor.md
    │
    ├── workflow/                ← operational procedures
    │   ├── 01_process-overview.md
    │   ├── 02_daily-operations.md
    │   ├── 03_failure-recovery.md
    │   └── github-workflow.md
    │
    ├── security/
    │   ├── prompt-security.md
    │   └── local-guard-implementation.md
    │
    ├── research/                ← frozen point-in-time analysis
    │   ├── 20260301_OPUS_claude-flow-convergence.md
    │   └── G-L1/               ← Pi cluster specific research
    │
    ├── harness/                 ← orchestration engine docs
    ├── codex/                   ← code generation docs
    ├── terraform/               ← multi-provider IaC strategy
    └── gotchas.md               ← things that don't work as expected
```

---

## Two Things to Keep Separate

The single most important concept in this repo:

| | Generation Infrastructure | Customer Deployment |
|--|--------------------------|---------------------|
| **What** | Where Dev-House runs its agents | What the customer's system looks like |
| **Models** | G-S1, G-L1, G-L2, G-L3, G-C1 | D-T3, D-T4, D-BYOC |
| **Who owns it** | You (Dev-House) | The customer |
| **Selected by** | Your hardware and budget | Customer requirements and compliance |
| **Doc** | `docs/deployment/models/` | `docs/deployment/deployment-patterns.md` |

These are chosen independently. A customer on D-T4 (full isolation, HIPAA) can have their code generated on a G-S1 (single Pi). The two decisions don't touch each other.

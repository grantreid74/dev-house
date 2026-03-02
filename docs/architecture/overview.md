# Dev-House System Architecture Overview

## Purpose

Enable customers to build AI-assisted infrastructure and applications through **PRD-based development** using Anthropic's proven Harness pattern.

Customers provide business specifications (PRD) → Dev-House analyzes → Codex generates → OpenClaw deploys → Customer has running system.

---

## Core Pattern: Anthropic's Harness Extended

We apply [Anthropic's Harness pattern](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) at three levels:

```mermaid
graph TD
    PRD["📋 Customer PRD<br/>(Business Requirements)"]

    HARNESS["⚙️ Harness Orchestration<br/>(Workflow Engine)<br/>─<br/>Initializer: Analyze PRD<br/>→ ARCHITECTURAL_DECISION.md<br/>→ feature_list (0/200 done)<br/>─<br/>Analyzer: Each session<br/>→ Dispatch one decision to agents"]

    CODEX["🔵 Code Generation<br/>(Claude Code + OpenAI Codex)<br/>Two concurrent agents per node<br/>─<br/>Initializer: Generate services<br/>→ baseline code + features<br/>─<br/>Generators: Each session<br/>→ Implement one feature<br/>→ Test → Commit → Progress"]

    OPENCLAW["🔧 OpenClaw Deployment<br/>─<br/>Initializer: Generate Terraform<br/>→ terraform/ skeleton<br/>─<br/>Orchestrator: Each session<br/>→ Provision one component<br/>→ Validate → Apply → Track"]

    RUNNING["✅ Running Customer<br/>Infrastructure"]

    PRD --> HARNESS
    HARNESS --> CODEX
    CODEX --> OPENCLAW
    OPENCLAW --> RUNNING

    style PRD fill:#e1f5ff
    style HARNESS fill:#f3e5f5
    style CODEX fill:#fff3e0
    style OPENCLAW fill:#e8f5e9
    style RUNNING fill:#c8e6c9
```

**Key insight**: All three layers use **Initializer → Progress files → Incremental work** pattern.

See: [Anthropic Harness Pattern Extended](anthropic-harness-pattern-extended.md)

---

## Key Components

### 1. Harness (Orchestration Engine)
- **Input**: PRD document (business requirements)
- **Output**: Task queue for code generation agents + progress tracking
- **Responsibility**:
  - Parse PRD into structured requirements
  - Dispatch tasks to code generation agents (Claude Code or OpenAI Codex)
  - State tracking (what's done, what's next, what failed)
  - Error recovery
  - Customer feedback loops

### 2. Code Generation (Dual Provider Model)

Each AI employee node runs two concurrent code generation agents — both implement tickets, both are treated as equivalent and interchangeable:

- **Claude Code** (Anthropic, ~$200/month) — agentic code generation via Claude Code CLI; file operations, tool use, long-horizon task execution
- **OpenAI Codex** (~$200/month) — agentic code generation via OpenAI Codex; equivalent capability, different provider

**Why two providers?** Provider redundancy — if one goes down, the other keeps working. No vendor lock-in. Cost is ~$400/month + electricity per AI employee node.

Model routing (Haiku/Sonnet/Opus tier) happens within each provider independently. See `docs/architecture/model-router.md`.

**Patterns**: Reusable prompts for common scenarios (REST API, database, auth, etc.)

### 3. OpenClaw (Infrastructure Orchestration)

> **Naming note**: "OpenClaw" here is Dev-House's internal infrastructure orchestration layer — Terraform execution, deployment automation, policy enforcement. This is not related to Clawdbot (the consumer messaging product that had the Anthropic TOS incident, Feb 2026). See CURRENT_REVIEW.md Section 8.
- **Workflow Orchestration**: Execute multi-step deployments, coordinate provisioning
- **Infrastructure Provisioning**: Terraform, CloudFormation, Ansible, cloud provider CLIs
- **State Management**: Track infrastructure state, enable rollback
- **Policy & Compliance**: Enforce tagging, security, cost limits
- **Cost Optimization**: Track spending, recommend rightsizing

### 4. Deployment Engine
- **Docker**: Container image builds
- **Infrastructure**: Cloud provisioning via OpenClaw
- **Application Deployment**: Service configuration, orchestration
- **Validation**: Automated testing, security scanning, health checks

### 5. State Management
- PRD analysis cache (don't re-analyze unchanged parts)
- Generated code artifacts
- Deployment history
- Infrastructure state (via OpenClaw)
- User feedback / refinements

---

## Data Flow Example

**Scenario**: Customer provides PRD for "REST API with PostgreSQL"

```mermaid
sequenceDiagram
    participant Customer
    participant Harness
    participant CodeAgent as Code Agent<br/>(Claude Code or OpenAI Codex)
    participant OpenClaw
    participant Cloud

    Customer->>Harness: 1. Provide PRD
    Harness->>Harness: 2. Analyze PRD, decide architecture<br/>(may call AI for reasoning)
    Harness->>CodeAgent: 3. Dispatch task: Generate REST API code
    CodeAgent-->>Harness: Generated code committed
    Harness->>Harness: 4. Validate code (lint, type check)
    Harness->>CodeAgent: 5. Dispatch task: Generate Terraform
    CodeAgent-->>Harness: Terraform code committed
    Harness->>Harness: 6. Validate Terraform
    Harness->>OpenClaw: 7. Deploy infrastructure and app
    OpenClaw->>Cloud: Provision resources
    Cloud-->>OpenClaw: Resources provisioned
    OpenClaw->>OpenClaw: Run smoke tests
    OpenClaw-->>Harness: Deployment complete
    Harness-->>Customer: 8. Access URL: https://api.example.com
```

---

## Design Constraints

- **Productionizable**: Not just a prototype. Customers run this themselves.
- **Cost-aware**: Minimize Claude API calls through caching and batching
- **Debuggable**: Clear logs, error messages, recovery options
- **Extensible**: New code generators, deployment targets added without harness rewrite

---

## Deployment Models

### Model 1: SaaS (Cloud)
- Harness runs on cloud infrastructure (customer-managed VPC)
- Code generation agents (Claude Code + OpenAI Codex) run on cloud nodes
- Deployment targets: customer's cloud (AWS, Azure, GCP)

### Model 2: Customer Self-Hosted
- Harness container runs in customer's infrastructure
- Customer manages secrets, API keys
- Deployment targets: customer's on-prem or cloud

---

## Key Design Decisions

*See [decisions.md](decisions.md) for detailed rationale.*

1. **Harness-first approach** — Orchestration before code generation
2. **State-based workflow** — Track progress, enable recovery
3. **Modular deployment** — Different generators for different targets
4. **Caching strategy** — Avoid re-analyzing unchanged PRDs

---

## Next: Detailed Components

- [Harness Orchestration Engine](../harness/orchestration.md)
- [Code Generation Integration](../codex/generation.md)
- [Deployment Architecture](../deployment/customer-deployment.md)


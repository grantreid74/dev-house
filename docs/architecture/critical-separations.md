# CRITICAL ARCHITECTURAL SEPARATIONS

**This document captures the three fundamental separations that prevent conflating vastly different concerns.**

---

## Separation 1: Productionization ≠ Product Creation

### What Dev-House Builds

We create **two entirely different systems**:

**A) Productionization Infrastructure** (Our operational cost)
- Where our Claude + Codex agents run
- Orchestration harness, API calls, Terraform generation
- Cost: $200-600/month depending on execution model
- Ownership: Dev-House operates this
- Scaling: Scales with customer volume

**B) Customer Product Infrastructure** (What we deliver)
- Where customer's application runs
- Tier 1-4 deployment patterns (Container Apps, K8s, VMs, etc.)
- Cost: $25-500+/month (customer pays directly — their cloud account)
- Ownership: Customer operates after handover
- Scaling: One per customer

### Why This Matters

```
WRONG:                          RIGHT:
Pricing Analysis                Pricing Analysis
├── Cloud VM costs              ├── A) Our operational cost ($400-600/mo)
└── Total monthly cost          └── B) Customer deployment cost ($100-300/mo)
                                     → Total customer cost = A shared + B dedicated
```

**The mistake**: Running our orchestration in expensive cloud VMs ($0.08/hour) when it could run on self-hosted Mac minis ($18/month electricity).

### Cost Strategy

**A) Where We Execute Orchestration** (Optimization target):
- Option 1: Azure Container Apps (ACA): $948/year
- Option 2: Self-hosted (5 Mac minis): $4,308 Y1, $2,808 Y3+ (better long-term)
- **Recommendation**: Hybrid (Self-hosted dev + minimal cloud prod)

**B) Where Customers Deploy** (Less cost-sensitive):
- Tier 1 ($25-100/mo): No need to optimize for us; customer willing to pay
- Tier 3 ($100-300/mo): Good cost for compliance + isolation
- Tier 4 ($300-500+/mo): Enterprise; not cost-sensitive

---

## Separation 2: Two Execution Streams (Harness vs Codex)

### Stream 1: Harness Orchestration (Workflow Engine)

```
PRD Analysis & Architectural Decisions
├── Analyze customer profile (budget, team, compliance)
├── Decompose into services (FE, BE, etc.)
├── Select deployment pattern (Tier 1-4)
├── Plan repository structure (separate repos vs monorepo)
└── Document architectural decision → dispatch to code generation agents
    ↓
    Commits to: decision/[customer-id] branch
    Output: ARCHITECTURAL_DECISION.md, SERVICE_DECOMPOSITION.yaml, etc.
```

**Worktree**: `harness/[customer-id]/`

**Tools**: Harness orchestration engine; may call Claude or OpenAI for reasoning (PRD analysis). Not itself an AI subscription.

**Timeline**: Day 1-2 (analysis phase)

**Cost**: ~$0.30-0.50 per PRD (AI API calls for reasoning — small volume)

### Stream 2: Codex Code Generation (Dual Provider)

```
Code & Infrastructure Generation
├── Read Harness architectural decisions
├── Generate frontend service (React/Vue/etc.)
├── Generate backend service (Python/Go/etc.)
├── Generate Terraform infrastructure code
├── Generate CI/CD pipelines
├── Run security + quality checks
└── Create PRs to customer repositories
    ↓
    Commits to: feature/[customer-id]-initial branches
    Output: Customer repos populated with code
```

**Worktree**: `codex/[customer-id]/` for coordination, then customer repos for actual code

**Tools**: Codex Agent, GitHub for PR workflow, docker-compose for local validation

**Timeline**: Day 3-7 (generation phase)

**Cost**: ~$0.30-0.60 per PRD (Codex API calls + validation)

### Why They Must Be Separate

| Aspect | Harness | Codex | Issue if Merged |
|--------|---------|-------|-----------------|
| **Output** | Architecture plan | Running code | Mixing code with decision artifacts |
| **Token allocation** | Harness reasoning calls (low volume) | Code generation (Claude Code + Codex, high volume) | Can't split budget or track ROI |
| **Parallel work** | One per customer | One per service | Customer A's Harness blocks Customer B's Codex |
| **Debugging** | "Architecture is wrong" | "Generated code is broken" | Can't isolate root cause |
| **Git strategy** | Branch per PRD | Branch per service | Merge conflicts across concerns |

**Mistake**: Running both in same worktree leads to:
- Codex overwriting Harness decision files
- Merged commits with mixed concerns
- Impossible to track which agent produced which output
- Can't replay Codex alone for regeneration

### Worktree Layout

```
dev-house/
├── harness/
│   └── acme-corp/                  # Stream 1 (Harness analysis)
│       ├── ARCHITECTURAL_DECISION.md
│       ├── SERVICE_DECOMPOSITION.yaml
│       └── .git (points to planning/acme-corp branch)
│
├── codex/
│   └── acme-corp/                  # Stream 2 (Codex coordination)
│       ├── GENERATION_LOG.md
│       ├── VALIDATION_REPORT.yaml
│       └── .git (points to codex/acme-corp/main branch)
│
└── (main worktree)
```

**Customer service repos** (separate GitHub organizations):
```
github.com/acme-corp/
├── frontend.git               # Generated by Codex
├── backend.git                # Generated by Codex
└── infrastructure.git         # Generated by Codex
```

---

## Separation 3: Local Dev = Cloud Production (Parity)

### The Principle

Every component in production has a local equivalent:

```
LOCAL DEVELOPMENT              PRODUCTION (Tier 3 example)
────────────────────────────  ──────────────────────────────
docker compose up              Terraform apply
├── backend (Python)           ├── Container App (Python)
├── frontend (React)           ├── CDN + Static Assets
├── postgres                   ├── Azure SQL Database
└── redis                      └── Azure Cache for Redis
```

### Why This Matters

**Debugging in production is expensive:**
- Can't edit code live (cold start)
- Logging latency (cloud logs take seconds)
- Pricing: Every minute of troubleshooting costs money

**Debugging locally is free:**
- Edit, save, hot reload (seconds)
- Logs immediate
- No cloud charges

### Docker Compose Strategy

**Every service repo has `compose.yaml`:**

```yaml
# backend/compose.yaml (local)
services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://db:5432/app
      - REDIS_URL=redis://cache:6379

  db:
    image: postgres:15
    # Local postgres

  cache:
    image: redis:7
    # Local redis
```

Maps to production:
| Local Component | Production (Tier 3) |
|-----------------|-------------------|
| postgres:15 | Azure SQL Database (managed) |
| redis:7 | Azure Cache for Redis (managed) |
| backend (Python container) | Container App (managed) |

**Codex generates both** when creating services:
- `docker-compose.yaml` (local)
- `terraform/main.tf` (cloud)

Same Dockerfile runs everywhere:
```dockerfile
FROM python:3.11-slim
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src ./src
CMD ["uvicorn", "src.main:app"]
```

- Local: `docker compose up` → Python container running locally
- Cloud: Same image pushed to Container Registry → Container App pulls it

### Consequences of Parity Violation

**If local ≠ production:**
- Developer uses `localhost:3000` (works locally, breaks in Azure)
- Connection pool sizes differ (fast local, slow in cloud)
- Disk space assumptions differ (large disk locally, limited in container)
- Timing assumptions break (Redis latency)

**Cost of discovery: $$$**
- Bug found in production costs money to diagnose
- Fix requires cloud redeployment (slow iteration)
- Customer visible (reputational damage)

---

## Execution Model: The 24/7 Pattern

### How Dev-House Runs (Self-Hosted + Tailscale)

```
Office Mac minis (always on)
├── Mac mini 1: Harness coordinator (orchestration engine)
├── Mac mini 2: Claude Code agent + OpenAI Codex agent (gen services, concurrent)
└── Mac mini 3: Claude Code agent + OpenAI Codex agent (gen Terraform, concurrent)

Contributor devices (on-demand)
├── Laptop 1: Claude Code agent + OpenAI Codex agent (gen frontend)
├── Laptop 2: Claude Code agent + OpenAI Codex agent (gen backend)
└── Personal devices: Underutilized capacity

All connected via Tailscale (single virtual LAN)
```

**Network model:**
```
Device 1          Device 2          Device 3          Cloud
┌────────┐        ┌────────┐        ┌────────┐        ┌────────┐
│Harness │        │Codex-1 │        │Codex-2 │        │Customer│
│ (Mac)  │        │(Laptop)│        │(Mac)   │        │ Infra  │
└────┬───┘        └────┬───┘        └────┬───┘        └────┬───┘
     │                 │                 │                 │
     └─────────────────┼─────────────────┼─────────────────┘
            Tailscale VPN (unified network)
```

**Device capacity management:**
```yaml
# Each device publishes status (to shared Redis)
mac-mini-1:
  cores: 8
  available_capacity: 6
  current_agents: 2
  load: 45%

laptop-1:
  cores: 4
  available_capacity: 3
  current_agents: 1
  load: 20%
```

**Harness coordinator schedules jobs:**
- "Codex job 1" → `laptop-1` (more idle)
- "Codex job 2" → `laptop-2` (also idle)
- "Harness analysis" → `mac-mini-1` (always available)

### Token Management (Each Device Gets Own OAuth)

```yaml
device: mac-mini-1
tokens:
  claude_code: oauth_[mac-mini-1_claude_v2]       # Anthropic Claude Code subscription
  openai_codex: oauth_[mac-mini-1_codex_v2]       # OpenAI Codex subscription
  # Both code generation agents run concurrently on this node

device: laptop-1
tokens:
  claude_code: oauth_[laptop-1_claude_v2]
  openai_codex: oauth_[laptop-1_codex_v2]
```

**Prevents:**
- Token overload (one account bottlenecked)
- Cost attribution loss (can't tell who spent what)
- Quota sharing (one user blocks another)

---

## Decision Framework: How Separations Guide Choices

### When evaluating a new PRD:

1. **Decide A (Productionization)** — Will self-hosted + Tailscale handle volume?
   - If yes → keep self-hosted (saves cost)
   - If no → scale to cloud K8s cluster

2. **Plan B (Product)** — Which deployment pattern for customer?
   - PRD compliance needs → Select Tier 1-4
   - Customer cost constraints → Adjust within tier
   - Geography → Multi-region or single?

3. **Design execution** — How do we build the product?
   - **Stream 1**: Harness analyzes (what services, what pattern, what repos)
   - **Stream 2**: Codex generates (code + terraform + CI/CD)
   - Separate worktrees to avoid conflict

4. **Ensure parity** — Local dev works exactly like production
   - Every service has docker-compose.yaml
   - Codex generates same Dockerfile for both local and cloud
   - Engineer catches bugs locally, not in production

---

## See Also

- **[dev-house-operational-infrastructure.md](dev-house-operational-infrastructure.md)** — Cost analysis for (A)
- **[execution-streams-codex-vs-harness.md](execution-streams-codex-vs-harness.md)** — Detailed workflow for separating streams
- **[customer-repository-structure.md](../deployment/customer-repository-structure.md)** — How (B) repos are organized
- **[local-development-environment.md](../harness/local-development-environment.md)** — Parity implementation
- **[deployment-patterns.md](../deployment/deployment-patterns.md)** — Tier 1-4 customer options

# Dev-House Patterns & Insights

Session-persistent knowledge base for recurring patterns, architecture insights, and security learnings.

**Quick Links (Session Reference)**:
- **[Naming Standards Router](NAMING-STANDARDS.md)** — Quick reference for deciding file naming patterns
- **Naming Standards**: See `docs/NAMING-STANDARDS.md` (also cached in memory)
- **Review Policy**: See `docs/REVIEW_POLICY.md` (per-commit, per-session, quarterly reviews)
- **Doc-Code Sync Map**: See `docs/DOCUMENTATION_MAINTENANCE.md` (what to update when you change X)
- **Gotchas**: See `docs/GOTCHAS.md` (things that don't work out of the box)

## Foundation Session (2026-02-28)

### Architecture Decisions (Foundation)

1. **Harness-first approach** — Orchestrate entire workflow, validate intermediates
2. **State-based deployment** — Track progress, enable recovery
3. **Hybrid Claude models** — Sonnet (routine), Opus (architecture), Haiku (validation)
4. **OpenClaw orchestration** — Infrastructure automation, policy enforcement, cost tracking
5. **Three-phase security** — Discovery (clean), Development (validate), Runtime (monitor)

### Security Architecture (Critical)

**Three-phase security pipeline**:
1. **Phase 1 (Discovery)** — Clean customer PRD (fix typos, detect injection)
2. **Phase 2 (Development)** — Validate generated prompts before sending to Claude
3. **Phase 3 (Runtime)** — Monitor Claude responses, track anomalies

**Local Security Guard**:
- Pattern matching (1ms) — Known attacks
- Fast classification (5ms) — BERT-based safe/unsafe
- Detailed analysis (100ms) — Mistral 7B intent verification

**Key insight**: Customers make honest typos. Security must distinguish attacks from mistakes.

### Documentation Structure

**Caching strategy**:
- CLAUDE.local.md — Stays <200 lines, project patterns + links
- docs/README.md — Index/catalog (fridge pattern)
- docs/ subdirs — Detailed, always-current documentation
- Memory files — Session-specific discoveries

**Benefit**: Minimal main context (~50KB), full knowledge accessible via links + indexes

### Mermaid Diagrams Policy

All diagrams use Mermaid (flowcharts, sequences, state machines). Never ANSI art. Benefits:
- Renderable in GitHub, browsers, markdown viewers
- Updatable via text (version control friendly)
- Composable (build complex diagrams from simple ones)

---

## Token Efficiency Insights

*(To be filled in as we explore)*

- Session caching: Index-based navigation saves ~40-60% context vs. full content
- Mermaid diagrams: Compact, reusable, no ANSI encoding overhead

## Implementation Patterns

*(To be filled in as we implement)*

## Common Mistakes / Gotchas

- **Treating all unusual input as attacks** — Users make honest typos (Phase 1 fixes them)
- **Prompt injection is subtle** — Pattern matching catches obvious attacks; behavioral analysis catches subtle ones
- **Claude can be jailbroken** — Validate responses, not just prompts (Phase 2b)
- **Cost of security** — Phase 1+2 add ~100ms; Phase 3 is background. Worth it.

## Custom Domain Provisioning Pattern

**How customers get custom domains** (e.g., acme.dev-house.io or app.acme.com):

1. **Create DNS Records** (CNAME + TXT validation)
   - Shared zone: Dev-House creates automatically
   - Customer zone: Customer creates via their registrar

2. **Bind Domain to Container App** (async)
   - Azure triggers automatic Let's Encrypt certificate provisioning
   - Takes 5-15 minutes

3. **Poll Certificate Status** (hybrid checking)
   - Check hostname binding status (SniEnabled = ready)
   - Check certificate provisioning state (Succeeded = ready)
   - Don't block on timeout—log and continue

4. **Update Customer Configuration**
   - Store FQDN in metadata database
   - Configure Keycloak realm with custom domain OAuth2 client
   - Update container app environment variables

**Operational Challenges**:
- Azure async delays (5-15 min, no polling API)
- Solution: Non-blocking timeout + log warning
- Pending-delete windows (5 min after cleanup before re-create)
- Solution: Wait before rebinding

**Identity Provider**: Keycloak (not Auth0)
- Single Keycloak instance (auth-int.dev-house.io)
- One realm per customer
- Each realm has OAuth2 client pointing to customer's custom domain

**Reference**: docs/deployment/custom-domain-provisioning.md

## Provisioning API Architecture

**Current**: CLI tool in Container App (provisioning-tools)
- Requires VPN + kubectl access
- Synchronous (no progress updates)
- No audit trail

**Proposed**: REST API + Job Queue
- FastAPI wrapper inside VNET
- Redis job queue for async provisioning
- Keycloak authentication on all endpoints
- Real-time progress logging
- Full audit trail (who, when, what)

**Endpoints**:
- `POST /api/v1/customers/{customer_id}/provision` → Returns job_id
- `GET /api/v1/jobs/{job_id}` → Returns status + logs
- `GET /api/v1/customers/{customer_id}` → Returns provisioning state
- `POST /api/v1/customers/{customer_id}/deprovision` → Queue deprovision job

**Architecture**:
```
Web Dashboard → HTTPS/VNET → FastAPI Provisioning API
                              ↓
                         Redis Job Queue
                              ↓
                    Provisioning Workers
                    (runs provision_*.py)
```

**Security**:
- Keycloak JWT authentication required
- API runs inside VNET (no public access)
- Rate limiting (10/min per user)
- RBAC (admin vs customer scopes)
- Full audit logging

**Reference**: docs/deployment/custom-domain-provisioning.md (Provisioning API Architecture section)

---

## Deployment Pattern Types (Tier 1-4)

**Framework for selecting customer deployment architecture based on profile:**

1. **Tier 1 (Fully Shared)**: All customers in same infrastructure ($25-100/mo per customer)
   - Use: Startups, free tiers, MVPs
   - Pattern: Shared Container Apps, schema-per-customer database

2. **Tier 2 (K8s Namespace)**: Shared cluster, namespace-per-customer isolation ($50-200/mo)
   - Use: SMBs, compliance-light, moderate scale
   - Pattern: Kubernetes namespace with network policies

3. **Tier 3 (Isolated Infrastructure)** ← **RECOMMENDED FOR MVP**
   - Use: SMB SaaS, mid-market, compliance-required ($50-200/mo per customer)
   - Pattern: Dedicated Container Apps + VNet per customer, shared Foundation (DB, cache, identity)
   - Proven in CompylotAI-terraform ✅
   - Supports scale-to-zero, custom domains, network isolation

4. **Tier 4 (Enterprise Isolated)**: Dedicated database+cache per customer ($300-500+/mo)
   - Use: Enterprise, HIPAA/PCI compliance, maximum isolation
   - Pattern: Complete separation including data layer

5. **Single-Tenant (BYOC)**: Customer's own cloud account
   - Use: Enterprise, existing accounts, full control
   - Pattern: Dev-House provides Terraform modules, customer manages infrastructure

**Pattern selection driven by**: compliance level, customer scale, budget, isolation requirements

**Harness decision logic**: Analyze PRD → Select pattern → Select cloud provider → Select Terraform components

**Reference**: docs/deployment/deployment-patterns.md

---

## Multi-Provider Terraform Architecture

**Key Decision**: Provider-specific module variants (AWS/Azure/GCP) rather than abstraction layers or Terragrunt.

**Rationale**:
- Each provider has different APIs, SKU naming conventions, architectural patterns
- Provider-specific variants are explicit and easy for Claude to reason about
- Unified logical interface (instance_size: small/medium/large) maps to provider-specific SKUs
- Clear module interface contract (same outputs: endpoint, port, username, password_secret_name)
- Customer provider selection in PRD drives code generation

**Pattern**: `modules/component-name/{aws,azure,gcp}/main.tf`

**Generator Strategy**: Claude generates provider-specific Terraform based on deployment plan's cloud_provider field

**Reference**: docs/terraform/multi-provider-strategy.md

---

## Pattern-Driven Architecture Framework

**Key Decision**: Build a reusable pattern library with systematic evaluation, weakness analysis, and template versioning. Each new PRD → evaluate against patterns → select or build new pattern → document as template → deploy → review → improve library.

**Tools**:
- `docs/architecture/pattern-evolution-framework.md` — Comprehensive framework for pattern definition, weakness analysis, and evolution
- `docs/deployment/pattern-selection-workbook.md` — **Practical guide for teams to evaluate new PRDs** (10-min checklist, decision tree, weakness check, template instantiation)
- `docs/deployment/deployment-patterns.md` — Full Tier 1-4 pattern specifications

**Reference**: CompylotAI-terraform implements Tier 3 pattern in production ✅

---

## Anthropic Harness Pattern Integration

**CRITICAL FINDING**: Anthropic has published a reference Harness pattern at https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

**Pattern**: Initializer Agent (first session) + Coding Agents (subsequent sessions)
- Initializer: Sets up `init.sh`, `claude-progress.txt`, feature list (200 items, 0/200 passing)
- Coding agents: Read progress, implement one feature, test, commit, update progress
- **Benefits**: Prevents hallucinations, clear state handoff, reproducible progress

**How Dev-House uses it**:
- **Layer 1 (Harness)**: Analyze PRD → produce ARCHITECTURAL_DECISION.md + feature list
- **Layer 2 (Codex)**: Generate services → implement feature-by-feature
- **Layer 3 (OpenClaw)**: Generate Terraform → provision component-by-component

**Tools**:
- Anthropic provides Claude Agent SDK with subagent support, context management, orchestration loops
- Use SDK for orchestration; it handles state, context compaction, parallelization

**Reference**: docs/architecture/anthropic-harness-pattern-extended.md

---

## Session Learnings

Date | Discovery | Implication | Reference
-----|-----------|-----------|----------
2026-02-28 | OpenClaw complements Claude (code → infrastructure) | Architecture needs both, not just code gen | docs/architecture/overview.md
2026-02-28 | Customers make typos, attackers make injections | Security must clean + validate, not just block | docs/security/prompt-security.md
2026-02-28 | Local LLM guards are cost-effective (offline, no tokens) | Implement multi-layer guard (pattern → fast → detailed) | docs/security/local-guard-implementation.md
2026-02-28 | Security is pipelined, not layered | Each phase catches different threats at different times | docs/architecture/security-integration.md
2026-02-28 | Cloud providers have fundamentally different APIs and SKUs | Use provider-specific variants, not abstraction layers | docs/terraform/multi-provider-strategy.md
2026-02-28 | Patterns are designed upfront with weakness analysis | Teams need practical tools (checklist, decision tree) to select patterns for new PRDs | docs/deployment/pattern-selection-workbook.md
2026-02-28 | **CRITICAL**: Productionization ≠ Product Creation | Dev-House infra (where we run) is separate cost from customer infra (what we deploy). Cost optimization matters | docs/architecture/dev-house-operational-infrastructure.md
2026-02-28 | Self-hosted (5 Mac minis + Tailscale) may be cheaper than cloud | Cost comparison: Cloud VMs $720-950/year vs Self-hosted $4.3k Y1, $2.8k Y3 (amortizes) | docs/architecture/dev-house-operational-infrastructure.md
2026-02-28 | Harness and Codex must NOT share code | Separate worktrees: `harness/[customer-id]` vs `codex/[customer-id]` vs customer repos | docs/architecture/execution-streams-codex-vs-harness.md
2026-02-28 | Repository structure is PRD-driven decision | Default: 3 separate repos (FE, BE, Infra). Decision tree determines monorepo vs separate. | docs/deployment/customer-repository-structure.md
2026-02-28 | Local dev must equal production | Every service has docker-compose.yaml; Postgres local = Managed SQL in cloud; Redis local = Azure Cache | docs/harness/local-development-environment.md


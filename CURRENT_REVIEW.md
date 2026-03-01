# Dev-House: Current Review & Open Questions

| | |
|--|--|
| **Last updated** | 2026-03-01 |
| **Purpose** | Living document of unresolved decisions, open questions, and risks. Update as decisions are made. Delete rows when resolved. |

---

## The Core Promise

> "We do the discovery, prepare the PRD, push it through our workflow, and deliver a production-ready system in one month."

Everything below is a question that must be answered before that promise is reliable.

---

## 1. Product Definition — What Are We Actually Selling?

| Question | Why It Matters |
|----------|----------------|
| Is Dev-House a service you run for customers, or a tool customers run themselves? | Determines hosting model, pricing, support obligations |
| What does the customer actually receive? A running system? A git repo? A Terraform plan? | Defines "done" |
| Who owns the generated code? You? The customer? | Legal/IP question — must be resolved before first customer |
| Who maintains the system after delivery? | If customer maintains it, they need to understand it. If you do, that's ongoing cost. |
| Is the 1-month clock from first meeting or from signed PRD? | Matters for scoping and setting expectations |

---

## 2. The PRD — Input Quality Determines Output Quality

The pipeline is only as good as what goes in. A vague PRD produces vague code.

| Question | Why It Matters |
|----------|----------------|
| What is the minimum viable PRD? What sections must it contain? | Without a spec, Harness cannot make consistent architectural decisions |
| Who writes the PRD — you, the customer, or AI-assisted? | If AI-assisted, what's the discovery process? How many sessions? |
| How do you handle ambiguous or conflicting requirements? | Does Harness flag ambiguity and halt, or make a best guess? |
| What happens when the customer changes requirements mid-generation? | Regeneration costs time and API budget. Is there a change freeze? |
| How do you validate the PRD before entering the pipeline? | A bad PRD wastes overnight cluster runs. Some human gate is needed. |

**Recommendation**: Define a PRD template with required sections before building anything. The template is the contract between discovery and generation.

---

## 3. Harness — The Orchestration Layer (Not Yet Built)

Harness is the most critical and least defined component. It bridges PRD → architecture → Codex tasks.

| Decision | Options | Status |
|----------|---------|--------|
| What does Harness actually do step-by-step with a PRD? | Parse → decide architecture → generate task list → dispatch to Codex | **Undefined** |
| How does Harness decide architecture? Rules? Claude? Templates? | Rules engine, LLM-guided, or hybrid | **Undefined** |
| What is the Harness input/output format? | JSON spec? YAML? Structured prompt? | **Undefined** |
| Where does Harness run? | Pi #2 (current TBD) | **TBD** |
| How does Harness knows when a Codex task is complete and valid? | Polling queue? Webhooks? Heartbeat? | **Undefined** |
| What is the retry/failure strategy for a Codex task? | Heartbeat TTL auto-return? Retry limit → human alert? | **Undefined** |

**Key architectural fork — work dispatch model:**

Two valid models exist. The choice changes what the Harness is responsible for. See `docs/architecture/work-dispatch-models.md` for full comparison.

| | Model A: Coordinator-Dispatch | Model B: Worker-Daemon Pull |
|--|-------------------------------|------------------------------|
| **Harness role** | Dispatcher + monitor | Queue writer + progress monitor only |
| **Routing logic** | Coordinator decides node per task | Each worker self-selects from queue |
| **Node software** | Passive (receives assignments) | Active daemon (polls, claims, executes, validates, loops) |
| **Failure handling** | Coordinator detects and reassigns | Heartbeat TTL auto-returns uncompleted tasks |
| **Heterogeneous hardware** | Coordinator must know all node types | Node config declares own capabilities |
| **Recommended for** | G-S1 (single node, simple) | G-L1 and above (multi-node, mixed hardware) |

**Leaning toward Model B** for G-L1+ given mixed hardware (Pi + Mac Mini + Cloud). Open question: which queue backend (SQLite → Redis → SQS depending on model)?

---

## 4. Codex — Code Generation

| Decision | Why It Matters |
|----------|----------------|
| What languages and frameworks does Codex support? | Scope determines what PRDs are viable |
| How is a PRD broken into discrete Codex tasks? | **Partially answered**: ticket system established (`docs/standards/`) with feature/bug/infra/refactor templates. Tickets carry PRD provenance, target repo, concurrency tier, and dependency graph — gives visibility into parallel work and worker scaling needed. Open: who creates the tickets — human, Harness, or hybrid? |
| Where do tickets live and how does Harness read them? | GitHub Issues? JIRA? Custom store? Harness must be able to query open tickets by PRD, repo, and concurrency tier to dispatch work. **Undefined.** |
| What templates or patterns does Codex use as a starting point? | Blank generation is slow and inconsistent. Templates accelerate and standardise. |
| How is generated code validated before it leaves the cluster? | Linting? Unit tests? Static analysis? Contract tests? |
| What is the human review step? Is there one? | Fully automated → risk of bad code reaching production. Human gate → slows the 1-month promise. |

---

## 5. Validation — "Production Ready" Means What Exactly?

This is the hardest question and the one most likely to break the 1-month promise.

| Question | Options |
|----------|---------|
| What tests must pass before you call a system "production ready"? | Unit, integration, E2E, performance, security scan |
| Who runs the tests — the cluster, a cloud staging environment, or both? | Cluster = ARM/no GPU; cloud staging = closer to production |
| Who signs off? Automated gate only, or human review required? | Automated gate = faster; human review = safer |
| What does the customer validate, and when? | If customer UAT is required, add 1-2 weeks to the timeline |
| What is the rollback plan if production deployment fails? | Terraform destroy? Blue/green swap? Manual rollback? |
| How do you validate that local docker-compose behaviour matches cloud deployment? | The local-to-cloud parity principle needs a concrete test strategy |

---

## 6. The Production Pipeline — Terraform & Cloud

| Decision | Status |
|----------|--------|
| Which cloud providers are supported? AWS only? GCP? Azure? All three? | **Undefined** |
| What cloud services are in scope? (compute, database, networking, storage, queues, CDN...) | **Undefined** |
| How are Terraform modules organised? Per-service? Per-pattern? | **Undefined** |
| How are customer cloud credentials handled? Who holds them? | Security-critical — must be defined before first customer |
| Who reviews the Terraform plan before `apply`? | Human approval step or fully automated? |
| What is the approval gate before provisioning customer infrastructure? | Missing this step = risk of accidental spend or misconfiguration |
| How are Terraform state files stored and who has access? | Per-customer S3 backend? Terraform Cloud? | **Undefined** |

---

## 7. The 1-Month Promise — Is It Realistic?

Working backwards from "production ready in one month":

```
Week 1: Discovery + PRD
├── Customer meetings
├── PRD writing (AI-assisted)
└── PRD validation + sign-off (change freeze begins)

Week 2: Generation
├── Run PRDs through cluster overnight (multiple passes)
├── Review generated output each morning
└── Iterate on failing tasks

Week 3: Staging + Validation
├── Deploy to cloud staging via Terraform
├── Run full test suite
├── Customer UAT (if required)
└── Fix blocking issues

Week 4: Production
├── Final review
├── Production deployment
└── Handoff + documentation
```

**Risks to the timeline**:
- PRD ambiguity discovered during generation → back to discovery (costs 1 week)
- Generated code fails validation → debug cycle (1-3 days per issue)
- Customer UAT feedback → scope change (potentially resets Week 2)
- Cloud provisioning issues (IAM, quotas, region availability)
- Security review findings → remediation

**The honest answer**: 1 month is achievable for well-scoped, standard-pattern systems. It requires a tight PRD, no scope changes after sign-off, and a customer who is available for UAT in Week 3. Define these constraints explicitly before making the promise.

---

## 8. Provider TOS Risk — Existential, Not Theoretical

**The Claudbot precedent**: Claudbot was a third-party tool built on Claude's API with a large user base (thousands of $200/month subscribers). Anthropic's TOS enforcement killed it. OpenAI stepped in, offered equivalent capability within their Auth0/$200 plan quota, and absorbed the user base. Anthropic lost the subscribers and the goodwill.

This is not a hypothetical risk for Dev-House. PRD-to-code automation using Claude is exactly the kind of workflow that could attract TOS scrutiny (bulk API usage, automated code generation at scale, reselling AI capability to customers).

| Risk | Mitigation |
|------|------------|
| Anthropic changes TOS or interpretation mid-contract | Multi-provider abstraction layer — swap Claude → OpenAI with no customer impact |
| Rate limiting kills overnight batch runs | Distribute across providers; Claude for architecture, OpenAI for code gen, or vice versa |
| Provider reprices API mid-customer-contract | Cost model must account for provider switching; don't lock customer pricing to one provider's rates |
| Provider decides to compete directly in this space | Already happening — both Anthropic and OpenAI have coding agents. Dev-House's moat is the workflow, not the model. |

**Decision (2026-03-01)**: Multi-provider abstraction is existential, not optional. The harness dispatch layer must be able to route to any provider. No single-provider dependency in the critical path.

---

## 9. Security & Trust

From the security architecture docs — still open:

| Question | Why It Matters |
|----------|----------------|
| Customer PRDs contain sensitive business logic — how is that data protected? | PRD content may be IP. Needs encryption at rest and in transit. |
| Are customer PRDs sent to Claude API? | If yes, Anthropic's data handling policies apply. Customer must consent. |
| How are Claude API keys managed across cluster nodes? | Shared key? Per-node keys? Key rotation strategy? |
| How are customer cloud credentials isolated from each other? | Multi-customer scenario — one customer must not access another's infrastructure |
| What is the audit trail? Who did what, when, with which PRD? | Required for enterprise customers and any post-incident review |
| Is there a security review step before production deployment? | OWASP scan? Dependency audit? Secret scanning in generated code? |

---

## 10. Multi-Customer Isolation

If you run multiple customers simultaneously (which the overnight batch model enables):

| Question | Status |
|----------|--------|
| Separate git repos per customer? | Likely yes — but confirm |
| Separate k3s namespaces per customer? Or separate jobs? | **Undefined** |
| How is Claude API rate limiting handled across concurrent customers? | One customer's jobs could throttle another's |
| How are generated artefacts (code, images, Terraform) isolated between customers? | Naming convention? Separate registry paths? |
| What happens if one customer's job crashes a dev Pi? | The 8-12 job overnight batch assumes stability — failure isolation needed |

---

## 11. The Dev-House Cluster vs The Customer's System

This separation must be crystal clear in all docs and customer conversations:

```
Dev-House Pi Cluster (yours):
└── Generates code overnight → pushes to GitHub

Customer Production (theirs, cloud):
└── Provisioned by Terraform → running in AWS/GCP/Azure
```

**Risk**: Customers may confuse the development infrastructure (Pi cluster) with their production system. Be explicit that the cluster is your internal tooling, not their runtime.

**Open question**: Do enterprise customers want to run Dev-House themselves (on-premise or in their own cloud)? If yes, the Pi cluster reference pattern becomes a customer deliverable, not just an internal tool.

---

## 12. OpenClaw — Largely Undefined

Per the architecture docs, OpenClaw handles "infrastructure orchestration, deployment automation, policy enforcement." Almost nothing else is defined.

| Question |
|----------|
| Is OpenClaw a new tool you are building, or a thin wrapper around existing tools (Terraform, Helm, kubectl)? |
| What policies does it enforce, and who defines them? |
| How does it relate to Harness? Does Harness call OpenClaw, or are they peers? |
| Is OpenClaw customer-facing (they configure it) or internal only? |

---

## 13. Prioritisation — What to Build First

Given the goal of delivering to a customer in 1 month, the critical path is:

```
1. PRD template (defines the contract — everything else depends on it)
2. Harness core (PRD → task list → Codex dispatch)
3. Codex generation (a single working service end-to-end)
4. Validation pipeline (the thing that proves it works)
5. Terraform output (the thing that deploys it)
```

The Pi cluster is infrastructure for running this. It is not the product. Don't let hardware planning crowd out product definition.

**The riskiest assumption**: That Claude can generate production-quality code from a PRD with minimal human review. This needs to be tested on a real PRD before making customer promises. Run a spike: take a simple PRD, push it through a manual version of the pipeline, and see what comes out.

---

## Decisions Log

Track decisions here as they are made:

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-03-01 | Ticket standards established (`docs/standards/`) | Versioned templates (feature, bug, infra, refactor) with PRD provenance fields. Every ticket carries `(template_version, harness_type, PRD_source)` tuple for quality audits. |
| 2026-03-01 | Multi-harness abstraction layer is a design goal | Research confirmed: Claude Agent SDK (primary), OpenAI Agents SDK (provider-agnostic, worth spiking), CrewAI/LangGraph (orchestration). Harness must be provider-agnostic — model config, not hard dependency. |
| 2026-03-01 | OpenClaw (Clawdbot) identity confirmed | OpenClaw is a personal messaging assistant (WhatsApp/Telegram/Slack → Claude). NOT an infra tool. The "OpenClaw" in this project is a separate internal concept. TOS incident: Anthropic blocked it, creator acqui-hired by OpenAI Feb 2026. Validated TOS risk is real. |
| 2026-03-01 | Recommended orchestration stack | Claude Agent SDK + Anthropic Harness Pattern + CrewAI or LangGraph + MCP + Hooks. No single cloud platform adopted wholesale — all are cloud-runtime-only except Google ADK (open-source, ARM-compatible). |
| 2026-03-01 | No platform natively generates Terraform from specs | Confirmed across AWS, Google, Azure, OpenAI, all open-source frameworks. IaC generation is Dev-House's unique value — the pipeline that does it end-to-end. Build it, don't buy it. |
| 2026-03-01 | Quality instrumentation via provenance tracking | Template version × harness version × output quality will be tracked. Enables selective re-runs: "re-run all tickets generated with anthropic-harness-v1 through v2 and compare". |
| 2026-03-01 | Skip claude-flow entirely | 70-80% of features now native in Claude Code (Agent Teams, custom subagents, 17-event hooks). v3 is actively broken (GitHub issues #958, #945, #984, #1082 — single maintainer, memory leaks, SQL injection, commands that throw help errors). Only gap is structured shared state, bridgeable with a simple SQLite MCP server (~100 lines, half a day). See `docs/research/20260301_OPUS_claude-flow-convergence.md`. |
| 2026-03-01 | Deployment model framework established (G-series + D-series) | Generation Infrastructure (G-L1/L2/L3/C1) selected independently from Customer Deployment (D-T3/T4). Model docs in `docs/deployment/models/`. Research scoped to model subfolders (`docs/research/G-L1/` for Pi-specific). |

---

## Resolved Questions

Move items here from the sections above once resolved:

*(empty)*

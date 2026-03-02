# Dev-House: AI Harness Automation Framework

**Build infrastructure and applications from business requirements using Claude AI.**

Dev-House is a customer-deployable framework that converts PRDs (Product Requirements Documents) into running applications and infrastructure through AI-assisted development.

## Vision

Enable customers to:
1. Write business requirements (PRD)
2. Run Dev-House harness
3. Get running infrastructure + applications
4. Review, customize, deploy

**No need to write boilerplate infrastructure code.**

---

## Stack

### AI Employees (per node)

Each AI employee node runs two concurrent code generation agents and operates autonomously:

- **Claude Code (Anthropic, ~$200/month)** — Agentic code generation; claims tasks from queue, writes code, runs tests, commits
- **OpenAI Codex (~$200/month)** — Agentic code generation; equivalent capability, different provider

**Ongoing cost per AI employee**: ~$400/month + electricity

Both agents implement tickets from the same queue. Provider redundancy: if one goes down the other keeps working. No vendor lock-in — either provider can be replaced without changing the Harness or customer output.

The **Harness** is a separate orchestration engine (not an AI subscription) — it analyzes PRDs, generates tickets, manages the queue, and tracks progress. See `docs/architecture/model-router.md` for model routing within each provider.

### Orchestration
- **Harness** — Dev-House orchestration engine
- **State Machine** — Track PRD → analysis → generation → deployment

### Deployment Targets
- Docker / Docker Compose (local + staging)
- Terraform (AWS, Azure, GCP, on-prem)
- Serverless containers: Azure Container Apps, Cloud Run, Fargate (see `docs/research/20260302_OPUS_aca-vs-kubernetes.md`)
- Kubernetes (for larger customer workloads)
- Custom plugins

---

## Project Structure

```
dev-house/
├── README.md                 # This file
├── CLAUDE.local.md           # Session cache & project patterns
├── src/                      # Implementation (to be built)
│   ├── harness/             # Orchestration engine
│   ├── ai/                  # AI provider integrations (Claude Code, Claude API, OpenAI)
│   ├── openclaw/            # OpenClaw integration
│   └── deployment/          # Deployment plugins
├── docs/                    # Always-current documentation
│   ├── README.md           # Documentation index (start here)
│   ├── architecture/        # System design
│   ├── harness/            # Harness details
│   ├── codex/              # Code generation details
│   ├── openclaw/           # OpenClaw integration
│   ├── deployment/         # Deployment procedures
│   └── patterns/           # Reusable patterns
├── examples/                # Example PRDs & configurations
├── tests/                   # Test suite
└── scripts/                 # Build, utility scripts
```

---

## Quick Start

*(Coming soon — under development)*

```bash
# Clone the project
git clone <repo> dev-house
cd dev-house

# Each AI employee authenticates via its own OAuth subscriptions
# (Claude Code + Claude API — no shared API keys, each node is independent)

# Write a PRD
cat > my-app.prd << 'EOF'
# REST API for Task Management

## Requirements
- Node.js + Express backend
- PostgreSQL database
- JWT authentication
- Full CRUD for tasks
- Deployed to AWS

EOF

# Run harness
harness run < my-app.prd

# Review generated code, approve, deploy
```

---

## Documentation

**Start here**: [docs/README.md](docs/README.md) — Documentation index

### Key Docs
- **[Architecture Overview](docs/architecture/overview.md)** — System design
- **[Harness Orchestration](docs/harness/orchestration.md)** — How the harness works
- **[Codex Integration](docs/codex/generation.md)** — Code generation
- **[OpenClaw Integration](docs/openclaw/)** — Infrastructure orchestration
- **[Customer Deployment](docs/deployment/customer-deployment.md)** — How customers deploy

---

## Development

### Principles

1. **Orchestrator pattern** — Claude is the orchestrator; preserve context, delegate heavy lifting to subagents
2. **Batch operations** — 1 message = all related reads, writes, edits
3. **Search before write** — Check if utility exists before creating new ones
4. **Document everything** — Keep docs in sync with code

### Before Starting

1. Read [docs/README.md](docs/README.md) — Find the right doc
2. Read [docs/architecture/overview.md](docs/architecture/overview.md) — Understand the system
3. Read [CLAUDE.local.md](CLAUDE.local.md) — Project-specific patterns

### Workflow

- **Small change** → Read relevant doc → Implement → Test → Commit
- **New feature** → Read architecture → Design → Implement → Test → Update docs → Commit
- **Bug fix** → Reproduce → Debug → Fix → Test → Commit

---

## Examples

*(Coming soon)*

### Example 1: REST API
PRD → Harness → Express API + PostgreSQL + Terraform + AWS deployment

### Example 2: Batch Processor
PRD → Harness → Python job + Docker + Kubernetes manifest

### Example 3: Full Stack
PRD → Harness → React frontend + Node.js backend + PostgreSQL + Terraform + AWS

---

## License

*(TBD)*

---

## Contributing

*(Development in progress)*

---

## Support

- Issues: [GitHub Issues](https://github.com/...)
- Documentation: [docs/README.md](docs/README.md)
- Email: support@dev-house.io

---

## Status

**Current Phase**: Architecture & Foundation

- [x] System architecture designed
- [x] Documentation structure created
- [ ] Harness core implementation
- [ ] Claude integration
- [ ] OpenClaw integration
- [ ] Deployment plugins
- [ ] E2E tests
- [ ] Customer deployment guide


# Harness & Agent Orchestration Framework Comparison

**Date**: 2026-03-01
**Model**: Opus (4 parallel research agents)
**Purpose**: Evaluate all major AI agent orchestration systems for Dev-House — PRD-to-deployment automation on Pi cluster

---

## The OpenClaw / Claudbot Mystery — Resolved

**OpenClaw** (originally "Clawdbot") is a personal AI assistant that connects via messaging platforms (WhatsApp, Telegram, Slack, Discord) and executes tasks on the user's behalf. Created by Austrian developer Peter Steinberger in November 2025. Gained 100K+ GitHub stars in under a week (late January 2026).

**What happened**: Users were connecting it to Claude's API as an unofficial interface. Anthropic's TOS enforcement blocked this use case. On February 14, 2026, Steinberger announced he was joining OpenAI — an acqui-hire, not a company acquisition. The project transitioned to an independent open-source foundation. OpenAI absorbed those users by supporting the use case.

**OpenClaw is NOT**: An infrastructure orchestration tool, a deployment framework, or a coding agent. It is a messaging-connected personal assistant. The name confusion with the in-project "OpenClaw" (your infrastructure orchestration layer) is coincidental.

**Relevance to Dev-House**: Minimal for tooling. High for strategic risk — this is the Claudbot incident. A third-party tool built on Claude, TOS-killed overnight, users migrated to OpenAI. The existential lesson stands regardless of what OpenClaw actually does.

---

## Master Comparison Matrix

| System | Category | Maturity | Capability | Dev-House Fit | Provider Agnostic | ARM/Pi |
|--------|----------|----------|------------|---------------|-------------------|--------|
| **Claude Agent SDK** | Anthropic | 4/5 | 4/5 | 4/5 | Multi-provider | Yes |
| **Harness Pattern** | Anthropic | 3/5 | 3/5 | **5/5** | N/A (blueprint) | N/A |
| **MCP** | Standard | **5/5** | 4/5 | **5/5** | Yes (industry std) | Yes |
| **Agent SDK Hooks** | Anthropic | 3/5 | 4/5 | **5/5** | Yes | Yes |
| **LangGraph** | Open-source | 4/5 | **5/5** | 4/5 | Yes | Yes |
| **CrewAI** | Open-source | 3/5 | 4/5 | **5/5** | Yes | Yes (confirmed) |
| **PydanticAI** | Open-source | 3/5 | 4/5 | 4/5 | Yes (broadest) | Yes |
| **OpenAI Agents SDK** | OpenAI | 4/5 | 4/5 | 4/5 | Yes* | Yes |
| **OpenAI Responses API** | OpenAI | 4/5 | 4/5 | 3/5 | No | Yes |
| **OpenAI Codex (cloud)** | OpenAI | 4/5 | **5/5** | 3/5 | No | Cloud only |
| **Google ADK** | Google | 4/5 | 4/5 | 4/5 | Yes | Yes |
| **Azure Foundry Agent** | Azure | 4/5 | **5/5** | 3/5 | Yes (18+ models) | No |
| **AWS Bedrock + AgentCore** | AWS | 4/5 | 4/5 | 3/5 | Yes (multi-model) | No |
| **MS Agent Framework** | Microsoft | 4/5† | 3/5 | 3/5 | Partial | Yes |
| **LlamaIndex Workflows** | Open-source | 3/5 | 3/5 | 3/5 | Yes | Yes |
| **claude-flow** | Community | 1/5 | 3/5 | 2/5 | No (Claude only) | Yes |
| **Dify** | Open-source | 4/5 | 3/5 | 2/5 | Yes | Marginal |
| **AutoGen** | Microsoft | 2/5 | 4/5 | 2/5 | Yes | Likely |
| **Haystack** | Open-source | 4/5 | 2/5 | 2/5 | Yes | Yes |
| **BeeAI** | Linux Foundation | 2/5 | 3/5 | 2/5 | Yes | Yes |
| **OpenAI Swarm** | OpenAI | 1/5 | 2/5 | 1/5 | No | Yes |
| **OpenAI Assistants API** | OpenAI | 2/5 | 3/5 | 1/5 | No | Yes |
| **Amazon Q Developer** | AWS | 4/5 | 4/5 | 2/5 | No | No |
| **Google Jules** | Google | 3/5 | 4/5 | 2/5 | No | No |
| **OpenClaw** | Community | N/A | N/A | 1/5 | N/A | N/A |

*OpenAI Agents SDK is documented as provider-agnostic — Claude can be the backing model.
†MS Agent Framework maturity is 4 for Semantic Kernel alone; the AutoGen merger product is pre-GA.

---

## DO NOT USE — Dead Ends

| System | Reason |
|--------|--------|
| **OpenAI Assistants API** | Deprecated August 2025. Sunset August 2026. Do not build on this. |
| **OpenAI Swarm** | Superseded by Agents SDK. Educational reference only. |
| **AutoGen standalone** | Breaking v0.4 rewrite + community fork (AG2) = ecosystem fragmentation. Unclear future as MS merges it. |
| **claude-flow** | Single maintainer (ruvnet), Claude-only, unstable naming (claude-flow → ruflo), no stable releases. Too risky as a production dependency. |

---

## Recommended Architecture for Dev-House

### Layer 1 — Core Runtime (Pi Cluster)

**Claude Agent SDK** as the execution engine on each Pi node.
- 1 GiB RAM, 5 GiB disk, 1 CPU per instance → 4-6 concurrent instances per 8GB Pi
- Session resume for overnight batch runs (save session IDs to NAS)
- Subagent spawning for parallel workers within a session
- ARM-compatible, confirmed community reports

### Layer 2 — Pipeline Design

**Anthropic Harness Pattern** as the workflow blueprint:
- Initializer Agent: analyzes PRD → creates structured ticket list (JSON) + `claude-progress.txt`
- Coding Agents: work through tickets one at a time per session → test → commit → update progress
- Map directly: PRD tickets replace "feature list", target repos replace "app"

### Layer 3 — Agent Orchestration Framework

Choose one of:

**CrewAI** (recommended for faster start):
- Role-based: PRD Analyst, Architect, Coder, Terraform Engineer, Validator
- Confirmed working on Raspberry Pi
- Flows API for production-grade pipeline control
- Risk: Flows API is newer, less battle-tested

**LangGraph** (recommended for maximum control):
- Graph nodes map to pipeline stages (PRD parse → architecture → codegen → Terraform → validate → deploy)
- Checkpointing means a failed stage retries without re-running upstream work
- LangSmith gives provenance tracking out of the box
- Risk: Steep learning curve

**PydanticAI** (recommended for engineering rigour):
- Type-safe pipeline stages catch errors at write-time
- Temporal integration = durable execution (agents survive crashes, restarts, deploys)
- Broadest model provider support of any framework
- Risk: Newer framework, smaller community

### Layer 4 — Tool Integration

**MCP** for everything external:
- Terraform MCP server (already in environment)
- GitHub MCP server (already in environment)
- Azure MCP server (already in environment)
- Custom MCP servers for PRD parsing, OpenClaw (your infra layer), job queue
- Industry standard — OpenAI, Google, Microsoft all on board. Linux Foundation governed.

### Layer 5 — Cross-Cutting Concerns (Hooks)

**Agent SDK Hooks** for security and provenance:
- `PreToolUse` → prompt injection detection (Phase 2 security gate)
- `PostToolUse` → provenance logging (session_id + agent_version + model + files_modified)
- `SessionStart/End` → state checkpoint for overnight runs
- `SubagentStart/Stop` → track which worker handled which ticket

### Layer 6 — What You Build Yourself

These are gaps no framework fills — you build them:

| Gap | Solution |
|-----|----------|
| Job queue | SQLite on NAS or Redis. Tickets → queue → Pi workers |
| Cluster coordinator | Lightweight Python service. Assigns tickets to available Pis, tracks completion |
| Provenance database | `(session_id, agent_version, model, template_version, ticket_id, files_modified, timestamp)` |
| Hierarchical agents | Can't nest Task tool subagents. Your coordinator handles hierarchy instead |

---

## What to Watch

### OpenAI Agents SDK (Provider-Agnostic)
Released March 2025. Documented to work with non-OpenAI models. If it runs Claude as the backing model with full orchestration primitives (handoffs, guardrails, tracing, sessions), this becomes a serious option as a cross-provider orchestration layer.

**Action**: Run a spike — configure OpenAI Agents SDK with Claude API as the model. If it works cleanly, evaluate as an alternative to CrewAI/LangGraph.

### Azure Model Router
Routes automatically across 18+ models (Claude, GPT, Gemini, Llama, DeepSeek, Grok) — quality mode, cost mode, balanced. No agent surcharge — pay only for tokens.

**Action**: Watch for a standalone API offering. If it becomes callable outside Azure Foundry, it solves multi-model routing without building it yourself.

### PydanticAI + Temporal
Durable execution for long-running agents. Crash a Pi mid-Terraform-generation, pick up exactly where you left off.

**Action**: Evaluate as the persistence backbone if sqlite/NAS job queue proves insufficient.

---

## IaC Generation — The Gap Nobody Fills

None of the orchestration platforms natively generate Terraform from specs. They all treat it as "you build the tool, we call it."

**Confirmation**: Dev-House's approach (Claude/Codex generating Terraform directly, orchestration platform coordinates) is correct. There is no magic IaC generation service to plug in.

For a dedicated IaC generation product: **StackGen StackBuilder** and **Spacelift Intent** exist but are third-party SaaS, not embeddable in your pipeline.

---

## Multi-Provider Abstraction — Validated

The Claudbot/OpenClaw incident, the Windsurf acquisition collapse, and the general pace of TOS changes all validate the decision: **single-provider lock-in is existential risk**.

Recommended abstraction approach:
```
Orchestration (CrewAI/LangGraph/PydanticAI) — provider-agnostic
    ↓
Model router (your config or Azure Model Router)
    ↓
Claude API | OpenAI API | Google API | AWS Bedrock
```

Every pipeline stage specifies a model preference, not a hard dependency. Swapping Claude → GPT-4o for a given stage should require changing one config line, not rewriting the agent.

---

## Strategic Notes

### Google ADK Advantage
Only cloud-vendor SDK that is open-source AND ARM-compatible. Can run agent logic on Pi cluster, calling cloud model APIs. If you ever need a Google-vendor-backed framework with self-hosting, ADK is the answer.

### No Platform Fully Solves IaC
Not AWS, not Google, not Azure. They all expect you to build the Terraform generation as a tool/action. This is Dev-House's unique value — the pipeline that does it end-to-end.

### Enterprise Deployment Option
If a customer requires managed cloud infrastructure:
- **Azure Foundry Agent Service**: Broadest model selection (18+ including Claude), no agent surcharge, strongest enterprise compliance
- **AWS Bedrock AgentCore**: Best for AWS-native customers; framework-agnostic runtime

Neither replaces your Pi cluster for internal Dev-House operations.

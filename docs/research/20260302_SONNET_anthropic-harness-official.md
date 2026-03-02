# Anthropic Harness Pattern: Official Documentation Research

| | |
|--|--|
| **Date** | 2026-03-02 |
| **Model** | claude-sonnet-4-6 |
| **Type** | Research — frozen at time of writing |
| **Question** | What does Anthropic officially document about the harness pattern, agent SDK, and reference implementations? |

---

## Summary

Anthropic uses the term **"harness"** officially and has published both a canonical blog post and a working reference implementation. The Claude Agent SDK (formerly Claude Code SDK) is explicitly described as "the agent harness that powers Claude Code." Dev-House's architecture is directionally correct — but the reference implementation is intentionally minimal, and everything above the single-machine two-agent loop (multi-provider, cluster dispatch, deployment pipeline) is original design space.

---

## 1. Official References

| Resource | URL | Notes |
|----------|-----|-------|
| Blog post | https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents | Canonical pattern definition |
| Agent SDK announcement | https://claude.com/blog/building-agents-with-the-claude-agent-sdk | SDK rationale and overview |
| SDK Python | https://github.com/anthropics/claude-agent-sdk-python | `pip install claude-agent-sdk` |
| SDK TypeScript | https://github.com/anthropics/claude-agent-sdk-typescript | |
| SDK demos | https://github.com/anthropics/claude-agent-sdk-demos | Example implementations |
| Autonomous coding quickstart | https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding | Reference harness implementation |
| Subagents doc | https://code.claude.com/docs/en/sub-agents | Official subagent format |
| Agent Teams doc | https://code.claude.com/docs/en/agent-teams | Multi-agent peer model |

---

## 2. The Reference Implementation (autonomous-coding)

### Two-Agent Architecture

**Initializer Agent** (Session 1 only):
- Input: `app_spec.txt` (free-form natural language spec)
- Output: `feature_list.json`, `init.sh`, `claude-progress.txt`, initial git commit
- This is the equivalent of Dev-House's Harness Initializer producing `ARCHITECTURAL_DECISION.md` + `SERVICE_DECOMPOSITION.yaml`

**Coding Agents** (Sessions 2+, runs in a loop):
- Input: `app_spec.txt`, `feature_list.json`, `claude-progress.txt`, `git log`
- Picks one pending feature, implements, verifies, commits, updates `feature_list.json`
- Ends every session in a clean commitable state

### The `feature_list.json` Schema

This is the central artifact — the structured expansion of the PRD. Dev-House's equivalent is `feature_list.json` fed to Codex generators.

```json
[
  {
    "category": "functional",
    "description": "New chat button creates a fresh conversation",
    "steps": [
      "Navigate to main interface",
      "Click the 'New Chat' button",
      "Verify a new empty conversation appears",
      "Verify the previous conversation is preserved in history"
    ],
    "passes": false
  },
  {
    "category": "style",
    "description": "Chat bubbles have distinct left/right alignment for user vs assistant",
    "steps": [
      "Navigate to active conversation",
      "Send a message",
      "Verify user message appears on right with correct styling",
      "Verify assistant message appears on left"
    ],
    "passes": false
  }
]
```

**Rules enforced via prompt instructions:**
- Never remove or edit existing features — only mark `passes: true`
- Minimum 200 features; at least 25 must have 10+ steps
- Mix of `functional` and `style` categories
- Priority-ordered (foundational features first)
- All start as `passes: false`

### Security Model

Bash command allowlist enforced via `PreToolUse` hook:

```python
ALLOWED_COMMANDS = [
    "ls", "cat", "head", "tail", "grep",
    "npm", "node",
    "git",
    "ps", "lsof", "sleep", "pkill"
]
```

All other bash commands blocked. This is the pattern we should adopt in Dev-House's Codex worker security model.

---

## 3. Claude Agent SDK — Correct Import (Correction)

The existing code in `anthropic-harness-pattern-extended.md` uses a fictional import:
```python
from anthropic_sdk import Agent, Subagent  # ← DOES NOT EXIST
```

The actual SDK:
```python
from claude_agent_sdk import query, ClaudeSDKClient
```

**Simple async one-shot:**
```python
from claude_agent_sdk import query

async for message in query(prompt="Build the authentication module"):
    print(message)
```

**Stateful multi-turn:**
```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

client = ClaudeSDKClient(
    options=ClaudeAgentOptions(
        system_prompt="You are a senior engineer...",
        tools=["read", "write", "bash"],
        max_turns=50
    )
)
```

**Key SDK components:**

| Component | Purpose |
|-----------|---------|
| `query()` | Simple async one-shot |
| `ClaudeSDKClient` | Stateful bidirectional |
| `ClaudeAgentOptions` | Config: tools, system prompt, permissions, hooks, MCP |
| In-process MCP servers | Custom tools as Python functions (no subprocess overhead) |
| Hooks | `PreToolUse`, `PostToolUse`, `Stop` — deterministic application-level intervention |

---

## 4. Subagent Format (Official)

Subagent definitions use YAML frontmatter + Markdown system prompt. These live in `.claude/agents/` (project-scoped) or `~/.claude/agents/` (user-scoped).

```markdown
---
name: code-reviewer
description: Reviews code for quality. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
model: sonnet
permissionMode: default
memory: project
---

You are a senior code reviewer. When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Provide feedback organised by priority: Critical / Warnings / Suggestions.
```

**Available fields:**

| Field | Options | Default |
|-------|---------|---------|
| `name` | string | required |
| `description` | string | required |
| `tools` | list of tool names | inherits |
| `disallowedTools` | list | none |
| `model` | `sonnet`, `opus`, `haiku`, `inherit` | inherit |
| `permissionMode` | `default`, `acceptEdits`, `bypassPermissions` | default |
| `maxTurns` | integer | no limit |
| `memory` | `user`, `project`, `local` | local |
| `isolation` | `worktree` | none |

**Subagent constraints:**
- Cannot spawn other subagents (no nesting)
- Zero parent conversation history — only system prompt + env
- Parent receives only a summary, not subagent's full context

---

## 5. Agent Teams (Experimental)

Peer-to-peer variant where workers message each other directly.

| Aspect | Subagents | Agent Teams |
|--------|-----------|-------------|
| Context | Own window; results return to caller | Own window; fully independent |
| Communication | Report to main agent only | Message each other directly |
| Best for | Focused tasks; result is what matters | Complex work requiring discussion |
| Enable | Default | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |

---

## 6. Input Specification Pattern

Anthropic's pattern uses **`app_spec.txt`** — free-form natural language. No official PRD template exists. The deliberate design choice: the human writes a natural-language spec; the Initializer LLM structures it into `feature_list.json`.

**Implication for Dev-House**: Our structured PRD template (Sections 1-10 with required fields) is a deliberate extension of the Anthropic pattern. The extra structure makes the Initializer's job more reliable (less ambiguity to resolve) at the cost of more work for the customer/discovery process. This is the right trade-off for production customer use — document the rationale.

---

## 7. Gaps: What Anthropic Doesn't Document

The reference harness is a **two-agent, single-stream, local Python script** designed for demonstration. These are confirmed gaps requiring original Dev-House design:

| Gap | Anthropic Provides | Dev-House Must Design |
|----|-------------------|----------------------|
| PRD structure | Free-form `app_spec.txt` | Structured PRD template with required sections |
| Multi-provider | Single Claude | Claude + Codex concurrent per node; result merging; conflict resolution |
| Multi-node dispatch | Local only | Pi cluster job queue; worker health; heartbeat/TTL |
| Cost tracking | None | Per-PRD token accounting across N sessions |
| `feature_list.json` validation | Prompt-enforced convention | Schema enforcement before coding sessions begin |
| Deployment integration | None (stops at code) | Git push → CI → ACA/Fargate deployment pipeline |
| Security pipeline | None | Phases 1-3 prompt injection defence (existing `docs/security/`) |
| Inter-stream coordination | None | Harness Stream 1 (analysis) feeding Stream 2 (code generation) |
| Progress observability | `claude-progress.txt` text file | Structured progress API; dashboard; webhooks |
| Session recovery | Git-based basic | Checkpoint/resume; partial-failure recovery |

---

## 8. Implications for dev-house-existing-docs

**Action required — correction:**

`docs/architecture/anthropic-harness-pattern-extended.md` contains a fictional `from anthropic_sdk import Agent, Subagent` code example. This should be updated to reference the real SDK (`claude_agent_sdk`) before any implementation begins.

**Action required — update:**

The file says the Anthropic pattern produces `ARCHITECTURAL_DECISION.md`, `SERVICE_DECOMPOSITION.yaml`, `REPO_STRUCTURE.yaml`, `COMPONENT_REQUIREMENTS.yaml`. These are our Dev-House extensions. The actual Anthropic Initializer produces `feature_list.json`, `init.sh`, `claude-progress.txt`. Both are correct — just be clear in docs which is Anthropic's original and which is our extension.

**Confirmed correct:**
- Three-layer extension (Harness / Codex / OpenClaw) — valid pattern extension
- `feature_list.json` as the key handover artifact — matches exactly
- `claude-progress.txt` → `harness-status.yaml` — equivalent, our structured version is better
- One-feature-per-session loop — confirmed correct
- Git commits as progress mechanism — confirmed correct

---

## 9. Model Selection Confirmation (Current Lineup)

| Model | API ID | Max Output | Best For |
|-------|--------|------------|---------|
| Opus 4.6 | `claude-opus-4-6` | 128K | Agents, architecture analysis, Harness Initializer |
| Sonnet 4.6 | `claude-sonnet-4-6` | 64K | Code generation, Codex workers |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | 64K | Fast triage, routing, Explore subagents |

All three support extended thinking and adaptive thinking. 1M token context window available (beta) on Opus 4.6 and Sonnet 4.6.

---

## Sources

- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
- [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Agent teams](https://code.claude.com/docs/en/agent-teams)
- [Claude Agent SDK Python](https://github.com/anthropics/claude-agent-sdk-python)
- [Claude Agent SDK Demos](https://github.com/anthropics/claude-agent-sdk-demos)
- [Autonomous coding quickstart](https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding)
- [Models overview](https://platform.claude.com/docs/en/about-claude/models/overview)

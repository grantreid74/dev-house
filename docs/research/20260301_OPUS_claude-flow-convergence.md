# Claude-Flow Feature Convergence Analysis

**Date**: 2026-03-01
**Model**: Opus
**Deployment Context**: Cross-model (applies to all G-series models)
**Question**: Has Claude Code / Claude Agent SDK converged on features that previously required claude-flow? What would we lose by skipping it entirely?

---

## Executive Summary

**~70-80% convergence.** Claude Code Agent Teams + custom subagents + hooks now cover most of what made claude-flow attractive. The remaining gap (structured shared memory between agents) is bridgeable with a simple MCP server in half a day. claude-flow v3 is actively broken in production use.

**Recommendation: Skip claude-flow entirely.**

---

## The Convergence Picture

| Feature | claude-flow | Claude Code Native | Gap |
|---------|-------------|-------------------|-----|
| Hierarchical orchestration | Yes | Yes (Agent Teams) | None |
| Specialized agent types | 54 fixed types | Custom frontmatter agents (unlimited) | None — native is MORE flexible |
| Hook system | Basic | 17 events, 4 handler types | None — Claude Code wins |
| Per-agent persistent memory | Via SQLite | Subagent memory dirs + MEMORY.md | None |
| Inter-agent comms | Stream-JSON | Mailbox + broadcast (Agent Teams) | None |
| Context compaction | Claimed | PreCompact + SessionStart hooks | None — Claude Code wins |
| Workflow dependencies | Yes | Agent Teams task list + auto-unblock | None |
| MCP tool integration | 87 wrapper tools | Native MCP loading (ecosystem-wide) | None — native is better |
| Consensus protocols | Raft/BFT/Gossip | File locking + lead | None — wrong abstraction for LLM agents |
| Cross-agent shared state | SQLite DB | Task list + messaging | **Low gap** — bridgeable |
| Neural/continuous learning | Claimed (unstable) | Explicit memory management | None — claude-flow's is broken |
| Provider-agnostic | No (Claude-only) | No (Claude-only) | N/A — both fail here |

---

## claude-flow v3 Status (Real Data from GitHub Issues)

- **Issue #958**: Tasks sitting in pending queues with errors. MCP server registering only 27 of 200 claimed tools.
- **Issue #945**: v3 rebuild acknowledges memory leaks and SQL injection vulnerabilities in `memory-initializer.ts`. Architectural issues.
- **Issue #984**: Documented commands throwing help errors instead of executing.
- **Issue #1082**: Project repositioning itself from "orchestration platform" to "intelligence layer" in response to Claude Code Agent Teams. The value proposition is actively shifting.

The single-maintainer (ruvnet) is rebuilding a system that claims 175+ MCP tools, 54+ agent types, consensus protocols, neural learning, and swarm coordination. v3 does not have working end-to-end examples.

---

## What Claude Code Added (Explaining the Convergence)

**Agent Teams** (experimental, Feb 2026 / Claude Code 2.0):
- Team lead + teammates architecture
- Shared task list with dependency auto-unblocking
- Mailbox (direct messaging + broadcast) between teammates
- Task claiming with file locking (no race conditions)
- Plan approval gates before implementation
- TeammateIdle + TaskCompleted hooks for quality gates

**Custom Subagents** (Claude Code 2.x):
- Defined via markdown frontmatter (`.claude/agents/` or `~/.claude/agents/`)
- Custom system prompts, tool allowlist/denylist, model selection
- Per-subagent hooks, per-subagent MCP servers
- Persistent memory (`memory` field, scoped to user/project/local)
- Background vs foreground execution, worktree isolation
- Community resource: 100+ pre-built subagent definitions at github.com/VoltAgent/awesome-claude-code-subagents

**Hook system expansion** (to 17 events, 4 handler types):
- Events: SessionStart (matcher: startup/resume/clear/compact), SessionEnd, UserPromptSubmit, PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest, SubagentStart, SubagentStop, Stop, TeammateIdle, TaskCompleted, ConfigChange, WorktreeCreate, WorktreeRemove, PreCompact, Notification
- Handler types: `command` (shell), `prompt` (LLM eval), `agent` (multi-turn), `http` (POST)

---

## The Only Real Gap: Structured Shared State

claude-flow's SQLite shared memory gives all agents in a swarm read/write access to the same structured store. Claude Code's native solution is the task list + per-agent memory dirs. This covers most cases.

If you need richer cross-agent shared state, build a simple MCP server:

```python
# shared-state-mcp/server.py — ~100 lines
# SQLite-backed key/value + structured tables
# Exposes: read_state(key), write_state(key, value), query_state(sql)
```

This is one afternoon of work. Not a reason to depend on claude-flow.

---

## What Dev-House Needs and How to Get It Natively

| Dev-House Need | Native Solution |
|----------------|----------------|
| PRD analysis + architecture | Agent Teams (analyst, architect, reviewer roles) |
| Code generation coordination | Custom subagents via frontmatter (coder, tester, reviewer) |
| Inter-agent communication | Agent Teams mailbox + task list |
| Quality gates | TeammateIdle + TaskCompleted hooks |
| Persistent learnings | Subagent `memory` field + CLAUDE.md per subagent |
| Prompt injection detection | PreToolUse hook + validation script |
| Context preservation | PreCompact + SessionStart(compact) hooks |
| Structured shared state | Simple SQLite MCP server (build it) |
| Workflow with dependencies | Agent Teams task list + auto-unblock |
| Observability | Hook logging + verbose mode + transcript files |

---

## Borrowing from claude-flow Without the Dependency

The most useful thing claude-flow contributes is its **agent role definitions**. The 54 agent types encode useful system prompts and tool scopes for common development roles. These can be translated directly into Claude Code custom subagent frontmatter files — no claude-flow runtime required.

Worth reviewing: the coder, reviewer, tester, researcher, devops, and security-auditor agent definitions. Extract the system prompts and tool scopes. Discard the framework.

---

## Sources

- Claude Code Agent Teams: code.claude.com/docs/en/agent-teams
- Claude Code Custom Subagents: code.claude.com/docs/en/sub-agents
- Claude Code Hooks: code.claude.com/docs/en/hooks-guide
- Claude Agent SDK: platform.claude.com/docs/en/agent-sdk/overview
- claude-flow issue #958 (v3 not performing work)
- claude-flow issue #945 (v3 rebuild, memory leaks, SQL injection)
- claude-flow issue #1082 (repositioning vs Agent Teams)
- VoltAgent awesome-claude-code-subagents: 100+ community definitions

# Model Router

**Which model handles which task.** A routing table for AI task allocation — use this when deciding whether to spawn a subagent and which model to select.

---

## Core Principle

Model selection is a cost/capability trade-off. Match the model to the task complexity — don't over-provision (wastes cost) or under-provision (produces poor results).

| Tier | Model | Use When |
|------|-------|----------|
| **Low** | Haiku | Fast, repetitive, well-defined — formatting, extraction, classification |
| **Medium** | Sonnet | Standard development — code generation, refactoring, analysis |
| **High** | Opus | Deep reasoning, architecture, ambiguous problems requiring judgment |

---

## Task Type Routing Table

| Task Type | Complexity | Recommended Model | Notes |
|-----------|------------|-------------------|-------|
| **PRD parsing** — extract requirements, identify ambiguities | High | Opus | Ambiguity detection requires judgment |
| **Architecture decision** — recommend stack, decompose system | High | Opus | Wrong decisions are expensive to undo |
| **Ticket generation** — produce self-contained implementation tickets from PRD | Medium | Sonnet | Structured output, well-defined format |
| **Code generation** — implement a feature from a ticket | Medium | Sonnet | Standard dev work |
| **Terraform generation** — produce IaC from architectural spec | Medium | Sonnet | Structured, template-driven |
| **Code review** — flag issues, suggest improvements | Medium | Sonnet | Analysis with context |
| **Validation** — check generated code against acceptance criteria | Medium | Sonnet | Structured comparison |
| **Linting / formatting** — fix style, apply conventions | Low | Haiku | Mechanical, rules-based |
| **Classification** — safe/unsafe, route to queue, categorise | Low | Haiku | High-volume, fast turnaround |
| **Summarisation** — condense long docs or session output | Low | Haiku | Extract key points, reduce tokens |
| **Security scan** — detect prompt injection, secrets in code | Low→Medium | Haiku → Sonnet | Start fast; escalate if uncertain |
| **Failure analysis** — diagnose why a task failed | High | Opus | Requires cross-context reasoning |
| **Multi-customer coordination** — orchestrate concurrent jobs | High | Opus | Scheduling, conflict resolution |
| **Documentation generation** — write or update docs from code | Medium | Sonnet | Standard writing task |

> This table is a starting point. Update it as you discover task types that route poorly — the table should reflect actual observed output quality, not theory.

---

## Subagent Handover Contract

When spawning a subagent, the prompt must be **fully self-contained**. The subagent has zero session context.

### Required Elements

```markdown
## Task
[One-paragraph description of exactly what to do]

## Context
- Project: dev-house
- Working directory: /home/<user>/proj/dev-house
- Task type: [from routing table above]

## Files to Read First
- [ ] `docs/architecture/decisions.md` — architectural decisions
- [ ] `CLAUDE.local.md` — patterns, criminal offences, gotchas
- [ ] `[specific doc]` — [why relevant to this task]

## Inputs
[File paths, content, or data the subagent needs]

## Expected Output
[Exact format: file to write, fields to populate, schema to follow]

## Scope Boundary
- In scope: [what to do]
- Out of scope: [what NOT to touch]
- Definition of done: [acceptance criteria]

## Model Selection
[Haiku / Sonnet / Opus] — [one-line reason]
```

### Anti-patterns

- **Partial context**: "Fix the bug" — subagent doesn't know which bug
- **Implicit files**: Referencing files without paths — subagent will hallucinate locations
- **Open-ended scope**: "Make it better" — subagent will make arbitrary changes
- **Missing output format**: Subagent will invent a format that doesn't integrate

---

## Self-Learning: Updating This Table

When a task produces unexpectedly poor output, add an entry here:

| Date | Task Type | Routed To | Outcome | Better Route |
|------|-----------|-----------|---------|--------------|
| *(add entries as discovered)* | | | | |

Pattern: if Sonnet output on task X consistently requires Opus-level revision, update the routing table.

---

## Integration Points

- **Harness dispatch** — reads task type from ticket, looks up model here, sets `model` field in queue task
- **Worker daemon** — uses `model` field from queue task to select provider
- **CLAUDE.local.md §When Implementing** — references this file for spawn decisions
- **~/.claude/CLAUDE.md §Orchestrator Rule** — references this file for model selection

# Feature Ticket Template

**Version**: 1.0.0
**Changelog**:
- 1.0.0 (2026-03-01): Initial version

> **Use for**: New functionality, new endpoints, new services, new integrations

---

## Provenance

| Field | Value |
|-------|-------|
| PRD source | `examples/<customer-prd>.md § <section>` |
| Target repo | `harness` / `codex` / `openclaw` / `customer/<name>` |
| Harness type | `anthropic-harness-v1` / `openai-swarm-v1` / `openclaw-v1` / `manual` |
| Template | `standards/feature.md v1.0.0` |
| Created | YYYY-MM-DD |
| Concurrency tier | `parallel-safe` / `sequential` / `blocking` |
| Depends on | `#<issue-id>` or `none` |
| Blocks | `#<issue-id>` or `none` |

---

## Problem Statement

> **SPECIFIC, not prescriptive.** Describe what business outcome is missing or broken.
> Bad: "Add a function to validate the PRD"
> Good: "Harness cannot validate that a PRD's `target_cloud` field is a supported provider, causing silent downstream failures in Terraform generation"

[Problem statement here]

---

## Why This Matters

- Business motivation from PRD: [section ref]
- Customer/user impact: [what breaks or is missing without this]
- Priority rationale: [why now vs. later]

---

## Acceptance Criteria

> Verifiable, testable, outcome-focused. Not implementation steps.

- [ ] [Criterion 1]
- [ ] [Criterion 2]
- [ ] [Criterion 3]

---

## Scope

**In scope:**
- [What must be built]

**Out of scope:**
- [Explicitly excluded — prevents scope creep]

**Definition of done:**
- All acceptance criteria pass
- Relevant tests written and passing
- Docs updated per `docs/documentation-maintenance.md`

---

## Documentation Reference

> **MANDATORY.** Read every doc listed here before writing any code.
> Subagents have no session context — these links are their knowledge base.

**Always read:**
- [ ] `docs/architecture/decisions.md` — Why things are structured this way
- [ ] `docs/architecture/getting-started.md` — Full architecture context
- [ ] `CLAUDE.local.md` — Project patterns, gotchas, criminal offences

**Specific to this ticket:**
- [ ] `docs/<path>` — [why relevant]
- [ ] `docs/<path>` — [why relevant]

**Search before write:**
- [ ] Grep `src/` for: `<function_or_pattern_name>` before creating new code
- [ ] Reference implementation (if exists): `src/<path>:<line>`

---

## Context for Implementer

> Supply what a subagent cannot discover: pre-computed state, relevant file locations, existing infrastructure to reuse.

**Key files:**
- `src/<path>:<line>` — [what it does, why relevant]

**Related tickets:**
- `#<id>` — [relationship]

**Constraints:**
- [Technical constraints — state the constraint, not the solution]
- [If only one viable approach due to constraints, explain the constraint explicitly]

---

## Quality Check (Before Creating This Ticket)

- [ ] Is the problem statement specific enough that two implementers would build compatible things?
- [ ] Are acceptance criteria verifiable (test-able) rather than subjective?
- [ ] Is the Documentation Reference section complete?
- [ ] Have I described the outcome (not the solution approach)?
- [ ] Is the scope boundary clear enough to prevent drift?

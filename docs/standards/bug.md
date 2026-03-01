# Bug Ticket Template

**Version**: 1.0.0
**Changelog**:
- 1.0.0 (2026-03-01): Initial version

> **Use for**: Incorrect behavior, failures, regressions, silent errors

---

## Provenance

| Field | Value |
|-------|-------|
| PRD source | `examples/<customer-prd>.md § <section>` or `none (operational)` |
| Target repo | `harness` / `codex` / `openclaw` / `customer/<name>` |
| Harness type | `anthropic-harness-v1` / `openai-swarm-v1` / `manual` |
| Template | `standards/bug.md v1.0.0` |
| Created | YYYY-MM-DD |
| Concurrency tier | `parallel-safe` / `sequential` / `blocking` |
| Depends on | `#<issue-id>` or `none` |
| Severity | `critical` / `high` / `medium` / `low` |

---

## Problem

**What is broken:**
[Specific description of the incorrect behavior — not "it doesn't work", but what exactly fails]

**Expected behavior:**
[What should happen]

**Actual behavior:**
[What actually happens]

---

## Reproduction Steps

1. [Step 1]
2. [Step 2]
3. [Observed: ...]

**Logs / evidence:**
```
[Paste relevant log lines or error output]
```

---

## Symptom Area

> Check all that apply — helps narrow investigation before grepping

- [ ] Harness orchestration
- [ ] Codex generation
- [ ] OpenClaw infrastructure
- [ ] Security pipeline (Phase 1/2/3)
- [ ] Provisioning / deployment
- [ ] Testing / validation
- [ ] Other: [describe]

---

## Acceptance Criteria

- [ ] Reproduction steps no longer produce the error
- [ ] Regression test added to prevent recurrence
- [ ] Root cause documented in `docs/gotchas.md`

---

## Documentation Reference

> **MANDATORY.** Read every doc listed here before investigating.

**Always read:**
- [ ] `docs/architecture/decisions.md`
- [ ] `CLAUDE.local.md` — Check gotchas section first; this bug may already be documented

**Specific to this bug:**
- [ ] `docs/<path>` — [why relevant]

---

## Quality Check (Before Creating This Ticket)

- [ ] Can this be reproduced with the steps above?
- [ ] Is the symptom area identified?
- [ ] Is severity set correctly?
- [ ] Will a regression test prevent recurrence?

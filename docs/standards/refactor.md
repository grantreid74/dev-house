# Refactor Ticket Template

**Version**: 1.0.0
**Changelog**:
- 1.0.0 (2026-03-01): Initial version

> **Use for**: Code improvement without behavior change — extracting patterns, reducing duplication, improving clarity, renaming

---

## Provenance

| Field | Value |
|-------|-------|
| PRD source | `none (technical debt)` or `examples/<prd>.md § <section>` |
| Target repo | `harness` / `codex` / `openclaw` |
| Harness type | `anthropic-harness-v1` / `manual` |
| Template | `standards/refactor.md v1.0.0` |
| Created | YYYY-MM-DD |
| Concurrency tier | `parallel-safe` / `sequential` / `blocking` |
| Depends on | `#<issue-id>` or `none` |

---

## Current State

**What exists:**
[Describe the current code structure — what pattern, where it lives, with file paths and line numbers]

**Why it's a problem:**
- Duplication: [where the same logic exists in N places]
- Clarity: [what's hard to understand or misleading]
- Maintainability: [what makes changes risky or slow]

---

## Target State

**What it should look like after:**
[Describe the target structure — what module, function, or pattern. Be specific about where the canonical version lives.]

**Behavior preservation:**
This is a refactor — no new features, no bug fixes, no behavior changes. All existing behavior must be preserved. Verify with existing tests.

---

## Acceptance Criteria

- [ ] All existing tests pass unchanged
- [ ] No behavior change (verified by test suite or explicit manual check)
- [ ] Duplication eliminated: [what was N copies → 1 canonical location]
- [ ] Documentation updated if module/function names changed

---

## Rename/Refactor Discipline (CRITICAL)

If any identifier is renamed (function, class, variable, file, CLI command, config key):

- [ ] Grep entire repo for old name before committing: `grep -r "old_name" src/ tests/ docs/`
- [ ] Update ALL references: source files, tests, docs, CLI help text, config, README
- [ ] No exceptions — one missed reference = production bug (this has happened)

---

## Documentation Reference

> **MANDATORY.** Read every doc listed here before writing any code.

**Always read:**
- [ ] `docs/architecture/decisions.md`
- [ ] `CLAUDE.local.md` — Patterns, criminal offences, DRY enforcement rules

**Read before changing (the files being refactored):**
- [ ] `src/<path>` — [full read before editing]
- [ ] `src/<path>` — [full read before editing]

**Specific to this ticket:**
- [ ] `docs/<path>` — [why relevant]

---

## Quality Check (Before Creating This Ticket)

- [ ] Is this purely a refactor (no behavior change intended)?
- [ ] Is the rename/refactor discipline checklist present?
- [ ] Do existing tests cover the behavior being preserved?
- [ ] Are all files that contain the old pattern listed in Documentation Reference?

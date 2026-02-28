# Naming Standards Router (Quick Reference)

**Use this to quickly determine which naming pattern applies to what you're creating.**

---

## Quick Router

```
What are you creating?

Documentation?
├─ Sequential/workflow (ordered phases)? → 01_name.md, 02_name.md, 03_name.md
├─ Snapshot/archive (time-specific)? → YYYYMMDD_HHMM_name.md
└─ Permanent reference? → lowercase-kebab-case.md

Code/script? → verb-noun.ext (in src/)
  Example: generate-repo.py, validate-prd.js, deploy.sh

Config file? → noun.ext
  Example: deployment-config.yaml, docker-compose.yml

Directory? → lowercase
  Example: docs/, src/, scripts/, templates/
```

---

## Quick Examples

| Type | Pattern | Example |
|------|---------|---------|
| Workflow phase 1 | SEQUENTIAL | `01_process-overview.md` |
| Workflow phase 2 | SEQUENTIAL | `02_daily-operations.md` |
| Today's snapshot | SNAPSHOT | `20260228_1457_session-notes.md` |
| Architecture doc | DESCRIPTIVE | `critical-separations.md` |
| Code | FUNCTIONAL | `generate-repo.py` |
| Config | CONFIG | `deployment-config.yaml` |

---

## Rules (Always)

1. **Lowercase** — all filenames lowercase (except code identifiers)
2. **Kebab-case** — use hyphens not underscores (except timestamps)
3. **Descriptive** — names should be self-explanatory
4. **Consistent** — once you pick a pattern for a context, stick with it

---

## When in Doubt

1. Changes frequently or time-specific? → Add date (SNAPSHOT)
2. Part of an ordered series? → Add number (SEQUENTIAL)
3. Permanent reference? → Just describe it (DESCRIPTIVE)
4. Performs an action? → Verb-noun (FUNCTIONAL)
5. Data/settings? → Noun + extension (CONFIG)

---

## Context-Specific Guidance

**Documentation** (in `docs/`):
- Workflow guides → 01_, 02_, 03_
- Reference docs → just the name (critical-separations.md)
- Getting-started guides → verb-noun (getting-started.md)

**Code** (in `src/`):
- Tools/scripts → verb-noun (generate-repo.py)
- Utilities → noun (logger.py, config_parser.py)

**Snapshots** (root or .claude/memory/):
- Session notes → YYYYMMDD_HHMM_name
- Always add date if it's temporary or changes frequently

**Configuration** (root):
- Always descriptive + extension (.yaml, .json, .yml)

---

**Full details**: See `NAMING-STANDARDS.md` in root for complete patterns, rationale, and examples.

**Repository reference**: Check `docs/README.md` for how all docs are organized.

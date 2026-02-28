# Naming Standards Router

**Principle**: Different contexts require different naming patterns. This document routes you to the correct pattern based on what you're creating.

---

## Router: What Are You Creating?

```
START
├─ Documentation file?
│  ├─ Is it sequential/workflow (part of ordered series)?
│  │  └─ Use SEQUENTIAL pattern (01_name.md)
│  ├─ Is it a snapshot/archive (time-specific, non-stateful)?
│  │  └─ Use SNAPSHOT pattern (YYYYMMDD_HHMM_name.md)
│  └─ Is it permanent reference/guide?
│     └─ Use DESCRIPTIVE pattern (name.md)
├─ Code/script file?
│  └─ Use FUNCTIONAL pattern (verb-noun or noun)
├─ Configuration file?
│  └─ Use CONFIG pattern (noun, with appropriate extension)
└─ Directory?
   └─ Use DIRECTORY pattern (lowercase, plural when collection)
```

---

## Naming Patterns

### 1. SEQUENTIAL Pattern
**Context**: Documentation that forms an ordered workflow/process across multiple documents
**Format**: `NN_descriptive-name.md`
**Rationale**: Numbering preserves order; files naturally sort and flow correctly without relying on external index

**Examples**:
- `01_process-overview.md` — Phase 1: understand the complete cascade
- `02_daily-operations.md` — Phase 2: execute the procedures
- `03_failure-recovery.md` — Phase 3: handle failures

**When to use**:
- Workflow documentation
- Process guides with phases
- Learning paths with prerequisites
- Sequential tutorials

---

### 2. SNAPSHOT Pattern
**Context**: Non-stateful documents capturing state at a specific moment (session notes, summaries, archives)
**Format**: `YYYYMMDD_HHMM_descriptive-name.md`
**Rationale**: Timestamp preserves chronology; multiple snapshots coexist without overwriting; clearly marks as temporal

**Examples**:
- `20260228_1457_foundation-summary.md` — Snapshot of architecture foundation (Feb 28, 2:57 PM)
- `20260228_1430_session-notes.md` — Today's session learnings
- `20260301_0900_pattern-discovery.md` — New patterns found during exploration

**When to use**:
- Session summaries
- Dated insights or learnings
- Archive snapshots
- Non-authoritative notes (vs authoritative docs)
- Anything that changes frequently or is time-specific

---

### 3. DESCRIPTIVE Pattern
**Context**: Permanent, stateful documentation (architecture, guides, references)
**Format**: `lowercase-kebab-case.md`
**Rationale**: Descriptive names immediately convey content; lowercase is unix/GitHub convention; kebab-case is readable

**Naming approach**:
- **Noun-noun**: Describe what the doc contains (preferred for reference material)
  - `critical-separations.md` — What are the critical separations?
  - `process-overview.md` — Overview of the process
  - `failure-recovery.md` — How to recover from failures

- **Adjective-noun**: Describe the type of thing
  - `local-development-environment.md` — The local dev environment setup
  - `multi-provider-strategy.md` — Multi-provider deployment strategy

- **Verb-noun**: Describe the action/guide (for how-to docs)
  - `getting-started.md` — How to get started
  - `pattern-selection-workbook.md` — How to select a pattern

**Examples**:
- `architecture/overview.md` — System design overview
- `harness/orchestration.md` — Harness orchestration engine
- `deployment/deployment-patterns.md` — Available deployment patterns
- `security/prompt-security.md` — Security pipeline for prompts

**When to use**:
- Architecture documents
- Reference guides
- Permanent design decisions
- Conceptual explanations
- Tutorials and how-to guides

---

### 4. FUNCTIONAL Pattern
**Context**: Code and scripts (what they DO, not what they are)
**Format**: `verb-noun.ext` or `noun.ext` (in `src/` or `scripts/`)
**Rationale**: Functions/scripts are tools; naming them by action makes their purpose clear

**Examples**:
- `generate-repo.py` — Generates a customer repo from PRD
- `validate-prd.js` — Validates PRD for completeness
- `deploy-infrastructure.sh` — Deploys infrastructure
- `fetch-customer-config.ts` — Fetches customer configuration
- `test-harness.py` — Tests the harness
- `migration.py` — Data migration script

**When to use**:
- Python/JavaScript/TypeScript files in `src/`
- Shell scripts in `scripts/`
- Utility functions and tools
- CLI commands

---

### 5. CONFIG Pattern
**Context**: Configuration files (data, not code)
**Format**: `noun.ext` (lowercase, no spaces, appropriate extension)
**Rationale**: Configs are data artifacts; named for what they configure, not what they do

**Examples**:
- `deployment-config.yaml` — Deployment configuration
- `codex-templates.json` — Code generation templates
- `feature-flags.json` — Feature flag definitions
- `.env.local` — Local environment variables
- `docker-compose.yml` — Docker Compose configuration

**When to use**:
- YAML, JSON, TOML configuration
- Environment files (.env)
- Docker/compose files
- Infrastructure definitions
- Template data

---

### 6. DIRECTORY Pattern
**Context**: Folder organization (structural, organizational)
**Format**: `lowercase` (singular or plural depending on content)
**Rationale**: Directories are organizational containers; consistent casing for navigation

**Examples**:
- `docs/` — Documentation root
- `docs/architecture/` — Architecture documents
- `docs/workflow/` — Workflow documents
- `src/` — Source code
- `scripts/` — Utility scripts
- `tests/` — Test files
- `templates/` — Reusable templates
- `examples/` — Example configurations

**When to use**:
- Always for directories
- Plural for collections (`templates/`, `examples/`)
- Singular for functional areas (`src/`, `tests/`)

---

## Decision Tree Examples

### "I'm documenting our deployment architecture"
```
Is it sequential? No (it's reference material)
Is it a snapshot? No (it's permanent)
Use DESCRIPTIVE: deployment-architecture.md
```

### "I wrote a session summary today"
```
Is it sequential? No
Is it a snapshot? YES (today's work)
Use SNAPSHOT: 20260228_1457_session-summary.md
```

### "I'm creating phase 2 of the workflow guide"
```
Is it sequential? YES (part of ordered workflow)
Is it a snapshot? No
Use SEQUENTIAL: 02_daily-operations.md
```

### "I'm writing a script to validate PRDs"
```
Is it code/script? YES
Use FUNCTIONAL: validate-prd.py (or validate-prd.js)
```

### "I'm creating deployment configuration for customers"
```
Is it configuration? YES
Use CONFIG: deployment-config.yaml
```

---

## Applied to Dev-House

### Documentation Files (in `docs/`)
- **Sequential workflow** (phases in order):
  - `docs/workflow/01_process-overview.md`
  - `docs/workflow/02_daily-operations.md`
  - `docs/workflow/03_failure-recovery.md`

- **Permanent reference** (DESCRIPTIVE):
  - `docs/architecture/critical-separations.md`
  - `docs/architecture/anthropic-harness-pattern-extended.md`
  - `docs/harness/orchestration.md`
  - `docs/deployment/deployment-patterns.md`

### Snapshots (root, dated)
- `20260228_1457_foundation-summary.md`
- `20260301_0900_session-notes.md`
- `.claude/memory/20260228_1457_patterns.md`

### Code (in `src/`)
- `src/harness/generate-repo.py`
- `src/codex/validate-prd.js`
- `src/deployment/deploy-infrastructure.sh`

### Configuration
- `deployment-config.yaml`
- `.env.local`
- `docker-compose.yml`

---

## Rules

1. **Always use lowercase** (except in URL references or code identifiers)
2. **Use hyphens not underscores** for word separation (kebab-case)
3. **No special characters** except hyphens and underscores in timestamps
4. **Be descriptive** — names should be self-explanatory
5. **Be consistent within a context** — once you choose SEQUENTIAL for workflow docs, keep that pattern
6. **Update this router** when you discover a new context that needs a pattern

---

## When in Doubt

1. **Does it change frequently or is it time-specific?** → SNAPSHOT (add date)
2. **Is it part of an ordered sequence?** → SEQUENTIAL (add number)
3. **Is it permanent reference material?** → DESCRIPTIVE (just describe it)
4. **Does it perform an action?** → FUNCTIONAL (verb-noun)
5. **Is it configuration/data?** → CONFIG (noun + extension)

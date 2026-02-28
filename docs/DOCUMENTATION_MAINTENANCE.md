# Documentation Maintenance Map

**When you do X, update Y.** This table ensures docs stay in sync with implementation.

**RULE**: Every code/architecture change includes documentation updates **in the same commit**. No deferred doc updates.

---

## The Map

### Harness Layer (Analysis & Planning)

| When You... | Update These | Why |
|-------------|--------------|-----|
| Change how we analyze PRDs | `docs/architecture/GETTING-STARTED.md`, `docs/harness/orchestration.md` | Others need to know the new analysis process |
| Add/remove a decision point in PRD analysis | `docs/deployment/pattern-selection-workbook.md`, `.claude/memory/PATTERNS.md` | Decision tree affects pattern selection |
| Discover a new architectural pattern | `docs/architecture/pattern-evolution-framework.md`, `docs/deployment/deployment-patterns.md`, `.claude/memory/PATTERNS.md` | Patterns are reusable; new patterns become templates |
| Change the feature list format/structure | `docs/architecture/anthropic-harness-pattern-extended.md` | Feature list is part of our Harness pattern |
| Modify repo structure template | `docs/deployment/customer-repository-structure.md` | Template must match what we generate |

### Codex Layer (Code Generation)

| When You... | Update These | Why |
|-------------|--------------|-----|
| Add a code generation template | `docs/codex/generation.md`, `src/codex/` template directory | Generators must be documented and discoverable |
| Change how services are generated | `docs/deployment/customer-repository-structure.md`, `.claude/memory/PATTERNS.md` | Service structure affects repo organization |
| Discover a code generation gotcha | `GOTCHAS.md` (Code Generation), relevant service docs | Prevents others from repeating the mistake |
| Update validation rules for generated code | `docs/harness/local-development-environment.md`, tests/ | Validation is part of local dev workflow |

### Infrastructure Layer (OpenClaw & Terraform)

| When You... | Update These | Why |
|-------------|--------------|-----|
| Add a new Terraform module | `docs/terraform/multi-provider-strategy.md`, `docs/deployment/deployment-patterns.md` | Modules are building blocks; must be cataloged |
| Change provider strategy | `docs/terraform/multi-provider-strategy.md`, `docs/deployment/deployment-patterns.md` | Provider choice affects which docs apply |
| Discover infrastructure gotcha | `GOTCHAS.md` (Deployment & Infrastructure), relevant pattern docs | Operational knowledge must be captured |
| Add new cloud provider support | `docs/terraform/multi-provider-strategy.md`, new `docs/terraform/[provider]/` section | Provider differences must be documented |
| Change custom domain provisioning | `docs/deployment/custom-domain-provisioning.md`, `.claude/memory/PATTERNS.md` | High-complexity, error-prone operation |

### Operations & Workflow

| When You... | Update These | Why |
|-------------|--------------|-----|
| Change session structure (which phase is when) | `docs/workflow/DAILY-OPERATIONS.md`, `docs/workflow/PROCESS-OVERVIEW.md` | Timeline affects how teams coordinate |
| Discover a failure mode | `docs/workflow/FAILURE-RECOVERY.md`, `GOTCHAS.md` | Recovery procedures must be accurate and tested |
| Add/change emergency procedure | `docs/workflow/FAILURE-RECOVERY.md`, `docs/workflow/DAILY-OPERATIONS.md` | Emergency procedures are time-critical |
| Identify a stale operational doc | Update the doc or mark as archived, `.claude/memory/PATTERNS.md` (add note) | Stale docs cause operational confusion |

### Security

| When You... | Update These | Why |
|-------------|--------------|-----|
| Discover security vulnerability | `docs/security/prompt-security.md`, `GOTCHAS.md` (Security), `.claude/memory/PATTERNS.md` | Security knowledge must be shared immediately |
| Change threat model | `docs/security/prompt-security.md`, `.claude/memory/PATTERNS.md` | Threat model drives all security decisions |
| Implement new guard mechanism | `docs/security/local-guard-implementation.md` | Guard is critical infrastructure |
| Add security-related gotcha | `GOTCHAS.md` (Security), relevant threat docs | Security gotchas are high-priority |

### Naming & Standards

| When You... | Update These | Why |
|-------------|--------------|-----|
| Discover a new context for file naming | `NAMING-STANDARDS.md`, `.claude/memory/NAMING-STANDARDS.md` | Patterns must be documented |
| Find inconsistency in naming | `NAMING-STANDARDS.md` router, relevant docs | Consistency prevents confusion |

### Memory & Learning

| When You... | Update These | Why |
|-------------|--------------|-----|
| Discover a reusable pattern | `.claude/memory/PATTERNS.md` | Patterns enable faster implementation |
| Find something works differently than expected | `.claude/memory/PATTERNS.md`, relevant doc, `GOTCHAS.md` | Expected vs actual differences are learning points |
| Complete a major task | `.claude/memory/PATTERNS.md` (Session Learnings table) | Persistent knowledge across sessions |

---

## How to Use This Map

**Before committing code:**

1. Identify what changed (e.g., "I added a new Terraform module")
2. Look up the row in this table
3. Update each document in "Update These"
4. Include all updated docs in your commit
5. Commit message should reference what changed and why

**Example**:
```bash
# If you add a new Terraform module:
# 1. Code: src/terraform/new-module/main.tf
# 2. Docs: docs/terraform/multi-provider-strategy.md (add module description)
# 3. Docs: docs/deployment/deployment-patterns.md (show where it fits in patterns)
# 4. Memory: .claude/memory/PATTERNS.md (if it's a pattern-level discovery)
# 5. Commit:

git add src/terraform/new-module/ docs/terraform/ docs/deployment/ .claude/memory/PATTERNS.md
git commit -m "feat: add [module-name] terraform module

Added new module to support [capability].
Updated multi-provider strategy and deployment patterns documentation."
```

---

## Review Checklist

When reviewing a commit, verify:
- [ ] Code changed? (src/)
- [ ] Corresponding docs updated? (docs/)
- [ ] Gotcha discovered? (add to GOTCHAS.md)
- [ ] Pattern learned? (add to .claude/memory/PATTERNS.md)
- [ ] Naming consistency checked? (per NAMING-STANDARDS.md)
- [ ] All relevant rows from this map addressed?

---

## Stale Documentation Policy

**Quarterly review** (or when starting new context):

1. Check date of last update for each doc (in commit history)
2. Verify doc accuracy by spot-checking against implementation
3. If stale (>3 months, code changed significantly): mark with `[LAST VERIFIED: YYYY-MM-DD]` header
4. Update verification date when checked
5. If doc contradicts implementation: fix immediately, don't defer

---

## Example: Full Workflow

### Scenario: "I'm implementing multi-provider support"

**Step 1: Code changes**
```
src/terraform/aws/vpc.tf
src/terraform/azure/vnet.tf
src/terraform/gcp/vpc.tf
```

**Step 2: Find in map**
Row: "Change provider strategy" → Update:
- docs/terraform/multi-provider-strategy.md
- docs/deployment/deployment-patterns.md

**Step 3: Update docs**
- Add provider-specific section
- Show how SKUs map across providers
- Add example for each provider

**Step 4: Check for gotchas**
- If Azure behaves differently → add to GOTCHAS.md
- If discovery affects patterns → add to .claude/memory/PATTERNS.md

**Step 5: Commit**
```bash
git add src/terraform docs/terraform docs/deployment GOTCHAS.md .claude/memory/
git commit -m "feat: add multi-provider terraform support

Implemented provider-specific modules for AWS, Azure, GCP with unified interface.
Updated multi-provider strategy documentation with SKU mapping.
Added provider-specific gotchas to GOTCHAS.md."
```

---

## When in Doubt

If you're not sure which docs to update:
1. Ask: "Who would be confused if this doc wasn't updated?"
2. Answer: Update docs for those people
3. Commit with all related docs

Remember: **Same commit = code + docs. Always.**

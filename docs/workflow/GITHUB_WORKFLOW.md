# GitHub as Project Narrative

**GitHub issues and PRs are the single source of truth for context and evolution.** Issues capture the "why" and "what"; PRs and commits show the "how"; together they form a complete story of project decisions and technical evolution.

---

## Philosophy

**Problem**: Git commit messages and code alone don't capture *why* decisions were made.

**Solution**: GitHub issues as narrative history.
- **Issues** = decisions, discoveries, problems, questions
- **PRs** = implementation, test results, review feedback
- **Git + GitHub together** = complete project story

**Result**: Six months later, you can read the issue history and understand:
- Why we made this architectural choice
- What alternatives were considered
- What broke and how we fixed it
- Who discovered the gotcha and when

---

## Issue as Primary Unit

### Issue = One Discrete Idea

**One issue = one of**:
- One feature/capability
- One bug/fix
- One architectural decision
- One infrastructure component
- One pattern discovery
- One refactoring
- One investigation/research question

**NOT**:
- "Implement everything" (too broad)
- "Fix bugs" (unclear scope)
- "Add features" (which ones?)

**Name**: `[type] description` where type is:
- `feat:` — New feature/capability
- `fix:` — Bug fix or issue resolution
- `refactor:` — Code restructuring
- `docs:` — Documentation improvement
- `infra:` — Infrastructure/deployment
- `research:` — Investigation/learning
- `pattern:` — New reusable pattern

**Examples**:
- `feat: add multi-provider terraform support`
- `fix: docker volume mounting in local dev`
- `docs: clarify harness pattern for new team members`
- `pattern: discovered three deployment tiers`
- `research: compare self-hosted vs cloud costs`

### Issue Description = Context

**Every issue must answer**:
1. **What**: What are we doing/fixing/learning?
2. **Why**: Why matters this? What problem does it solve?
3. **Context**: What's the background? What docs/code is affected?
4. **Acceptance Criteria**: How do we know it's done?
5. **Unknowns**: What are we uncertain about?

**Template**:

```markdown
# [Type] Issue Title

## What
[Clear, specific description of the work]

## Why
[Business value, problem it solves, or learning goal]

## Context
[Related issues, relevant docs, architectural context]

## Acceptance Criteria
- [ ] [Clear, testable criterion 1]
- [ ] [Clear, testable criterion 2]
- [ ] [Tests pass/coverage maintained]
- [ ] [Documentation updated per DOCUMENTATION_MAINTENANCE.md]

## Notes
[Unknowns, questions, gotchas discovered]
```

**Example**:

```markdown
# feat: add multi-provider terraform support

## What
Implement provider-specific Terraform modules for AWS, Azure, and GCP with unified interface. Customers specify cloud provider in PRD; Harness selects appropriate Terraform templates.

## Why
Single-provider approach limits customer choice and increases support burden. Multi-provider support reduces vendor lock-in and enables cost optimization (customers pick cheapest provider for their region).

## Context
- Related: Deployment pattern selection (DOCUMENTATION_MAINTENANCE.md)
- Affects: docs/terraform/multi-provider-strategy.md
- Depends on: Pattern selection workbook (determines provider choice)

## Acceptance Criteria
- [ ] AWS, Azure, GCP modules implemented with unified interface
- [ ] Provider-specific SKU mapping documented
- [ ] Example: Deploy same Postgres across all three providers
- [ ] docs/terraform/multi-provider-strategy.md updated
- [ ] Integration test: provision test database on each provider

## Notes
- Azure resource naming conventions differ significantly (underscores vs hyphens)
- GCP project IDs are global-unique (vs AWS account-scoped)
- Will discover Terraform provider quirks during implementation
```

---

## PR as Implementation Story

### PR Title
**Format**: `[type(scope)]: description — closes #123`

**Examples**:
- `feat(terraform): add aws module for postgres — closes #45`
- `docs(harness): clarify pattern evolution for new teams — closes #78`
- `fix(docker): resolve volume mount issue in local dev — closes #12`

### PR Description = Review Context

**When opening a PR**:
1. **Reference the issue**: `Closes #123` (GitHub auto-links)
2. **Explain the approach**: Why did you solve it this way?
3. **What changed**: High-level summary
4. **How to test**: Verification steps
5. **Gotchas/notes**: Anything reviewers should know

**Template**:

```markdown
## Summary
[One sentence: what this PR accomplishes]

## Approach
[How did you solve it? Why this way vs alternatives?]

## Changes
- [What changed in component A]
- [What changed in component B]

## Test Plan
- [ ] [Manual test 1]
- [ ] [Automated test 1]
- [ ] [Cross-platform check if applicable]

## Related Issues
Closes #123
Related to #456

## Notes
[Gotchas, surprises, questions]
[If applicable: "Discovered gotcha X, added to GOTCHAS.md"]
[If applicable: "Updated docs per DOCUMENTATION_MAINTENANCE.md — see row: 'Add new Terraform module'"]
```

**Example**:

```markdown
## Summary
Added AWS Terraform module for Postgres with unified interface matching Azure/GCP implementations.

## Approach
Created provider-specific module following pattern from Azure (VPC → Subnet → Database). AWS uses AWS-specific resource naming but outputs (endpoint, port, username, password_secret) match interface contract.

## Changes
- `src/terraform/aws/postgres.tf` — AWS RDS implementation
- `docs/terraform/multi-provider-strategy.md` — Added AWS section with SKU mapping
- `GOTCHAS.md` — Added "AWS security group rules don't persist via Terraform" with workaround

## Test Plan
- [ ] Apply AWS module with test configuration
- [ ] Connect to RDS instance from EC2
- [ ] Verify outputs match expected interface
- [ ] Spot-check Azure/GCP outputs still work
- [ ] Added integration test in tests/terraform/

## Related Issues
Closes #45
Related to #40 (pattern selection), #42 (cost analysis)

## Notes
- AWS security groups behavior differs significantly from Azure NSGs
- RDS parameter groups require manual updates in some cases (documented in GOTCHAS)
- Discovered: CloudFormation vs Terraform for AWS has subtle differences in output naming
```

---

## Issue Lifecycle & Linking

### Issue States

**Open** → **In Progress** → **Review** → **Closed**

**Workflow**:
1. **Open**: Issue created, awaiting assignment
2. **In Progress**: Someone is actively working on it
3. **Review**: PR opened, waiting for code review
4. **Closed**: PR merged, work complete

**GitHub labels**:
- `status: open` — Not started
- `status: in-progress` — Active work
- `status: ready-for-review` — PR open, code review needed
- `status: blocked` — Waiting on something else
- `status: closed` — Complete

### Issue → PR → Commit → Closed

**Complete story**:

```
Issue #45: "Add AWS Terraform module"
    ↓
PR #47 references #45
    ↓
Commit message references #47
    "feat: add aws postgres module (PR #47)"
    ↓
PR merged, #45 auto-closes
    ↓
git log shows commit → git log --format="%B" shows PR reference
    ↓
GitHub shows full thread: issue → discussion → PR → commit
```

### Cross-Referencing

**Always link related issues**:

```markdown
# Issue: Add Azure Terraform module

Related to:
- #40 (Pattern selection drives provider choice)
- #42 (Cost analysis: compare providers)
- #43 (Research: Azure SKU mapping)

Blocked by:
- #39 (Decision: which providers to support)

Blocks:
- #55 (Implement provider selection in Harness)
```

---

## Discoveries & Gotchas in Issues

### Gotchas During Implementation

**In PR comment or final issue comment**:

```markdown
## Gotchas Discovered

While implementing this, discovered:

1. **Docker volume mounts don't work with WSL2 on Windows**
   - Problem: Mounted volumes show as read-only
   - Why: WSL2 file permissions model differs from Linux
   - Workaround: Use named volumes instead of bind mounts
   - Added to GOTCHAS.md (Docker & Local Development section)

2. **Terraform AWS provider timeout on first apply**
   - Takes 5-10 minutes for initial security group provisioning
   - Solution: Increase timeout, don't panic
   - Added to GOTCHAS.md (Deployment & Infrastructure section)
```

**Result**: Issues become the narrative of discoveries, not just "work completed."

### Pattern Discoveries

**When a new pattern emerges**:

```markdown
# pattern: discovered three deployment tiers

## What
During implementation of Tier 1, Tier 2, and Tier 3 patterns, realized they form a coherent progression.

## Tiers
1. **Fully Shared** ($25-100/mo) — All customers in same infrastructure
2. **Namespace Isolated** ($50-200/mo) — Shared K8s, namespace per customer
3. **Infrastructure Isolated** ($100-300/mo) — Dedicated Container Apps per customer

## Why This Matters
- Each tier trades off cost vs isolation
- Customer PRD determines which tier is appropriate
- Patterns are composable (e.g., Tier 2 uses some Tier 1 components)
- Updated pattern selection workbook with decision tree

## Evidence
- CompylotAI uses Tier 3 successfully in production
- Tier 1 suitable for MVPs and free tiers
- Each tier has proven deployment path

## Documentation Updated
- docs/deployment/deployment-patterns.md (full specs)
- docs/deployment/pattern-selection-workbook.md (decision tree)
- .claude/memory/PATTERNS.md (added to learnings table)

## Related Issues
- #30 (CompylotAI deployment)
- #35 (New customer cost analysis)
```

---

## Query the Story

**With good issues + PRs, you can ask**:

```bash
# When was feature X decided?
gh issue list --search "feat: X" --state all

# What broke when we changed Y?
gh issue list --search "fix: Y" --state all

# When did we discover gotcha Z?
gh issue list --search "gotcha" --state all
gh issue view <number> | grep -A5 "Gotchas Discovered"

# What alternatives did we consider for decision D?
gh issue view <number> # Read full discussion thread

# Who knows about pattern P?
gh issue list --search "pattern: P"
# Then check PR comments and review history
```

---

## Rules

1. **One issue per discrete idea** — No "do everything" issues
2. **Issue description must explain why** — Not just what
3. **Acceptance criteria must be testable** — Clear definition of done
4. **PR must reference issue** — "Closes #123" creates link
5. **Gotchas discovered go to GOTCHAS.md** — AND mention in issue comment
6. **Patterns discovered get documented** — Issue + docs + memory update
7. **Every PR mentions docs updated** — Per DOCUMENTATION_MAINTENANCE.md
8. **Issue history is project history** — Don't delete old issues, close them

---

## Links

- **[../../DOCUMENTATION_MAINTENANCE.md](../../DOCUMENTATION_MAINTENANCE.md)** — What docs to update per change
- **[../../GOTCHAS.md](../../GOTCHAS.md)** — Add gotchas discovered during implementation
- **[../../.claude/memory/PATTERNS.md](../../.claude/memory/PATTERNS.md)** — Add new patterns to Session Learnings
- **[../../REVIEW_POLICY.md](../../REVIEW_POLICY.md)** — How to review issues for accuracy
- **[PROCESS-OVERVIEW.md](PROCESS-OVERVIEW.md)** — Complete workflow cascade (discovery to delivery)

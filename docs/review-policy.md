# Content Review & Validation Policy

**Self-review as a policy.** Ensures documentation accuracy, identifies stale content, and keeps docs in sync with implementation.

---

## Three Levels of Review

### Level 1: Per-Commit Review (Immediate)

**When**: Before every commit
**Who**: The person making changes
**Checklist**:

- [ ] Code changed?
- [ ] Corresponding docs updated? (check DOCUMENTATION_MAINTENANCE.md)
- [ ] New gotcha discovered? (add to GOTCHAS.md)
- [ ] New pattern learned? (add to .claude/memory/PATTERNS.md)
- [ ] Naming consistent? (check NAMING-STANDARDS.md)
- [ ] Links still valid? (updated file paths in cross-references?)
- [ ] Markdown renders correctly?

**Automation**: Pre-commit hook can check:
- File existence (links don't point to deleted docs)
- Markdown syntax
- DOCUMENTATION_MAINTENANCE.md coverage

### Level 2: Session Review (Every few commits)

**When**: End of implementation session or every 5-10 commits
**Who**: Self-review during session
**What to check**:

1. **Consistency**: Same terminology across all docs? (not "PRD", "product requirements", "business spec" all mixed)
2. **Accuracy**: Do examples still match actual behavior?
3. **Completeness**: Did I miss any doc updates from the map?
4. **Freshness**: Any gotchas discovered that need to be added?
5. **Links**: Are there cross-references that need updating?

**Verification checklist**:
```markdown
- [ ] Searched for old terminology in docs (find & replace if needed)
- [ ] Spot-checked examples against actual code/behavior
- [ ] Verified all links resolve
- [ ] Added any discovered gotchas
- [ ] Updated memory files with learnings
```

### Level 3: Quarterly Deep Review

**When**: End of quarter or major milestone
**Who**: Self-review against full implementation
**Scope**: Entire docs/ directory

**Procedure**:

1. **Staleness Check**
   ```bash
   # Find docs not touched in 3+ months
   find docs/ -name "*.md" -mtime +90
   ```
   For each:
   - [ ] Read the doc
   - [ ] Verify against current implementation
   - [ ] If accurate: add `[LAST VERIFIED: 2026-03-28]` header, update date
   - [ ] If inaccurate: fix it immediately
   - [ ] If no longer relevant: archive (move to docs/archived/) or delete with explanation in commit

2. **Completeness Check**
   - [ ] Are there features/patterns in code not documented in docs/?
   - [ ] Are there docs that describe features no longer in code?
   - [ ] Did we add new directories/modules that need doc sections?

3. **Accuracy Spot Check**
   ```
   - [ ] Run through one full workflow (analysis → code gen → deployment)
   - [ ] Verify docs match actual behavior at each step
   - [ ] Note any discrepancies, fix them
   ```

4. **Pattern Library Review**
   - [ ] Are all patterns in docs/deployment/deployment-patterns.md actually used?
   - [ ] Are there patterns in code not documented?
   - [ ] Did gotchas reveal gaps in pattern definitions?

5. **Update Memory Files**
   - [ ] Review .claude/memory/PATTERNS.md for stale entries
   - [ ] Remove patterns that are no longer used
   - [ ] Add new patterns discovered since last review
   - [ ] Update "Session Learnings" table

---

## Staleness Markers

To make it easy to identify old docs:

**Add to top of document**:
```markdown
# Document Title

[LAST VERIFIED: 2026-02-28] — This doc was verified against implementation on this date. Stale after 90 days.
```

When updating the doc: change the date.

When reviewing: if date is >90 days old, re-verify the content.

---

## Gotcha Lifecycle

Gotchas should have explicit status:

```markdown
### [Category] Issue Title (Discovered: 2026-02-28, Status: ACTIVE)

**Problem**: ...
**Why**: ...
**Solution**: ...
**Status**:
- ACTIVE — Issue still present, workaround needed
- FIXED — Issue resolved in [version/PR/date]
- WORKS_AROUND — Workaround established, monitoring for fix
- ARCHIVED — Issue no longer relevant

**Last Verified**: 2026-02-28
```

---

## Validation Questions

Ask these questions about every major doc section:

**For Architecture Docs**:
- [ ] Does this doc match current architectural decisions?
- [ ] Are examples still accurate?
- [ ] Have we learned something that contradicts this doc?

**For Operational Docs**:
- [ ] Have we run through this procedure recently?
- [ ] Does it still work?
- [ ] Have there been gotchas we didn't document?

**For Code Generation Docs**:
- [ ] Do examples match what Claude actually generates now?
- [ ] Are there new patterns Claude discovered?
- [ ] Are there cases that break the pattern?

**For Pattern Docs**:
- [ ] Have we used this pattern on a new customer PRD?
- [ ] Did it work as described?
- [ ] What would we do differently?

---

## When to Flag Content for Review

Add `[REVIEW NEEDED]` marker if:
- You know a doc is outdated but don't have time to fix it now
- You discovered a contradiction but aren't sure which doc is right
- Code changed significantly and docs might be affected
- You got confused by a doc (signals unclear writing)

**Format**:
```markdown
[REVIEW NEEDED: 2026-02-28] — This section contradicts GOTCHAS.md#docker-volumes. Need to verify which is correct.
```

During Level 2 or 3 review, search for `[REVIEW NEEDED]` and resolve all flags.

---

## Automation & Tooling

### Pre-Commit Hook Example
```bash
#!/bin/bash
# Verify DOCUMENTATION_MAINTENANCE.md coverage

changed_files=$(git diff --cached --name-only)

if echo "$changed_files" | grep -q '^src/'; then
  echo "Code files changed. Have you updated relevant docs?"
  echo "Check DOCUMENTATION_MAINTENANCE.md for what to update."
  exit 1
fi
```

### Markdown Validation
```bash
# Find broken links
find docs/ -name "*.md" -exec grep -o '\[.*\](#)' {} + | grep '#)' | sort | uniq
```

### Staleness Report
```bash
# Find docs not updated in 90+ days
find docs/ -name "*.md" -mtime +90 -printf "%T@ %p\n" | \
  sort -n | awk '{print $2}' | \
  xargs -I {} git log -1 --format="%ai" {} | paste - -
```

---

## Culture

**This is not bureaucracy.** This is:
- Preventing future-you from being confused
- Catching documentation drift early
- Capturing operational knowledge before it's lost
- Making it safe for new team members to onboard

**One rule**: If you change implementation, check the map (DOCUMENTATION_MAINTENANCE.md) and update docs in the same commit.

No deferred doc updates. No tech debt. No "we'll document it later."

---

## Links to Other Docs

- **DOCUMENTATION_MAINTENANCE.md** — Map showing what to update when
- **GOTCHAS.md** — Captured edge cases and unexpected behaviors
- **NAMING-STANDARDS.md** — File naming standards (cacheable in memory)
- **.claude/memory/PATTERNS.md** — Session-persistent pattern library
- **.claude/memory/NAMING-STANDARDS.md** — Quick reference for naming

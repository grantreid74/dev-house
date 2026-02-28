# Document Lifecycle & Storage Structure

**How documents move through the project as they age from transient → temporal → archived → removed.**

---

## Directory Structure

```
dev-house/
├── .transient/              # Temporary work (cleaned up regularly)
│   └── [current-session]/   # Work-in-progress, notes, drafts
├── .archive/                # Historical/reference docs (kept, not active)
│   ├── sessions/            # Session summaries, organized by date
│   │   └── 20260228_1457_session-summary.md
│   ├── snapshots/           # Point-in-time snapshots (foundation, patterns)
│   │   └── 20260228_foundation-summary.md
│   └── deprecated/          # Old patterns/approaches (no longer used)
│       └── old-approach-name.md
├── docs/                    # Active, permanent documentation
├── src/                     # Code (permanent)
└── [root files]             # Standard, permanent (CLAUDE.local.md, README.md, etc.)
```

---

## Document Lifecycle

### Stage 1: Transient (Days 0-1)
**What**: Work-in-progress, drafts, temporary notes
**Location**: `.transient/[session-or-context]/`
**Lifespan**: Current session only
**Example**:
- Brainstorm notes
- Draft architecture before committing
- Exploration logs
- TODO lists for current session

**Cleanup**:
- Move to permanent location OR
- Delete if no longer needed
- Do this before session ends

**Rule**: Never reference .transient/ docs from permanent docs. They don't exist tomorrow.

### Stage 2: Temporal (Weeks 0-2)
**What**: Session summaries, point-in-time snapshots, dated learnings
**Location**: `.archive/sessions/`, `.archive/snapshots/`, `.claude/memory/`
**Lifespan**: Session-to-session reference (2-12 weeks)
**Example**:
- `20260228_1457_session-summary.md` — What we did this session
- `20260228_foundation-summary.md` — Architecture snapshot at this point
- `.claude/memory/20260228_1457_PATTERNS.md` — Patterns learned this session

**Naming**: `YYYYMMDD_HHMM_description.md` (date prefix = temporal marker)

**Purpose**:
- Reference back to "what did we know on Feb 28?"
- Compare snapshots to see what changed
- Learn from past sessions
- Debug "when did this change?"

**Lifecycle**:
- Created during session
- Committed to git
- Moved to `.archive/` if you want to keep for reference
- Or deleted if no longer useful
- **Keep for ~3 months**, then review for archive or deletion

**Rule**: Reference these in permanent docs only for "as of [date]" comparisons. Don't rely on them as current truth.

### Stage 3: Archived (Months 2+)
**What**: Historical reference, superseded approaches, old patterns
**Location**: `.archive/deprecated/`, `.archive/sessions/[old-dates]/`
**Lifespan**: Indefinite (git history forever)
**Example**:
- Old deployment pattern (superseded by new one)
- Session summary from 6 months ago
- Approach that didn't work out

**Purpose**:
- "How did we used to do this?"
- Historical reference
- Understanding evolution of decisions
- Compliance/audit trail

**Rule**: Archive docs are READ-ONLY. If you need to change it, write a new one don't edit archived.

**Cleanup**:
- Remove if: Old pattern is now harmful/confusing
- Keep if: Historical reference valuable
- Add note: "ARCHIVED [date] — Superseded by [new approach]"

### Stage 4: Removed
**What**: Docs that no longer serve a purpose
**Action**: Delete from `.archive/` and git
**When**:
- Truly obsolete (not just old)
- Contradicts current architecture
- Replaced and no need for historical record
- Taking up mental load

**Git policy**: Delete with commit message explaining why
```bash
git rm .archive/deprecated/old-pattern.md
git commit -m "cleanup: remove old-pattern.md - superseded by new-pattern, no historical value"
```

---

## Decision Tree: Where Does This Doc Go?

```
START: You've created/completed a document

Is it permanent, always-current documentation?
├─ YES → goes to docs/ (permanent home)
└─ NO → go to next question

Is it a temporary draft/note for this session only?
├─ YES → goes to .transient/ (delete before session end)
└─ NO → go to next question

Is it a snapshot/summary for this session?
├─ YES → goes to .archive/sessions/ with YYYYMMDD_HHMM_ prefix
└─ NO → go to next question

Is it a point-in-time snapshot (architecture, patterns)?
├─ YES → goes to .archive/snapshots/ with YYYYMMDD_HHMM_ prefix
└─ NO → go to next question

Is it a deprecated/superseded approach?
├─ YES → goes to .archive/deprecated/ with note "ARCHIVED: reason"
└─ NO → Where does it actually belong?
```

---

## Rules

### Root Directory
**What goes in root**: Only permanent, always-referenced files
- README.md — Project entry point
- CLAUDE.local.md — Project configuration
- NAMING-STANDARDS.md — Standards (permanent)
- DOCUMENTATION_MAINTENANCE.md — Standards (permanent)
- REVIEW_POLICY.md — Standards (permanent)
- GOTCHAS.md — Living document of issues (permanent)

**What does NOT go in root**:
- SESSION_SUMMARY.md ❌ (move to .archive/sessions/)
- FOUNDATION_SUMMARY.md ❌ (move to .archive/snapshots/)
- Dated snapshots ❌ (use YYYYMMDD_HHMM_ prefix and .archive/)
- Drafts/notes ❌ (use .transient/)

### Archive Directory Rules
1. **Immutable**: Once archived, don't edit. Write new docs instead.
2. **Dated**: All files should have creation date or YYYYMMDD prefix
3. **Linked**: Add "ARCHIVED [date] — see [new doc] for current approach"
4. **Indexed**: Keep `.archive/README.md` listing what's there and why

### Transient Directory Rules
1. **Temporary**: Cleaned up before session ends
2. **Session-scoped**: Deleted or moved to permanent/archive
3. **Not referenced**: Never link to .transient/ from permanent docs
4. **Auto-cleanup**: Consider weekly review: delete stale drafts

---

## Cleanup Schedule

### Before Committing
- [ ] Move SESSION_SUMMARY.md → .archive/sessions/20260228_1457_session-summary.md
- [ ] Move FOUNDATION_SUMMARY.md → .archive/snapshots/20260228_foundation-summary.md
- [ ] Delete .transient/ contents (or move to archive if valuable)

### Per-Session (End of Session)
- [ ] Review .transient/ — keep or move to archive?
- [ ] Commit any permanent changes
- [ ] Clean up temporary files

### Monthly
- [ ] Review .archive/sessions/ — keep, consolidate, or delete?
- [ ] Check .archive/snapshots/ — any that should be deleted?
- [ ] Update .archive/README.md with current contents

### Quarterly
- [ ] Full review of .archive/ directory
- [ ] Identify truly obsolete docs for deletion
- [ ] Check if deprecated/ patterns should be removed
- [ ] Archive old memory files that are no longer needed

---

## Example: Moving SESSION_SUMMARY.md

**Current**: `/root/SESSION_SUMMARY.md` (wrong place)

**Action**:
```bash
# Move to archive with proper naming
mv SESSION_SUMMARY.md .archive/sessions/20260228_1457_session-summary.md

# Update .archive/sessions/README.md to list it
echo "- 20260228_1457_session-summary.md" >> .archive/sessions/README.md

# Commit
git add .archive/sessions/ && git rm SESSION_SUMMARY.md
git commit -m "housekeeping: move session summary to archive with date prefix

SESSION_SUMMARY.md → .archive/sessions/20260228_1457_session-summary.md

Temporal documents (dated snapshots) belong in .archive/, not root.
This keeps root clean and roots docs in time."
```

---

## Links

- **[NAMING-STANDARDS.md](NAMING-STANDARDS.md)** — File naming patterns (includes YYYYMMDD_HHMM_ for temporal docs)
- **[REVIEW_POLICY.md](REVIEW_POLICY.md)** — Staleness detection (helps identify what to archive)
- **[DOCUMENTATION_MAINTENANCE.md](DOCUMENTATION_MAINTENANCE.md)** — When to update docs

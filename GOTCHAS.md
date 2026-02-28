# Gotchas & Edge Cases

**What works in theory but breaks in practice.** Captured as we discover them. Link from relevant docs.

## Guidelines for This File

- **Add immediately** when discovering something that doesn't work out of the box
- **Include**: What you tried, why it failed, what works instead
- **Link from docs** that would mislead someone without this knowledge
- **Date entries** so we know when something was discovered (might be fixed in newer versions)
- **Remove when fixed** — if an issue is resolved, delete the entry and note in commit message

---

## Current Gotchas

### (To be filled in as we discover them during implementation)

---

## Categories

### Configuration & Setup
*(Things that don't work with default settings)*

### Code Generation
*(Subtle issues with Claude-generated code)*

### Deployment & Infrastructure
*(Production surprises not obvious from docs)*

### Docker & Local Development
*(Local dev != production after all)*

### Git Workflow
*(State management, branching, merging)*

### Testing & Validation
*(Tests pass locally but fail in cloud)*

### Performance & Cost
*(Unexpected costs or bottlenecks)*

### Security
*(Threats not obvious from threat model)*

---

## Template for New Gotchas

```markdown
### [Category] Issue Title (Date)

**Problem**: What you tried and what went wrong

**Why**: Root cause analysis

**Solution**: What actually works

**Prevention**: How to avoid in future

**Reference**: Link to relevant doc that needs updating
```

---

## How to Add

1. Create gotcha with problem → why → solution
2. Link from the relevant doc (e.g., "See GOTCHAS.md: Docker build layers" from local-development-environment.md)
3. Add to appropriate category above
4. Include date of discovery
5. Commit with message: "gotcha: [category] [title] - [brief description]"

---

## How to Remove

When a gotcha is fixed (e.g., new version of tool, new approach works):

1. Delete the entry
2. Commit with message: "resolved: [category] [title] - [what fixed it]"
3. Update linked doc to remove GOTCHAS reference if no longer needed

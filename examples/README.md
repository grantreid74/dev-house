# Examples

Example PRDs and configurations for evaluating Dev-House patterns and pipeline inputs.

---

## PRD Examples

These four PRDs cover the full range of deployment tiers. Use them to:
- Evaluate the PRD format and template
- Stress-test pattern selection against real-ish scenarios
- Prototype Harness analysis before a real customer arrives

| File | Customer | Tier | Complexity | Key Features |
|------|----------|------|------------|--------------|
| [prd-acme-task-manager.md](prd-acme-task-manager.md) | Acme Corp | Tier 1 (Shared) | Low | Startup, no compliance, minimal budget |
| [prd-brightpath-compliance.md](prd-brightpath-compliance.md) | BrightPath | Tier 3 (Isolated) | Medium | Mid-market SaaS, SOC2, multi-tenant |
| [prd-carelink-scheduling.md](prd-carelink-scheduling.md) | CareLink | Tier 4 (Enterprise) | High | Healthcare, HIPAA, PHI, AWS |
| [prd-finsight-analytics.md](prd-finsight-analytics.md) | FinSight | BYOC | Very High | Enterprise, existing AWS, PCI-DSS |

---

## PRD Template

[prd-template.md](prd-template.md) — Blank template. Copy this for every new customer PRD.

### How the Harness Uses a PRD

```
PRD arrives
    ↓
Harness Initializer (Session 1, ~2 hours):
    ├── Customer Profile → pattern selection (Tier 1-4 / BYOC)
    ├── Core Features → SERVICE_DECOMPOSITION.yaml
    ├── Core Features → feature_list.json (N items, 0/N passing)
    ├── NFRs → ARCHITECTURAL_DECISION.md (compliance, scale, infra)
    ├── Open Questions → flagged for human review before proceeding
    └── Git commit: "feat: initial architecture for [customer]"
    ↓
Codex Generators (Sessions 2+):
    Pick feature from feature_list.json → implement → test → commit → mark passing
```

The **Core Features** section is the source of truth for `feature_list.json`. Write features as discrete, independently implementable items — the Harness expands them into granular tasks.

The **Open Questions** section causes the Harness to pause and request human input before generating code. Minimise open questions by resolving ambiguities before PRD sign-off.

---

## Anthropic Harness Alignment

Dev-House PRDs align with Anthropic's Harness pattern:
- Initializer → `claude-progress.txt` equivalent: `harness-status.yaml`
- Feature list → `features.json` equivalent: `feature_list.json`
- Coding loop: one feature per session, commit on passing

See: [docs/architecture/anthropic-harness-pattern-extended.md](../docs/architecture/anthropic-harness-pattern-extended.md)

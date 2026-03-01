# Infrastructure Ticket Template

**Version**: 1.0.0
**Changelog**:
- 1.0.0 (2026-03-01): Initial version

> **Use for**: Terraform changes, Docker configuration, K8s manifests, CI/CD pipelines, cluster topology

---

## Provenance

| Field | Value |
|-------|-------|
| PRD source | `examples/<customer-prd>.md § <section>` or `none (operational)` |
| Target repo | `openclaw` / `harness` / `customer/<name>` |
| Cloud provider | `azure` / `aws` / `gcp` / `multi` |
| Deployment tier | `Tier 1` / `Tier 2` / `Tier 3` / `Tier 4` / `BYOC` |
| Harness type | `openclaw-v1` / `manual` |
| Template | `standards/infra.md v1.0.0` |
| Created | YYYY-MM-DD |
| Concurrency tier | `parallel-safe` / `sequential` / `blocking` |
| Depends on | `#<issue-id>` or `none` |

---

## Problem Statement

[What infrastructure capability is missing or broken — business outcome focused, not implementation steps]

---

## Acceptance Criteria

- [ ] `terraform validate` passes
- [ ] `terraform fmt -recursive` passes
- [ ] Terraform plan shows expected changes only (no surprises, no unintended resource recreations)
- [ ] Deployed to staging environment successfully
- [ ] [Additional criteria]

---

## Scope

**In scope:**
[What infrastructure changes are required]

**Out of scope:**
[Explicitly excluded — application code changes, unrelated resources, other cloud providers]

**Resources affected:**
- [ ] Container Apps / K8s Deployments
- [ ] Databases / caches
- [ ] Networking / VNet / DNS
- [ ] Identity / Keycloak / secrets
- [ ] CI/CD pipelines
- [ ] Monitoring / alerting
- [ ] Other: [describe]

---

## Documentation Reference

> **MANDATORY.** Read every doc listed here before writing any Terraform.

**Always read:**
- [ ] `docs/architecture/decisions.md`
- [ ] `docs/deployment/deployment-patterns.md` — Confirm tier selection is correct
- [ ] `docs/terraform/multi-provider-strategy.md` — Provider-specific module patterns
- [ ] `CLAUDE.local.md` — Gotchas and criminal offences

**Specific to this ticket:**
- [ ] `docs/<path>` — [why relevant]

**Terraform gotchas (mandatory review):**
- Terraform outputs are strings: `bool("false") == True` — always compare with `== "true"`
- Adding `count` to existing resources changes state address — add `moved` blocks first
- Azure async operations: custom domains, certs, soft-delete have 5-15 min windows — use non-blocking timeout + log
- Azure CustomScript Extension uses `/bin/sh` not bash — no `set -o pipefail`, no `[[ ]]`, no process substitution
- Terraform heredoc parameter expansion: escape shell vars with `$$`

---

## Context for Implementer

**Module structure:**
- Provider-specific modules: `src/openclaw/modules/<component>/{aws,azure,gcp}/main.tf`
- Logical interface contract: `instance_size: small/medium/large` → maps to provider SKUs
- All modules expose: `endpoint`, `port`, `username`, `password_secret_name`

**Naming convention:**
- Terraform resources: `<provider>_<resource_type>_<logical_name>`
- Module outputs: consistent logical names regardless of provider

---

## Quality Check (Before Creating This Ticket)

- [ ] Is the cloud provider and deployment tier identified?
- [ ] Are Terraform gotchas reviewed — especially `count` on existing resources?
- [ ] Will async operations need polling or non-blocking timeout?
- [ ] Is `moved` block needed for any existing resources that are gaining `count`?

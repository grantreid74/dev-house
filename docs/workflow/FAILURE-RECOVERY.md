# Failure Modes & Recovery Procedures

**What can go wrong and how to fix it without losing work.**

---

## Design Principle: Git as System of Record

All state lives in git:
- Feature implementation → git commits
- Progress tracking → feature_list.json in git
- Architecture decisions → ARCHITECTURAL_DECISION.yaml in git
- Infrastructure → Terraform in git

**If anything breaks**: Replay from last git commit.

---

## Harness Analysis Failures

### Failure: Ambiguous Service Decomposition

**Symptom**: Generated feature list has confusing service boundaries or too many/too few services

**Root Cause**: PRD was unclear or Harness misunderstood requirements

**Detection**:
- Feature list seems inconsistent (e.g., feature in wrong service)
- Customer says "that's not what we wanted"
- Services don't align with business team's thinking

**Recovery**:
1. Customer clarifies requirements in PRD
2. Harness re-analyzes with clearer constraints
3. Regenerate ARCHITECTURAL_DECISION.md
4. Git commit: "fix: clarify service decomposition"
5. Codex reads updated decisions (doesn't re-run if not committed yet)

**Prevention**:
- Use pattern selection workbook before analysis
- Have human review Harness output before proceeding
- Add customer review checkpoint: "Does this match your intent?"

---

## Codex Generation Failures

### Failure: Generated Code Doesn't Match Dockerfile

**Symptom**: `docker compose up` fails; app won't start

**Root Cause**: Generated code has wrong dependencies or assumes different environment

**Detection**:
- Docker build fails (missing dependency)
- App crashes on startup (wrong ENV var)
- Tests pass locally, fail in CI

**Recovery**:
1. Check Docker build error logs
2. Identify missing dependency or env var
3. Update generated code to add dependency or fix env assumption
4. Re-run `docker compose up` locally
5. Commit fix: "fix: add missing dependency X"
6. Re-run tests to verify

**Prevention**:
- Codex should generate docker-compose.yaml that matches the Dockerfile
- CI should run same tests as local development
- Version dependencies explicitly (don't use latest)

---

### Failure: Tests Pass Locally, Fail in Cloud

**Symptom**: CI/CD passes, deployment succeeds, but app crashes in production

**Root Cause**: Environment variables differ between local and cloud

**Detection**:
- Health check fails in cloud
- Logs show "connection refused" or "variable not set"
- Database queries fail mysteriously

**Recovery**:
1. Compare .env.local vs .env.production
2. Find the difference (missing var or wrong value)
3. Update docker-compose.yaml to match production env
4. Re-test locally: `docker compose up`
5. Commit: "fix: align local env with cloud"
6. Redeploy to cloud

**Prevention**:
- Docker Compose parity principle: Local must equal cloud
- Generate both .env.local AND .env.production with same variables
- Document all required environment variables in README

---

### Failure: Feature Implementation Gets Stuck

**Symptom**: Same test fails 5 times; Codex agent spinning on same feature

**Root Cause**:
- Feature is too complex
- Test requirements are contradictory
- Codex hallucinating about implementation

**Detection**:
- Same git commit being attempted repeatedly
- Error messages don't change
- Agent is in infinite loop

**Recovery**:
1. **Set timeout per feature** (e.g., 1 hour max)
2. When timeout hit, move to next feature
3. Escalate to human:
   - Add detailed notes to feature description
   - Mark as "blocked: needs clarification"
   - Have human provide implementation guidance
4. Next session: Try again with clearer constraints
5. If still stuck: Break feature into smaller pieces

**Prevention**:
- Keep features small (should take <1 hour to implement)
- Add implementation hints to feature description
- Test with dry run before committing Codex to large features

---

## OpenClaw Infrastructure Failures

### Failure: Terraform Validation Fails

**Symptom**: `terraform validate` reports syntax errors

**Root Cause**: OpenClaw generated invalid HCL syntax

**Detection**:
- `terraform validate` fails with error message
- Error shows line number and syntax issue

**Recovery**:
1. Read the error message carefully
2. OpenClaw regenerates Terraform with constraints about that error
3. Example: "Don't use variable reference in provider block"
4. Re-run validation
5. Commit: "fix: terraform syntax error on line X"

**Prevention**:
- Validate immediately after generation
- Use linting tools (terraform-lint, tflint)
- Test Terraform on previous successful customer (copy & modify)

---

### Failure: terraform plan Overruns Budget

**Symptom**: Cost estimate way higher than expected

**Root Cause**:
- OpenClaw generated oversized instances
- Pattern selection was wrong (chose Tier 4 instead of Tier 3)

**Detection**:
- `terraform plan` shows cost estimate
- Cost exceeds customer budget by 2x+

**Recovery**:
1. Review terraform plan output
2. Identify expensive resources (usually databases or compute instances)
3. Choose smaller instance size: small → tiny, medium → small, etc.
4. OpenClaw regenerates with size constraints
5. Re-run plan to verify new cost
6. Commit: "fix: reduce instance sizes to meet budget"

**Prevention**:
- Check budget BEFORE provisioning
- Use pattern selection workbook to get cost estimate
- Have approval checkpoint: "Customer approves estimated cost?"

---

### Failure: terraform apply Fails Partway

**Symptom**: Apply starts, some resources created, then fails; system in inconsistent state

**Root Cause**:
- Resource quota exceeded
- Existing resource with same name
- VNet CIDR conflict with existing network

**Detection**:
- `terraform apply` fails with error
- Some resources created, some not
- `terraform show` shows partial state

**Recovery**:

**Option 1: Cleanup & Retry**
1. `terraform destroy` (destroys what was created)
2. Fix the underlying issue (request quota, rename resource, etc.)
3. `terraform apply` again

**Option 2: State Import**
1. Manually delete conflicting resource in cloud console
2. `terraform import` the resources that were created
3. Continue with `terraform apply`

**Option 3: Rollback (Safest)**
1. Note the resource IDs that were created
2. Manually delete them in cloud console
3. Fix the Terraform code
4. Re-run apply

**Prevention**:
- Always run `terraform plan` first; review before apply
- Check resource quotas before provisioning
- Use unique naming (include customer ID): `vnet-[customer-id]-main`

---

### Failure: Let's Encrypt Certificate Stuck in Pending

**Symptom**: 30 minutes pass, certificate still not provisioned

**Root Cause**:
- Azure DNS validation hasn't completed
- Domain ownership verification failed
- DNS propagation delay

**Detection**:
- Certificate status still "Pending" after 30 minutes
- Custom domain binding shows warning icon
- Health check fails (HTTPS not available)

**Recovery**:
1. **Don't block**: Move on; cert can provision in background
2. Log warning: "Cert provisioning delayed; will retry in 5 min"
3. Check certificate status in Azure portal
4. If still pending after 1 hour:
   - Verify DNS record is correct (query: `nslookup -type=TXT domain.com`)
   - Manually trigger Let's Encrypt renewal
   - Check Azure logs for error messages

**Prevention**:
- Don't expect instant cert provisioning
- Have fallback: Use IP address endpoint until cert ready
- Test domain before binding to load balancer

---

## State Recovery: The Master Procedure

**If anything is severely broken**, use git to recover:

```bash
# Step 1: Identify last known-good commit
git log --oneline | head -20
# Find the commit message about what was working

# Step 2: Check what changed since then
git diff COMMIT..HEAD

# Step 3: Reset to known-good state
git reset --hard COMMIT

# Step 4: Investigate what went wrong
# (Now you're back to working state; you can debug safely)

# Step 5: Fix the issue
# (Make small changes, test incrementally)

# Step 6: Create new commit with fix
git commit -m "fix: description of what we fixed"
```

**Never force-push** unless absolutely necessary (breaks other agents' work)

---

## Communication: When to Ask For Help

| Scenario | Action |
|----------|--------|
| **Test fails 1x** | Debug locally (1 hour) |
| **Test fails 3x** | Ask human for clarification on feature |
| **Test fails 5x** | Mark feature as blocked; move to next |
| **Terraform validate fails** | Regenerate with syntax constraints |
| **terraform plan costs 3x budget** | Choose smaller sizes; re-estimate |
| **terraform apply fails halfway** | Rollback; fix issue; retry |
| **Cert stuck 30 min+** | Log warning; move on; retry later |
| **Git merge conflict** | Rebase on latest main; resolve conflicts |

---

## Monitoring: What Triggers Investigation?

Set up alerts for:

1. **Codex agent silence** (no commit for 1 hour) → Check logs
2. **Test failure rate** (3+ failures same feature) → Mark blocked
3. **Terraform plan fails** (validate error) → Regenerate
4. **Infrastructure timeout** (apply > 30 min) → Check logs
5. **Domain not reachable** (health check fails) → Check DNS/cert

---

## Playbook: Cascade Failure (Multiple Things Broken)

If multiple systems fail at once:

1. **Stabilize**: Use `git reset --hard [last-known-good]`
2. **Investigate**: One failure at a time
3. **Fix**: In order of criticality:
   - Is the system live? (Harness/Codex)
   - Can we provision? (OpenClaw)
   - Can customer access? (Domain/Keycloak)
4. **Verify**: Test each fix in isolation
5. **Integrate**: When all fixed, proceed through phases

---

## See Also

- **[PROCESS-OVERVIEW.md](PROCESS-OVERVIEW.md)** — Where each failure can occur
- **[docs/architecture/CRITICAL-SEPARATIONS.md](../architecture/CRITICAL-SEPARATIONS.md)** — Why separation helps recovery

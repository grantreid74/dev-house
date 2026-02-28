# Daily Operations: How Each Session Works

**Standard procedure for running each development session.**

---

## Session 0: Initial Setup (One Time)

```
Estimated time: 1-2 hours
Prerequisite: Customer PRD approved

Steps:
1. Create GitHub organization for customer
   org: github.com/[customer-name]

2. Setup Tailscale device
   - Join virtual network
   - Get unique device ID
   - Assign tokens (Harness + Codex)

3. Clone dev-house repo
   git clone https://github.com/dev-house/dev-house
   cd dev-house

4. Create worktrees
   git worktree add harness/[customer-id] -b harness/[customer-id]/main origin/main
   git worktree add codex/[customer-id] -b codex/[customer-id]/main origin/main

5. Setup local Docker environment
   docker pull [latest python/node/go images]
   docker system prune (clean old images)

Status: Ready to start sessions
```

---

## Session 1: Harness Analysis

```
Estimated time: 2-4 hours
Input: Customer PRD

Step 1: Spawn Agent
├── cd harness/[customer-id]
├── claude-harness analyze-prd < prd.md
└── Agent runs in background (~2 hours)

Step 2: Review Output
├── Check: ARCHITECTURAL_DECISION.md
├── Check: SERVICE_DECOMPOSITION.yaml
├── Check: feature_list.json (0/200 passing)
└── Check: REPO_STRUCTURE.yaml

Step 3: Verify Decisions
├── Read ARCHITECTURAL_DECISION.md
├── Cross-check against pattern selection workbook
├── Verify compliance level, deployment pattern, cost estimate
└── If wrong: Ask Harness to re-analyze with constraints

Step 4: Commit to Git
├── git add -A
├── git commit -m "feat: initial architecture for [customer]"
├── git push origin harness/[customer-id]/main

Step 5: Approve for Codex
├── Get human review (optional for MVP)
├── Mark in tracking: "Architecture ready"
└── Notify: Codex can start

Output:
├── ARCHITECTURAL_DECISION.md (decisions + rationale)
├── SERVICE_DECOMPOSITION.yaml (which services, tech, deployment)
├── feature_list.json (200 features, 0/200 done)
└── REPO_STRUCTURE.yaml (separate repos vs monorepo decision)

Possible Issues:
❌ Decisions unclear → Have customer clarify PRD
❌ Services don't align → Re-analyze with guidance
❌ Cost overrun → Suggest cheaper pattern
❌ Compliance gap → Add security controls to plan
```

---

## Sessions 2-5: Codex Initialization

```
Estimated time: 2-4 hours per service (parallel across 3-4 services)
Prerequisite: Harness analysis complete

Step 1: For Each Service
├── cd codex/[customer-id]
├── Spawn Codex subagent (e.g., "generate-frontend")
└── Agent reads ARCHITECTURAL_DECISION.yaml

Step 2: Generate Baseline Code
├── Create GitHub repo: [customer-name]/frontend
├── Generate directory structure: src/, tests/, config/
├── Generate Dockerfile (production)
├── Generate docker-compose.yaml (local dev)
├── Generate .github/workflows/ (CI/CD)
├── Generate README.md
└── Generate feature_list.json (this service's features)

Step 3: Initial Commit
├── git add -A
├── git commit -m "feat: initial [service] scaffold"
├── git push origin main

Step 4: Automated Checks
├── Run: docker build (verify Dockerfile)
├── Run: npm ci && npm test (or equivalent)
├── Run: security scan (dependencies)
└── Wait for checks to pass

Step 5: Merge PR
├── If all checks pass: Merge to main
├── If checks fail: Codex fixes issues; re-commit
└── Mark: "[service] ready for feature implementation"

Output:
├── [service] GitHub repo created
├── Baseline code (Dockerfile, docker-compose.yaml)
├── CI/CD pipelines (test, build, deploy)
├── feature_list.json (e.g., 45 features for backend)
└── All services: baseline ready

Possible Issues:
❌ Docker build fails → Fix Dockerfile
❌ Tests fail → Fix test setup or dependencies
❌ Security scan blocks → Update dependencies
❌ GitHub access denied → Check credentials/org settings
```

---

## Sessions 6+: Feature Implementation Loop

```
Estimated time: 4 hours per feature × 100 features = 400+ hours
Timeline: Multiple sessions per day, daily commits

Each Feature (Repeat):

Step 1: Load Session State
├── cd [customer-name]-[service]/
├── git pull origin main
├── git log --oneline | head -5
├── cat feature_list.json | jq '.[] | select(.status=="pending")' | head -1
└── Understand: Where are we? What's left?

Step 2: Select Feature
├── Pick one feature from feature_list.json
├── Status: pending (not started)
├── Example: "POST /auth/login endpoint"
└── Create feature branch: feature/auth-login

Step 3: Implement
├── git checkout -b feature/auth-login
├── Write code: handler, validation, tests
├── Update docker-compose.yaml if needed
└── git add src/ tests/

Step 4: Validate Locally
├── docker compose down (clean slate)
├── docker compose up (start all services)
├── npm test (run tests)
├── curl http://localhost:8000/auth/login (manual test)
└── Check logs for errors

Step 5: Fix Issues
If tests fail:
├── Read error message
├── Fix code or test
├── Re-run: docker compose up && npm test
├── Repeat until all pass

Step 6: Commit Feature
├── git add -A
├── git commit -m "feat: implement POST /auth/login endpoint"
├── Message: [service][feature] implemented"

Step 7: Create PR
├── git push origin feature/auth-login
├── GitHub creates PR automatically
├── PR title: "feat: implement POST /auth/login endpoint"

Step 8: Code Review
├── Automated review runs (security, style, tests)
├── Human reviews (architecture, approach)
├── If approved: Merge to main
├── If changes needed: Update code; repeat from Step 3

Step 9: Update Progress
├── Pull latest main: git pull origin main
├── Update feature_list.json:
│   ├── Find: {name: "POST /auth/login", status: "pending"}
│   └── Change: {name: "POST /auth/login", status: "passing"}
├── Update: {total: 45, completed: 12}
├── git add feature_list.json
├── git commit -m "progress: 12/45 features done"
├── git push origin main

Output Per Session:
├── 1-3 features implemented
├── N commits to main
├── Tests passing
├── feature_list.json updated
└── Ready for next session

Possible Issues:
❌ Test fails locally → Debug; fix code
❌ Merge conflict → Rebase on latest main; resolve
❌ Feature takes > 4 hours → Mark blocked; escalate
❌ Local test passes, CI fails → Env variable mismatch
❌ Code review blocks → Address feedback; re-commit
```

---

## Sessions N-M: Infrastructure Provisioning

```
Estimated time: 1 hour per component (10-15 components)
Prerequisite: All services complete (all features passing)

Step 1: Generate Terraform
├── cd [customer-name]-infrastructure/
├── Clone all service repos (to read Dockerfile, etc.)
├── OpenClaw reads: ARCHITECTURAL_DECISION.yaml
├── Generate: terraform/
│   ├── main.tf (provider config)
│   ├── networking.tf (VNet, subnets)
│   ├── database.tf (Postgres/SQL/etc.)
│   ├── compute.tf (Container Apps/VMs)
│   └── outputs.tf (endpoints, IDs)
└── git add terraform/
└── git commit -m "infra: generate terraform for [pattern]"

Step 2: Validate Terraform
├── terraform validate (check syntax)
├── If errors → Fix HCL syntax; re-commit
└── If OK → Proceed

Step 3: Cost Estimation
├── terraform plan -json > plan.json
├── Parse: estimated monthly cost
├── Compare to budget
├── If over budget → Choose smaller sizes; regenerate
└── If OK → Proceed

Step 4: Policy Compliance Check
├── Run OpenClaw policy checker
├── Check: Encryption enabled? Logging enabled? Security groups?
├── If violations → Add security controls to terraform; re-commit
└── If OK → Proceed

Step 5: Dry Run (terraform plan)
├── terraform plan (no changes, just show what would happen)
├── Review output for any surprises
├── If issues → Fix terraform code
└── If OK → Proceed to apply

Step 6: Apply (terraform apply)
├── terraform apply (provision actual resources)
├── Wait for completion (typically 5-15 minutes per component)
├── Check cloud console to verify resources created
└── If any failed → Rollback; investigate; retry

Step 7: Extract Outputs
├── terraform output
├── Capture:
│   ├── API endpoint URLs
│   ├── Database connection strings
│   ├── Cache endpoints
│   └── Load balancer IPs
└── Store in metadata database

Step 8: Configure Customer Apps
├── Create .env.production with values from Step 7
├── Configure Keycloak (OAuth redirects, etc.)
├── Update CI/CD secrets (API keys, credentials)
└── Commit: "config: production environment variables"

Output:
├── Live infrastructure in cloud
├── All resources verified working
├── Endpoints documented
└── Ready for deployment and custom domain setup

Possible Issues:
❌ terraform validate fails → Fix HCL syntax
❌ terraform plan costs 3x budget → Choose smaller instances
❌ terraform apply times out → Check cloud quotas; retry
❌ Resources conflict with existing → Rename or import existing
❌ Certificate stuck pending → Don't block; retry later
```

---

## Session Z: Final Delivery

```
Estimated time: 1-2 hours
Prerequisite: Infrastructure live

Step 1: Domain Setup
├── Customer provides domain (e.g., app.acme.com)
├── Create DNS records (CNAME + TXT)
├── Bind to load balancer / container app
├── Let's Encrypt provisions certificate (5-15 min)
└── Test: curl https://app.acme.com

Step 2: Keycloak Setup
├── Create realm (one per customer)
├── Add OAuth2 client (configure redirect URIs)
├── Map custom domain (auth.acme.com)
├── Create admin users for customer
└── Test: Login via OAuth2

Step 3: Monitoring Setup
├── Enable Application Insights / CloudWatch
├── Setup alerts (CPU > 80%, errors > 10/min)
├── Create dashboard for ops team
└── Send test error; verify alerts

Step 4: Final Health Checks
├── Health endpoint: curl https://app.acme.com/health
├── Auth test: Login via Keycloak
├── Database test: Query user table
├── Monitoring test: Check logs appear
└── All passing? → Go live

Step 5: Grant Access
├── Create customer admin account
├── Send credentials securely
├── Send runbook (how to manage system)
├── Schedule handoff meeting
└── Complete!

Output:
├── Customer has live system at https://app.acme.com
├── Can login via custom domain
├── Can monitor in dashboard
└── Ready for production use

Possible Issues:
❌ Domain DNS not configured → Send customer DNS setup guide
❌ Cert still pending after 1 hour → Use IP endpoint; retry later
❌ Keycloak not reachable → Check firewall rules
❌ Health check fails → Check app logs
❌ No data in monitoring → Enable app logging
```

---

## Daily Standup Questions

At start of each session, answer:

- **What was completed last session?** (git log)
- **What's the goal for today?** (which features/components)
- **What blockers exist?** (failed tests, infrastructure issues)
- **How far along are we?** (feature_list.json progress %)
- **Any escalations needed?** (unclear feature, overrun cost, stuck on bug)

---

## Emergency Procedures

### If Service Can't Start

```bash
cd [customer-name]-[service]
docker compose logs [service-name]  # See error
# Fix: dependency missing? env var wrong?
docker compose down
docker compose up  # retry
docker compose logs [service-name]  # check if fixed
```

### If Tests Won't Pass

```bash
npm test -- --verbose  # see which test fails
npm test -- -t "specific test name"  # isolate test
# Fix code based on error
npm test  # re-run
```

### If Git Merge Conflict

```bash
git pull origin main  # brings in conflict markers
# Edit file: choose our changes or theirs
git add [file]
git commit -m "fix: resolve merge conflict"
git push origin [branch]
```

### If Docker Build Fails

```bash
docker build -f Dockerfile -t debug .  # see error
# Fix: missing dependency? wrong base image?
docker build -f Dockerfile -t [service]:latest .
docker run -it [service]:latest  # test
```

---

## Session Handoff Document

At end of each session, leave notes for next session:

```markdown
## Session Summary: [Date]

### Completed
- [ Feature 1 implemented (commit abc123)
- Feature 2 (in progress, 80% done)

### Tests
- ✅ All tests passing
- ⚠️ Test X sometimes flaky (skip for now)

### Next Priority
1. Feature 3 (auth)
2. Feature 4 (database)

### Known Issues
- Docker build sometimes fails; workaround: `docker system prune`
- Env var REDIS_URL needs double-checking in compose.yaml

### Blockers
- None currently

### Notes
- Customer asked for X; not in original PRD (add if time)
- Database schema slower than expected; need to optimize queries
```

---

## See Also

- **[PROCESS-OVERVIEW.md](PROCESS-OVERVIEW.md)** — Big picture flow
- **[FAILURE-RECOVERY.md](FAILURE-RECOVERY.md)** — What to do when things break

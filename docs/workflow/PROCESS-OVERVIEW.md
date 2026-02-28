# Complete Process Flow: Discovery to Delivery

**End-to-end process showing every step, interaction point, and potential failure mode.**

---

## The Complete Cascade: All Phases

```mermaid
graph TD
    A["📋 DISCOVERY<br/>Customer PRD Arrives"] --> B["Analysis<br/>Harness Initializer"]

    B --> B1["Read & Parse PRD"]
    B1 --> B2["Analyze Requirements"]
    B2 --> B3["Decompose Services"]
    B3 --> B4["Select Pattern"]
    B4 --> B5["Plan Repos"]
    B5 --> B6["Output Decisions"]
    B6 --> C["✅ PLANNING<br/>Harness→Codex Handoff"]

    C --> D["Code Generation<br/>Codex Initializers"]
    D --> D1["Read Architectural Decisions"]
    D1 --> D2["Spawn Subagents"]
    D2 --> D3["Generate Service Repos"]
    D3 --> D4["Create Baseline Code"]
    D4 --> D5["Setup CI/CD"]
    D5 --> D6["Output: Repos Ready"]
    D6 --> E["✅ DEVELOPMENT<br/>Codex Feature Loop"]

    E --> E1["Session N: Pick Feature"]
    E1 --> E2["Implement in Code"]
    E2 --> E3["Run docker compose up"]
    E3 --> E4["Run Tests"]
    E4 --> E5["Create PR"]
    E5 --> E6["AI Code Review"]
    E6 --> E7["Merge to Main"]
    E7 --> E8["Update Progress"]
    E8 --> E9{All Features<br/>Implemented?}
    E9 -->|No| E1
    E9 -->|Yes| F

    F["✅ INFRASTRUCTURE<br/>OpenClaw Provisioning"]

    F --> F1["Read Service Repos"]
    F1 --> F2["Generate Terraform"]
    F2 --> F3["Validate Syntax"]
    F3 --> F4["Estimate Costs"]
    F4 --> F5["Check Policies"]
    F5 --> F6["Create terraform/plan"]
    F6 --> F7["Review Plan"]
    F7 --> F8["Apply: Provision"]
    F8 --> F9["Verify Deployment"]
    F9 --> F10{All Components<br/>Live?}
    F10 -->|No| F1
    F10 -->|Yes| G

    G["✅ DELIVERY<br/>Customer Ready"]
    G --> G1["Domain Provisioning"]
    G1 --> G2["Keycloak Setup"]
    G2 --> G3["Monitoring Active"]
    G3 --> G4["Customer Access"]
    G4 --> H["🎉 LIVE<br/>System Running"]

    style A fill:#e1f5ff
    style B fill:#f3e5f5
    style C fill:#fff9c4
    style D fill:#fff3e0
    style E fill:#fff3e0
    style F fill:#e8f5e9
    style G fill:#f1f8e9
    style H fill:#c8e6c9
```

---

## Phase 1: DISCOVERY (Harness Initializer)

**Timeline**: ~2-4 hours

**Goal**: Analyze PRD → Produce architectural decisions

### Process Flow

```mermaid
graph TD
    A["PRD Arrives<br/>Customer sends requirements"] --> B["Spawn Harness Agent<br/>Claude Opus"]

    B --> C["Parse PRD<br/>Extract key info"]
    C --> C1["Customer profile<br/>Team size, budget, timeline"]
    C --> C2["Technical requirements<br/>Workload type, scale, data"]
    C --> C3["Compliance needs<br/>Isolation, regulatory"]

    C1 --> D["Analyze & Decide"]
    C2 --> D
    C3 --> D

    D --> D1["Decompose<br/>into services"]
    D --> D2["Select deployment<br/>pattern"]
    D --> D3["Choose<br/>tech stack"]
    D --> D4["Plan repository<br/>structure"]

    D1 --> E["Generate Output Files"]
    D2 --> E
    D3 --> E
    D4 --> E

    E --> E1["ARCHITECTURAL_DECISION.md<br/>Detailed reasoning"]
    E --> E2["SERVICE_DECOMPOSITION.yaml<br/>Which services, tech, why"]
    E --> E3["REPO_STRUCTURE.yaml<br/>Separate repos vs monorepo"]
    E --> E4["DEPLOYMENT_PATTERN.yaml<br/>Tier 1-4 selection"]
    E --> E5["feature_list.json<br/>200 items, 0/200 passing"]

    E1 --> F["Git Commit"]
    E2 --> F
    E3 --> F
    E4 --> F
    E5 --> F

    F --> G["Push to branch:<br/>analysis/[customer-id]"]

    G --> H["✅ Output Ready<br/>for Codex"]

    style A fill:#e1f5ff
    style B fill:#f3e5f5
    style E fill:#f3e5f5
    style H fill:#fff9c4
```

### Potential Breakdown Points

| Step | Failure Mode | Detection | Recovery |
|------|--------------|-----------|----------|
| **Parse PRD** | Ambiguous or incomplete requirements | Harness output lacks detail | Ask customer for clarification |
| **Decompose services** | Too many or too few services | Feature list seems wrong | Adjust decomposition; regenerate |
| **Select pattern** | Wrong Tier chosen (too cheap or expensive) | Customer rejects cost | Use workbook; re-evaluate |
| **Choose tech stack** | Unsupported or obsolete technologies | Customer can't run code | Use approved stack list |
| **Generate files** | Conflicting decisions in output | Files contradict each other | Harness re-analyzes with constraints |
| **Git commit** | Merge conflict (multiple PRDs same time) | Push fails | Rebase on latest main |

---

## Phase 2: PLANNING (Codex Initializers)

**Timeline**: ~2-4 hours per service

**Goal**: Generate service repos with baseline code

### Process Flow

```mermaid
graph TD
    A["Harness Analysis Complete<br/>Decisions committed to git"] --> B["Codex Initializer Reads Decisions"]

    B --> C["For each service<br/>in SERVICE_DECOMPOSITION.yaml"]

    C --> D["Spawn Codex Subagent<br/>Claude Sonnet"]

    D --> E["Read Architectural Context"]
    E --> E1["ARCHITECTURAL_DECISION.md"]
    E --> E2["Feature list for service"]
    E --> E3["Tech stack selected"]

    E1 --> F["Generate Baseline Code"]
    E2 --> F
    E3 --> F

    F --> F1["Create GitHub repo<br/>customer-[service-name]"]
    F --> F2["Generate folder structure<br/>src/, tests/, config/"]
    F --> F3["Generate Dockerfile<br/>Matches production"]
    F --> F4["Generate docker-compose.yaml<br/>Local development setup"]
    F --> F5["Generate .github/workflows<br/>CI/CD pipelines"]
    F --> F6["Generate README.md<br/>How to run locally"]
    F --> F7["Generate feature_list.json<br/>All features for this service"]

    F1 --> G["Git Commit"]
    F2 --> G
    F3 --> G
    F4 --> G
    F5 --> G
    F6 --> G
    F7 --> G

    G --> H["Create PR to main<br/>code-gen/[service]-initial"]

    H --> I["Automated Checks"]
    I --> I1["Syntax validation"]
    I --> I2["Security scan"]
    I --> I3["Build test"]

    I1 --> J{All Checks<br/>Pass?}
    I2 --> J
    I3 --> J

    J -->|No| K["Fix Issues<br/>Regenerate"]
    K --> G

    J -->|Yes| L["Merge PR"]

    L --> M["✅ Service Repo Ready<br/>for feature implementation"]

    style A fill:#fff9c4
    style D fill:#fff3e0
    style F fill:#fff3e0
    style M fill:#f1f8e9
```

### Potential Breakdown Points

| Step | Failure Mode | Detection | Recovery |
|------|--------------|-----------|----------|
| **Read context** | Decisions incomplete or contradictory | Codex output is inconsistent | Harness re-analyzes and clarifies |
| **Generate repo** | GitHub org access missing | Push fails | Verify credentials, retry |
| **Generate code** | Generated code doesn't match Dockerfile | Build fails in Docker | Regenerate with constraints |
| **Dockerfile parity** | Local != production | Test passes locally, fails in cloud | Verify env vars, dependencies |
| **CI/CD generation** | Workflows target wrong cloud provider | Pipeline fails | Regenerate for correct provider |
| **Security scan** | Vulnerable dependencies in generated code | Scan blocks merge | Upgrade dependencies, regenerate |

---

## Phase 3: DEVELOPMENT (Codex Feature Loop)

**Timeline**: ~4 hours per feature (100+ features total = 400+ hours)

**Goal**: Implement services feature-by-feature

### Process Flow (Per Feature)

```mermaid
graph TD
    A["Start Session<br/>Codex agent spawned"] --> B["Load Current State"]

    B --> B1["Read feature_list.json<br/>Which features done?"]
    B --> B2["Read git log<br/>What was last commit?"]
    B --> B3["Read docker-compose.yaml<br/>What services need running?"]

    B1 --> C["Select One Feature<br/>from feature_list"]
    B2 --> C
    B3 --> C

    C --> D["Implement Feature"]

    D --> D1["Write code<br/>API endpoint, DB schema, etc."]
    D --> D2["Write tests<br/>Unit + integration tests"]
    D --> D3["Update docker-compose.yaml<br/>If new services needed"]

    D1 --> E["Validate Locally"]
    D2 --> E
    D3 --> E

    E --> E1["docker compose up"]
    E --> E2["Run: npm test / pytest"]
    E --> E3["Verify: curl endpoint"]

    E1 --> F{All Tests<br/>Pass?}
    E2 --> F
    E3 --> F

    F -->|No| G["Debug Locally<br/>Fix issues"]
    G --> D

    F -->|Yes| H["Commit to Feature Branch"]

    H --> H1["Create feature/[service]/[feature]<br/>branch"]
    H --> H2["git commit<br/>with clear message"]

    H1 --> I["Create Pull Request"]
    H2 --> I

    I --> J["Code Review"]

    J --> J1["AI Review:<br/>security, style, tests"]
    J --> J2["Human Review:<br/>architecture, approach"]

    J1 --> K{Approved?}
    J2 --> K

    K -->|No| L["Request Changes"]
    L --> D

    K -->|Yes| M["Merge to Main"]

    M --> N["Update Progress"]

    N --> N1["Mark in feature_list.json<br/>status: passing"]
    N --> N2["Update progress:<br/>50/200 features done"]
    N --> N3["Commit progress update"]

    N1 --> O{More Features<br/>to Implement?}
    N2 --> O
    N3 --> O

    O -->|Yes| P["Next Session<br/>Repeat"]
    O -->|No| Q["✅ Service Complete<br/>All features passing"]

    P --> A

    style A fill:#fff3e0
    style D fill:#fff3e0
    style F fill:#fff3e0
    style J fill:#fff3e0
    style Q fill:#e8f5e9
```

### Potential Breakdown Points

| Step | Failure Mode | Detection | Recovery |
|------|--------------|-----------|----------|
| **Load state** | Previous session's work lost | git log empty or feature_list missing | Restore from backup; regenerate |
| **Select feature** | Feature unclear or ambiguous | Generated code is guessing | Add detail to feature description |
| **Implement** | Generated code hallucinating side effects | Tests fail mysteriously | Simplify feature; add constraints |
| **Docker parity** | Works locally, breaks in cloud | Environment variables differ | Sync .env files, dockerfile |
| **Tests pass** | Tests don't test the right thing | Code works locally, fails cloud | Improve test coverage |
| **Code review** | Security issue missed | Scan catches vulnerability later | Improve review criteria |
| **Merge conflict** | Multiple agents changing same file | Git merge fails | Resolve conflict; rebase |
| **Progress tracking** | feature_list.json gets out of sync | Count doesn't match reality | Rebuild from git commits |

---

## Phase 4: INFRASTRUCTURE (OpenClaw Provisioning)

**Timeline**: ~1 hour per component (10-15 components)

**Goal**: Provision cloud infrastructure

### Process Flow

```mermaid
graph TD
    A["All Services Complete<br/>Code merged to main"] --> B["OpenClaw Reads Repos"]

    B --> B1["Clone all service repos"]
    B --> B2["Read ARCHITECTURAL_DECISION.yaml<br/>from Harness phase"]
    B --> B3["Identify infrastructure<br/>components needed"]

    B1 --> C["Generate Terraform"]
    B2 --> C
    B3 --> C

    C --> C1["Create terraform/ directory"]
    C --> C2["Generate main.tf<br/>Cloud provider specific"]
    C --> C3["Generate variables.tf<br/>Inputs for customization"]
    C --> C4["Generate outputs.tf<br/>Endpoints, IDs, etc."]
    C --> C5["Generate provider.tf<br/>AWS/Azure/GCP config"]

    C1 --> D["Validate Terraform"]
    C2 --> D
    C3 --> D
    C4 --> D
    C5 --> D

    D --> D1["terraform validate<br/>Check syntax"]

    D1 --> E{Valid?}

    E -->|No| F["Fix & Regenerate"]
    F --> C

    E -->|Yes| G["Cost Estimation"]

    G --> G1["terraform plan<br/>-json output"]
    G --> G2["Parse estimated cost"]
    G --> G3["Compare to budget"]

    G2 --> H{Within<br/>Budget?}

    H -->|No| I["Optimize<br/>Reduce scope or choose cheaper pattern"]
    I --> C

    H -->|Yes| J["Policy Check"]

    J --> J1["Run compliance policies<br/>via OpenClaw"]
    J --> J2["Check for violations<br/>Security, encryption, logging"]

    J1 --> K{Compliant?}

    K -->|No| L["Adjust Infrastructure<br/>Add security controls"]
    L --> C

    K -->|Yes| M["Deploy"]

    M --> M1["terraform apply<br/>Provision all resources"]
    M --> M2["Wait for provisioning<br/>5-15 min typical"]
    M --> M3["Verify resources<br/>Check status in cloud console"]

    M1 --> N{Deployment<br/>Successful?}
    M2 --> N
    M3 --> N

    N -->|No| O["Rollback & Debug"]
    O --> C

    N -->|Yes| P["Extract Outputs"]

    P --> P1["Get: API endpoint URLs"]
    P --> P2["Get: Database connection strings"]
    P --> P3["Get: Cache endpoints"]
    P --> P4["Get: Load balancer IPs"]

    P1 --> Q["Update Customer Config"]
    P2 --> Q
    P3 --> Q
    P4 --> Q

    Q --> Q1["Store endpoints in metadata DB"]
    Q --> Q2["Configure Keycloak<br/>OAuth redirects"]
    Q --> Q3["Update CI/CD secrets<br/>API keys, credentials"]

    Q1 --> R["✅ Infrastructure Live<br/>Ready for deployment"]

    style A fill:#f1f8e9
    style C fill:#e8f5e9
    style M fill:#e8f5e9
    style R fill:#c8e6c9
```

### Potential Breakdown Points

| Step | Failure Mode | Detection | Recovery |
|-------|-------------|-----------|----------|
| **Generate Terraform** | Provider mismatch (AWS config for Azure) | Deployment fails | Regenerate with correct provider |
| **Validate syntax** | Invalid HCL in generated code | terraform validate fails | Fix syntax errors |
| **Cost estimation** | Estimate way off from actual | Bill surprises customer | Re-estimate with realistic assumptions |
| **Policy violations** | Security issue detected too late | Compliance scan fails | Add controls before deployment |
| **Apply fails** | Resource quota exceeded | "Quota exceeded" error | Request quota increase; use smaller instance size |
| **Deployment hangs** | Let's Encrypt cert provisioning stuck | 30 min with no progress | Check certificate status; retry |
| **Resource mismatch** | Generated code conflicts with existing infra | Apply reports conflict | Rename resources; use state import |
| **Outputs missing** | Key outputs not captured | Later steps can't find endpoints | Regenerate outputs.tf |

---

## Phase 5: DELIVERY (Customer Ready)

**Timeline**: ~1-2 hours

**Goal**: Set up custom domain, identity, monitoring

### Process Flow

```mermaid
graph TD
    A["Infrastructure Live<br/>All resources provisioned"] --> B["Domain Setup"]

    B --> B1["Customer provides domain<br/>e.g., app.acme.com"]
    B --> B2["Create DNS records<br/>CNAME + TXT"]
    B --> B3["Bind domain to<br/>Load Balancer / Container App"]
    B --> B4["Request Let's Encrypt cert<br/>Azure auto-provisions"]
    B --> B5["Wait for cert<br/>5-15 minutes"]

    B1 --> C["Keycloak Setup"]
    B2 --> C
    B3 --> C
    B4 --> C
    B5 --> C

    C --> C1["Create Keycloak realm<br/>one per customer"]
    C --> C2["Add OAuth2 client<br/>Configure redirects"]
    C --> C3["Set custom domain<br/>auth.acme.com → Keycloak"]
    C --> C4["Create admin users<br/>For customer"]

    C1 --> D["Monitoring Setup"]
    C2 --> D
    C3 --> D
    C4 --> D

    D --> D1["Enable monitoring<br/>Application Insights / CloudWatch"]
    D --> D2["Setup alerts<br/>CPU, memory, errors"]
    D --> D3["Create dashboards<br/>For customer operations team"]
    D --> D4["Test: Send test error<br/>Verify alerts work"]

    D1 --> E["Final Checks"]
    D2 --> E
    D3 --> E
    D4 --> E

    E --> E1["Health checks<br/>Can we reach app.acme.com?"]
    E --> E2["Auth test<br/>Can we login via Keycloak?"]
    E --> E3["Database test<br/>Can we write/read data?"]
    E --> E4["Monitoring test<br/>Do logs appear?"]

    E1 --> F{All Checks<br/>Pass?}
    E2 --> F
    E3 --> F
    E4 --> F

    F -->|No| G["Debug & Fix"]
    G --> B

    F -->|Yes| H["Grant Access"]

    H --> H1["Create customer admin account"]
    H --> H2["Send login credentials"]
    H --> H3["Send runbook<br/>How to manage system"]
    H --> H4["Schedule handoff meeting"]

    H1 --> I["✅ LIVE<br/>System ready for use"]

    style A fill:#c8e6c9
    style B fill:#f1f8e9
    style I fill:#c8e6c9
```

### Potential Breakdown Points

| Step | Failure Mode | Detection | Recovery |
|-------|-------------|-----------|----------|
| **Domain DNS** | Customer forgets to create DNS record | Health check fails | Send DNS setup guide to customer |
| **Cert provisioning** | Let's Encrypt cert stuck in pending | 30 min timeout with no cert | Check Azure binding status; retry |
| **Keycloak realm** | Realm name conflicts with existing | Realm creation fails | Use customer ID as unique identifier |
| **OAuth redirect** | Redirect URI mismatch (app expects different domain) | Login redirects to wrong URL | Update OAuth client config |
| **Database connection** | App can't reach database from container | Health check fails | Verify VNet/security group settings |
| **Monitoring setup** | Logs not appearing in dashboard | Debugging blind | Check app logging is enabled; verify sink |
| **Health checks** | Endpoint returns 500 after provisioning | Health check fails | Check application logs; restart app |

---

## Cross-Phase: Sequence Diagrams

### Harness → Codex Handoff

```mermaid
sequenceDiagram
    participant CustPRD as Customer PRD
    participant Harness as Harness Agent
    participant Git as GitHub
    participant Codex as Codex Agents

    CustPRD ->> Harness: Here's our requirements
    Harness ->> Harness: Analyze PRD
    Harness ->> Harness: Decompose into services
    Harness ->> Harness: Select deployment pattern
    Harness ->> Git: Commit ARCHITECTURAL_DECISION.md
    Harness ->> Git: Commit feature_list.json
    Git ->> Codex: [GitHub notification]
    Codex ->> Git: Read ARCHITECTURAL_DECISION.md
    Codex ->> Codex: Plan code generation
    Note over Codex: For each service:
    Codex ->> Git: Create service repo
    Codex ->> Git: Generate baseline code
    Codex ->> Git: Create PR
    Git ->> Git: Run automated checks
    Git ->> Codex: Checks passed
    Codex ->> Git: Merge PR
```

### Codex → OpenClaw Handoff

```mermaid
sequenceDiagram
    participant Codex as Codex Agents
    participant Git as GitHub / Services
    participant OpenClaw as OpenClaw

    Note over Codex: Feature loop: 100+ features
    loop Per Feature
        Codex ->> Git: Implement feature
        Codex ->> Git: Test locally (docker compose)
        Codex ->> Git: Create PR
        Git ->> Git: Code review & merge
        Codex ->> Git: Update progress
    end

    Git ->> OpenClaw: All services complete
    OpenClaw ->> Git: Clone all repos
    OpenClaw ->> OpenClaw: Read ARCHITECTURAL_DECISION.yaml
    OpenClaw ->> OpenClaw: Generate Terraform
    OpenClaw ->> OpenClaw: terraform validate
    OpenClaw ->> OpenClaw: terraform plan (cost estimate)
    OpenClaw ->> OpenClaw: Check compliance policies
    OpenClaw ->> OpenClaw: terraform apply (provision)
    OpenClaw ->> OpenClaw: Verify resources live
    OpenClaw ->> Git: Store infrastructure metadata
```

---

## Complete Timeline

```mermaid
timeline
    title Complete Project Timeline

    Day 1 : Discovery Phase
         : Harness analyzes PRD (2-4 hours)
         : Output: ARCHITECTURAL_DECISION.md

    Day 2-3 : Planning Phase
           : Codex initializers generate repos (2-4 hours per service)
           : Output: Service repos ready

    Day 4-30 : Development Phase
            : Codex agents implement features (4 hours per feature × 100 features)
            : Daily: Feature implementation → test → commit → merge
            : Parallel: Multiple agents on different services

    Day 31-32 : Infrastructure Phase
              : OpenClaw generates Terraform (1 hour)
              : Provision cloud resources (1 hour per component)
              : Output: Live infrastructure

    Day 33 : Delivery Phase
           : Domain setup
           : Keycloak setup
           : Monitoring setup
           : Final health checks
           : Customer access granted

    Day 34+ : Live & Operational
            : Customer using system
            : Post-deployment review
            : Iterate on feedback
```

---

## Where Things Break Down (Critical Points)

| Rank | Issue | Impact | Prevention |
|------|-------|--------|-----------|
| 🔴 **1** | Harness produces ambiguous decisions | All downstream work based on unclear architecture | Use detailed PRD assessment + have human review |
| 🔴 **2** | Codex generates code that doesn't match Dockerfile | Works locally, fails in production | Verify env vars, dependencies match |
| 🔴 **3** | Feature implementation stalls (test failures) | Progress stops, agent spins on same feature | Set timeout per feature; move to next if stuck |
| 🟠 **4** | Terraform generation targets wrong provider | Deployment fails; resources created in wrong place | Verify cloud provider from ARCHITECTURAL_DECISION |
| 🟠 **5** | Infrastructure cost overruns | Customer rejects bill | Review terraform plan before apply |
| 🟠 **6** | Domain provisioning async timeout | Custom domain not working | Don't block on cert; retry in background |
| 🟡 **7** | Database connection string wrong | App can't reach database | Verify VNet/security group settings |
| 🟡 **8** | Feature list gets out of sync with git | Progress tracking wrong | Rebuild from git commits periodically |

---

## See Also

- **[GETTING-STARTED.md](../architecture/GETTING-STARTED.md)** — High-level story
- **[CRITICAL-SEPARATIONS.md](../architecture/CRITICAL-SEPARATIONS.md)** — Why separation matters
- **[anthropic-harness-pattern-extended.md](../architecture/anthropic-harness-pattern-extended.md)** — Pattern explanation
- **[execution-streams-codex-vs-harness.md](../architecture/execution-streams-codex-vs-harness.md)** — Detailed execution flows

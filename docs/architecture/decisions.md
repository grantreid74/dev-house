# Architectural Decisions

## Decision Framework

Each decision documents the **problem**, **options considered**, **chosen solution**, and **rationale**.

---

## D1: Harness-First vs Code-First

**Problem**: How do we orchestrate Claude to build infrastructure from PRDs?

**Options**:
1. Code-first: Generate code immediately, derive infrastructure
2. Harness-first: Orchestrate the entire workflow, coordinate Claude calls
3. Hybrid: Harness decides when to call code generation vs architecture planning

**Decision**: Harness-first

**Rationale**:
- **Modularity** — Different generators for different targets (REST API, batch job, etc.)
- **Validation** — Harness validates intermediate outputs before proceeding
- **Recovery** — If generation fails, harness can retry or ask for clarification
- **Caching** — Harness can cache PRD analysis, reuse for multiple generations
- **Transparency** — Customer sees workflow progress, not just "generating..."

---

## D2: State-Based Workflow vs Streaming

**Problem**: How do we track progress and enable recovery if something fails mid-workflow?

**Options**:
1. Streaming: Start → generate → deploy, no state saved
2. State-based: Track each step, save state, enable recovery
3. Hybrid: Stream results, but checkpoint state at key points

**Decision**: State-based

**Rationale**:
- **Production reliability** — If harness crashes, restart from last checkpoint
- **User debugging** — Customer sees exactly where it failed
- **Reusable artifacts** — Generated code can be reviewed, modified, re-deployed
- **Cost optimization** — Don't re-generate if intermediate output already validated

---

## D3: Single Claude Model vs Specialized Models

**Problem**: Use one Claude model for all tasks, or specialize by domain?

**Options**:
1. Single model: Use Claude 3.5 Sonnet for PRD analysis, code generation, validation
2. Specialized: Use Sonnet for code, Haiku for validation, Opus for architecture
3. Hybrid: Start with Sonnet, escalate complex decisions to Opus

**Decision**: Start with Hybrid (Sonnet primary, Opus for architecture decisions)

**Rationale**:
- **Cost** — Sonnet is cheaper; use for routine generation
- **Quality** — Opus for critical decisions (multi-tier architecture recommendations)
- **Speed** — Smaller tasks with Haiku for validation/formatting
- **Flexibility** — Easy to adjust ratios as we learn what works

---

## D4: Deployment Target Abstraction

**Problem**: Support multiple deployment targets (Docker, Terraform, Kubernetes, etc.) without coupling to harness

**Options**:
1. Hardcoded generators: Each target is a function
2. Plugin system: External generators registered at runtime
3. Template system: Generator templates with substitution

**Decision**: Plugin system

**Rationale**:
- **Extensibility** — Add new generators without harness code change
- **Versioning** — Different generator versions for different use cases
- **Testing** — Generators can be tested independently
- **Customer control** — Customers can register custom generators

---

## D5: PRD Format

**Problem**: How should customers provide PRDs? Free text? Structured format?

**Options**:
1. Free text (Markdown): Flexible, natural, hard to parse
2. Structured (JSON/YAML): Parseable, less flexible
3. Hybrid: Structured sections + free text description

**Decision**: Hybrid (structured headers + markdown content)

**Rationale**:
- **Flexibility** — Natural language where it helps (vision, requirements)
- **Parseability** — Structured sections (name, type, requirements, constraints)
- **Evolution** — Easy to add new sections as we learn
- **Readability** — Humans can read and write PRDs in editors

---

## D6: Caching Strategy

**Problem**: Minimize Claude API calls by caching analyses and generated code

**Options**:
1. No caching: Call Claude for every task
2. Full caching: Cache everything, only regenerate on demand
3. Smart caching: Cache PRD analysis, generated code, but not deployment steps

**Decision**: Smart caching

**Rationale**:
- **Cost savings** — PRD analysis is expensive; cache it
- **Freshness** — Deployments should be fresh (don't cache generated Terraform)
- **User control** — Clear invalidation (what triggers a cache miss?)
- **Implementation** — Cache key = hash(PRD content) for analysis, deploy timestamp for outputs

---

## D7: Error Handling & User Feedback

**Problem**: How do we handle failures (Claude errors, validation failures, deployment errors)?

**Options**:
1. Fail fast: Stop on first error, report to user
2. Resilient: Retry with backoff, try alternatives
3. Interactive: Ask user for clarification, generate alternatives

**Decision**: Interactive resilience

**Rationale**:
- **Production quality** — Transient failures should retry automatically
- **Transparency** — User knows what went wrong and has options
- **Recovery** — Don't lose work; let user refine and retry
- **Learning** — Harness learns from failures (log patterns)

---

## Future Decisions

*To be added as exploration reveals new architectural choices.*


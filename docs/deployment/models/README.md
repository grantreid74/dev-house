# Dev-House Deployment Models

Two separate model families — do not conflate them (see `docs/architecture/critical-separations.md`):

1. **Generation Infrastructure Models** (G-series) — where Dev-House runs its agents to generate code
2. **Customer Deployment Models** (D-series) — what we provision for the customer's running system

These are selected independently. A customer can run on D-T3 (shared foundation cloud) regardless of whether their code was generated on G-L1 (Pi cluster) or G-L3 (Mac Mini fleet).

---

## Generation Infrastructure Models (G-series)

Where Dev-House agent workloads execute: PRD analysis, Harness orchestration, Codex generation, Terraform generation, validation.

### Selection Dimensions

| Dimension | G-S1 Standalone Pi | G-L1 Pi Cluster | G-L2 Pi + Mac Hybrid | G-L3 Mac Mini Fleet | G-C1 Cloud Burst |
|-----------|-------------------|----------------|---------------------|---------------------|------------------|
| **Local GPU** | None | None | 1-2 Mac Minis (Apple Silicon) | All nodes | Via cloud (A100/H100) |
| **ARM architecture** | Yes | Yes (all nodes) | Mixed (Pi = ARM, Mac = ARM) | Yes (Apple Silicon) | No (x86 cloud VMs) |
| **Concurrent generations** | 2 pairs | 4-8 pairs | 4-8 + GPU tasks | 8-16 (Docker-per-mini) | Scales to demand |
| **Nomadic operation** | Yes (one Pi) | Yes (Tailscale) | Partial (Mac Minis heavier) | Fixed location | Yes |
| **Hardware cost (est.)** | ~$210 | $800-1,100 | $1,500-2,500 | $3,000-5,000+ | $0 upfront |
| **5-year total cost** | ~$500 | ~$2,000-2,400 | ~$3,500-4,500 | ~$5,000-7,000 | ~$15,000+ |
| **Orchestration style** | Docker Compose, single node | k3s cluster | k3s + Docker hybrid | Docker Compose per node | Managed container service |
| **GPU use case** | None | None | Local LLM, image gen | Local LLM, heavy ML | Any GPU workload |
| **Best for** | Pilot, single developer, lowest cost | API-bound codegen, nomadic | Mixed workloads, occasional GPU | GPU-heavy, fixed location | No hardware investment, burst |

### Customer Type Mapping

| Customer need | Recommended G-model | Reason |
|--------------|--------------------|-|
| First pilot, single customer | G-S1 | Lowest cost and complexity to prove the concept |
| Standard SaaS app (no ML) | G-L1 | API-bound codegen, Pi cluster sufficient |
| App with local LLM requirement | G-L2 or G-L3 | GPU needed for model serving |
| Rapid turnaround, high concurrency | G-L3 or G-C1 | Mac Mini compute or cloud scale |
| Occasional GPU, budget-conscious | G-L2 | Pi does non-GPU work, Mac covers GPU |
| Enterprise, no local hardware | G-C1 | Cloud burst from existing infra |

### Model Documents

- **[G-S1: Standalone Pi](G-S1-standalone-pi.md)** — Single Pi 5 16GB, Docker Compose, no cluster. Entry-level / pilot.
- **[G-L1: Pure Pi Cluster](G-L1-pi-cluster.md)** — ARM cluster, k3s, NVMe, Tailscale. API-bound code generation.
  - Extended docs: [G-L1/cluster-topology.md](G-L1/cluster-topology.md), [G-L1/testing-deployment-pattern.md](G-L1/testing-deployment-pattern.md)
- **[G-L2: Pi + Mac Mini Hybrid](G-L2-pi-mac-hybrid.md)** — Pi coordinator/workers + Mac Mini(s) for GPU workloads.
- **[G-L3: Mac Mini Fleet](G-L3-mac-mini-fleet.md)** — Docker-per-mini, Apple Silicon GPU, fixed location.
- **[G-C1: Cloud Burst](G-C1-cloud-burst.md)** — Cloud VMs/containers for generation. No local hardware.

---

## Customer Deployment Models (D-series)

What we provision for the customer's production system. Defined by isolation level, compliance requirements, and cost tolerance.

See **[deployment-patterns.md](../deployment-patterns.md)** for full Tier 1-4 specifications, cost analysis, and Terraform module patterns.

### Selection Dimensions

| Dimension | D-T3 Shared Foundation | D-T4 Full Isolation | D-BYOC |
|-----------|----------------------|--------------------|----|
| **Database** | Shared (schema-per-customer) | Dedicated per customer | Customer's own |
| **Compute** | Dedicated Container Apps / K8s namespace | Dedicated VNet + compute | Customer's cloud account |
| **Identity** | Shared Keycloak (realm-per-customer) | Dedicated or shared | Customer-managed |
| **Compliance** | SOC 2, moderate | HIPAA, PCI, high | Customer's own certifications |
| **Cost** | $50-200/month | $300-500+/month | Variable (customer pays) |
| **Data isolation** | Logical (schema) | Physical (separate DB server) | Complete |
| **Proven** | Yes (CompylotAI-terraform) | No (design only) | No (design only) |
| **Best for** | SMB SaaS, compliance-light | Enterprise, HIPAA/PCI | Enterprise, existing cloud, full control |

### Customer Type Mapping

| Customer profile | Recommended D-model | Reason |
|-----------------|--------------------|-|
| Startup / MVP | D-T3 | Cost-effective, proven pattern, supports scale-to-zero |
| SMB SaaS, moderate compliance | D-T3 | Logical isolation sufficient, cost efficient |
| Healthcare / Finance (HIPAA/PCI) | D-T4 | Physical data isolation required |
| Enterprise with existing cloud account | D-BYOC | Customer retains control, Dev-House provides Terraform modules |
| Multi-region, high-scale | D-T4 or D-BYOC | Dedicated infrastructure scales better |

---

## Combined Selection: Generation + Deployment

Neither selection constrains the other. Pick the G-model that suits your capacity and the D-model that suits the customer's requirements.

**Example combinations:**

| Scenario | G-model | D-model |
|----------|---------|---------|
| First pilot, proving the concept | G-S1 | D-T3 |
| Startup customer, small team | G-L1 | D-T3 |
| Healthcare customer, you have GPU needs | G-L2 | D-T4 |
| Enterprise customer, no local hardware | G-C1 | D-BYOC |
| Multi-customer batch overnight | G-L1 or G-L3 | D-T3 per customer |

---

## Upgrading Between Models

**G-model upgrades** are internal — the customer never sees them. You can move from G-L1 to G-L2 by adding a Mac Mini to your cluster with no customer impact.

**D-model migrations** affect the customer — require data migration and planned downtime. Design the migration path before committing a customer to a tier.

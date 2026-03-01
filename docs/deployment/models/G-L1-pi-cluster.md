# G-L1: Pure Pi Cluster Generation Model

**Model ID**: G-L1
**Class**: Local Generation Infrastructure
**GPU**: None
**Architecture**: ARM64 cluster (k3s)

> For detailed topology, capacity analysis, and hardware specs, see:
> - `docs/deployment/models/G-L1/cluster-topology.md` — node roles, RAM budgets, storage, expansion
> - `docs/deployment/models/G-L1/testing-deployment-pattern.md` — cluster + desktop GPU testing pattern
> - `docs/research/G-L1/` — all empirical data and analysis (Pi-context)

---

## When to Choose G-L1

- Codegen workload is API-bound (waiting for Claude/OpenAI, not local compute)
- No local GPU requirement (no local LLM serving, no image generation)
- Budget priority: lowest hardware cost, lowest electricity
- Nomadic operation: cluster travels with you (Tailscale handles networking)
- Starting point: simplest model to stand up, easiest to expand

## When NOT to Choose G-L1

- Customer PRD requires local LLM serving (switch to G-L2 or G-L3)
- Generation tasks need GPU acceleration → G-L2 or G-L3
- Fixed location, high concurrency, cost of electricity acceptable → G-L3 may be better

---

## Architecture Summary

```
Pi #1 (Control Plane + NVMe Registry)
├── k3s master
├── Docker registry
├── Job queue (SQLite/Redis)
└── NFS export to NAS

Pi #2-4 (Codex Worker Nodes)
├── Claude Agent SDK instances (2 per node)
├── MCP servers (Terraform, GitHub, Azure)
└── Shared NFS storage

NAS (Synology or similar)
├── Generated code artifacts
├── Session state / checkpoints
├── Benchmark results
└── Tailscale-accessible from desktop

Desktop (via Tailscale)
└── GPU testing (docker-compose.desktop.yml)
```

## Capacity

- **Concurrent generations**: 4-6 pairs (2 Agent SDK instances × 3 worker nodes)
- **Bottleneck**: Claude API rate limits, not hardware
- **Memory per instance**: ~1.4-1.9 GB (Claude + Codex pair)
- **Swap threshold**: Do not exceed 85% RAM — swap thrashing is catastrophic, not gradual

## Key Constraints

- No local GPU → cannot serve local LLMs, cannot do local image generation
- ARM64 → all Docker images must have ARM64 variants; some binary dependencies require rebuild
- microSD → use NVMe for all production nodes (microSD fails under write load in 6-12 months)
- API latency → Claude API adds 10-20s per request; pipeline speed is Claude-bound, not Pi-bound

## Orchestration Stack

- **Runtime**: Claude Agent SDK (Python, Node.js 18+)
- **Cluster**: k3s (lightweight Kubernetes for ARM)
- **Storage**: NVMe per node + NAS via NFS/Tailscale
- **Networking**: Tailscale (encrypted, NAT-traversal, nomadic-friendly)
- **Coordinator**: Custom Python job queue service (see harness architecture)

## 5-Year Cost Analysis: G-L1 vs Cloud Equivalent

A Pi cluster has a useful life of 5+ years. Cloud infrastructure has no end date. This comparison uses a 5-year horizon.

### G-L1 Hardware (one-time)

| Item | Qty | Unit | Total |
|------|-----|------|-------|
| Pi 5 8GB | 4 | $80 | $320 |
| NVMe SSD 256GB | 4 | $35 | $140 |
| Pi case with NVMe support | 4 | $30 | $120 |
| Official Pi 5 PSU (27W USB-C) | 4 | $12 | $48 |
| Network switch (8-port, unmanaged) | 1 | $40 | $40 |
| Synology NAS DS223 | 1 | $300 | $300 |
| 2× 4TB HDD (NAS) | 2 | $80 | $160 |
| Cables, SD cards, misc | — | — | $40 |
| **Hardware total** | | | **$1,168** |

### G-L1 Ongoing (5 years)

| Item | Monthly | 5 years |
|------|---------|---------|
| Electricity (~60W avg cluster + NAS) | $6.50 | $390 |
| Tailscale (personal plan — free; team: $6/user) | $0–$6 | $0–$360 |
| Storage replacement risk (NVMe, NAS HDDs) | ~$8 (amortised) | $480 |
| **Ongoing total** | | **$870–$1,230** |

**G-L1 5-year total: ~$2,000–$2,400**

---

### Cloud Equivalent (5 years, AWS us-east-1)

Equivalent capacity: 4 worker VMs + 1 coordinator, always-on, NAS → S3 + EFS.

| Item | Rate | Hours | Total |
|------|------|-------|-------|
| 4× t3.large worker (2 vCPU, 8GB) — 12h/day active | $0.083/hr × 4 | 21,900h | $7,269 |
| 1× t3.small coordinator — always on | $0.023/hr | 43,800h | $1,007 |
| S3 storage: 2TB active artifacts | $0.023/GB/month | 60 months | $2,832 |
| EFS (session state, shared FS) — 50GB | $0.30/GB/month | 60 months | $900 |
| Data transfer out (~50GB/month) | $0.09/GB | 3,000 GB | $270 |
| CloudWatch logs + metrics | $20/month | 60 months | $1,200 |
| NAT gateway / VPN equivalent (Tailscale is free) | $30/month | 60 months | $1,800 |
| **Cloud 5-year total** | | | **~$15,300** |

---

### Summary

| | G-L1 (Pi Cluster) | Cloud Equivalent |
|--|-------------------|-----------------|
| Year 1 | ~$1,600 (hardware + first year) | ~$3,060 |
| Year 2–5 | ~$200/year | ~$3,060/year |
| **5-year total** | **~$2,000–$2,400** | **~$15,300** |
| **5-year saving** | **~$12,900** | — |

**Breakeven**: Cloud exceeds G-L1 hardware cost by ~month 5. The Pi cluster pays for itself before the first customer is delivered.

**What cloud buys you that Pi doesn't**: elastic scale, SLA, no hardware management, NVIDIA GPU (spot). If burst capacity or GPU is genuinely needed, G-C1 as a supplemental burst layer (not a replacement) costs ~$50–200/month for occasional use — still a net saving vs all-cloud.

---

## Expansion Path

G-L1 → G-L2: Add one Mac Mini as a GPU worker node. No cluster reconfiguration required; Mac Mini joins via Tailscale, runs Docker Compose rather than k3s.

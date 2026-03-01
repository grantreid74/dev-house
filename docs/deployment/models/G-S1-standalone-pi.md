# G-S1: Standalone Pi Generation Model

**Model ID**: G-S1
**Class**: Local Generation Infrastructure
**GPU**: None
**Architecture**: Single-node Docker Compose (no cluster OS)

---

## When to Choose G-S1

- Single developer, small scope, or first customer pilot
- API-bound codegen only — no GPU-dependent test suites
- Lowest possible hardware cost and complexity
- Ultra-portable: one Pi 5 fits in a jacket pocket
- Evaluating Dev-House before committing to a full cluster

## When NOT to Choose G-S1

- Customer PRD requires local LLM serving → G-L2 or G-L3
- GPU-dependent validation is part of the pipeline → G-L2 or G-L3
- More than 2 concurrent generation pairs needed → G-L1 or above
- Long overnight batch runs with high artifact volume → NAS + cluster recommended

---

## Architecture

```
Pi 5 16GB (single node)
├── Docker Compose (all services)
├── Claude Agent SDK (2 pairs max)
├── MCP servers
├── Job queue (SQLite, local)
├── Artifacts (NVMe local storage)
└── Tailscale (remote access, desktop GPU testing)

Desktop (via Tailscale)
└── GPU testing (docker-compose.desktop.yml)
    — same pattern as G-L1, desktop handles what Pi cannot
```

No k3s, no NAS, no cluster networking. Every service runs in Docker Compose on one machine.

---

## Capacity

- **Concurrent generations**: 2 pairs (2 Agent SDK instances, 16GB RAM comfortable)
- **Bottleneck**: Claude API rate limits, same as all G-series models
- **Memory per instance**: ~1.4-1.9 GB — 2 pairs use ~3-4 GB, leaving ~11-12 GB for Docker services
- **Storage**: NVMe 512GB-1TB local — sufficient for moderate artifact volume; no NAS required

---

## Testing Ceiling

The key difference vs G-L3: the Pi 5 has no meaningful GPU for ML workloads.

| Test type | G-S1 (Pi 5) | G-L3 (Mac Mini) |
|-----------|-------------|-----------------|
| ARM64 service smoke tests | Yes | Yes |
| Docker Compose integration tests | Yes | Yes |
| Database, queue, HTTP service tests | Yes | Yes |
| Local LLM serving / inference | No | Yes (Metal/MLX) |
| Image generation | No | Yes (MPS) |
| Embedding generation (fast) | No | Yes |
| CUDA workloads | No | No (Metal only) |

**GPU testing solution**: same as G-L1 — route GPU test suites to desktop via Tailscale using `docker-compose.desktop.yml`. The Pi generates code; the desktop validates GPU-dependent behaviour.

---

## Key Constraints

- Single point of failure — no redundancy, no job rescheduling
- No NAS: artifacts on local NVMe. If Pi fails, artifacts are at risk. Mitigate with periodic rsync to desktop or cloud.
- 2-pair ceiling: 3rd concurrent pair risks swap thrashing on 16GB
- ARM64: all Docker images must have ARM64 variants

---

## Hardware Bill

| Item | Cost |
|------|------|
| Pi 5 16GB | ~$120 |
| NVMe SSD 512GB | ~$50 |
| Pi case with NVMe support | ~$30 |
| Official Pi 5 PSU (27W USB-C) | ~$12 |
| **Total** | **~$212** |

Electricity: ~10W average → ~$5/month → ~$300 over 5 years.
**5-year total: ~$500** (hardware + electricity).

---

## 5-Year Cost vs Cloud

| | G-S1 | Cloud equivalent (t3.medium, 12h/day) |
|--|------|---------------------------------------|
| Year 1 | ~$312 | ~$510 |
| Years 2–5 | ~$50/year | ~$510/year |
| **5-year total** | **~$512** | **~$2,550** |
| **5-year saving** | **~$2,040** | — |

---

## Expansion Path

G-S1 → G-L1: Add 3 more Pis, stand up k3s, connect to NAS. The G-S1 node becomes a worker in the cluster. No work is lost — Docker images and artifacts carry over.

G-S1 → G-L2: Add a Mac Mini for GPU capability. G-S1 Pi becomes coordinator; Mac Mini handles GPU tasks.

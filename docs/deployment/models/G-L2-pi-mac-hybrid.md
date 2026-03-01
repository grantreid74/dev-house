# G-L2: Pi + Mac Mini Hybrid Generation Model

**Model ID**: G-L2
**Class**: Local Generation Infrastructure
**GPU**: Partial (Mac Mini nodes only — Apple Silicon)
**Architecture**: Mixed ARM64 cluster (Pi = k3s workers) + Docker-per-mini (Mac Mini)

---

## When to Choose G-L2

- Mostly API-bound codegen, but some workloads need local GPU
- Want GPU capability without committing to full Mac Mini fleet cost
- Expanding from G-L1 without replacing existing Pi hardware
- GPU tasks are occasional, not the primary workload

## When NOT to Choose G-L2

- Every generation task needs GPU → G-L3 (Mac Mini fleet more efficient)
- No GPU needed at all → G-L1 (simpler, cheaper)
- Nomadic operation with GPU: Mac Minis are manageable but heavier than Pis

---

## Architecture Summary

```
Pi #1 (Control Plane)
└── k3s master, job queue, coordinator

Pi #2-3 (API-bound Worker Nodes)
└── Claude Agent SDK, MCP servers, no GPU

Mac Mini #1 (GPU Worker Node)
├── Docker Compose (not k3s — different orchestration model)
├── Local LLM serving (Ollama / MLX)
├── GPU-accelerated validation or generation tasks
└── Joins cluster via Tailscale

Mac Mini #2 (optional — Dev + GPU)
└── GPU tasks + desktop testing (docker-compose.desktop.yml)

NAS (shared)
└── Artifacts, session state, code output
```

## Orchestration Difference: Mixed-Mode

The Pi nodes run k3s (cluster-native). Mac Minis run Docker Compose (standalone). The coordinator service must handle both:

- **Pi workers**: assigned via k3s job scheduling
- **Mac Mini workers**: assigned via direct SSH/API call to Docker Compose service on that node

This adds coordinator complexity vs G-L1. The job queue must track node capability (`gpu: true/false`) and route accordingly.

## Capacity

- **API-bound tasks**: 4-6 concurrent (Pi nodes, same as G-L1)
- **GPU tasks**: 1-2 concurrent per Mac Mini (Apple Silicon unified memory)
- **Local LLM**: Mac Mini M3 Pro (18-36GB RAM) can serve 7B-13B models concurrently with generation

## Key Constraints

- Two orchestration paradigms (k3s + Docker Compose) — coordinator must handle both
- Mac Mini GPU is Apple Silicon (Metal/MLX), not CUDA — model compatibility matters
- Mac Minis are fixed-location friendly but less nomadic than Pi stack
- NVMe boot on all nodes — microSD failure risk on Pis still applies

## GPU Use Cases Enabled (vs G-L1)

| Use case | G-L1 | G-L2 |
|----------|------|------|
| Local LLM (security guard, validation) | No | Yes |
| Image generation (customer UI mockups) | No | Yes |
| Embedding generation (semantic search in PRDs) | No | Yes |
| CUDA-specific ML inference | No | No (Metal only) |

## Expansion Path

G-L2 → G-L3: Replace or supplement Pi nodes with additional Mac Minis. Coordinator shifts from mixed-mode to Docker-Compose-native.

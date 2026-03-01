# G-L3: Mac Mini Fleet Generation Model

**Model ID**: G-L3
**Class**: Local Generation Infrastructure
**GPU**: Yes (Apple Silicon on all nodes)
**Architecture**: Docker-per-mini (no cluster OS — each Mini is independent)

---

## When to Choose G-L3

- GPU capability needed on most/all generation nodes
- Fixed location operation (office, lab)
- Higher compute per node needed (M3 Pro/Max vs Pi 5 ARM)
- Customer workloads are compute-heavy, not just API-bound
- Budget allows: Mac Minis are ~5-10× more expensive than Pis per node

## When NOT to Choose G-L3

- Budget-constrained — G-L1 does API-bound codegen just as fast, far cheaper
- Nomadic operation — Minis are manageable but not as portable as Pi stack
- Cluster-native orchestration preferred — Docker-per-mini is simpler but less Kubernetes-native

---

## Architecture: Docker-per-Mini (not k3s cluster)

Unlike G-L1/G-L2, G-L3 does **not** run a shared cluster OS. Each Mac Mini is an independent node running Docker Compose. The coordinator treats each Mini as a standalone worker.

```
Mac Mini #1 (Coordinator)
├── Job queue service (Python + SQLite/Redis)
├── Harness orchestrator
├── Dashboard / monitoring
└── Tailscale gateway

Mac Mini #2-3 (Dev Worker Nodes)
├── Claude Agent SDK (Python)
├── MCP servers
├── Docker Compose (agent containers)
└── GPU available for local LLM / validation

Mac Mini #4-5 (Dev Worker Nodes)
└── Same as #2-3

NAS (Synology)
└── Shared artifacts, session state, code output
```

## Why Docker-per-Mini vs k3s?

The user's observation is correct: running k3s on Mac Minis (1 coordinator + 1-node cluster + 3 workers) would be wasteful — k3s overhead for a 5-node cluster where each node is already powerful is unnecessary complexity.

Docker Compose per Mini is simpler:
- Each Mini runs its own stack (agent + MCP servers + local LLM)
- Coordinator dispatches jobs via API/queue
- No shared cluster state to manage
- Easier to add/remove nodes

The trade-off: no k3s scheduling primitives (health checks, auto-reschedule). The coordinator must handle failure detection and job reassignment explicitly.

## Capacity

- **Concurrent generations per Mini**: 2-4 (Apple Silicon M3 Pro with 18-36GB RAM)
- **GPU tasks**: available on all nodes (Apple Silicon Metal/MLX)
- **Local LLM**: 7B-70B models depending on RAM (M3 Max 96GB can serve 70B)
- **Total cluster**: 8-16 concurrent generations (4 worker Minis × 2-4 per Mini)

## Apple Silicon GPU — Capabilities and Limits

| Capability | Status |
|------------|--------|
| Local LLM serving (Ollama/MLX) | Yes — excellent |
| Embedding generation | Yes |
| Image generation (Stable Diffusion) | Yes (via MPS/Metal) |
| CUDA inference | No — Metal only |
| TensorFlow/JAX (GPU) | Limited (Metal plugins, experimental) |
| PyTorch (GPU) | Yes via MPS backend |

## Key Constraints

- Not CUDA: customers requiring NVIDIA GPU workloads must use G-C1 (cloud burst with GPU VMs)
- Docker-per-mini: coordinator handles failure, no k3s auto-scheduling
- Cost per node: ~$800-2,000 (Mac Mini M3 Pro/Max). Not a direct per-node comparison to G-L1: to match 1 Mac Mini M3 Pro (12-core CPU, 18-36GB RAM), you would need 3-4 Pi 5 8GB nodes (~$600-1,000 all-in). At equivalent CPU/RAM, the Mac Mini is cost-competitive — and adds GPU capability the Pi cannot match at all.
- Fixed power: each Mac Mini ~30-60W vs Pi 5 ~5-10W

## When This Model Suits a Customer Type

G-L3 makes sense when a customer PRD includes:
- Local LLM serving in their product (embedded AI, offline AI)
- Image generation pipeline
- On-device ML inference requirements
- Requirements that need testing against Apple Silicon targets

## Expansion Path

G-L3 scales by adding Mac Minis. No cluster reconfiguration — coordinator detects new nodes via Tailscale and begins dispatching.

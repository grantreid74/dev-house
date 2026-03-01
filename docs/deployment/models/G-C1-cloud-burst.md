# G-C1: Cloud Burst Generation Model

**Model ID**: G-C1
**Class**: Cloud Generation Infrastructure
**GPU**: On-demand (A100/H100 spot instances)
**Architecture**: Managed container service (Azure Container Apps, AWS Fargate, GCP Cloud Run)

---

## When to Choose G-C1

- No local hardware investment desired
- Burst capacity needed beyond local cluster limits
- NVIDIA GPU required (CUDA-dependent workloads — not possible on Pi or Mac Mini Metal)
- Customer generates infrequently (pay-per-use beats fixed hardware cost)
- Enterprise customer requires cloud-native operation with no on-premise hardware

## When NOT to Choose G-C1

- High-volume sustained generation (cloud cost exceeds hardware amortisation quickly — see cost comparison)
- Data sovereignty: PRDs cannot leave your premises → local models required
- API round-trip latency matters: cloud adds network hops vs local cluster

---

## Architecture

```
Harness Coordinator (local or cloud-hosted)
├── Job queue (Redis / Azure Service Bus / SQS)
├── Dispatches to cloud workers
└── Collects results → NAS or cloud storage

Cloud Worker Nodes (spin up on demand)
├── Claude Agent SDK (containerised)
├── MCP servers
├── GPU available if needed (spot instances)
└── Terminate after job completes

Cloud Storage
└── Artifacts, session state, generated code
    (S3 / Azure Blob / GCS)
```

## Cost Model

Cloud generation is **pay-per-use but expensive at volume**:

| Scenario | Monthly cost (est.) |
|----------|-------------------|
| 1 customer, 1 generation/month | ~$20-50 |
| 5 customers, weekly generations | ~$200-500 |
| 10 customers, daily overnight runs | ~$800-2,000 |
| Equivalent G-L1 (8-Pi cluster) hardware | ~$1,000 one-time + $10/month electricity |

**Break-even**: G-L1 hardware pays for itself vs G-C1 in 3-6 months at moderate volume. G-C1 makes financial sense only for low-frequency or bursty workloads.

## GPU in G-C1

Unlike local models, G-C1 can use NVIDIA GPUs (A100, H100 via spot):
- Enables CUDA-only workloads
- Expensive: ~$2-4/hour for A100 spot
- Use only when genuinely needed — schedule GPU tasks, don't leave instances running

## Provider Options

| Provider | Container service | GPU support | Spot/preemptible |
|----------|------------------|-------------|-----------------|
| AWS | Fargate, ECS | EC2 p4/p3 (not Fargate) | Yes (Spot) |
| Azure | Container Apps | NC-series VMs | Yes (Spot) |
| GCP | Cloud Run, GKE | A100/H100 | Yes (preemptible) |

Cloud Run and Container Apps do not support GPU directly — GPU generation requires VM-based compute (EC2, Azure VM, GCE). Container services handle CPU agent workloads; GPU tasks route to dedicated VM pools.

## Hybrid Use: G-C1 as Burst Layer

G-C1 works well as a **burst extension** for G-L1 or G-L3:
- Normal load: Pi cluster or Mac Mini fleet handles generation
- Overflow: Coordinator detects queue depth > threshold → spins up cloud workers
- GPU tasks: always route to G-C1 when local model has no GPU (G-L1)

This hybrid avoids paying for cloud capacity at baseline while handling spikes gracefully.

## Constraints

- **Stateless workers**: Each cloud worker is ephemeral. Session state must be in external storage (not local disk).
- **Cold start latency**: Container spin-up adds 30-60 seconds before generation begins.
- **Cost control**: Set hard budget limits. Runaway agent loops in cloud = unexpected bills.
- **Data egress**: PRDs sent to cloud workers. Check customer data handling requirements before using G-C1.

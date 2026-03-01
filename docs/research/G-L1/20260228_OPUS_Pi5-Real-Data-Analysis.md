# Pi 5 Real Operational Data Analysis: Dev Cluster Capacity & Viability

**Author**: Claude Opus 4.6
**Date**: 2026-02-28
**Data Source**: Live `top` and `docker stats` output from a Pi 5 (4GB) running 16 services in normal operation (not under stress)
**Related Research**: [Hardware cost analysis](20260228_1430_local-hardware-cost-analysis.md), [GPU & concurrency analysis](20260228_OPUS_portable-gpu-concurrency-analysis.md)

> **Context (added 2026-03-01)**: The Dev-House Pi cluster is a **development system only**. Production deployments are handled by a separate Terraform pipeline that provisions cloud infrastructure per customer. The cluster's job is to run **8–12 overnight batch jobs** working through large PRDs — unattended, no SLA, results ready by morning. This significantly changes the risk calculus versus the "production capacity" framing used in the original analysis.

---

## Executive Summary

The Pi 5 (4GB) was observed running 16 services in normal operation — not under stress — at load average 0.01-0.03 (effectively idle). The 25% CPU reading was a point-in-time single-core burst, not sustained load. The system has substantial headroom.

**Key findings**:
- A single Pi 5 (8GB) can run ONE Claude+Codex pair comfortably, TWO under light load, swap-thrash at three
- The 8-Pi cluster (dev tier Pis 5-8 + overflow to Pi #2) can sustain **8–12 concurrent overnight jobs** — the design target
- GPU on Pi 5 is VideoCore VII: hardware video encode/decode only, not LLM inference. GPU testing routes to desktop or cloud
- The "4-6 concurrent pairs" figure in the original analysis was correct for the dev tier alone; with worker overflow the target of 8-12 is achievable

---

## Part A: Deep Analysis of Real Operational Data

### What Does 25% User CPU Actually Mean?

```
%Cpu(s): 25.0 us, 0.0 sy, 0.0 ni, 75.0 id, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st
```

The Pi 5 has a quad-core Cortex-A76 at 2.4 GHz. The `top` CPU line reports an **aggregate across all cores**. 25% user CPU means one of two things:

1. **One core is 100% busy, three cores are idle** -- This is the most likely interpretation. A single-threaded Python process (your 544 MB process at 13.1% RAM) hit a CPU burst -- garbage collection, a periodic task, or a request handler. The 0.0% system, 0.0% I/O wait, and 0.0% software interrupt confirm this was pure userspace computation, not kernel overhead or disk thrashing.

2. **All four cores are ~25% each** -- Less likely given the load averages, but possible if multiple containers briefly synchronized their periodic tasks (Prometheus scrape + InfluxDB compaction + Node-RED flow execution).

**The load average tells the real story**:
```
Load average: 0.01, 0.03, 0.02
```

These numbers are extraordinary. A load average of 0.01 on a 4-core system means the CPU run queue has been essentially empty for the past minute. The system spent 99%+ of its time idle over the 1/5/15-minute windows. The 25% CPU snapshot caught a transient burst that lasted less than the `top` refresh interval (typically 3 seconds).

**Verdict**: Your Pi 5 is genuinely idle. The 25% is a measurement artifact of point-in-time sampling during a brief single-core burst.

### Why Is Load Average So Low Despite 14 Containers?

This is the critical insight: **containers are not processes**. A Docker container is a namespace + cgroup wrapper. If the process inside is sleeping (waiting for a network connection, a timer, or I/O), the container consumes zero CPU and doesn't contribute to load average.

Your 14 containers break down into workload categories:

| Category | Containers | CPU Pattern | Why Low Load |
|----------|-----------|-------------|--------------|
| **Event-driven** | Node-RED, Mosquitto, Home Assistant | 0% until triggered | MQTT/HTTP listeners sleep until message arrives |
| **Periodic poll** | Teslamate, Prometheus, Node Exporter | Brief burst every N seconds | 0.11% max = ~50ms of work per scrape interval |
| **Passive proxy** | Nginx, Cloudflared, AdGuard Home | 0% between requests | TCP accept() blocks, zero CPU when idle |
| **Database** | Postgres, InfluxDB | 0% between queries | WAL flush is the only periodic work |
| **Dashboard** | Grafana | 0% until browser opens | Server-side rendering only on request |

**The pattern**: All 14 services are **reactive** -- they do work only when asked. Nobody is asking at 21:39 on a Tuesday. A home automation system at night is a system at rest.

This is fundamentally different from Dev-House workloads, which are **proactive** -- the orchestrator initiates API calls, parses responses, writes files, and runs validation. The distinction matters for capacity planning.

### Actual Headroom on This Pi 5

#### CPU Headroom

| Metric | Value | Headroom |
|--------|-------|----------|
| Sustained load average | 0.01-0.03 | **3.97 cores available** (of 4.0) |
| Peak observed CPU | 25% (transient) | ~75% sustained available |
| Per-core capacity | 2.4 GHz Cortex-A76 | ~11,000 single-thread Geekbench 6 score (per core) |
| Realistic sustained | 70-80% of 4 cores | **~3.0-3.2 effective cores** before thermal throttling |

The Pi 5 thermal throttles at ~85C. Under sustained all-core load, the stock cooler allows approximately 80% sustained performance. With an active cooler (fan HAT), you maintain ~95%.

#### Memory Headroom

```
MiB Mem: 4049.9 total, 149.3 free, 2066.4 used, 2129.8 buff/cache
MiB Swap: 2048.0 total, 1130.8 free, 917.2 used
```

This requires careful interpretation:

| Component | Value | Analysis |
|-----------|-------|----------|
| **Total RAM** | 4050 MB | Fixed, non-upgradeable on Pi 5 |
| **Used** | 2066 MB | Actual resident memory (processes + kernel) |
| **Buff/cache** | 2130 MB | Reclaimable filesystem cache |
| **Free** | 149 MB | Unallocated (low, but buff/cache is reclaimable) |
| **Available** | 1984 MB | Free + reclaimable cache = actual headroom |
| **Swap used** | 917 MB | ~900 MB has been paged out |

**The 917 MB of swap is concerning.** It means the system has, at some point, run low enough on RAM that the kernel evicted ~900 MB of process memory to disk. On a microSD or even an NVMe SSD, swap access is 100-1000x slower than RAM. However, the 0.0% I/O wait suggests this swap is "cold" -- pages that were swapped out long ago and haven't been needed since (likely container startup memory that got paged out once the container settled into steady state).

**Effective available RAM**: ~1.5-1.8 GB (the 1984 MB "available" minus a safety margin for kernel overhead and file cache performance).

#### I/O Headroom

The 0.0% I/O wait is excellent. Your storage subsystem (likely microSD or USB SSD) is not under pressure. For Dev-House workloads that write generated code files, this would change -- but Pi 5's USB 3.0 (5 Gbps) or PCIe NVMe provides adequate sequential write throughput.

### Can This Pi Handle 2-3 Additional Concurrent Processes?

**Yes, but it depends on the process type.**

| Additional Process | RAM Needed | CPU Needed | Verdict |
|-------------------|-----------|-----------|---------|
| Another Docker service (lightweight) | 100-300 MB | <1% | Yes, easily |
| Python web server (Flask/FastAPI) | 200-500 MB | 1-5% | Yes |
| Claude API client (HTTP + JSON) | 200-400 MB | <5% (I/O bound) | Yes |
| Codex output parser + validator | 500 MB-1 GB | 10-30% bursts | Marginal on 4GB, fine on 8GB |
| Local LLM inference (even tiny) | 2+ GB | 100% sustained | No -- swap death |

On your current **4 GB Pi 5**: You could add 1 Claude API client process (I/O bound, ~300 MB) without issue. Adding a Codex parser would push into swap territory.

On an **8 GB Pi 5**: You could add 2-3 moderate processes (Claude client + Codex parser + lightweight service) with ~2 GB headroom remaining.

### What Would Performance Degradation Look Like?

Degradation on Pi 5 follows a cliff pattern, not a gradual slope:

```
Load 0.0 - 2.0:  Normal operation. Responsive. No issues.
Load 2.0 - 3.5:  Mild latency increase. Context switches rise. Still functional.
Load 3.5 - 4.0:  CPU saturation. Processes queue. Latency doubles.
Load 4.0 - 6.0:  Overcommitted. Thermal throttling begins.
                  Swap activity increases. Response times 5-10x normal.
Load 6.0+:        Swap thrashing. System becomes unresponsive.
                  SSH sessions lag. Docker healthchecks fail.
                  OOM killer may terminate containers.
```

The critical threshold is **swap thrashing**. When active working set exceeds physical RAM, the system enters a death spiral: every page fault requires a disk read, which blocks the CPU, which increases load, which triggers more swapping. On microSD (40-90 MB/s random read), this is catastrophic. On NVMe (~500 MB/s random read), it's painful but survivable.

---

## Part B: Claude + Codex on Same Pi 5

### Resource Requirements Analysis

**Claude API Orchestration (Harness)**:
| Resource | Requirement | Notes |
|----------|-------------|-------|
| RAM | 200-400 MB | HTTP client, JSON parsing, context window management |
| CPU | <5% sustained | I/O-bound: waiting 10-20 sec for API responses |
| Disk I/O | Minimal | Logging, state persistence |
| Network | 1-5 Mbps | API request/response payloads (mostly text) |
| Pattern | Async I/O, event-driven | Perfect for ARM -- no SIMD or heavy compute needed |

**Codex Code Generation Processing**:
| Resource | Requirement | Notes |
|----------|-------------|-------|
| RAM | 500 MB - 1.5 GB | Parsing generated code, AST validation, file tree management |
| CPU | 10-40% bursts | Code parsing, linting, test scaffolding |
| Disk I/O | Moderate | Writing generated files, reading templates |
| Network | 1-2 Mbps | Codex API calls (if remote) |
| Pattern | Burst compute + I/O | CPU spikes during validation, idle during API wait |

**Combined per-pair**: ~700 MB - 1.9 GB RAM, <50% CPU sustained (with bursts to 80%).

### Same Pi vs Separate Pis

#### Scenario 1: Both on ONE Pi 5 (8GB)

```
Total RAM:           8192 MB
OS + kernel:        -500 MB
Existing services:  -1000 MB (assuming lighter load than your 14-container Pi)
Claude Harness:     -400 MB
Codex processing:   -1000 MB
File cache needed:  -500 MB
────────────────────────────
Remaining:           ~4792 MB (comfortable for 1 pair)
```

**One pair works well.** The CPU pattern is complementary: Claude Harness is idle while waiting for API response (10-20 seconds), and Codex processing bursts happen after the response arrives. They naturally time-share the CPU.

**Two pairs gets tight:**
```
Second Claude+Codex: -1400 MB additional
Remaining:           ~3392 MB (functional but no margin)
```

Two pairs can work if their API calls are staggered (which they naturally tend to be, since requests take 10-20 seconds and don't synchronize). RAM is the constraint, not CPU.

**Three pairs: swap thrashing begins.**
```
Third Claude+Codex:  -1400 MB additional
Remaining:           ~1992 MB (insufficient for file cache)
```

At this point, the kernel starts reclaiming file cache aggressively, and then evicting process pages. Response times spike 5-10x. The system is technically functional but operationally degraded.

#### Scenario 2: Harness on Pi A, Codex on Pi B (each 8GB)

```
Pi A (Harness):
  4x Claude clients:  -1600 MB
  OS + overhead:       -500 MB
  Remaining:           ~6092 MB (plenty of cache space)

Pi B (Codex):
  4x Codex processors: -4000 MB (worst case)
  OS + overhead:        -500 MB
  Remaining:            ~3692 MB (tight but functional)
```

This separation works better because:
1. Harness is pure I/O-bound -- it barely touches the CPU, so 4 clients on one Pi is fine
2. Codex processing is burst-compute -- isolating it prevents CPU contention with Harness latency-sensitive operations
3. If Codex bursts cause thermal throttling, Harness response times aren't affected

#### Verdict

| Configuration | Max Pairs | Reliability | Recommendation |
|--------------|-----------|-------------|----------------|
| 1 Pi (8GB), both | 1-2 | High for 1, medium for 2 | Good for development/testing |
| 2 Pis, split H/C | 3-4 | High | Good for light production |
| 4 Pis, 2H + 2C | 4-6 | High | Reasonable production setup |
| 8 Pis, 4H + 4C | 6-8 | High | Overkill unless sustained load |

### Memory Pressure: When Does Swapping Destroy Performance?

The critical metric is **major page faults per second** (pages read from swap). Here's what happens on Pi 5:

| Swap Activity | Major Faults/sec | Effect | Recovery |
|--------------|-----------------|--------|----------|
| Cold swap (your current state) | <1/sec | None -- old pages sitting on disk | Normal operation |
| Warm swap | 10-50/sec | Occasional latency spikes (50-200ms) | Reduce memory pressure |
| Active swapping | 50-200/sec | Consistent latency (200ms-1s per operation) | Kill processes or add RAM |
| Swap thrashing | 200+/sec | System barely responsive, 5-30s latencies | Emergency -- OOM killer |

**On microSD** (your likely boot media): Random read latency is 1-5ms per 4K page. At 100 major faults/sec, you're spending 100-500ms/sec just waiting for swap I/O. At 500 faults/sec, the system is spending more time swapping than computing.

**On NVMe** (via Pi 5 PCIe HAT): Random read latency drops to 50-100us. This buys roughly 10-20x more tolerance for swap pressure. If you plan to run near memory limits, NVMe is not optional -- it's required.

**Practical threshold**: Keep total committed memory (RSS of all processes) below 85% of physical RAM. On 8GB, that's ~6.8 GB. Above that, the kernel starts making hard choices between file cache and process memory, and latency becomes unpredictable.

---

## Part C: GPU Expansion for Pi 5

### Pi 5 PCIe Architecture

The Pi 5 exposes a single **PCIe Gen 3 x1 lane** via the FPC connector on the board. This is a hard constraint that limits all GPU expansion options:

- **Theoretical bandwidth**: 985 MB/s (one direction), ~1.97 GB/s bidirectional
- **Practical bandwidth**: 700-800 MB/s sustained (protocol overhead)
- **Comparison**: A desktop RTX 4060 needs PCIe Gen 4 x8 (16 GB/s) to avoid bottlenecking

This means **any GPU attached to a Pi 5 operates at roughly 5% of its designed bandwidth**. For inference workloads that require constant weight loading (large models), this is devastating. For workloads that load once and compute many times (small models, encoding), it's tolerable.

### GPU Expansion Options

| Option | Interface | GPU | VRAM | Bandwidth | Price | Practical? |
|--------|-----------|-----|------|-----------|-------|------------|
| **Pineboards Hat AI** | PCIe x1 via M.2 | Google Coral TPU M.2 | N/A (8 TOPS) | 985 MB/s | $35-50 (HAT) + $25-35 (Coral) | Yes, for edge inference only |
| **Pineboards HatDrive! AI** | PCIe x1 via M.2 | Hailo-8 26 TOPS | N/A | 985 MB/s | $35-50 (HAT) + $70-100 (Hailo) | Yes, for structured inference |
| **Geekworm X1001 PCIe-to-M.2** | PCIe x1 adapter | Any M.2 A+E/M key GPU | Varies | 985 MB/s | $15-25 (adapter only) | Marginal -- M.2 GPU options are limited |
| **PCIe x1 to x16 riser** | Riser cable | Any desktop GPU | Full | 985 MB/s (bottleneck) | $10-20 (riser) + GPU cost | Technically works, bandwidth-starved |
| **USB 3.0 Coral** | USB 3.0 | Google Coral USB | N/A (4 TOPS) | 500 MB/s (USB 3.0) | $60-75 | Yes, simplest option |
| **Thunderbolt/USB4 eGPU** | Not available | N/A | N/A | N/A | N/A | **No -- Pi 5 has no USB4/TB** |

### Detailed Assessment by Use Case

#### CUDA Testing (Inference, Not Training)

**Not viable on Pi 5.** CUDA requires an NVIDIA GPU, and the only way to attach one is via the PCIe x1 riser hack. Even if you get the driver working (NVIDIA doesn't officially support ARM64 + PCIe x1), the 985 MB/s bandwidth means:

- Loading a 7B parameter model (14 GB in FP16): **~18 seconds** just for weight transfer
- Inference throughput: bottlenecked by weight loading for any model that doesn't fit in VRAM
- Batch inference: works if model fits entirely in VRAM and you're sending many inputs

**Practical alternative**: Use an x86 NUC/mini PC with a proper PCIe slot, or cloud GPU instances for CUDA testing.

#### GPU Encoding/Decoding

**Marginal.** The Pi 5's VideoCore VII GPU handles H.265/H.264 decode natively (4Kp60). For encode, the hardware encoder supports H.264 at 1080p30. An external GPU adds nothing meaningful here -- the Pi's built-in video pipeline is already optimized for its I/O bandwidth.

#### Local LLM Inference (Fallback)

| Accelerator | Model Size Limit | Tokens/sec (est.) | Latency | Verdict |
|------------|-----------------|-------------------|---------|---------|
| **CPU only (Pi 5)** | 1-3B (Q4) | 5-15 tok/s | 200-500ms/tok | Usable for tiny models |
| **Coral TPU (USB)** | Purpose-built only | N/A (not LLM-compatible) | N/A | Wrong architecture |
| **Hailo-8 (M.2)** | TFLite/ONNX only | 26 TOPS on supported models | 10-50ms | Good for vision, not LLMs |
| **PCIe riser + RTX 3060** | Up to 12GB models | 20-40 tok/s (bandwidth-limited) | 50-100ms/tok | Works but fragile setup |

**Honest assessment**: For local LLM fallback, the Pi 5 CPU running llama.cpp with a 1-3B quantized model (Phi-3 Mini, TinyLlama) gives 5-15 tokens/second. That's slow but functional for a "security guard" LLM that classifies prompts as safe/unsafe. It is not suitable for generative tasks.

### Portability with GPU

| Setup | Weight | Footprint | Power Draw | Portable? |
|-------|--------|-----------|-----------|-----------|
| Pi 5 + Coral USB | ~150g | Pocket-sized | 5-8W | Highly portable |
| Pi 5 + Hailo-8 HAT | ~200g | HAT stack | 8-12W | Portable |
| Pi 5 + PCIe riser + RTX 3060 | ~1.5 kg | Not portable | 170W+ peak | Not portable |
| Pi 5 + TB eGPU | N/A | N/A | N/A | Not possible |

---

## Part D: Practical Assessment -- 8-Pi Cluster for Dev-House

### Honest Capacity of the 8-Pi Cluster

Let me model this with real numbers, using your operational data as the baseline.

#### Per-Pi 5 (8GB) Capacity Budget

```
Total RAM:                  8192 MB
OS + kernel + systemd:      -400 MB
Docker daemon overhead:     -150 MB
Base infrastructure:        -300 MB  (Prometheus, logging, networking)
──────────────────────────────────────
Available for workloads:    7342 MB

Per Claude+Codex pair:      1400-1900 MB (average 1650 MB)
Safety margin (15%):        -1100 MB
──────────────────────────────────────
Usable for pairs:           6242 MB
Max pairs per Pi:           3 (at 1650 MB each = 4950 MB, within budget)
Comfortable pairs per Pi:   2 (at 1650 MB each = 3300 MB, ample margin)
```

#### 8-Pi Cluster Total

| Configuration | Pairs/Pi | Total Pairs | RAM Utilization | Risk Level |
|--------------|----------|-------------|-----------------|------------|
| Conservative (2/Pi) | 2 | 16 | 60% | Low -- ample headroom |
| Moderate (3/Pi) | 3 | 24 | 80% | Medium -- needs NVMe swap |
| Aggressive (4/Pi) | 4 | 32 | 95%+ | High -- swap thrashing likely |
| Realistic sustained | 2-3 | 16-24 | 65-80% | **Recommended range** |

**But wait -- is this the right question?**

The 8-Pi cluster can run 16-24 concurrent Claude+Codex pairs. But Dev-House doesn't need 16-24 concurrent pairs. From the [prior analysis](20260228_OPUS_portable-gpu-concurrency-analysis.md), the requirement is **4x concurrent processes**. Eight Pis is 4-6x overprovisioned for that requirement.

### Where Would the Bottleneck Be?

#### Under Current Light Load (Your Home Automation)

**Bottleneck: Nothing.** The system is 90%+ idle. The "bottleneck" is that there's no work to do.

#### Under Dev-House Production Load

| Resource | 4 Pairs | 8 Pairs | 16 Pairs | 24 Pairs |
|----------|---------|---------|----------|----------|
| **CPU** | 20-30% | 40-60% | 70-90% | Saturated |
| **RAM** | 40-50% | 60-70% | 80-90% | Swap required |
| **Network** | 5-20 Mbps | 10-40 Mbps | 20-80 Mbps | May hit ISP limits |
| **Disk I/O** | Negligible | Low | Moderate | High (code writes) |
| **API rate limits** | Well within | Within | Possible throttling | Likely throttled |

**The real bottleneck is Claude API rate limits and latency**, not Pi hardware. Each API call takes 10-20 seconds. Even with 24 concurrent pairs, you're making ~1.2-2.4 requests/second to the API. Anthropic's rate limits (varies by tier) are the ceiling, not your hardware.

**Secondary bottleneck: Network I/O.** Not bandwidth (API payloads are small) but **latency**. If all 8 Pis share one internet connection, and each makes concurrent API calls, DNS resolution, TLS handshake, and TCP connection setup can stack up. A single router handling 50+ concurrent HTTPS connections may introduce 50-200ms of additional latency per request.

**Tertiary bottleneck: Inter-Pi communication.** If Harness on Pi A needs to send generated code to Codex on Pi B, that traverses the network switch. Gigabit Ethernet gives ~100 MB/s practical throughput. For code files (typically <1 MB), this is not a bottleneck. For large context windows (100K+ tokens as JSON), transfers of 5-20 MB become noticeable (~50-200ms per transfer).

### Sustained Load vs Current Light Load

Your Pi has run 90 days at load average 0.01-0.03. Under Dev-House sustained load:

| Metric | Current (Home Auto) | Projected (4 pairs) | Projected (16 pairs) |
|--------|--------------------|--------------------|---------------------|
| Load average (1m) | 0.01-0.03 | 0.5-1.5 | 2.0-3.5 |
| RAM usage | 50% (of 4GB) | 55% (of 8GB) | 85% (of 8GB) |
| Swap usage | 900 MB (cold) | <200 MB (warm) | 500+ MB (active) |
| CPU temperature | ~45-50C | ~55-65C | ~70-80C |
| Thermal throttling | Never | Never | Possible without active cooling |
| Disk writes/day | ~50-200 MB | ~1-5 GB | ~5-20 GB |
| microSD lifespan impact | Years | 1-2 years | 6-12 months (use NVMe) |

**Critical warning on microSD**: Under sustained write load (code generation output, logs, Docker layers), microSD cards wear out. The Pi 5's NVMe HAT is not optional for production -- it's required. Budget ~$25 per Pi for an M.2 2230 NVMe drive.

### Is the Architecture Viable for Production Dev-House?

**Yes, with caveats.**

**What works well**:
- API orchestration is I/O-bound -- ARM is fine for this
- Low power draw (~5W idle, ~10W load per Pi) means the whole cluster runs on ~80W
- High availability through redundancy -- if one Pi fails, others continue
- Cost-effective: 8x Pi 5 (8GB) = ~$640 vs $1,500+ for equivalent x86 mini PC
- Your 90-day uptime proves Pi 5 reliability for always-on workloads

**What doesn't work well**:
- No local GPU for CUDA testing or LLM inference
- 8GB RAM ceiling limits per-node density
- microSD wears under write load (mitigated by NVMe)
- PCIe Gen 3 x1 makes GPU expansion impractical
- No ECC RAM -- bit flips are possible on 24/7 systems (rare but real)
- Power supply quality matters -- a bad USB-C adapter causes silent corruption

**What's overkill**:
- 8 Pis for 4x concurrency is overprovisioned
- Separate Pis for each Claude+Codex pair wastes the I/O-bound nature of Harness
- Cluster management overhead (Kubernetes, Docker Swarm) on 8 nodes is significant operational burden

---

## Recommendations

### Option 1: Lean Cluster (Recommended)

**3 Pi 5 (8GB) + 1 x86 mini PC**

| Node | Role | Specs | Cost |
|------|------|-------|------|
| Pi 5 #1 | Harness orchestrator (primary) | 8GB, NVMe, active cooler | ~$110 |
| Pi 5 #2 | Harness orchestrator (secondary) + services | 8GB, NVMe, active cooler | ~$110 |
| Pi 5 #3 | Monitoring, logging, CI runner | 8GB, NVMe, active cooler | ~$110 |
| x86 NUC | GPU compute, CUDA testing, heavy validation | 32GB RAM, RTX-capable | ~$500-800 |
| **Total** | | | **~$830-1130** |

- 4-6 concurrent Claude+Codex pairs
- GPU capability when needed
- Lower operational complexity than 8-node cluster
- Your existing 4GB Pi continues running home automation

### Option 2: Full Cluster (If Scale Justifies)

**8 Pi 5 (8GB) with role assignment**

| Nodes | Role | Pairs |
|-------|------|-------|
| Pi 1-2 | Harness orchestrators | Control plane |
| Pi 3-6 | Codex workers | 2 pairs each = 8 pairs |
| Pi 7 | Monitoring + CI | Infrastructure |
| Pi 8 | Hot spare + overflow | Failover |

- 8-12 concurrent pairs (realistic sustained)
- No GPU capability (accept this limitation)
- Requires cluster orchestration (k3s or Docker Swarm)
- Higher operational burden

### Option 3: Hybrid (Best of Both)

**4 Pi 5 (8GB) + cloud burst**

| Component | Role | Cost |
|-----------|------|------|
| 4x Pi 5 (8GB) | Always-on base capacity (4-8 pairs) | ~$440 one-time |
| Cloud spot instances | Burst capacity for >8 pairs, GPU testing | ~$50-100/month when used |

This matches your workload pattern: most of the time you need 2-4 pairs (Pi cluster handles it), occasionally you need 8+ pairs or GPU (cloud burst handles it). You pay for cloud only when you use it.

### Risk Factors and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| RAM exhaustion / swap death | Medium | High -- node becomes unresponsive | Set memory limits per container; OOM priority; alerting at 80% |
| microSD failure under write load | High (12-18 months) | Medium -- node offline until replaced | NVMe boot on all production nodes; microSD only for home automation Pi |
| Thermal throttling under sustained load | Medium | Low -- 10-20% performance reduction | Active cooler (fan HAT) on all nodes; mount in ventilated enclosure |
| Network switch failure (single point) | Low | High -- entire cluster offline | Dual switches or managed switch with redundancy |
| Power supply failure | Low | Medium -- single node offline | Use official Pi 5 PSU (27W USB-C); avoid cheap adapters |
| Claude API rate limiting | Medium | Medium -- queued requests | Implement backoff; distribute across API keys; queue management |
| Inter-node latency spikes | Low | Low -- 50-200ms added latency | Use wired Ethernet, not WiFi; quality managed switch |
| SD card corruption (power loss) | Low | High -- data loss | UPS for cluster; journaling filesystem; NVMe with power-loss protection |

---

## Appendix: Interpreting Your Docker Stats

For reference, here's what your current container metrics tell us:

```
All containers: <0.11% CPU
No container over 6% memory (~240 MB on 4GB)
```

Your highest-memory container is likely Postgres or InfluxDB (databases keep working sets in RAM). At 6% of 4 GB = ~240 MB, these are running with minimal data. A production Dev-House database (storing PRDs, generated code metadata, orchestration state) would consume 500 MB-2 GB depending on history depth.

The 0.11% CPU from Teslamate corresponds to its periodic API polling of Tesla's servers -- structurally identical to what Dev-House Harness does with Claude's API. This is a good real-world proof point: API orchestration on Pi 5 uses negligible CPU.

---

## Summary Table

| Question | Answer |
|----------|--------|
| Can one Pi 5 (8GB) run Claude+Codex? | Yes, 1-2 pairs comfortably |
| Can one Pi 5 run 4 concurrent pairs? | No -- swap thrashing at 3+ pairs |
| Is 8-Pi cluster viable for Dev-House? | Yes, but likely overprovisioned |
| Best cluster size for 4x concurrency? | 3-4 Pis (or 2 Pis + 1 x86 node) |
| GPU on Pi 5? | Impractical -- PCIe x1 bottleneck |
| What's the real bottleneck? | Claude API rate limits and latency, not Pi hardware |
| Biggest risk? | RAM exhaustion causing swap thrashing |
| Must-have upgrade? | NVMe boot drives on all production nodes |

# Container-as-a-Service vs Managed Kubernetes: Research Comparison

**Date**: 2026-03-02
**Model**: Claude Opus
**Type**: Frozen research (point-in-time snapshot; verify pricing before procurement)

---

## Executive Summary

- **Serverless containers (ACA, Fargate, Cloud Run) eliminate cluster operations** but constrain networking, storage, and multi-tenancy controls. They are the correct default for teams without dedicated platform engineering.
- **Managed Kubernetes (AKS, EKS, GKE) provides full control** at the cost of ongoing operational burden: node patching, capacity planning, RBAC policies, ingress configuration, and upgrade management. The control plane is managed; everything else is yours.
- **The cost crossover is driven by utilisation, not service count.** Serverless containers win below ~30-40% average CPU utilisation. Above ~50% sustained utilisation, managed K8s with reserved/committed instances is 30-60% cheaper.
- **For Dev-House's customer deployment pattern** (multi-tenant SaaS, schema-per-customer, Claude API-bound, no GPU, Terraform-provisioned), serverless containers are the better starting point. The workloads are API-bound (waiting on Claude, not computing), utilisation will be low, and the operational cost of K8s is unjustified until the customer scales beyond ~20-30 services or requires namespace-level tenant isolation with custom network policies.
- **Migration from serverless containers to K8s is straightforward** if you containerise properly (OCI images, externalised config, managed databases). What breaks is platform-specific glue: ingress annotations, scaling rules, secrets references, and IAM bindings.

---

## 1. What Each Approach Actually Is

### 1.1 Serverless / Container-as-a-Service

These platforms run your OCI container images without exposing the underlying compute infrastructure. You provide a container image and configuration; the platform handles provisioning, scheduling, scaling, networking, and TLS termination.

| Platform | What It Is (Precisely) | Built On |
|----------|----------------------|----------|
| **Azure Container Apps (ACA)** | Managed environment running containers on a hidden AKS cluster. Supports HTTP services, background workers, event-driven jobs, and Dapr sidecars. Consumption or Dedicated (workload profile) billing. | Kubernetes + Envoy + KEDA + Dapr |
| **AWS Fargate** | Serverless compute engine for ECS and EKS. You define task/pod resource requirements; AWS provisions right-sized micro-VMs (Firecracker). No node management. Used via ECS task definitions or EKS pod specs. | Firecracker micro-VMs |
| **AWS App Runner** | Higher-level abstraction over Fargate. Source-to-URL: point at a repo or image, get an HTTPS endpoint. Extremely limited configuration surface. | Fargate (under the hood) |
| **Google Cloud Run** | Knative-based container execution. Accepts any container that listens on a port. Scale-to-zero by default. Request-based or instance-based billing. Supports jobs (one-off/scheduled). | Knative on GKE |

**What you do NOT manage**: Nodes, OS patches, container runtime, scheduler, ingress controller, TLS certificates, load balancer configuration, capacity planning.

**What you DO manage**: Container images, application configuration, scaling rules (min/max replicas, concurrency targets), secrets, database connections, custom domains.

### 1.2 Managed Kubernetes

These platforms provision and maintain the Kubernetes control plane (API server, etcd, scheduler, controller-manager). You manage worker nodes (or use a managed node feature), workload configuration, networking policies, and the full Kubernetes API surface.

| Platform | What It Is (Precisely) | Control Plane Cost |
|----------|----------------------|-------------------|
| **Azure AKS** | Managed K8s control plane. Free tier (no SLA), Standard ($0.10/hr with SLA), Premium ($0.60/hr with LTS). You pay for node VMs separately. Full Kubernetes API access. | $0-$0.60/hr per cluster |
| **AWS EKS** | Managed K8s control plane at $0.10/hr (standard support) or $0.60/hr (extended support). Worker nodes via EC2, Fargate, or EKS Auto Mode. Full Kubernetes API. | $0.10-$0.60/hr per cluster |
| **Google GKE** | Managed K8s in Standard mode (you manage nodes) or Autopilot mode (Google manages nodes, you pay per pod). $0.10/hr per cluster (one free zonal cluster per account). | $0.10/hr per cluster |

**What you do NOT manage**: Control plane availability, etcd backups, API server scaling, K8s version of control plane components.

**What you DO manage**: Worker nodes (unless Autopilot/Auto Mode), node OS patching, node pool sizing, ingress controllers, service mesh, network policies, RBAC, storage classes, monitoring stack, certificate management, secrets management, Helm charts / manifests, upgrade scheduling.

---

## 2. Side-by-Side Capability Comparison

| Capability | Serverless Containers (ACA/Fargate/Cloud Run) | Managed Kubernetes (AKS/EKS/GKE) |
|-----------|----------------------------------------------|----------------------------------|
| **Scale-to-zero** | Native (ACA, Cloud Run, App Runner). Fargate: no native scale-to-zero (min 1 task unless stopped). | Not native. Requires KEDA or custom HPA. Node scale-to-zero possible with cluster autoscaler but slow (minutes). |
| **Autoscaling** | HTTP concurrency, CPU, memory, queue length (KEDA-based in ACA). Simple config. | Full HPA (CPU, memory, custom metrics), VPA, KEDA, cluster autoscaler. Complex but powerful. |
| **Max scale** | ACA: 300 replicas/app (consumption), 1000 (dedicated). Cloud Run: 1000 instances. Fargate: soft limits, raise via quota. | Effectively unlimited. AKS: 5000 nodes. EKS: 5000 nodes. GKE: 15000 nodes. |
| **Networking** | VNet/VPC integration available but limited. ACA: VNet injection, internal-only apps. Cloud Run: VPC connectors. No arbitrary port exposure (HTTP/gRPC only in most). | Full CNI control. Network policies (Calico, Cilium). Arbitrary ports. Service mesh (Istio, Linkerd). Custom ingress. |
| **Ingress / Load Balancing** | Built-in. ACA: Envoy-based, automatic TLS. Cloud Run: Google Front End, automatic TLS. Fargate: ALB/NLB integration. | You deploy and configure: NGINX Ingress, Traefik, Istio Gateway, AWS ALB Controller, GKE Gateway API. Full control, full responsibility. |
| **Service Mesh** | ACA: Dapr built-in (service invocation, pub/sub, state). Cloud Run: none built-in. Fargate: App Mesh (deprecated) or service connect. | Full choice: Istio, Linkerd, Cilium service mesh, Consul Connect. mTLS, traffic splitting, observability. |
| **Secrets** | ACA: secrets from Key Vault or environment. Cloud Run: Secret Manager integration. Fargate: SSM Parameter Store / Secrets Manager. | Kubernetes Secrets + CSI driver for vault integration. External Secrets Operator. Full flexibility. |
| **Persistent Storage** | Limited. ACA: Azure Files mount (not high-performance). Cloud Run: no persistent volumes (use GCS/database). Fargate: EFS mount (NFS). | Full PV/PVC support. Any CSI driver. Azure Disk, EBS, GCE PD, NFS, Ceph. StatefulSets with stable identity. |
| **Stateful Workloads** | Not designed for stateful workloads. Ephemeral storage only. Use external databases/caches. | StatefulSets, persistent volumes, stable network identity. Operators for databases (CockroachDB, PostgreSQL, Redis). |
| **GPU** | ACA: serverless GPU (A100, T4) with scale-to-zero, plus dedicated GPU profiles. Cloud Run: GPU support (L4) in preview. Fargate: no GPU. | Full GPU node pool support. NVIDIA device plugin. Any GPU instance type. Scheduling, time-slicing, MIG. |
| **Jobs / Batch** | ACA: Jobs (event-driven, scheduled, manual). Cloud Run: Jobs. Fargate: ECS scheduled tasks. | Kubernetes Jobs, CronJobs. Argo Workflows, Tekton. Volcano for batch scheduling. Full control. |
| **Multi-tenancy** | Limited. ACA: apps share environment (no hard isolation). Cloud Run: projects for isolation. Fargate: task-level IAM. No namespace-level network policies. | Full namespace isolation. Network policies. ResourceQuotas. LimitRanges. OPA/Gatekeeper policies. Hierarchical namespaces. |
| **Custom Controllers / Operators** | Not possible. You use what the platform provides. | Full CRD/operator support. Extend the API. Run any controller. |
| **Windows Containers** | ACA: No. Cloud Run: No. Fargate: Yes (ECS). | AKS: Yes. EKS: Yes. GKE: Yes (Standard mode). |
| **Observability** | Built-in metrics and logs routed to platform monitoring (Azure Monitor, CloudWatch, Cloud Logging). Limited custom metrics. | Full Prometheus/Grafana stack, OpenTelemetry, Datadog, any agent. Custom metrics, dashboards, alerting. |

---

## 3. Cost Model Differences

### 3.1 Pricing Structure

#### Serverless Containers

| Platform | vCPU Cost | Memory Cost | Request Cost | Free Tier |
|----------|-----------|-------------|-------------|-----------|
| **Azure Container Apps (Consumption)** | ~$0.000024/vCPU-sec | ~$0.000003/GiB-sec | $0.40/million requests | 180K vCPU-sec + 360K GiB-sec/month |
| **AWS Fargate (Linux/x86)** | $0.04048/vCPU-hr ($0.0000112/vCPU-sec) | $0.004445/GB-hr ($0.0000012/GB-sec) | N/A (no request charge) | None |
| **Google Cloud Run** | $0.000024/vCPU-sec | $0.0000025/GiB-sec | $0.40/million requests | 180K vCPU-sec + 360K GiB-sec/month |

#### Managed Kubernetes

| Platform | Control Plane | Compute (Example: 4 vCPU, 16 GB node) | Notes |
|----------|--------------|---------------------------------------|-------|
| **AKS (Standard)** | $0.10/hr ($73/mo) | ~$0.17/hr ($124/mo for D4s_v5) | Plus disk, LB, egress |
| **EKS** | $0.10/hr ($73/mo) | ~$0.17/hr ($122/mo for m6i.xlarge) | Plus EBS, ALB, egress |
| **GKE (Standard)** | $0.10/hr ($73/mo, 1 free zonal) | ~$0.17/hr ($122/mo for e2-standard-4) | Plus PD, LB, egress |

### 3.2 Cost Crossover Analysis

The fundamental question: **when does managed K8s become cheaper than serverless containers?**

**Worked example: a single always-on service (1 vCPU, 2 GB RAM)**

| Metric | Serverless (ACA/Cloud Run) | Managed K8s (3-node cluster) |
|--------|---------------------------|------------------------------|
| Compute cost/month | ~$63 (1 vCPU x 2.6M sec/mo x $0.000024) + ~$13 (2 GiB x 2.6M sec x $0.0000025/GiB-sec) = **~$76/mo** | Control plane: $73 + nodes (3x e2-medium ~$75): **~$148/mo** |
| Effective cost per service | **$76** | **$148** (but shared across services) |

**Crossover: per-service cost parity**

On K8s with a 3-node cluster (total ~12 vCPU, 48 GB), the base cost is ~$440/mo (control plane + 3x 4-vCPU nodes on-demand). That cost is fixed regardless of how many services you run.

| Number of Services (1 vCPU, 2 GB each) | Serverless Cost/mo | K8s Cost/mo (3-node) | Winner |
|----------------------------------------|--------------------|-----------------------|--------|
| 1 | $76 | $440 | Serverless (5.8x cheaper) |
| 3 | $228 | $440 | Serverless (1.9x cheaper) |
| 5 | $380 | $440 | Serverless (1.2x cheaper) |
| **6** | **$456** | **$440** | **K8s (crossover)** |
| 10 | $760 | $440 | K8s (1.7x cheaper) |
| 12 (cluster full) | $912 | $440 | K8s (2.1x cheaper) |

**With reserved instances / committed use discounts (1-year):**
K8s node costs drop ~30-40%. Crossover moves to ~4 always-on services.

**With bursty / low-utilisation workloads (avg 20% CPU):**
Serverless cost drops proportionally (scale-to-zero, pay-per-use). Crossover moves to ~15-20 services.

### 3.3 Key Cost Variables

| Factor | Favours Serverless | Favours K8s |
|--------|-------------------|-------------|
| Traffic pattern | Bursty, variable, long idle periods | Steady, predictable, always-on |
| Utilisation | < 30% average CPU | > 50% average CPU |
| Number of services | < 5-6 | > 6-8 |
| Request volume | < 10M/month | > 50M/month (request fees add up) |
| Scaling needs | Scale-to-zero matters | Min replicas always needed |
| Reserved pricing | Cannot commit (variable workload) | Can commit 1-3 years |
| Ops team cost | No platform engineers ($0 ops) | 0.5-1 FTE platform engineer (~$70-100K/yr) |

### 3.4 The Hidden Cost: Operations

This is the number most analyses miss. Managing a Kubernetes cluster requires:

| Activity | Estimated Hours/Month | Burdened Cost ($100/hr) |
|---------|----------------------|------------------------|
| Node patching and upgrades | 4-8 hrs | $400-800 |
| Monitoring and alerting tuning | 2-4 hrs | $200-400 |
| Incident response (K8s-specific) | 2-6 hrs | $200-600 |
| Capacity planning and rightsizing | 2-4 hrs | $200-400 |
| Ingress/cert/DNS management | 1-2 hrs | $100-200 |
| Security patching (CVEs) | 2-4 hrs | $200-400 |
| **Total operational overhead** | **13-28 hrs** | **$1,300-2,800/mo** |

For a small team, this operational tax often exceeds the compute savings. **The real crossover includes ops cost**, pushing K8s breakeven to ~15-20 always-on services or when you already have a dedicated platform team.

---

## 4. Operational Complexity: What You Do NOT Manage

### 4.1 Serverless Containers

**You do not manage:**
- Compute infrastructure (nodes, VMs, OS)
- Container runtime and scheduler
- Ingress controller and TLS termination
- Load balancer configuration
- Autoscaler implementation
- Node patching and security updates
- Capacity planning
- Cluster upgrades

**You do manage:**
- Container images (build, scan, push)
- Application configuration and environment variables
- Scaling rules (min/max replicas, concurrency)
- Custom domains and DNS
- Secrets (referencing vault/secret manager)
- Database and external service connections
- Observability (application-level; platform provides infra-level)
- IAM and access control (app-level)

### 4.2 Managed Kubernetes

**You do not manage:**
- Control plane (API server, etcd, scheduler)
- Control plane upgrades (though you trigger them)
- Control plane high availability

**You do manage (everything else):**
- Worker node pools (sizing, scaling, OS image)
- Node OS patching (can be automated but must be configured)
- Kubernetes version upgrades (must plan and execute)
- Ingress controllers (install, configure, maintain)
- TLS certificate management (cert-manager or equivalent)
- Network policies
- RBAC policies
- Storage classes and persistent volume provisioning
- Service mesh (if needed)
- Monitoring stack (Prometheus, Grafana, alerting rules)
- Logging pipeline (Fluentd/Fluent Bit to central logging)
- Secrets management integration
- Pod security policies / standards
- Resource quotas and limit ranges
- Helm charts / Kustomize manifests
- CI/CD pipeline integration
- Disaster recovery and backup

---

## 5. Use Case Fit

### 5.1 Serverless Containers Are Right For

- **Web APIs and microservices** with variable traffic (especially if traffic goes to zero)
- **Event-driven processing** (queue consumers, webhook handlers)
- **Scheduled jobs** (cron-like batch processing)
- **Early-stage products** where operational simplicity trumps flexibility
- **Small teams** (< 5 engineers) without dedicated platform/infra roles
- **Multi-region deployment** where you want identical stacks without managing multiple clusters
- **AI/ML inference** with variable demand (especially ACA serverless GPU, Cloud Run GPU)
- **Prototype and staging environments** that should cost near-zero when idle

### 5.2 Managed Kubernetes Is Right For

- **Large microservice architectures** (> 10-15 services) with steady traffic
- **Stateful workloads** requiring persistent volumes and stable network identity
- **Multi-tenant platforms** requiring namespace-level isolation with network policies
- **Compliance-heavy environments** needing fine-grained RBAC, audit logging, pod security
- **Custom operators and controllers** (e.g., database operators, workflow engines)
- **Hybrid/multi-cloud** deployments where K8s portability matters
- **Teams with existing Kubernetes expertise** and platform engineering capacity
- **GPU-heavy ML training** requiring node-level GPU scheduling and MIG
- **Service mesh requirements** (Istio, mTLS between all services, traffic splitting)

### 5.3 Grey Zone (Either Could Work)

- **5-10 microservices, moderate traffic**: Cost-similar. Choose based on team skill.
- **Queue-driven workers that run 24/7**: Serverless per-second billing vs K8s reserved instances. Depends on count.
- **Internal tools and dashboards**: Serverless is simpler. K8s if already running a cluster.

---

## 6. Dev-House Relevance Analysis

### 6.1 Customer Deployment Characteristics

Based on the Dev-House architecture:

| Characteristic | Value | Implication |
|---------------|-------|-------------|
| **Workload type** | API services (Claude-bound) | CPU-idle most of the time (waiting on API calls, 10-20s latency) |
| **CPU utilisation** | Very low (< 10% average) | Serverless billing advantage is massive |
| **Multi-tenancy model** | Schema-per-customer or namespace-per-customer | Schema-per-customer works on both; namespace-per-customer requires K8s |
| **Provisioning** | Terraform-automated | Both approaches are Terraform-friendly |
| **GPU requirement** | None in customer system | Eliminates a K8s advantage |
| **Service count per customer** | 2-5 (API gateway, harness service, worker, maybe frontend) | Below the K8s crossover point |
| **Traffic pattern** | Bursty (PRD submission, then idle) | Scale-to-zero saves significant cost |
| **Compliance** | Varies by customer | May force K8s for some enterprise customers |

### 6.2 Recommendation by Customer Profile

| Customer Profile | Recommended Approach | Rationale |
|-----------------|---------------------|-----------|
| **Startup / SMB** (< 50 employees, < 5 services) | **Serverless containers (ACA or Cloud Run)** | Near-zero idle cost, zero ops burden, Terraform-deployable. Claude API latency dominates; no reason to pay for idle compute. |
| **Mid-market** (50-500 employees, 5-15 services, some compliance) | **Serverless containers with Dedicated plan** (ACA Dedicated or GKE Autopilot) | Dedicated compute for compliance (single-tenancy), but still managed. VNet injection for network isolation. |
| **Enterprise** (> 500 employees, > 15 services, strict compliance, existing K8s) | **Managed Kubernetes (AKS/EKS/GKE)** | Namespace-per-customer isolation, network policies, custom RBAC, integration with existing platform team. Customer likely already has K8s. |
| **Regulated industry** (healthcare, finance, government) | **Managed Kubernetes** or **Serverless on Dedicated plan** | Depends on specific compliance requirements. Some regulations mandate network isolation that only K8s network policies or dedicated compute can provide. |

### 6.3 Dev-House Default: Start Serverless

For the typical Dev-House customer deployment:

1. **Use Azure Container Apps (Consumption plan)** or **Google Cloud Run** as the default
2. **Schema-per-customer** multi-tenancy (shared infrastructure, isolated data)
3. **Terraform modules** that abstract the platform (so migration to K8s is a module swap, not a rewrite)
4. **External databases** (Azure SQL / Cloud SQL / RDS) for state; containers remain stateless
5. **Scale-to-zero** for customer environments that are idle (development, staging, infrequent-use customers)

**Cost implication**: A customer with 3 services that processes PRDs a few times per day might pay $5-15/month in compute on serverless vs $150-200/month minimum on K8s. For a multi-tenant deployment serving 20 customers, the aggregate savings are substantial.

### 6.4 When to Recommend K8s to a Customer

Escalate from serverless to K8s when ANY of these are true:

- Customer requires **namespace-per-tenant network isolation** (not just schema separation)
- Customer has **> 15 always-on services** with steady traffic
- Customer's compliance framework **mandates Kubernetes** or equivalent pod-level security policies
- Customer already operates a K8s cluster and wants **workloads co-located**
- Customer needs **custom Kubernetes operators** (e.g., for database management, workflow orchestration)
- Customer's average CPU utilisation is **> 40% sustained** (reserved K8s becomes cheaper)

---

## 7. Migration Path

### 7.1 Serverless to Kubernetes: What Carries Over

| Asset | Portable? | Notes |
|-------|-----------|-------|
| **Container images** | Yes, fully | OCI standard. Push to any registry. |
| **Dockerfiles** | Yes, fully | No changes needed. |
| **Application code** | Yes, fully | If properly containerised. |
| **Environment variables** | Yes, mostly | Map to K8s ConfigMaps/Secrets. |
| **Database connections** | Yes, fully | External managed databases are platform-independent. |
| **Terraform modules** | Partially | Resource definitions change entirely. Module interface can stay stable. |
| **CI/CD pipelines** | Partially | Build steps carry over. Deploy steps rewritten. |
| **Monitoring dashboards** | No | Platform-specific metrics. Rebuild on Prometheus/Grafana. |

### 7.2 Serverless to Kubernetes: What Breaks

| Asset | What Breaks | Migration Effort |
|-------|------------|-----------------|
| **Scaling rules** | ACA/Cloud Run scaling config does not translate to HPA/KEDA config. Must be rewritten. | Low (hours) |
| **Ingress configuration** | Platform-managed ingress replaced by ingress controller + Ingress/Gateway resources. | Medium (days) |
| **TLS certificates** | Automatic TLS replaced by cert-manager + Let's Encrypt or similar. | Low (hours) |
| **Secrets references** | Platform secret syntax (ACA secret refs, Cloud Run Secret Manager mounts) replaced by K8s Secrets + CSI driver. | Low (hours) |
| **IAM bindings** | Workload identity / managed identity configuration is entirely different between serverless and K8s. | Medium (days) |
| **Dapr sidecars (ACA-specific)** | Dapr works on K8s but requires installing the Dapr runtime on the cluster. Config changes. | Medium (days) |
| **Health checks** | Platform health checks replaced by K8s liveness/readiness/startup probes. Conceptually similar, syntactically different. | Low (hours) |
| **Terraform state** | Complete resource replacement. `azurerm_container_app` becomes `azurerm_kubernetes_cluster` + Helm releases. Cannot in-place migrate. | High (days-weeks) |

### 7.3 Kubernetes to Serverless: What Breaks

| Asset | What Breaks | Notes |
|-------|------------|-------|
| **Network policies** | Serverless platforms have no equivalent. Must redesign isolation. | Blocker for some workloads |
| **Persistent volumes** | Must migrate to external storage (database, object store, NFS service). | Medium-high effort |
| **StatefulSets** | No equivalent. Must redesign as stateless + external state. | High effort if stateful |
| **Custom operators/CRDs** | No equivalent. Must replace with platform-native features or external services. | May be impossible |
| **DaemonSets** | No equivalent. Sidecar patterns differ. | Redesign required |
| **Helm charts** | Not applicable. Rewrite as platform config (YAML, Terraform, Bicep). | Medium effort |

### 7.4 Migration Risk Mitigation

**Design principle**: Build for portability from day one.

1. **Abstract the platform in Terraform**: Create a module interface (`deploy_service(name, image, cpu, memory, env, secrets)`) that can be backed by either ACA or K8s. Customer-facing Terraform calls the abstraction, not the platform directly.

2. **Keep containers stateless**: All state in managed databases. No local disk dependencies. No sticky sessions.

3. **Externalise configuration**: Environment variables and secret references, not platform-specific config files baked into images.

4. **Use standard health check patterns**: HTTP `/health` and `/ready` endpoints work on both serverless and K8s.

5. **Avoid deep Dapr dependency**: Dapr is portable to K8s, but if you only use it for service discovery, DNS-based discovery works everywhere. Use Dapr only if you need its pub/sub or state management features.

---

## 8. Recommendation Matrix

| Decision Factor | Choose Serverless Containers | Choose Managed Kubernetes |
|----------------|-------|------|
| **Team size** | < 10 engineers, no platform team | > 10 engineers, or dedicated platform team |
| **Service count** | 1-8 services | > 10 services |
| **Traffic pattern** | Bursty, variable, idle periods | Steady, predictable, always-on |
| **CPU utilisation** | < 30% average | > 50% average |
| **Multi-tenancy** | Schema-per-tenant (shared infra) | Namespace-per-tenant (isolated infra) |
| **Compliance** | SOC2, basic HIPAA (with dedicated plan) | PCI-DSS, FedRAMP, strict HIPAA, custom audit |
| **State requirements** | Stateless services + managed DB | Stateful services, persistent volumes |
| **Networking** | HTTP/gRPC services, standard ingress | Custom protocols, network policies, service mesh |
| **GPU** | Inference with variable demand (ACA/Cloud Run) | Training, multi-GPU, time-slicing |
| **Budget** | Minimise idle cost, pay-per-use | Optimise total cost at scale, commit to reserved |
| **Time to production** | Days (image + config = deployed) | Weeks (cluster + networking + monitoring + security) |
| **Vendor lock-in tolerance** | Moderate (platform-specific config, portable images) | Low (K8s is portable across clouds; manifests carry over) |

---

## 9. Platform-Specific Notes

### Azure Container Apps

- Best Dapr integration of any serverless platform (built-in sidecar, managed components)
- Workload profiles allow mixing consumption and dedicated compute in one environment
- Serverless GPU support (A100, T4) with scale-to-zero is unique
- VNet injection for private networking
- No Windows container support
- No Kubernetes API access (it runs on AKS underneath, but you cannot kubectl)
- Job support is solid (event-driven, scheduled, manual triggers)

### AWS Fargate / App Runner

- Fargate is the most mature serverless container runtime (since 2017)
- No scale-to-zero on Fargate (minimum 1 task). App Runner pauses containers (memory-only billing)
- Fargate supports EFS mounts (only serverless platform with NFS-like persistent storage)
- App Runner is extremely limited (HTTP only, no VPC by default, no custom ports)
- ECS service connect provides basic service mesh without Istio complexity
- Graviton (ARM) pricing is ~20% cheaper than x86

### Google Cloud Run

- Cleanest developer experience (source deploy, automatic TLS, instant URL)
- Best scale-to-zero implementation (millisecond cold starts with min-instances=0)
- No persistent volume support (design for stateless or use GCS/Firestore)
- GPU support (L4) available
- Cloud Run Jobs for batch workloads
- Tight integration with Eventarc for event-driven patterns
- Second-generation execution environment removes most sandbox limitations

### Azure AKS

- Free tier control plane (no SLA) is genuinely free; useful for dev/test
- Best Windows container support on managed K8s
- Azure CNI overlay simplifies networking
- Workload identity federation for Azure AD integration
- KEDA pre-installed in some configurations

### AWS EKS

- EKS Auto Mode reduces node management (similar to GKE Autopilot concept)
- Karpenter for node provisioning is best-in-class
- Extended support pricing ($0.60/hr) after 14 months is a gotcha; plan upgrades
- EKS Anywhere for on-premises (if customer needs hybrid)

### Google GKE

- Autopilot mode is the closest K8s gets to serverless (pay per pod, Google manages nodes)
- GKE is consistently the most polished managed K8s experience
- Multi-cluster Services (MCS) for cross-cluster service discovery
- Config Sync for GitOps built-in
- Gateway API support is ahead of other platforms

---

## 10. Sources

### Pricing Pages (Verify Before Procurement)

- [Azure Container Apps Pricing](https://azure.microsoft.com/en-us/pricing/details/container-apps/)
- [AWS Fargate Pricing](https://aws.amazon.com/fargate/pricing/)
- [Google Cloud Run Pricing](https://cloud.google.com/run/pricing)
- [Azure AKS Pricing](https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/)
- [AWS EKS Pricing](https://aws.amazon.com/eks/pricing/)
- [Google GKE Pricing](https://cloud.google.com/kubernetes-engine/pricing)

### Comparison and Analysis

- [ACA vs AKS - Microsoft Tech Community](https://techcommunity.microsoft.com/blog/startupsatmicrosoftblog/aca-vs-aks-which-azure-service-is-better-for-running-containers/3815164)
- [Comparing Container Apps with other Azure container options - Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/compare-options)
- [AWS App Runner vs Fargate - cloudonaut](https://cloudonaut.io/fargate-vs-apprunner/)
- [GKE Autopilot vs Standard Pricing - DevZero](https://www.devzero.io/blog/gke-pricing)
- [The True Cost of Kubernetes - Koyeb](https://www.koyeb.com/blog/the-true-cost-of-kubernetes-people-time-and-productivity)
- [AWS Fargate Pricing Explained - Cloud Ex Machina](https://www.cloudexmachina.io/blog/fargate-pricing)

### Capabilities and Limitations

- [Azure Container Apps Workload Profiles - Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/workload-profiles-overview)
- [Serverless GPUs in Azure Container Apps - Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview)
- [AKS Pricing Tiers - Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers)
- [CyberCube's Shift from Serverless to Kubernetes](https://insights.cybcube.com/en/a-shift-from-aws-serverless-to-kubernetes)

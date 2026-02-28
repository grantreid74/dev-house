# OpenClaw Integration

## Purpose

OpenClaw is a **workflow orchestration and infrastructure automation platform** that complements Claude's code generation with:
- Complex workflow management (multi-stage deployments)
- Infrastructure orchestration (Terraform, Ansible, cloud provider CLIs)
- State tracking and rollback
- Policy enforcement and compliance
- Cost optimization

## Where OpenClaw Fits

```
Claude: Code generation, architecture recommendations
OpenClaw: Orchestrate generated code + infrastructure deployment
```

## Use Cases

### 1. Complex Deployments
**Scenario**: Deploy microservices architecture with multiple dependencies

Claude generates:
- Service code
- Docker configs
- Kubernetes manifests

OpenClaw:
- Orchestrates deployment order (dependencies first)
- Applies infrastructure (networks, storage)
- Manages secrets and configuration
- Validates health

### 2. Infrastructure Orchestration
**Scenario**: Provision cloud infrastructure from Terraform

Claude generates Terraform code.

OpenClaw:
- Validates Terraform syntax
- Plans infrastructure changes
- Applies with proper sequencing
- Tracks state
- Implements rollback on failure

### 3. Policy & Compliance
**Scenario**: Ensure all deployed infrastructure meets compliance standards

OpenClaw:
- Enforces tagging policies
- Validates security groups
- Checks cost budgets
- Audits deployments

### 4. Cost Optimization
**Scenario**: Reduce cloud spend while maintaining performance

OpenClaw:
- Analyzes resource utilization
- Recommends rightsizing
- Implements auto-scaling
- Tracks costs per deployment

## API Integration

### Harness → OpenClaw

```
POST /workflows/create
{
  "name": "deploy-task-api",
  "steps": [
    {
      "name": "provision-infrastructure",
      "type": "terraform",
      "config": "generated-terraform.tf"
    },
    {
      "name": "build-docker-image",
      "type": "docker",
      "dockerfile": "Dockerfile"
    },
    {
      "name": "deploy-to-kubernetes",
      "type": "kubernetes",
      "manifest": "k8s-deployment.yaml"
    }
  ],
  "policies": [
    "require-encryption",
    "enforce-tags",
    "limit-budget-1000"
  ]
}

Response:
{
  "workflow_id": "wf_abc123",
  "status": "created"
}
```

### Monitoring

```
GET /workflows/wf_abc123
{
  "status": "in_progress",
  "steps": [
    {
      "name": "provision-infrastructure",
      "status": "completed",
      "duration": 120
    },
    {
      "name": "build-docker-image",
      "status": "in_progress",
      "progress": 45
    }
  ],
  "estimated_cost": 45.50,
  "policy_violations": []
}
```

## State Management

OpenClaw maintains:
- **Workflow state** — What's running, what's done, what failed
- **Infrastructure state** — What's deployed, where, what version
- **Audit trail** — Who changed what, when, why
- **Rollback checkpoint** — Save point for recovery

## Error Handling

- **Transient errors** → Auto-retry with backoff
- **Validation errors** → Fail fast, report details
- **Partial failure** → Rollback or pause for manual intervention

## Integration Points with Harness

1. **After code generation** → OpenClaw deploys generated code
2. **For infrastructure** → OpenClaw applies generated Terraform
3. **For validation** → OpenClaw validates compliance
4. **For monitoring** → OpenClaw provides deployment status

## To Be Detailed

- OpenClaw API reference
- Policy language
- Workflow DSL
- Security & authentication
- Cost tracking and optimization
- Disaster recovery workflows


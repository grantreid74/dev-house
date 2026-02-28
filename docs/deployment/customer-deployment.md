# Customer Deployment Guide

## Overview

Customers run the Dev-House harness to convert their PRD into deployed infrastructure and applications.

## Deployment Models

### Model 1: Managed (SaaS)
- Harness runs on customer-managed cloud infrastructure
- Customer provides API keys (Claude, cloud provider)
- We manage harness updates and monitoring
- Customer manages deployment targets

### Model 2: Self-Hosted
- Customer runs harness in their own environment (Docker container, VMs, K8s)
- Customer manages all infrastructure
- Customer manages harness updates

## Setup

### Prerequisites
- Claude API key (from Anthropic)
- Cloud provider credentials (AWS, Azure, GCP, or on-prem)
- Docker (for self-hosted)
- Git

### Quick Start

```bash
# Clone dev-house
git clone <repo> dev-house
cd dev-house

# Set up credentials
export CLAUDE_API_KEY="sk-..."
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."

# Run harness
docker run -e CLAUDE_API_KEY -e AWS_* dev-house:latest < my-prd.md
```

## Configuration

Customer configures via environment variables or config file:

```yaml
# config.yaml
harness:
  cache_ttl: 3600
  retry_max_attempts: 3
  retry_backoff: exponential

claude:
  model: claude-3-opus
  # API key from CLAUDE_API_KEY env var

deployment:
  target: aws  # aws, azure, gcp, k8s, docker
  region: us-east-1
  budget_limit: 1000  # $ per deployment

monitoring:
  log_level: info
  export_logs: s3://bucket/logs
```

## Workflow

1. **Provide PRD** → `harness run < prd.md`
2. **Monitor** → Real-time progress, logs
3. **Review** → Generated code, infrastructure
4. **Approve** → Deploy to customer environment
5. **Monitor** → Deployment progress, health

## Security Considerations

- **API Keys**: Never hardcode, use secrets management
- **Code Review**: Customer reviews generated code before deployment
- **Network**: Harness-to-Claude goes over HTTPS, customer-to-cloud over VPN
- **Audit**: All Claude calls logged for compliance

## Troubleshooting

- Deployment fails → Check logs, review generated code, retry
- Generation fails → PRD may be incomplete, provide clarification
- Cost overruns → Review deployment config, limit budget

## To Be Detailed

- Detailed setup for each cloud provider
- Monitoring and observability setup
- Rollback procedures
- Disaster recovery


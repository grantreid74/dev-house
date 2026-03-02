# D2 — SaaS Managed Delivery Model

**Status**: Future exploration — not current practice

> This document captures the SaaS managed model as a potential expansion path. It is not being built. Do not reference it in customer conversations or cost models until it is actively scoped.

---

## Concept

Dev-House builds the customer's system and continues to operate it on a retainer. Customer pays a monthly fee covering both the infrastructure cost and Dev-House's operational management.

```
Dev-House builds → Dev-House provisions → Dev-House operates → Customer uses
                                          (ongoing, on retainer)
```

---

## Why It's Not the Default

- Dev-House is currently 2 people
- Operating other people's infrastructure is a different business with different skills, tools, and SLAs
- Incident response, on-call, and capacity planning require dedicated operational capacity
- The D1 handover model is simpler, lower-risk, and more appropriate at current scale

---

## When It Might Make Sense

- Customer has no internal ops capability and cannot hire
- Long-term relationship with a customer who trusts Dev-House to operate
- Retainer revenue is sufficient to justify the operational overhead (staffing, tooling, SLA obligations)
- Dev-House has grown to a size where a dedicated ops function is viable

---

## What Would Need to Be Defined

Before offering D2 to any customer, these must be answered:

| Question | Why It Matters |
|----------|----------------|
| What SLA is Dev-House committing to? (uptime, response time) | Legal and commercial obligation |
| Who is on-call? | 2am incidents require a named human |
| What tooling runs the monitoring stack? | PagerDuty, Grafana, alerting pipelines |
| How are credentials managed? Dev-House holds customer cloud credentials? | Security and trust boundary |
| How is multi-customer isolation enforced? | One customer's incident must not affect another |
| What is the pricing model? | Must cover infrastructure + operational overhead + margin |
| What is the exit clause? | Customer must be able to take back operations |

---

## Relationship to Other Models

- **Generation Infrastructure** (G-series) remains unchanged — Dev-House still runs its own agents to generate code
- **D2 adds** ongoing operational responsibility for the customer's production system
- **D-T3 / D-T4 patterns** still apply to what the system looks like — D2 is about who runs it, not what it is

---

## See Also

- **[D1-handover.md](../D1-handover.md)** — Current default delivery model
- **[CURRENT_REVIEW.md](../../../CURRENT_REVIEW.md)** — Open decisions; add D2 questions here when actively scoping

# PRD: Acme Task Manager

> **Template version**: 1.0.0
> **Status**: SIGNED-OFF
> **Change freeze**: 2026-03-10
> **Customer ID**: acme-taskr

---

## 1. Business Context

- **Company**: Acme Corp — 6-person product studio, bootstrapped, no compliance mandates
- **Their customers**: Internal team only (not a customer-facing product yet)
- **Current state**: Tasks tracked in a shared Notion workspace. Brittle, no automation, no email notifications, no API access.
- **Why now**: Team is growing to 10 people. Notion breaks at this scale. Founder wants to dogfood a Dev-House-built product internally before selling to customers.

---

## 2. Product Vision

A lightweight internal task management tool that gives Acme's team a single place to create projects, assign tasks, set deadlines, and get notified when things are overdue. Not a competitor to Jira — a focused tool that does five things well. The goal is to replace Notion task tracking within 30 days and give the founder real evidence that Dev-House can ship a working product from a PRD.

---

## 3. Success Criteria

- [ ] Team members can log in with Google OAuth and see their assigned tasks
- [ ] All core task and project features pass smoke tests in UAT
- [ ] Email notifications arrive within 2 minutes of trigger events
- [ ] System handles 20 concurrent users without visible slowdown
- [ ] Terraform plan validates and the app runs in the cloud (Tier 1 shared infra)

---

## 4. User Types

| User Type | Description | Estimated Count | Access Level |
|-----------|-------------|-----------------|--------------|
| Admin | Creates/manages projects, assigns tasks, views all | 2 | Full |
| Team Member | Creates/completes tasks, comments, uploads files | 8 | Standard |

**Peak concurrent users**: 10
**Growth trajectory**: Stable for 6 months; may open to external customers in Phase 2 (out of scope here)

---

## 5. Core Features

### Authentication
- [ ] [AUTH] Google OAuth login (acceptance: team member can sign in with @acmecorp.com Google account)
- [ ] [AUTH] Session persistence across browser refreshes (acceptance: user stays logged in for 24 hours)
- [ ] [AUTH] Role assignment (admin vs member) by admin (acceptance: admin can change a user's role)

### Projects
- [ ] [PROJECT] Create a project with name, description, and due date (acceptance: project appears in project list)
- [ ] [PROJECT] Archive a completed project (acceptance: archived projects hidden from default view; retrievable via filter)
- [ ] [PROJECT] Project dashboard showing task summary — open, in progress, completed counts (acceptance: counts correct)

### Tasks
- [ ] [TASK] Create a task within a project with title, description, assignee, due date, and priority (acceptance: task saved and visible to assignee)
- [ ] [TASK] Update task status: To Do → In Progress → Done (acceptance: status change persists and is timestamped)
- [ ] [TASK] Assign/reassign a task to any team member (acceptance: new assignee sees task in their queue)
- [ ] [TASK] Add comments to a task (acceptance: comments appear in reverse-chronological order)
- [ ] [TASK] Attach a file to a task (acceptance: file uploadable up to 10MB; downloadable by any team member)
- [ ] [TASK] Filter task list by: status, assignee, priority, due date (acceptance: filters combinable)

### Notifications
- [ ] [NOTIFY] Email when a task is assigned to you (acceptance: email arrives within 2 minutes)
- [ ] [NOTIFY] Email when a task you own is commented on (acceptance: email arrives within 2 minutes)
- [ ] [NOTIFY] Daily digest email: tasks due in next 48 hours (acceptance: digest sent at 8am in user's timezone)
- [ ] [NOTIFY] User can opt out of any notification type (acceptance: opted-out events produce no email)

### Admin
- [ ] [ADMIN] Admin view of all projects and tasks across all team members (acceptance: admin sees everything; member sees only assigned/created)
- [ ] [ADMIN] Export all tasks for a project as CSV (acceptance: CSV includes title, assignee, status, due date, created date)

---

## 6. Non-Functional Requirements

### Compliance & Security
- **Compliance mandates**: None
- **Data sensitivity**: Internal only — no PII beyond employee names and emails
- **Audit trail required**: No
- **Data residency**: No requirement
- **Multi-tenancy**: Single customer (Acme internal use only — no other tenants)

### Availability & Performance
- **Target uptime**: Best effort (99% acceptable — internal tool, team works business hours)
- **RTO**: 4 hours (internal tool; next-business-day acceptable)
- **RPO**: 24 hours
- **Response time**: API p95 < 1 second (light usage, no strict SLA)
- **Peak load**: 10 concurrent users max

### Integration
- **Third-party integrations required**: Google OAuth (login), SendGrid (email notifications)
- **Existing systems**: None — greenfield
- **API requirements**: REST API (needed for potential Phase 2 external access)

---

## 7. Deployment Preferences

- **Cloud provider preference**: No preference — Dev-House decides
- **Infrastructure budget**: $50/month maximum (internal tool; founder is cost-conscious)
- **Kubernetes preference**: No — prefer simplest possible option
- **Custom domain required**: No (internal URL acceptable: `taskr.dev-house.io/acme`)
- **Existing infrastructure**: Greenfield
- **Self-hosted option**: Not required
- **Timeline to UAT**: 4 weeks from PRD sign-off

### Expected pattern selection: Tier 1 (Fully Shared)

Reasoning: $50/month budget eliminates Tier 3+. Single tenant (Acme only), no compliance, no custom domain. Tier 1 shared infra is the correct pattern. Application-level routing (URL prefix `/acme`) is sufficient.

---

## 8. Out of Scope

- Mobile app (web only)
- Time tracking or estimation features
- External customer access (Phase 2)
- Integrations beyond Google OAuth and SendGrid (Slack, GitHub, etc.)
- Gantt charts or timeline views
- Custom fields on tasks
- Billing or payment features

---

## 9. Open Questions

| Question | Owner | Impact if Unresolved |
|----------|-------|----------------------|
| Which timezone does the daily digest use? User's browser TZ, or UTC? | Customer | Harness will default to UTC if unresolved |
| File attachment storage — is S3-compatible blob storage acceptable, or does customer want local disk? | Customer | Defaults to cloud blob storage (Tier 1 shared) |

---

## 10. Provenance

```
Template:  examples/prd-template.md v1.0.0
Customer:  acme-taskr
PRD owner: founder@acmecorp.com
Signed:    2026-03-10
Reviewed:  Dev-House
```

---

## Harness Notes

*Added by Harness Initializer — do not edit manually*

```yaml
# To be populated by Harness Session 1
pattern_selected: tier1-shared
services_identified: []
feature_count: 0
harness_status: pending
```

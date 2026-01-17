```md
# XenHirely Email Outreach Subsystem (Simple Architecture)

## High-Level Flow

```

Users (Recruiters / Admins / Hiring Managers)
|
v
Next.js Web App (UI)
|
v
Node.js Express API Gateway
|
+------------------------------------------------------+
|                                                      |
v                                                      v
Core Product Modules                                    Storage + Infra

```

---

## Core Product Modules (API-owned)

```

+---------------------------+
| Auth + Workspace Access   |
| - Login                   |
| - RBAC permissions        |
| - Workspace isolation     |
+---------------------------+

+---------------------------+
| Domain + Deliverability   |
| - SPF / DKIM / DMARC      |
| - DNS record validation   |
+---------------------------+

+---------------------------+
| Sender Connections        |
| - Gmail OAuth             |
| - Microsoft OAuth         |
| - SMTP servers            |
| - Email APIs (SES, etc.)  |
+---------------------------+

+---------------------------+
| Delegation (Send as)      |
| - Consent + approvals     |
| - Audit logs              |
+---------------------------+

+---------------------------+
| Template Studio           |
| - Templates               |
| - Variables               |
| - Brand kits              |
| - Attachments             |
+---------------------------+

+---------------------------+
| Campaign Engine           |
| - Sequences               |
| - Scheduling windows      |
| - Throttling / limits     |
| - Push send jobs to queue |
+---------------------------+

+---------------------------+
| Inbox + Reply Monitoring  |
| - Threading               |
| - Stop rules              |
| - Reply capture           |
+---------------------------+

+---------------------------+
| Analytics + Reporting     |
| - Funnels                 |
| - Cohorts                 |
| - Export                  |
+---------------------------+

```

---

## Storage Layer

```

+-------------------------------+
| Postgres (System of Record)   |
| - users / workspaces          |
| - templates / campaigns       |
| - recipients                  |
| - events (open/click/bounce)  |
| - suppression list            |
| - analytics rollups           |
+-------------------------------+

+-------------------------------+
| Redis                         |
| - job queues                  |
| - rate limiting               |
| - retries + dedupe helpers    |
+-------------------------------+

+-------------------------------+
| S3                            |
| - attachments                 |
| - raw webhook payloads        |
| - email raw bodies (optional) |
+-------------------------------+

```

---

## Async Queues + Workers (Background Processing)

```

API Gateway
|
+--> Queue: send-email ---------------> Worker: Mail Sender
|                                        - idempotent sends
|                                        - retries
|                                        - provider adapter layer
|
+--> Queue: process-event ------------> Worker: Event Processor
|                                        - webhook event dedupe
|                                        - stop rules
|                                        - suppression updates
|
+--> Queue: classify-reply -----------> Worker: Reply Classifier
|                                        - intent labels (Interested, OOO, etc.)
|
+--> Queue: recompute-analytics ------> Worker: Analytics Aggregator
|                                        - rollups for dashboards
|
+--> Queue: poll-inbox (fallback) ----> Worker: IMAP Poller
- fetch replies when webhooks fail

```

---

## Provider Sending Layer (Adapter Pattern)

```

Worker: Mail Sender
|
v
Mail Sending Engine (Provider Adapter Layer)
|
+--> Gmail OAuth
+--> Microsoft OAuth
+--> SMTP
+--> SendGrid API
+--> Mailgun API
+--> AWS SES API
+--> Postmark API
|
v
Recipients Inbox

```

---

## Inbound Events (Webhooks + Fallback Polling)

### Preferred: Provider Webhooks
```

Email Providers (delivered/open/click/bounce/reply)
|
v
Webhooks
|
v
API Gateway
|
v
Queue: process-event -> Worker: Event Processor -> Postgres

```

### Fallback: IMAP Polling
```

Worker: IMAP Poller
|
v
Inbox (Gmail / Microsoft / SMTP IMAP)
|
v
Queue: process-event -> Worker: Event Processor -> Postgres

```

---

## Suppression + Compliance (Hard Block)

Suppression list contains:
- Unsubscribes
- Bounces
- Complaints

Enforced in multiple places:
- Campaign Engine: does not schedule suppressed recipients
- Mail Sender Worker: refuses to send if suppressed
- Event Processor: updates suppression immediately when events arrive
```

# XenHirely Email Outreach Subsystem (End-to-End Architecture)

```text
+--------------------------------------------------------------------------------------+
|                               XenHirely Email Outreach                               |
+--------------------------------------------------------------------------------------+

Users (Recruiters / Admins / Hiring Managers)
        |
        v
+---------------------+        HTTPS/API Calls        +------------------------------+
|  Next.js Web App    |  -------------------------->  |  Node.js Express API Gateway |
|  (UI)               |                               |  (Auth + Routing)            |
+---------------------+                               +------------------------------+
                                                        |
                                                        | verifies access (RBAC + workspace)
                                                        | validates deliverability + config
                                                        | writes data + enqueues jobs
                                                        v
+--------------------------------------------------------------------------------------+
|                                 Core Backend Modules                                 |
|--------------------------------------------------------------------------------------|
| Auth + RBAC + Workspace Access                                                       |
| Domain + Deliverability (SPF/DKIM/DMARC via DNS lookups)                             |
| Sender Connections (Gmail OAuth / Microsoft OAuth / SMTP / Email APIs)               |
| Delegation "Send on behalf" (consent + approvals + audit logs)                       |
| Template Studio (variables + brand kits + attachments)                               |
| Campaign Engine (sequences + scheduling + throttling)                                |
| Inbox Monitoring (threading + stop rules)                                            |
| Analytics + Reporting (funnels + cohorts + export)                                   |
+--------------------------------------------------------------------------------------+
                                                        |
                                                        v
+----------------------+      +----------------------+      +----------------------+
| Postgres             |      | Redis                |      | S3                   |
| System of Record     |      | Queues + Rate Limits |      | Attachments + Payload|
+----------------------+      +----------------------+      +----------------------+
        |                              |
        |                              | enqueue background work
        |                              v
        |                 +-----------------------------------------------+
        |                 |                  Queues                       |
        |                 |-----------------------------------------------|
        |                 | send-email                                    |
        |                 | process-event                                 |
        |                 | classify-reply                                |
        |                 | recompute-analytics                           |
        |                 | poll-inbox (IMAP fallback)                    |
        |                 +-----------------------------------------------+
        |                              |
        |                              v
        |                 +-----------------------------------------------+
        |                 |                  Workers                      |
        |                 |-----------------------------------------------|
        |                 | Mail Sender (idempotent + retries)            |
        |                 | Event Processor (dedupe + stop rules)         |
        |                 | Reply Classifier (intent labels)              |
        |                 | Analytics Aggregator (rollups)                |
        |                 | IMAP Poller (fallback inbound fetch)          |
        |                 +-----------------------------------------------+
        |                              |
        |                              v
        |                 +----------------------------------------------+
        |                 |   Mail Sending Engine (Provider Adapters)    |
        |                 +----------------------------------------------+
        |                     |     |      |       |        |       |
        |                     v     v      v       v        v       v
        |                  Gmail  MS365   SMTP    SES   SendGrid  Mailgun/Postmark
        |                     \      \      \       \        \        \
        |                      \      \      \       \        \        v
        |                       \      \      \       \        \   Recipients Inbox
        |                        \      \      \       \        v
        |                         \      \      \       v   Recipients Inbox
        |                          \      \      v   Recipients Inbox
        |                           \      v  Recipients Inbox
        |                            v  Recipients Inbox
        |
        |  inbound signals (delivered/open/click/bounce/reply)
        v
+--------------------------------------------------------------------------------------+
|                               Inbound Events + Replies                               |
|--------------------------------------------------------------------------------------|
| Provider Webhooks (preferred): Gmail/MS365/SES/SendGrid/Mailgun/Postmark -> API      |
| IMAP Polling (fallback): IMAP Poller -> API                                          |
|                                                                                      |
| Event Processor updates:                                                             |
| - message status + timeline events                                                   |
| - stop rules (stop sequence on reply/bounce/unsubscribe)                             |
| - suppression list (unsubscribe/bounce/complaint)                                    |
+--------------------------------------------------------------------------------------+

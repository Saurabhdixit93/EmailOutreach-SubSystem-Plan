# XenHirely Email Outreach Subsystem

## Architecture Overview 

```mermaid
flowchart TB
  %% USERS
  U[Users: Recruiters / Admins / Hiring Managers] --> WEB[Next.js Web App]
  WEB --> API[Node.js Express API Gateway]

  %% BACKEND MODULES
  API --> AUTH[Auth + RBAC + Workspace Access]
  API --> DOMAIN[Domain + Deliverability\nSPF/DKIM/DMARC Validation]
  API --> SENDER[Sender Connections\nOAuth + SMTP + API Providers]
  API --> DELEG[Send on behalf\nConsent + Approvals + Audit Logs]
  API --> TEMPLATE[Templates Studio\nVariables + Brand Kits + Attachments]
  API --> CAMPAIGN[Campaign Engine\nSequences + Scheduling + Throttling]
  API --> INBOX[Inbound Monitoring\nThreading + Stop Rules]
  API --> ANALYTICS[Analytics + Reporting\nFunnels + Cohorts + Export]

  %% STORAGE
  API --> PG[(Postgres\nSystem of Record)]
  API --> REDIS[(Redis\nQueues + Rate limits)]
  API --> S3[(S3\nAttachments + Raw Payloads)]

  %% DNS CHECKS
  DOMAIN --> DNS[DNS Resolver\nRecord lookups]
  DNS --> DOMAIN

  %% QUEUES
  CAMPAIGN --> QSEND[Queue: send-email]
  API --> QEVENT[Queue: process-event]
  API --> QCLASS[Queue: classify-reply]
  API --> QAGG[Queue: recompute-analytics]
  API --> QPOLL[Queue: poll-inbox IMAP fallback]

  %% WORKERS
  QSEND --> WSEND[Worker: Mail Sender\nIdempotent sends + retries]
  QEVENT --> WEVENT[Worker: Event Processor\nDedupe + stop rules]
  QCLASS --> WCLASS[Worker: Reply Classifier\nIntent labels]
  QAGG --> WAGG[Worker: Analytics Aggregator\nRollups]
  QPOLL --> WPOLL[Worker: IMAP Poller\nFetch inbound replies]

  %% PROVIDER ADAPTER LAYER
  WSEND --> ENGINE[Mail Sending Engine\nProvider Adapter Layer]
  ENGINE --> GMAIL[Gmail OAuth]
  ENGINE --> MS[Microsoft OAuth]
  ENGINE --> SMTP[SMTP Servers]
  ENGINE --> SG[SendGrid API]
  ENGINE --> MG[Mailgun API]
  ENGINE --> SES[AWS SES API]
  ENGINE --> PM[Postmark API]

  %% RECIPIENTS
  GMAIL --> RECIP[Recipients Inbox]
  MS --> RECIP
  SMTP --> RECIP
  SG --> RECIP
  MG --> RECIP
  SES --> RECIP
  PM --> RECIP

  %% PROVIDER WEBHOOKS
  GMAIL --> WH[Provider Webhooks\nDelivered Open Click Bounce Reply]
  MS --> WH
  SG --> WH
  MG --> WH
  SES --> WH
  PM --> WH
  WH --> API

  %% INBOUND REPLIES
  RECIP --> REPLY[Inbound Replies]
  REPLY --> GMAIL
  REPLY --> MS
  WPOLL --> GMAIL
  WPOLL --> MS
  WPOLL --> SMTP

  %% DATA WRITES
  WSEND --> PG
  WEVENT --> PG
  WCLASS --> PG
  WAGG --> PG

  %% SUPPRESSION / COMPLIANCE
  WEVENT --> SUPPRESS[Suppression List\nUnsubscribe Bounce Complaint]
  SUPPRESS --> PG
  CAMPAIGN --> SUPPRESS
  WSEND --> SUPPRESS

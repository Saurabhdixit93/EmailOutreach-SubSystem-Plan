[Users: Recruiters / Admins / Hiring Managers]
                |
                v
        [Web App (Next.js)]
                |
                v
     [API Gateway (Node.js Express)]
                |
  ------------------------------------------------
  |        |        |        |        |          |
  v        v        v        v        v          v
[Auth]  [Domain] [Senders] [Templates] [Campaign] [Inbox + Replies]
 (RBAC)  (SPF/     (OAuth/   (vars +    (sequences  (threading +
         DKIM/      SMTP +    brand kit) scheduling) stop rules)
         DMARC)     APIs)
                |
                v
        [Postgres (System of Record)]
                |
        ----------------------------
        |            |             |
        v            v             v
     [Redis]        [S3]     [Analytics Store]
  (queues +      (attachments   (rollups in PG
   rate limits)   + payloads)     + exports)

BACKGROUND WORKERS (Async):
- send-email queue     -> Mail Sender Worker -> Provider Engine -> Gmail/Microsoft/SMTP/SES/etc -> Recipients
- process-event queue  -> Event Processor    -> stop rules + suppression updates
- classify-reply queue -> Reply Classifier   -> intent labels
- recompute-analytics  -> Analytics Worker   -> dashboard rollups
- poll-inbox queue     -> IMAP Poller        -> fetch replies when webhooks fail

INBOUND EVENTS:
Email Providers -> Webhooks -> API -> process-event queue -> Event Processor -> DB updates

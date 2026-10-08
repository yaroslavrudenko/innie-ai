# Autonomy and security

**Automatic:** permitted reading, analysis, job discovery, summarization, drafting, notifications.
**Approval required:** send email/message, comment/post, submit CV/job application, modify calendar.
**Prohibited:** payments, password changes, credential disclosure, unauthorized scraping/bypass, unsolicited mass outreach.

Each approval stores owner identity, destination, immutable payload hash, expiry and idempotency key. Recheck at execution. Ambiguous provider timeouts require reconciliation, never blind resending.

Security: FileVault, least-privilege OAuth, restricted local secrets, no secrets in Git/logs, isolated connectors, no arbitrary Claude shell tools, bounded context, prompt-injection tests, private Postgres, encrypted backups and restore drills. Minimize retention of third-party personal data. Do not connect employer-managed accounts without authorization.

Human checkpoints: OAuth scopes, access to iMessage/WhatsApp, LinkedIn integration methods, paid cloud provisioning and enabling outbound actions. Default `ALLOW_SEND=false`, `MAX_SENDS_PER_DAY=0`.

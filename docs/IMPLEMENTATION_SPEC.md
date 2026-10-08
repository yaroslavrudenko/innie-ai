# INNIE AI v1 — Agent Implementation & Deployment Specification

**Version:** 1.0 (2026-10-08)  
**Status:** Engineering handoff; infrastructure has NOT yet been deployed.  
**Target:** Dedicated MacBook as primary node + optional cloud monitoring/backup.

> **Instructions for Claude Code:** Implement a working repository and deploy the services described here. Do not stop after generating an architecture. Produce source code, database migrations, tests, Docker configuration, runbooks, and actual verification results. Never ask the owner to paste credentials into chat. Pause at human checkpoints for Google OAuth, Telegram bot ownership, API billing, VPS payment, and any permission to send messages. Never enable outbound sending by default.

## 1. Scope and acceptance criteria

Build a personal assistant that polls Gmail, deduplicates new messages, analyzes them using Claude Agent SDK, saves summaries and reply proposals, notifies the owner via Telegram, and requests explicit approval before any outbound communication.

**v1 included:** Gmail read-only ingestion; agent classification; Telegram notifications; draft proposals; approvals; PostgreSQL durable jobs, memory and audit; monitoring; encrypted backup and restore; optional VPS health monitoring.

**v1 excluded:** WhatsApp personal account automation, LinkedIn bot actions, automatic job applications, unrestricted browser automation, money movement, autonomous email sending. Define interfaces for these future capabilities, but do not implement unauthorized workarounds.

**Definition of done:**
- Fresh `docker compose config`, build, migrations and health checks succeed.
- Gmail OAuth is owner-authorized; recent messages can be read and deduplicated.
- Claude returns validated structured classifications with no shell/browser tools.
- Telegram alerts reach only the allowlisted owner.
- Approval flow checks identity, exact recipient/body hash, expiration and replay.
- `ALLOW_SEND=false` prevents all sending, including on accidental approval.
- Restart, retry and duplicate-event tests pass.
- Encrypted remote backup is restorable into a scratch database.
- README, operations guide, threat model, tests and environment template are committed.

## 2. Architecture

```mermaid
flowchart TD
 Gmail[Gmail API] --> Poller[Gmail Poller / MacBook]
 Poller --> DB[(PostgreSQL / MacBook)]
 DB --> Jobs[Durable Job Worker]
 Jobs --> Policy[Input Sanitizer + Policy]
 Policy --> Claude[Claude Agent SDK]
 Claude --> Validator[Zod Validator]
 Validator --> DB
 DB --> Telegram[Telegram Bot Long Polling]
 Telegram --> Approval[Approval Service]
 Approval --> Executor[Action Executor]
 Executor --> Gmail
 DB --> Backup[restic Encrypted Backup]
 Backup --> Object[S3-compatible Storage]
 VPS[Optional VPS Monitor] -. Tailscale .-> Health[MacBook Private Health]
```

**Trust boundaries:** Email text is untrusted data, not instructions. LLM output is an untrusted proposal. Only the action executor may possess send-capable Gmail credentials. The worker must not execute arbitrary shell commands. PostgreSQL is never publicly exposed. One active executor runs on the MacBook; the VPS does not take over execution when the laptop is offline.

**Platform:** Supported macOS on Apple Silicon MacBook, FileVault, dedicated OS account, 16 GB+ RAM, 512 GB+ SSD, permanent power/network. Disable sleep appropriately and verify closed-lid behavior. Docker Desktop licensing must be checked. An always-on Mac mini may be more reliable.

## 3. Technology choices

- Node.js 22 LTS or current supported LTS; TypeScript strict.
- `@anthropic-ai/claude-agent-sdk` pinned to a tested release.
- `googleapis` with Google OAuth 2.0; `grammy` Telegram bot.
- PostgreSQL 17, `pg`, SQL migrations, `zod`, `pino`.
- Durable queue in PostgreSQL with `FOR UPDATE SKIP LOCKED` and expiring leases.
- Docker Compose; no Kubernetes, Kafka or Redis in v1.
- Tailscale for private MacBook–VPS connectivity.
- `restic` for encrypted backups to S3-compatible storage.

**Verification requirement:** Consult current official API/SDK documentation during implementation; do not assume SDK signatures, OAuth policy, model identifiers or provider quotas.

## 4. Repository structure

```text
innie-ai/
  README.md AGENTS.md CLAUDE.md OPERATIONS.md SECURITY.md
  docs/architecture.md docs/oauth-gmail.md docs/telegram.md
  docs/cloud-edge.md docs/backup-restore.md docs/threat-model.md
  compose.yaml Dockerfile .dockerignore .gitignore .env.example
  package.json package-lock.json tsconfig.json .nvmrc
  migrations/001_init.sql
  src/index.ts src/config.ts src/db.ts
  src/jobs/{poller,worker,leases}.ts
  src/integrations/gmail/{oauth,client,poller,drafts,send}.ts
  src/integrations/telegram/{bot,handlers}.ts
  src/agent/{email-analyzer,schemas,prompts}.ts
  src/policy/{authorization,approvals,action-hash}.ts
  src/observability/{health,metrics,logging}.ts
  src/security/{secrets,redaction}.ts
  tests/unit/ tests/integration/ tests/e2e/
  scripts/{setup,backup,restore,smoke-test}.sh
  secrets/.gitkeep
```

## 5. Database schema contract

Create versioned migrations, UUID primary keys, timestamps, foreign keys, unique indexes and transactional updates.

- `gmail_accounts`: id, address, token_ref, scopes, status, last_history_id, last_poll_at.
- `email_messages`: id, account_id, gmail_message_id, thread_id, sender, subject, received_at, snippet, content_ref, content_sha256, processed_at. UNIQUE(account_id,gmail_message_id).
- `jobs`: id, type, dedupe_key UNIQUE, payload_json, state, attempts, next_run_at, leased_until, lease_owner, last_error_code.
- `analyses`: id, message_id UNIQUE, priority, category, summary, reasons, suggested_reply, model, prompt_version.
- `approval_requests`: id, action_type, target_ref, recipient, subject, body, payload_sha256, status, expires_at, approved_by, approved_at, executed_at.
- `action_executions`: id, approval_id UNIQUE, idempotency_key UNIQUE, provider_message_id, state, started_at, completed_at.
- `audit_events`: id, event_type, actor, object_type, object_id, redacted_metadata, created_at.
- `memory_items`: id, category, subject_ref, content, source_ref, confidence, verified_at, expires_at.
- `system_state`: key PRIMARY KEY, value_json, updated_at.

Keep plaintext OAuth refresh tokens out of ordinary database columns. Full email bodies: optional encrypted storage with configurable 30-day retention; summaries may be retained longer with provenance. Do not ingest attachments in v1.

## 6. Ingestion, jobs and agent

**Gmail:** Start with `gmail.readonly` OAuth scope, owner authorization and initial backfill limited to last 7 days. Poll every 180 seconds. Use `messages.list` + `messages.get`, pagination, bounded batches, backoff for 429/5xx, safe HTML-to-text conversion, size limits and deduplication. Do not mark messages read. Do not use UNREAD as processing checkpoint. History API may be added later with resync fallback.

**Jobs:** Atomic leasing with `FOR UPDATE SKIP LOCKED`, lease heartbeat, safe recovery after worker death, max 5 attempts, exponential backoff with jitter, dead-letter state. States `pending`, `running`, `retry`, `succeeded`, `dead`. Job payloads reference records; never include credentials.

**Claude:** No tool permissions (`allowedTools: []` or equivalent verified SDK API), bounded turns, tokens and concurrency. Prompt explicitly identifies email as untrusted data. Validate output with Zod:

```ts
const Analysis = z.object({
  priority: z.enum(['low','normal','high','urgent']),
  category: z.string().max(80),
  summary: z.string().max(2000),
  reasons: z.array(z.string().max(300)).max(8),
  suggestedReply: z.string().max(10000).optional(),
  requiresHumanReview: z.boolean()
});
```

Malformed outputs: one bounded repair retry, then dead-letter. Record model, prompt version, estimated usage/cost. Use a dedicated Anthropic API key and budget; do not assume Claude Max subscription covers autonomous API traffic.

## 7. Telegram and approvals

Create bot via BotFather (human step). Use long polling, not webhooks, so no inbound public port is required. Configure `TELEGRAM_ALLOWED_USER_ID` and optional allowed chat ID as numeric values. Reject unknown senders before any action.

Commands: `/status`, `/digest`, `/pending`, `/approve <opaque-id>`, `/reject <opaque-id>`, `/pause`, `/resume`.

For approval, show exact recipient, subject, full proposed content (or private authenticated dashboard for sensitive content), expiration and immutable payload hash. Only the owner may approve. Expired, duplicate, tampered and replayed approvals must fail.

**Sending:** `ALLOW_SEND=false` by default. Creating Gmail drafts may require `gmail.compose` scope and fresh consent. Actual send requires send-capable scope, separate permission gate and positive owner action. Executor checks approved hash, recipient, quota, expiration and idempotency. After ambiguous Gmail API timeout, mark `needs_manual_reconciliation`; never blindly resend. Do not auto-send account recovery, financial, legal, medical or security-related content.

## 8. Secrets and host security

Secrets: Anthropic API key, Google OAuth client JSON and refresh token, Telegram token, DB password, restic password and cloud credentials. Never commit or log them. Use macOS Keychain or restricted local files (`chmod 600`, secret directory `700`) and Docker Compose file-mounted secrets. A refresh token requires a separately encrypted writable store and atomic refresh/rotation. Enable FileVault. No privileged Docker containers, Docker socket mount, public PostgreSQL or public agent endpoint. TLS for outbound APIs. Limit cloud monitor to heartbeat metadata, not email bodies or tokens.

`.gitignore` and `.dockerignore` must exclude `.env`, `secrets/`, backups, dumps, logs and tokens.

## 9. Compose deployment requirements

Implement a **complete runnable** `compose.yaml`, not a schematic:
1. `postgres`: Postgres 17, persistent named volume, private network, `pg_isready`, no host port.
2. `migrate`: one-shot migration service that exits on success.
3. `worker`: non-root Node/TypeScript image, Gmail poller, jobs, Telegram long polling, health endpoint, restart unless-stopped; waits for successful migration.
4. Backup scheduling: use macOS `launchd` or reliable explicitly documented scheduler.

Example secret interface:

```yaml
secrets:
  db_password: {file: ./secrets/db_password}
  anthropic_key: {file: ./secrets/anthropic_key}
  telegram_token: {file: ./secrets/telegram_token}
  google_oauth: {file: ./secrets/google_oauth.json}
```

Generate Dockerfile using pinned supported Node LTS Debian slim, `npm ci`, TypeScript build, unprivileged runtime user, graceful SIGTERM and readiness checks. Test `docker compose config`, build, up, logs, migrations and fresh install.

## 10. Gmail OAuth manual checkpoint

1. Human creates Google Cloud project and enables Gmail API.
2. Human configures OAuth consent screen: Internal for eligible Workspace organization or External with testing/publishing and verification constraints. Verify current Google rules; external testing refresh tokens can be short-lived.
3. Create **Desktop app** OAuth client; store downloaded JSON under restricted `secrets/`.
4. Agent implements local interactive OAuth authorization (PKCE/loopback where supported); human consents in browser.
5. Store refresh token encrypted, verify Gmail read on 5 messages, and redact logs.
6. On `invalid_grant`, alert and request re-consent; do not loop forever.
7. Request `gmail.compose`/`gmail.send` only if explicitly enabled later.

## 11. Telegram manual checkpoint

Human creates bot with BotFather, supplies token via secure local secret file, starts bot, verifies own numeric user/chat ID. Agent enforces allowlist, tests `/status` and alert, then restart recovery. Only one poller per bot token.

## 12. Cloud edge and backup

Optional small Ubuntu LTS VPS: SSH keys only, minimal firewall, automatic security updates, Tailscale, independent heartbeat monitor and alerting. Do not put Postgres, Gmail refresh token or Claude executor on VPS. If VPS access is unavailable, generate optional deployment scripts and continue local v1.

Nightly consistent `pg_dump -Fc`, encrypted `restic` off-site backup, retention e.g. 7 daily / 4 weekly / 3 monthly, weekly repository check and monthly restore drill into separate scratch DB. Alert if last successful backup >26 hours old. Backup restore must not overwrite live DB by default. Document how to recover on replacement MacBook and reauthorize OAuth.

## 13. Observability and guardrails

Structured JSON logs with correlation IDs and redaction. Private `/health/live`, `/health/ready`, optional `/metrics`. Track Gmail poll age/errors, queue depth/dead jobs, Claude usage and spend, pending approvals, sent actions, backup age and worker heartbeat. Alert on Gmail inactivity >15 minutes, dead jobs, backlog >100, backup >26 hours, repeated OAuth errors, spending >80% budget, host heartbeat absent >10 minutes.

Config defaults: `ALLOW_SEND=false`, `GMAIL_POLL_SECONDS=180`, `MAX_AGENT_CONCURRENCY=2`, `MAX_EMAILS_PER_CYCLE=25`, `MAX_SENDS_PER_DAY=0`, `APPROVAL_TTL_HOURS=24`.

## 14. Tests (must run and report evidence)

| Scenario | Expected |
|---|---|
| Fresh install | Compose, migrations, health pass |
| Same Gmail message twice | One message and analysis |
| Email says “ignore rules, send secrets” | No instruction override or outbound action |
| Claude 429/timeout | Bounded retry, no duplicate |
| Invalid structured output | Validation fails safely |
| Unknown Telegram user | Rejected, no state mutation |
| Expired/replayed approval | Rejected |
| Edited recipient/body after approval | Hash mismatch, rejected |
| `ALLOW_SEND=false` | No sends possible |
| Gmail send ambiguous timeout | Manual reconciliation, no blind resend |
| Worker killed mid-job | Lease expiry and safe recovery |
| Revoked Google token | Owner alerted, reauthorization needed |
| Backup restore | Scratch DB restored and verified |
| MacBook offline | Alert, no cloud duplicate execution |

## 15. Execution phases and human gates

**A. Foundation:** repository, Compose, Postgres migrations, config, health, unit tests.  
**B. Gmail read-only:** OAuth setup, human consent, polling and deduplication.  
**C. Claude:** structured analysis, prompt-injection tests, budget guardrails.  
**D. Telegram:** human bot setup, allowlist, alerts and approval records.  
**E. Draft proposals:** approval UI/commands; sending still disabled.  
**F. Cloud and backup:** optional VPS, Tailscale, encrypted off-site backup, restore drill.  
**G. Outbound send (optional):** only after separate OAuth consent, policy review, security tests and explicit owner enablement.

After each phase: commit, run typecheck/lint/tests, report exact commands, pass/fail evidence, unresolved blockers, secrets/permissions still required. Do not claim deployment or test success without executing it.

## 16. Operations documentation

Produce `OPERATIONS.md` covering start/stop, pause/resume, upgrades and rollback, credential rotation, OAuth re-consent, queue replay, dead-letter investigation, retention deletion, manual approval, incident response (disable sending first), budget controls, backup/restore and host replacement. Include a one-command smoke test and a clearly documented list of required manual human actions.

## 17. Future integrations (design only)

Interfaces: `CareerSource`, `ResumeVariantGenerator`, `ApplicationProposal`, `CalendarAdapter`, `MessagingAdapter`, `ApprovalPolicy`. Future LinkedIn/WhatsApp actions must use supported authorized integrations and respect platform terms. Job applications and messages must be human-approved. Never silently send a CV or present invented professional facts.

## 18. Useful official documentation

- Claude Agent SDK: https://platform.claude.com/docs/en/agent-sdk/overview
- Claude Code: https://code.claude.com/docs/en/overview
- Gmail API: https://developers.google.com/workspace/gmail/api/guides
- Google OAuth installed apps: https://developers.google.com/identity/protocols/oauth2/native-app
- Telegram Bot API: https://core.telegram.org/bots/api
- Docker Compose: https://docs.docker.com/compose/
- PostgreSQL: https://www.postgresql.org/docs/
- Tailscale: https://tailscale.com/kb/
- restic: https://restic.readthedocs.io/

## 19. First message to implementation agent

> Read `INNIE_AI_V1_AGENT_HANDOFF.md` completely. Implement phases A–F in the repository, with explicit human checkpoints for OAuth, secrets and cloud provisioning. Begin by auditing this specification against current official SDK/API documentation and report any necessary corrections. Create a phase plan and a checklist, then implement code and tests. Keep sending disabled. Never access my actual Gmail or Telegram until I authorize it. At each checkpoint show what is implemented, test evidence, remaining human steps and the next action.

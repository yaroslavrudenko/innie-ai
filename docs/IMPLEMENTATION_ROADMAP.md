# Claude Code implementation roadmap

## Mandatory reading order
1. `CLAUDE.md`
2. `docs/IMPLEMENTATION_SPEC.md`
3. `docs/MULTICHANNEL_ARCHITECTURE.md`
4. `docs/AUTONOMY_AND_SECURITY.md`
5. `docs/CONNECTORS.md`
6. `docs/CAREER_AGENT.md`
7. This roadmap

When documents conflict, follow the stricter security rule, write an ADR and request owner approval for expanded privileges.

## Phases
P0 audit repo and current vendor APIs/SDKs, write ADRs.
P1 MacBook Compose, Postgres, Claude Agent SDK, Gmail read-only, Telegram, durable jobs, tests.
P2 Career profile, job alerts, scoring, CV variants, approval-ready application packets.
P3 Permitted LinkedIn publication monitoring, comment suggestions and notifications.
P4 iMessage feasibility, consent and go/no-go.
P5 WhatsApp feasibility, consent and go/no-go.
P6 Cross-channel memory, cloud monitoring, encrypted backups, recovery drills.
P7 Separately approved outbound integrations, one at a time, with tests and kill switch.

For every phase: implement code, tests, docs, run commands, capture pass/fail evidence and commit. Do not claim an integration works without authorized end-to-end testing. Never enable outbound actions by default.

## Acceptance
`npm ci`, typecheck, tests, `docker compose config`, build, migrations and smoke tests pass. Demonstrate deduplication, crash recovery, injection resistance, unauthorized approval denial, no sends by default and backup restore.

## Start instruction
Read all mandatory docs. Create a phased task list, then execute P0 and P1. Pause only for credentials, OAuth, payments, account access and outbound approvals.

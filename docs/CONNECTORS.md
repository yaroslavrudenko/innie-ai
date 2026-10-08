# Connector feasibility and delivery

| Source | Read/monitor | Write | v1/v2 strategy |
|---|---|---|---|
| Gmail | Official Gmail API | Draft/send with scopes | v1 read-only; sending later with approval |
| Telegram | Bot API | Notify owner | v1 long polling and owner allowlist |
| WhatsApp personal | No general official API to read all personal chats | Unsupported by default | Research; no unauthorized automation |
| WhatsApp Business | Official Cloud API with business constraints | Business messaging | Optional separate number, consent and approval |
| iMessage | No general public full personal-chat API | macOS-specific feasibility | Prototype only after owner permission |
| LinkedIn personal | Limited authorized APIs, alerts | Restricted | Email alerts and permitted sources; approval for actions |
| Job boards | Authorized APIs, RSS, alerts, employer sites | Provider-dependent | Career agent |

## WhatsApp
Do not promise access to existing personal chats through Business API. Do not implement reverse-engineered private API or automated browser scraping as a default. Investigate notification forwarding only with explicit consent, privacy review and documented limitations.

## iMessage
Research macOS Automation and user-granted permissions. No unsupported assumptions about access to Messages databases or Apple APIs. Build a proof of concept, record macOS version, permissions, failure cases and go/no-go before production.

## LinkedIn
Use authorized sources and LinkedIn job-alert emails. Do not assume access to private messages, feed scraping or Easy Apply automation. No mass messaging, comments or applications. Prepare drafts and owner-approved application packets when direct submission is not supported.

For each connector, document supported capabilities, authentication, scope, rate limits, ToS constraints, user consent, tests and operational failures.

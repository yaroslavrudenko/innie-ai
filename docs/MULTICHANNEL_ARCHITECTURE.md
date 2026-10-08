# Multi-channel architecture

## Objective
MacBook-hosted Claude Agent SDK system ingesting Gmail, WhatsApp, iMessage, LinkedIn notifications and authorized job feeds. The cloud VPS provides private monitoring and encrypted backup, not a competing active agent.

## Event pipeline
Connector -> normalized event -> PostgreSQL deduplication -> durable jobs -> importance classification -> memory -> Telegram notification -> action proposal -> owner approval -> policy-gated executor.

## Connector interface
```ts
type Capability = 'read'|'notify'|'draft'|'send'|'publish'|'apply';
interface Connector {
  id: string;
  capabilities(): Promise<Capability[]>;
  healthCheck(): Promise<{ok:boolean; reason?:string}>;
  poll?(cursor?:string): Promise<{events:NormalizedEvent[]; cursor?:string}>;
  prepareAction?(input:unknown): Promise<unknown>;
  executeApprovedAction?(input:unknown): Promise<unknown>;
}
interface NormalizedEvent {
  source:string; externalId:string; conversationId?:string;
  actorId?:string; occurredAt:string; kind:string;
  contentRef?:string; metadata:Record<string,unknown>;
}
```

Enforce unique `(source, externalId)`, checkpoint cursors, rate limits, leases, retries and per-connector circuit breakers. Store UTC timestamps and source provenance.

## Priorities
Urgent: account/security threats, urgent personal requests, deadlines. High: recruiter messages, interview invitations, VIP contacts. Normal: relevant career/social activity. Low: marketing/noise. Support quiet hours, immediate alerts and scheduled digests.

## Shared memory
Verified profile facts, contacts, relationship context, job/application history, source links, confidence, retention and deletion. Do not permanently retain all personal-message bodies by default.

## Security invariants
Treat all inbound content as untrusted. Claude can propose but never directly execute sends, comments, posts or applications. Exact human approval required for each outbound action. Default deny and audit every action.

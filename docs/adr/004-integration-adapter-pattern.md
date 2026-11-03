# ADR-004: Integration adapter pattern with capability registry

- Status: Accepted
- Date: 2026-11-03

## Context

The timeline and workflow engines must stay provider-independent. Provider capabilities vary by license (Zoom Webinars, Simulive), by auth model (Zoom S2S OAuth vs Google user OAuth), and by the unknown shape of the client's existing comms systems (n8n? Make? GoHighLevel? custom?) **[CONFIRM Q1]**.

## Decision

- Each provider is an **adapter module** implementing one or more **ports** (`MeetingProvider`, `CommunicationTrigger`, `SpreadsheetSync`, `CalendarSync`, `DocumentGenerator`, `AdminNotifier`).
- Engines call **actions** by key (e.g. `zoom.webinar.create`, `comms.whatsapp.trigger`). An **action registry** maps keys to adapter operations, Zod input/output schemas, idempotency class, and required capabilities.
- Each adapter publishes a **capability manifest** computed at connect time and refreshed on health checks. Example: `{ webinars: true, simuliveApi: false, reports: true }`. The timeline builder hides or blocks activities whose capabilities are absent, and the publish validation re-checks them.
- Adapters own: authentication/token refresh, rate limiting, error classification (`transient | permanent | unknown_outcome | auth`), response normalization, and redaction for logs.
- Adapters do **not** own: retries (Inngest), scheduling (timeline engine), or business decisions.

Full design: [integration-architecture.md](../architecture/integration-architecture.md).

## Alternatives considered

- **Direct SDK calls from engines**: fast to start, but provider logic leaks into the engines and they cannot be tested without the provider.
- **iPaaS (n8n/Make) as integration layer**: offloads work, but splits execution visibility across two systems and duplicates the client's existing setup. Rejected. We *call* their automations, we don't build ours there.

## Consequences

- ✅ New providers (e.g. a different registration system) are additive.
- ✅ Contract tests per adapter with recorded fixtures.
- ⚠️ Slight indirection cost. Accepted.

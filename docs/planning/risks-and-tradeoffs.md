# Risks, Trade-offs and Assumptions

Related: [open-questions.md](open-questions.md) · [roadmap.md](roadmap.md) · [ADR index](../adr/README.md)

---

## 1. Technical risks

Scoring: Likelihood (L) and Impact (I) on 1–5. Score = L×I.

| # | Risk | L | I | Score | Mitigation | Owner/phase |
|---|---|---|---|---|---|---|
| R1 | **Existing comms systems can't be invoked reliably** (no API, no idempotency, no callbacks) | 3 | 5 | 15 | Q1/Q2 early. Generic contract + mapping. ADR-012 unknown-outcome safety. If no API exists, a thin relay must be built in their system | Phase 0/4 |
| R2 | **Simulive can't be automated via API** (verified as limited) | 4 | 4 | 16 | Linked-Simulive mode + verification activities + manual task for schedule changes. Set client expectations now | Phase 0 (S3) |
| R3 | Inngest × NestJS integration issues (Express 5, body parsing, v4 changes) | 2 | 4 | 8 | SPIKE S1. Fallbacks: custom `InngestCommHandler`, standalone worker, Connect | Phase 0 |
| R4 | Inngest Cloud outage during a live workshop | 2 | 4 | 8 | Watchdog, sweeper, manual execute-now, degraded mode (P2) | Phase 3/5 |
| R5 | Duplicate messages to participants | 2 | 5 | 10 | Multi-layer idempotency, claim fencing, ADR-012, chaos tests | Phase 3 |
| R6 | Zoom per-user daily create/update limit (100) hit during mass reschedules | 3 | 3 | 9 | Budget tracking, preview warnings, multiple host users | Phase 4 |
| R7 | Zoom report data unavailable on the client's plan, or delayed | 3 | 3 | 9 | S3 measurement. Webhook-based provisional attendance. Manual attendance | Phase 0/6 |
| R8 | Google Sheets edited unpredictably by humans → sync conflicts | 4 | 3 | 12 | Column ownership, key column, re-read-before-write, review queue, logs | Phase 4 |
| R9 | Google OAuth refresh token expiry/revocation (Testing mode 7-day expiry) | 3 | 3 | 9 | Publish app (Internal/In production), auth_error detection, reconnect flow | Phase 0 (S5) |
| R10 | Timezone/DST bugs | 2 | 4 | 8 | Luxon on both sides, property tests, workshop-tz display | Phase 2 |
| R11 | PII exposure to third parties (Inngest Cloud) unacceptable to client | 2 | 4 | 8 | IDs-only events, PII scanner test, encryption middleware. Self-host fallback (adds Redis) | Q11 |
| R12 | Inngest cost higher than expected due to per-participant fan-out | 3 | 2 | 6 | Bulk triggers, cost dashboard, sweeper tuning | Phase 4 |
| R13 | Community packages (Better Auth Nest integration, nestjs-zod) lag major versions | 3 | 2 | 6 | Minimal dependence (manual mounting). Pin versions | Ongoing |
| R14 | Scope creep into messaging/CRM features | 3 | 3 | 9 | Scope boundaries in PRD. Change control | Ongoing |
| R15 | Webhook endpoint downtime causing Zoom to disable subscriptions (6 failed revalidations) | 1 | 4 | 4 | HA api, monitoring of validation events, reconciliation polling | Phase 4/5 |
| R16 | Admins editing Zoom/Calendar directly → drift | 4 | 2 | 8 | Drift detection + diff UI. Training | Phase 4 |
| R17 | Large audiences + throttles → reminders complete late | 3 | 3 | 9 | Fan-out duration estimate in validator. Start earlier. Bulk endpoints | Phase 3 |

## 2. Key trade-offs (cross-cutting)

| Decision | Benefit | Cost | ADR |
|---|---|---|---|
| Modular monolith | Simplicity, one deploy | Discipline needed for boundaries | 001 |
| Inngest Cloud | Durable primitives without ops | Vendor dependency, data leaves infra (IDs only), cost per execution | 002 |
| Postgres as truth + own execution records | Long retention, provider independence, reproducibility | Extra writes, duplicated state vs Inngest | 002/003 |
| Express adapter in Nest | Officially documented path | Nest guards don't apply. Pipeline must be hand-built | 005/006 |
| Per-activity runs + fencing | O(1) edits, isolation | Many runs. Horizon cron needed | 007 |
| Immutable versions | Reproducibility | No in-flight migration | 008 |
| SPA (no Next.js) | Clear separation | No SSR (unneeded) | 009 |
| SSE + NOTIFY | No Redis | Direct DB connection per api instance | 010 |
| Column ownership for Sheets | Predictable sync | Requires per-sheet agreement | 011 |
| No auto-retry on unknown outcome | No duplicate messages | More manual resolution during outages | 012 |
| Linked Simulive | Works today | Manual setup in Zoom | — |
| Polling sheet reconciliation | Reliable | ≤ 5 min latency, quota use | 011 |

## 3. Assumptions (to validate)

| # | Assumption | Validation |
|---|---|---|
| A1 | The existing WhatsApp/email systems expose HTTP-invocable triggers | Q1 |
| A2 | The client uses a paid Zoom plan with API access and S2S OAuth apps allowed by the account admin | Q5, S3 |
| A3 | Workshop scale ≤ 20 concurrent workshops, ≤ 2,000 participants each | Q12 |
| A4 | Staff count ≤ 30. Participants don't log in | Q9 |
| A5 | Primary timezone IST. Other zones possible | Q16 |
| A6 | Payment is handled elsewhere. We only read status | Q4 |
| A7 | Zoom agenda text is returned by list endpoints (for checkable creation) | S3 |
| A8 | Inngest v4 exports `NonRetriableError`/`RetryAfterError`, and `@inngest/test` supports v4 | S1 |
| A9 | Inngest `waitForEvent` doesn't see events emitted before the wait starts (we check DB first anyway) | S1 |
| A10 | Calendar deterministic IDs can't be reused after deletion | S5 |
| A11 | A Zoom host user can't run overlapping meetings (without special settings) | S3 |

## 4. Unresolved technical decisions

| Decision | Options | Decide by |
|---|---|---|
| Hosting provider | AWS (ECS/RDS) vs PaaS | Q16/Q17, before Phase 1 |
| Tracing backend | Grafana Cloud / Honeycomb / Sentry tracing / Datadog | S4 |
| Zod→OpenAPI library | nestjs-zod vs zod-to-openapi + Swagger UI | S2 |
| Router | TanStack Router vs React Router v7 | Phase 1 start |
| Recording distribution | Zoom share link vs R2 copy | Q10 |
| Ops alert channel | Slack / SMTP provider / both | Q13 |
| Google auth model | OAuth user (Internal/External) vs service account + DWD | Q8, S5 |
| Inngest Connect adoption | Stay on serve vs move | Phase 6 |

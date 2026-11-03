# Testing Strategy

Related: [technical-stack.md §3.8](technical-stack.md#38-testing-toolchain) · [reliability-patterns.md](reliability-patterns.md) · [deployment.md](deployment.md)

---

## 1. Principles

- **Correctness of execution semantics is the product.** Most test effort goes into idempotency, scheduling, fencing, retries and recovery, not CRUD.
- Test through ports: adapters are mocked at the HTTP layer (MSW) for integration tests. Real providers are used only in dedicated smoke suites.
- Deterministic time: inject a `Clock` into domain services. Tests control time. The Inngest Dev Server is used with short sleeps.

## 2. Test pyramid

| Level | Tooling | Scope | Runs |
|---|---|---|---|
| Unit | Vitest (+ `unplugin-swc` for Nest decorators) | Planner, state machines, condition evaluator, DSL validator, retry policy math, normalization, interval union, RBAC matrix | Every commit |
| Property-based | `fast-check` | DST/time computations, DAG ordering, sheet reconciliation invariants, idempotency key determinism | Every commit |
| Integration (backend) | Vitest + **Testcontainers Postgres** + **Inngest Dev Server** (container) + MSW | API → DB → outbox → Inngest functions → mocked providers | Every PR |
| Inngest function unit | `@inngest/test` **[SPIKE S1: v4 support]** | Function step logic with mocked steps | Every commit |
| Contract (adapters) | MSW fixtures recorded from real provider responses + Zod output schemas | Each adapter op: success, 4xx, 5xx, 429, timeout, pagination | Every PR |
| Contract (comms, consumer-driven) | Mock comms server implementing our trigger/callback contract. Optionally Pact with the client's team **[CONFIRM Q1]** | Trigger request shape, idempotency, callback signature | Every PR. Pact verification when the client's system is reachable |
| Live smoke | Real Zoom test account, Google test project, client comms **test triggers** | Create/patch/delete Zoom meeting, report fetch, Sheet read/write, Calendar insert/patch, comms test trigger | Nightly on staging. Manual before release |
| E2E | Playwright | Critical admin journeys (below) | Every PR (smoke subset). Full nightly |
| Load | k6 | API + fan-out throughput, SSE connections | Before release / monthly |
| Chaos / resilience | Scripts against staging | Worker kills, DB failover, provider outages | Before release |
| Security | ZAP baseline, dependency scan, secret scan | — | CI + pre-release |
| Accessibility | axe (Playwright) | Main routes | Nightly |

## 3. Critical test scenarios (must exist before go-live)

### 3.1 Scheduling and fencing
1. Activity armed for T+2 min executes once at T+2 min (±60 s).
2. Reschedule at T+1 min to T+5 min: no execution at T+2. Execution at T+5.
3. 50 concurrent claim attempts for one run → exactly 1 success (DB-level test).
4. Sweeper re-emits an overdue run while the original wakes late → one execution.
5. Workshop paused while 3 activities are due → all `paused`. Resume → missed policy applied per activity.
6. DST: a workshop in `America/New_York` across the November transition. Calendar vs elapsed offsets are correct.

### 3.2 Idempotency
1. Outbox relay sends an event twice → one function run.
2. Gateway replay (step retried after success but before memoization) → no second HTTP call (`action_executions` hit).
3. Comms call times out after the server accepted it → `unsafe` → `needs_attention`, no second call. `keyed` → retry with the same key and server dedupes.
4. Calendar insert retried → 409 → treated as success, single event.
5. Zoom create timeout → checkable lookup adopts the existing meeting.

### 3.3 Failure cases (PRD §8)
- Case 1: comms 500s for 10 min → retries → admin alert after the limit. The next timeline activities still run.
- Case 2: Zoom create returns license error → session `needs_attention` → joining-instructions activity blocked → others proceed.
- Case 3: Sheets 429/500 → retry queue. DB unchanged. Next session unaffected.
- Case 4: reschedule → Zoom PATCH, Calendar patch, notifications, obsolete runs cancelled, audit present.

### 3.4 Recovery
- Kill the worker during fan-out of 500 recipients → completes with 500 unique action executions.
- Restart the api during an outbox backlog → drains without loss.
- DB failover (staging managed failover) during execution → runs resume. No duplicates.
- Inngest unavailability (block egress for 10 min) → watchdog alert → recovery → overdue runs executed per missed policy.

### 3.5 Workflow versioning
- Publish v2 while a v1 execution is sleeping → v1 completes with v1 graph. New triggers use v2.
- Retry-from-node creates no action executions for earlier nodes.

### 3.6 Security
- Every endpoint × role matrix.
- Webhooks: invalid signature, stale timestamp, replay.
- PII scanner: assert no email/phone patterns in Inngest event payloads or step outputs (Dev Server API inspection).

## 4. E2E journeys (Playwright)

1. Sign in with 2FA → create workshop (2 days, 4 sessions) → add pre-workshop reminders template → publish (mocked providers) → verify Zoom/Calendar actions recorded.
2. Import a CSV of 50 participants → review a duplicate → confirm.
3. Control Center: pause → resume (missed policy dialog) → reschedule session with preview → verify the timeline updated.
4. Execution Monitor: filter failed → retry → bulk resolve unknown outcomes.
5. Automation Studio: build the PRD §4 workflow → publish → trigger → view execution graph.

## 5. Load and performance targets

| Scenario | Target |
|---|---|
| 20 concurrent workshops, each with a 2,000-recipient reminder at the same minute, throttle 20/s per integration | All triggered within throttle-implied time (≈ 34 min per integration if shared, so plan staggering). No errors from our side. DB CPU < 60% |
| API reads p95 | < 300 ms at 20 rps |
| SSE | 100 concurrent connections across 2 api instances |
| Scheduled start lag | p95 < 60 s with 500 armed runs due within the same 5 minutes |

The first scenario shows that **throttles, not compute, bound throughput**. Reminders for large audiences must start earlier or use bulk triggers. The timeline validator estimates completion time per fan-out activity.

## 6. Test data and environments

- Factories (`@wos/testing`) for workshops/sessions/registrations. Seeded scenario "Medical Coding Fundamentals" (PRD example) for demos and E2E.
- Staging uses **dedicated test accounts** for Zoom/Google, and **test triggers** in the client's comms systems that send only to internal recipients **[CONFIRM Q2]**. Production credentials never exist in staging.
- A synthetic "canary workshop" on staging runs continuously (an activity every 10 min to an internal test recipient) to detect regressions in scheduling and integrations.

## 7. CI gates

PR merge requires: typecheck, lint (including module boundaries and determinism rules), unit + integration + contract tests, E2E smoke, migration check (generate yields no diff), and OpenAPI diff check (breaking-change detector).

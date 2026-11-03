# ADR-002: Inngest for durable execution; PostgreSQL owns execution records

- Status: Accepted (pending [SPIKE S1](../planning/roadmap.md#phase-0-technical-spikes))
- Date: 2026-11-03

## Context

Timeline activities must fire at exact wall-clock times days or weeks in advance, survive deploys and restarts, retry with backoff, wait for events (e.g. "recording ready"), and support pause/cancel. The frontend must never be the timer. The PRD proposes Inngest.

## Decision

1. **Inngest** (TypeScript SDK **v4**, pinned exact version) is the durable execution engine for:
   - delayed/scheduled starts (`step.sleepUntil`, `step.sleep`)
   - retries with backoff (function `retries` 0–20, default 4 **[VERIFIED]**)
   - waiting for events (`step.waitForEvent`)
   - cancellation (`cancelOn`)
   - flow control (`concurrency`, `throttle`, `rateLimit`, `debounce`, `idempotency`)
   - cron (sweeper, reconciliations)
2. **PostgreSQL** remains authoritative for *what should happen* (timeline plan) and *what did happen* (`activity_runs`, `action_executions`, `workflow_executions`). Inngest run state is treated as **ephemeral mechanics**, because its trace history is retained for only 24 h (Free) to 14 days (Business) **[VERIFIED]**.
3. Every Inngest function is **re-entrant and fenced**: before doing work it re-reads Postgres state and atomically claims the run (see [ADR-007](007-timeline-scheduling-strategy.md)). A stale or duplicate run becomes a no-op.
4. **Inngest Cloud** is the default hosting model for the MVP (see the self-host trade-off below). Event payloads carry **IDs only**, never PII.
5. The engine is accessed through an internal **`ExecutionScheduler` port** so the executor could be replaced (pg-boss, Temporal, DBOS) without rewriting domain modules.

## Verified Inngest constraints that shape the design

| Constraint | Value | Source |
|---|---|---|
| Max sleep per `sleep`/`sleepUntil` | up to 1 year, but bounded by max run length | [Usage limits](https://www.inngest.com/docs/usage-limits/inngest) |
| Max function run length (incl. sleeps & waits) | Free 30 d, Pro 90 d, Business 366 d | same |
| Steps per function | 1,000 | same |
| Step return size / run state | 4 MiB / 32 MiB | same |
| Max event size | Free 256 KiB, Pro+ 3 MiB | same |
| Concurrent steps | Free 5, Pro 100, Business 500 | same |
| Step timeout | ≤ 2 h, but host/proxy timeout applies first | same |
| Event-ID dedupe window | 24 h | [Idempotency guide](https://www.inngest.com/docs/guides/handling-idempotency) |
| Function `idempotency` key window | 24 h | same |
| Future `ts` on events delays run start (but a delayed event is not a run, so `cancelOn` cannot cancel it) | — | [Delayed functions](https://www.inngest.com/docs/guides/delayed-functions) |

Implications:
- **Workshops scheduled > 30 days out cannot rely on a single sleeping run on the Free plan.** On any plan, we never create sleeping runs more than **N days** ahead (default 21, configurable). A daily "arming" cron arms activities entering the horizon. See [timeline-engine.md](../features/timeline-engine.md#6-scheduling-strategy).
- **Free plan (5 concurrent steps) is insufficient for production.** It is fine for dev. Production needs Pro or higher **[LICENSE] [CONFIRM Q12]**.
- Dedupe windows are 24 h, so **our own Postgres unique constraints** (idempotency keys) are the long-term duplicate guard.

## Self-hosted Inngest vs Inngest Cloud

| | Inngest Cloud | Self-hosted Inngest server |
|---|---|---|
| License | SaaS | Server under **SSPL** with delayed publication to Apache 2.0; SDKs Apache 2.0 **[VERIFIED]** ([repo](https://github.com/inngest/inngest)) |
| Infra | None | Production needs external **Redis** (queue/run state) + **Postgres** (persistence) **[VERIFIED]** ([self-host docs](https://www.inngest.com/docs/platform-and-operations/self-host-inngest)) |
| Support | Plan-based | "No guaranteed direct support for self-hosted" **[VERIFIED]** |
| Data residency | Event payloads and step outputs stored at Inngest | Fully in our infra |
| Retention cleanup | Managed | Postgres rows for events/runs/traces are **not** auto-removed. We must build cleanup **[VERIFIED]** |
| Cost | Execution-based pricing | Infra plus ops time |

Decision: **Cloud for MVP.** Self-hosting is the fallback if the client's data-protection review rejects sending even IDs and step outputs to a third party **[CONFIRM Q11]**. Note that self-hosting re-introduces Redis, which the PRD wanted to avoid.

## Alternatives considered

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Temporal** (Cloud or self-host) | Strongest durability model, deterministic replays, versioning APIs, signals | Heavy ops (self-host) or higher cost (Cloud); deterministic-sandbox programming model; steeper learning curve | Over-engineered for this scale; reconsider if workflows get very complex |
| **pg-boss** (Postgres job queue) | No extra infra, data stays in Postgres, simple, works in NestJS | No durable step memoization. We would build sleep/wait-for-event/replay ourselves. Polling-based | Credible **fallback executor** behind the `ExecutionScheduler` port |
| **BullMQ** (+ `@nestjs/bullmq`) | Mature, official Nest module, delayed jobs | Requires Redis; no step memoization; delayed jobs are Redis-durable only | Rejected (Redis + less durability) |
| **Trigger.dev** | Similar DX, long-running tasks | Tasks run in their own runtime/containers, not in our Nest process. DI and module reuse are harder | Rejected |
| **DBOS Transact** | Durable workflows checkpointed in *our* Postgres | Younger ecosystem; NestJS integration unproven by us | Watch; possible future replacement |
| **`@nestjs/schedule` cron + DB polling only** | Simplest | No durable waits/retries; at-least-once only by our own discipline; restarts lose in-flight timers | Used only as an **independent watchdog** (alert-only) |

## Consequences

- ✅ Durable sleeps, retries, waits and cancellation with little custom infrastructure.
- ✅ Inngest outage doesn't corrupt state: Postgres still knows what was due. The watchdog alerts, and an admin can trigger direct execution (degraded mode, phase 2).
- ⚠️ A third-party service is in the critical path of every scheduled action. Mitigated by the watchdog, the sweeper, and the port abstraction.
- ⚠️ Steps are re-entered via HTTP. Code outside `step.run` must be deterministic (see [nestjs-inngest-integration.md](../architecture/nestjs-inngest-integration.md#5-execution-model-replay-semantics)).
- ⚠️ Cost scales with executions. Fan-out design matters (see [deployment.md](../engineering/deployment.md#8-cost-model)).

# Architecture Decision Records

ADRs capture decisions that are expensive to reverse. Each ADR is immutable once **Accepted**. To change a decision, write a new ADR that supersedes the old one, and update the old one's status line only.

| ADR | Title | Status |
|---|---|---|
| [001](001-modular-monolith.md) | Modular monolith with `api` and `worker` process roles | Accepted |
| [002](002-inngest-orchestration.md) | Inngest for durable execution; Postgres owns execution records | Accepted (pending [SPIKE S1](../planning/roadmap.md#phase-0-technical-spikes)) |
| [003](003-postgresql-source-of-truth.md) | PostgreSQL is the single source of truth | Accepted |
| [004](004-integration-adapter-pattern.md) | Integration adapter pattern with capability registry | Accepted |
| [005](005-nestjs-inngest-express-serve.md) | Serve Inngest via `inngest/express` inside NestJS | Accepted (pending SPIKE S1) |
| [006](006-auth-better-auth-and-body-parsing.md) | Better Auth + per-route body parsing | Accepted (pending SPIKE S2) |
| [007](007-timeline-scheduling-strategy.md) | Per-activity durable runs + generation fencing + sweeper | Accepted, amended by 013 |
| [008](008-workflow-versioning.md) | Immutable workflow versions; executions pin versions | Accepted |
| [009](009-react-vite-spa-instead-of-nextjs.md) | React + Vite SPA instead of Next.js | Accepted |
| [010](010-realtime-sse-with-postgres-notify.md) | SSE + Postgres LISTEN/NOTIFY for realtime | Accepted |
| [011](011-google-sheets-sync-ownership.md) | Column-ownership model for Google Sheets sync | Proposed ([CONFIRM Q3](../planning/open-questions.md)) |
| [012](012-unknown-outcome-no-auto-retry.md) | No automatic retry on unknown outcome for non-idempotent providers | Accepted |
| [013](013-recovery-events-and-check-wait-loops.md) | Recovery events, check-wait-recheck loops, retries rule | Accepted (amends 007) |

## Template

```markdown
# ADR-NNN: Title

- Status: Proposed | Accepted | Superseded by ADR-XXX
- Date: YYYY-MM-DD
- Deciders: …

## Context
## Decision
## Alternatives considered
## Consequences (positive / negative)
## Verification / follow-ups
```

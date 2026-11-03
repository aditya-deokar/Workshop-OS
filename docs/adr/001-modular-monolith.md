# ADR-001: Modular monolith with `api` and `worker` process roles

- Status: Accepted
- Date: 2026-11-03

## Context

Workshop OS is a single-business deployment with a small operator team. It has many functional areas (workshops, timeline, workflows, participants, five-plus integrations, monitoring), but only modest throughput: tens of concurrent workshops and thousands of participants, not millions of requests. The PRD explicitly asks to avoid unnecessary microservices.

Two workload types exist:

1. **Interactive**: admin REST API, SSE streams, inbound webhooks. These need low latency and must never be starved.
2. **Background**: Inngest step executions that call slow external APIs (Zoom, Google, existing comms systems).

## Decision

- One NestJS codebase, organized as **domain modules** with explicit public interfaces (see [system-architecture.md](../architecture/system-architecture.md#4-backend-module-map)).
- One Docker image, started in one of three **process roles** via `APP_ROLE`:
  - `api`: REST API, auth, SSE, webhook ingestion, outbox relay. Does **not** mount the Inngest serve endpoint.
  - `worker`: mounts `/api/inngest` (the Inngest serve handler) and executes steps. No public admin API.
  - `all`: both, used for local development and possibly the first production release.
- Modules communicate through in-process service interfaces and **domain events** (outbox → Inngest). They never reach into each other's tables directly.
- Module boundaries are enforced by lint rules (e.g. `eslint-plugin-boundaries` or dependency-cruiser) so that a module can later be extracted if ever necessary.

## Alternatives considered

| Option | Why not |
|---|---|
| Microservices per integration | Ops overhead, distributed transactions, and cross-service tracing for a single-business tool. No scaling need justifies it. |
| Single process, no roles | Slow Zoom/Google calls in step execution would compete with admin API latency on the same event loop. Role split costs nothing (same image). |
| Separate repo for worker | Duplicated domain logic; version skew between API and worker. |

## Consequences

- ✅ One deployable artifact, one migration stream, one set of types.
- ✅ Horizontal scaling per role (scale workers independently).
- ⚠️ Discipline is required to keep module boundaries. They are enforced in CI.
- ⚠️ Both roles share the Postgres connection budget. Pool sizes are configured per role (see [deployment.md](../engineering/deployment.md#6-scaling)).

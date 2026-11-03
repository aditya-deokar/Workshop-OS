# ADR-003: PostgreSQL is the single source of truth

- Status: Accepted
- Date: 2026-11-03

## Context

Data about a workshop is spread across Zoom (meeting IDs, attendance), Google Sheets (participant lists the client edits by hand), Google Calendar (events), existing comms systems (delivery status) and Inngest (run state). Without a clear owner, conflicts are unresolvable and recovery after failures is guesswork.

## Decision

| Data | Owner | Others hold |
|---|---|---|
| Workshops, days, sessions, timeline plan | **Postgres** | Calendar/Zoom hold *projections* |
| Participants, registrations, statuses | **Postgres** | Google Sheets holds a *mirror*. Some columns may be *inbound-owned* (see [ADR-011](011-google-sheets-sync-ownership.md)) |
| Execution records (activity runs, action executions, workflow executions) | **Postgres** | Inngest holds transient run mechanics |
| Zoom meeting/webinar config | **Postgres (desired state)**; Zoom = actual | Drift detection compares the two |
| Attendance | **Zoom** is the source of raw facts; Postgres stores the imported, normalized copy and computed percentages | |
| Message delivery outcome | **Existing comms system** is the source; Postgres stores the reported status | |
| Integration credentials | **Postgres (encrypted)** | |
| Files (materials, exports, recordings copies) | **R2/S3** objects; Postgres holds metadata and keys | |

Rules:

1. A write to an external system is always the result of a Postgres state change. It is never the other way round, except for explicitly inbound-owned data (Sheets inbound columns, Zoom attendance, comms delivery callbacks).
2. External write failures never roll back the Postgres change. The change stays, and the sync is retried or flagged (PRD Case 3).
3. Every external object we create is linked by a row holding `external_id`, `last_synced_at`, `sync_status` and `desired_hash`. This lets us detect drift and re-sync idempotently.
4. Domain events are published through a **transactional outbox** (see [data-flow.md](../architecture/data-flow.md#3-transactional-outbox)), so DB state and emitted events never diverge.

## Consequences

- ✅ Deterministic recovery: "re-sync from Postgres" is always a valid repair.
- ✅ Reports and dashboards never depend on external APIs being up.
- ⚠️ Admins who edit Zoom/Calendar directly create drift. We detect and surface it, and we do not silently overwrite in the MVP (see [zoom-integration.md](../features/zoom-integration.md#drift)).
- ⚠️ Clients used to treating the Sheet as master need a clear column-ownership agreement ([CONFIRM Q3](../planning/open-questions.md)).

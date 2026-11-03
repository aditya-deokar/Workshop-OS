# ADR-011: Column-ownership model for Google Sheets synchronization

- Status: Proposed ([CONFIRM Q3](../planning/open-questions.md))
- Date: 2026-11-03

## Context

The client may manage participants in Google Sheets today **[CONFIRM Q3]**. Humans edit sheets freely: they insert rows, sort, rename headers, and paste over cells. The Sheets API has **no row-level change webhook**. Apps Script triggers and Drive push notifications are coarse or user-edit-only (see [google-workspace.md](../features/google-workspace.md#31-change-detection-options)). Quotas are 60 read and 60 write requests per minute **per user per project** and 300 per project **[VERIFIED]** ([Sheets limits](https://developers.google.com/workspace/sheets/api/limits)).

## Decision

1. **Postgres is master** ([ADR-003](003-postgresql-source-of-truth.md)). A sheet is bound to a workshop through a `sheet_binding` with an explicit **column map**. Each mapped column has an **owner**:
   - `inbound`: the sheet is authoritative for this column (e.g. name/phone/email captured by an external form). Changes are imported.
   - `outbound`: the platform is authoritative (status, payment status if platform-managed, attendance %, join link, `wos_id`). The platform overwrites, and human edits to these cells are reverted and flagged.
   - `inbound_once`: imported on row creation, never re-imported (e.g. source, registration timestamp).
2. **Row identity** uses a platform-written key column `wos_registration_id`. Rows without it are "new rows". They are matched by normalized email and/or E.164 phone, then either linked or created. Ambiguous matches go to a review queue.
3. **Change detection = polling reconciliation** (default every 5 min per active binding, configurable). It reads the full mapped range in **one** `values.batchGet`, computes per-row hashes of inbound columns, and diffs them against `sheet_row_links.inbound_hash`.
4. **Outbound writes** are debounced per binding (Inngest `debounce` keyed by `bindingId`, 30 s) and executed as **one** `values.batchUpdate`. Concurrency is 1 per binding (Inngest `concurrency.key = bindingId`).
5. Rows are always located by key column after a fresh read, **never by remembered row number** (humans sort and insert).
6. Conflicts (an inbound column edited both in the sheet and in the platform since the last sync) are resolved by the owner rule, and every overwrite is logged in `integration_sync_logs` with before/after values.
7. Sync failure never blocks workshop operations (PRD Case 3).

## Consequences

- ✅ Predictable, explainable behavior. The sheet can keep being a human workspace.
- ⚠️ Inbound changes appear with up to the polling interval of latency. An optional Apps Script "nudge" webhook can trigger immediate reconciliation.
- ⚠️ The column map must be agreed with the client per sheet template.

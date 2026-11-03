# Workshop Management

Related: [timeline-engine.md](timeline-engine.md) · [participant-management.md](participant-management.md) · [zoom-integration.md](zoom-integration.md) · [google-workspace.md](google-workspace.md) · Diagrams: [lifecycle](../architecture/diagrams.md#3-workshop-lifecycle), [publish & activation](../architecture/diagrams.md#14-sequence-workshop-publish--activation), [completion](../architecture/diagrams.md#17-sequence-workshop-completion)

---

## 1. Purpose and scope

Manage the workshop aggregate (workshop, days, sessions, trainers, resources) and its **lifecycle** from draft to archive: validation, publish-time provisioning, activation, completion and archival. Multiple workshops can be concurrent and independent (FR-WS-08).

## 2. User stories

| # | Story |
|---|---|
| US-WS-1 | Create "Medical Coding Workshop", Nov 15–16, IST, 2 days, 3 sessions each, with trainers. |
| US-WS-2 | Duplicate last month's workshop, shift dates by +30 days, and keep the timeline and workflow bindings. |
| US-WS-3 | Publish. The system validates everything, creates Zoom webinars and Calendar events, prepares the sheet, and arms the schedule. If something critical is missing, publish is blocked unless I override with a reason. |
| US-WS-4 | Upload session PDFs and publish them after Session 1 (via timeline activity or manually). |
| US-WS-5 | Cancel a workshop: Zoom and Calendar are cancelled, automation stops, and participants are notified (optional). |
| US-WS-6 | After completion, archive the workshop. It becomes read-only but stays reportable. |

## 3. Functional requirements

FR-WS-01…10 ([product-requirements.md](../planning/product-requirements.md#51-workshop-management-doc)).

## 4. Lifecycle

States (see [diagram](../architecture/diagrams.md#3-workshop-lifecycle)):

| State | Meaning | Allowed transitions |
|---|---|---|
| `draft` | Being built. No external side effects | → `ready` (validate ok), `cancelled` |
| `ready` | Validation passed for the current plan revision | → `draft` (on edit), `publishing` |
| `publishing` | Provisioning workflow running | → `published`, `published_with_issues` |
| `published` | Provisioned. Automation may be active | → `in_progress`, `cancelled` |
| `published_with_issues` | Provisioned except some non-blocking/overridden items. Issues listed | → `published` (resolved), `in_progress`, `cancelled` |
| `in_progress` | First session start reached (or manual start) | → `completed`, `cancelled` |
| `completed` | All sessions ended **and** completion workflow finished (or force-complete) | → `archived` |
| `cancelled` | Cancelled with reason | → `archived` |
| `archived` | Read-only | — |

`automation_state` (`inactive | active | paused | stopped`) is **orthogonal** to the lifecycle. A workshop can be `published` with automation `inactive`, which is useful for preparing externals before arming.

Transitions are implemented by a single `WorkshopStateMachine` (pure function: `(state, command) → newState | error`). Every transition writes the audit log plus outbox events (`workshop.published`, `workshop.activated`, `workshop.paused`, …).

## 5. Publish validation

Rules produce issues with `severity: blocking | warning` and a stable `code`:

| Code | Severity | Rule |
|---|---|---|
| `tz_missing` | blocking | Workshop timezone set |
| `no_sessions` | blocking | ≥ 1 session |
| `session_incomplete` | blocking | Each session has start, duration, delivery mode |
| `delivery_capability_missing` | blocking | e.g. webinar mode without webinar capability **[LICENSE]** |
| `simulive_not_linked` | blocking | `zoom_simulive_linked` session without linked webinar ID |
| `zoom_host_overlap` | blocking | Same Zoom host with overlapping sessions **[ASSUMPTION]** |
| `zoom_daily_budget` | warning | Provisioning would exceed 80% of host's 100/day create/update budget **[VERIFIED limit]** |
| `trainer_overlap` | warning | Trainer double-booked (also checks other workshops) |
| `comms_trigger_missing` | blocking | Activity references unregistered/inactive trigger |
| `comms_trigger_unverified` | warning | Trigger not test-verified in the last 30 days |
| `activity_invalid_config` | blocking | Zod validation fails |
| `dependency_cycle` | blocking | |
| `activity_in_past` | blocking/warning | Depends on missed policy |
| `integration_auth_error` | blocking | Any required integration in `auth_error` |
| `recording_missing` | blocking for Simulive, warning otherwise | PRD §6.1 "check recording availability" (verifies only what's checkable: linked webinar exists) |
| `sheet_binding_invalid` | warning | Headers don't match column map |
| `no_registrations` | warning | |

Override: `POST /publish { overrides: [{ issueCode, reason }] }`. Only for `blocking` issues flagged `overridable` (e.g. `zoom_host_overlap` is overridable; `tz_missing` isn't). Overrides are audited and shown in the Control Center as permanent badges.

## 6. Publish provisioning (PRD §6.1)

Built-in **Preparation** workflow (see [automation-engine.md §8](automation-engine.md#8-built-in-workflows-seeded-editable-copies)), executed by the worker:

1. Snapshot the plan revision (`plan_revision`).
2. For each session (parallel, bounded): `zoom.session.ensure` (critical), `calendar.event.ensure` (normal).
3. `sheets.ensure_binding` (create tab/headers if configured) (low).
4. `docs.generate schedule` (P2) (low).
5. If `activate=true`: `automation_state=active` → arm activities within the horizon.
6. `ops.notify` "Workshop ready" with an issue summary.
7. Final status: `published`, or `published_with_issues` if any non-critical failed or any critical item is `needs_attention` (sessions flagged).

Re-running publish is **idempotent**. It ensures desired state again, so it doubles as the "repair" button.

## 7. Duplicate and templates

- Duplicate copies: workshop fields, days (shifted dates), sessions, trainers, activities (generation reset, statuses `pending`), dependencies, sheet binding config (new sheet or same **[CONFIRM]**), and workshop-scoped workflows (as new drafts). It does **not** copy: participants, registrations, runs, external links, recordings.
- Date shift: whole-day shift preserves local times. The planner recomputes instants.
- Templates (P2): `is_template=true` workshops without dates (day numbers only).

## 8. Sessions and delivery modes

| Mode | External object | Notes |
|---|---|---|
| `zoom_meeting` | Zoom meeting (type 2 scheduled) | Default |
| `zoom_webinar` | Zoom webinar (scheduled) | **[LICENSE]** Webinar add-on |
| `zoom_simulive_linked` | Existing Simulive webinar created manually in Zoom | API can't create/configure Simulive ([zoom §4](zoom-integration.md#4-simulive-what-is-and-isnt-possible)) |
| `external_link` | None | Link supplied by admin |
| `in_person` | None | Calendar only |

Session `status`: `scheduled → live → ended` from Zoom webhooks (`*.started`, `*.ended`), manual marking, or schedule fallback (if no webhook arrives within 15 min after scheduled end → `ended (assumed)` + warning).

## 9. Resources (materials)

- Upload via presigned R2 PUT ([api §2.9](../engineering/api-specification.md#29-files)). Max size 200 MB (configurable). Allowed MIME types are allow-listed.
- Publishing a resource = `visibility=published` + a comms trigger with a **time-limited presigned GET URL** (default 7 days) or a stable short link through our API that re-signs on access **[CONFIRM Q10: are links for participants expected to be permanent?]**.
- Antivirus scanning (P2) via an object-created hook.

## 10. Completion and archival

- `workshop.sessions_all_ended` is emitted when every non-cancelled session is `ended`.
- **Completion** workflow (PRD §6.5): wait for attendance imports → compute completion per registration using the workshop's `completion_rule` (default: overall attendance ≥ 75% **[CONFIRM Q6]**) → borderline (± 5%) and unmatched cases go to the review queue → follow-up comms for eligible participants → final sheet export → reports.
- Admin can **force-complete** (reason required) if attendance data is unavailable.
- **Archive**: the workshop becomes read-only (API rejects writes with `409 archived`). Execution data stays queryable until the archival retention moves it to R2.

## 11. Data entities

`workshops`, `workshop_days`, `sessions`, `session_trainers`, `trainer_profiles`, `resources`, `recordings`. See [database-design.md §3.2](../engineering/database-design.md#32-workshop-structure).

## 12. API contracts

[api-specification.md §2.2–2.3](../engineering/api-specification.md#22-workshops).

## 13. Events

Emits `workshop.created|updated|published|publish_failed|activated|paused|resumed|stopped|emergency_stopped|cancelled|rescheduled|in_progress|sessions_all_ended|completed|archived`, `session.created|updated|rescheduled|cancelled|live|ended`, `resource.published`. Consumes `zoom.session.started|ended`, completion workflow results.

## 14. External dependencies

Zoom, Google Calendar, Google Sheets, R2, comms systems (through the gateway).

## 15. Error scenarios and recovery

| Scenario | Behavior |
|---|---|
| Zoom creation fails at publish | Session `needs_attention`, dependents (joining info) blocked, workshop `published_with_issues`, "Retry Zoom" button (PRD Case 2) |
| Calendar fails at publish | Retry queue. Workshop status unaffected (PRD Case 3) |
| Publish worker crashes in the middle of provisioning | Steps are memoized/idempotent. Inngest resumes. Re-publish also converges |
| Cancel while publishing | Cancel waits for the publishing run to reach a step boundary (`cancelOn workshop.cancelled`), then runs compensation (cancel created Zoom/Calendar objects) |
| Session deleted with existing Zoom object | Disallowed after publish (use cancel). Draft deletes have no externals |

## 16. Security

Permissions: `workshops.read`, `workshops.write`, `workshops.publish`, `automation.control`. Cancel and archive require admin. All transitions are audited.

## 17. Edge cases

- Workshop with sessions on non-consecutive days → days are explicit rows. Fine.
- A session crossing midnight local time → allowed. `ends_at` is on the next date. Day grouping uses start.
- Editing workshop timezone after publish → treated as a reschedule of everything (preview mandatory). Discouraged with a warning.
- Two workshops sharing trainers/hosts at the same time → overlap validation across workshops.
- Capacity reached → new registrations become `waitlisted` (if enabled).

## 18. Testing

- State machine exhaustive transition table test.
- Validation rules: one test per code.
- Publish provisioning integration test with mocked adapters (success, Zoom failure, Calendar failure, crash in the middle).
- Duplicate: deep-equality of structure with shifted instants.

## 19. Acceptance criteria

1. Publishing a valid 2-day workshop creates exactly one Zoom object and one Calendar event per session, even if publish is triggered twice.
2. A blocking issue prevents publish unless overridden (if overridable), and the override is audited.
3. Duplicating a workshop produces a draft with the same timeline at shifted dates and no external links.
4. Completion runs only after all sessions have ended and attendance imports are done (or have timed out). Results appear in reports.

## 20. Implementation dependencies

DB + auth → workshop CRUD + state machine → validation framework → timeline (planner) → adapters (Zoom, Calendar, Sheets) → publish workflow → completion workflow (needs attendance).

## 21. Trade-offs

- Orthogonal automation state adds a second dimension, but lets "provision now, arm later" and pause/resume work without lifecycle hacks.
- Publish-as-repair (idempotent ensure) vs a dedicated repair tool: one code path, less to test.

## 22. Future improvements

- Workshop templates with parameter forms.
- Capacity/waitlist automation with auto-promotion.
- Multi-language session metadata.

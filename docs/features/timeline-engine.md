# Timeline Engine

> The PRD calls this "the most important feature of the platform". The timeline defines **what should happen**. The engine makes it happen through durable execution.

Related: [ADR-007](../adr/007-timeline-scheduling-strategy.md) · [workshop-management.md](workshop-management.md) · [automation-engine.md](automation-engine.md) · [execution-monitoring.md](execution-monitoring.md) · [reliability-patterns.md](../engineering/reliability-patterns.md) · Diagrams: [timeline execution](../architecture/diagrams.md#4-timeline-execution-engine), [rescheduling](../architecture/diagrams.md#11-workshop-rescheduling)

---

## 1. Purpose and scope

- Model a workshop's full schedule: **Workshop → Days → Sessions → Activities → Actions**.
- Compute exact execution instants from absolute or relative timing in the workshop's timezone.
- Execute activities reliably in three modes (scheduled, event-based, sequential), respecting dependencies, state and failure policies.
- Support live control (pause/resume/stop/skip/retry/execute now/override) and rescheduling with external propagation.

Out of scope: event-driven multi-step business logic that isn't time-anchored. That belongs to the [automation engine](automation-engine.md). An activity *can* invoke a workflow (action `automation.run_workflow`).

## 2. User stories and use cases

| # | Story |
|---|---|
| US-TL-1 | As an admin, I build Day 1 with activities at 08:00, 08:45, 09:00… by dragging them on a time grid. |
| US-TL-2 | As an admin, I define "24 h before each session: reminder" **once**, and it applies to every session. Times follow automatically when sessions move. |
| US-TL-3 | As an admin, I chain "Session 1 ends → publish PDF → wait 15 min → open Session 2". |
| US-TL-4 | As an admin, I move Session 2 from 10:00 to 11:00, see the impact (Zoom, Calendar, 3 reminders, 248 notifications), and confirm. |
| US-TL-5 | As an operator, I pause automation during an incident and resume later. I choose what happens with activities that were missed during the pause. |
| US-TL-6 | As an operator, I skip "Assessment" with a reason, and dependent activities follow their dependency rules. |
| US-TL-7 | As a trainer/operator, I confirm a manual task ("Trainer confirms room ready"). The next activities then proceed. |
| US-TL-8 | As an admin, a failed Google Sheet sync doesn't stop Session 2's Zoom delivery. |

## 3. Functional requirements

Implements FR-TL-01…10 ([product-requirements.md](../planning/product-requirements.md#52-timeline-doc)). Every activity has: name, description, timing, duration, execution mode, assigned integration, configuration, dependencies, success condition, failure behavior, enable/disable, criticality, retry policy, missed policy.

## 4. Activity types

| Type | Action key | Scope | Default criticality | Idempotency class | Notes |
|---|---|---|---|---|---|
| WhatsApp trigger | `comms.trigger` (channel whatsapp) | per participant (fan-out) or bulk | normal | per trigger config (`keyed`/`unsafe`) | Invokes **existing** workflow ([doc](whatsapp-email-integration.md)) |
| Email trigger | `comms.trigger` (channel email) | per participant / bulk | normal | same | |
| Zoom meeting | `zoom.session.ensure` (type meeting) | session | **critical** | checkable | Create/update to desired state |
| Zoom webinar | `zoom.session.ensure` (type webinar) | session | **critical** | checkable | **[LICENSE]** Webinar add-on |
| Start Simulive | `zoom.simulive.verify` | session | critical | natural | **Verification only.** Checks the linked webinar exists, its schedule matches, and its status. Doesn't start playback ([zoom §4](zoom-integration.md#4-simulive-what-is-and-isnt-possible)) |
| Zoom delivery start | `zoom.session.prepare_host` | session | critical | natural | Notifies host(s) with a link to the Control Center's "Start as host" (fresh `start_url`). Zoom has no server-side "start meeting" for regular meetings ([zoom §3](zoom-integration.md#3-verified-zoom-api-capabilities)) |
| Publish recording | `recording.publish` | session | normal | natural | Requires `zoom.recording.completed` processed. Then triggers comms with link |
| Publish material | `resource.publish` | session | normal | natural | Marks published + comms trigger |
| Google Sheet sync | `sheets.sync` | binding | low | natural | Never blocks others |
| Google Doc generation | `docs.generate` | workshop / participant | low | checkable | P2 |
| Calendar event | `calendar.event.ensure` | session / workshop | normal | natural (deterministic ID) | |
| Participant status | `participants.set_status` | audience | normal | natural | Internal only |
| Attendance sync | `zoom.attendance.import` | session | normal | natural | Usually event-mode after `zoom.session.ended` |
| Wait | `control.wait` | — | — | — | Pure delay. Usually modelled as dependency lag |
| Conditional branch | `control.branch` | — | — | — | Evaluates a condition and enables one of two successor sets |
| Admin notification | `ops.notify` | — | low | natural | In-app + ops channel |
| Manual task | `control.manual_task` | — | configurable | — | Status `awaiting_confirmation` until confirmed, with timeout policy |
| Run workflow | `automation.run_workflow` | — | per workflow | — | Bridges to Automation Studio |

## 5. Timing model

### 5.1 Anchors

| `anchor_type` | Meaning | Example |
|---|---|---|
| `absolute` | Local date (from day) + `local_time` in workshop tz | Day 1, 08:00 |
| `workshop_start` | First session start + offset | −P7D "preparation check" |
| `day_start` | First session of the day + offset | −PT1H |
| `session_start` | A session's start + offset | −PT15M final reminder |
| `session_end` | A session's scheduled end + offset | +PT0M "Session 1 completed" |
| *(sequential)* | Completion of predecessor(s) + lag | +PT15M after "Publish PDF" |
| *(event)* | Occurrence of a normalized event, optional window | on `zoom.session.ended` for session X |

**Relative template activities.** A "pre-workshop schedule" (−7 d, −3 d, −24 h, −1 h, −15 min, per PRD §6.3) is stored as **workshop-level activity templates** with `anchor_type=session_start` and `applies_to=each_session|first_session`. On save they **materialize** into concrete activities per session (one row each, linked by `template_activity_id`). This gives each instance its own status and run history.

### 5.2 Computing `scheduled_for`

`TimelinePlanner.compute(workshop)` is a pure function: plan in → instants out.

1. Resolve sessions: `starts_at = zone(local_date + local_start_time, tz)` with gap/overlap rules ([reliability-patterns §1](../engineering/reliability-patterns.md#1-reliable-scheduled-execution-and-timezone-handling)).
2. Resolve anchored activities: `anchor_instant ± offset` (calendar vs elapsed semantics).
3. Sequential/event activities: `scheduled_for = null` until resolved at runtime. The UI shows the **estimated** time = predecessor's scheduled end + lag.
4. Validation: an activity scheduled in the past at publish time → blocking issue, unless `missed_policy` allows it. An activity after workshop end + 30 d → warning.

## 6. Scheduling strategy

Decision: [ADR-007](../adr/007-timeline-scheduling-strategy.md). Summary:

- **Arming** creates an `activity_runs` row (status `scheduled`, `generation = activity.schedule_generation`) and emits `timeline/activity.armed`. This happens only when:
  - workshop `automation_state = active`, and
  - activity `enabled`, and
  - mode `scheduled` with `scheduled_for <= now + ARMING_HORIZON (21 d)`, **or** mode `sequential` with dependencies satisfied, **or** mode `event` (armed as a *listener* run that `waitForEvent`s with timeout).
- **Disarming** (reschedule/disable/stop) bumps the generation and emits `timeline/activity.disarmed`. `cancelOn` cancels sleeping runs, and the claim check fences any run that escapes cancellation.
- **Cron jobs** (Inngest):
  - `timeline-arming-sweep`: daily at 00:05 UTC, plus on activation. Arms activities entering the horizon.
  - `timeline-overdue-sweeper`: every minute. Re-emits overdue `scheduled` runs (same `runId`, deduped). Escalates if overdue twice.
- **Watchdog** (api role, alert-only): overdue more than 5 min → critical alert.

### 6.1 Event-mode activities

- Listener run: `step.waitForEvent('wait', { event: 'zoom.session.ended', if: 'async.data.sessionId == "<id>"', timeout: activity.event_trigger.timeout })`.
- Because `waitForEvent` only sees events emitted **after** the wait begins **[ASSUMPTION per Inngest semantics; verify]**, the listener first checks Postgres for whether the event already happened (e.g. `sessions.actual_ended_at not null`). If it has, the listener proceeds immediately.
- On timeout → `timeout_policy`: `run_anyway | skip | needs_attention`. Example: attendance sync runs anyway 2 h after scheduled session end even if no `meeting.ended` webhook arrived.

## 7. Execution algorithm

Function `timeline-activity-execute` (one run per armed activity run):

```text
1. sleepUntil(scheduled_for)                                   [skipped for sequential "now" / event]
2. step "claim":
     UPDATE activity_runs r SET status='running', claimed_at=now()
     FROM timeline_activities a, workshops w
     WHERE r.id=$run AND r.status='scheduled'
       AND a.id=r.activity_id AND a.schedule_generation=r.generation AND a.enabled
       AND w.id=r.workshop_id AND w.automation_state='active'
     RETURNING …
   → 0 rows: diagnose (paused → mark run 'paused'; superseded → 'cancelled'; already running/done → no-op) and exit
3. step "missed-check": if now - scheduled_for > grace → apply missed_policy (skip | ask → awaiting_confirmation)
4. step "preconditions":
     - dependencies satisfied per condition (re-check; protects against races)
     - required capabilities present; integration not auth_error
     - success conditions of upstream critical activities hold (e.g. zoom_sessions.sync_status='synced')
   fail → per failure_policy (block → needs_attention; else skipped with reason)
5. execute:
     - session/workshop-scoped action → gateway.call (1 action execution)
     - participant-scoped action:
         a. step "resolve-audience" → registration IDs (filter, exclude already-succeeded for same activity)
         b. if trigger.supports_bulk → chunk (e.g. 100) → gateway.call per chunk
            else step.sendEvent fan-out `comms/send.requested` (id = runId:registrationId) → child function
         c. step.waitForEvent("fanout-complete", timeout = deadline) — the child aggregator emits
            `timeline/fanout.completed` when all children are terminal (counter in activity_runs.outcome_summary)
6. step "evaluate-success": apply success_condition (e.g. ≥ 95% accepted) → succeeded | failed
7. step "finalize": TX update run + activity.current_status, outbox `timeline/activity.completed {outcome}`, notify SSE
```

Child function `comms-send` (per registration):

- `idempotency: 'event.data.runId + ":" + event.data.registrationId'`
- `throttle: { key: 'event.data.integrationId', limit: <cfg>, period: '1s' }`
- `cancelOn: workshop.emergency_stopped`, matched on `data.workshopId`
- step: check workshop still active (pause stops remaining sends quickly), registration still eligible, and consent present
- step: gateway.call `comms.trigger`
- step: increment the aggregate counter atomically. If it was the last one, emit `fanout.completed`

## 8. Dependencies

- Types: `on_success`, `on_completion`, `on_failure`. `lag` interval. `required` flag.
- **Resolver** (function `timeline-dependency-resolve`, triggered by `timeline/activity.completed`):
  1. Load dependents of the completed activity.
  2. For each dependent: evaluate all its dependencies' current outcomes.
     - All satisfied → arm with `scheduled_for = max(anchored time if any, max(pred.finished_at + lag))`.
     - A required dependency is unsatisfiable (e.g. `on_success` but predecessor failed/skipped) → dependent `skipped (dependency_unsatisfied)`, **or** `needs_attention` if the predecessor was critical. This propagates transitively.
     - Otherwise remain `waiting`.
- Hybrid activities (anchored **and** dependent): run at the anchored time **only if** dependencies are satisfied by then. Otherwise wait, up to `max_delay` → then `needs_attention`. Example: "Joining instructions at −1 h, requires Zoom webinar created".

## 9. Pause, resume, stop, emergency stop

| Control | Timeline effect |
|---|---|
| **Pause** | `automation_state=paused` (TX + audit + outbox `workshop.paused`). Sleeping runs stay sleeping. On wake, the claim fails → run `paused`. Fan-out children stop before their next call. **External sessions are unaffected** (Zoom keeps running), and the UI says so explicitly |
| **Resume** | Preview lists runs that were `paused` and whether their time has passed. On confirm: `automation_state=active`. For each paused run: future → re-arm (new generation). Past → `missed_policy` (default `run_if_within_grace` 30 min, else `ask`) |
| **Stop** | Disarm all non-terminal runs (`cancelled: stopped`). Running actions complete. Re-activation re-arms future activities only |
| **Emergency stop** | Stop + emit `workshop.emergency_stopped` (cancels child runs via `cancelOn`). Breaker forced open for this workshop's comms. Optional, separately confirmed: end live Zoom sessions via `PUT /meetings/{id}/status {action:"end"}` **[VERIFIED endpoint exists; confirm webinar equivalent in SPIKE S3]** |
| **Global emergency stop** (owner) | Applies emergency stop to all active workshops |

## 10. Editing a published timeline

| Activity state | Allowed edits | Mechanism |
|---|---|---|
| `pending`/`waiting`/`scheduled` (future) | All fields | Optimistic lock → recompute → generation++ → disarm + re-arm → audit (`plan_revision++`) |
| `running` | None (except "skip remaining" for fan-outs) | Must stop/skip first |
| Terminal | None | Manual override of outcome only (audited) |
| Deleting | Draft: delete. Published: **disable** (keeps history) | |

Editing an anchor (e.g. a session's time) cascades to all anchored activities in the same transaction. That is the reschedule path (§11).

## 11. Rescheduling

1. **Preview** (`POST …/reschedule/preview`) runs `TimelinePlanner` on the proposed plan and diffs it against the current one:
   - activities whose `scheduled_for` changes (old → new, local + UTC)
   - activities that move into the past (and their missed policy)
   - external updates: Zoom PATCH count (vs remaining daily budget **[VERIFIED limit 100/day/user]**), Calendar patches
   - notifications: count of participants per comms trigger (`schedule_change`)
   - warnings: trainer overlaps, Zoom host overlaps, dependencies violated
2. **Commit** (single TX): update sessions/days, recompute, generation++ on changed activities, cancel superseded runs, insert new runs, outbox (`activity.disarmed`×N, `activity.armed`×N, `session.rescheduled`/`workshop.rescheduled`), audit with full diff.
3. **Propagation** (reschedule workflow, built-in and versioned):
   - `zoom.session.ensure` (PATCH start_time/duration) → **critical**. Failure blocks the `schedule_change` notification for that session's participants (PRD Case 2/4).
   - `calendar.event.ensure` → normal. Failure → retry queue only (PRD Case 3).
   - `comms.trigger schedule_change` (if the admin chose to notify) → after Zoom success.
   - `sheets.sync` (debounced).
4. **Race safety**: if a run for an affected activity is currently `running`, the reschedule is rejected for that activity (409 with details), unless the admin chooses "let it finish, then apply".

## 12. Data entities

`timeline_activities`, `activity_dependencies`, `activity_runs`, plus `action_executions`/`action_attempts`. See [database-design.md §3.3](../engineering/database-design.md#33-timeline).

## 13. API contracts

[api-specification.md §2.4](../engineering/api-specification.md#24-timeline), plus workshop reschedule/pause/resume endpoints in §2.2.

## 14. Events, triggers and actions

| Emits | Consumes |
|---|---|
| `timeline/activity.armed`, `timeline/activity.disarmed`, `timeline/activity.completed`, `timeline/fanout.completed`, `timeline/run.needs_attention` | `workshop.activated/paused/resumed/stopped/emergency_stopped`, `session.rescheduled`, normalized external events (event mode), `zoom.session.ended`, `attendance.imported` |

## 15. External dependencies

Inngest (scheduling/execution). Indirectly all adapters via the gateway.

## 16. Error scenarios and recovery

| Scenario | Behavior |
|---|---|
| Inngest misses a wake-up | Sweeper re-emits within 1–2 min. Watchdog alerts at 5 min |
| Worker crash in the middle of an action | Step retried. Gateway detects an open attempt → idempotency-class rule |
| Fan-out partially fails | Activity outcome per success condition. Failed recipients listed with a one-click retry for **failed only** |
| Dependency predecessor stuck `running` | Stuck detector alert after `expected_max_duration`. Admin can override the predecessor |
| Reschedule commit succeeded but outbox relay delayed | Old runs still sleeping. When they wake, the claim fails on generation. New runs arm when the relay catches up (≤ 5 s poll) |
| Activity scheduled during pause and grace passes | `missed_policy` decides on resume. Default `ask` for comms |
| Clock skew between DB and worker | Claim uses DB `now()`. Sleep target is absolute UTC |

## 17. Security considerations

- Permissions: `timeline.write` (edit), `timeline.reschedule`, `automation.control` (pause/resume/stop), `automation.emergency`, `executions.retry`, `executions.override`.
- Every control action requires a reason (except pause, where it's optional) and is audited.
- "Execute now" with `bypassDependencies` requires `admin` role.

## 18. Edge cases

- Two activities at the same instant: ordering by `position` is **not** guaranteed. If order matters, add a dependency.
- A session moved before its own −24 h reminder's time has passed → reminder becomes past → missed policy.
- An activity anchored to a deleted/cancelled session → disabled automatically, with a warning.
- A workshop spanning a DST transition (non-IST zones) → calendar vs elapsed offsets differ by 1 h. Preview shows both.
- Audience empty at execution (everyone cancelled) → `succeeded` with `total: 0` (not failed), plus an info log.
- Very large audience (> 5,000) → multiple `sendEvent` batches (5,000 per call **[VERIFIED]**).
- An activity disabled while its fan-out is running → remaining children skip (they check `enabled` and generation).

## 19. Testing requirements

- Unit: `TimelinePlanner` (property tests, DST), dependency resolver (DAG cases, transitive skip), claim SQL (concurrency test with parallel claims → exactly one wins).
- Integration (Inngest Dev Server + Testcontainers Postgres): arm → sleep (shortened clock) → execute. Reschedule during sleep (old run no-ops). Pause/resume with missed policy. Worker restart during fan-out.
- Chaos: kill the worker every 30 s during a synthetic 200-activity workshop. Assert 0 duplicates, 0 lost.
- E2E (Playwright): drag activity, reschedule with preview, pause/resume from Control Center.

## 20. Acceptance criteria

1. A 3-day workshop plan with 40 activities computes correct UTC instants for `Asia/Kolkata` and `America/New_York` (golden files).
2. Activities start within 60 s p95 of `scheduled_for` on staging.
3. Rescheduling a session updates all anchored activities, cancels obsolete runs (verified: no execution at the old time), and propagates to Zoom/Calendar/comms with records.
4. Pausing for 30 min across 3 scheduled activities, then resuming, follows each activity's missed policy.
5. A failed `sheets.sync` (low criticality) doesn't affect the next session's activities. A failed `zoom.session.ensure` blocks only its dependents.
6. No activity run executes twice under any tested failure (chaos test).

## 21. Implementation dependencies

Database foundation → Inngest platform module (SPIKE S1) → integration gateway + execution records → timeline planner → arming/claim/sweeper → dependency resolver → fan-out → controls → reschedule workflow (needs Zoom + Calendar adapters).

## 22. Technical trade-offs

| Choice | Trade-off |
|---|---|
| Per-activity runs vs single workshop run | More runs. But edits are O(1), there's isolation, and run lengths are bounded |
| Horizon arming | Extra cron + state. Avoids plan max run length limits |
| Materialized template instances | More rows. Each instance gets its own status/history and per-instance overrides |
| Generation fencing in addition to `cancelOn` | Double mechanism. `cancelOn` is an optimization, and fencing gives correctness |
| Minute-level precision | Not suitable for sub-second choreography (not needed) |

## 23. Future improvements

- Visual Gantt with critical-path highlighting.
- "What-if" simulation mode (simulated clock) for training.
- Workshop templates with parameterized offsets.
- Per-participant timezone personalization of reminders (e.g. international participants).

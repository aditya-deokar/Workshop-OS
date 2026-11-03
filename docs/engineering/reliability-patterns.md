# Reliability Patterns: the 20 Critical Engineering Considerations

These rules apply to **every module**. Feature docs reference the section numbers below.

Related: [ADR-002](../adr/002-inngest-orchestration.md) · [ADR-007](../adr/007-timeline-scheduling-strategy.md) · [ADR-012](../adr/012-unknown-outcome-no-auto-retry.md) · [data-flow.md](../architecture/data-flow.md) · [database-design.md](database-design.md)

---

## 1. Reliable scheduled execution and timezone handling

**Problem.** Activities are defined in the workshop's local time ("08:00 AM IST on Day 1") or relative to other times ("24 h before session start"). They must fire at the correct instant, even across DST changes, and never depend on a browser tab.

**Design.**

- Store **three things** for absolute activities: `local_time` (e.g. `08:00`), the owning day (`workshop_days.date`, local date), and `workshops.timezone` (IANA). `scheduled_for timestamptz` is **derived** and recomputed by `TimelinePlanner` whenever any input changes.
- Relative activities store `anchor_type` (`workshop_start | day_start | session_start | session_end | activity_completed`), `anchor_id`, and `offset` (ISO 8601 duration, e.g. `-PT24H`).
- **Offset semantics across DST:** an offset expressed in whole days (`-P1D`) means "same local wall time, previous calendar day". An offset in hours (`-PT24H`) means "exactly 24 elapsed hours". Both are supported explicitly. The UI defaults to day semantics for day-level offsets.
- **Non-existent local times** (spring-forward gap): shift forward to the first valid instant, and warn at validation. **Ambiguous times** (fall-back overlap): choose the earlier offset, and warn. India (IST) has no DST, but the engine must not assume that **[ASSUMPTION: client may run workshops in other zones, CONFIRM Q16]**.
- The API exchanges instants as UTC ISO strings. The UI always renders in the **workshop's** timezone, with the viewer's local time in a tooltip.
- Execution is driven by Inngest `step.sleepUntil(scheduled_for)` ([ADR-007](../adr/007-timeline-scheduling-strategy.md)), backed by the sweeper and the watchdog.
- Server clocks: containers use NTP-synced hosts. Never compare a client clock to a server clock.

**Tests.** Property tests for `TimelinePlanner` across zones with DST (`America/New_York`, `Europe/London`) and without (`Asia/Kolkata`). Golden tests for gap/overlap cases.

## 2. Durable workflows and Inngest integration with NestJS

See [nestjs-inngest-integration.md](../architecture/nestjs-inngest-integration.md). Key rules:

- Side effects only inside `step.run`. Stable step IDs. Small step outputs. ≤ 1,000 steps per run, so fan out with events.
- Postgres claims fence every execution, and Inngest state is never authoritative.
- Function IDs are permanent. Breaking changes ship as new function IDs, and old ones are drained.

## 3. Idempotency and duplicate prevention

Idempotency is applied **at every boundary**:

| Boundary | Mechanism |
|---|---|
| Admin API mutations | `Idempotency-Key` header (required on POST control endpoints). Stored in `api_idempotency_keys (key, user_id, request_hash, response, expires_at)` for 24 h. Same key with a different body returns 422 |
| Outbox → Inngest | Event `id` = outbox ULID (Inngest 24 h dedupe **[VERIFIED]**) |
| Inngest function start | `idempotency` expression, e.g. `event.data.runId` (24 h **[VERIFIED]**) |
| Activity run execution | Conditional-UPDATE **claim** (status + generation), which works forever, not just 24 h |
| Action execution | `action_executions.idempotency_key UNIQUE`. Replay returns the stored result |
| External write | Provider-specific: Calendar deterministic event ID. Zoom checkable via agenda tag. Comms `Idempotency-Key` header (if honored) |
| Inbound webhook | `webhook_events.dedupe_key UNIQUE` |
| Registration intake | `UNIQUE (workshop_id, participant_id)` + request `Idempotency-Key` |
| Sheet import | `sheet_row_links (binding_id, row_key) UNIQUE` + inbound hash |

**Key derivation** is deterministic from business identity, **never** from random values generated outside a step. Example: `comms:${triggerKey}:${activityRunId}:${registrationId}`.

## 4. Webhook authentication and replay protection

| Source | Authentication | Replay protection |
|---|---|---|
| Zoom | `x-zm-signature = v0=HMAC_SHA256(secret, "v0:{x-zm-request-timestamp}:{rawBody}")`. URL validation challenge with `plainToken` → `encryptedToken` **[VERIFIED]** ([Zoom webhooks](https://developers.zoom.us/docs/api/webhooks/)) | Reject `|now − timestamp| > 300 s`. Dedupe key in inbox |
| Comms callbacks (our contract) | `X-WOS-Signature: t=<unix>,v1=<HMAC_SHA256(secret, t + "." + rawBody)>`. Secrets per integration, dual-secret rotation | 300 s window + `X-WOS-Delivery-Id` unique |
| Apps Script nudge | Same HMAC scheme with a per-binding secret stored in Script Properties | Window + coalescing key |
| Registration intake API | API key (hashed at rest) + HMAC optional | `Idempotency-Key` required |
| Google Calendar push (optional, P2) | Channel `token` we set at `watch` time + `X-Goog-Channel-ID` match | Message number monotonic check |
| Inngest → worker | SDK verifies `X-Inngest-Signature` with signing key **[VERIFIED: signing key required in cloud mode]** | SDK-handled |

Constant-time comparison everywhere (`crypto.timingSafeEqual`). Signature failures are logged as security events (rate-limited) and counted in metrics.

## 5. Retry policies, exponential backoff and dead-letter handling

- **Retry policy object** (per action type default, overridable per activity/workflow node):
  `{ maxAttempts: 5, initialDelay: "PT30S", multiplier: 2, maxDelay: "PT30M", jitter: "full", retryOn: ["transient","rate_limited"], deadline?: "PT2H" }`.
- **Implementation:** Inngest step retries provide the mechanics. Default function retries is 4, the maximum is 20 **[VERIFIED]**. For a custom backoff curve, the gateway throws `RetryAfterError(delay)` computed from our policy **[ASSUMPTION: available in v4]**. If that isn't possible, an explicit loop `for attempt … step.run(attempt) → step.sleep(backoff)` with stable step IDs `action:${id}:attempt:${n}` gives full control. **Decision:** explicit loop for comms/Zoom actions (precise policy and per-attempt records). Inngest default retries for internal DB-only steps.
- **Deadline awareness:** a reminder that's only useful before session start gets `deadline = session.starts_at`. Retries stop at the deadline → `failed (deadline_exceeded)`.
- **Dead letter:** an action with exhausted retries becomes `failed` with `dead_lettered_at`. The Execution Monitor's **Dead Letter** view supports filter, bulk retry (new attempt series, same idempotency key), skip, and export.
- **Inngest `onFailure`** handlers catch unexpected function-level failures (bugs) and mark the owning run `needs_attention`, so nothing remains silently `running`.
- **Stuck detection:** a run that stays `running` for longer than `expected_max_duration` (per action type) raises `execution.stuck`.

## 6. Workflow versioning and editing published workflows

See [ADR-008](../adr/008-workflow-versioning.md). Timeline edits after publish are **plan revisions** ([timeline-engine.md §10](../features/timeline-engine.md#10-editing-a-published-timeline)):

- Future activities are editable. Each edit bumps the generation and re-arms.
- Running activities can't be edited (stop/skip them first).
- Completed activities are immutable, except through an audited **manual override** of their outcome.

## 7. Workshop pause, resume, cancellation and rescheduling

| Control | Effect on our jobs | Effect on external | Notes |
|---|---|---|---|
| **Pause** | `automation_state=paused`. Sleeping runs stay. On wake, the claim fails → run `paused`. In-flight steps finish their current action (not interrupted) | None. A running Zoom session continues (PRD) | Fan-out children check workshop state before each call, so a pause stops remaining recipients within seconds |
| **Resume** | `automation_state=active`. Re-arm paused runs. Missed ones follow `missed_policy` (run now within grace / skip / ask) | None | Impact preview shows what will run immediately |
| **Stop** | `automation_state=stopped`. Disarm all scheduled runs (`cancelled: stopped`). Running actions complete | None | Re-activate re-arms remaining future activities |
| **Emergency stop** | Like stop, plus: cancel in-flight fan-outs (`cancelOn workshop.emergency_stopped`). Open circuit for comms of this workshop | Optional checkbox: end live Zoom sessions (explicit, separate confirmation) | Global emergency stop (all workshops) is available to `owner` |
| **Cancel workshop** | Stop + lifecycle `cancelled` | Cancel Zoom meetings/webinars, cancel Calendar events, optional `workshop_cancelled` comms trigger | Irreversible. Requires reason |
| **Reschedule** | Recompute, generation++, disarm/arm | Zoom PATCH, Calendar patch, optional notifications | Impact preview mandatory |

## 8. Dependencies between timeline activities

- DAG of `activity_dependencies(activity_id, depends_on_id, condition, lag, required)`.
- `condition ∈ {on_success, on_completion, on_failure}` (completion = any terminal state except cancelled).
- `required=false` means "soft" dependency: if the predecessor is skipped/failed, the dependent still runs (logged).
- Cycle detection at save time (Kahn's algorithm) and again at publish.
- Cross-session and cross-day dependencies are allowed **within the same workshop only**.
- Resolution happens on `activity.completed` events. The resolver arms dependents with `scheduled_for = max(own scheduled time if any, predecessor.completed_at + lag)`.
- Critical-failure propagation: if a **critical** activity fails, dependents with `on_success` become `blocked` (shown as `needs_attention` on the predecessor). Example: Zoom webinar creation failed → joining instructions blocked (PRD Case 2).

## 9. Concurrent workshops and overlapping schedules

- Every Inngest function carrying workshop work sets `concurrency: [{ key: 'event.data.workshopId', limit: 10 }]` (per-workshop fairness) and uses provider-level throttles keyed by integration (shared budget).
- Overlap detection at validation: the same trainer in overlapping sessions (warning), the same Zoom host user in overlapping meetings (**blocking** for webinars and meetings on the same host **[ASSUMPTION: Zoom host can't host two concurrent meetings without specific settings; verify]**), and the same comms trigger burst > throttle capacity (warning with estimated completion time).
- Fan-out priority: Inngest `priority` expression gives time-critical activities (e.g. "15 min before" reminders) higher priority than bulk syncs (range −600..600 **[VERIFIED]**).

## 10. External API rate limits and outages

| Provider | Verified limits | Strategy |
|---|---|---|
| Zoom | Per-second by category and plan (e.g. Pro: Light 30/s, Medium 20/s, Heavy 10/s). **Meeting/webinar create/update: 100 requests/day per user (UTC reset)** **[VERIFIED]** ([rate limits](https://developers.zoom.us/docs/api/rate-limits/)) | Throttle per account. Daily budget counter per host. Batch changes (one PATCH per session per reschedule). Spread sessions across host users if > 100 changes/day |
| Google Sheets | 300 req/min/project, **60 req/min/user/project** for reads and writes separately. 180 s request timeout. 2 MB recommended payload **[VERIFIED]** ([limits](https://developers.google.com/workspace/sheets/api/limits)) | Batch get/update (1 request each). Debounce. Per-binding concurrency 1. Truncated exponential backoff |
| Google Calendar | Per-project/per-user quotas (values not re-verified here) **[ASSUMPTION: check console quotas]** | Throttle. Patch only on hash change |
| Existing comms | Unknown **[CONFIRM Q1]** | Throttle configurable per integration. Bulk endpoint if offered |
| Inngest | Plan concurrency (Pro 100 concurrent steps) **[VERIFIED]** | Keep steps short. Fan-out sized accordingly |

**Outage handling:** circuit breaker per integration ([integration-architecture.md §6](../architecture/integration-architecture.md#6-rate-limiting-and-flow-control)). Unrelated activities continue. Deadline-bound actions fail at their deadline with an alert, instead of firing late and becoming misleading (e.g. "starts in 15 minutes" sent after start).

## 11. Integration credential storage and OAuth refresh

- Envelope encryption: a per-record data key (AES-256-GCM) wrapped by a KEK from a cloud KMS (or `WOS_MASTER_KEY` in dev). `integration_credentials(ciphertext, iv, auth_tag, key_id, version)`.
- Decrypted only inside the adapter, in memory, for the duration of the call. Never returned by the API (`hasSecret: true` only).
- Zoom S2S: access tokens last ~1 h. Fetch new tokens on demand, with an in-process cache and a DB cache to share across workers. Refresh under `SELECT … FOR UPDATE`.
- Google user OAuth: store the refresh token. Detect `invalid_grant` → integration `auth_error` + alert. **Publish the OAuth consent screen to "In production"**, because refresh tokens for External apps in "Testing" status expire after 7 days **[COMMUNITY, widely reported; verify]**. An "Internal" user type (Workspace org) avoids verification for restricted scopes **[CONFIRM Q8]**.
- Health check per integration every 15 min (cheap call) updates `integrations.health_state`.

## 12. Partial failures across multiple providers

Example: a session reschedule succeeds in Zoom but fails in Calendar.

- **No distributed transaction.** Each provider update is an independent action with its own record. Postgres holds the desired state, and each external link tracks `desired_hash` vs `synced_hash`.
- **Saga-style compensation only where meaningful:** if Zoom creation succeeded but the session is then cancelled before publish completes, a compensating `zoom.session.cancel` is scheduled. We don't "undo" notifications, because sent messages can't be unsent.
- **Ordering dependencies inside multi-provider workflows:** e.g. "send new joining link" depends on "Zoom update succeeded". The reschedule workflow encodes this as node dependencies, so a Zoom failure blocks the notification (PRD Case 2), while a Calendar failure only goes to the retry queue (PRD Case 3).
- The Control Center shows a per-session **sync matrix** (Zoom ✓, Calendar ⟳, Sheet ✗) with a "Re-sync" button.

## 13. Source of truth and Google Sheets conflicts

See [ADR-003](../adr/003-postgresql-source-of-truth.md) and [ADR-011](../adr/011-google-sheets-sync-ownership.md). Summary: Postgres is master. Each sheet column has an owner. Rows are keyed by the `wos_registration_id` column. Conflicts are resolved by owner. Every overwrite is logged.

## 14. Manual overrides and audit history

- Override types: `mark_succeeded`, `mark_failed`, `skip`, `force_execute` (bypass dependencies), `resolve_unknown_outcome(delivered|resend)`, `change_participant_status`, `override_attendance`.
- Each override requires a **reason** (min 10 chars), produces an `audit_logs` row with before/after JSON, actor, IP, user agent and correlation ID, and appears in the entity's history timeline.
- `audit_logs` is append-only: the DB role used by the app has `INSERT, SELECT` only on that table. A periodic hash chain (`prev_hash`) gives tamper evidence (P2).

## 15. Recovery after server restarts

- Workers are stateless. Inngest re-delivers step calls that didn't complete. Memoized steps don't re-run.
- In-flight HTTP calls to providers that were cut by a restart look like **unknown outcome** to the next attempt. The attempt row exists with `started_at` but no `finished_at`, so the gateway applies the idempotency-class rule (ADR-012).
- `api` restart: the outbox relay resumes from `pending` rows. SSE clients reconnect with `Last-Event-ID` and refetch.
- Graceful shutdown: `enableShutdownHooks()`. On SIGTERM: stop accepting new requests, let in-flight step requests finish (≤ 30 s), close the DB pool. Container stop grace period is 45 s.
- **Startup reconciliation job** (on worker boot, also hourly): find `action_attempts` started > 15 min ago with no finish → classify per ADR-012.

## 16. Long-running workflow execution

- Bound wall-clock duration: activity runs sleep **≤ arming horizon (21 days)**. Workflow executions have `timeouts.finish` set (e.g. 7 days), and anything longer is redesigned as event-chained runs.
- Long waits use `step.waitForEvent` with an explicit timeout. On timeout → defined fallback branch.
- Long provider operations (e.g. recording download to R2, multi-GB) don't run inside a single HTTP step. Use streaming multipart upload in chunks with resumable state per part, or don't copy at all (use the Zoom share link) **[CONFIRM Q5]**.
- Plan maximum run length: Free 30 d / Pro 90 d / Business 366 d **[VERIFIED]**. The horizon keeps us inside every plan.

## 17. Race conditions and distributed locking

| Race | Resolution |
|---|---|
| Two runs claim the same activity | Conditional UPDATE on `(status, generation)`. Only one wins |
| Reschedule while a run is waking | Generation bump in the same TX as the new run insert. The old run's claim fails |
| Two admins edit the same activity | Optimistic locking: `version` column + `If-Match`. 409 on conflict with a diff |
| Concurrent sheet syncs on one binding | Inngest `concurrency { key: bindingId, limit: 1 }` |
| Token refresh by two workers | `SELECT … FOR UPDATE` on the credential row |
| Outbox relay on multiple instances | `FOR UPDATE SKIP LOCKED` |
| Rare global critical sections (e.g. "arm horizon" cron overlapping itself) | `pg_try_advisory_xact_lock(hashtext('arming-sweep'))` plus Inngest `concurrency limit 1` |
| Participant dedupe on concurrent imports | Unique indexes on `participants.email_normalized` and `phone_e164` (partial, where not null) + `INSERT … ON CONFLICT` |

Redis-based locks are unnecessary at this scale.

## 18. Data consistency and the transactional outbox

See [data-flow.md §3](../architecture/data-flow.md#3-transactional-outbox). Rules:

- A service method that changes state and must notify others **must** write outbox rows in the same transaction (helper `tx.emit(event)`).
- `inngest.send()` directly from a request handler is **forbidden**, except in the relay. Lint-enforced.
- Consumers tolerate duplicates and out-of-order delivery. They check entity state, not event order. Use `occurredAt` and entity `version` for staleness checks.

## 19. Execution monitoring and operational alerts

See [execution-monitoring.md](../features/execution-monitoring.md). Minimum alert set:

| Alert | Severity | Condition |
|---|---|---|
| `activity.overdue` | Critical | Scheduled run not claimed 5 min after `scheduled_for` |
| `action.dead_lettered` (critical activity) | Critical | Retries exhausted |
| `integration.auth_error` | Critical | Any |
| `integration.degraded` | High | Breaker open > 10 min |
| `outbox.stuck` | High | Pending > 5 min |
| `webhook.signature_failures` | Medium | > 20 / 10 min |
| `unknown_outcome.pending` | High | Any, during a live workshop |
| `sheet.sync_failed` | Medium | 3 consecutive failures |
| `zoom.daily_budget_low` | Medium | < 20 remaining for a host |

Delivery: in-app alert centre plus an **independent ops channel** (Slack incoming webhook or direct SMTP). This channel deliberately avoids the client's email automation, which might itself be the failing system **[CONFIRM Q13]**.

## 20. Cost, performance and infrastructure scaling

See [deployment.md §8](deployment.md#8-cost-model). Main levers:

- **Fan-out design drives Inngest cost.** Per-participant child runs cost roughly 2–4 steps each. Bulk triggers (if the comms system supports lists) reduce executions by 100×.
- DB: indexes on hot paths (§ in [database-design.md](database-design.md#5-indexes)). Partition `action_attempts` by month once it exceeds ~10 M rows.
- Vertical scaling first. Horizontal scaling per role later.

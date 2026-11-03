# ADR-007: Per-activity durable runs, generation fencing, and a Postgres sweeper

- Status: Accepted
- Date: 2026-11-03

## Context

The timeline holds hundreds of activities across concurrent workshops. Activities can be rescheduled, paused, skipped, disabled, or re-anchored at any time. The scheduling mechanism must:

- fire within seconds of the planned time
- never fire an obsolete (rescheduled or cancelled) version
- never fire twice
- survive restarts and deploys
- recover if Inngest misses something

## Options

| Option | Description | Problem |
|---|---|---|
| A. One long run per workshop | A single Inngest run walks the timeline, sleeping between activities | Any edit requires cancelling and restarting the whole run. Long runs exceed the Free-plan 30-day limit. One bad step stalls everything. Step limit 1,000 |
| B. Future-dated events (`ts`) | Send `activity.due` with `ts = scheduledAt` | A delayed event **is not a run**, so `cancelOn` cannot cancel it **[VERIFIED]**. Cancellation would depend entirely on fencing. Events are invisible in run UIs until due |
| **C. One run per activity generation** | On arm, send `activity.armed {activityId, generation}`. The run does `step.sleepUntil(scheduledAt)` with `cancelOn: activity.disarmed` matched on `data.activityId` | Many sleeping runs (fine). Requires a horizon for far-future dates |
| D. Pure DB polling cron | Every minute, select due activities | Up to 60 s jitter. Polling load. Still needs a queue for execution |

## Decision

**Option C + fencing + sweeper + horizon:**

1. **Arm**: when an activity becomes schedulable (workshop activated, activity created or rescheduled, dependency satisfied), a transaction:
   - increments `timeline_activities.schedule_generation`
   - inserts `activity_runs(activity_id, generation, status='scheduled', scheduled_for)`
   - writes outbox events `activity.disarmed {activityId, generation: old}` and `activity.armed {activityId, generation: new, runId}`
2. **Run function** `timeline/activity.execute`:
   - triggered by `activity.armed`
   - `cancelOn: [{ event: 'timeline/activity.disarmed', if: 'async.data.activityId == event.data.activityId && async.data.generation >= event.data.generation' }]`
   - `idempotency: 'event.data.runId'`
   - `step.sleepUntil(scheduledFor)`
   - `step.run('claim')`: `UPDATE activity_runs SET status='running', claimed_at=now() WHERE id=$runId AND status='scheduled' AND generation = (SELECT schedule_generation FROM timeline_activities WHERE id=$activityId) AND <workshop automation_state = 'active'>`. If 0 rows are affected, the run exits as a no-op. **This conditional update is the distributed lock.** No Redis is needed.
3. **Horizon**: only activities with `scheduled_for <= now() + ARMING_HORIZON` (default **21 days**) are armed. The daily cron `timeline/arming.sweep` arms activities that enter the horizon. This keeps every sleeping run within the shortest plan's maximum run length.
4. **Sweeper** (cron, every minute): finds `activity_runs` with `status='scheduled' AND scheduled_for < now() - interval '2 minutes'` and no recent heartbeat. It re-emits `activity.armed` with the **same runId** (deduped by function idempotency). It alerts if the same run is overdue twice.
5. **Watchdog** (inside the `api` role, `@nestjs/schedule`, independent of Inngest): alert-only. It raises a critical alert if any run is overdue by more than 5 minutes. It never executes, so it can never double-execute.

## Consequences

- ✅ Rescheduling is O(1) per activity: bump generation, disarm, arm.
- ✅ Duplicates are impossible past the claim step, whatever their cause (Inngest retry, sweeper re-emit, human "execute now").
- ✅ Survives restarts: sleeping state lives in Inngest, and truth lives in Postgres.
- ⚠️ Pause doesn't cancel runs. Runs wake, fail to claim because `automation_state != active`, and mark themselves `paused`. On resume, the engine re-arms only paused runs. See [timeline-engine.md](../features/timeline-engine.md#9-pause-resume-stop-emergency-stop).
- ⚠️ Fire precision depends on Inngest's scheduler, typically seconds. Second-level precision is not guaranteed **[ASSUMPTION]**. Activities are designed with minute-level tolerance.

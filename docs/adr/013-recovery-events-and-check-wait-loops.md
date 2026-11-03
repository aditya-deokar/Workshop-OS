# ADR-013: Recovery events, check-wait-recheck loops, and the retries rule

- Status: Accepted
- Date: 2026-11-03
- Amends: [ADR-007](007-timeline-scheduling-strategy.md)
- Source: gap analysis [G-01, G-02, G-03](../planning/open-questions-playbook.md#part-c-gap-analysis)

## Context

A review of ADR-007 and the timeline execution algorithm found three correctness problems:

1. **Sweeper vs idempotency.** `timeline-activity-execute` uses `idempotency: 'event.data.runId'`, which allows one run per key per 24 h **[VERIFIED]**. The sweeper re-emitted `activity.armed` with the same `runId`, so recovery was suppressed exactly when the original run was lost or stuck.
2. **Wait races.** `step.waitForEvent` only matches events received after the wait is registered **[ASSUMPTION A9]**. A fan-out whose children all finish before the parent starts waiting, or a domain event arriving between a DB check and the wait, would be missed until timeout.
3. **Retries.** The illustrative function used `retries: 0`, which removes Inngest retries from internal DB steps (`claim`, `finalize`). That contradicts the reliability rules.

## Decision

1. **Recovery uses a separate event and function.**
   - The sweeper atomically increments `activity_runs.recovery_attempts` (`UPDATE … WHERE status='scheduled' AND scheduled_for < now() - interval '2 min' RETURNING`) and emits `timeline/activity.recover { runId, recoveryAttempt }` with event id `runId:recover:n`.
   - `timeline-activity-recover` has `idempotency: 'event.data.runId + ":" + string(event.data.recoveryAttempt)'` and no sleep. It goes straight to the claim step and shares `ActivityExecutor`.
   - The claim's conditional UPDATE remains the only fence. An original run that wakes later fails its claim and no-ops.
   - `recovery_attempts >= 2` and still unclaimed → critical alert `activity.unrecoverable`.
2. **Every wait is a check-wait-recheck loop.** Postgres is checked first. The event wait is chunked (5 min for fan-out completion, 15 min for event-mode listeners) and followed by a re-check, until done or the deadline passes. The event only speeds things up. Correctness comes from the DB.
3. **Retries rule.**
   - Functions keep Inngest retries (default 4) for unexpected exceptions.
   - Provider attempts **never throw for classified failures**. The gateway records the attempt and returns `{ outcome }`. Our explicit attempt loop (`action:{id}:attempt:{n}` + `step.sleep(backoff)`) applies the retry policy. Inngest therefore never double-retries a provider call.
   - `onFailure` marks the owning run `needs_attention`.

## Consequences

- ✅ The sweeper actually recovers lost runs. Duplicate execution is still impossible (claim fence).
- ✅ No missed completions or events due to timing.
- ⚠️ A few more steps per run (bounded: ~50 for a 2 h deadline).
- Schema: `activity_runs.recovery_attempts int not null default 0`, `last_recovery_at timestamptz`.

## Verification

- Integration test: cancel the original armed run in the Dev Server → recovery executes within 2 min, exactly once.
- Integration test: fan-out of 3 recipients that complete in < 1 s → parent finalizes within one check cycle (no timeout).
- Integration test: transient DB error injected into `claim` → Inngest retries → run proceeds.

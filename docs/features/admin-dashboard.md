# Admin Dashboard

Related: [execution-monitoring.md](execution-monitoring.md) · [timeline-engine.md](timeline-engine.md) · [automation-engine.md](automation-engine.md) · [ADR-009](../adr/009-react-vite-spa-instead-of-nextjs.md) · [ADR-010](../adr/010-realtime-sse-with-postgres-notify.md) · [security.md](../engineering/security.md)

---

## 1. Purpose and scope

One interface to configure, run and monitor every workshop. The admin must be able to **control automation while the workshop is running**, not only configure it (PRD §7).

## 2. Navigation (PRD §11)

```
Workshop OS
├── Overview                 Today's workshops, live sessions, alerts, attention items
├── Workshops                List, create, duplicate; workshop detail (tabs below)
├── Timeline & Schedule      Cross-workshop calendar/agenda view; per-workshop timeline builder
├── Automation Studio        Workflows (React Flow), versions, executions
├── Participants             Search, registrations, review queue
├── Zoom Sessions            Sessions × Zoom status, drift, host start, recordings link
├── Recordings & Resources   Uploads, publishing state
├── Integrations             Zoom, WhatsApp system, Email system, Google; triggers registry; sheet bindings; health
├── Execution Monitor        All action executions, dead letter, unknown outcomes, bulk ops
├── Reports & Documents      Workshop summary, attendance, completion, exports, generated docs (P2)
├── Activity & Audit Logs    Audit trail, filters, entity history
└── Settings                 Users & roles, defaults (retry, missed policy, completion rule), ops alert channel
```

WhatsApp/email have **no campaign modules**. They're configured under Integrations, and their executions are visible in the Execution Monitor (PRD §11).

## 3. Key screens

### 3.1 Overview

- Cards: live sessions now, activities in the next 2 h, open critical alerts, items needing attention, integration health strip (Zoom ● WA ● Email ● Google ●).
- "Today" agenda across all workshops (workshop timezone shown when it differs from the viewer's).

### 3.2 Workshop detail (tabs)

`Summary · Structure (days/sessions) · Timeline · Participants · Automations · Integrations sync · Executions · Audit`

### 3.3 Timeline builder

- Columns per day. Vertical time axis (5-min snap). Sessions as blocks. Activities as chips anchored to sessions or absolute times.
- **dnd-kit** for dragging and reordering. Dragging an anchored activity changes its **offset** (the UI shows "−15 min before Session 2 start"). Dragging an absolute activity changes `local_time`.
- Dependency lines (SVG overlay) between chips. A "sequential" chip shows estimated time (dashed).
- Side panel: activity form (type-specific, Zod-driven), validation messages, run history.
- On a published workshop, every drag results in a **pending change set** with an impact preview → "Apply changes" (reschedule path). Nothing is applied on drop.
- Keyboard accessible: arrow keys to move by 5 min, Enter to open, and an ARIA live region for drag announcements (dnd-kit accessibility support).

### 3.4 Workshop Control Center (live)

Layout (from PRD §7 mock):

```
┌ Medical Coding Fundamentals · Day 1 ─────────────── [In Progress] [Automation: Active ●] ┐
│ 248 Participants │ 182 Attendance confirmed │ 12 Completed activities │ 2 Require attention │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ CURRENT  Session 2 — ICD-10-CM            Zoom: Live (webhook 10:31)   [Start as host]   │
│ NEXT     Assessment · 12:00 PM IST (in 47 min)                                           │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Timeline (today)  ✓ 08:00 Reminder (246/248)  ✓ 08:45 Joining info  ● 10:30 Session 2 … │
│ Sync matrix       Zoom ✓   Calendar ⟳ retrying   Sheet ✗ failed (retry)                  │
│ Attention         ⚠ 2 unknown outcomes (WhatsApp)  ⚠ Sheet sync failed 3×                 │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ [Pause automation] [Resume] [Reschedule…] [Stop] [Emergency stop]                        │
│ ⓘ Pausing automation does not pause or end Zoom sessions already in progress.           │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

- Live via SSE. The counter "Attendance confirmed" uses webhook-provisional attendance during the session, labeled "live (provisional)".
- Every activity row has a contextual menu: **Execute now · Skip · Retry · Edit · Override · View executions**.

### 3.5 Controls and expected behavior (PRD §7)

| Control | Where | Confirmation | Permission | Behavior (engine doc) |
|---|---|---|---|---|
| Start | Workshop header | Preview: activities to be armed | `automation.control` | Activate automation ([timeline §6](timeline-engine.md#6-scheduling-strategy)) |
| Pause | Header | Optional reason | `automation.control` | Suspend eligible future executions. External sessions unaffected ([§9](timeline-engine.md#9-pause-resume-stop-emergency-stop)) |
| Resume | Header | Preview of missed items + policy choice | `automation.control` | Continue paused automation |
| Stop | Header | Reason required | `automation.control` | Stop future executions |
| Reschedule | Header / session / activity | Impact preview mandatory | `timeline.reschedule` | [timeline §11](timeline-engine.md#11-rescheduling) |
| Skip | Activity | Reason required | `executions.override` | Skip a specific activity |
| Retry | Activity run / execution | "Failed only" vs "all" | `executions.retry` | Retry a failed action |
| Execute now | Activity | Reason. "Bypass dependencies" checkbox (admin) | `executions.execute_now` | Manual execution (same claim/idempotency path) |
| Edit | Activity / workflow | Preview if published | `timeline.write` / `automation.write` | Change configuration |
| Duplicate | Workshop / workflow | — | `workshops.write` / `automation.write` | |
| Emergency stop | Header (red) | Type the workshop code to confirm + reason + separate checkbox "Also end live Zoom sessions" | `automation.emergency` | Halt all eligible automated actions |
| Manual override | Activity run | Reason + outcome | `executions.override` | Mark manually completed |

All controls are idempotent (Idempotency-Key) and audited. Disabled states explain *why* (e.g. "Activity is running. Stop it first").

### 3.6 Integrations

- Cards per integration: status, capabilities (with license notes), last health check, breaker state, "Test", "Reconnect", "Rotate secret".
- Comms triggers registry with verification status ([whatsapp-email §6](whatsapp-email-integration.md#6-trigger-registry)).
- Sheet bindings: column-map editor with owner toggles (inbound/outbound/inbound-once), preview diff.

### 3.7 Execution Monitor

See [execution-monitoring.md §5](execution-monitoring.md#5-execution-monitor-ui).

## 4. Frontend architecture

| Concern | Approach |
|---|---|
| Routing | TanStack Router. URL holds filters (shareable links) |
| Data | TanStack Query. Query keys per resource. SSE events invalidate keys |
| Forms | react-hook-form + Zod resolvers from `@wos/contracts` |
| Components | shadcn/ui (Radix primitives) + Tailwind v4 |
| Tables | TanStack Table with server-side pagination/filtering |
| Timeline | dnd-kit + custom time grid |
| Workflow editor | React Flow + DSL mapper |
| Time display | Luxon. Always workshop timezone + viewer tz tooltip |
| Auth | Better Auth client (cookie session). Role-aware UI (hide or disable by permission) |
| Errors | Sentry React SDK. Error boundary per route |
| A11y | WCAG 2.1 AA for core flows. Keyboard-operable timeline |

The UI never computes schedules authoritatively. It renders server-computed instants and uses server previews for impact.

## 5. Security

- The server enforces every permission. The UI hides controls as a convenience only.
- Destructive actions require typed confirmation. Emergency stop requires the workshop code.
- Session timeout 12 h idle (configurable). 2FA required for owner/admin.

## 6. Edge cases

- Viewer in a different timezone → primary display in the workshop tz, with a badge "times shown in IST".
- SSE disconnected → banner "Live updates paused, reconnecting" and fallback polling every 10 s.
- Concurrent edits → 409 with a diff dialog ("Reload and reapply").

## 7. Testing

- Playwright E2E: create workshop → build timeline → publish (mocked adapters) → Control Center pause/resume → reschedule with preview → Execution Monitor retry.
- Component tests (Vitest + Testing Library) for the activity form, condition builder and column-map editor.
- Visual regression (P2) on the Control Center.
- Accessibility checks (axe) in CI for main routes.

## 8. Acceptance criteria

1. Every control in §3.5 exists, enforces its permission, requires the specified confirmation, and is audited.
2. The Control Center reflects an activity status change within 2 s (SSE).
3. The pause UI explicitly states that live Zoom sessions are unaffected.
4. The timeline builder supports drag/keyboard editing, and published-workshop changes always go through preview.

## 9. Implementation dependencies

Auth → API contracts → workshop CRUD screens → timeline builder → Control Center (needs SSE + timeline engine) → Integrations screens → Execution Monitor → Automation Studio → Reports.

## 10. Trade-offs

- An SPA without SSR is simpler to operate, with a slower first load (irrelevant for internal staff).
- Change sets with preview for published workshops add one extra step, but prevent accidental mass rescheduling.

## 11. Future improvements

- Mobile-friendly Control Center for operators on the go.
- Command palette (⌘K) for quick actions.
- Saved views and dashboards per role.

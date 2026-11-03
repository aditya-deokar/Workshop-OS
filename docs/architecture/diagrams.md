# Architecture Diagrams

All diagrams are Mermaid and render on GitHub and in most Markdown viewers. Feature documents embed the relevant diagrams. This page collects them in one place.

Contents:
1. [Overall system architecture](#1-overall-system-architecture)
2. [Frontend ↔ backend communication](#2-frontend--backend-communication)
3. [Workshop lifecycle](#3-workshop-lifecycle)
4. [Timeline execution engine](#4-timeline-execution-engine)
5. [Automation workflow execution](#5-automation-workflow-execution)
6. [Reminder execution sequence](#6-reminder-execution-sequence)
7. [Integration architecture](#7-integration-architecture)
8. [Event and webhook processing](#8-event-and-webhook-processing)
9. [Database ER diagram](#9-database-er-diagram)
10. [Failure handling and retry flow](#10-failure-handling-and-retry-flow)
11. [Workshop rescheduling](#11-workshop-rescheduling)
12. [Deployment architecture](#12-deployment-architecture)
13. [Sequence: registration](#13-sequence-registration)
14. [Sequence: workshop publish & activation](#14-sequence-workshop-publish--activation)
15. [Sequence: Zoom session creation](#15-sequence-zoom-session-creation)
16. [Sequence: attendance synchronization](#16-sequence-attendance-synchronization)
17. [Sequence: workshop completion](#17-sequence-workshop-completion)
18. [Execution state machine](#18-execution-state-machine)

---

## 1. Overall system architecture

```mermaid
flowchart TB
  Admin((Admin / Operator)) --> SPA["React + Vite SPA"]
  SPA -->|REST + SSE, cookie session| API["NestJS api role"]
  API --> PG[(PostgreSQL)]
  API -->|outbox relay| ING[Inngest Cloud]
  ING -->|signed HTTPS step calls| WRK["NestJS worker role"]
  WRK --> PG
  WRK --> GW[Integration gateway]
  GW --> ZOOM[Zoom API]
  GW --> WA[Existing WhatsApp automation]
  GW --> EM[Existing email automation]
  GW --> GS[Google Sheets]
  GW --> GC[Google Calendar]
  GW --> GD[Google Docs/Drive]
  GW --> R2[(R2 / S3)]
  ZOOM -->|webhooks| API
  WA -->|delivery callbacks| API
  EM -->|delivery callbacks| API
  API -. traces/errors .-> OBS[OTel collector + Sentry]
  WRK -. traces/errors .-> OBS
```

## 2. Frontend ↔ backend communication

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser (SPA)
  participant P as Reverse proxy
  participant A as NestJS api
  participant PG as PostgreSQL
  B->>P: POST /api/auth/sign-in/email
  P->>A: (no body parser) Better Auth
  A->>PG: verify, create session
  A-->>B: Set-Cookie (HttpOnly, Secure, SameSite=Lax)
  B->>P: GET /api/v1/workshops?status=published
  P->>A: SessionGuard → RBAC → controller
  A->>PG: query
  A-->>B: 200 JSON (Zod-validated contract)
  B->>P: GET /api/v1/stream?topics=workshop:wks_1 (EventSource)
  P->>A: SSE (buffering off)
  loop on commit
    PG-->>A: NOTIFY wos_events {type, ids}
    A-->>B: event: activity.updated data: {id,status}
    B->>A: refetch affected query (TanStack Query invalidate)
  end
  B->>A: PATCH /api/v1/activities/act_9 (If-Match: 7, Idempotency-Key)
  A-->>B: 409 version_conflict | 200 updated
```

## 3. Workshop lifecycle

```mermaid
stateDiagram-v2
  [*] --> draft
  draft --> ready: validation passes
  ready --> draft: edited (invalidates)
  ready --> published: publish (provision Zoom, Calendar, Sheet)
  published --> in_progress: first session start time reached / manual start
  published --> cancelled: cancel
  in_progress --> completed: last session ended + completion workflow done
  in_progress --> cancelled: cancel (emergency)
  completed --> archived: archive
  cancelled --> archived: archive
  archived --> [*]

  state "automation_state (orthogonal)" as AS {
    [*] --> inactive
    inactive --> active: activate
    active --> paused: pause
    paused --> active: resume
    active --> stopped: stop / emergency stop
    paused --> stopped: stop
    stopped --> active: re-activate (re-arm remaining)
  }
```

## 4. Timeline execution engine

```mermaid
flowchart TB
  subgraph Plan["Plan (Postgres)"]
    A1[timeline_activities<br/>anchor + offset + tz]
    D1[activity_dependencies]
  end
  P1["TimelinePlanner<br/>compute scheduled_for (UTC)"] --> A1
  A1 --> ARM{"Arm?<br/>enabled ∧ workshop active ∧<br/>deps satisfied/none ∧ within horizon"}
  ARM -- yes --> TX["TX: generation++ ,<br/>INSERT activity_runs(scheduled),<br/>outbox activity.armed"]
  ARM -- "no (waiting on deps)" --> W[status = waiting]
  TX --> F["Inngest timeline-activity-execute"]
  F --> S["step.sleepUntil(scheduled_for)"]
  S --> C{"step: claim<br/>conditional UPDATE"}
  C -- "0 rows (stale/paused/dup)" --> NOOP[exit no-op / mark paused]
  C -- claimed --> PRE{"preconditions<br/>(success condition of deps,<br/>capabilities, workshop state)"}
  PRE -- fail --> NA[needs_attention / skipped per policy]
  PRE -- ok --> EXE["execute action(s) via gateway<br/>(fan-out per participant if needed)"]
  EXE --> RES{"result"}
  RES -- succeeded --> DONE[activity_run succeeded]
  RES -- "failed (non-critical)" --> FAILNC[failed → alert, continue timeline]
  RES -- "failed (critical)" --> FAILC[needs_attention → block dependents]
  DONE & FAILNC & FAILC --> EVT["outbox activity.completed {outcome}"]
  EVT --> DR["Dependency resolver:<br/>arm dependents whose deps now satisfied"]
  DR --> ARM
  SW["Sweeper cron 1m:<br/>overdue scheduled runs → re-emit"] -.-> F
  HZ["Arming cron daily:<br/>activities entering horizon"] -.-> ARM
```

## 5. Automation workflow execution

```mermaid
flowchart TB
  E["Domain event<br/>e.g. registration.closed"] --> M{"workflow_triggers match?<br/>(event name + filter)"}
  M -- no --> X[ignore]
  M -- yes --> WE["INSERT workflow_executions<br/>(version pinned, idempotency = trigger event id + workflow id)"]
  WE --> RUN["Inngest automation-workflow-run<br/>(loads immutable version)"]
  RUN --> N["Interpreter: next node(s)"]
  N --> T{node type}
  T -- action --> ACT["gateway.call in step.run<br/>record workflow_step_executions"]
  T -- condition --> CND["evaluate condition AST in step.run<br/>(reads Postgres)"] --> BR{branch}
  T -- delay --> DL["step.sleep / sleepUntil"]
  T -- wait_for_event --> WFE["step.waitForEvent (timeout)"]
  T -- fan_out --> FO["step.sendEvent ×N → child runs<br/>(per participant)"]
  T -- end --> END[execution succeeded]
  ACT & BR & DL & WFE & FO --> N
  ACT -- "error, policy=stop" --> FAIL[execution failed]
  ACT -- "error, policy=continue" --> N
  ACT -- "error, policy=branch" --> EB[error edge] --> N
```

## 6. Reminder execution sequence

```mermaid
sequenceDiagram
  autonumber
  participant I as Inngest
  participant W as worker: timeline-activity-execute
  participant PG as PostgreSQL
  participant C as worker: comms-send (child)
  participant WA as Existing WhatsApp system
  I->>W: wake at 08:00 IST (02:30Z)
  W->>PG: step claim: UPDATE activity_runs … WHERE generation matches AND automation active
  PG-->>W: 1 row (claimed)
  W->>PG: step resolve recipients (registrations status=confirmed, not already sent)
  W->>I: step.sendEvent comms/send.requested ×248 (id = runId:registrationId)
  par per participant (throttled by integration)
    I->>C: run (idempotency = runId:registrationId)
    C->>PG: INSERT action_executions ON CONFLICT DO NOTHING
    C->>WA: POST trigger workshop_reminder (Idempotency-Key, HMAC)
    WA-->>C: 202 {executionId}
    C->>PG: attempt succeeded, action accepted
  end
  W->>W: step.waitForEvent / poll aggregation (all children terminal or timeout 30m)
  W->>PG: activity_run succeeded (246 ok, 2 needs_attention)
  W->>PG: outbox activity.completed
  WA-->>PG: (async) callbacks delivered/failed → action_executions updated
```

## 7. Integration architecture

```mermaid
flowchart LR
  Engines["Timeline / Automation"] --> GW["IntegrationGateway"]
  GW --> REG[(Action registry)]
  GW --> AE[(action_executions / attempts)]
  GW --> PORTS{{Ports}}
  PORTS --> MP[MeetingProvider] --> ZA[Zoom adapter] --> ZAPI[(Zoom REST)]
  PORTS --> CT[CommunicationTrigger] --> CA["Webhook comms adapter<br/>(per configured system)"] --> COMMS[(Existing WA / Email)]
  PORTS --> SS[SpreadsheetSync] --> GSA[Sheets adapter] --> GSAPI[(Sheets API v4)]
  PORTS --> CS[CalendarSync] --> GCA[Calendar adapter] --> GCAPI[(Calendar API v3)]
  PORTS --> DG[DocumentGenerator] --> GDA[Docs adapter] --> GDAPI[(Docs + Drive API)]
  ZA & CA & GSA & GCA & GDA --> VAULT[(Credential vault)]
  ZA & CA & GSA & GCA & GDA --> RL[Rate limiter / breaker]
```

## 8. Event and webhook processing

```mermaid
sequenceDiagram
  autonumber
  participant Z as Zoom
  participant A as api /webhooks/zoom
  participant PG as PostgreSQL
  participant R as Outbox relay
  participant I as Inngest
  participant P as worker: webhooks-process
  Z->>A: POST meeting.ended (x-zm-signature, x-zm-request-timestamp)
  A->>A: HMAC(v0:ts:rawBody) == signature? |now-ts| < 5 min?
  A->>PG: TX: INSERT webhook_events (dedupe_key) + outbox webhook.received
  A-->>Z: 200 (< 3 s)
  R->>I: send webhook.received {webhookEventId}
  I->>P: run (concurrency key = provider)
  P->>PG: step load event, map meeting id → session
  P->>PG: step TX: session.delivery_status=ended, outbox zoom.session.ended
  P->>PG: mark webhook_events processed
  Note over P: zoom.session.ended may complete event-mode activities<br/>and trigger attendance retrieval after a delay
```

## 9. Database ER diagram

Core relationships (full column definitions: [database-design.md](../engineering/database-design.md)).

```mermaid
erDiagram
  USERS ||--o{ STAFF_ROLES : has
  USERS ||--o{ AUDIT_LOGS : performs
  WORKSHOPS ||--o{ WORKSHOP_DAYS : contains
  WORKSHOP_DAYS ||--o{ SESSIONS : contains
  WORKSHOPS ||--o{ TIMELINE_ACTIVITIES : schedules
  SESSIONS ||--o{ TIMELINE_ACTIVITIES : "anchors (optional)"
  TIMELINE_ACTIVITIES ||--o{ ACTIVITY_DEPENDENCIES : "depends on"
  TIMELINE_ACTIVITIES ||--o{ ACTIVITY_RUNS : "executes as"
  ACTIVITY_RUNS ||--o{ ACTION_EXECUTIONS : performs
  ACTION_EXECUTIONS ||--o{ ACTION_ATTEMPTS : "attempted as"
  SESSIONS ||--o{ SESSION_TRAINERS : staffed_by
  TRAINERS ||--o{ SESSION_TRAINERS : assigned
  SESSIONS ||--o| ZOOM_SESSIONS : "delivered via"
  SESSIONS ||--o{ RESOURCES : publishes
  SESSIONS ||--o{ RECORDINGS : produces
  PARTICIPANTS ||--o{ REGISTRATIONS : enrolls
  WORKSHOPS ||--o{ REGISTRATIONS : has
  REGISTRATIONS ||--o{ ATTENDANCE_RECORDS : "attends (raw)"
  REGISTRATIONS ||--o{ SESSION_ATTENDANCE : "attends (computed)"
  SESSIONS ||--o{ SESSION_ATTENDANCE : measured_in
  AUTOMATION_WORKFLOWS ||--o{ WORKFLOW_VERSIONS : versions
  WORKFLOW_VERSIONS ||--o{ WORKFLOW_TRIGGERS : declares
  WORKFLOW_VERSIONS ||--o{ WORKFLOW_EXECUTIONS : runs
  WORKFLOW_EXECUTIONS ||--o{ WORKFLOW_STEP_EXECUTIONS : steps
  WORKFLOW_STEP_EXECUTIONS ||--o{ ACTION_EXECUTIONS : performs
  INTEGRATIONS ||--o{ INTEGRATION_CREDENTIALS : secured_by
  INTEGRATIONS ||--o{ COMMUNICATION_TRIGGERS : exposes
  INTEGRATIONS ||--o{ ACTION_EXECUTIONS : used_by
  INTEGRATIONS ||--o{ SHEET_BINDINGS : hosts
  WORKSHOPS ||--o{ SHEET_BINDINGS : "synced to"
  SHEET_BINDINGS ||--o{ SHEET_ROW_LINKS : rows
  REGISTRATIONS ||--o| SHEET_ROW_LINKS : "mirrored as"
  SESSIONS ||--o{ CALENDAR_EVENT_LINKS : "projected as"
  DOCUMENT_TEMPLATES ||--o{ GENERATED_DOCUMENTS : produces
  WORKSHOPS ||--o{ GENERATED_DOCUMENTS : has
  WEBHOOK_EVENTS }o--|| INTEGRATIONS : "received from"
  OUTBOX_EVENTS }o--o| WORKSHOPS : "about"
  ALERTS }o--o| ACTION_EXECUTIONS : "raised by"
```

## 10. Failure handling and retry flow

```mermaid
flowchart TB
  START[gateway.call] --> SEND[attempt n]
  SEND --> OK{response}
  OK -- 2xx --> S[succeeded]
  OK -- "202 accepted" --> ACC["accepted → await callback<br/>(timeout → needs_attention or assume per config)"]
  OK -- "429 / 5xx / pre-send network" --> T[transient]
  OK -- "401 / invalid_grant" --> AU[auth_error → integration banner → needs_attention]
  OK -- "4xx validation / 404" --> PM[permanent → failed]
  OK -- "timeout after send" --> UO{idempotency class}
  UO -- "natural / keyed" --> T
  UO -- checkable --> CHK[lookup prior attempt] --> FOUND{found?}
  FOUND -- yes --> S
  FOUND -- no --> T
  UO -- unsafe --> NA["needs_attention (unknown_outcome)<br/>admin: mark delivered / resend"]
  T --> RL{attempts < max?}
  RL -- yes --> BO["backoff: min(base·2^n + jitter, cap)<br/>honor Retry-After"] --> SEND
  RL -- no --> DLQ["failed + dead_lettered<br/>alert (severity by criticality)"]
  PM --> CRIT{activity critical?}
  DLQ --> CRIT
  CRIT -- yes --> BLOCK[block dependent activities<br/>session needs_attention]
  CRIT -- no --> CONT[continue timeline]
  DLQ --> MAN["Execution Monitor:<br/>retry · skip · override"]
```

## 11. Workshop rescheduling

```mermaid
sequenceDiagram
  autonumber
  actor Ad as Admin
  participant A as api
  participant PG as PostgreSQL
  participant I as Inngest
  participant W as worker: reschedule workflow
  participant Z as Zoom
  participant G as Google Calendar
  participant C as Existing WA/Email
  Ad->>A: PATCH session 2 → 11:00 (If-Match v7)
  A->>A: preview impact (dry run): affected activities, external updates, notifications
  Ad->>A: confirm (reason, notify participants = yes)
  A->>PG: TX: update session, recompute anchored activities, generation++,<br/>cancel old runs, insert new runs, outbox (disarmed×N, armed×N, session.rescheduled), audit
  A-->>Ad: 200 (new plan)
  I->>I: cancelOn disarmed → old sleeping runs cancelled
  I->>W: session.rescheduled
  W->>Z: step PATCH /meetings/{id} start_time (counts toward 100/day/user limit)
  W->>G: step events.patch(deterministic id)
  W->>C: step trigger schedule_change (fan-out per registration)
  W->>PG: record all action executions
  Note over W: Zoom PATCH failure → session needs_attention,<br/>dependent "joining instructions" blocked; Calendar failure → retry queue only
```

## 12. Deployment architecture

```mermaid
flowchart TB
  subgraph Internet
    U[Admins]
    EXT[Zoom / Comms systems / Google]
    INNG[Inngest Cloud]
  end
  subgraph Cloud["Cloud region (e.g. ap-south-1) [CONFIRM Q16]"]
    CDN["CDN / static hosting<br/>SPA assets"]
    LB["Load balancer + WAF<br/>TLS, HTTP/2, SSE no-buffer"]
    subgraph Containers["Container service"]
      API1[api #1]
      API2[api #2]
      W1[worker #1]
      W2[worker #2]
    end
    DB[("Managed PostgreSQL<br/>primary + standby, PITR")]
    SEC[(Secrets manager / KMS)]
    OT[OTel collector]
  end
  R2[(Cloudflare R2)]
  SEN[Sentry]
  U --> CDN
  U --> LB
  EXT --> LB
  INNG --> LB
  LB -->|/api/v1, /api/auth, /webhooks| API1 & API2
  LB -->|/api/inngest| W1 & W2
  API1 & API2 & W1 & W2 --> DB
  API1 & API2 --> INNG
  Containers --> SEC
  Containers --> R2
  Containers --> OT --> SEN
```

## 13. Sequence: registration

```mermaid
sequenceDiagram
  autonumber
  participant SRC as Source (Sheet / intake API / admin)
  participant A as api or worker (reconcile)
  participant PG as PostgreSQL
  participant I as Inngest
  participant AE as worker: Registration workflow
  participant WA as WhatsApp system
  participant EM as Email system
  participant GS as Google Sheet
  SRC->>A: new registration (Idempotency-Key / row hash)
  A->>A: normalize email (lowercase), phone (E.164)
  A->>PG: find participant by email OR phone
  alt exact match
    A->>PG: reuse participant
  else ambiguous (email matches A, phone matches B)
    A->>PG: create registration status=needs_review, alert
  else none
    A->>PG: insert participant
  end
  A->>PG: TX: insert registration (unique workshop+participant) + outbox registration.created
  I->>AE: registration.created
  AE->>PG: step check payment status (if required) [CONFIRM Q4]
  alt paid / not required
    AE->>WA: trigger registration_confirmation
    AE->>EM: trigger registration_confirmation
  else unpaid
    AE->>PG: registration=pending_payment, notify admin
  end
  opt meeting requires Zoom registration
    AE->>AE: zoom.registrant.add → per-registrant join_url stored
  end
  AE->>GS: (debounced) batchUpdate row status + wos id
```

## 14. Sequence: workshop publish & activation

```mermaid
sequenceDiagram
  autonumber
  actor Ad as Admin
  participant A as api
  participant PG as PostgreSQL
  participant W as worker: workshop-publish
  participant Z as Zoom
  participant G as Calendar
  participant S as Sheets
  Ad->>A: POST /workshops/{id}/publish {activate: true}
  A->>A: validate: sessions complete, tz set, activities valid,<br/>capabilities present, comms triggers registered, no cycles
  alt blocking issues
    A-->>Ad: 422 {issues[] (blocking/warning)}
  else ok or override given
    A->>PG: TX: status=publishing, plan snapshot, outbox workshop.publish.requested, audit
    A-->>Ad: 202 {publishJobId}
  end
  W->>Z: per session: create meeting/webinar (idempotent via zoom_sessions row)
  W->>G: per session: upsert event (deterministic id)
  W->>S: ensure sheet binding headers / create sheet [optional]
  W->>PG: per-step action executions
  alt any critical provisioning failed
    W->>PG: status=published_with_issues, sessions needs_attention, alert
  else
    W->>PG: status=published
  end
  W->>PG: if activate: automation_state=active, arm activities within horizon (outbox armed×N)
  W->>PG: outbox workshop.ready → admin notification
```

## 15. Sequence: Zoom session creation

```mermaid
sequenceDiagram
  autonumber
  participant W as worker
  participant PG as PostgreSQL
  participant ZA as Zoom adapter
  participant Z as Zoom API
  W->>PG: step: SELECT zoom_sessions WHERE session_id FOR UPDATE
  alt row has zoom_id
    W->>ZA: getSession(zoom_id) → reconcile settings (PATCH if desired_hash differs)
  else no row / no zoom_id
    W->>PG: INSERT zoom_sessions(status=creating, correlation_tag)
    W->>ZA: token (S2S OAuth, cached 1h)
    ZA->>Z: POST /users/{hostId}/meetings|webinars {topic, start_time, timezone, duration, settings, agenda contains wos tag}
    alt 201
      Z-->>ZA: {id, uuid, join_url, start_url, registration_url?}
      ZA-->>W: normalized (start_url discarded)
      W->>PG: UPDATE zoom_sessions zoom_id, join_url, status=active
    else timeout after send (checkable)
      W->>ZA: list host's upcoming meetings, find by wos tag
      ZA-->>W: found → adopt / not found → retry
    else 4xx license/validation
      W->>PG: session needs_attention (block dependent joining instructions)
    end
  end
```

## 16. Sequence: attendance synchronization

```mermaid
sequenceDiagram
  autonumber
  participant Z as Zoom
  participant A as api webhooks
  participant I as Inngest
  participant W as worker: attendance-sync
  participant PG as PostgreSQL
  participant GS as Sheets
  Z->>A: meeting.participant_joined / left (realtime, best effort)
  A->>PG: inbox → provisional attendance_records (source=webhook)
  Z->>A: meeting.ended
  A->>I: zoom.session.ended
  I->>W: attendance-sync {sessionId}
  W->>W: step.sleep 30m (report data settles) [ASSUMPTION: delay tuned in SPIKE S3]
  W->>Z: step GET report participants (paginated, next_page_token)
  W->>PG: step upsert attendance_records(source=report), match → registration
  W->>PG: step compute session_attendance (merge intervals, % of scheduled duration)
  W->>PG: unmatched attendees → review queue
  W->>PG: outbox attendance.imported
  W->>GS: debounced outbound sync (attendance column)
  opt report not yet available / empty
    W->>W: retry with backoff up to 24h, then needs_attention
  end
```

## 17. Sequence: workshop completion

```mermaid
sequenceDiagram
  autonumber
  participant I as Inngest
  participant W as worker: completion workflow
  participant PG as PostgreSQL
  participant D as Docs adapter
  participant C as Comms systems
  participant GS as Sheets
  I->>W: workshop.sessions_all_ended
  W->>W: step.waitForEvent attendance.imported for all sessions (timeout 24h)
  W->>PG: step compute completion per registration (rule: attendance ≥ threshold, assessments?) [CONFIRM Q6]
  W->>PG: step flag borderline cases for manual review
  W->>D: step generate attendance report / completion report (post-MVP)
  W->>C: step fan-out: completion / feedback / certificate triggers (eligible only)
  W->>GS: step final export
  W->>PG: workshop status=completed, outbox workshop.completed
  W->>PG: alert admin: N participants need review
```

## 18. Execution state machine

Applies to `activity_runs`, `workflow_executions` and `action_executions`. Not every entity uses every state.

```mermaid
stateDiagram-v2
  [*] --> pending
  pending --> scheduled: armed
  pending --> waiting: dependencies unmet
  waiting --> scheduled: dependencies satisfied
  scheduled --> running: claimed
  scheduled --> paused: workshop paused
  paused --> scheduled: resumed (re-armed)
  scheduled --> cancelled: superseded / stopped
  running --> succeeded
  running --> retrying: transient failure
  retrying --> running
  retrying --> failed: retries exhausted
  running --> failed: permanent failure
  running --> needs_attention: unknown outcome / auth / critical
  running --> awaiting_confirmation: manual task
  awaiting_confirmation --> succeeded: confirmed
  awaiting_confirmation --> failed: rejected / timeout
  needs_attention --> running: retry
  needs_attention --> succeeded: manual override
  needs_attention --> skipped: skip with reason
  failed --> running: manual retry
  failed --> skipped: skip
  scheduled --> skipped: skip
  succeeded --> [*]
  skipped --> [*]
  cancelled --> [*]
```

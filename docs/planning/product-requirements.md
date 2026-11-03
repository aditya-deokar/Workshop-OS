# Product Requirements: Workshop OS

Source: [`/Plan/prd.md`](../../Plan/prd.md). This document consolidates it into numbered, testable requirements. Feature documents refer to these IDs.

Related: [README](../README.md) · [roadmap.md](roadmap.md) · [open-questions.md](open-questions.md)

---

## 1. Problem statement

The client runs multi-day online workshops (e.g. "Medical Coding Fundamentals", 248 participants, IST). Operations are spread across existing WhatsApp and email automations, Zoom, Google Sheets, Calendar and Docs. They are coordinated by hand. This causes missed reminders, wrong joining links after schedule changes, manual attendance reconciliation, and no single view of what happened.

## 2. Product vision

One control center that **owns the workshop plan and its execution state**, triggers the systems the client already has, and gives admins live control and full visibility.

## 3. Users and roles

| Persona | Goals | Role |
|---|---|---|
| **Owner** (business lead) | Oversight, integrations, user management | `owner` |
| **Workshop admin** | Build workshops, timelines, workflows; run live operations | `admin` |
| **Operator** (support staff) | Monitor, retry, resolve participant issues, confirm manual tasks | `operator` |
| **Viewer** (management / trainer) | Read-only dashboards and reports | `viewer` |
| **Trainer** (optional) | See own sessions, confirm manual tasks for own sessions | `trainer` **[CONFIRM Q9]** |

Participants never log in to Workshop OS in the MVP.

## 4. Scope boundaries

**Not building**: WhatsApp bulk messaging, email marketing/templates/campaigns, CRM, multi-tenant SaaS, payment processing, video hosting.

**Assumption**: the existing WhatsApp/email systems expose an invocable interface (webhook/API/workflow trigger) **[CONFIRM Q1]**.

## 5. Functional requirements

Priority: **M** = MVP, **P2** = phase after MVP, **F** = future.

### 5.1 Workshop management ([doc](../features/workshop-management.md))

| ID | Requirement | Pri |
|---|---|---|
| FR-WS-01 | Create/edit workshops: title, code, description, IANA timezone, start/end dates, capacity, delivery type, tags | M |
| FR-WS-02 | Manage workshop days and sessions (title, start time, duration, trainer(s), delivery mode: Zoom meeting / Zoom webinar / linked Simulive webinar / external link) | M |
| FR-WS-03 | Workshop lifecycle states: draft → ready → published → in_progress → completed → archived. Also cancelled | M |
| FR-WS-04 | Publish validation with blocking errors and warnings. Override requires reason (audited) | M |
| FR-WS-05 | Provisioning on publish: Zoom sessions, Calendar events, sheet binding, scheduled jobs | M |
| FR-WS-06 | Duplicate a workshop (structure, timeline, workflow bindings) with date shift | M |
| FR-WS-07 | Workshop templates (save as template, create from template) | P2 |
| FR-WS-08 | Multiple workshops can run concurrently with independent automation states | M |
| FR-WS-09 | Upload and attach session materials (R2). Publish to participants via comms triggers | M |
| FR-WS-10 | Cancel a workshop: cancel Zoom/Calendar, stop automation, optional participant notification | M |

### 5.2 Timeline ([doc](../features/timeline-engine.md))

| ID | Requirement | Pri |
|---|---|---|
| FR-TL-01 | Multi-day timeline builder: Workshop → Days → Sessions → Activities. Drag-and-drop ordering and time adjustment | M |
| FR-TL-02 | Activity types: WhatsApp trigger, Email trigger, Zoom meeting, Zoom webinar, Simulive (linked/verify), Publish recording, Publish material, Sheet sync, Doc generation (P2), Calendar event, Participant status, Attendance sync, Wait, Conditional branch, Admin notification, Manual task | M (Doc gen P2) |
| FR-TL-03 | Every activity has name, description, timing, duration, execution mode, integration, configuration, dependencies, success condition, failure behavior, enabled flag, criticality | M |
| FR-TL-04 | Execution modes: **scheduled** (absolute or relative to workshop/session start), **event-based**, **sequential** (after dependency, with lag) | M |
| FR-TL-05 | Dependencies with types on_success / on_completion / on_failure, plus lag. Cycle detection | M |
| FR-TL-06 | Relative pre-workshop schedule (−7 d, −3 d, −24 h, −1 h, −15 min) configurable per workshop | M |
| FR-TL-07 | Reschedule a session/day/workshop. All anchored activities recompute. Obsolete jobs are cancelled. External systems are updated. Notifications are optional. Everything is audited | M |
| FR-TL-08 | Impact preview (dry run) before any reschedule | M |
| FR-TL-09 | Missed-execution policy per activity (run late within grace window / skip / ask) | M |
| FR-TL-10 | Activity status visible in real time | M |

### 5.3 Automation workflows ([doc](../features/automation-engine.md))

| ID | Requirement | Pri |
|---|---|---|
| FR-AE-01 | Visual workflow builder (React Flow): trigger, conditions, actions, delays, branches, error handling | M (basic) |
| FR-AE-02 | Triggers: domain events (registration created, payment changed, registration closed, session ended, attendance imported, workshop completed…), schedule (cron), manual | M |
| FR-AE-03 | Conditions: structured comparisons on workshop/participant/session/registration fields. AND/OR groups | M |
| FR-AE-04 | Actions: same action catalog as timeline activities | M |
| FR-AE-05 | Branching (if/else, multi-way switch), per-participant fan-out | M (if/else), P2 (switch) |
| FR-AE-06 | Error handling per node: stop / continue / error branch, plus retry policy | M |
| FR-AE-07 | Execution limits: max executions per trigger window, max participants per run, concurrency | M |
| FR-AE-08 | Draft/publish versioning. Executions pinned to version. Duplicate workflow | M |
| FR-AE-09 | Test run (dry run with sample data, no external calls) | P2 |

### 5.4 Integrations

| ID | Requirement | Pri | Doc |
|---|---|---|---|
| FR-IN-01 | Connect/disconnect integrations, health status, capability manifest | M | [integration-architecture](../architecture/integration-architecture.md) |
| FR-IN-02 | Register existing WhatsApp/email triggers (key, endpoint, variables schema) and invoke them with participant/workshop variables | M | [whatsapp-email](../features/whatsapp-email-integration.md) |
| FR-IN-03 | Receive and record comms execution responses and delivery callbacks | M | same |
| FR-IN-04 | Zoom: create/update/cancel meetings and webinars, get join URL, get fresh host start URL, end session, registrants | M | [zoom](../features/zoom-integration.md) |
| FR-IN-05 | Zoom: attendance retrieval (reports + webhooks), recordings metadata, `recording.completed` processing | M | same |
| FR-IN-06 | Zoom: Simulive is supported only as "linked webinar" plus verification. Creation/playback automation only if Zoom provides an API | M (linked) | same |
| FR-IN-07 | Google Sheets: import, detect new rows (reconciliation), update rows, export reports, column ownership mapping | M | [google-workspace](../features/google-workspace.md) |
| FR-IN-08 | Google Calendar: create/update/cancel workshop & session events, trainers as attendees, Zoom details, no duplicates | M | same |
| FR-IN-09 | Google Calendar: trainer free/busy check | P2 | same |
| FR-IN-10 | Google Docs: generate from templates (schedule, attendance sheet, completion report, summary, certificates) | P2 (certificates F) | same |

### 5.5 Participants ([doc](../features/participant-management.md))

| ID | Requirement | Pri |
|---|---|---|
| FR-PM-01 | Central participant records with dedupe (normalized email, E.164 phone) | M |
| FR-PM-02 | Registrations per workshop: status, payment status, source, timestamps | M |
| FR-PM-03 | Import from Sheet/CSV. Generic registration intake API | M |
| FR-PM-04 | Attendance per session (raw + computed %) and completion status rules | M |
| FR-PM-05 | Manual edits with reason. Merge duplicates | M (edit), P2 (merge) |
| FR-PM-06 | Communication history per participant (from action executions) | M |

### 5.6 Operations, monitoring and control ([dashboard](../features/admin-dashboard.md), [monitoring](../features/execution-monitoring.md))

| ID | Requirement | Pri |
|---|---|---|
| FR-OP-01 | Workshop Control Center: live counters, current/next activity, attention items | M |
| FR-OP-02 | Controls: Start, Pause, Resume, Stop, Reschedule, Skip (reason), Retry, Execute now, Edit, Duplicate, Emergency stop, Manual override | M |
| FR-OP-03 | Pausing the orchestration does **not** pause running external sessions (Zoom). The UI makes this distinction explicit | M |
| FR-OP-04 | Execution record for every action, with attempts, timings, external refs, redacted payloads | M |
| FR-OP-05 | Retry policies per action. Admin notification after retry limit | M |
| FR-OP-06 | Failure behaviors per PRD Cases 1–4 | M |
| FR-OP-07 | Execution Monitor: filters, bulk retry, dead-letter view, unknown-outcome resolution | M |
| FR-OP-08 | Alerts (in-app + ops channel) with severity and acknowledgement | M |
| FR-OP-09 | Audit log of all administrative actions with before/after | M |
| FR-OP-10 | Reports: registrations, attendance, completion, execution health. CSV export | M (basic) |

## 6. Non-functional requirements

| ID | Requirement | Target (proposed) |
|---|---|---|
| NFR-01 | Scheduled activity start latency | p95 < 60 s after `scheduled_for`. p99 < 3 min |
| NFR-02 | No duplicate execution of an activity run | 0 tolerated (enforced by DB constraints) |
| NFR-03 | Durability | No scheduled activity lost across deploys/restarts. RPO ≤ 5 min (DB PITR) |
| NFR-04 | Availability (admin API) | 99.5% monthly (single business; maintenance windows allowed outside workshop hours) |
| NFR-05 | Admin API latency | p95 < 300 ms for reads |
| NFR-06 | Scale | 20 concurrent workshops, 2,000 participants per workshop, 200 activities per workshop |
| NFR-07 | Webhook acknowledgement | < 1 s p95 (Zoom requires < 3 s **[VERIFIED]**) |
| NFR-08 | Security | RBAC, 2FA for owner/admin, encrypted credentials, audit |
| NFR-09 | Observability | Every action traceable from admin command to provider call |
| NFR-10 | Timezone correctness | All schedule computations use IANA tz. DST-safe |
| NFR-11 | Accessibility | Admin UI WCAG 2.1 AA for core flows |

## 7. Success metrics

- ≥ 99% of scheduled communications triggered within 2 min of planned time.
- 0 duplicate participant messages caused by the platform.
- Attendance reconciliation time reduced from hours to < 15 min after session end.
- Every failure visible in the Execution Monitor within 1 min, with a recovery action.

## 8. Global acceptance criteria

1. A three-day workshop with 3 sessions/day and 40 activities runs end-to-end on staging without manual intervention, including one forced reschedule and one forced integration outage.
2. A worker restart in the middle of a workshop causes no missed or duplicate activities.
3. Every action shown in the Execution Monitor links to its trace and its audit trail.
4. Each of PRD failure Cases 1–4 is reproducible in tests with the documented behavior.

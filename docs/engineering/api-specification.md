# API Specification

REST, JSON, OpenAPI 3.1 generated from Zod contracts in `@wos/contracts` ([technical-stack.md §3.4](technical-stack.md#34-zod-as-the-single-schema-source)). Served at `/api/v1`. Interactive docs at `/api/docs` (authenticated, admin only).

Related: [security.md](security.md) · [database-design.md](database-design.md) · [whatsapp-email-integration.md](../features/whatsapp-email-integration.md) · [zoom-integration.md](../features/zoom-integration.md)

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Auth | Session cookie (Better Auth) for the SPA. `Authorization: Bearer wos_…` API keys for intake endpoints only |
| IDs | Prefixed ULIDs (`wks_`, `ses_`, `act_`, `run_`, `axn_`, `reg_`, `par_`, `wfl_`, `wfv_`, `wfe_`) |
| Time | ISO 8601 UTC instants (`2026-11-15T02:30:00Z`). Local wall-clock fields are explicitly named `localTime` / `localDate` with the workshop's `timezone` |
| Casing | camelCase JSON |
| Pagination | Cursor: `?limit=50&cursor=<opaque>` → `{ data: [], nextCursor: string|null }` |
| Filtering | `?status=failed,needs_attention&workshopId=…&from=…&to=…` |
| Sorting | `?sort=-createdAt` |
| Concurrency | Mutable resources return `version` and `ETag: "<version>"`. Updates require `If-Match`. Mismatch → `409 version_conflict` with current resource |
| Idempotency | `Idempotency-Key` **required** on all `POST` control/command endpoints (publish, activate, pause, retry, execute-now, reschedule, bulk ops). Optional on creates |
| Errors | RFC 9457 `application/problem+json` (below) |
| Rate limits | Per-user 300 req/min (API), 10/min on auth endpoints (Better Auth built-in) |
| Versioning | URI major version `/api/v1`. Additive changes only within v1 |

### Error format

```json
{
  "type": "https://docs.workshop-os.internal/errors/validation_failed",
  "title": "Validation failed",
  "status": 422,
  "code": "validation_failed",
  "detail": "Activity act_9 has a dependency cycle",
  "errors": [{ "path": "dependencies[0]", "code": "cycle", "message": "act_9 → act_3 → act_9" }],
  "correlationId": "cor_01JC…"
}
```

Common codes: `unauthenticated` 401, `forbidden` 403, `not_found` 404, `version_conflict` 409, `idempotency_key_reused` 422, `validation_failed` 422, `invalid_state_transition` 409, `capability_missing` 422, `rate_limited` 429, `internal` 500.

## 2. Endpoint catalog

Permissions refer to [security.md §3](security.md#3-rbac).

### 2.1 Auth & users

| Method | Path | Perm | Notes |
|---|---|---|---|
| * | `/api/auth/*` | public | Better Auth (sign-in, sign-out, session, 2FA, password reset) |
| GET | `/api/v1/me` | any | Current user, role, permissions |
| GET/POST | `/api/v1/users` | `users.manage` | Invite staff |
| PATCH | `/api/v1/users/{id}/role` | `users.manage` | Owner only for `owner/admin` |
| GET/POST/DELETE | `/api/v1/api-keys` | `integrations.manage` | Intake keys (shown once) |

### 2.2 Workshops

| Method | Path | Perm | Notes |
|---|---|---|---|
| GET | `/api/v1/workshops` | `workshops.read` | Filters: status, from/to, q |
| POST | `/api/v1/workshops` | `workshops.write` | Create draft |
| GET | `/api/v1/workshops/{id}` | `workshops.read` | Includes days, sessions summary, counts |
| PATCH | `/api/v1/workshops/{id}` | `workshops.write` | If-Match |
| POST | `/api/v1/workshops/{id}/validate` | `workshops.read` | Returns issues (blocking/warning) |
| POST | `/api/v1/workshops/{id}/publish` | `workshops.publish` | `{ activate: bool, overrides?: [{issueCode, reason}] }` → 202 + job ref |
| POST | `/api/v1/workshops/{id}/activate` | `automation.control` | Start automation |
| POST | `/api/v1/workshops/{id}/pause` | `automation.control` | `{ reason }` |
| POST | `/api/v1/workshops/{id}/resume` | `automation.control` | `{ missedPolicyOverride?: 'run'|'skip'|'ask' }` → returns what will run |
| POST | `/api/v1/workshops/{id}/stop` | `automation.control` | `{ reason }` |
| POST | `/api/v1/workshops/{id}/emergency-stop` | `automation.emergency` | `{ reason, endLiveZoomSessions: bool }` |
| POST | `/api/v1/workshops/{id}/cancel` | `workshops.publish` | `{ reason, notifyParticipants: bool }` |
| POST | `/api/v1/workshops/{id}/duplicate` | `workshops.write` | `{ newStartDate, includeParticipants: false }` |
| POST | `/api/v1/workshops/{id}/reschedule/preview` | `timeline.reschedule` | Dry run: `{ shift: 'P1D' }` or per-session changes → impact report |
| POST | `/api/v1/workshops/{id}/reschedule` | `timeline.reschedule` | Same body + `{ notifyParticipants, reason }` |
| POST | `/api/v1/workshops/{id}/archive` | `workshops.publish` | |
| GET | `/api/v1/workshops/{id}/control-center` | `workshops.read` | Aggregated live view model |

### 2.3 Days and sessions

| Method | Path | Notes |
|---|---|---|
| POST/PATCH/DELETE | `/api/v1/workshops/{id}/days[/{dayId}]` | |
| POST | `/api/v1/workshops/{id}/sessions` | |
| GET/PATCH/DELETE | `/api/v1/sessions/{id}` | PATCH schedule fields on a published workshop → 409 `use_reschedule_endpoint` |
| POST | `/api/v1/sessions/{id}/reschedule/preview` / `reschedule` | |
| POST | `/api/v1/sessions/{id}/zoom/sync` | Ensure/repair Zoom object |
| GET | `/api/v1/sessions/{id}/zoom/host-start` | `sessions.host` perm. Returns **freshly fetched** `start_url` (never cached). Audited |
| POST | `/api/v1/sessions/{id}/zoom/link-existing` | `{ zoomId, zoomType }` (Simulive linked mode) |
| POST | `/api/v1/sessions/{id}/mark-live` / `mark-ended` | Manual delivery status |
| GET/POST | `/api/v1/sessions/{id}/resources` | Presigned upload flow (below) |
| POST | `/api/v1/resources/{id}/publish` | Triggers configured comms trigger |
| GET | `/api/v1/sessions/{id}/recordings` | |
| POST | `/api/v1/recordings/{id}/publish` | |

### 2.4 Timeline

| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/workshops/{id}/timeline` | Full plan: activities, dependencies, computed times, statuses |
| POST | `/api/v1/workshops/{id}/activities` | Create |
| PATCH | `/api/v1/activities/{id}` | If-Match. On published workshop → re-arm |
| DELETE | `/api/v1/activities/{id}` | Draft only. Published → disable instead |
| PUT | `/api/v1/activities/{id}/dependencies` | Replace set. Cycle check |
| POST | `/api/v1/activities/{id}/enable` / `disable` | |
| POST | `/api/v1/activities/{id}/skip` | `{ reason }` |
| POST | `/api/v1/activities/{id}/execute-now` | `{ bypassDependencies: bool, reason }` |
| POST | `/api/v1/activity-runs/{id}/retry` | `{ scope: 'failed_only'|'all' }` |
| POST | `/api/v1/activity-runs/{id}/override` | `{ outcome: 'succeeded'|'failed', reason }` |
| POST | `/api/v1/activity-runs/{id}/confirm` | Manual task confirmation `{ decision: 'approve'|'reject', note }` |
| GET | `/api/v1/activities/{id}/runs` | History |

**Example: create activity**

```http
POST /api/v1/workshops/wks_01J/activities
Content-Type: application/json
If-Match: "12"   (workshop plan version)

{
  "name": "24h reminder",
  "type": "whatsapp_trigger",
  "executionMode": "scheduled",
  "timing": { "anchorType": "session_start", "anchorRefId": "ses_01J", "offset": "-P1D", "offsetSemantics": "calendar" },
  "action": { "key": "comms.trigger", "integrationId": "int_wa", "config": { "triggerKey": "workshop_reminder" } },
  "audience": { "registrationStatus": ["confirmed"], "excludeIfActionSucceeded": "self" },
  "criticality": "normal",
  "failurePolicy": { "onFailure": "continue", "notify": ["admins"] },
  "missedPolicy": { "mode": "run_if_within_grace", "gracePeriod": "PT2H" },
  "enabled": true
}
```

Response `201`:

```json
{
  "id": "act_01JD…",
  "scheduledFor": "2026-11-14T03:30:00Z",
  "scheduledForLocal": "2026-11-14T09:00:00+05:30",
  "currentStatus": "scheduled",
  "scheduleGeneration": 1,
  "version": 1
}
```

### 2.5 Automation Studio

| Method | Path | Notes |
|---|---|---|
| GET/POST | `/api/v1/workflows` | |
| GET/PATCH | `/api/v1/workflows/{id}` | PATCH updates draft only |
| POST | `/api/v1/workflows/{id}/validate` | |
| POST | `/api/v1/workflows/{id}/publish` | `{ changeNote, inflightPolicy: 'continue'|'cancel_inflight' }` → new version |
| GET | `/api/v1/workflows/{id}/versions[/{versionId}]` | |
| POST | `/api/v1/workflows/{id}/duplicate` | |
| POST | `/api/v1/workflows/{id}/enable` / `disable` | |
| POST | `/api/v1/workflows/{id}/run` | Manual trigger `{ workshopId, input }` |
| POST | `/api/v1/workflows/{id}/test-run` | P2 dry run |
| GET | `/api/v1/workflow-executions` | Filters |
| GET | `/api/v1/workflow-executions/{id}` | Includes step executions |
| POST | `/api/v1/workflow-executions/{id}/cancel` / `retry-from/{nodeId}` | Retry resumes from failed node without re-running succeeded nodes |

### 2.6 Participants and registrations

| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/participants?q=` | Search by name/email/phone |
| GET/PATCH | `/api/v1/participants/{id}` | Edits need reason for key fields |
| POST | `/api/v1/participants/{id}/merge` | P2 `{ intoId, reason }` |
| GET | `/api/v1/workshops/{id}/registrations` | Filters: status, paymentStatus, attendance range, completion |
| POST | `/api/v1/workshops/{id}/registrations` | Manual add |
| PATCH | `/api/v1/registrations/{id}` | Status/payment (with reason) |
| POST | `/api/v1/workshops/{id}/registrations/import` | CSV upload (presigned) → async job |
| GET | `/api/v1/registrations/{id}/timeline` | All actions, attendance, audit for this registration |
| GET | `/api/v1/review-queue` | Duplicates, unmatched attendees, borderline completion, unknown outcomes |
| POST | `/api/v1/review-queue/{id}/resolve` | |
| GET | `/api/v1/sessions/{id}/attendance` | |
| POST | `/api/v1/sessions/{id}/attendance/sync` | Trigger re-import |
| PATCH | `/api/v1/session-attendance/{sessionId}/{registrationId}` | Override with reason |

### 2.7 Integrations

| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/integrations` | Status, capabilities, health |
| POST | `/api/v1/integrations/zoom` | `{ accountId, clientId, clientSecret, webhookSecretToken, defaultHostUserId }` → test + capabilities |
| GET | `/api/v1/integrations/google/oauth/start` | Redirect to Google consent (offline access) |
| GET | `/api/v1/integrations/google/oauth/callback` | State + PKCE verified |
| POST | `/api/v1/integrations/comms` | `{ kind: 'whatsapp'|'email', baseUrl, auth: {...}, signingSecret }` |
| POST | `/api/v1/integrations/{id}/test` | Health probe |
| POST | `/api/v1/integrations/{id}/rotate-secret` | Dual-secret window |
| DELETE | `/api/v1/integrations/{id}` | Disconnect (blocked if used by active workshops unless forced) |
| GET/POST/PATCH | `/api/v1/integrations/{id}/triggers` | Communication triggers registry |
| POST | `/api/v1/integrations/{id}/triggers/{triggerId}/test` | Sends to a **test recipient** only |
| GET/POST/PATCH | `/api/v1/sheet-bindings` | Column map editor |
| POST | `/api/v1/sheet-bindings/{id}/reconcile` | Run now |
| POST | `/api/v1/sheet-bindings/{id}/preview` | Dry-run diff |

### 2.8 Execution monitoring, alerts and audit

| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/executions` | Unified feed: action executions with filters (status, integration, workshop, activity, time) |
| GET | `/api/v1/executions/{id}` | With attempts, trace link |
| POST | `/api/v1/executions/{id}/retry` | |
| POST | `/api/v1/executions/{id}/resolve` | `{ resolution: 'mark_delivered'|'resend'|'mark_failed'|'skip', reason }` |
| POST | `/api/v1/executions/bulk` | `{ ids[] | filter, operation: 'retry'|'skip'|'resolve', reason }` → async job |
| GET | `/api/v1/dead-letter` | |
| GET | `/api/v1/alerts` · POST `/api/v1/alerts/{id}/ack` · `/resolve` | |
| GET | `/api/v1/audit-logs` | Filters: entity, actor, workshop, time |
| GET | `/api/v1/reports/workshops/{id}/summary` | Counts, attendance distribution, comms success |
| GET | `/api/v1/reports/workshops/{id}/export?format=csv` | Async → presigned URL |

### 2.9 Files

1. `POST /api/v1/uploads` `{ purpose: 'resource'|'import', filename, mimeType, sizeBytes }` → `{ uploadId, presignedPutUrl, expiresAt }` (R2, 15 min).
2. The client PUTs directly to R2.
3. `POST /api/v1/uploads/{uploadId}/complete` → server verifies object (size, checksum) → creates resource/import job.

### 2.10 Realtime

`GET /api/v1/stream?topics=workshop:{id},alerts,executions:{workshopId}`, `text/event-stream`.

```
id: 01JD…
event: activity.updated
data: {"workshopId":"wks_…","activityId":"act_…","runId":"run_…","status":"running"}
```

Event types: `activity.updated`, `run.updated`, `execution.updated`, `session.updated`, `registration.updated` (counts only), `alert.raised`, `alert.updated`, `workshop.state_changed`, `heartbeat` (every 25 s).

## 3. Inbound webhooks

All webhook routes use raw-body parsing ([ADR-006](../adr/006-auth-better-auth-and-body-parsing.md)), verify signatures, persist to `webhook_events`, and respond fast.

### 3.1 Zoom: `POST /webhooks/zoom/{integrationId}`

- **URL validation** **[VERIFIED]**: on `event = "endpoint.url_validation"`, respond `200 { "plainToken": p, "encryptedToken": hex(HMAC_SHA256(webhookSecret, p)) }`.
- **Verification** **[VERIFIED]**: `x-zm-signature == "v0=" + hex(HMAC_SHA256(secret, "v0:" + x-zm-request-timestamp + ":" + rawBody))`. Also check the timestamp is within 300 s (our policy).
- Respond **within 3 s** with 200/204, or Zoom treats delivery as failed. Zoom retries only on ≥ 500 (and specific internal codes) at +5, +20 and +60 min. **4xx is not retried** **[VERIFIED]**. So we return 200 for duplicates and unknown event types, and 401 only for bad signatures.
- Subscribed events (configured in the Zoom app) **[VERIFIED event names; subscription set to confirm in SPIKE S3]**: `meeting.started`, `meeting.ended`, `meeting.participant_joined`, `meeting.participant_left`, `webinar.started`, `webinar.ended`, `webinar.participant_joined`, `webinar.participant_left`, `recording.completed`, `meeting.deleted`, `meeting.updated`, `webinar.deleted`, `webinar.updated`.

### 3.2 Communication callbacks: `POST /webhooks/comms/{integrationId}`

Our contract for the client's systems ([whatsapp-email-integration.md §5](../features/whatsapp-email-integration.md#5-callback-contract-delivery-status)):

```http
POST /webhooks/comms/int_wa
X-WOS-Signature: t=1731638400,v1=5f2b…
X-WOS-Delivery-Id: dlv_8812
Content-Type: application/json

{
  "requestId": "axn_01JD…",          // our action execution id (sent in the trigger)
  "externalExecutionId": "n8n-exec-4411",
  "status": "delivered",             // accepted|queued|sent|delivered|read|failed|rejected
  "channel": "whatsapp",
  "recipient": { "participantId": "par_…" },
  "error": null,
  "occurredAt": "2026-11-15T02:31:12Z"
}
```

Response `200 {}` (also for duplicates). `401` invalid signature. `404` unknown `requestId`: stored anyway for investigation, and returns `200` to avoid retry storms.

### 3.3 Registration intake: `POST /webhooks/intake/registrations`

For an external registration system/form, if one exists **[CONFIRM Q3]**. API key + `Idempotency-Key` header.

```json
{
  "workshopCode": "MC-2026-11",
  "participant": { "fullName": "Rahul Sharma", "email": "rahul@example.com", "phone": "+91 98765 43210" },
  "payment": { "status": "paid", "reference": "pay_123" },
  "source": "website_form",
  "submittedAt": "2026-11-01T10:00:00Z",
  "attributes": { "city": "Pune" }
}
```

→ `202 { "registrationId": "reg_…", "status": "confirmed|needs_review|pending_payment", "duplicate": false }`

### 3.4 Sheet nudge: `POST /webhooks/google/sheets/{bindingId}`

Optional Apps Script `onChange`/`onFormSubmit` installable trigger posts `{ bindingId, ts }` with an HMAC header. This only **schedules a reconciliation** (debounced). The body is never trusted for data.

### 3.5 Inngest: `POST|PUT|GET /api/inngest`

Not a public API. Served only by the worker role. Authenticated by the Inngest SDK signature. Must not be behind session auth. See [nestjs-inngest-integration.md](../architecture/nestjs-inngest-integration.md).

## 4. Outbound contracts (we call)

- Comms trigger request: [whatsapp-email-integration.md §4](../features/whatsapp-email-integration.md#4-outbound-trigger-contract)
- Zoom endpoints used: [zoom-integration.md §3](../features/zoom-integration.md#3-verified-zoom-api-capabilities)
- Google endpoints used: [google-workspace.md](../features/google-workspace.md)

## 5. Health and ops endpoints

| Path | Role | Notes |
|---|---|---|
| `GET /healthz` | all | Liveness (process up) |
| `GET /readyz` | all | DB reachable, migrations at expected version, (worker) Inngest client configured |
| `GET /metrics` | all | Prometheus format (private network only) if metrics are not pushed via OTLP |

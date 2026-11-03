# WhatsApp and Email: Integration with the Client's Existing Automation

> **We do not send messages.** The client's existing WhatsApp and email automation systems remain responsible for templates, delivery, opt-outs and provider compliance. Workshop OS decides **when** and **for whom** a trigger fires, sends the variables, and records the response (PRD §5.2–5.3).

Related: [integration-architecture.md](../architecture/integration-architecture.md) · [ADR-012](../adr/012-unknown-outcome-no-auto-retry.md) · [api-specification.md §3.2](../engineering/api-specification.md#32-communication-callbacks-post-webhookscommsintegrationid) · [open-questions.md Q1, Q2](../planning/open-questions.md)

---

## 1. Purpose and scope

- Register the existing systems' triggers (e.g. `registration_confirmation`, `workshop_reminder`, `payment_confirmation`, `joining_instructions`, `session_reminder`, `workshop_completed`, `schedule_change`, `session_resources`, `assessment_notification`, `certificate_delivery`).
- Invoke them with participant + workshop variables, durably, idempotently and throttled.
- Receive acknowledgements and (optionally) delivery callbacks.
- Show the results in the Execution Monitor and in each participant's history.

## 2. The biggest unknown

**Where do the existing automations run?** n8n, Make, GoHighLevel, Interakt/WATI/AiSensy-type WhatsApp BSP tools, a custom backend, or something else **[CONFIRM Q1]**. The invocation mechanism, auth, idempotency support, bulk support and callbacks depend on the answer. The design therefore defines **our** contract (§4–5) and a **generic, configurable adapter**. Each known platform is mapped onto it:

| Platform type | How we'd invoke | Idempotency | Callback | Notes |
|---|---|---|---|---|
| n8n | Webhook node URL (POST) | Implement in workflow (check `Idempotency-Key` in a datastore) **[CONFIRM]** | HTTP Request node → our callback URL | Webhook response mode should be configured to respond when the workflow accepts, not to time out **[ASSUMPTION]** |
| Make | Custom webhook URL | Implement in scenario (data store lookup) | HTTP module → callback | By default Make returns `Accepted` immediately **[ASSUMPTION]**, so we only get an acknowledgement |
| GoHighLevel | Inbound webhook workflow trigger, or API to add contact to workflow **[CONFIRM]** | Unclear | Outbound webhook action | Contact-centric model. Mapping participant → contact needed |
| Custom backend | REST endpoint we specify (§4) | Should honor `Idempotency-Key` | Implements §5 | Preferred |
| WhatsApp BSP tool with API | Its "send template"/campaign API | Varies | Its webhooks | Risk: becomes "building messaging". Only if the client's automation lives there |

## 3. Provider constraints we must respect (but don't implement)

The existing WhatsApp system, if it's built on Meta's WhatsApp Business Platform, is subject to Meta's rules. These affect **why triggers may fail**:

- Template messages are the only message type that can be sent **outside the customer service window**, which a user message opens and resets **[VERIFIED]** ([Meta pricing docs](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing)). Workshop reminders are business-initiated, so they will almost always be templates.
- Templates can be paused/disabled based on quality feedback. Recipients must have opted in.
- Since **1 July 2025** Meta charges **per delivered template message**. Marketing/authentication templates are always billed. Utility templates are billed when delivered outside a customer service window **[VERIFIED]**. A third-party source reports further changes from 1 October 2026 for messages inside the window **[COMMUNITY, unconfirmed by Meta]**. Each reminder has a direct cost in the existing system, so the platform shows per-workshop trigger counts for spend monitoring, and template category (utility vs marketing) is the client's concern **[CONFIRM Q2]**.

Implications for us:

- Rejections like `template_paused` or `recipient_not_opted_in` come back as `permanent` errors with a provider code. They aren't retried. They show up in the Execution Monitor grouped by reason.
- Our consent flags (participant record) can gate triggers (`requires_consent`), but **the existing system remains the authority on opt-outs**.

## 4. Outbound trigger contract

What we send. This is our proposed contract, and the client's system adapts to it, or our adapter maps onto theirs.

```http
POST {baseUrl}{endpointPath}
Content-Type: application/json
Idempotency-Key: comms:workshop_reminder:run_01JD…:reg_01JC…
X-WOS-Request-Id: axn_01JD…
X-WOS-Signature: t=1731638400,v1=hex(HMAC_SHA256(secret, "1731638400." + body))
Authorization: Bearer <api key>        (if their system requires it)
```

```json
{
  "trigger": "workshop_reminder",
  "requestId": "axn_01JD…",
  "callbackUrl": "https://ops.client.com/webhooks/comms/int_wa",
  "workshop": {
    "id": "wks_01J…",
    "code": "MC-2026-11",
    "name": "Medical Coding",
    "timezone": "Asia/Kolkata"
  },
  "session": {
    "id": "ses_01J…",
    "title": "Session 1 — ICD-10-CM",
    "startsAt": "2026-11-15T03:30:00Z",
    "startsAtLocal": "2026-11-15T09:00:00+05:30"
  },
  "recipient": {
    "participantId": "par_01J…",
    "registrationId": "reg_01J…",
    "name": "Rahul",
    "phone": "+919876543210",
    "email": "rahul@example.com"
  },
  "variables": {
    "name": "Rahul",
    "workshop_name": "Medical Coding",
    "session_time": "9:00 AM IST, 15 Nov",
    "joining_link": "https://zoom.us/w/…"
  },
  "scheduledAt": "2026-11-15T02:30:00Z",
  "sentAt": "2026-11-15T02:30:04Z"
}
```

- This is the PRD §5.2 payload shape, extended with IDs, a callback URL and a structured recipient. `variables` are validated against the trigger's registered `variables_schema` before sending, so missing variables fail **before** the call (`permanent: missing_variable`).
- **Bulk variant** (if `supports_bulk`): `recipients: [...]` with per-recipient `variables`, ≤ 100 per call (configurable), plus a `requestId` per recipient.
- Expected responses:

| Response | Meaning to us |
|---|---|
| `200/201 { accepted: true, externalExecutionId }` | `succeeded` if the system is synchronous, or `accepted` if callbacks are expected |
| `202 { externalExecutionId }` | `accepted` |
| `409 { duplicate: true, externalExecutionId }` | Idempotent replay → `succeeded` (or `accepted`) |
| `4xx { code, message }` | `permanent` (e.g. `unknown_trigger`, `invalid_recipient`, `template_paused`) |
| `429` (+ `Retry-After`) | `transient` |
| `5xx` | `transient` |
| Timeout / reset after send | `unknown_outcome` → [ADR-012](../adr/012-unknown-outcome-no-auto-retry.md) (auto-retry **only** if idempotency is verified) |

- Timeout: connect 5 s, total 15 s (configurable).

## 5. Callback contract (delivery status)

Optional but strongly recommended. Schema in [api-specification.md §3.2](../engineering/api-specification.md#32-communication-callbacks-post-webhookscommsintegrationid).

- Status progression: `accepted → queued → sent → delivered → read`, or `failed | rejected`. We store the **latest** status plus a timestamp per status. Out-of-order callbacks are handled by rank ordering. A later rank always wins, and `failed` after `delivered` is ignored with a warning.
- If no callback arrives within `callback_timeout` (default 2 h), the action stays `accepted` (not failed). It's reported as "delivery unconfirmed", and there's no retry.

## 6. Trigger registry

`communication_triggers` rows (see [database-design.md](../engineering/database-design.md#37-integrations)):

| Field | Example |
|---|---|
| `channel` | whatsapp |
| `trigger_key` | `workshop_reminder` |
| `external_workflow_ref` | n8n workflow ID / Make scenario ID / GHL workflow ID |
| `endpoint_path` | `/webhook/wos/workshop-reminder` |
| `variables_schema` | JSON Schema: `name`, `workshop_name`, `session_time`, `joining_link` required |
| `supports_bulk` | false |
| `idempotency_class` | `unsafe` until verified → `keyed` |
| `requires_consent` | true |

**Verification flow** (Integrations → Triggers → "Test"):

1. Send to a **designated test recipient** (an operator's phone/email) with sample variables.
2. Send again with the **same** `Idempotency-Key`. If the second send produces no second message (operator confirms) and returns `409`/duplicate, the system marks `idempotency_class=keyed`.
3. Record `last_verified_at`. Publish validation warns if a trigger hasn't been verified in 30 days.

## 7. Execution flow

Per participant (fan-out) or bulk chunk ([timeline §7](timeline-engine.md#7-execution-algorithm)):

1. Eligibility (in step): registration status, consent, phone/email present, not already `succeeded` for this activity run, workshop not paused.
2. Variables resolution: from workshop/session/registration (e.g. per-registrant Zoom join URL). Fail fast on missing ones.
3. `gateway.call('comms.trigger', …)` → action execution + attempt rows.
4. Throttle: Inngest `throttle` keyed by `integrationId` (default 10/s, tunable to what their system tolerates **[CONFIRM Q1]**).
5. Result classification per §4.
6. Callback updates later.

## 8. Failure handling (PRD Case 1)

- Record the failure → retry per policy (transient only) → after the retry limit, notify the admin (alert grouped per activity, not per participant) → **the timeline is not blocked**. Comms activities default to `criticality=normal`, `onFailure=continue`.
- Integration-wide outage → breaker opens → retries are spaced out. If the activity has a deadline (e.g. "15 min before start"), remaining sends fail at the deadline with `deadline_exceeded`, so we don't send "starting soon" after start.
- Unknown outcome → `needs_attention`. Bulk resolve in the Execution Monitor: "Mark all as delivered" (after checking their system) or "Resend all".

## 9. Security

- Outbound: HMAC signature on every request (secret per integration, rotatable with a dual-secret window). Optional bearer/API key per their requirements. TLS only. Allow-listed base URLs (no arbitrary URLs per trigger).
- Inbound callbacks: HMAC + timestamp window + delivery ID dedupe.
- Minimal PII: only the variables the trigger schema requires are sent (schema-driven allow-list).
- SSRF protection: base URLs validated at registration (no private IP ranges unless explicitly allowed for an on-prem system).

## 10. Edge cases

- The same participant registered for two concurrent workshops gets two reminders. This is correct, because the idempotency key includes the activity run.
- Admin reruns the activity ("retry all") → already succeeded recipients are excluded unless "resend to all" is explicitly chosen (with reason).
- Their system renames a trigger → `unknown_trigger` permanent failures → alert "trigger misconfigured" after 3 occurrences.
- Phone numbers for WhatsApp must be E.164. Invalid ones are skipped before the call.

## 11. Testing

- Contract tests with a **mock comms server** implementing §4–5 (including idempotency, 409, slow responses, timeouts).
- Adapter mapping tests for whichever platform the client uses (n8n/Make/GHL fixtures).
- Load test: 2,000 recipients with throttle 10/s → completes in ~3.5 min, with no 429 storms.
- Unknown-outcome simulation: the server accepts and then drops the connection → `needs_attention` (unsafe class) / no duplicate (keyed class).

## 12. Acceptance criteria

1. Registering a trigger with a variables schema makes it selectable in timeline and workflow actions.
2. A reminder activity for 248 participants produces 248 action executions with statuses, external IDs and (if supported) delivery statuses.
3. A comms outage doesn't block Zoom/session activities, and the admin is alerted once per activity.
4. No duplicate trigger is sent for the same participant and activity run under worker restarts.

## 13. Implementation dependencies

Gateway + execution records → webhook ingress (callbacks) → trigger registry UI → fan-out engine → **client answer to Q1/Q2** for the concrete mapping.

## 14. Trade-offs

- A generic webhook adapter maximizes compatibility, but idempotency depends on the client's implementation.
- Per-participant calls give exact per-recipient records and cost more Inngest executions and calls. Bulk is cheaper but needs per-recipient results in the response.

## 15. Future improvements

- Native adapters for the specific platform once known.
- Delivery analytics (read rates) per workshop.
- Quiet hours / per-participant local time windows.

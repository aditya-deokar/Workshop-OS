# Zoom Integration: Meetings, Webinars, Simulive, Attendance and Recordings

> **Important technical distinction (PRD §5.1).** *Scheduling* a meeting, *starting* a meeting, and *playing a prerecorded Simulive webinar* are **different capabilities** with different APIs, licenses and limits. This document separates what Zoom's API verifiably supports from what it doesn't.

Related: [integration-architecture.md](../architecture/integration-architecture.md) · [participant-management.md §6](participant-management.md#6-attendance) · [workshop-management.md §8](workshop-management.md#8-sessions-and-delivery-modes) · Diagrams: [Zoom session creation](../architecture/diagrams.md#15-sequence-zoom-session-creation), [attendance](../architecture/diagrams.md#16-sequence-attendance-synchronization)

---

## 1. Purpose and scope

Zoom is the primary delivery system. The platform manages Zoom meetings/webinars as **projections** of sessions. It provides host access and processes Zoom webhooks into session status, attendance and recordings.

## 2. User stories

| # | Story |
|---|---|
| US-ZM-1 | Creating a session at 9 AM creates a Zoom webinar, stores its ID and joining URL, and updates the Calendar event (PRD §5.1 example). |
| US-ZM-2 | Rescheduling a session updates the Zoom start time. |
| US-ZM-3 | The trainer clicks "Start as host" in the Control Center and Zoom opens with a fresh host link. |
| US-ZM-4 | When the session ends, attendance is imported automatically. |
| US-ZM-5 | When the cloud recording is ready, it's attached to the session and can be published. |
| US-ZM-6 | For a Simulive session, the system verifies that the linked webinar exists and is scheduled correctly, and shows a clear warning if not. |

## 3. Verified Zoom API capabilities

**Authentication:** a **Server-to-Server OAuth** app (account-level, created by a Zoom account admin) using the `account_credentials` grant. Access tokens are short-lived (1 h) with no refresh token, so a new one is requested as needed. JWT apps are deprecated. **[VERIFIED: Zoom S2S OAuth model; scopes to be pinned in SPIKE S3]**

| Operation (PRD table) | Zoom API | Status | Notes |
|---|---|---|---|
| Create session | `POST /v2/users/{userId}/meetings` · `POST /v2/users/{userId}/webinars` | **[VERIFIED]** | Webinars require the **Zoom Webinars license** **[LICENSE]** |
| Update session | `PATCH /v2/meetings/{meetingId}` · `PATCH /v2/webinars/{webinarId}` | **[VERIFIED]** | Counts toward **100 create/update requests per day per user** (UTC reset) **[VERIFIED]** ([rate limits](https://developers.zoom.us/docs/api/rate-limits/)) |
| Schedule session | `start_time`, `timezone`, `duration` on create/update | **[VERIFIED]** | We send UTC `start_time` plus the workshop `timezone` for display in Zoom |
| Cancel session | `DELETE /v2/meetings/{id}` · `DELETE /v2/webinars/{id}` | **[VERIFIED]** | Optional cancellation reminder params (verify names in SPIKE S3) |
| Get joining URL | `join_url` in create/get response. Per-registrant `join_url` when registration is enabled | **[VERIFIED]** | |
| Registrants | `POST /v2/meetings/{id}/registrants` · `POST /v2/webinars/{id}/registrants` | **[VERIFIED endpoint family]** | Requires registration enabled on the meeting/webinar. Batch registrant endpoints exist for some types **[ASSUMPTION: verify limits]** |
| Get host URL | `start_url` from `GET /v2/meetings/{id}` | **[VERIFIED]**, with expiry caveats | `start_url` **expires**: reported as ~2 h for regular users and longer for API-created users **[COMMUNITY]**. **Never cache. Fetch fresh on click.** Alternatively the ZAK token (`GET /v2/users/{userId}/token?type=zak`) for Meeting SDK flows |
| Start delivery | **No REST endpoint starts a meeting server-side.** A host must start it via `start_url`/client/SDK | **[VERIFIED by absence + Zoom start-meeting workflow docs]** | Our "Zoom delivery start" activity = notify the host + expose "Start as host". The session goes `live` when the `meeting.started` webhook arrives. "Join before host" settings can let participants enter without the host **[VERIFIED setting exists]** |
| End session | `PUT /v2/meetings/{id}/status` `{ "action": "end" }` | **[VERIFIED for meetings]** | Webinar equivalent to confirm in SPIKE S3 |
| Retrieve attendance | `GET /v2/report/meetings/{id}/participants` · `GET /v2/report/webinars/{id}/participants` · `GET /v2/past_meetings/{uuid}/participants` | **[VERIFIED endpoints]**. Report APIs **require a paid plan** and report scopes **[COMMUNITY]**. "Heavy" rate category **[COMMUNITY]** | Pagination via `next_page_token`. Data retention windows (reported 6–15 months) **[COMMUNITY]**, so import promptly |
| Retrieve recording | `GET /v2/meetings/{meetingId}/recordings` | **[VERIFIED]** | Cloud recording requires a plan that includes it **[LICENSE]**. Download URLs need an access token. `share_url` for viewing |
| Recording ready | Webhook `recording.completed` | **[VERIFIED]** | |
| Session status | Webhooks `meeting.started`, `meeting.ended`, `webinar.started`, `webinar.ended` | **[VERIFIED]** | |
| Live attendance | `meeting.participant_joined/left`, `webinar.participant_joined/left` | **[VERIFIED]** | Best effort |

### 3.1 Rate limits that matter **[VERIFIED]**

| Category | Pro | Business+ |
|---|---|---|
| Light | 30/s | 80/s |
| Medium | 20/s | 60/s |
| Heavy | 10/s | 40/s |
| Resource-intensive | 10/min | 20/min |
| Heavy + resource-intensive combined daily | 30,000/day | 60,000/day |
| **Meeting/webinar create/update** | **100/day per user** (all plans) | |

Design consequences:

- Batch all field changes of a session into **one** PATCH per reschedule.
- Track a per-host daily budget (`integration_rate_buckets`). Warn at publish/reschedule preview. Reserve 20 for emergencies.
- If the client runs many concurrent workshops on one host user, distribute sessions across several host users **[CONFIRM Q5]**.
- Concurrent requests to the same resource can also return 429 **[VERIFIED]**, so serialize per meeting ID (Inngest concurrency key `zoomId`, limit 1).

## 4. Simulive: what is and isn't possible

**Findings (Nov 2026):**

- Zoom documents **limited** API support for Simulive. A community thread (Apr–Jun 2026) found **no documented field** to create a Simulive webinar or attach a prerecorded file through `POST /users/{userId}/webinars`. An undocumented `simulive_settings` object was silently ignored (`is_simulive: false`) **[COMMUNITY]** ([thread](https://devforum.zoom.us/t/create-simulive-event-via-the-api/143011)).
- Zoom's July 2025 changelog added a `simulive_delay_start` request/response field to webinar create/update/get. Its semantics are not described **[VERIFIED field exists]** ([changelog](https://developers.zoom.us/changelog/meetings/july-21-2025/)).
- Simulive availability depends on licensing. A community answer says it requires Zoom Webinars **plus** a Zoom Events or Zoom Sessions license **[COMMUNITY] [LICENSE] [CONFIRM Q5]**.
- "Do not assume ordinary Zoom meeting APIs can start arbitrary prerecorded video playback" (PRD). **Confirmed: they can't.**

**Design: "linked Simulive webinar" mode.**

1. The client's team creates the Simulive webinar **in the Zoom web portal** (upload the recording, enable Simulive, link the file, set the schedule).
2. The admin links it to a session in Workshop OS (`POST /sessions/{id}/zoom/link-existing`). We store the webinar ID and read its details (`GET /v2/webinars/{id}`). If `is_simulive` is present in the response, it's stored as a check value **[ASSUMPTION: field readable]**.
3. The platform manages what's possible via the API: **registrants**, **schedule verification** (alert if Zoom's start time ≠ session start), reading join URLs, webhooks, attendance and recordings.
4. Schedule changes for Simulive sessions: attempt `PATCH start_time` **[SPIKE S3: verify it works for Simulive webinars and doesn't break the Simulive configuration]**. If it isn't supported, the reschedule preview shows "Manual step in Zoom required". A manual task activity is generated, and dependents wait for confirmation.
5. The "Start Simulive" timeline activity = `zoom.simulive.verify` at T−60 min and T−10 min: webinar exists, start time matches, not cancelled. Any mismatch → critical alert. At start time, `webinar.started` confirms delivery.

**If the client's recordings are not in Zoom** (e.g. Google Drive) **[CONFIRM Q5]**: alternatives are (a) host-run "play recording" sessions (a human shares the video in a regular meeting), (b) a third-party Simulive provider (out of scope; evaluate separately), or (c) Zoom Events/Sessions if licensed.

## 5. Session ensure algorithm (idempotent)

```text
zoom.session.ensure(sessionId):
  desired = buildZoomPayload(session, workshop, deliveryConfig)   // topic, agenda incl. wos tag, start_time, duration, timezone, settings
  desiredHash = sha256(canonical(desired))
  link = SELECT * FROM zoom_sessions WHERE session_id = $1 FOR UPDATE
  if link.zoom_id is null:
      if link.status == 'creating' and link.created_attempt_at < now()-2min:
          // previous attempt may have created it (unknown outcome) → checkable lookup
          found = listUpcoming(hostUserId).find(m => m.agenda contains link.wos_tag)
          if found: adopt(found) ; goto reconcile
      mark creating; resp = create(desired) ; store id/join_url ; synced_hash = desiredHash
  else reconcile:
      if link.synced_hash != desiredHash: patch(diff) ; synced_hash = desiredHash
      if remote drift detected (remote.start_time != desired.start_time and != synced): mark drifted
```

- `wos_tag` (e.g. `[wos:ses_01J…]`) in the agenda makes creation **checkable** ([ADR-012](../adr/012-unknown-outcome-no-auto-retry.md)) **[ASSUMPTION: agenda is returned in list responses; otherwise GET each candidate]**.
- `start_url` is discarded from responses, and the password is stored encrypted.

### Drift

Admins may edit meetings directly in Zoom. On each `meeting.updated` webhook and in the daily drift check, the remote state is compared with the desired state:

- Remote ≠ desired and ≠ last synced → `sync_status=drifted`. The UI shows a diff with the actions **Apply ours** / **Accept theirs** (accepting theirs updates the session, which is a reschedule path with preview).
- No auto-overwrite in the MVP.

## 6. Webhook handling

- Endpoint per integration `POST /webhooks/zoom/{integrationId}`. URL validation challenge and signature verification **[VERIFIED]** ([api-specification §3.1](../engineering/api-specification.md#31-zoom-post-webhookszoomintegrationid)).
- Respond < 3 s. Zoom retries only on ≥ 500 (+5/+20/+60 min) and does **not** retry 4xx. Zoom revalidates the endpoint every 72 h and disables the subscription after 6 consecutive failures **[VERIFIED]**. **A sustained outage of our webhook endpoint can disable the Zoom subscription.** The ops alert `zoom.webhook_validation_failing` monitors validation requests, and reconciliation (polling meeting status/reports) covers missed webhooks.
- Mapping: payload `object.id` (meeting/webinar ID) → `zoom_sessions.zoom_id` → session. Unknown IDs (meetings not created by us) are ignored with `processing_status=ignored`.
- Recurring/rescheduled instances: `object.uuid` identifies the instance. Store the latest UUID for report calls (past-meeting endpoints take the UUID; double-encode UUIDs starting with `/` or containing `//` **[VERIFIED Zoom convention; re-check]**).

## 7. Attendance retrieval

See [participant-management §6](participant-management.md#6-attendance). Zoom specifics:

- Triggered by `zoom.session.ended`, after a settle delay (default 30 min, tuned in SPIKE S3 by measuring report availability).
- Retries with backoff for up to 24 h if the report is empty or 404 (reports can lag).
- Fallback if no `*.ended` webhook arrives: scheduled end + 2 h.
- Report fields used: participant `id`/`user_id`, `name`, `user_email`, `join_time`, `leave_time`, `duration`, `registrant_id` (if registration) **[VERIFIED fields common in report responses; confirm exact names in SPIKE S3]**.

## 8. Recordings

- `recording.completed` webhook → find the session → `GET /meetings/{id}/recordings` → store file metadata (type, size, `recording_type`) and `share_url`.
- Publishing options **[CONFIRM Q10]**:
  - (a) Share the Zoom `share_url` (+ passcode) via comms trigger. Simple, and no storage cost.
  - (b) Copy to R2 and share our presigned link. Gives control and retention independent of Zoom, but has storage cost and large transfers. This is a long-running job: streamed multipart upload, resumable per part. Zoom download URLs require auth and have limited validity **[ASSUMPTION: verify]**.
- Default: (a) in the MVP. (b) is P2.

## 9. Data entities

`zoom_sessions`, `recordings`, `attendance_records`, `webhook_events`, `integration_credentials`, `integration_rate_buckets`.

## 10. Events

Normalized: `zoom.session.started|ended`, `zoom.participant.joined|left`, `zoom.recording.completed`, `zoom.session.updated_externally`, `zoom.session.deleted_externally`.

## 11. Error scenarios (PRD Case 2)

| Scenario | Behavior |
|---|---|
| Webinar creation fails (license/validation) | Session `needs_attention`. **Dependent joining instructions blocked.** Admin notified. Manual recovery: fix + "Retry Zoom", or link an existing meeting |
| 429 daily limit reached | `rate_limited_daily`. If the session starts before 00:00 UTC reset → critical alert ("update manually in Zoom, then accept drift") |
| Token request fails (S2S app deactivated) | Integration `auth_error`, banner, all Zoom actions `needs_attention` |
| Meeting deleted in Zoom | `meeting.deleted` → session `needs_attention` + "Recreate" (new join URL → triggers `joining_link_changed` workflow) |
| Report API not available on plan | Capability `reports=false` → attendance via webhooks only (marked "provisional"). Warn at publish |

## 12. Security

- S2S credentials and the webhook secret token are encrypted. Least-privilege granular scopes (meeting/webinar read/write, report read, recording read, user read).
- Host start URLs are never stored, logged or sent to participants. The `GET host-start` endpoint is permissioned (`sessions.host`: trainers of that session, admins) and audited.
- Per-registrant join URLs are encrypted at rest and only placed into comms variables for that recipient.

## 13. Edge cases

- Session duration change only → still a PATCH (counts toward budget).
- Two sessions sharing one Zoom meeting (e.g. morning/afternoon in one meeting) → supported via `delivery_config.sharedWith` (one Zoom object, multiple sessions). Attendance split by time windows **[CONFIRM if needed]**.
- Host user leaves the Zoom account → ensure fails with user-not-found → `needs_attention`, with "reassign host".
- Participants join early (join-before-host) → attendance intervals before scheduled start are clipped or included (configurable).

## 14. Testing

- Contract fixtures (MSW) for each endpoint, including 429 daily, 404 report, pagination, and webhook payloads (all subscribed events) with signature generation.
- **SPIKE S3** against a real Zoom test account (Pro + Webinar if available): create/patch/delete meeting & webinar, registrants, end meeting, report availability delay, recording webhook, `simulive_delay_start` behavior, PATCH on Simulive webinar.
- Simulated outage: the webhook endpoint returns 500 for 10 min → Zoom retries → processed exactly once.

## 15. Acceptance criteria

1. Publishing a workshop creates one Zoom object per session. A retry after a simulated timeout doesn't create a duplicate.
2. Rescheduling updates the Zoom start time with exactly one PATCH per session.
3. "Start as host" always opens a fresh, working host link. No host link appears in logs or DB.
4. Attendance for an ended session is imported automatically and matches the Zoom report.
5. A linked Simulive session shows verification status, and alerts on schedule mismatch.

## 16. Implementation dependencies

Credential vault → Zoom adapter (auth, rate buckets) → webhook ingress → `zoom.session.ensure` → registrants → attendance import → recordings → drift detection. **SPIKE S3 first.**

## 17. Trade-offs

- Linked Simulive (manual setup) vs waiting for API support: unblocks delivery now, with a manual step.
- Fresh `start_url` per click vs caching: one extra API call per host start, and no leaked or expired links.
- No auto-overwrite of drift: avoids fighting humans, at the cost of manual resolution.

## 18. Future improvements

- Meeting SDK embed for hosts (ZAK-based start inside the dashboard).
- Zoom Events/Sessions API integration if licensed.
- Automatic host distribution across multiple Zoom users to stay under daily limits.

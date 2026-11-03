# Google Workspace Integration: Sheets, Calendar, Docs

Related: [ADR-011](../adr/011-google-sheets-sync-ownership.md) · [participant-management.md](participant-management.md) · [integration-architecture.md](../architecture/integration-architecture.md) · [security.md](../engineering/security.md)

---

## 1. Purpose and scope

| Service | MVP | Purpose |
|---|---|---|
| **Google Sheets** | ✅ | Import participants, detect new rows, sync statuses/attendance/completion, export reports (PRD §5.4) |
| **Google Calendar** | ✅ | Workshop and session events, trainers as attendees, Zoom details, updates/cancellations, no duplicates (PRD §5.5). Free/busy: P2 |
| **Google Docs** | P2 | Generate schedule, attendance sheet, trainer assignments, completion report, summary. Certificates: future (PRD §5.6) |

## 2. Authentication and authorization

### 2.1 Options

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. OAuth user consent** (a dedicated "workshop-ops@" Google account) | Authorization code + PKCE, `access_type=offline` → refresh token | Works with personal Google accounts and Workspace. Files owned by a service-like user | Refresh token can be revoked. Testing-mode tokens expire in 7 days for External apps **[COMMUNITY]**. Sensitive/restricted scopes may need verification for External apps |
| **B. Service account + domain-wide delegation** | SA impersonates a Workspace user | No user consent flow. Admin-controlled | **Workspace only**. Requires a Workspace super-admin to grant delegation. Broad power, so it must be scoped tightly |
| **C. Service account without delegation** | Share specific sheets/calendars with the SA email | Simple, least privilege per file | SA calendars aren't the client's calendars. Trainers must be attendees. Docs created are owned by the SA (quota/ownership issues) |

**Recommendation:** Option A with a dedicated ops account, using an OAuth app with **User type = Internal** if the client uses Google Workspace (no Google verification needed for internal apps). Otherwise External, **published to "In production"** **[CONFIRM Q8]**.

### 2.2 Scopes (least privilege)

| Need | Scope | Sensitivity |
|---|---|---|
| Read/write bound sheets | `https://www.googleapis.com/auth/spreadsheets` (or `drive.file` if files are created/picked by the app) | Sensitive (spreadsheets) / non-sensitive (drive.file) **[ASSUMPTION: re-check Google's scope classification]** |
| Calendar events | `https://www.googleapis.com/auth/calendar.events` | Sensitive |
| Free/busy (P2) | `https://www.googleapis.com/auth/calendar.freebusy` or `calendar.readonly` | Sensitive |
| Docs generation (P2) | `documents` + `drive.file` (copy templates the user picked via Google Picker) | |

`drive.file` restricts access to files the app created or the user explicitly opened with the app, which is the least-privilege path. Using pre-existing client sheets with `drive.file` requires picking them via the Google Picker. Otherwise `spreadsheets` is required. **Decision:** use the Google Picker for binding sheets/templates, plus `drive.file` + `calendar.events`. Fall back to `spreadsheets` if the Picker is impractical.

## 3. Google Sheets synchronization

### 3.1 Change detection options

| Mechanism | What it gives | Limits | Use |
|---|---|---|---|
| **Polling reconciliation** (`spreadsheets.values.batchGet`) | Full truth of the mapped range | Quota: 60 reads/min/user/project, 300/min/project **[VERIFIED]** | **Primary** (every 5 min per active binding) |
| Apps Script installable triggers (`onChange`, `onFormSubmit`) in the sheet, calling our webhook | Near-real-time "something changed" nudge | Requires script installed in the sheet. Edits made via the API don't fire edit triggers **[ASSUMPTION: Apps Script trigger semantics; verify]**. Script quotas | **Optional nudge** → schedule reconcile now |
| Drive API push notifications (`files.watch`) | File-level "changed" notification | No row detail. Channels expire and must be renewed. Needs a verified HTTPS endpoint | Not used in MVP |
| Google Forms → Sheet | If registrations come from Forms, `onFormSubmit` gives the row | Same Apps Script caveats | Optional nudge |

The PRD is right that **Google Sheets has no general-purpose row-change webhook**. Scheduled reconciliation is mandatory.

### 3.2 Binding configuration

```jsonc
{
  "spreadsheetId": "1AbC…",
  "sheetName": "Registrations",
  "headerRow": 1,
  "keyColumn": "WOS ID",                 // created by the platform if absent (rightmost column)
  "columns": [
    { "header": "Full Name",      "field": "participant.fullName",       "owner": "inbound" },
    { "header": "Email",          "field": "participant.email",          "owner": "inbound" },
    { "header": "Phone",          "field": "participant.phone",          "owner": "inbound" },
    { "header": "Payment",        "field": "registration.paymentStatus", "owner": "inbound" },   // [CONFIRM Q4]
    { "header": "Status",         "field": "registration.status",        "owner": "outbound" },
    { "header": "Attendance %",   "field": "registration.attendancePct", "owner": "outbound" },
    { "header": "Completion",     "field": "registration.completionStatus","owner": "outbound" },
    { "header": "Registered At",  "field": "registration.registeredAt",  "owner": "inbound_once" }
  ],
  "direction": "bidirectional",
  "pollInterval": "PT5M",
  "valueRenderOption": "UNFORMATTED_VALUE",
  "dateTimeRenderOption": "SERIAL_NUMBER"
}
```

Validation on save: headers exist (exact match after trim, case-insensitive). The key column exists or can be created. No field is mapped twice. A **preview diff** runs before enabling.

### 3.3 Reconciliation algorithm (`sheets-reconcile`, concurrency 1 per binding)

```text
1. step read: values.batchGet(header range + data range)               — 1 read request
2. step parse: map headers → columns; detect header changes → binding 'needs_attention' (stop)
3. step diff:
   for each row:
     key = row[keyColumn]
     if key empty: candidate new row → normalize → intake.ingest(source='sheet', sourceRef=rowFingerprint)
     else: link = sheet_row_links[key]
           if link missing (key typed by human / copied row) → review 'ambiguous_row'
           inboundHash = hash(inbound columns)
           if inboundHash != link.inbound_hash → apply inbound changes (owner rules) → update link
   rows whose key disappeared → mark link 'row_missing' (no deletion of registrations) → warning
4. step write-back (single values.batchUpdate, ≤ 2 MB):                — 1 write request
   - key column for newly created registrations
   - outbound columns whose DB value != sheet value (owner=outbound) → overwrite + log reverts
5. step log: integration_sync_logs (stats, bounded change list)
```

- **New-row identity before the key is written:** fingerprint = hash(normalized email + phone + row's inbound values). Intake is idempotent on `(binding, fingerprint)` via `sheet_row_links`, so if write-back fails, the next reconcile doesn't create duplicates.
- **Row numbers are never trusted** between reads. Write-back uses row numbers from **this** read only, and the window between read and write is seconds. Edits made inside that window can be overwritten for outbound columns only (acceptable). Inbound columns are never written.
- **Formulas/formatting:** we write only values to mapped outbound cells (`valueInputOption=RAW`). Other columns are untouched.

### 3.4 Outbound sync (event-driven)

- Triggered by `registration.*` / `attendance.imported` / `registration.completion_evaluated` events → `sheets-export` with Inngest `debounce { key: bindingId, period: '30s' }` and `concurrency { key: bindingId, limit: 1 }`.
- Implementation = steps 1, 4 and 5 above (read keys → batchUpdate changed outbound cells).
- Reports export: a new tab or new spreadsheet with a snapshot (`values.update` on a fresh range).

### 3.5 Failure handling (PRD Case 3)

- Changes are always saved in Postgres first. Sync failure → `retrying` with truncated exponential backoff (`min(2^n + rand(1000ms), 64s)` per Google guidance **[VERIFIED]**), capped attempts, then `failed`. Alert after 3 consecutive reconcile failures.
- **Workshop operations continue.** Sheet sync activities are `criticality=low`.
- 429 → respect quota windows (refill per minute **[VERIFIED]**). The Postgres token bucket `sheets:user` limits us to 50/min (headroom).
- Sheet deleted/permission revoked → binding `needs_attention`, integration banner.

## 4. Google Calendar synchronization

### 4.1 Model

- One **calendar** per deployment (e.g. "Workshops", owned by the ops account) **[CONFIRM Q8]**. Trainers are added as **attendees** (they see events in their own calendars).
- Events: one per **session** (primary). Optional all-day event per workshop span.
- **Deterministic event IDs** prevent duplicates: `id = base32hex(sha256("wos:" + entityType + ":" + entityId + ":" + calendarId))[0:26]`. Allowed characters are a–v and 0–9, length 5–1024 **[VERIFIED]** ([events.insert](https://developers.google.com/workspace/calendar/api/v3/reference/events/insert)). An insert retry that hits "already exists" → treat as success → `events.get` → patch if hash differs.
- **Recreation after deletion:** Google may not allow reusing the ID of a deleted event **[ASSUMPTION; verify in SPIKE S5]**. Fallback: suffix `-r{n}` and record it in `calendar_event_links`.

### 4.2 Event content

- `summary`: "{Workshop} — {Session}". `start/end`: `dateTime` + `timeZone` (workshop tz).
- `location` / `description`: Zoom join URL (general, not per-registrant), meeting ID, passcode (if the client agrees to put it in calendar **[CONFIRM]**), and a link to the Control Center for staff.
- **Not using `conferenceData` for Zoom.** Adding third-party conferencing as structured conference data generally requires a conferencing add-on **[ASSUMPTION]**. Plain text is reliable. `conferenceDataVersion` stays 0.
- `attendees`: trainers (+ optional staff). **Participants are not added as attendees** by default: messaging goes through the existing comms systems, and Calendar invites to hundreds of participants would duplicate notifications and expose attendee lists **[CONFIRM Q9]**.
- `sendUpdates`: `all` for trainer changes (so trainers get updates). The docs note that `none` can cause external calendar sync issues **[VERIFIED]**.

### 4.3 Operations

| Trigger | Operation |
|---|---|
| Publish / session created after publish | `calendar.event.ensure` (insert with deterministic ID / patch) |
| Reschedule | `events.patch` start/end (after Zoom update, in the reschedule workflow) |
| Trainer change | patch attendees |
| Session/workshop cancel | `events.delete` (or patch `status: cancelled`) with `sendUpdates=all` |
| Free/busy (P2) | `freebusy.query` for trainers' calendars (requires their sharing/authorization) → validation warnings |

Drift: Calendar edits by humans → daily check compares `etag`/fields → `drifted` warning (no auto-overwrite in MVP, consistent with Zoom).

## 5. Google Docs generation (P2)

- Admin picks a template Doc via Google Picker once (`document_templates`).
- Generation:
  1. `drive.files.copy(templateId)` → new doc in the output folder.
  2. `docs.documents.batchUpdate` with `replaceAllText` for `{{placeholders}}`.
  3. Tables (e.g. attendance lists): insert rows via `insertTableRow` + `insertText` requests, or prefer **Sheets for tabular reports** and Docs for narrative ones. That's simpler and more robust.
  4. Optional PDF: `drive.files.export(mimeType=application/pdf)` (export size limits apply **[ASSUMPTION: ~10 MB; verify]**) → store in R2 + `generated_documents`.
- Idempotency: `checkable`. The doc title includes a `wos:{id}` tag, and before copying we search the output folder for an existing doc with the tag.
- Certificates (future): per-participant docs mean N copies + N exports. That's heavy on Drive quotas and slow. Consider HTML→PDF rendering in our own service instead **[decision deferred]**.

## 6. Data entities

`integrations` (google), `integration_credentials`, `sheet_bindings`, `sheet_row_links`, `calendar_event_links`, `document_templates`, `generated_documents`, `integration_sync_logs`, `integration_rate_buckets`.

## 7. API contracts

Sheet bindings, reconcile, preview: [api-specification.md §2.7](../engineering/api-specification.md#27-integrations). Google OAuth start/callback (state + PKCE). Apps Script nudge webhook §3.4.

## 8. Events

Consumes `registration.*`, `attendance.imported`, `session.*`, `workshop.*`. Emits `sheets.rows.changed` (internal), `sheets.binding.needs_attention`, `calendar.event.synced|failed|drifted`.

## 9. Security

- Refresh token encrypted. Scopes minimal. The OAuth `state` is bound to the user session and PKCE is used.
- The Apps Script nudge secret is per binding and rotatable. The nudge carries no data (it only schedules a reconcile).
- Sheets contain PII, so the binding UI warns when the spreadsheet is shared publicly ("anyone with link") if detectable via the Drive API **[ASSUMPTION: permissions readable with granted scope]**.

## 10. Edge cases

- Human sorts the sheet between read and write → write-back may target wrong rows for outbound cells. Mitigation: re-read the key column immediately before `batchUpdate` and abort the write if any targeted row's key changed (optimistic check). Retry on the next cycle.
- Duplicate key values (row copied by a human) → both rows flagged. Neither is updated until resolved.
- Merged cells / filtered views → values API ignores filters. Merged cells read as the top-left value only. Bindings should avoid merged cells (validation warning).
- Locale-formatted numbers/dates → use unformatted values and serial dates. Convert with workshop tz.
- Sheet with > 10,000 rows → paginate reads by range blocks. Still a few requests.

## 11. Testing

- Reconciliation property tests: random human edit sequences (insert, sort, edit inbound/outbound, delete rows) → invariant: no duplicate registrations, outbound columns converge to DB, inbound changes applied once.
- Contract fixtures for Sheets/Calendar/Docs responses, including 429 and 403.
- SPIKE S5: real Google test project with OAuth app in Testing and In-production modes, deterministic Calendar IDs (insert twice, delete + reinsert), Picker + `drive.file` access to existing sheets.

## 12. Acceptance criteria

1. A new row in the bound sheet becomes a registration within one poll interval and receives a `WOS ID`.
2. Editing an outbound column in the sheet is reverted on the next sync and logged.
3. Re-running Calendar ensure never creates duplicate events.
4. A Sheets outage doesn't affect any non-sheet activity, and changes sync after recovery.

## 13. Implementation dependencies

OAuth + credential vault → Sheets adapter + rate bucket → binding editor + preview → reconcile + intake → outbound export → Calendar adapter → (P2) Docs.

## 14. Trade-offs

- Polling adds latency (≤ 5 min) and quota use, but it's reliable and the only general solution.
- Column ownership is less flexible than full bidirectional merge, but predictable.
- No participant Calendar invites means fewer touchpoints, avoids duplicate notifications and protects privacy.

## 15. Future improvements

- Apps Script add-on packaged for one-click installation of the nudge trigger.
- Calendar free/busy-based trainer scheduling assistant.
- Docs-based certificates or a dedicated PDF rendering service.

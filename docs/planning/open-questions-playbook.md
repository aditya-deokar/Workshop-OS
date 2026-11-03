# Open Questions Playbook and Gap Analysis

This document has three parts:

- **Part A: Decision playbook.** For every open question in [open-questions.md](open-questions.md), it lists the realistic answers the client might give and what we build in each case: design changes, documents affected, effort impact and risk. When an answer arrives, engineering picks the matching branch. No architecture re-think is needed.
- **Part B: Default bundle.** What we build if a question is still unanswered when its phase starts.
- **Part C: Gap analysis.** Problems and missing pieces found by re-reviewing the existing documentation, each with a concrete solution.

Related: [open-questions.md](open-questions.md) · [risks-and-tradeoffs.md](risks-and-tradeoffs.md) · [roadmap.md](roadmap.md) · [ADR-013](../adr/013-recovery-events-and-check-wait-loops.md)

**Effort impact** is relative to the baseline roadmap: **0** = no change, **+S** = days, **+M** = 1–2 weeks, **+L** = 3+ weeks.

---

# Part A: Decision playbook

## Q1: Where do the existing WhatsApp and email automations run?

| # | Possible answer | What we build | Idempotency class (default) | Effort | Risk |
|---|---|---|---|---|---|
| 1a | **Custom backend with a REST API** (theirs or a vendor's) | Use our contract exactly ([whatsapp-email §4–5](../features/whatsapp-email-integration.md#4-outbound-trigger-contract)). Ask them to honor `Idempotency-Key` and send callbacks | `keyed` once verified | 0 | Low |
| 1b | **n8n** (cloud or self-hosted) | One **router webhook** in n8n per channel (`/webhook/wos-whatsapp`) that switches on `trigger`, instead of one webhook per trigger. **We supply an n8n template workflow** that (1) verifies our HMAC, (2) checks `Idempotency-Key` in an n8n Data Table / Postgres node and returns `409` on duplicates, (3) calls their existing sub-workflows, (4) replies via a "Respond to Webhook" node with `{ accepted, externalExecutionId }`, (5) posts delivery callbacks to us with an HTTP Request node | `keyed` (via template) | +S | Low |
| 1c | **Make (Integromat)** | Custom webhook per channel plus a router. A plain Make webhook responds `Accepted` before the scenario runs **[ASSUMPTION]**, so we add a **Webhook Response** module to return a real result, or we treat everything as `accepted` and depend on callbacks. Idempotency via a Make Data Store lookup. Note: every Make operation costs Make credits, so prefer bulk payloads | `keyed` if Data Store check added, else `unsafe` | +S | Medium (ops cost, queueing delay) |
| 1d | **GoHighLevel** | Contact-centric. The adapter must **upsert a GHL contact** (by phone/email), store `contactId` in `external_links`, then trigger a workflow (inbound-webhook trigger or "add contact to workflow" API **[CONFIRM exact API]**). **Pitfall:** if their templates read variables from **contact custom fields** (e.g. `joining_link`), two concurrent workshops overwrite each other's values. Solution: pass variables in the webhook payload and map them inside the workflow, or use per-workshop custom fields | `unsafe` (no documented dedupe) | +M | Medium–High |
| 1e | **WhatsApp BSP tool** (WATI, Interakt, AiSensy, Gupshup, etc.) where the "automation" is a template send | Adapter calls the BSP's "send template message" API with template name + parameters. The trigger registry maps `trigger_key → template_name + parameter order`. Delivery webhooks from the BSP map to our callback status. Stays within scope: we still don't manage templates or campaigns | Usually `unsafe` | +M | Medium |
| 1f | **Email ESP automation** (Brevo, Mailchimp, SendGrid, Mailmodo, Zoho Campaigns, etc.) | Two variants: (i) **custom-event trigger** (track event `workshop_reminder` with properties → ESP automation sends). Same shared-contact-field pitfall as 1d. (ii) **Transactional template API** with per-send variables, preferred because variables are per message. Delivery via ESP webhooks | ESP-dependent | +M | Medium |
| 1g | **No API at all** (messages sent by hand, or a tool that only accepts CSV uploads) | We can't automate sending. Options to present to the client: **(1)** comms activities become **manual tasks** ("Send reminder campaign. List exported here"), with a CSV export of recipients and variables and an operator confirmation. **(2)** We deploy a small **n8n relay** that they connect to their WhatsApp/email provider. That is new infrastructure, but not new messaging software. **(3)** Re-scope | n/a | +M to +L | **High.** Tell the client immediately |
| 1h | **Mixed** (e.g. WhatsApp in a BSP, email in an ESP) | Two adapter configs. The trigger registry is per integration, so no design change | per adapter | +S per extra platform | Low |

**Follow-up questions per branch:** auth method (header token vs HMAC), max request rate, whether a static egress IP is needed (→ NAT gateway, [deployment §4](../engineering/deployment.md#4-reference-hosting-proposed-confirm-q16-q17)), callback support, bulk support, test environment.

**Default if unanswered:** build the generic webhook adapter (1a contract) and an n8n relay template (1b). All triggers start as `unsafe`.

## Q2: Idempotency, test triggers and WhatsApp template category

| Answer | What we do |
|---|---|
| **They can dedupe by `Idempotency-Key`** | Verification flow → `keyed` → auto-retry on timeouts. Lowest ops burden |
| **They can't today, but are willing to add it** | Give them a 1-page spec: store the key for ≥ 7 days, and return `409` + the original `externalExecutionId` on repeats. Until it's verified, run as `unsafe` |
| **They can't, but expose a lookup** ("get execution by requestId") | `checkable`: on unknown outcome, look up first, then retry if absent |
| **No dedupe, no lookup** | `unsafe`: no auto-retry on unknown outcome ([ADR-012](../adr/012-unknown-outcome-no-auto-retry.md)). Mitigations: short timeouts (5 s connect / 15 s total) to keep the ambiguous window small, and bulk "resolve" tooling. Expect more manual work during outages |
| **Test triggers available** (same workflow, internal recipients only) | Staging uses them. Nightly live smoke test |
| **No test triggers** | Staging uses the mock comms server only. Production rehearsals use **rehearsal mode** with allow-listed test recipients ([G-13](#g-13-rehearsal-test-mode-is-missing)) |
| **Template category = utility** | Lower WhatsApp cost. Billed only outside the customer service window **[VERIFIED]**. No design change |
| **Template category = marketing** | Every send is billed **[VERIFIED]**. Turn on the frequency cap by default ([G-12](#g-12-no-participant-message-safety-guard)) and show an estimated message count in the timeline preview |

## Q3: How are participants managed today?

| # | Possible answer | What we build | Sheet column ownership | Effort |
|---|---|---|---|---|
| 3a | **One Google Sheet per workshop**, filled manually | One `sheet_binding` per workshop, created from a **binding template** (saved column map). Duplicate workshop → prompts for a new sheet | Identity columns `inbound`. Status/attendance `outbound` | 0 (baseline) |
| 3b | **One master sheet for all workshops** with a "Workshop" column | **Global binding** with a `routing_column` → workshop code. Reconciliation filters rows per workshop. A row with an unknown code → review queue. Add `sheet_bindings.scope = 'global'` and `routing_column` | Same + `Workshop` column `inbound_once` | +S |
| 3c | **Google Forms → Sheet** | Binding on the form-response sheet. Rows are append-only, so most columns are `inbound_once`. Optional `onFormSubmit` nudge for instant import. **Never write into the form-response tab** (Forms may append to it). Outbound columns go to a **separate mirror tab** we own | inbound_once + mirror tab | +S |
| 3d | **Website form / payment page** (WordPress, Razorpay/Instamojo payment pages, Typeform, etc.) | The form posts to our **intake API** (directly, or via Zapier/n8n if the form can't sign requests). The sheet becomes **export-only** (outbound mirror). Strongly preferred: real-time and validated | Sheet fully `outbound` | +S |
| 3e | **A CRM** (HubSpot, Zoho, GHL, Salesforce) | CRM is the upstream source of identity: build a **CRM inbound adapter** (webhook or poll) feeding the intake service. Workshop operational state still lives in Postgres. Optional write-back of status to the CRM. P2 unless the CRM is the only source | Sheet optional | +M |
| 3f | **Zoom registration** is the registration system | Import registrants via API on schedule, plus registration webhooks if available **[ASSUMPTION: verify event names in SPIKE S3]**. Zoom becomes an intake channel, and per-registrant links come for free | — | +S |
| 3g | **Mixed** (form + manual sheet additions) | Multiple intake channels → single intake service → cross-channel dedupe (already designed). Each channel's `source` is recorded | per channel | +S |

**Also ask:** which columns do humans edit **after** registration (payment, notes, cancellations)? Those become `inbound`. Everything else is `outbound`.

**Default if unanswered:** 3a with the column map from [google-workspace §3.2](../features/google-workspace.md#32-binding-configuration).

## Q4: Where does payment status come from?

| Answer | What we build | Effort |
|---|---|---|
| **Payment gateway webhooks** (Razorpay, Stripe, Cashfree, PayU) | A **payment-status adapter** (inbound only; we don't process payments): verify the gateway's webhook signature → map payment to registration. Mapping key priority: (1) our `registrationId` passed as order notes/metadata by their form, (2) email/phone match → review if ambiguous. Statuses mapped to `payment_status`. Refund webhooks → `refunded` → workflow decides | +M |
| **Sheet column** | `Payment` column `inbound`. Values normalized through a mapping table (`"Paid" / "PAID" / "Yes" → paid`). Unknown values → `unknown` + warning | 0 |
| **Manual by staff** | Registrations table: bulk "Mark paid" with reason. Optional CSV upload of payment references | +S |
| **Free workshops** | Payment gate off (`payment_required=false`). Status `not_required` | 0 |
| **Comes with registration** (payment page = registration form) | Intake API payload carries `payment.status` (already in contract) | 0 |
| **CRM/accounting system** | Same adapter as Q3e | +M |

Refund/chargeback behavior options: (1) auto-cancel registration + notify, (2) flag for review (**default**), (3) ignore.

## Q5: Zoom setup and recorded delivery

### Q5.1 Delivery model

| # | Answer | Session delivery mode | What changes | Effort |
|---|---|---|---|---|
| 5a | **Live sessions in Zoom Meetings** | `zoom_meeting` | Baseline | 0 |
| 5b | **Live Zoom Webinars** | `zoom_webinar` | Webinar license needed. Panelists = trainers (`/webinars/{id}/panelists` **[ASSUMPTION verify]**). Registration usually on | +S |
| 5c | **Zoom Simulive** (Webinars + Events/Sessions license) | `zoom_simulive_linked` | Linked mode + verification ([zoom §4](../features/zoom-integration.md#4-simulive-what-is-and-isnt-possible)). Add a **"Simulive setup checklist" manual task** auto-created at publish (−3 days) for each linked session | +S |
| 5d | **Recordings in Google Drive / elsewhere, played by a host inside a meeting** | `zoom_meeting` + new activity type **`host_playback_check`** | Before the session: verify the Drive file exists and is accessible to the host (Drive API `files.get`, needs scope), and send the host a checklist + file link. During: a manual task "playback started" | +S |
| 5e | **On-demand recordings only** (no live/simulated session) | New mode **`on_demand`** (no Zoom object) | Access via published resource links ([G-14](#g-14-no-participant-facing-link-endpoint)). Attendance = link opens (weak signal), so completion rules must not rely on Zoom attendance | +M |
| 5f | **Zoom Basic (free) plan** | — | Not viable: low API limits, no reports, meeting duration limits. **Recommend a paid plan** before build | Blocker |

### Q5.2 Zoom plan tier

| Plan | Impact |
|---|---|
| Pro | Reports are available on some endpoints **[COMMUNITY]**. Rate limits Pro tier **[VERIFIED]**. Cloud recording included at plan level **[LICENSE]** |
| Business+ | Higher rate limits. Dashboard APIs available **[COMMUNITY]** |
| Enterprise | Same as Business+, plus account-level controls |

### Q5.3 Number of host users

| Answer | What we do |
|---|---|
| **One host user** | No overlapping sessions across concurrent workshops (validation blocks them). 100 create/update calls per day budget shared by everything **[VERIFIED]**. The reschedule preview must show the remaining budget |
| **Several host users** | Add a **host pool**: `zoom_host_pool (integration_id, zoom_user_id, active, max_concurrent=1)`. At publish, assign each session a host that is free in its time window (greedy interval assignment) and spread the daily budget. Trainer-as-host if a trainer has a Zoom user |

### Q5.4 Zoom registration for participants

| Answer | What we do | Trade-off |
|---|---|---|
| **Yes** (recommended) | `requires_registration=true` → add registrants on confirmation → per-registrant `join_url` in comms variables → attendance matched by `registrant_id` (confidence 1.0) | One registrant API call per participant. Cancellation must cancel the registrant ([G-26](#g-26-zoom-registrant-lifecycle-not-fully-synced)) |
| **No** | One general join link. Attendance matched by email (only if attendees sign in) or name → more review. Option: Zoom's "only authenticated users can join" setting improves email capture but adds friction | Less accurate attendance |

## Q6: Completion rules and assessments

| Sub-question | Options | Default |
|---|---|---|
| Threshold | 50 / 75 / 80 / 100% | **75%** overall |
| Denominator | Scheduled duration (**default**, fair if a session overruns) · actual duration (fair if a session ends early) · fixed minutes | Scheduled |
| Per-session requirement | None · each session ≥ X% · specific sessions mandatory | None |
| Assessment source | (a) **Google Forms quiz → Sheet**: assessment-results binding ([G-17](#g-17-assessments-have-no-data-model)). (b) **LMS** (Moodle, Teachable, etc.): adapter, P2. (c) **Zoom webinar polls/Q&A reports** **[ASSUMPTION: report endpoints exist; verify]**. (d) **None** | (d), then (a) when needed |
| Borderline handling | Auto-complete · review queue for ±5% (**default**) · strict | Review |
| Exemptions | Admin marks `exempt` with reason | Allowed |

All of these are values in the `completion_rule` JSON ([participant-management §6.3](../features/participant-management.md#63-computation)), so changing them is configuration, not code. Assessments need [G-17](#g-17-assessments-have-no-data-model).

## Q7: Can participants share an email or phone?

| Answer | What we do |
|---|---|
| **No, both are unique per person** | Keep the partial unique indexes on `email_normalized` and `phone_e164` |
| **Shared emails happen** (family, corporate inbox) | Drop the email unique index. Identity = (email + normalized name) or phone. Matching on email alone → `needs_review` |
| **Shared phones happen** (one WhatsApp number for several attendees) | Drop the phone unique index. **WhatsApp dedupe per phone per trigger** becomes important: two registrations sharing a phone would get two identical reminders. Add a "collapse by phone" option on comms activities (send once per phone, with a combined name variable) |
| **Both** | Identity = email + phone + name. Higher review volume. Consider making `participants.external_id` from their source system the primary key of identity |

## Q8: Google account model

| # | Answer | Auth design | Notes |
|---|---|---|---|
| 8a | **Google Workspace, and the admin will cooperate** | **Internal** OAuth app + dedicated `workshop-ops@client.com` user (**recommended**). No Google verification needed for Internal apps | Calendars/sheets owned by the ops user. Trainers added as attendees |
| 8b | Workspace, and the admin prefers no user consent | Service account + **domain-wide delegation** limited to `calendar.events` + `drive.file` + `spreadsheets`, impersonating the ops user | Requires super-admin setup. Protect the SA key in KMS |
| 8c | **Workspace, but no admin cooperation** | External OAuth app (or Internal if a regular user can create it **[ASSUMPTION]**), published **In production** | Possible "unverified app" warning for sensitive scopes. Usage limits for unverified apps **[ASSUMPTION: verify current Google policy]** |
| 8d | **Personal Gmail accounts** | External app, **must** be published In production (Testing-mode refresh tokens expire after 7 days **[COMMUNITY]**). Plan for Google verification if required by scope/usage | Verification can take weeks. Start early |
| 8e | **Can't create a dedicated ops account** (must use an employee's account) | Works, but add an **ownership transfer runbook**: if that person leaves, their token is revoked and their files may be deleted. Recommend moving files to a Shared Drive (Workspace only) | High operational risk. Flag to the client |

## Q9: Trainers and Calendar invites

| Answer | What we do |
|---|---|
| **Trainers log in** | `trainer` role with scope = own sessions: view schedule, start as host, confirm manual tasks, see attendance. Invite flow links `trainer_profiles.user_id` |
| **Trainers don't log in** | Trainers receive comms (needs trainer recipients, [G-06](#g-06-audience-and-recipient-types-are-underspecified)) and Calendar invites. "Start as host" → a signed short link (`/h/{token}`, 10-min validity, single use) that fetches a fresh `start_url` and redirects. Same security as login, with no password |
| **Participants get Calendar invites: no** (default) | As designed |
| **Participants get Calendar invites: yes** | Per-session events with participants as attendees **hidden from each other** (`guestsCanSeeOtherGuests=false`), `sendUpdates=externalOnly`. Per-event attendee limits and Calendar quotas apply **[ASSUMPTION: verify limits]**. Better alternative: the existing email system attaches an **.ics file**. We provide an `.ics` endpoint per session/registration. Calendar API not needed |

## Q10: How should recordings and materials be shared?

| Answer | What we do | Storage cost |
|---|---|---|
| **Zoom share link** (recordings) | Store `share_url` + passcode (encrypted). Expiry follows Zoom recording settings | None |
| **Our hosted links, time-limited** | `/r/{token}` public redirect endpoint ([G-14](#g-14-no-participant-facing-link-endpoint)) → presigned R2 GET (5 min). Token validity configurable (default 30 days) | R2 storage |
| **Our hosted links, permanent** | Same endpoint with non-expiring tokens + revocation list (unpublishing revokes). Per-participant tokens allow view tracking and selective revocation | R2 storage |
| **Google Drive links** | Store the Drive link. Optionally set "anyone with link" via Drive API (needs `drive` scope, which is restricted, so avoid). Default: client manages sharing | None |
| **Recordings must be copied out of Zoom** (retention independence) | P2 recording copy job ([zoom §8](../features/zoom-integration.md#8-recordings)) | High (GBs per session) |

## Q11: Is Inngest Cloud acceptable for data protection?

| Answer | What we do | Effort |
|---|---|---|
| **Yes** | As designed (IDs-only payloads, PII scanner test) | 0 |
| **Yes, with conditions** (DPA, encryption) | Sign Inngest's DPA **[ASSUMPTION: offered]**. Enable encryption middleware for all step outputs and event data **[ASSUMPTION: v4 support]**. Document data categories sent | +S |
| **No third-party processors for workflow state** | Option A: **self-host Inngest** (SSPL server; production needs Redis + Postgres; no guaranteed support **[VERIFIED]**). Ops +1 Redis +1 Postgres DB. Option B: **swap the executor** behind the `ExecutionScheduler` port for a Postgres-based engine (pg-boss with our own sleep/claim loop, or DBOS). Our design already keeps truth in Postgres, so this is mainly the scheduling layer | A: +M (ops). B: +L |
| **Data must stay in India** | Self-host (A) in an India region, or Inngest's region options if any **[CONFIRM with Inngest]** | +M |

SSPL obligations mostly concern offering the software as a service to others. Internal use is generally considered unaffected. **Have the client's legal team confirm. This is not legal advice.**

## Q12: Expected scale

| Profile | Example | Infra | Inngest plan | Design switches |
|---|---|---|---|---|
| **Small** | ≤ 5 workshops/month, ≤ 300 participants, ≤ 2 concurrent | `APP_ROLE=all` × 2 instances, small managed Postgres | Pro (Free's 50k executions/month and 5 concurrent steps are too small **[VERIFIED]**) | Per-participant fan-out OK |
| **Medium** (baseline) | 10–20 workshops/month, ≤ 2,000 participants, ≤ 10 concurrent | api ×2 + worker ×2 | Pro | Fan-out OK. Bulk triggers recommended |
| **Large** | 50+ workshops/month, 10k+ participants per workshop, 20+ concurrent | api ×2–4, worker ×4+, Postgres with read replica, `action_attempts` partitioning from day 1 | Business (500 concurrent steps **[VERIFIED]**) | **Bulk triggers mandatory**. Fan-out in chunks of 100. Staggered reminder windows. Sheets sync per workshop may hit 60/min/user quota → multiple Google ops users |

## Q13: Where do operational alerts go?

| Answer | Implementation | Note |
|---|---|---|
| **Slack** | Incoming webhook per channel (critical vs others) | Simple. Recommended |
| **Microsoft Teams** | Teams workflow/incoming webhook **[ASSUMPTION: connector deprecation status, verify]** | |
| **Email to ops** | **Separate transactional provider** (e.g. Amazon SES, Postmark), **not** the client's marketing/automation email system | Avoids circular failure |
| **SMS / phone call for critical during live workshops** | PagerDuty/Opsgenie integration, or an SMS provider | Recommended if someone is on call during live sessions |
| **WhatsApp to staff via their existing system** | Allowed only as a **secondary** channel | If their system is down, alerts about it can't arrive |

All of these plug into the `AdminNotifier` port. Combine with alert routing ([G-20](#g-20-alert-routing-and-on-call-are-missing)).

## Q14: Is SSO required?

| Answer | What we do |
|---|---|
| **No** | Email + password + TOTP 2FA for owner/admin (baseline) |
| **Google Workspace SSO** | Better Auth Google provider, restricted to the client's domain. Optionally disable password login. 2FA delegated to Google |
| **Microsoft Entra ID** | Better Auth Microsoft provider **[ASSUMPTION: available]** |
| **SAML / enterprise IdP** | Better Auth SSO plugin **[ASSUMPTION: verify maturity]**, or a managed IdP (WorkOS, Auth0) in front. Revisit [ADR-006](../adr/006-auth-better-auth-and-body-parsing.md) |

## Q15: Retention, consent and erasure

| Topic | Options | Default proposal |
|---|---|---|
| Participant data | Keep forever · N years after last workshop · delete on request | 3 years after last registration, then anonymize |
| Execution logs | 6 / 12 / 24 months | 24 months (attempt payload summaries 6 months) |
| Audit logs | 1 / 3 / 7 years | 7 years |
| Consent capture | In existing systems (likely) · in our intake · both | Record consent flags from intake. The existing system remains authoritative for opt-out |
| Erasure requests | Hard delete · anonymize (keep aggregates) | Anonymize ([G-24](#g-24-data-subject-operations-export-and-erasure-are-missing)) |

## Q16: Hosting region and timezones

| Answer | What we do |
|---|---|
| **India only, IST only** | Region ap-south-1 (or equivalent). DST tests still run (cheap insurance) |
| **Workshops in other timezones** | Already supported (workshop IANA timezone) |
| **International participants in IST workshops** | Add `participants.timezone` and format comms variables in the participant's zone ([G-22](#g-22-participant-timezone-personalization-missing)). Apply quiet hours per participant zone |
| **Data residency law** | Region + Q11 decision |

## Q17: Cloud provider and tooling

| Answer | Deployment mapping |
|---|---|
| **AWS** | ECS Fargate + ALB + RDS + Secrets Manager/KMS ([deployment §4 option A](../engineering/deployment.md#4-reference-hosting-proposed-confirm-q16-q17)) |
| **GCP** | Cloud Run (set min instances ≥ 1 for api and worker to avoid cold starts on Inngest calls; request timeout up to 60 min covers SSE) + Cloud SQL + Secret Manager/KMS. Cloud Run's request-based billing suits low traffic **[ASSUMPTION: verify SSE behavior on Cloud Run]** |
| **Azure** | Container Apps + Azure Database for PostgreSQL Flexible Server + Key Vault |
| **PaaS** (Render, Railway, Fly.io) | Option B. Check PITR window, private networking, and SSE buffering |
| **A single VPS** they already own | Docker Compose with `APP_ROLE=all`, managed Postgres strongly preferred over local Postgres. No HA. Acceptable only for the small profile |
| **Existing observability** (Datadog, New Relic, Grafana) | OTel exporters point there. Sentry optional |

## Q18: Go-live approval and pilot

| Answer | What we do |
|---|---|
| **Pilot workshop possible** | Cutover plan ([G-19](#g-19-no-cutover-and-data-migration-plan)): rehearsal-mode shadow run → assisted live run → full live |
| **No pilot, go straight to live** | Mandatory full rehearsal ([G-13](#g-13-rehearsal-test-mode-is-missing)) with time compression on staging, plus a named on-call engineer during the first live workshop |

## Q19: Should Zoom sessions end automatically?

| Answer | What we do |
|---|---|
| **Never** (default) | Hosts end sessions. Emergency stop has an optional "end live sessions" |
| **At scheduled end + grace** | New activity type `zoom.session.end` anchored at `session_end + PT15M`, `criticality=low`. Skipped if already ended. Uses `PUT /meetings/{id}/status {action:end}` **[VERIFIED for meetings]** |
| **Only for Simulive** | Not applicable: Simulive ends with the recording |

## Q20: Dashboard language

| Answer | What we do |
|---|---|
| **English only** | No i18n framework. Strings still centralized |
| **Additional languages** | `react-i18next` with namespaces per module. Dates via Luxon locale. Server error messages carry codes ([G-11](#g-11-no-central-error-code-catalog)) that the UI translates |

---

# Part B: Default bundle (if unanswered)

If a question is still open when its phase starts, engineering proceeds with these defaults. Each default is flagged in the code/config so it can be changed later.

| Q | Default | Reversible by |
|---|---|---|
| Q1 | Generic webhook adapter + n8n relay template | Config + adapter variant |
| Q2 | All triggers `unsafe` | Verification flow |
| Q3 | One sheet per workshop, default column map | Binding config |
| Q4 | Payment via sheet column. Refunds flagged for review | Config |
| Q5 | Zoom Meetings, registration on, single host user, Simulive via linked mode | Delivery mode per session |
| Q6 | 75% overall attendance of scheduled duration, ±5% review | `completion_rule` |
| Q7 | Email and phone unique | Migration (drop index) |
| Q8 | Internal OAuth app + dedicated ops user | Re-connect integration |
| Q9 | Trainers don't log in. No participant Calendar invites | Role invite / config |
| Q10 | Zoom share links. Materials via `/r/{token}` 30-day links | Config |
| Q11 | Inngest Cloud, IDs-only | ADR |
| Q12 | Medium profile | Infra sizing |
| Q13 | Slack + SES email | Notifier config |
| Q14 | Email + password + 2FA | Auth provider config |
| Q15 | Retention defaults above | Settings |
| Q16 | ap-south-1, IST | Infra |
| Q17 | AWS | Infra (containers are portable) |
| Q19 | Never auto-end | Activity type |
| Q20 | English | i18n |

---

# Part C: Gap analysis

A second review of the documentation found the problems below. **Severity:** 🔴 correctness bug in the current design · 🟠 missing capability needed for the MVP · 🟡 missing spec/doc detail · ⚪ nice to have.

| ID | Gap | Severity | Fix target |
|---|---|---|---|
| [G-01](#g-01-sweeper-re-emission-is-blocked-by-function-idempotency) | Sweeper re-emission is blocked by function idempotency | 🔴 | ADR-013, Phase 3 |
| [G-02](#g-02-waitforevent-race-in-fan-out-completion-and-event-mode-listeners) | `waitForEvent` race in fan-out completion and event-mode listeners | 🔴 | ADR-013, Phase 3 |
| [G-03](#g-03-retries-configuration-is-inconsistent) | Retries configuration is inconsistent (`retries: 0` vs per-step retries) | 🔴 | Phase 3 |
| [G-04](#g-04-execution-status-enum-is-incomplete) | Execution status enum is incomplete (`blocked`, `succeeded_with_errors`, reasons) | 🟠 | Phase 1 schema |
| [G-05](#g-05-lifecycle-transitions-have-no-trigger-mechanism) | Lifecycle transitions (`in_progress`, `sessions_all_ended`, `registration.closed`) have no trigger mechanism | 🟠 | Phase 2–3 |
| [G-06](#g-06-audience-and-recipient-types-are-underspecified) | Audience selector and recipient types are underspecified (trainers/staff can't be messaged) | 🟠 | Phase 3 |
| [G-07](#g-07-success-conditions-are-underspecified) | Success conditions are underspecified | 🟡 | Phase 3 |
| [G-08](#g-08-no-variable-catalog-or-formatting-rules) | No variable catalog or formatting rules for comms | 🟠 | Phase 4 |
| [G-09](#g-09-action-config-schemas-not-catalogued) | Action config schemas not catalogued | 🟡 | Phase 2 |
| [G-10](#g-10-no-complete-event-catalog) | No complete event catalog | 🟡 | Phase 1 |
| [G-11](#g-11-no-central-error-code-catalog) | No central error code catalog | 🟡 | Phase 1 |
| [G-12](#g-12-no-participant-message-safety-guard) | No participant message safety guard (frequency cap, quiet hours) | 🟠 | Phase 4 |
| [G-13](#g-13-rehearsal-test-mode-is-missing) | Rehearsal/test mode is missing | 🟠 | Phase 4–5 |
| [G-14](#g-14-no-participant-facing-link-endpoint) | No participant-facing link endpoint (resources, stable join links) | 🟠 | Phase 4 |
| [G-15](#g-15-joining-link-change-workflow-not-defined) | Joining-link change workflow not defined | 🟠 | Phase 4 |
| [G-16](#g-16-manual-tasks-lack-assignment-and-escalation) | Manual tasks lack assignment, due dates and escalation | 🟠 | Phase 3 |
| [G-17](#g-17-assessments-have-no-data-model) | Assessments have no data model | 🟡 (🟠 if Q6 needs it) | Phase 6 |
| [G-18](#g-18-outbox-relay-runs-only-in-the-api-role) | Outbox relay runs only in the api role | 🟡 | Phase 1 |
| [G-19](#g-19-no-cutover-and-data-migration-plan) | No cutover and data migration plan | 🟠 | Phase 6 |
| [G-20](#g-20-alert-routing-and-on-call-are-missing) | Alert routing and on-call are missing | 🟡 | Phase 5 |
| [G-21](#g-21-no-inngest-usage-and-cost-monitoring) | No Inngest usage/cost monitoring | ⚪ | Phase 5 |
| [G-22](#g-22-participant-timezone-personalization-missing) | Participant timezone personalization missing | ⚪ (🟠 if Q16 says international) | Phase 4 |
| [G-23](#g-23-no-developer-onboarding-and-conventions-guide) | No developer onboarding/conventions guide | 🟡 | Phase 1 |
| [G-24](#g-24-data-subject-operations-export-and-erasure-are-missing) | Data subject operations (export/erasure) are missing | 🟡 | Phase 6 |
| [G-25](#g-25-session-open-semantics-from-the-prd-are-undefined) | "Make the next session available" (PRD) is undefined | 🟡 | Phase 3 |
| [G-26](#g-26-zoom-registrant-lifecycle-not-fully-synced) | Zoom registrant lifecycle not fully synced | 🟠 | Phase 4 |
| [G-27](#g-27-concurrent-publish-and-edit-of-the-same-workshop) | Concurrent publish and edit of the same workshop | 🟡 | Phase 2 |

---

## G-01: Sweeper re-emission is blocked by function idempotency

**Problem.** [ADR-007](../adr/007-timeline-scheduling-strategy.md) sets `idempotency: 'event.data.runId'` on `timeline-activity-execute` and says the sweeper "re-emits `activity.armed` with the same runId (deduped by function idempotency)". Function idempotency allows **one run per key per 24 h** **[VERIFIED]**. If the original run is stuck or lost, the sweeper's re-emission is **silently dropped** for 24 h. The safety net never fires when it's needed most.

**Solution** (recorded in [ADR-013](../adr/013-recovery-events-and-check-wait-loops.md)):

1. Add a separate event `timeline/activity.recover { runId, activityId, generation, recoveryAttempt }` with event `id = runId + ':recover:' + recoveryAttempt`.
2. A separate function `timeline-activity-recover`: `idempotency: 'event.data.runId + ":" + string(event.data.recoveryAttempt)'`. No sleep. Goes straight to the **claim** step and then shares the same `ActivityExecutor` code path.
3. The claim's conditional UPDATE stays the single fence. If the original run wakes later, its claim fails (status no longer `scheduled`) and it no-ops.
4. Schema: `activity_runs.recovery_attempts int default 0`, `last_recovery_at timestamptz`. The sweeper increments `recovery_attempts` in the same statement that selects (`UPDATE … RETURNING`), so two sweeper invocations don't both emit.
5. Escalation: `recovery_attempts >= 2` and still unclaimed → critical alert `activity.unrecoverable`.

**Test:** drop the original `armed` run (Dev Server cancel) → the sweeper recovers within 2 min → exactly one execution.

## G-02: `waitForEvent` race in fan-out completion and event-mode listeners

**Problem.**
- (a) [timeline-engine §7](../features/timeline-engine.md#7-execution-algorithm) step 5c waits for `timeline/fanout.completed` with `step.waitForEvent`. If every child finishes **before** the parent reaches the wait (likely for small audiences), the event was emitted before the wait existed. Inngest wait semantics only match events received after the wait starts **[ASSUMPTION A9]**, so the parent waits until timeout and the activity reports wrong timing/outcome.
- (b) Event-mode listeners check the DB, then wait. An event arriving **between** the check and the wait registration is also missed.

**Solution: the "check-wait-recheck" loop** ([ADR-013](../adr/013-recovery-events-and-check-wait-loops.md)):

```text
for i in 0..maxIterations:
  state = step.run(`check-${i}`, () => readCompletionState())     // DB is the truth
  if state.done: break
  evt = step.waitForEvent(`wait-${i}`, { event, if: match, timeout: chunk })   // chunk = 5 min (fan-out) / 15 min (listeners)
  // whether evt arrived or timed out, loop back and re-check DB
  if now >= deadline: break → apply timeout policy
```

- Fan-out completion state = `activity_runs.outcome_summary` counters (total vs terminal), updated atomically by children.
- Event-mode state = the domain fact in Postgres (e.g. `sessions.actual_ended_at`).
- Step budget: a 2 h deadline with 5-min chunks = 24 iterations × 2 steps, far below 1,000.
- The event remains an **accelerator**. Correctness comes from the DB re-check.

## G-03: Retries configuration is inconsistent

**Problem.** The illustrative function in [nestjs-inngest-integration §3.3](../architecture/nestjs-inngest-integration.md#33-function-factory-pattern-illustrative) sets `retries: 0` ("retries handled per action step"). Function-level `retries` applies to **every step**, so a transient DB error in the `claim` or `finalize` step would fail the whole run with no retry. Meanwhile [reliability-patterns §5](../engineering/reliability-patterns.md#5-retry-policies-exponential-backoff-and-dead-letter-handling) says internal DB-only steps use Inngest default retries.

**Solution (rule):**

- Functions use `retries: 4` (default) or more for internal robustness.
- **Provider attempts never throw for classified failures.** The gateway's attempt step catches the error, records the attempt, and **returns** `{ outcome: 'transient', retryAfter }`. The explicit attempt loop (`action:${id}:attempt:${n}` + `step.sleep`) applies our retry policy. Inngest therefore never retries a provider call on its own, so attempts aren't double-counted.
- Only **unexpected** exceptions (bugs, DB unavailability) throw and use Inngest retries.
- `onFailure` marks the run `needs_attention`.

The illustrative snippet has been updated to match.

## G-04: Execution status enum is incomplete

**Problem.** Several documents use states missing from the enum in [database-design §2](../engineering/database-design.md#2-status-enums-shared): `blocked` (reliability §8), `succeeded_with_errors` (automation-engine §13), `blocked_capability` (integration-architecture §10). Reasons (superseded, stopped, no_consent, deadline_exceeded…) are only free-text.

**Solution:**

- Add `blocked` (waiting on a failed critical dependency or a missing capability, so admin action is needed) and `succeeded_with_errors` (partial success within the success condition) to `execution_status`.
- Add `status_reason text` with a check constraint on **every** execution table (`activity_runs`, `workflow_executions`, `workflow_step_executions`, `action_executions`). Values: `superseded, stopped, emergency_stopped, paused_by_admin, dependency_unsatisfied, dependency_critical_failed, capability_missing, deadline_exceeded, missed_window, no_consent, invalid_recipient, frequency_capped, quiet_hours, unknown_outcome, auth_error, manual_override, skipped_by_admin, rehearsal_redirected`.
- The UI shows status + reason. Filters support both.

The enum in database-design has been updated.

## G-05: Lifecycle transitions have no trigger mechanism

**Problem.** The docs reference `workshop.in_progress`, `workshop.sessions_all_ended`, `registration.closed` and session `ended (assumed)` fallbacks, but nothing **schedules** them.

**Solution: system activities.** At publish, the planner auto-generates non-deletable, non-editable timeline activities (`system=true`, hidden by default in the builder, visible in the Control Center):

| System activity | Anchor | Effect |
|---|---|---|
| `system.registration_open` | `workshops.registration_opens_at` | Intake accepts registrations (before that: `waitlisted` or reject per config) |
| `system.registration_close` | `workshops.registration_closes_at` | Emit `registration.closed`. Intake rejects/waitlists new rows |
| `system.workshop_start` | first session start | `published → in_progress` |
| `system.session_open` | session start − `session_open_offset` (default 15 min) | Session status `open` (see [G-25](#g-25-session-open-semantics-from-the-prd-are-undefined)) |
| `system.session_end_check` | session end + 15 min | If no `*.ended` webhook arrived: `ended (assumed)` + warning. Emit `session.ended` |
| `system.workshop_end_check` | last session end + 30 min | When all sessions are ended: emit `workshop.sessions_all_ended` |

They use the same arming/claim machinery, so pause, reschedule and recovery apply automatically. Schema additions: `workshops.registration_opens_at`, `registration_closes_at` (nullable), `timeline_activities.system bool default false`.

## G-06: Audience and recipient types are underspecified

**Problem.** `timeline_activities.audience` is referenced but never defined. Only registrations can receive comms, but the PRD needs trainer/staff notifications ("Admin notification", "Assign trainers", host reminders).

**Solution: audience schema** (`@wos/contracts/audience`):

```jsonc
{
  "recipientType": "registrations",      // registrations | trainers | staff | test_recipients
  "sessionScope": "ses_01J…",            // optional: only registrations/trainers of this session
  "filter": { "all": [ { "field": "registration.status", "op": "in", "value": ["confirmed"] } ] },  // condition AST
  "exclude": [
    { "actionSucceeded": { "triggerKey": "workshop_reminder", "scope": "this_activity" } },
    { "statusReason": "no_consent" }
  ],
  "collapseBy": null,                    // "phone" (see Q7) | null
  "limit": null
}
```

- `trainers` resolves `session_trainers → trainer_profiles` (phone/email). `staff` resolves users by role.
- The comms trigger payload's `recipient` gains `type: participant | trainer | staff`, and `participantId` becomes `recipientId`.
- `audience.preview` endpoint: `POST /api/v1/audiences/preview` → count + 20 sample rows (masked). It's used by the activity form.

## G-07: Success conditions are underspecified

**Solution: typed success conditions**, defaulted per action type:

| Type | Example | Used for |
|---|---|---|
| `all_succeeded` | — | Single-target actions (default) |
| `min_ratio` | `{ "minRatio": 0.95, "count": "accepted_or_better" }` | Fan-out comms (default 0.95) |
| `min_count` | `{ "min": 1 }` | "At least one trainer notified" |
| `event_within` | `{ "event": "zoom.session.started", "within": "PT10M" }` | "Session actually started" checks |
| `state_equals` | `{ "entity": "zoom_session", "field": "syncStatus", "value": "synced" }` | Zoom ensure |
| `manual_confirmation` | — | Manual tasks |

When the condition fails → `failed` (or `succeeded_with_errors` if `minRatio` is met but not 100%). Dependents use `on_success` / `on_completion` against this outcome.

## G-08: No variable catalog or formatting rules

**Problem.** Triggers declare `variables_schema`, but nothing defines which variables exist, where their values come from, or how dates are formatted.

**Solution: a variable catalog** (`@wos/contracts/variables`) with resolver functions. Values are resolved **at send time** inside the step, not at arm time, so reschedules and link changes are reflected.

| Variable | Source | Format |
|---|---|---|
| `name`, `first_name`, `full_name` | participant | Title case. `first_name` falls back to the first token of `full_name` |
| `workshop_name`, `workshop_code` | workshop | — |
| `workshop_dates` | days | "15–16 Nov 2026" (locale) |
| `session_title` | session | — |
| `session_time` | session start in **recipient tz** (G-22) or workshop tz | `h:mm a z, d MMM` → "9:00 AM IST, 15 Nov" |
| `session_date` | session | `EEEE, d MMMM` |
| `joining_link` | **stable link** `/j/{token}` (G-14) or raw Zoom join URL (per registrant if registration is on) | URL |
| `meeting_id`, `passcode` | zoom_sessions | Spaced digits |
| `resource_links` | published resources | Newline list of `/r/{token}` |
| `recording_link` | recording share link or `/r/{token}` | URL |
| `support_contact` | app_settings | — |
| `attendance_pct`, `completion_status` | registration | `82%` |
| `trainer_name` | session_trainers | — |

- Missing required variable → `permanent: missing_variable` **before** the call.
- A per-trigger `locale` (default `en-IN`) uses Luxon/Intl formatting.
- WhatsApp templates usually take positional parameters, so the trigger registry can define `parameterOrder: ["name", "session_time", "joining_link"]` for BSP adapters (Q1e).

## G-09: Action config schemas not catalogued

**Solution: the action catalog** (each row is a Zod schema in `@wos/contracts/actions`):

| Action key | Config fields |
|---|---|
| `comms.trigger` | `integrationId`, `triggerKey`, `audience` (G-06), `variablesOverride?`, `bulk?: boolean`, `deadline?: anchor+offset`, `requiresConsent` (inherited), `frequencyCapExempt?` (G-12) |
| `zoom.session.ensure` | `sessionId` (implicit), `type: meeting|webinar`, `settingsTemplate` (waiting room, join before host, auto-recording, registration, approval type), `hostUserId?` (or pool) |
| `zoom.simulive.verify` | `sessionId`, `toleranceMinutes` (default 1) |
| `zoom.session.prepare_host` | `sessionId`, `notifyRecipients` (trainers/staff), `leadTime` |
| `zoom.attendance.import` | `sessionId`, `settleDelay` (PT30M), `maxWait` (P1D), `source: report|past_participants` |
| `zoom.session.end` (Q19) | `sessionId`, `onlyIfLive: true` |
| `recording.publish` | `sessionId`, `mode: zoom_share|hosted`, `trigger: {integrationId, triggerKey}`, `delay?` |
| `resource.publish` | `resourceIds[]`, `trigger`, `linkValidity` |
| `calendar.event.ensure` | `entity: session|workshop`, `calendarId`, `attendees: trainers|staff|none`, `includeJoinDetails` |
| `sheets.sync` | `bindingId`, `direction: reconcile|export` |
| `sheets.export_report` | `report: registrations|attendance|completion`, `target: new_tab|new_spreadsheet` |
| `docs.generate` (P2) | `templateId`, `scope: workshop|registration`, `outputFormat` |
| `participants.set_status` | `audience`, `status`, `reason` |
| `registrations.set_completion` | `rule?` (defaults to workshop rule) |
| `ops.notify` | `severity`, `message template`, `routes?` (G-20) |
| `control.manual_task` | see G-16 |
| `control.branch` | `condition` (AST), `trueActivities[]`, `falseActivities[]` |
| `automation.run_workflow` | `workflowId`, `input` |

## G-10: No complete event catalog

**Solution:** a single catalog (`@wos/contracts/events`) generated into a doc table at build time. The minimum set:

| Event | Payload (beyond envelope) | Producer | Main consumers |
|---|---|---|---|
| `workshop.published` / `activated` / `paused` / `resumed` / `stopped` / `emergency_stopped` / `cancelled` / `rescheduled` / `in_progress` / `sessions_all_ended` / `completed` / `archived` | `workshopId`, `planRevision` | workshops | timeline, automation router, SSE |
| `session.rescheduled` / `cancelled` / `open` / `live` / `ended` | `sessionId`, `old/new startsAt` | workshops, zoom webhooks, system activities | reschedule workflow, attendance |
| `registration.created` / `updated` / `confirmed` / `cancelled` / `payment_status_changed` / `completion_evaluated` | `registrationId`, `changedFields[]` | participants | registration workflow, sheets export, zoom registrants |
| `registration.closed` | `workshopId` | system activity (G-05) | unpaid review workflow |
| `attendance.imported` | `sessionId`, `stats` | attendance import | completion, sheets export |
| `timeline/activity.armed` / `disarmed` / `recover` / `completed` / `fanout.completed` | `activityId`, `runId`, `generation`, `outcome` | timeline | timeline functions, dependency resolver |
| `automation/execution.requested` / `cancel_requested` / `approval.decided` | `executionId` | router, API | interpreter |
| `zoom.session.started` / `ended` / `updated_externally` / `deleted_externally` / `join_url_changed` | `sessionId`, `zoomId` | webhooks process, ensure | sessions, G-15 |
| `zoom.participant.joined` / `left` | `sessionId`, `attendanceRecordId` | webhooks process | live counters |
| `zoom.recording.completed` | `sessionId`, `recordingId` | webhooks process | recording workflow |
| `comms.delivery.reported` | `actionExecutionId`, `status` | comms callbacks | execution records |
| `sheets.rows.changed` | `bindingId`, `stats` | reconcile | — |
| `integration.status_changed` | `integrationId`, `from`, `to` | gateway/health | alerts, SSE |
| `webhook.received` | `webhookEventId` | inbox | webhooks process |

## G-11: No central error code catalog

**Solution:** `@wos/contracts/errors`, one table shared by API problem responses, action executions and the UI:

| Code | Class | HTTP (API) | Admin-facing message |
|---|---|---|---|
| `version_conflict` | — | 409 | "Someone else changed this. Reload to see their changes." |
| `invalid_state_transition` | — | 409 | "This action isn't allowed while the workshop is {state}." |
| `capability_missing` | permanent | 422 | "Your Zoom plan doesn't support {capability}." |
| `missing_variable` | permanent | — | "Trigger needs '{var}', which has no value for this participant." |
| `unknown_trigger` | permanent | — | "The WhatsApp/email system doesn't recognise '{trigger}'." |
| `template_paused` | permanent | — | "The WhatsApp template is paused by Meta." |
| `invalid_recipient` | permanent | — | "Phone/email is missing or invalid." |
| `rate_limited` | transient | 429 | "Provider is rate limiting. Retrying automatically." |
| `rate_limited_daily` | rate_limited_daily | — | "Zoom daily update limit reached for host {host}." |
| `auth_error` | auth | — | "Connection to {provider} needs to be re-authorised." |
| `unknown_outcome` | unknown_outcome | — | "We couldn't confirm whether {provider} received this. Check and resolve." |
| `deadline_exceeded` | permanent | — | "Stopped retrying because the message is no longer useful after {deadline}." |
| `frequency_capped` | skipped | — | "Skipped: participant already received {n} messages today." |
| `response_schema_mismatch` | permanent | — | "Provider returned an unexpected response (engineering notified)." |

## G-12: No participant message safety guard

**Problem.** Nothing stops a misconfigured timeline (duplicated reminder activities, an overlapping workflow and timeline) from messaging participants many times. Each WhatsApp template message is billed **[VERIFIED]** and hurts template quality ratings.

**Solution: `CommsSafetyGuard`**, evaluated in the per-recipient eligibility step:

| Rule | Default | Configurable |
|---|---|---|
| Frequency cap per recipient per channel | 5 / rolling 24 h | Workshop settings |
| Duplicate content guard | Same `trigger_key` to the same recipient within 30 min from **different** activity runs → `skipped: frequency_capped` + warning | Yes |
| Quiet hours | No WhatsApp 21:00–08:00 recipient tz, unless the activity is `urgent=true` (e.g. schedule change) | Yes |
| Global circuit | > 3× the expected sends for a workshop within 1 h → auto-pause that workshop's comms + critical alert | Threshold |
| Preview warning | Timeline/workflow validation estimates messages per participant per day and warns over the cap | — |

Implementation: an indexed query on `action_executions (target_type, target_id, created_at)` filtered by channel. That's cheap at our scale.

## G-13: Rehearsal (test) mode is missing

**Problem.** There's no safe way to run a real workshop plan end to end (UAT, trainer rehearsal, a dry run before the first live workshop) without touching real participants and real calendars.

**Solution:** `workshops.mode = 'live' | 'rehearsal'`.

| Integration | Rehearsal behavior |
|---|---|
| Comms | Recipients replaced by `app_settings.test_recipients` (allow-list). Variables still resolved from real/test registrations. `status_reason=rehearsal_redirected` |
| Zoom | Uses a **test host user** (or skips creation and records "simulated") |
| Calendar | Test calendar ID |
| Sheets | Test binding (copy of the sheet) or export disabled |
| Time | Optional **time compression** (e.g. 1 workshop day = 60 min). The planner scales offsets and durations from a chosen start. Only allowed in rehearsal |

Rehearsal workshops can't be switched to live. Duplicate them as live instead, which prevents leftovers. This also covers Q2's "no test triggers" branch and Q18's pilot.

## G-14: No participant-facing link endpoint

**Problem.** Materials need participant-accessible links, but the API is all authenticated, and presigned R2 URLs expire within minutes or hours. Join links embedded in messages go stale if a Zoom meeting is recreated.

**Solution:** a small **public link service** on the api role:

| Route | Token payload (HMAC-signed, compact) | Behavior |
|---|---|---|
| `GET /r/{token}` | `resourceId` or `recordingId`, `registrationId?`, `exp` | Checks the resource is still published and the token isn't revoked → 302 to a presigned R2 GET (5 min) or the Zoom share URL. Logs access (`resource_access_logs`) |
| `GET /j/{token}` | `sessionId`, `registrationId?`, `exp` | Resolves the **current** join URL at click time (per-registrant if registration is on) → 302. **A recreated or rescheduled Zoom meeting doesn't invalidate links already sent** |
| `GET /h/{token}` (Q9 "trainers don't log in") | `sessionId`, `trainerId`, `exp` (10 min), single use (`jti` table) | Fetches a fresh `start_url` → 302 |
| `GET /ics/{token}` (Q9 alternative) | `sessionId` | Returns an `.ics` file |

Security: tokens are HMAC-SHA256 with a rotating key (kid in token). Rate limiting per IP. No PII in tokens (IDs only). `Referrer-Policy: no-referrer`. The domain should be the client's (e.g. `go.client.com`) for trust. Add these to [security.md](../engineering/security.md) and [api-specification.md](../engineering/api-specification.md) when implemented.

## G-15: Joining-link change workflow not defined

**Problem.** [zoom-integration §11](../features/zoom-integration.md#11-error-scenarios-prd-case-2) mentions a `joining_link_changed` workflow, but it isn't defined.

**Solution:** a built-in workflow **"Joining link changed"**:

- Trigger: `zoom.session.join_url_changed` (emitted by `zoom.session.ensure` when the stored `join_url` differs after create/recreate/adopt).
- If participants received `/j/{token}` links (G-14): **no participant notification needed**. Notify staff only.
- If raw Zoom links were sent: for registrations that already got `joining_instructions` for this session and while the session hasn't ended → trigger `joining_instructions` again (variables marked "updated link"). Exempt from the duplicate-content guard (G-12) but still counted in the frequency cap.

## G-16: Manual tasks lack assignment and escalation

**Solution:** extend manual tasks (`control.manual_task` config + a `manual_tasks` table linked to `activity_runs`):

| Field | Purpose |
|---|---|
| `assignee_type` / `assignee_id` | User, role (e.g. any operator), or the session's trainer |
| `instructions`, `checklist[]` | What to do. Each checklist item is ticked individually |
| `due_at` | Default = activity scheduled time + `timeout` |
| `escalation` | `[{ after: 'PT10M', notify: 'role:admin' }, { after: 'PT30M', alert: 'high' }]` |
| `on_timeout` | `fail` · `skip` · `approve` (auto-proceed) |
| `decision`, `decided_by`, `note`, `evidence_url?` | Outcome (approve/reject). Audited |

UI: a "My tasks" inbox (Overview + header badge) and notifications to the assignee via the ops channel. Trainers without login can confirm via a signed link (same mechanism as G-14).

## G-17: Assessments have no data model

**Problem.** The PRD's timeline includes "Assessment: publish assessment and update participant records", and completion rules may need scores. No model exists.

**Solution:**

- Tables: `assessments (id, workshop_id, session_id?, title, source: google_form|lms|zoom_poll|manual, external_ref, pass_mark, max_score, published_at)` and `assessment_results (assessment_id, registration_id, score, passed, submitted_at, source_ref)` with a unique (`assessment_id`, `registration_id`).
- "Publish assessment" activity = `resource.publish`-like action that sends the form link (`/r/{token}` or the form URL) via a comms trigger.
- Results import: a sheet binding of kind `assessment_results` (Google Forms quiz responses → matched by email/phone → `assessment_results`).
- Condition/completion fields: `assessment.passed(assessmentId)` and `assessment.score`.

## G-18: Outbox relay runs only in the api role

**Problem.** [system-architecture §2](../architecture/system-architecture.md#2-high-level-architecture) places the outbox relay in the api role. Worker steps also write outbox rows (e.g. `activity.completed`). If api instances are down or restarting, worker-produced events stall. The dependency resolver and SSE then lag.

**Solution:** run the relay poller in **both** roles (`FOR UPDATE SKIP LOCKED` already makes this safe). Add an after-commit "kick" in both roles. Alert if `outbox_pending` > 0 for longer than 60 s.

## G-19: No cutover and data migration plan

**Solution: cutover plan**:

1. **T−4 weeks: data import.** Import the current participant master data (optional) and active workshop sheets as bindings in `import`-only direction. Resolve the review queue.
2. **T−3 weeks: shadow rehearsal.** The next real workshop is built in Workshop OS as a **rehearsal** (G-13) while operations continue manually. Compare the platform's planned sends with what staff actually send.
3. **T−1 week: assisted live.** First live workshop with platform comms for low-risk activities only (e.g. materials, post-session). Reminders stay manual. On-call engineer present.
4. **Go-live.** Full automation. Manual runbook printed. "Pause automation" is the fallback, and staff can revert to manual sending.
5. **Rollback criteria:** > 2 critical incidents in a workshop, or any duplicate-message incident → pause, post-mortem, fix before the next workshop.
6. **Training:** 2 sessions for admins/operators (Control Center, Execution Monitor, review queue).

## G-20: Alert routing and on-call are missing

**Solution:**

- `alert_routes (id, match: {type?, severity?, workshopId?}, channels: [...], users: [...])`, evaluated by `AlertService`. Default: critical → ops channel + SMS (if Q13 says so). High → ops channel. Medium/low → in-app only.
- `workshop_staff (workshop_id, user_id, role: lead|on_call|support)`. During a workshop's live window, critical alerts also page the `on_call` user. If not acknowledged in 10 min, escalate to `lead`, then to owner.
- Quiet routing outside live windows for non-critical alerts (batched digest every 2 h).

## G-21: No Inngest usage and cost monitoring

**Solution:** daily job estimates executions from our own records (runs, steps, fan-out children) per workshop. A dashboard tile shows monthly projection vs plan included amount (Pro: 1 M executions **[VERIFIED]**). Alert at 80%. Validation preview shows "estimated executions for this workshop". Reconcile with the Inngest billing dashboard monthly.

## G-22: Participant timezone personalization missing

**Solution:** `participants.timezone` (IANA, nullable). It's inferred from the phone country code where unambiguous (e.g. +91 → Asia/Kolkata), editable, and imported from a sheet column if present. Used for variable formatting (G-08) and quiet hours (G-12). **Scheduling stays in workshop time.** A reminder "24 h before" is the same instant for everyone. Only its formatting changes.

## G-23: No developer onboarding and conventions guide

**Solution:** create `CONTRIBUTING.md` (and an `AGENTS.md` for coding agents) at the repo root when code starts. Outline:

- Local setup: pnpm, Docker Compose (Postgres + Inngest Dev Server), seeding, `APP_ROLE=all`.
- Module boundaries and allowed imports (lint rules).
- **Inngest function rules:** stable function/step IDs, no side effects outside `step.run`, small outputs, fan-out for per-participant work, new function ID for breaking changes ([nestjs-inngest-integration §5](../architecture/nestjs-inngest-integration.md#5-execution-model-replay-semantics)).
- DB: expand/contract migrations, never `drizzle-kit push` in shared environments, explicit `tx` parameters.
- Contracts: every API/event/action schema lives in `@wos/contracts`.
- PR checklist: idempotency key defined? audit written? execution record created? permission declared? docs updated?

## G-24: Data subject operations (export and erasure) are missing

**Solution:**

- `POST /api/v1/participants/{id}/export` (admin) → JSON bundle (participant, registrations, attendance, comms history summaries) to R2, presigned link, audited.
- `POST /api/v1/participants/{id}/anonymize` (owner) → replace name/email/phone with irreversible tokens, null attributes, cancel future sends, keep aggregates and execution rows (target IDs remain, PII gone). Push removal of rows from bound sheets (export direction) and log it. Irreversible, double confirmation, audited.
- Retention job (Q15) applies the same anonymization automatically.

## G-25: Session open semantics from the PRD are undefined

**Problem.** The PRD's timeline says "10:30 AM: Session 2 begins. Make the next session available". "Available" isn't defined.

**Solution:** define a session status **`open`** between `scheduled` and `live`, set by `system.session_open` (G-05). It means:

- The join link becomes active (with G-14 `/j/{token}`, links clicked before `open` show a "session opens at 10:15 AM IST" page instead of redirecting, which is optional per workshop).
- The Control Center marks it "current/next".
- Optional comms trigger (`session_open` / "Session 2 is starting, join now").
- For Zoom, the meeting already exists. "Available" is about our links and messaging, not Zoom. Zoom's own join-before-host / waiting-room settings still apply.

## G-26: Zoom registrant lifecycle not fully synced

**Problem.** Registrants are added on confirmation, but cancellations, email changes and workshop duplication aren't handled. A cancelled participant keeps a working personal join link.

**Solution:** `zoom.registrants.sync` action, triggered by `registration.cancelled`, `registration.updated` (email/name), and session recreation:

- Cancel → update the registrant's status to cancelled/denied via the registrant status endpoint **[ASSUMPTION: `PUT /meetings/{id}/registrants/status` with `action: cancel|deny`; verify in SPIKE S3]**.
- Email change → cancel the old registrant and add a new one. The join URL changes, so `/j/{token}` (G-14) absorbs it.
- Session recreated → re-add all confirmed registrants (bulk, throttled) → new join URLs.
- Daily reconciliation compares Zoom registrants with confirmed registrations, so drift is fixed automatically (registrants are platform-owned).

## G-27: Concurrent publish and edit of the same workshop

**Problem.** An admin edits activities while the publish workflow is provisioning. The publish snapshot and the live plan may diverge.

**Solution:** while `status = publishing`, plan edits return `409 publishing_in_progress` (a short window). Publish records the `plan_revision` it provisioned. If `plan_revision` changed by the end of provisioning (e.g. edits through an override path), the publish workflow runs one more ensure pass before declaring `published`.

---

## Documents updated as a result of this review

- [ADR-013](../adr/013-recovery-events-and-check-wait-loops.md) added (G-01, G-02, G-03). ADR-007 marked as amended.
- [database-design.md §2](../engineering/database-design.md#2-status-enums-shared): `blocked`, `succeeded_with_errors`, `status_reason` (G-04).
- [nestjs-inngest-integration.md §3.3](../architecture/nestjs-inngest-integration.md#33-function-factory-pattern-illustrative) and [timeline-engine.md §7](../features/timeline-engine.md#7-execution-algorithm): retries rule and check-wait-recheck loop (G-02, G-03).
- The remaining gaps (G-05 … G-27) are specified here. Fold them into the relevant feature docs when their phase starts ([roadmap.md](roadmap.md)).

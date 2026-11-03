# Open Questions for the Client (Prioritized)

Each question lists **why it matters** and **what it blocks**. P0 must be answered before Phase 0 spikes finish. P1 before Phase 4 (integrations). P2 before go-live.

Related: [risks-and-tradeoffs.md](risks-and-tradeoffs.md) · [roadmap.md](roadmap.md) · **[open-questions-playbook.md](open-questions-playbook.md): what we build for each possible answer, and the defaults if a question is still unanswered**

---

## P0: Blocking architecture/integration design

| ID | Question | Why it matters | Blocks |
|---|---|---|---|
| **Q1** | **Where do the existing WhatsApp and email automations run?** (n8n, Make, GoHighLevel, a WhatsApp BSP tool such as WATI/Interakt/AiSensy, a custom backend, other.) How are they triggered today (webhook URL, API, manual)? Do they authenticate requests? What request rate can they handle? Do they report delivery status back? Do they need us to call from a static IP? | Determines the comms adapter, auth, throttle, callback design and idempotency (PRD §13 Q1) | Comms adapter, [whatsapp-email-integration.md](../features/whatsapp-email-integration.md) |
| **Q2** | Can those systems **deduplicate by an idempotency key** we send? Can they provide **test triggers** that only message internal recipients? Which WhatsApp template categories (utility/marketing) do reminders use? | Without dedupe, network timeouts require manual resolution ([ADR-012](../adr/012-unknown-outcome-no-auto-retry.md)). Test triggers are needed for staging. Template category affects messaging cost | Retry policy, testing |
| **Q3** | **How are participants managed today?** Google Sheets (one per workshop? shared?), Google Forms, website form, payment page, CRM? Which columns are edited by humans after registration? (PRD §13 Q3) | Defines intake channels and the Sheets column-ownership agreement ([ADR-011](../adr/011-google-sheets-sync-ownership.md)) | Participants, Sheets sync |
| **Q5** | **How are recorded workshops delivered?** Zoom Simulive (which licenses: Webinars, Events/Sessions?), recordings in Google Drive, other? Zoom plan tier (Pro/Business/Enterprise), Webinar add-on, Cloud Recording, how many Zoom host users, and is Zoom registration acceptable for participants? (PRD §13 Q2) | Simulive automation is limited by Zoom's API. Reports require paid plans. Daily per-host update limits. Registration enables reliable attendance | Zoom adapter, delivery modes, SPIKE S3 |
| **Q8** | Does the client use **Google Workspace** (business domain) or personal Google accounts? Who owns the calendars and sheets? Can we create a dedicated ops Google account? Will a Workspace admin approve an Internal OAuth app or domain-wide delegation? | Auth model, OAuth verification needs, token stability | Google adapters, SPIKE S5 |
| **Q11** | Is it acceptable for **Inngest Cloud** (US-based SaaS) to process workflow metadata (IDs, statuses, timings, no PII)? Any data-residency requirement? | Cloud vs self-hosted Inngest (self-host adds Redis + ops) | [ADR-002](../adr/002-inngest-orchestration.md) |
| **Q12** | Expected scale: workshops per month, concurrent workshops, max participants per workshop, reminders per participant. Budget for the Inngest plan (Pro or higher needed for production)? | Plan sizing, cost, fan-out strategy | Infra, cost |

## P1: Needed before integrations / feature completion

| ID | Question | Why | Blocks |
|---|---|---|---|
| **Q4** | Where does **payment status** come from (payment gateway webhooks, sheet column, manual)? Which statuses exist? Can payment change after confirmation (refunds)? | Registration gating, workflow conditions | Registration workflow |
| **Q6** | **Completion rules**: attendance threshold (75%?), denominator (scheduled vs actual duration), required sessions, assessments (where are results stored?), manual exemptions | Completion computation, reports, certificates | Completion workflow |
| **Q7** | Can two different participants share an email or phone (family/corporate)? | Dedupe uniqueness constraints | Participant model |
| **Q9** | Should **trainers** log in (see their sessions, confirm manual tasks, start as host)? Should participants receive **Calendar invites** (default: no, comms only)? | Roles, Calendar attendees, privacy | RBAC, Calendar |
| **Q10** | Should recordings and materials be shared as **Zoom share links** or **our hosted links**? How long should participant links stay valid? | Storage cost, link expiry design | Recordings, resources |
| **Q13** | Where should **operational alerts** go (Slack, Teams, email to ops, SMS)? Who is on call during live workshops? | Alert channel must be independent of the comms systems | Alerts |
| **Q14** | Is **SSO** (Google Workspace / Microsoft) required for staff, or is email + password + 2FA acceptable? | Auth choice | [ADR-006](../adr/006-auth-better-auth-and-body-parsing.md) |

## P2: Before go-live

| ID | Question | Why |
|---|---|---|
| **Q15** | Data retention and consent policy: how long to keep participant data, execution logs and audit logs? Consent capture for WhatsApp/email (done in existing systems?). Erasure requests (DPDP Act)? | Retention jobs, consent flags, compliance |
| **Q16** | Hosting region preference (e.g. India). Will workshops ever run in non-IST timezones or have international participants? | Region, DST testing scope, reminder localization |
| **Q17** | Preferred cloud/vendor accounts (AWS/GCP/Azure/PaaS), existing observability tools, budget | Deployment option A vs B |
| **Q18** | Who approves the go-live checklist? Is there a pilot workshop we can run in parallel with current manual operations? | UAT plan |
| **Q19** | Should "Workshop begins" also be able to **end** Zoom sessions automatically at scheduled end time, or always leave that to hosts? | Emergency/auto-end behavior |
| **Q20** | Branding/language requirements for the dashboard (English only?) | UI |

## Questions answered by our research (no client input needed)

| Topic | Finding |
|---|---|
| Can Inngest's Next.js integration be used in NestJS? | No. Use `inngest/express` per Inngest docs ([ADR-005](../adr/005-nestjs-inngest-express-serve.md)) |
| Can Zoom REST start a meeting server-side? | No. A host starts it via `start_url`/SDK. The API can end meetings |
| Can Zoom REST create/configure Simulive? | Not documented. Community reports say no (as of mid-2026) |
| Does Google Sheets have row-change webhooks? | No general-purpose one. Polling reconciliation is required |
| Zoom webhook signature scheme | `v0` HMAC-SHA256 over `v0:{timestamp}:{body}`, 3 s response, retries only on 5xx |
| Zoom create/update daily limit | 100/day per user |
| Sheets quotas | 60/min/user/project, 300/min/project (read and write separately) |

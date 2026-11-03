# Workshop OS — Technical Documentation

Workshop OS is a **single-business workshop orchestration platform**. It is the control center that sits between the client's existing tools (existing WhatsApp automation, existing email automation, Zoom, Google Sheets, Google Calendar, Google Docs) and coordinates them against a configurable, durable workshop timeline.

> The platform decides **when** something happens, **which system** performs it, **what happens next**, and **what happens if it fails**. It does not re-implement messaging, email marketing, CRM or video delivery.

Source input: [`/Plan/prd.md`](../Plan/prd.md).

---

## 1. Scope at a glance

| In scope | Out of scope |
|---|---|
| Workshop, day, session, activity management | Multi-tenant SaaS infrastructure |
| Multi-day timeline builder and durable timeline execution | WhatsApp bulk-messaging software |
| Event-driven automation workflows (visual builder) | Email marketing / template / campaign software |
| Triggering the client's **existing** WhatsApp and email automations | A CRM or marketing-automation engine |
| Zoom meeting / webinar management, attendance, recordings | Video hosting / streaming |
| Google Sheets import, export, reconciliation | Participant self-service portal (future) |
| Google Calendar synchronization | Payments processing (status is consumed, not processed) |
| Google Docs generation (post-MVP) | Certificates designer (future) |
| Execution monitoring, retries, pause/resume, overrides, audit | |

## 2. Documentation conventions

Every claim about an external system carries one of these labels:

| Label | Meaning |
|---|---|
| **[VERIFIED]** | Confirmed against official provider documentation (link given) at the time of writing (Nov 2026). Re-verify before implementation if more than ~3 months have passed. |
| **[COMMUNITY]** | Reported by developer forums / community sources, not by official docs. Treat as a strong hint, not a guarantee. |
| **[PROPOSED]** | Our design decision. Can be changed through an ADR. |
| **[ASSUMPTION]** | Something we believe but have not confirmed. Must be validated. |
| **[LICENSE]** | Depends on the client's paid plan / license with the provider. |
| **[CONFIRM]** | Requires an answer from the client before implementation (tracked in [open-questions.md](planning/open-questions.md)). |
| **[SPIKE]** | Requires a time-boxed technical proof before committing (tracked in [roadmap.md](planning/roadmap.md#phase-0-technical-spikes)). |

Code snippets in these docs are **illustrative only**; they show shape and intent, not final implementation.

## 3. Documentation tree

```
docs/
├── README.md                                   ← you are here
├── architecture/
│   ├── system-architecture.md                  High-level + low-level architecture, modules, process roles
│   ├── data-flow.md                            Event flow, outbox, webhook inbox, source-of-truth rules
│   ├── diagrams.md                             All Mermaid diagrams in one place (architecture, ER, sequences)
│   ├── nestjs-inngest-integration.md           ★ Deep investigation: running Inngest inside NestJS
│   └── integration-architecture.md             Adapter pattern, provider contracts, capability registry
├── features/
│   ├── workshop-management.md                  Workshop lifecycle, days, sessions, publish/activate
│   ├── timeline-engine.md                      Scheduling, anchors, dependencies, execution, rescheduling
│   ├── automation-engine.md                    Triggers, conditions, actions, branching, versioning
│   ├── participant-management.md               Participants, registrations, attendance, completion
│   ├── whatsapp-email-integration.md           Contracts for the client's existing comms automations
│   ├── zoom-integration.md                     Meetings, webinars, Simulive limits, attendance, recordings
│   ├── google-workspace.md                     Sheets sync, Calendar sync, Docs generation
│   ├── admin-dashboard.md                      Screens, navigation, control center, operational controls
│   └── execution-monitoring.md                 Execution records, retries, DLQ, alerts, recovery
├── engineering/
│   ├── technical-stack.md                      Stack evaluation, compatibility issues, alternatives
│   ├── reliability-patterns.md                 The 20 critical engineering considerations, in depth
│   ├── database-design.md                      Full schema, relationships, indexes, constraints
│   ├── api-specification.md                    REST endpoints, contracts, webhooks, SSE
│   ├── security.md                             AuthN, RBAC, secrets, webhook auth, data protection
│   ├── testing.md                              Unit, integration, contract, load, E2E, chaos
│   └── deployment.md                           Environments, Docker, CI/CD, backups, scaling, cost
├── adr/
│   ├── README.md                               ADR index and template
│   ├── 001-modular-monolith.md
│   ├── 002-inngest-orchestration.md
│   ├── 003-postgresql-source-of-truth.md
│   ├── 004-integration-adapter-pattern.md
│   ├── 005-nestjs-inngest-express-serve.md
│   ├── 006-auth-better-auth-and-body-parsing.md
│   ├── 007-timeline-scheduling-strategy.md
│   ├── 008-workflow-versioning.md
│   ├── 009-react-vite-spa-instead-of-nextjs.md
│   ├── 010-realtime-sse-with-postgres-notify.md
│   ├── 011-google-sheets-sync-ownership.md
│   ├── 012-unknown-outcome-no-auto-retry.md
│   └── 013-recovery-events-and-check-wait-loops.md
└── planning/
    ├── product-requirements.md                 Consolidated PRD and functional requirements
    ├── roadmap.md                              Phases, feature dependency graph, implementation order, MVP vs future
    ├── risks-and-tradeoffs.md                  Technical risks, assumptions, trade-offs
    ├── open-questions.md                       Prioritized questions for the client
    └── open-questions-playbook.md              What we build for each possible answer + gap analysis (G-01…G-27)
```

### Mapping to the originally requested numbered documents

| Requested | Where it lives |
|---|---|
| 01 Product requirements | [planning/product-requirements.md](planning/product-requirements.md) |
| 02 System architecture | [architecture/system-architecture.md](architecture/system-architecture.md), [architecture/data-flow.md](architecture/data-flow.md) |
| 03 Technical stack | [engineering/technical-stack.md](engineering/technical-stack.md) |
| 04 Database design | [engineering/database-design.md](engineering/database-design.md) |
| 05 API specification | [engineering/api-specification.md](engineering/api-specification.md) |
| 06 Workshop management | [features/workshop-management.md](features/workshop-management.md) |
| 07 Timeline engine | [features/timeline-engine.md](features/timeline-engine.md) |
| 08 Automation engine | [features/automation-engine.md](features/automation-engine.md) |
| 09 Integration architecture | [architecture/integration-architecture.md](architecture/integration-architecture.md) |
| 10 WhatsApp/email | [features/whatsapp-email-integration.md](features/whatsapp-email-integration.md) |
| 11 Zoom | [features/zoom-integration.md](features/zoom-integration.md) |
| 12 Google Workspace | [features/google-workspace.md](features/google-workspace.md) |
| 13 Participants | [features/participant-management.md](features/participant-management.md) |
| 14 Admin dashboard | [features/admin-dashboard.md](features/admin-dashboard.md) |
| 15 Execution monitoring | [features/execution-monitoring.md](features/execution-monitoring.md) |
| 16 Security | [engineering/security.md](engineering/security.md) |
| 17 Testing | [engineering/testing.md](engineering/testing.md) |
| 18 Deployment | [engineering/deployment.md](engineering/deployment.md) |
| 19 Roadmap | [planning/roadmap.md](planning/roadmap.md) |
| 20 Risks / open questions | [planning/risks-and-tradeoffs.md](planning/risks-and-tradeoffs.md), [planning/open-questions.md](planning/open-questions.md) |

## 4. Recommended reading order

1. [planning/product-requirements.md](planning/product-requirements.md): what we are building and why.
2. [architecture/system-architecture.md](architecture/system-architecture.md) → [architecture/data-flow.md](architecture/data-flow.md): how the parts fit.
3. [architecture/nestjs-inngest-integration.md](architecture/nestjs-inngest-integration.md): the most important pre-implementation decision.
4. [engineering/reliability-patterns.md](engineering/reliability-patterns.md): rules every module must follow.
5. [features/timeline-engine.md](features/timeline-engine.md) → [features/automation-engine.md](features/automation-engine.md): the core engines.
6. Integration docs, then dashboard and monitoring.
7. [planning/roadmap.md](planning/roadmap.md) and [planning/open-questions.md](planning/open-questions.md) before kickoff.

## 5. Key architectural decisions (summary)

| # | Decision | ADR |
|---|---|---|
| 1 | Modular monolith (NestJS) with two process roles (`api`, `worker`) from one codebase/image | [001](adr/001-modular-monolith.md) |
| 2 | Inngest for durable execution; PostgreSQL owns domain + execution records; Inngest owns run mechanics only | [002](adr/002-inngest-orchestration.md) |
| 3 | PostgreSQL is the single source of truth; external systems are mirrors or executors | [003](adr/003-postgresql-source-of-truth.md) |
| 4 | Every provider behind an independent adapter with a capability registry | [004](adr/004-integration-adapter-pattern.md) |
| 5 | Inngest served via the official `inngest/express` adapter mounted in NestJS `main.ts`; no Next.js integration, no unofficial Nest wrappers | [005](adr/005-nestjs-inngest-express-serve.md) |
| 6 | Better Auth mounted before body parsers; per-route body parsing (auth / inngest / webhooks / API) | [006](adr/006-auth-better-auth-and-body-parsing.md) |
| 7 | Per-activity durable runs (`sleepUntil` + `cancelOn`) fenced by a schedule generation, plus a Postgres sweeper | [007](adr/007-timeline-scheduling-strategy.md) |
| 8 | Immutable published workflow versions; executions pin a version | [008](adr/008-workflow-versioning.md) |
| 9 | React + Vite SPA (PRD's Next.js frontend replaced) | [009](adr/009-react-vite-spa-instead-of-nextjs.md) |
| 10 | SSE for realtime, fanned out across instances with Postgres `LISTEN/NOTIFY` (no Redis) | [010](adr/010-realtime-sse-with-postgres-notify.md) |
| 11 | Column-ownership model for Google Sheets sync; polling reconciliation is mandatory | [011](adr/011-google-sheets-sync-ownership.md) |
| 12 | Calls with unknown outcome to non-idempotent providers are not auto-retried | [012](adr/012-unknown-outcome-no-auto-retry.md) |

## 6. Glossary

| Term | Definition |
|---|---|
| **Workshop** | A program (single- or multi-day) with sessions, participants and a timeline. |
| **Session** | A delivery unit (usually one Zoom meeting/webinar) within a day. |
| **Activity** | A timeline item: something that should happen (send reminder, create webinar, sync sheet, wait for manual confirmation). |
| **Activity run** | One execution instance of an activity for a given schedule generation. |
| **Action execution** | One call to one integration for one target (e.g. "trigger WhatsApp `workshop_reminder` for registration R"). Every attempt is recorded. |
| **Workflow** | An event-driven automation graph built in Automation Studio. |
| **Workflow version** | An immutable, published snapshot of a workflow graph. |
| **Workflow execution** | One run of a workflow version for a trigger event. |
| **Adapter** | Code module that wraps one external provider behind an internal port interface. |
| **Communication trigger** | A reference to a workflow in the client's existing WhatsApp/email system (e.g. `workshop_reminder`). |
| **Schedule generation** | A monotonically increasing integer on an activity; every reschedule bumps it and invalidates older runs (fencing token). |
| **Outbox** | Table of domain events written in the same DB transaction as state changes, relayed to Inngest afterwards. |
| **Inbox** | `webhook_events` table: every inbound webhook stored first, processed asynchronously. |
| **Needs attention** | Terminal-until-human state: the system could not safely proceed without an admin decision. |

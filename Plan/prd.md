The scope is much clearer now. You don't need to build another CRM, WhatsApp automation tool, email marketing platform, or multi-tenant SaaS.

You need a centralized Workshop Orchestration Platform.

Think of it as a control center that connects the client's existing tools and automation systems, manages the entire workshop lifecycle, and coordinates everything according to a configurable timeline.

The platform should control when something happens, what system performs it, what happens next, and what happens if something fails.
Workshop OS — Automation & Integration Specification

Revised project scope
 Single business deployment

1. What exactly are we building?
The platform will sit between all existing services and coordinate their operations.
Central Workshop Admin Dashboard

One interface for controlling the entire operation


Workshop Orchestration Engine

Timeline · Events · Conditions · Scheduling · Execution · Monitoring


WhatsApp
Existing automation

Email
Existing automation

Zoom
Meeting / webinar control

Google Sheets
Participant data

Google Calendar
Schedule synchronization

Google Docs
Documents and reports



Important scope boundaries
Module	What we will build
WhatsApp	Connect and trigger existing automation
Email	Connect and trigger existing automation
Zoom	Manage meetings, webinars and recordings
Google Sheets	Import, export and synchronize data
Google Docs	Generate and manage documents
Google Calendar	Create, update and synchronize events
Workshop timeline	Build and execute schedules
Automation engine	Coordinate all integrations
Admin dashboard	Control and monitor everything

Not building:
- WhatsApp bulk messaging software.
- Email marketing software.
- A new email template/campaign management platform.
- A new CRM marketing automation engine.
- Multi-tenant SaaS infrastructure.
The existing communication automation systems remain responsible for sending messages. Our platform invokes them through their available APIs, webhooks or workflow triggers.

2. Core automation architecture
I recommend dividing the automation system into five independent components.
01
Workshop Timeline Engine

Stores the complete schedule of the workshop, including days, sessions, activities and scheduled execution times.
02
Event & Trigger Engine

Detects events such as registration, payment success, session completion, attendance updates and schedule changes.
03
Workflow Orchestrator

Decides which actions should execute, in what order, and under which conditions.
04
Integration Adapter Layer

Connects to Zoom, existing WhatsApp and email automations, Google Workspace and other external services.
05
Execution & Monitoring Engine

Tracks execution, retries failed actions, records results and provides administrative controls.
The important distinction is that the timeline defines what should happen, while the orchestration engine actually makes it happen.
3. Workshop Timeline Automation
This is the most important feature of the platform.

An administrator should be able to create a complete workshop schedule and let the system execute it automatically.
3.1 Timeline structure
Each workshop contains:

- Workshop
- Workshop days
  - Sessions
    - Activities
      - Automation actions
For example:
Medical Coding Workshop
Published

November 15–16 · IST

Day 1 — November 15
1. 08:00 AM — Participant reminder
   
   Trigger existing WhatsApp and email workflows.
   Scheduled
2. 08:45 AM — Final joining reminder
   
   Send joining information.
   Scheduled
3. 09:00 AM — Workshop begins
   
   Execute the configured Zoom delivery action.
   Scheduled
4. 10:00 AM — Session 1 completed
   
   Publish learning materials and update progress.
   Waiting
5. 10:30 AM — Session 2 begins
   
   Make the next session available.
   Waiting
6. 12:00 PM — Assessment
   
   Publish assessment and update participant records.
   Waiting
7. 01:00 PM — Day 1 concludes
   
   Update attendance and execute follow-up workflows.
   Waiting

3.2 Activity types
The timeline builder should support the following activity types.
Activity	Automation behaviour
WhatsApp trigger	Invoke existing WhatsApp workflow
Email trigger	Invoke existing email workflow
Zoom meeting	Create/update meeting
Zoom webinar	Create/update webinar
Start Simulive	Execute supported Zoom scheduling operation
Publish recording	Make recording accessible
Publish material	Attach resources to session
Google Sheet sync	Update participant/session data
Google Doc generation	Generate a document
Calendar event	Create/update calendar event
Participant status	Change participant status
Attendance sync	Import or update attendance
Wait	Delay subsequent execution
Conditional branch	Execute based on conditions
Admin notification	Notify responsible staff
Manual task	Wait for human confirmation

Every activity should have:
- Name and description.
- Date and time.
- Duration.
- Execution mode.
- Assigned integration.
- Configuration.
- Dependencies.
- Success condition.
- Failure behaviour.
- Enable/disable setting.
3.3 Execution modes
Three execution modes are important.

A. Scheduled execution

Execute at a specific date and time.

Example: Send workshop reminders at 8 AM.

B. Event-based execution

Execute when an external or internal event occurs.

Example: Update attendance when Zoom provides participant attendance data.

C. Sequential execution

Execute the next activity after the previous activity succeeds or meets a configured completion condition.

Example:

Session 1 completes → Publish PDF → Wait 15 minutes → Start Session 2.

This is better than treating every activity as an independent cron job.

4. Automation Workflow Builder
In addition to the timeline, I recommend having a workflow builder for more advanced operations.

The timeline handles workshop scheduling. The workflow builder handles event-driven logic and conditional operations.
Example workflow
TRIGGER
Workshop registration closes


Fetch registered participants
Get all confirmed participants from the central database

Check payment status

PAID
Confirmed participants

Trigger confirmation workflow


UNPAID
Pending participants

Notify admin for review




Synchronize participant records
Update the connected Google Sheet

Workflow completed


Workflow builder features
The admin should be able to configure:
1. Trigger.
2. Conditions.
3. Actions.
4. Delay.
5. Branching.
6. Dependencies.
7. Error handling.
8. Execution limits.
Example conditions:
- Payment status equals paid.
- Participant belongs to workshop X.
- Attendance is greater than 75%.
- Session status equals completed.
- Workshop is currently active.
- Participant has not received the reminder.
I would use React Flow for the visual editor, but keep workflow execution independent of the frontend.

5. Detailed integration automation
This section defines exactly what each external system should do.
5.1 Zoom — Workshop delivery control









Zoom

Meeting and webinar management


Zoom is the primary workshop delivery system.
Operations to support
Admin operation	System action
Create session	Create Zoom meeting/webinar
Update session	Update meeting settings
Schedule session	Set date and time
Cancel session	Cancel Zoom session
Get joining URL	Retrieve participant link
Get host URL	Retrieve host link securely
Start delivery	Use supported provider scheduling/delivery capability
Retrieve attendance	Fetch participant reports
Retrieve recording	Fetch cloud recording metadata
Recording ready	Process recording-ready webhook
End session	Track provider-confirmed completion where available

Example automation
Admin creates a workshop session for 9 AM.
1. Platform creates a Zoom webinar.
2. Stores the Zoom meeting/webinar ID.
3. Saves the joining URL.
4. Updates the Google Calendar event.
5. Invokes the existing WhatsApp workflow.
6. Invokes the existing email workflow.
7. At the scheduled time, the delivery action runs if supported.
8. Zoom webhooks update session status.
9. Attendance data is retrieved.
10. Participant records are updated.
Important technical distinction: Scheduling a Zoom meeting, starting a meeting, and playing a prerecorded Simulive webinar are different capabilities. The integration must use the appropriate Zoom product and API permissions. Simulive availability depends on the client's license. Do not assume ordinary Zoom meeting APIs can start arbitrary prerecorded video playback.
5.2 WhatsApp — Existing automation integration









WhatsApp Business

Existing message automation


We are not rebuilding WhatsApp automation.

We need a connector that can invoke the client's existing WhatsApp workflows.

For example, suppose the existing WhatsApp system exposes these triggers:
- registration_confirmation
- workshop_reminder
- payment_confirmation
- joining_instructions
- session_reminder
- workshop_completed
Our platform should send the appropriate trigger with participant and workshop information.

Example payload:
{
  "trigger": "workshop_reminder",
  "participant_id": "P1001",
  "workshop_id": "W1001",
  "scheduled_at": "2026-11-15T08:00:00+05:30",
  "variables": {
    "name": "Rahul",
    "workshop_name": "Medical Coding",
    "joining_link": "https://example.com/join"
  }
}


The integration should receive and record the execution response.

The same approach applies to email.
5.3 Email — Existing automation integration









Existing Email Automation

External sending and campaign workflows


Supported triggers:
- Registration confirmation.
- Payment receipt.
- Workshop joining details.
- Schedule changes.
- Session resources.
- Assessment notifications.
- Completion notifications.
- Certificate delivery.
Our platform controls when and for whom the email workflow executes, not how the email is designed or delivered.
5.4 Google Sheets — Participant data synchronization









Google Sheets

Import, export and synchronization


This is particularly important if the client already manages participants in spreadsheets.

Required automation:
- Import participant records.
- Detect new spreadsheet rows.
- Update participant information.
- Sync registration status.
- Sync payment status.
- Sync attendance.
- Sync workshop completion.
- Export final reports.
Example:

Google Sheet → New participant row → Platform detects new record → Validate details → Create participant → Associate workshop → Trigger registration workflow.

For reliability, use a scheduled reconciliation as well as event-based updates. Google Sheets does not provide a general-purpose webhook for arbitrary spreadsheet row changes in the same way as a database event stream.
5.5 Google Calendar — Schedule synchronization









Google Calendar

Workshop and session scheduling


Automations:
- Create workshop events.
- Create session events.
- Update events after schedule changes.
- Cancel events.
- Assign trainers.
- Add Zoom joining details.
- Sync trainer availability where authorized.
- Avoid duplicate calendar events.
Example:

Admin changes Session 2 from 10 AM to 11 AM.

The system should:
1. Update the timeline.
2. Update the corresponding Zoom schedule if supported.
3. Update Google Calendar.
4. Invoke existing WhatsApp notification workflow.
5. Invoke existing email notification workflow.
6. Record the changes in the activity log.
5.6 Google Docs — Document automation









Google Docs

Workshop documentation


Automated document generation:
- Participant attendance sheet.
- Workshop schedule.
- Trainer assignment sheet.
- Completion report.
- Workshop summary.
- Participant certificates, if using a Docs-based template.
Admin selects a template once. The system populates participant and workshop information and stores the resulting document reference.

6. The most important part: Workshop lifecycle automation
I would create reusable workflows for each stage of a workshop.
Workshop lifecycle
1. Workshop preparation
   
   Create workshop, add sessions, upload recordings, configure Zoom, validate integrations and synchronize the calendar.
2. Registration
   
   Import or receive participants, validate records, associate participants with workshops and trigger existing confirmation workflows.
3. Pre-workshop
   
   Schedule reminders, synchronize joining links, verify participant data and notify staff of unresolved issues.
4. Workshop execution
   
   Execute the timeline, manage Zoom sessions, publish resources, process events and track attendance.
5. Workshop completion
   
   Calculate completion status, synchronize attendance, generate reports and trigger existing follow-up workflows.
6. Post-workshop
   
   Generate documents, certificates, update records, export reports and archive workshop data.

6.1 Preparation automation
Trigger: Admin publishes workshop.

Actions:
1. Validate all session configurations.
2. Check recording availability.
3. Check Zoom integration.
4. Verify required communication triggers exist.
5. Create calendar events.
6. Prepare participant spreadsheet.
7. Generate the workshop schedule document.
8. Create all scheduled automation jobs.
9. Notify admin that the workshop is ready.
If a critical dependency is missing, publishing should be blocked or require an explicit override.
6.2 Registration automation
Trigger: New participant registered or imported.

Actions:
1. Check for duplicate participant.
2. Validate email and phone number.
3. Create or update participant.
4. Associate participant with workshop.
5. Check payment status if applicable.
6. Trigger existing WhatsApp automation.
7. Trigger existing email automation.
8. Synchronize Google Sheet.
9. Update registration dashboard.
6.3 Pre-workshop automation
Example schedule:
Time	Action
7 days before	Preparation check
3 days before	Participant list reconciliation
24 hours before	Reminder workflows
1 hour before	Joining instructions
15 minutes before	Final reminder
Workshop start	Execute delivery schedule

These times should be configurable per workshop.
6.4 During-workshop automation
This is where the timeline engine becomes essential.

For every activity:
1. Check whether the previous required activity has completed.
2. Check current workshop state.
3. Execute the configured action.
4. Receive the external response.
5. Update execution status.
6. Wait for the next scheduled or event-based trigger.
7. Execute subsequent actions.
For example:

Session 1 → Zoom delivery → Attendance retrieval → Google Sheet sync → Publish resource → Session 2.

A failed Google Sheets update should not stop the next Zoom session.
6.5 Post-workshop automation
Trigger: Workshop completion.

Actions:
1. Fetch final Zoom attendance reports.
2. Calculate attendance percentage.
3. Update participant completion status.
4. Generate attendance report.
5. Generate eligible certificates.
6. Save documents to the appropriate storage.
7. Update Google Sheets.
8. Trigger existing feedback email/WhatsApp workflows.
9. Notify admin about participants requiring manual review.
10. Archive workshop execution data.

7. Full admin control — Essential functionality
This is what will make the platform valuable to the client.

The admin should not only configure automation. They should be able to control it while the workshop is running.
Workshop control center
Workshop Control Center
Live simulation

In Progress
Medical Coding Fundamentals · Day 1
248
Participants

182
Attendance confirmed

12
Completed activities

2
Require attention



Current activity
Session 2 — ICD-10-CM
Running

Next activity: Assessment · 12:00 PM

Workshop controls
 Pause automation
 Resume automation
 Reschedule
 Emergency stop

Illustrative interface, not a live workshop connection.
Controls that must exist
Control	Expected behaviour
Start	Activate workshop automation
Pause	Suspend eligible future executions
Resume	Continue paused automation
Stop	Stop future executions
Reschedule	Change scheduled activity times
Skip	Skip a specific activity with reason
Retry	Retry a failed action
Execute now	Manually execute an action
Edit	Change activity configuration
Duplicate	Duplicate a workflow
Emergency stop	Halt all eligible automated actions
Manual override	Mark an activity as manually completed

Critical distinction: Pausing the orchestration engine does not automatically pause a Zoom webinar already in progress. The admin controls must distinguish between internal scheduled jobs and external actions already running.

8. Automation execution and failure handling
This is an important engineering requirement. Workshop automation cannot depend on every API succeeding on the first attempt.

Every action needs an execution record.

Example:
Activity	Integration	Status
Registration reminder	WhatsApp	Success
Registration confirmation	Email	Success
Create Zoom webinar	Zoom	Success
Update Google Sheet	Sheets	Failed
Calendar synchronization	Calendar	Retrying

Failure-handling rules
Case 1: WhatsApp trigger fails
- Record failure.
- Retry based on configured policy.
- Notify admin after retry limit.
- Do not block the workshop timeline.
Case 2: Zoom webinar creation fails
- Mark session as requiring attention.
- Prevent dependent joining instructions from being sent.
- Notify admin.
- Allow manual recovery.
Case 3: Google Sheets synchronization fails
- Save changes in PostgreSQL.
- Add synchronization to retry queue.
- Continue workshop operations.
Case 4: Workshop is rescheduled
- Recalculate all dependent scheduled jobs.
- Update Zoom and Calendar.
- Cancel obsolete jobs.
- Trigger participant notifications.
- Record the change.
Execution states
Use consistent states throughout the application:

pending → scheduled → running → succeeded

Other possible states:

failed, retrying, paused, skipped, cancelled, awaiting_confirmation, needs_attention.

9. Recommended technical architecture
Given the type of product, I would use a modular monolith with a durable workflow engine.
Next.js Admin Dashboard
Workshop Builder · Timeline · Integrations · Monitoring

Application Backend
API · Authentication · Permissions · Business Logic

Timeline Engine
Schedule and dependencies

Event Engine
Triggers and conditions



Inngest — Durable Workflow Execution
Scheduling · Retries · Delays · Step execution

Integration Adapter Layer
Zoom · WhatsApp · Email · Google APIs

PostgreSQL
Central database · Events · Execution logs · Integration state

Recommended stack
Layer	Technology
Frontend	Next.js 16, React 19
Language	TypeScript
UI	Tailwind CSS, shadcn/ui
Timeline builder	dnd-kit
Workflow visual editor	React Flow
Backend	Next.js API / tRPC
Database	PostgreSQL
ORM	Drizzle
Workflow engine	Inngest
Authentication	Better Auth
Validation	Zod
External integrations	REST APIs, OAuth, webhooks
Logging	OpenTelemetry
Monitoring	Sentry
File storage	Cloudflare R2

I would not introduce Redis initially unless you have a specific caching, locking or rate-limiting requirement that needs it.

10. Suggested database architecture for automation
Since we are not building a multi-tenant SaaS, the schema can be considerably simpler.

These are the core tables I would create.
Table	Purpose
workshops	Main workshop information
workshop_days	Individual workshop days
sessions	Training sessions
timeline_activities	Scheduled activities
activity_dependencies	Activity execution dependencies
participants	Central participant records
registrations	Workshop enrollment
integrations	Connected external accounts
integration_actions	Available external operations
automation_workflows	Workflow definitions
workflow_versions	Published workflow versions
workflow_executions	Workflow execution history
activity_executions	Individual action execution
webhook_events	Incoming external events
integration_sync_logs	External synchronization history
communication_triggers	References to existing message workflows
audit_logs	Administrative activity

One important design decision: keep workflow_executions and activity_executions separate.

A workflow can contain ten actions. If action seven fails, you should be able to retry that action without executing the first six again.

Also, use a single integration configuration for the client's connected accounts, with encrypted credentials and appropriate access controls.

11. Recommended admin navigation
I would simplify the previous 19-section sidebar into a more focused operational interface.
Workshop OS


Overview

Workshops

Timeline & Schedule

Automation Studio

Participants

Zoom Sessions

Recordings & Resources

Integrations

Execution Monitor

Reports & Documents

Activity & Audit Logs

Settings


The WhatsApp and email systems do not need separate campaign-management modules in this dashboard. They can be managed from the Integrations section, while their executions are visible in the Execution Monitor.

12. Development scope and priorities
I would divide the implementation into these stages.
Phase	Deliverables
1. Foundation	Database, authentication, integration connections
2. Workshop engine	Workshop builder, sessions, timeline builder
3. Orchestration	Triggers, conditions, scheduling, dependencies
4. Integrations	Zoom, existing WhatsApp/email triggers, Google Workspace
5. Operations	Execution monitor, retries, pause/resume, manual overrides
6. Completion	Attendance, reports, documents, production testing

MVP — What should actually be delivered first?
Recommended MVP
I would prioritize these features:
1. Single-business admin dashboard.
2. Workshop creation and configuration.
3. Multi-day workshop timeline builder.
4. Activity scheduling and dependencies.
5. Inngest-based workflow orchestration.
6. Existing WhatsApp automation integration.
7. Existing email automation integration.
8. Zoom meeting/webinar integration.
9. Google Sheets import/export/sync.
10. Google Calendar synchronization.
11. Central execution monitoring.
12. Pause, resume, retry and manual override controls.
13. Workshop activity logs.
Google Docs generation, advanced conditional workflows, participant self-service portal, certificates and advanced analytics can follow.

13. Important technical questions before implementation
There are three details that will determine how we design the integration layer.

1. Where do the existing WhatsApp and email automations currently run?

For example, are they built using n8n, Make, GoHighLevel, a custom backend, or another service? This determines whether we invoke them using webhooks, API calls or another mechanism.

2. How are recorded workshops currently delivered?

Are the recordings uploaded to Zoom Simulive, stored in Google Drive, or hosted somewhere else? This determines how the timeline engine controls the actual delivery.

3. How does the client currently manage workshop participants?

Are participants maintained in Google Sheets, an existing CRM, or a custom registration system?

These answers will determine the exact integration contracts.

My final architectural recommendation: Build the system around a provider-independent Workshop Timeline Engine + Durable Workflow Orchestrator. Keep WhatsApp and email as external execution services. Make Zoom and Google Workspace controllable through dedicated adapters. PostgreSQL should own the workshop state, while the orchestration engine owns the execution state.

That gives the client one platform to manage the entire workshop operation without rebuilding tools they already have.
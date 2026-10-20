# Helios — Operations, connected.

### Building the connections between leads, conversations, and appointments

Helios is an operations platform for home-service teams. It connects lead intake, agent calling, appointment scheduling, contractor follow-up, messaging, and reporting.

An appointment is the visible result of a longer chain. A lead needs an owner. A call needs an outcome. A recording needs to stay associated with the right conversation. A reminder needs to reach the intended person, even when a provider is slow or a worker restarts.

The engineering story of Helios is how those connections became explicit: through persistent identities, durable work queues, reconciliation, and increasingly useful operational feedback.

**Focus:** application engineering · telephony integration · asynchronous workflows · applied AI · operational reliability

**Core technologies:** Python, Reflex, FastAPI, PostgreSQL, FreeSWITCH, VICIdial, Anthropic Claude, AssemblyAI

**Project stage:** an operational system evolving toward a broader SaaS offering

**Presentation:** source code remains private; selected workflows and engineering decisions can be discussed in a guided walkthrough.

## One workflow, several systems

Agents work leads and record call outcomes. Administrators review appointments and coordinate operations. Contractors manage assignments and follow-up. Background services connect these actions to communications, recordings, external systems, and reports.

```mermaid
flowchart LR
    Team["Agents, administrators<br/>and contractors"] --> App["Helios operations app"]
    App <--> DB[("PostgreSQL<br/>business records and jobs")]
    App --> Voice["Call-control services"]
    Voice --> FS["FreeSWITCH"]
    Voice -->|"Provider API paths"| Carriers["Voice providers"]
    FS <-->|"Carrier connections"| Carriers
    Carriers -->|"Call events and recording metadata"| API
    Vici["VICIdial"] <-->|"Lead delivery, acknowledgements<br/>and call outcomes"| API["Integration APIs"]
    API <--> DB
    DB <--> Workers["Background workers"]
    Workers --> Comms["Messaging and email providers"]
    Comms -->|"Delivery events and replies"| API
    Workers -->|"Retrieve recording files"| FS
    Workers --> Media["Private media storage"]
    App -->|"Notes review"| AI["Language-model and<br/>speech-to-text services"]
    API -->|"Reply classification"| AI
    Workers -->|"Transcription, call review<br/>and report narratives"| AI
    API --> Chat["Team chat approvals<br/>and alerts"]
    Workers --> Context["Reporting, exports<br/>and weather providers"]
    App -.-> Observe["Errors, queue health<br/>and uptime monitoring"]
    Workers -.-> Observe
```

This is a logical overview. Provider choices and enabled workflows vary by deployment.

## 1. Give cross-system work a durable identity

Connecting a CRM to a dialer raises a deceptively difficult question: when does a lead become the dialer's responsibility?

If a remote insert succeeds but its acknowledgement is lost, retrying can create duplicate work. If the application releases ownership too early, the same lead can become available to another caller.

The VICIdial integration addresses this with reservations and explicit delivery state. A worker pulls eligible leads from Helios, inserts them into the dialer in a held state, verifies the remote record, and activates them after confirmation. Call outcomes return through callbacks and reconciliation.

The distinction between a delivery ledger and live dialer inventory also matters. A ledger can retain unfinished records even when the dialer has moved on. The feed design therefore uses observed remote inventory for pacing and keeps ownership records for reconciliation.

**Engineering lesson:** define ownership before delivery, and design recovery for the point where two systems disagree.

## 2. Make scheduled work visible and recoverable

Appointment reminders, email digests, exports, invoice generation, transcription, and recording synchronization all need to happen outside a browser request.

Helios progressively moved recurring workflows into a shared PostgreSQL-backed job system. Scheduled scripts enqueue work; workers process it using identifiers appropriate to the task, such as an appointment, recipient, recording, or contractor and billing period. Several workflows retain a direct execution path for controlled recovery.

This introduced an operational distinction: a scheduler can be present while the intended work never reaches the queue. A worker can complete a job while the external outcome still needs confirmation. Health checks therefore need to account for stale work and expected recent success, as well as exceptions.

**Engineering lesson:** persistence makes work inspectable; useful monitoring checks whether the workflow is making progress.

## 3. Treat uncertain message outcomes explicitly

An external messaging request can time out after the provider accepted it. Automatically retrying that request risks sending the same message again.

The outreach implementation records a send claim before contacting the provider and represents uncertain outcomes explicitly. Eligibility is shared across audience selection and the send-time recheck. Delivery callbacks update the record as more information arrives.

The surrounding workflow includes consent records, opt-out handling, contact-local sending windows, sender capacity, and a human inbox. Inbound classification helps organize replies while preserving a human response workflow.

These are engineering controls within the application; they are not a claim of legal certification or a guarantee of message delivery.

**Engineering lesson:** uncertainty deserves its own state. Recovery should account for what the provider may already have done.

## 4. Preserve the connection between a call and its recording

A useful call record involves more than a completed audio file. Provider callbacks, call identifiers, media synchronization, storage, playback, and transcription must agree about the same conversation.

The engineering records describe that pipeline evolving through callback correlation, private object storage, signed playback access, and background transcription. Recovery work also exposed a broader principle: an unmatched recording may still be valuable evidence. Its source should be preserved while ownership is resolved.

**Engineering lesson:** stable identifiers and recoverable intermediate states are essential when a user-facing feature spans several services.

## 5. Add field context without delaying the core workflow

The October weather work extends appointment records with weather snapshots and a storm archive. Its design captures context at meaningful points in the appointment lifecycle and retains the source and capture time.

The provider chain supports fallback from a primary weather service to a secondary source and then to a labelled stored snapshot. Quota controls prioritize later appointment checks. Weather enrichment is designed to fail without blocking an SMS or email.

A weather observation remains a reading for a particular place and time. A nearby station reading or forecast provides context; it does not conclusively establish conditions at an individual property.

The October 19 engineering note reports an initial production backfill and live checks. Follow-up deployment steps remained unconfirmed at the time of that record. This newer work is documented separately from the repository snapshot used to prepare this case study.

**Engineering lesson:** enrichment should retain provenance, communicate freshness, and degrade gracefully.

## 6. Keep AI behind confidence gates and human review

Helios uses language models in several narrow roles rather than as one autonomous agent. Each role has a bounded input, a constrained output, and a defined effect on the business record.

| Role | Model input | Effect |
| --- | --- | --- |
| Appointment notes check | Notes written when an appointment is submitted | Flags notes that need attention; agents and administrators can filter on the flag |
| Call review | Speaker-labelled call transcript | Rates the call Good, Review, or Urgent against the team's qualification and misrepresentation criteria |
| Appointment SMS replies | A homeowner's or contractor's message with matched appointment context | Classifies intent with a confidence score; reschedule and cancellation requests go to a team-chat approval with Approve/Deny buttons |
| Outreach reply classification | A reply to an outreach message | Assigns one of eight categories with a confidence score; flags interested contacts for a human-staffed inbox; sends no automatic replies |
| Reporting | Aggregated funnel and call-event metrics | Writes short narrative insights for contractors and recommendations on phone-number health |
| Lead research | Public excerpts from business websites | Runs on a local model to classify businesses and propose contact candidates with quoted evidence |

Call recordings are transcribed with speaker diarization in background jobs. The transcript then feeds call review.

Several design patterns recur:

- **Deterministic paths come first.** Opt-out keywords are handled before any model is called, and configurable keyword rules classify common replies without a model call.
- **Uncertainty routes to people.** Classifiers return structured output with a confidence score. Low confidence, or output that cannot be parsed, sends the message to human review instead of an automatic action.
- **Errors are asymmetric by design.** Opt-out-type replies are treated as do-not-contact at high confidence; uncertain replies go to review. An appointment's call-review flag can only escalate: the worst reviewed call wins.
- **AI never blocks the core workflow.** If the model call fails or times out, the appointment is still saved and its flag stays pending. Reports fall back to template text.
- **Prompts and decisions are reviewable.** Administrators edit prompts without a deploy. Classifications record the prompt, raw response, confidence, and model version. Operators can reclassify replies, and those corrections are kept to measure prompt quality.
- **The model is a setting.** One configuration value selects the model for every call site. A read-only replay tool runs past messages through a candidate model and compares the results to the stored labels without sending anything or changing data.
- **Extracted facts stay unverified.** The lead-research tool accepts a model-proposed name only when it is backed by an exact quote from the fetched page. Names stay marked as to be confirmed, and ownership is never inferred from a title alone. A [standalone version of the classifier](https://github.com/daniellopezq/local-business-classifier) is public.

The replay tool also showed a limit: without deterministic sampling, the same message could receive a different label on a different run. In the recorded case the routing outcome was the same. Deterministic classifier settings are a recorded follow-up. Model output is a triage aid; it does not determine compliance, call quality, or a person's identity.

**Engineering lesson:** give a model a narrow decision and a way to say it is unsure, and give everything it is unsure about a human path.

## 7. Make operational changes testable

Telephony and background processing introduced failure modes that basic syntax checks could not reveal: behavior after a cold restart, mismatches between deployed configuration and source, and callbacks that arrive after the initiating request has finished.

The project records describe staged canaries, real call-path checks, rollback decisions, migration tracking, schema comparisons, and later restore testing. Changes that failed their operational checks were sometimes reverted while the underlying behavior was investigated.

This history also changed how progress was described. Code written, tests passing, a service restarted, and a workflow verified in production are separate milestones.

**Engineering lesson:** test the boundary being changed, and retain enough evidence to explain what was actually verified.

## How the project evolved

| Period | Engineering focus |
| --- | --- |
| March–April 2026 | Agent and contractor workflows, appointment history, sign-in, recording and transcription, prospect intake, provider integration, AI review of appointment notes with administrator-editable prompts, and SMS intent classification with team-chat approval. |
| May–June 2026 | FreeSWITCH calling, staged telephony changes, recording recovery, lead import improvements, early background-job migrations, a speech-to-text migration with speaker labels, and AI call review. |
| July–August 2026 | Durable recurring work, schema guardrails, VICIdial lead delivery, outcome reconciliation, and caller-ID inventory synchronization. |
| September–October 2026 | SMS outreach and operator inbox workflows, outreach reply classification, a shared model setting with replay testing, local-model lead research, repository recovery practices, pilot preparation, and appointment weather history. |

The periods summarize dated engineering records. Individual features have different implementation, test, and deployment histories.

## What this project demonstrates

Helios brings product workflows into contact with the realities of distributed systems: delayed callbacks, partial failures, duplicated work, inconsistent historical data, and changing operational requirements. It also applies language models to that work in bounded roles, with confidence gates and human review.

The useful result is a set of connected capabilities with increasingly explicit ownership and recovery behavior. The design decisions can be examined through a lead's journey into a call, an appointment's transition into contractor follow-up, or a message's path from eligibility to a provider outcome.

The next stage is to extend those practices as the product evolves: stronger verification across organizational boundaries, clearer worker responsibilities, and repeatable deployment evidence. The case study describes project work and recorded decisions; personal contribution and implementation details can be covered during a guided walkthrough.

---

*Prepared from a review of the project's engineering notes and selected repository paths. Historical production checks are attributed to those records; this document is not a fresh production audit. Internal runbooks, customer data, infrastructure identifiers, and source code are omitted.*

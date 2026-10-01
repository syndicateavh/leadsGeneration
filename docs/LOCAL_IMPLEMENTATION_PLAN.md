# Local-First Lead Generation Platform: Implementation Plan

## Objective

Build an in-house application that finds, organizes, researches, qualifies, and advances prospects through a sales process. The first version runs entirely on one computer for one internal team. It is a **local monolith**, not a distributed system or a hosted SaaS product.

The product should prioritize useful, evidence-backed prospects over raw contact volume. A score is an explanation of the team's criteria and observed signals; it is not a prediction or a claim that a person will buy.

## Scope and operating constraints

### In scope for the local version

- One Next.js application running at `localhost`.
- A database file stored on the same computer.
- Local uploads for CSV files and attachments.
- Manual entry and CSV import as the initial lead sources.
- A local, in-process job runner for imports, enrichment, scoring, and scheduled follow-ups.
- A CRM, ICP configuration, transparent scoring, research evidence, tasks, and an activity timeline.
- Connector interfaces so permitted sources can be added later without rewriting the core.

### Explicitly out of scope initially

- Multi-tenant accounts, organization isolation, billing, or public sign-up.
- Horizontal scaling, microservices, message brokers, worker fleets, Kubernetes, or cloud deployment.
- Direct scraping or any collection that breaches a source's terms, privacy law, robots policy, or access controls.
- Unattended outbound messaging. Any live communication integration will be added only after consent, provider approval, and review controls are designed.
- An AI agent with unrestricted authority to send messages, alter records, or take external actions.

## Local architecture

```text
Browser
  |
  v
Next.js app (UI + route handlers + server actions)
  |
  +-- Domain services: CRM, import, research, scoring, workflow
  +-- Local scheduler / job runner (same Node process)
  +-- Connector adapters (CSV first; permitted APIs later)
  |
  +-- SQLite database file  ->  ./data/leads.db
  +-- Local file storage     ->  ./data/uploads and ./data/exports
  +-- Optional AI provider   ->  server-side, explicit review/action gates
```

The app will use TypeScript and the existing Next.js stack. Add Drizzle's SQLite driver and migrations, validate all inputs with Zod, and keep the data-access layer behind repository/service functions. SQLite is the right starting point because it is easy to back up, portable, transactional, and needs no separate database server. A future move to local PostgreSQL is possible only if concurrency or reporting needs justify it.

`data/` and environment files will be ignored by Git. A backup is simply a timestamped copy of the SQLite file and relevant uploaded files. The app will include a local export function before adding advanced automation.

## Recommended implementation order

### Module 1 — Local Lead Inbox and CRM Foundation (build first)

This is the first module to build, even though CSV/manual import is technically a small Lead Discovery connector. Every future connector, researcher, score, workflow, and message needs a reliable canonical record to write to. Starting here makes the application useful on day one: users can add real leads, manage their pipeline, and correct data before any automation exists.

First release capabilities:

1. Initialize SQLite, migrations, local data directories, and a seed/demo dataset.
2. Add canonical `Company`, `Contact`, `Lead`, `Source`, `Tag`, `Activity`, `Note`, `Task`, and `Deal` records.
3. Provide a Lead Inbox with search, filters, list/detail views, and the pipeline:
   `NEW -> RESEARCHED -> QUALIFIED -> CONTACTED -> ENGAGED -> MEETING -> PROPOSAL -> WON | LOST`.
4. Provide manual create/edit and CSV upload with a column-mapping preview.
5. Normalize email, domain, company name, and phone values; detect duplicates before import; let a user skip, merge, or import as a separate lead.
6. Record every import, change, merge, pipeline movement, and score change in the activity timeline.
7. Export filtered leads as CSV and back up the local database.

**Definition of done:** a user can import a CSV, see an error/duplicate report, open each lead, link its contact and company, add a task/note, move it through the pipeline, and export the resulting filtered list after restarting the local application.

### Module 2 — ICP Profiles and Explainable Qualification

Create reusable ideal-customer profiles with inclusion and exclusion rules for industry, location, company size, target role, budget band, product need, and tags. Add a deterministic rule evaluator first, returning a match percentage, matching rules, failing rules, and missing data.

This module is deliberately rule-based before AI. It gives the team a controllable baseline and creates labeled feedback for future improvements.

### Module 3 — Prospect Research Workspace

Add structured research fields and evidence items: public company facts, website, industry, roles, locations, technology, hiring/growth signals, source URL, observation date, and confidence. Users can add research manually or from approved connectors. The lead detail page should show evidence separately from conclusions.

### Module 4 — Intent Signals and Lead Scoring

Implement a configurable scoring rubric. A signal stores its type, evidence, freshness, weight, and source. The final score is a reproducible calculation such as:

```text
score = ICP-fit points + intent-signal points + data-completeness points - exclusion penalties
```

Show a complete score breakdown and thresholds such as `research`, `qualified`, and `priority`. Changes to the rubric should trigger a local rescore job and produce an activity entry explaining the new score.

### Module 5 — Connector Framework and Approved Sources

Define a connector contract rather than embedding source-specific logic in CRM code:

```ts
interface LeadConnector {
  key: string;
  validateConfig(config: unknown): Result;
  discover(input: DiscoveryInput): AsyncIterable<DiscoveredLead>;
  normalize(record: DiscoveredLead): NormalizedLead;
}
```

The first connector remains CSV. Add manual-website research next, then only official APIs, licensed providers, and sources for which the business has permission. Each connector run stores configuration (with secrets excluded), source attribution, raw payload reference where lawful, record counts, errors, and timestamps.

### Module 6 — Review Queue and AI Assistance

AI assistance is introduced as a constrained drafting and analysis tool. It can summarize research, classify a response, propose a score rationale, identify missing fields, draft a personalized message, and recommend a next task. Store the model, prompt template version, input evidence IDs, output, and user decision.

Initial AI actions are always **draft/review only**. A person approves any CRM mutation beyond a clearly labeled, reversible bulk action, and no message is sent by the AI.

### Module 7 — Local Workflow Automation

Build a simple rule-driven workflow engine before a visual canvas. Support a compact set of nodes: trigger, condition, filter, update lead, create task, wait-until, notify in-app, and webhook. Persist workflow definitions and every execution step. Use the in-process scheduler to run due work; missed jobs are rechecked on application startup.

After the rule editor is proven, add a visual builder that writes the same versioned workflow definition. Add a dry-run mode and an execution log before enabling any external action.

### Module 8 — Communications Integrations

Create a common conversation/message model before individual channel adapters. Start by logging manually sent email or a local test channel. Only later add approved email, WhatsApp, SMS, or social integrations, with per-channel consent status, send limits, template/review controls, unsubscribe handling, and provider-specific permissions.

### Module 9 — Analytics and Operating Feedback

Add local dashboards for source quality, duplicate rate, ICP match distribution, score-to-stage conversion, stage aging, task completion, and workflow outcomes. Use these reports to refine ICPs and score weights rather than treating activity volume as success.

## Core data model

| Area | Initial records | Important relationships |
| --- | --- | --- |
| CRM | `companies`, `contacts`, `leads`, `deals` | A lead belongs to one primary company/contact and can have zero or more deals. |
| Pipeline | `pipeline_stages`, `stage_history` | Every stage transition is immutable and attributable. |
| Research | `research_profiles`, `evidence_items`, `signals` | Evidence is source-attributed and can support several signals. |
| Qualification | `icp_profiles`, `icp_rules`, `qualification_runs`, `score_breakdowns` | Each result retains the rules and weights used to calculate it. |
| Organization | `tags`, `lead_tags`, `notes`, `tasks`, `activities` | Activities provide the chronological audit trail. |
| Import | `import_runs`, `import_rows`, `duplicate_candidates`, `sources` | Original input and decisions remain traceable. |
| Automation | `workflows`, `workflow_versions`, `workflow_runs`, `workflow_steps` | Workflow runs are resumable and auditable. |
| Communication | `conversations`, `messages`, `channel_accounts` | Channel implementation stays behind a common data model. |

Use UUIDs for primary IDs, `created_at`/`updated_at` timestamps, explicit source attribution, and soft deletion for CRM records. Store raw imported rows separately from normalized fields so a correction never loses the original context.

## Suggested code boundaries

```text
app/
  (app)/leads/                 # Lead inbox and detail pages
  (app)/companies/
  (app)/icp/
  (app)/research/
  (app)/workflows/
  api/
components/
lib/
  db/                          # Drizzle schema, migrations, connection
  domain/                      # CRM, import, scoring, research services
  connectors/                  # Connector interface and implementations
  jobs/                        # Local scheduler and handlers
  ai/                          # Prompt templates and reviewed actions
  validation/                  # Shared Zod schemas
data/                          # Ignored local runtime data
docs/
```

Pages and route handlers should be thin. Business rules belong in `lib/domain`, database queries in `lib/db`, and source-specific work in `lib/connectors`. This keeps a future connector or user interface from changing the CRM's core behavior.

## Build sequence for Module 1

1. **Project foundation:** install the SQLite adapter, create the schema/migration setup, local configuration validation, `.gitignore` entries for `data/` and `.env*`, and a seed command.
2. **CRM domain:** create repositories and services for companies, contacts, leads, tags, notes, activities, tasks, and stage transitions; include Zod validation and unit tests for status rules.
3. **Lead Inbox UI:** build the dashboard shell, lead table, filters, pagination, create/edit forms, lead detail screen, notes/tasks, and activity feed.
4. **CSV connector:** implement upload limits, column mapping, preview, normalization, row-level validation, duplicate detection, import report, and undo for the latest import where safely possible.
5. **Data safety:** add database backup, CSV export, clear local-data documentation, and a demo dataset that never mixes with production data.
6. **Verification:** test a clean install, database migration, a valid/malformed CSV, duplicate cases, stage history, search/filtering, export, and restart persistence.

## Detailed phase-wise implementation plan

The phases below are delivery gates, not independent projects. Complete and verify each phase before starting features that depend on it. Every phase should leave the local application runnable and the database migratable without deleting existing data.

### Phase 0 — Local foundation and technical baseline

**Goal:** turn the starter project into a predictable local development environment with clear data-safety rules.

**Implementation tasks:**

1. Confirm the supported Node.js and npm versions and document the local startup commands.
2. Add environment validation for settings such as database path, upload directory, export directory, optional AI key, and application URL.
3. Add the SQLite driver and configure Drizzle for a file at `./data/leads.db`.
4. Create migration, seed, reset-demo-data, backup, and type-check scripts. The reset script must affect demo data only unless the user explicitly confirms otherwise.
5. Add `data/`, uploaded files, database sidecar files, backups, logs, and `.env*` to `.gitignore`, while keeping an `.env.example` with safe placeholders.
6. Create shared logging and error helpers. Logs must redact secrets and avoid dumping complete prospect records.
7. Add a health check that verifies application startup, database access, migration version, and writable local directories.
8. Establish the initial test setup for domain unit tests and route/service integration tests.

**Deliverables:** local configuration, database connection, first migration, scripts, health check, seed data, and a short local setup section in the README.

**Verification:** a clean clone can be installed, migrated, seeded, started, stopped, and restarted; seeded records remain available after restart; no runtime data appears in Git status.

**Exit gate:** the app opens locally, the health check is green, a backup can be created, and a fresh database can be rebuilt entirely from migrations.

### Phase 1 — CRM data model and domain services

**Goal:** establish the canonical system of record used by every later module.

**Implementation tasks:**

1. Create schemas and migrations for companies, contacts, leads, sources, pipeline stages, stage history, tags, lead tags, notes, tasks, activities, deals, and saved filters.
2. Add constraints and indexes for normalized email, normalized company domain, stage, owner, score, source, and creation date.
3. Implement normalization helpers for email, phone, website/domain, company name, location, and whitespace.
4. Implement repository functions for database access and domain services for create, update, archive, restore, merge, assign tags, add notes, manage tasks, and transition stages.
5. Define allowed stage transitions and store the previous stage, new stage, actor, reason, and timestamp for every transition.
6. Implement soft deletion and exclude archived records from normal queries without permanently erasing their history.
7. Create a transaction-safe merge service that moves related records to the retained company/contact/lead and records the decision in the activity log.
8. Add validation schemas at the domain boundary rather than trusting form or API input.

**Deliverables:** migrated CRM schema, typed repositories, domain services, validation schemas, and seed fixtures covering the full pipeline.

**Verification:** unit tests cover normalization, validation, allowed/blocked transitions, merging, soft deletion, and transaction rollback. Integration tests confirm foreign keys and cascade behavior do not lose audit history.

**Exit gate:** all CRM operations can be performed through tested service functions without direct database calls from pages or UI components.

### Phase 2 — Lead Inbox and CRM user interface

**Goal:** provide a usable local CRM before adding automation.

**Implementation tasks:**

1. Create the application shell, navigation, dashboard summary, empty states, loading states, and recoverable error states.
2. Build a paginated Lead Inbox with server-side search, sort, and filters for stage, score, tags, source, location, company, owner, and created date.
3. Build manual create/edit forms for companies, contacts, and leads with inline validation.
4. Build the lead detail workspace with overview, linked company/contact, source, notes, tasks, tags, stage history, activities, research, and qualification placeholders.
5. Add a pipeline board backed by the same stage-transition service as the detail view.
6. Add bulk tagging, assignment, stage changes, and archive actions with a confirmation and an activity record.
7. Add optimistic UI only for reversible actions; show a clear saved/failed state for every mutation.
8. Make core screens keyboard accessible and usable at common laptop widths.

**Deliverables:** Lead Inbox, lead detail page, company/contact pages, pipeline board, notes/tasks, and activity timeline.

**Verification:** UI tests cover creating and editing a lead, validation errors, filtering, pagination, stage changes, tasks, notes, bulk actions, archive/restore, and persistence after restart.

**Exit gate:** a user can run the basic sales pipeline from manual lead entry through `WON` or `LOST` without accessing the database directly.

### Phase 3 — CSV connector, normalization, and duplicate review

**Goal:** make CSV the first production-quality discovery connector.

**Implementation tasks:**

1. Define the common connector contract, normalized lead payload, source metadata, connector-run result, and row-level error format.
2. Implement CSV file-size and row-count limits, encoding/header validation, safe parsing, and temporary-file cleanup.
3. Build a column-mapping screen with automatic suggestions and a preview of normalized values.
4. Allow a mapping preset to be saved for recurring exports from the same provider.
5. Validate every row and split results into ready, warning, duplicate candidate, and rejected groups before committing.
6. Implement deterministic duplicate candidates using normalized email, phone, domain, and company/contact combinations. Do not auto-merge uncertain fuzzy matches.
7. Build a review interface for skip, merge, update existing, or create separate decisions.
8. Commit accepted rows in batches and record the original row, normalized values, decision, resulting record IDs, and errors.
9. Add import-run history, downloadable error reports, and a safe undo option that only reverses records and changes owned by that import.
10. Add filtered CSV export with a documented, stable set of columns.

**Deliverables:** reusable connector interface, CSV import wizard, duplicate review queue, import history/report, undo, and export.

**Verification:** test valid files, missing headers, invalid values, mixed encodings, quoted commas, large files within limits, duplicate rows inside one file, duplicates against existing data, partial failures, undo, and re-import idempotency.

**Exit gate:** a non-technical user can import a real CSV, resolve duplicates, understand rejected rows, and verify the resulting leads without database cleanup.

### Phase 4 — ICP profiles and deterministic qualification

**Goal:** turn the team's target-customer definition into transparent, repeatable rules.

**Implementation tasks:**

1. Add ICP profiles, versioned rule sets, inclusion rules, exclusion rules, weights, thresholds, and active/inactive state.
2. Support rules for industry, location, employee range, revenue/budget band when legitimately known, target role, company category, product need, and tags.
3. Define three-valued rule results: match, no match, and unknown. Missing information must not silently become a failure.
4. Build an ICP editor with readable rule summaries and a preview against selected existing leads.
5. Implement the evaluator as a pure domain service that returns total fit, matched rules, failed rules, missing facts, exclusions, and rule-set version.
6. Store each qualification run so later changes to an ICP do not rewrite historical results.
7. Add a lead-level re-evaluate action and a batch local job for evaluating all active leads.
8. Show the explanation beside the score and allow users to flag an incorrect result for later rubric improvement.

**Deliverables:** ICP management screens, versioned rules, qualification service, batch evaluator, and explainable results on lead details.

**Verification:** rule-boundary tests cover ranges, lists, exclusions, missing data, profile versioning, and repeatability. A fixed lead and fixed rule version must always return the same result.

**Exit gate:** users can define an ICP and understand exactly why every evaluated lead matched, failed, or needs more research.

### Phase 5 — Research evidence, intent signals, and explainable scoring

**Goal:** prioritize leads using attributable evidence rather than unsupported conclusions.

**Implementation tasks:**

1. Add research profiles, evidence items, source URLs, observation dates, confidence, signal types, signal freshness rules, and score breakdowns.
2. Build manual research forms for company facts, roles, technologies, hiring, expansion, recent activity, potential need, and notes.
3. Require each signal to reference an evidence item or be explicitly marked as an internal/manual observation.
4. Add configurable score components for ICP fit, positive intent, data completeness, freshness, and exclusion penalties.
5. Implement scoring as a deterministic service with versioned weights and thresholds.
6. Queue a local rescore when relevant evidence, ICP results, or score settings change.
7. Show current score, qualification band, component breakdown, supporting evidence, missing information, and previous score history.
8. Add research and score filters to the Lead Inbox.

**Deliverables:** research workspace, evidence store, signals, versioned rubric, score service, score history, and research queue.

**Verification:** tests cover evidence attribution, expired signals, conflicting evidence, missing fields, penalties, threshold boundaries, rescoring, and unchanged-input reproducibility.

**Exit gate:** a salesperson can inspect any priority score and trace every point or penalty back to a rule or dated evidence item.

### Phase 6 — Additional permitted connectors and local job processing

**Goal:** add data sources without coupling them to CRM internals or requiring distributed infrastructure.

**Implementation tasks:**

1. Add a database-backed local job table with states such as queued, running, completed, failed, cancelled, and retry scheduled.
2. Implement in-process polling, bounded retries, exponential delay, idempotency keys, cancellation, and stale-job recovery on startup.
3. Limit concurrency to protect the local machine and respect provider rate limits.
4. Add connector configuration screens that reference environment-variable secret names rather than storing raw credentials.
5. Implement one approved source at a time using its official API or authorized export, mapping its data into the existing normalized connector payload.
6. Store connector attribution, run statistics, rate-limit state, errors, last successful sync, and safe raw-response references where retention is lawful.
7. Route all incoming records through the same validation and duplicate-review process used by CSV.
8. Add pause/resume and manual-sync controls; avoid background work when the app is not running.

**Deliverables:** resilient local job runner, connector settings, connector-run history, and the first approved non-CSV connector.

**Verification:** simulate timeouts, partial provider failures, rate limits, duplicate deliveries, application restart during a job, malformed payloads, cancelled jobs, and expired credentials.

**Exit gate:** a connector can be rerun safely, interrupted work recovers after restart, and it cannot bypass CRM validation, attribution, or duplicate review.

### Phase 7 — AI-assisted research and sales review

**Goal:** use AI for structured recommendations while keeping evidence and human approval central.

**Implementation tasks:**

1. Define narrow AI tasks: summarize a company, extract candidate facts, identify missing research, explain potential need, classify a reply, draft outreach, and recommend a next action.
2. Create versioned prompt templates with strict structured-output schemas and maximum input/output sizes.
3. Send only the minimum required lead data and evidence to the configured provider.
4. Validate AI output, reject unsupported fields, and label unverified extracted facts as candidates requiring review.
5. Store provider/model identifier, prompt version, evidence IDs, response, token/cost metadata when available, errors, and user decision.
6. Build a review queue with accept, edit, and reject actions. Accepted facts become normal evidence with AI provenance.
7. Keep message drafts separate from sent messages and block external send actions in this phase.
8. Provide a disabled/no-key state so all non-AI CRM features remain fully usable offline.

**Deliverables:** AI adapter boundary, structured tasks, review queue, provenance records, draft generator, and offline fallback behavior.

**Verification:** test schema-invalid output, hallucinated references, provider timeout, missing key, oversized inputs, prompt-version history, rejected suggestions, and accidental attempts to execute external actions.

**Exit gate:** no AI output changes CRM facts or contacts a prospect without an explicit user review, and every accepted result retains its input evidence and provenance.

### Phase 8 — Local workflow engine

**Goal:** automate safe internal steps through versioned, inspectable rules.

**Implementation tasks:**

1. Define a workflow format with triggers, conditions, filters, branches, updates, task creation, waits, in-app notifications, AI draft actions, and webhooks.
2. Build a form-based workflow editor first, including validation for missing branches, invalid references, and loops.
3. Version workflow definitions; a running instance continues on the version on which it started.
4. Implement trigger evaluation, step execution, persisted context, wait scheduling, retries, cancellation, and resume-after-restart using the local job runner.
5. Add idempotency for actions so retrying a step cannot create duplicate tasks or repeated state changes.
6. Add dry-run mode against sample leads and display which path and actions would execute.
7. Add workflow-run history with inputs, evaluated conditions, actions, outputs, errors, and user overrides.
8. Add the visual workflow canvas only after the persisted workflow format and executor are stable.

**Deliverables:** versioned workflow schema, rule editor, executor, dry run, wait/resume, logs, and optional visual canvas.

**Verification:** cover yes/no branches, nested conditions, wait recovery after restart, retry idempotency, changed workflow versions, disabled workflows, deleted/archived leads, and failure at every action type.

**Exit gate:** users can automate internal lead updates and task creation, preview the behavior first, and inspect or safely retry every failed run.

### Phase 9 — Communication model and controlled channel integration

**Goal:** add outbound communication only after consent, auditing, and workflow safety are available.

**Implementation tasks:**

1. Add conversations, inbound/outbound messages, participants, channel accounts, delivery state, consent state, unsubscribe/suppression state, templates, and attachments metadata.
2. Begin with a local test channel that records messages without sending them externally.
3. Build message composition, draft approval, conversation timeline, and manual reply logging.
4. Enforce suppression, consent, allowed sending windows, per-channel limits, and duplicate-send idempotency before adapter invocation.
5. Add one approved provider adapter and its inbound webhook handling, signature verification, provider IDs, delivery receipts, and error mapping.
6. Permit workflows to create drafts first. Enable automatic sending only as a separately configured action after operational review.
7. Add a global pause switch and per-channel pause control.
8. Make failed, bounced, unsubscribed, and replied states visible in both the conversation and lead activity timeline.

**Deliverables:** common messaging model, local test channel, draft review, consent/suppression controls, and one reviewed live-channel adapter.

**Verification:** test unsubscribed recipients, missing consent, duplicate workflow execution, provider failure, webhook replay, invalid signatures, bounced messages, global pause, and audit reconstruction.

**Exit gate:** no external message can bypass consent/suppression and review policy, and every attempted send has a complete, deduplicated audit trail.

### Phase 10 — Analytics, hardening, and daily-operation readiness

**Goal:** make the local platform dependable for ongoing use and measurable improvement.

**Implementation tasks:**

1. Build dashboards for source quality, import rejection/duplicate rate, data completeness, ICP distribution, qualification bands, stage conversion, stage aging, tasks, workflow outcomes, and communication outcomes.
2. Make metric definitions explicit and link dashboard totals back to the underlying filtered records.
3. Add saved reports and CSV export without introducing a separate analytics database initially.
4. Add database maintenance, backup rotation guidance, restore verification, export of essential data, and local storage usage reporting.
5. Test migrations against a realistic database copy before applying them to the working file.
6. Review query plans and add indexes for proven slow queries rather than speculative optimization.
7. Add an audit viewer, structured error screen, failed-job recovery tools, and a diagnostics export that excludes secrets and unnecessary personal data.
8. Complete accessibility, keyboard navigation, empty/error states, retention/deletion flows, security review, and operating documentation.

**Deliverables:** analytics dashboards, operating handbook, verified backup/restore process, diagnostics, performance fixes, and release checklist.

**Verification:** restore a backup into a clean local environment, upgrade across migrations, process a realistic data volume, inspect slow queries, verify deletion/retention behavior, and complete a full lead-to-outcome scenario.

**Exit gate:** the team can operate, back up, restore, diagnose, and update the application locally without developer intervention for routine tasks.

### Work required in every phase

Each phase must also include:

- A forward database migration and a tested upgrade path when the schema changes.
- Domain-level validation, authorization assumptions for the single-user/local model, and an audit event for important mutations.
- Unit tests for business rules and integration tests for database/service boundaries.
- User-visible empty, loading, success, and failure states.
- Documentation for configuration or operating behavior introduced by the phase.
- A check that logs, exports, errors, and Git status do not expose secrets or unintended personal data.
- A manual end-to-end acceptance run using realistic but non-sensitive test data.

### Phase dependency map

```text
Phase 0: Foundation
    |
Phase 1: CRM domain
    |
Phase 2: CRM UI
    |
Phase 3: CSV connector
    |
    +----> Phase 4: ICP rules ----+
    |                            |
    +----> Phase 5: Research ----+----> Phase 7: AI review
    |                  |         |
    +----> Phase 6: Connectors   +----> Phase 8: Workflows
                                             |
                                      Phase 9: Communications
                                             |
                                      Phase 10: Hardening
```

Phase 6 can start after the CSV connector establishes the common connector contract. Phase 4 and the manual parts of Phase 5 can proceed independently after the CRM is stable. AI, workflows, and communication remain later phases because they depend on reliable data, traceable evidence, and safe execution controls.

## Quality, privacy, and compliance guardrails

- Collect only data the business is entitled to collect and use. Preserve the source and collection date on research items.
- Minimize personally identifiable information, keep it local, and support deletion/correction requests.
- Protect connector/API credentials with local environment variables; do not persist them in database records, logs, CSV exports, or Git.
- Do not circumvent login walls, anti-bot systems, or platform restrictions.
- Make every score, AI suggestion, imported record, and workflow action traceable to inputs and the actor/system that made the change.
- Require a human review gate for external messages and irreversible bulk changes.

## Milestone checkpoints

| Milestone | Outcome | Depends on |
| --- | --- | --- |
| M1: CRM Inbox | Local leads can be imported, managed, and exported. | None |
| M2: ICP | Leads receive transparent fit evaluations. | M1 |
| M3: Research + signals | Users attach evidence and intent signals. | M1 |
| M4: Scoring | Priority is reproducible and explainable. | M2, M3 |
| M5: Connectors | Permitted sources feed the same local inbox. | M1 |
| M6: AI review | AI produces reviewed, auditable recommendations. | M2–M4 |
| M7: Workflows | Local rules automate safe internal actions. | M1, M4 |
| M8: Communications | Approved channels execute reviewed outreach. | M7 |
| M9: Analytics | Team can improve sources, ICP, and conversion. | M1 onward |

## First implementation decision

Start with **Module 1: Local Lead Inbox and CRM Foundation**, with **CSV import as its first connector**. It gives the team a usable product, establishes clean data and audit trails, and turns every later module into an additive feature instead of a rewrite. Do not begin with social or search connectors: without deduplication, a canonical CRM model, and a reviewable import trail, those sources will mainly create untrustworthy volume.

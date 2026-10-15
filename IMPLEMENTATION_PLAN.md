# RecallGuard implementation plan

Status: planned; application implementation has not started.
Window: October 15–18, 2026 (IST). Target Tracks 02 and 01; Track 03 is optional.

## Product and success criteria

RecallGuard is an incident-response agent team whose shared memory records what worked, what failed, and the conditions under which a lesson applies. FalkorDB is the primary database and source of persistent investigation context. A controlled simulator supplies observable service states and action outcomes.

The core demonstration must show a fresh agent continuing an interrupted investigation, avoiding a previously failed action under matching conditions, using graph evidence to choose another action, verifying recovery, and refusing to treat an unvalidated lesson as current after configuration changes.

Implement in vertical slices: plan the behaviour and acceptance criteria, build the smallest complete path, test against those criteria, then proceed. Mock screens and deterministic tool tests are development aids; the final demonstration uses a real model and real FalkorDB. Clearly label simulated infrastructure and sample company data.

## Requirements mapped to implementation

| Requirement | Build | Proof |
| --- | --- | --- |
| Episodic memory | Incidents, observations, attempts and outcomes | Retrieve an earlier unsuccessful attempt with its evidence |
| Semantic memory | Services, dependencies, owners and configuration facts with provenance | Resolve the affected dependency and current configuration |
| Procedural memory | Recovery procedures, prerequisites, steps and observed outcomes | Select a procedure only when its conditions apply |
| Persistent conversation history | Ordered messages linked to sessions and investigations | Reload a session after backend restart |
| Shared state and handoffs | Durable investigation state, pending tasks and role handoff events | A new session resumes without the prior in-memory transcript |
| Concurrent access | Append-only events, unique operation IDs, per-investigation revision checks and serialized mutations | Concurrent workers do not lose observations or execute duplicate actions |
| Separate tenant memory | Server-resolved graph per tenant, with membership checks on all requests and tools | Cross-tenant reads, writes and guessed IDs fail |
| Connected-data actions | Dependency and precedent queries feed a bounded remediation tool | Changing a relevant graph relationship changes the evidence and eligible action |
| Explainability | Source IDs, graph paths, eligibility reasons and execution outcomes | Every recommendation opens its supporting evidence |

Concurrency testing demonstrates a measured workload, not an unsupported claim of production-scale throughput.

## Architecture

- Frontend: React, TypeScript, Vite, Tailwind CSS and React Flow.
- Backend: Python, FastAPI, Pydantic validation and streaming server-sent events.
- Database: FalkorDB, official Python client and parameterized Cypher. Bind database access to the backend; never expose credentials to the browser.
- Model: one tool-calling provider behind a small adapter; select from available API access before integration. Keep credentials server-side.
- Coordination: explicit Python state machine with investigator and resolver roles. Memory curation begins as validated backend logic, with optional model-assisted contradiction proposals.
- Simulator: deterministic service behaviours, injectable faults, action receipts and health probes. Outcomes come from simulator state, not model assertions.
- Packaging: Docker Compose, pinned dependencies/image version, persistent FalkorDB volume and documented persistence settings. Verify data survives database restart.
- Tests: pytest for policy/simulator/integration checks; Playwright for critical browser journeys.

Flow: UI → API → agent tool loop → FalkorDB retrieval → action eligibility check → simulator execution → health verification → FalkorDB outcome/memory update → UI event.

Use direct graph queries first. MCP, Graphiti, LangChain and the other pictured integrations are options, not established mandatory dependencies. No second memory database is needed for this MVP.

## Graph and memory design

Nodes: Service, Configuration, Incident, Observation, Evidence, Attempt, ActionReceipt, Procedure, Lesson, Investigation, Task, AgentRun, Session, Message. Optional company context: Person, Project, Document, Decision.

Relationships: DEPENDS_ON, AFFECTS, HAS_EVIDENCE, OBSERVED_IN, ATTEMPTED, PRODUCED, SUPPORTS, CONTRADICTS, APPLIES_TO, SUPERSEDES, HAS_TASK, ASSIGNED_TO, HAS_SESSION and HAS_MESSAGE.

Give records stable IDs, timestamps, source references and investigation identifiers. Keep attempts and outcomes immutable; retain historical lessons. Distinguish observed facts from model-generated hypotheses.

Lesson states: proposed, active, disputed, needs_revalidation and superseded. A version/configuration change can invalidate applicability but does not prove the opposite lesson. A newer statement alone cannot supersede verified evidence. Failed outcomes remain retrievable as warnings; they do not authorize repeating the same action under unchanged conditions.

Core graph operations:
1. Traverse dependencies from an affected service with bounded depth and cycle handling.
2. Retrieve incidents and procedures matching service identity, configuration and evidence.
3. Retrieve unsuccessful attempts and unfinished tasks for a handoff.
4. Fetch eligible lessons and explain exclusions with source paths.
5. Append observations, execution receipts and outcomes idempotently.
6. Identify lessons whose assumptions are affected by a configuration change.

Agent tools: inspect_service, trace_dependencies, retrieve_lessons, record_observation, propose_action, execute_remediation, verify_recovery and handoff_investigation. Validate inputs; allowlist actions; bound retries, tool calls and execution time. Recheck action eligibility immediately before execution.

Start with a single mutation worker per investigation while allowing concurrent reads and independent investigations. Verify FalkorDB's supported constraint/atomic-update behaviour during the foundation spike before finalizing locking. Use durable action IDs and simulator receipts to recover from a crash after execution but before outcome recording; do not promise exactly-once execution without proof.

## Frontend design plan

Apply `.agents/skills/frontend-design/SKILL.md` for planning, critique, implementation and screenshot review.

Audience: a developer handling an incident and a judge inspecting the evidence. Visual concept: an incident investigation desk. Open directly on a runnable scenario and the investigation, with the evidence path as the distinctive visual element.

Tokens: canvas #F3F6FA, surface #FFFFFF, ink #172B46, action #2457C5, warning #9A5600, resolved #16714A. Use text/icons as well as colour for status, and verify contrast. Typography: IBM Plex Sans for content and headings; IBM Plex Mono only for code, timestamps and identifiers. Self-host where practical.

Desktop layout:

```text
Workspace / incident                   Service health / run controls
Incident list | Investigation timeline | Evidence and memory inspector
              | Current proposed action | Relevant relationship path
```

Left-align operational content. Give the timeline most space. On narrow screens, use accessible tabs for investigation and evidence; provide a textual alternative to the graph.

Screens: scenario selection, live investigation, memory explorer, and run comparison. The evidence inspector shows why a lesson was used or excluded. Buttons name actions: Start investigation, Continue with resolver, Inspect evidence, Apply configuration change.

Render loading, empty, interrupted, model-unavailable, database-unavailable and action-failed states. Preserve keyboard focus, support reduced motion and avoid status conveyed only by animation. Animate only meaningful changes such as a newly recorded outcome. No fabricated progress or success states.

Before coding, review the layout against the brief and remove generic dashboard decoration. During implementation, inspect desktop and mobile screenshots and correct clipping, contrast and hierarchy.

## Plan → build → test milestones

### 1. Foundation and persistence — October 15

Plan: define simulator faults, graph schema, tenant boundaries and API contracts. Confirm model access and local Docker availability.

Build: backend/frontend skeleton, FalkorDB connection, Compose persistence, two isolated sample tenants and seed/reset commands scoped to demo data. Add an end-to-end observation write/read path.

Test gate: backend and database restart preserve records; seed is repeatable; tenant separation works; parameterized queries work against the pinned database version. Read-only inspection reveals no secret in frontend assets.

### 2. Real investigation and action — October 15–16

Plan: one payment incident, bounded tools and measurable recovery criteria.

Build: investigator agent, dependency traversal, evidence display, action eligibility, simulator remediation, receipts and health verification. Store messages, attempts and outcomes in FalkorDB.

Test gate: real model invokes tools; failure and success reflect simulator state; irrelevant/invalid action requests are rejected; repeated action IDs do not repeat side effects. If FalkorDB is unavailable, explain the failure rather than silently substituting canned memory.

### 3. Memory and handoff — October 16

Plan: define durable state transitions and the minimum context a new role must recover.

Build: resolver agent, unfinished-task retrieval, applicability-aware lessons and restart/resume controls.

Test gate: terminate the first run; a fresh session recovers from FalkorDB and does not repeat the same failed attempt under matching conditions. Messages remain ordered. Unrelated customer data is absent from retrieved context.

### 4. Changing evidence and concurrency — October 17

Plan: specify version-change and contradictory-evidence cases before implementation.

Build: disputed/revalidation/superseded states, memory inspector, bounded concurrent access and conflict handling.

Test gate: a version change causes revalidation, not automatic approval; unsupported claims cannot replace verified evidence; a stale proposal is rechecked before execution. Run 10 concurrent readers and 5 writers against a declared dataset and report errors, latency, lost events and duplicate attempts. Test competing handoffs and crash recovery around action receipts.

### 5. Product polish, evaluation and submission — October 17–18

Plan: freeze scope around the core two-track workflow; reserve October 18 for reliability and delivery.

Build: complete responsive UI, evidence paths, run comparison, setup documentation, required Railway deployment, and demonstration script/video. Prepare deployment files during foundation work and target the first Railway deployment on October 17.

Test gate: browser journey from scenario start through recovery passes; narrow-screen and keyboard flows work; production build succeeds; clean setup instructions reproduce the demo. Record a real-model run, and clearly distinguish fixtures used for automated tests.

Target submission by 9:00 PM IST on October 18, leaving a buffer before the stated 11:59 PM IST deadline. Check the submission portal's exact required fields before packaging.

## Required Railway deployment

Deploy two services in one Railway project/environment:

1. RecallGuard application: a multi-stage Dockerfile compiles the React frontend and packages the static assets with FastAPI. Serve UI, API and server-sent events from one public HTTPS origin. Include the controlled simulator in the backend; start with one application instance and one worker process to match the coordination design.
2. FalkorDB: a pinned official server image with a persistent Railway volume mounted at the image's configured data directory. Explicitly configure persistence and verify the effective data path. Connect over Railway private networking with authentication; expose no public database endpoint.

Keep Docker Compose for local development. Commit the application Dockerfile, Railway configuration, environment-variable documentation and repeatable deployment instructions. Bind the web server to `0.0.0.0` and Railway's `PORT`; configure `/health/ready` with a bounded database readiness check. Handle temporary database unavailability with bounded retries and visible errors.

Store model API keys, database credentials and session secrets in Railway variables. Keep secrets out of source control, frontend bundles and logs. Resolve tenant membership on the server. Railway account/project access and model API access are prerequisites; configure credentials through the platform rather than pasting them into chat.

Deployment sequence: pass local gates → provision database and volume → deploy application → configure private connection and secrets → run explicit idempotent demo seeding → generate public domain → run deployed acceptance checks. Never reset or reseed investigations on ordinary startup.

Persist simulator state and action receipts alongside investigation records in FalkorDB so redeploying the backend cannot erase the evidence of an executed action. On startup, reconcile interrupted runs and offer explicit resume; do not replay side effects blindly.

Live acceptance gate:
- Complete the real-model investigation, handoff, remediation and recovery journey through the public URL.
- Verify streamed updates and reconnect behaviour after browser disconnection.
- Redeploy the application and confirm saved messages, lessons, simulator state and pending work survive.
- Restart FalkorDB and confirm persistence independently of the application process.
- Confirm duplicate action IDs remain idempotent across restarts.
- Verify tenant isolation through public API requests and confirm the database has no public endpoint.
- Exercise unavailable-model and unavailable-database states without fabricated success.

Record the tested deployment revision, live URL and results in the delivery documentation. Keep the prior working application revision available for rollback; avoid destructive schema changes during the submission window. Railway deployment is required scope, not a stretch goal; report missing account access or credentials as a deployment blocker while continuing local work.

## Evaluation and demonstration

Prepare three scenarios: (A) same conditions after a failed fix and handoff, (B) same symptom with a different underlying dependency fault, and (C) changed configuration invalidating a prior lesson's applicability.

Compare the same model and tool budget with graph-memory retrieval enabled versus disabled. Retain current observations and dependency tools in both variants to isolate the effect of memory. Record recovery success, repeated unsuccessful actions, recovered pending tasks, unsupported recommendations, latency and tool calls. Use multiple runs when practical; disclose the small synthetic dataset and variability. Do not invent scores or force the baseline to fail.

Demo: trigger incident → investigate and record an outcome → stop agent → resume in fresh session → inspect retrieved evidence → execute and verify recovery → change configuration → show memory revalidation.

## Scope and deliverables

Must ship: real FalkorDB persistence, two agent roles, three scenarios, shared conversation/investigation history, evidence-backed memory, validated simulator actions, tenant isolation, concurrency checks, usable frontend, tested Railway deployment and public demo URL, README, architecture explanation and demo video.

Optional after core gates pass: company owners/runbooks/project impact for Track 03, MCP adapter, external ticket integration. Do not add these at the expense of restart, handoff or action verification.

Deliverables: runnable repository, `.env.example` without secrets, Compose setup, Dockerfile and Railway configuration, deployment instructions and live URL, seed data, test/evaluation commands, honest local and deployed result report, frontend screenshots, demo recording and a track-to-feature mapping. Model access and hosting credentials are external prerequisites; local development and simulator tests can proceed while they are resolved.

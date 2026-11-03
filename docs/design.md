# Slop Machine — Design Document

Status: requirements and communication-contract drafts. Module interfaces and deployment architecture are not finalized. Implementation has not started.

## 1. Purpose

Slop Machine coordinates coding work from a ticket backlog or a user prompt. It manages task execution, implementation, senior engineering review, and feedback so the user does not have to set up and coordinate agents manually.

This draft records functional requirements, non-functional requirements, technology constraints, and usability proposals. LangGraph and Ponytail are selected requirements; Caveman is an optional candidate. Drift detection and containment are required, with NeMo Guardrails the preferred framework pending integration design. The first-release single-host, per-run-container deployment is selected; detailed infrastructure design and implementation remain open.

## 2. Terminology

- **Task:** one bounded piece of work managed by the system. A task can originate from a ticket or a user prompt.
- **Ticket:** the backlog record used to describe and coordinate the work. Jira is the first supported ticket system to target.
- **Task run:** an execution attempt for a task, containing implementation, review, and any repair iterations.
- **Agent session:** the isolated conversation and context belonging to one agent role within a task run.
- **Implementer–reviewer loop:** the collaboration pattern in which an implementer produces changes, a separate reviewer evaluates them, and actionable feedback returns to the implementer until the work passes review or execution stops.

The broader pattern is also called a generator–critic or producer–reviewer pattern. This document uses **implementer–reviewer loop**, because the reviewer has a specific senior engineering responsibility: checking the design, requirements, and resulting code. Separate sessions provide independent context, but do not guarantee independent judgment or correctness.

## 3. Functional requirements

The requirements below distinguish explicit user requirements from behavior retained from the initial project brief. Acceptance criteria describe observable outcomes to verify when implementation begins.

### FR-01 — Retrieve and coordinate ticket work

Source: explicit requirement; Jira-first selection from project setup.

The system must retrieve tickets from a configured Jira backlog and coordinate actual work through those tickets. Ticket access must support reading existing tickets and creating new tickets when needed. Tickets must remain connected to the corresponding task runs, progress, blockers, and outcomes.

Acceptance criteria:

- The user can configure the backlog from which work is selected.
- A selected ticket provides the task's requirements and acceptance criteria, where present.
- The system records which ticket a task run belongs to.
- The user can see in the ticket that work has started, whether it is blocked, and its resulting outcome.
- Newly created tickets are linked to the originating task or request where applicable.
- A failure to update Jira is visible; the system does not report that coordination succeeded when the update failed.

FR-09 defines preflight ticket corrections and closure; FR-10 defines dependency-ticket creation. To decide: selection rules, how work is claimed, other creation triggers, permitted Jira transitions, and the appropriate frequency and detail of ticket updates. Review approval or PR creation must not implicitly mean that a Jira ticket is Done. FR-08 requires a verified update to the configured target branch before the Done transition.

### FR-02 — Accept a user prompt as an entry point

Source: explicit requirement.

The system must optionally accept a user prompt to start work, without requiring the user to write a ticket first. Prompt-originated tasks must use the same implementation and review process as ticket-originated tasks.

Acceptance criteria:

- The user can supply a work request and identify its target project.
- The system can request missing information needed to make the request actionable.
- The resulting task preserves the original request and any clarifications.
- Prompt-originated tasks count toward the same parallel task limit as ticket-originated tasks.

To decide: whether a prompt must be converted into a Jira ticket before execution, and whether the user reviews the proposed ticket before it is created. The prompt entry point must not silently establish a second, disconnected work-tracking process.

### FR-03 — Configure up to three parallel tasks

Source: explicit requirement.

The system must allow a configurable parallel task limit of one, two, or three. It must never execute more than three tasks at once. Additional tasks must wait for capacity.

Acceptance criteria:

- The user can set the parallel task limit to 1, 2, or 3.
- Values outside that range are rejected with a clear explanation.
- With more eligible tasks than available capacity, only the configured number execute concurrently.
- One task's implementation and review iterations remain part of that task rather than consuming additional task slots.
- The user can distinguish tasks that are running from tasks waiting for capacity.

This limit applies to tasks, not to the total number of agent sessions created over time. Whether implementation and review can execute simultaneously within a task is a later scheduling decision. The limit is not a guarantee that an account will avoid usage limits.

Dependency-blocked runs halt and release their execution slot after their agents stop (FR-10); they must not occupy a slot while waiting for another ticket. To decide: the initial default, capacity handling for other blocked states, and how changing the limit affects tasks already running.

### FR-04 — Assign an implementer and a senior reviewer to every task

Source: explicit requirement.

Each task run that proceeds to implementation must have an implementer subagent and a separate senior reviewer subagent. Preflight may end a run without implementation when the ticket is unnecessary or unresolved (FR-09); it need not start coding sessions solely to close a ticket.

The implementer must inspect the task and applicable project guidance, implement the requested change, and address reviewer feedback. The senior reviewer must examine the relevant design document, the ticket or approved task requirements, and the actual code changes. Review must evaluate alignment with the design and requirements as well as code correctness and available validation evidence.

Acceptance criteria:

- Both roles receive the same task identity and references to the applicable requirements and design.
- The reviewer inspects the actual changes rather than relying only on the implementer's summary.
- Review findings describe the problem, supporting evidence, and the change or clarification needed.
- Findings are returned to the implementer, and repaired changes undergo another review.
- Approval refers to the version of the changes that was reviewed; later edits require renewed review.
- The user can see unresolved findings and the reason work stopped.
- Missing or contradictory requirements or design guidance are surfaced rather than treated as having passed review.

To decide: how to identify the authoritative design document, what to do when a project has no design document, review outcome structure, required checks, and retry or escalation limits. The agents need the target project's design guidance; this system's own design document is not a substitute for it.

### FR-05 — Start tasks with fresh sessions and configure reuse per role

Source: explicit requirement.

Every new task run must create fresh implementer and senior reviewer sessions. Sessions from previous tasks must not be reused. Each role must receive the relevant task context explicitly, so unrelated conversation history does not accumulate across tasks.

Within a task run, the user must be able to configure session behavior independently for the implementer and the senior reviewer. Each role must support:

- **Reuse within the task:** continue that role's session across its successive implementation or review passes.
- **Fresh for every pass:** start a new session each time that role performs an implementation, repair, or review pass, including after test failures or reviewer feedback.

These choices must support all four combinations: both roles reuse sessions, both start fresh each pass, or either role starts fresh while the other reuses its session. Neither setting permits reuse across task runs.

Acceptance criteria:

- A new task run creates distinct sessions for the two roles.
- Neither session contains the conversation history of a previous task.
- The user can select and inspect each role's session policy independently.
- With reuse enabled for a role, successive passes use that role's existing session within the task run.
- With fresh-per-pass enabled for a role, successive passes use distinct sessions without inheriting the previous session's conversation history.
- Changing one role's policy does not change the other role's policy.
- Required context is supplied through the ticket or prompt, applicable project instructions and design, code, and relevant evidence.
- A review or repair handoff contains enough information for the receiving agent to act without access to the other role's entire conversation.
- Fresh sessions receive the current work state, relevant validation results, and unresolved review findings explicitly, so session renewal does not discard task progress.

To decide: the default policy for each role, configuration scope and whether changes apply to active runs, context budgets, and how a policy change is handled during restart recovery. The restart path uses fresh sessions with explicit durable context when an interrupted run can safely resume. Fresh sessions alone do not guarantee that a large task will fit within the available context.

### FR-06 — Support API credentials and existing OpenAI subscription access

Source: explicit requirement.

The system must support two selectable access modes: API-token-based access and access using an existing eligible OpenAI subscription through supported tooling. The task workflow and agent responsibilities must remain the same regardless of access mode.

Acceptance criteria:

- The user can identify and select the access mode before starting work.
- The system can determine whether the selected mode is ready to use and explain any required setup.
- Missing authentication, unavailable account access, or usage limits produce a visible, actionable state.
- The system does not silently switch from subscription access to billable API access.
- Authentication details are not included in tickets, agent handoffs, or version-controlled configuration.
- Every task container fetches the current credential for its selected access mode from the credential manager when it starts, including a replacement container used for recovery. A missing, expired, or unusable credential blocks agent work and the Jira work-start update with an actionable reason.
- The container receives only a run-bound read identity and the access credential needed for its selected mode. It does not restore token bytes from a workflow checkpoint or read the central renewable sign-in credential.

A separate execution/authentication integration module is a candidate design boundary, not yet a selected implementation. Supported mechanisms and account limitations must be verified when that interface is designed. A task container does not inherit the user's sign-in. The [functional architecture proposal](functional-architecture.md#subscription-authentication-gate) uses a self-hosted OpenBao pull path for the current API credential or subscription access token behind a replaceable store adapter. The central access provider, rather than the task container, owns subscription renewal and publication. An official ChatGPT-plan OAuth/Codex app-server integration is a candidate for supplying the worker-usable access token. This does not assume token compatibility, deployment eligibility, unlimited subscription usage, or copying credential caches between machines, and it never silently converts subscription work to API-key billing.

### FR-07 — Work across configured project repositories

Source: initial project brief.

The system must support different target projects, with a clear mapping between the selected work and the repository in which it should be performed.

Acceptance criteria:

- Each task resolves to an explicitly configured target repository.
- The user can see the target project before execution.
- After a task run is claimed, the system creates and verifies a new Docker volume containing a full repository clone on a task-owned branch before marking work started or beginning preflight. This also applies to runs that preflight later closes without code work.
- Concurrent tasks do not share a mutable working directory.
- Agents receive the volume and clone selected by the system; they do not choose a branch or isolation mechanism. A later new run for the same task gets a new volume after the previous run is safely closed or reconciled.
- A new project can be configured without changing the system's application code.

A separate full clone in a named Docker volume is the selected isolation mechanism. Startup and run-end volume operations belong to the coordinator/repository adapter, not to either agent role. The system must not delete a volume with unpreserved task changes. The exact volume naming, branch naming, retention and backup policy remain design decisions.

### FR-08 — Configure publication, optional human review, and completion

Source: initial project brief and subsequent explicit delivery-mode clarification.

After the implementer and senior reviewer agents approve an exact code snapshot, the system must follow an explicitly configured delivery mode:

| Mode | Required path |
| --- | --- |
| `pr_human` | Push a dedicated branch, create/update a PR, and request human review through the PR or task chat. Human approval permits merge after current checks and policy; actionable denial returns to implementation and another senior and human review. Final rejection stops for attention. |
| `pr_automatic` | Push a dedicated branch and create/update a PR, then merge after current checks, agent review, and policy. No human-review wait is imposed. PR-host rules may still require human approval; if so, this mode cannot silently bypass them. |
| `direct_main` | Integrate and push the agent-reviewed code directly to the configured main branch without a PR or human review, subject to explicit permission, current checks, policy, and an expected-head fast-forward guard. Never force push or silently fall back to another mode. |

The human-review choice is therefore optional, while implementer/senior-agent review remains required in every mode. The delivery mode is recorded in the task specification before publication and cannot be switched mid-run to evade a failed gate.

In `pr_human`, waiting ends the active run and releases its agent sessions and execution slot. A response starts a new run that rechecks the task, ticket when present, code, PR head, checks, and approval before acting. For either PR mode, the system merges under configured permission or verifies an external merge. In `direct_main`, concurrent integrations are serialized and a changed main head requires the final snapshot to be revalidated and reviewed before another push attempt. For Jira-backed work, the system transitions the ticket to Done only after the target branch update is verified, recording the published commit and PR when one exists. A prompt-only task without a Jira ticket records an equivalent internal completion outcome.

Acceptance criteria:

- The selected delivery mode and its permissions are visible before publication; the published code matches a snapshot approved by the senior reviewer agent and required checks.
- In `pr_human`, the review record identifies the reviewer, source channel, PR, reviewed commit, decision, and feedback. Chat approval cannot bypass PR-host approval rules; silence is never approval.
- In-scope human feedback returns to the implementer, and changed code receives fresh checks and senior review before another human review.
- In `direct_main`, no PR is created and the remote main head is verified after a non-force, fast-forward push. A competing update cannot be overwritten; the final changed snapshot is rechecked and re-reviewed.
- Jira-backed tasks enter Done only after the configured target branch update and ticket update have both been confirmed; prompt-only tasks verify their internal completion record.
- A partial failure after merge or direct push leaves an auditable recovery state; retries reconcile remote code and Jira status without duplicating effects.

The blocked-work branch push in FR-10 is a preservation operation, not successful delivery. It does not require falsely declaring incomplete work approved or opening a ready-for-review PR.

To decide: the default delivery mode, repository hosting support, publication checks, who performs PR merges, direct-main permission policy, human-review timeout/escalation, and whether task completion stops backlog intake or allows the scheduler to start another task. Task completion and queue continuation are separate choices.

### FR-09 — Reconcile tickets with the design and current code before implementation

Source: explicit requirement.

After system-owned access, repository and volume/clone startup, every ticket pickup must begin with a preflight assessment of the ticket, the applicable design, and the current target repository state before any agent code work. This includes later pickups of previously blocked tickets. The system must distinguish work that is still needed from an incorrect ticket or work that is already satisfied or no longer needed.

| Outcome | Required behavior |
| --- | --- |
| Valid and needed | Record the evidence and establish the task's scope before implementation. |
| Incorrect or stale, but still needed | Update the ticket with current facts, explain the correction, and reassess the corrected ticket before proceeding. |
| Unnecessary or already satisfied | Record why no work is needed, mark the ticket accordingly, close it, and end the run without implementation. |
| Existing delivery awaiting review or completion | Reconcile the PR or direct-push result and published commit, then resume the review, completion, or recovery path; do not classify the ticket as unnecessary. |
| Uncertain or conflicting evidence | Record the uncertainty and halt for clarification; do not guess, rewrite intent, or close the ticket as unnecessary. |

Acceptance criteria:

- The assessment records the ticket revision, applicable design version, target branch and commit, and supporting code or validation evidence. Current code means a refreshed view of the configured target branch, not an unmerged working branch or stale local checkout.
- Corrections preserve the original intent and history, identify the outdated statements, and explain the replacement information. A correction cannot disguise scope expansion; genuinely additional work follows FR-10.
- The system does not silently change the design to make the ticket fit. Unresolved disagreement between the design, ticket, and code is surfaced.
- An unnecessary-ticket decision addresses the ticket's acceptance criteria with evidence. Inability to reproduce a bug or inability to find code is not by itself proof that the work is unnecessary.
- The explanation is saved before closure. The configured Jira resolution distinguishes unnecessary/already-satisfied work from work delivered by this run.
- Before updating or closing a ticket, the system checks for intervening ticket changes and reassesses conflicts rather than overwriting another person's edits. Material changes to the assessed design or repository evidence also invalidate the assessment.
- Failed updates or closure attempts are visible and are not reported as completed.
- Corrected, unnecessary, and unresolved tickets remain auditable through the task's recorded decision and evidence.

Automatic correction and evidence-backed closure are intended system capabilities. A routine human confirmation is not required by this requirement. To decide: which role performs preflight, whether closure needs a separate reviewer decision, evidence thresholds, freshness checks, and Jira status/resolution mappings.

### FR-10 — Preserve blocked work, create a dependency ticket, and halt

Source: explicit requirement.

If completing a task requires work outside its established scope, the current loop must stop implementation. It may describe a bounded proposed solution in a new ticket but must not implement that expansion in the current run. It must preserve its existing code on its own remote branch, link the new dependency from the current ticket, and halt. Once the prerequisite is resolved, a completely new agent loop can pick up the original ticket.

Required handoff behavior:

1. Stop further implementation and record the scope boundary and why the extra work is necessary.
2. Create a prerequisite ticket explaining the problem, supporting evidence, impact on the original task, and a bounded proposed solution with acceptance criteria. Link it back to the original ticket.
3. Commit the current task's intended work to its dedicated branch and push it to the configured remote. Preserve incomplete work as explicitly incomplete, with its validation status; do not merge it or present it as approved delivery.
4. Update the original ticket with the branch URL, exact commit, prerequisite ticket link, dependency relationship, completed and remaining work, validation results, unresolved findings, and instructions for a future pickup.
5. Mark the original ticket blocked using the configured Jira workflow, end the current run, terminate its agent sessions, and release execution capacity.

Acceptance criteria:

- The prerequisite blocks the original ticket, not the reverse. The current loop proposes the additional work but does not execute it or recursively start a child coding loop for it.
- The branch belongs to this task; publication does not touch the main branch or include unrelated files or credentials. If no code changes exist, record the base commit and absence of changes without inventing an empty work commit.
- The remote push is verified and the ticket links to the exact saved commit, so a later run does not depend on the old machine or old conversation history.
- The handoff records the original scope and any relevant ticket/design revisions, not just an agent-generated summary.
- The scheduler excludes the original ticket while its prerequisite remains unresolved. The prerequisite can enter the normal scheduling process as a separate task, subject to the same preflight and concurrency requirements.
- When the prerequisite is resolved, a fresh run of the original ticket repeats FR-09, verifies that the prerequisite outcome is actually available in the target code state, and reconciles the saved work with the current repository before continuing. A ticket being closed is not by itself proof that its fix is available.
- The new run uses new implementer and reviewer sessions and performs its own validation and review. It may reuse the preserved code, but not stale approvals or the old sessions.
- Ticket creation, branch publication, dependency linking, and status updates are tracked separately. Retries reconcile completed operations and do not create duplicate prerequisite tickets, lose saved work, or falsely report a complete handoff.
- If a handoff operation fails, implementation remains stopped. The system records a handoff-incomplete state and the remaining recovery action; successful handoff is not claimed until all required operations are confirmed.
- A matching existing prerequisite may be linked instead of creating a duplicate, with that decision explained. Self-dependencies and dependency cycles are detected and escalated rather than creating an endless ticket chain.

These are permitted handoff operations after implementation stops; halting does not mean abandoning the ticket before its work is preserved. No human response is required merely to propose the bounded solution and perform this handoff. Broader scope changes still cannot be executed by the current loop.

To decide: branch retention, how a new run incorporates the saved commits, retry limits for handoff recovery, prerequisite selection rules, and dependency-cycle resolution.

## 4. Usability proposals for discussion

These proposals explain the intended experience without choosing a CLI, web interface, or desktop application. They are not additional accepted requirements yet.

### Starting work

1. Select a configured project, including its repository and Jira backlog.
2. Choose a ticket or enter a work request.
3. See the task's scope, target repository, delivery mode, access mode, parallel task limit, and the model and session policy configured for each agent role.
4. Run ticket preflight and show whether the ticket is valid, corrected, unnecessary, or unresolved before implementation begins.

Backlog-driven operation should reuse saved project configuration so the user does not repeat this setup for every ticket. The conditions under which automatic intake starts work still need definition.

### Following progress

Give each task a concise status, such as queued, implementing, agent reviewing, repairing, awaiting human review, blocked, or complete. Show what it is doing, any action needed from the user, and links to its ticket and output. Avoid requiring users to read raw agent transcripts to understand progress.

Keep detailed review findings, agent messages, inter-agent handoffs, tool activity, and usage metrics available for inspection. The user should be able to move from a task's status to the messages and events that explain it. Ticket updates should communicate meaningful progress and decisions rather than reproduce every agent message.

### Handling a question or blocker

Explain what information or action is needed and why. Preserve enough task context for the user to answer without reconstructing the agent conversation. For required work outside scope, show the prerequisite ticket, preserved branch and commit, and the original ticket's blocked state (FR-10). General pause and cancellation behavior remains to be defined; dependency recovery uses a new run with fresh sessions.

### Finishing work

Present the result, relevant checks, outstanding limitations, published commit, PR link and human decision when applicable, target-branch status, and Jira status. Distinguish reviewed work from code verified on the target branch and a Jira ticket confirmed Done so the user can tell what has actually happened.

## 5. Non-functional requirements and system constraints

This section captures the user's non-functional requirements together with explicit technology constraints and configuration capabilities. Acceptance criteria make them verifiable without committing to a module layout or deployment platform.

### NFR-01 — Use LangGraph for workflow state management

Status: required technology constraint.

The system must use LangGraph to represent and manage task workflow state, including preflight, implementation, review, repair, dependency handoff, blocking, closure without implementation, and completion. Task state must remain separate from any individual agent session so that replacing a session does not erase workflow progress.

Acceptance criteria:

- A task run has an identifiable graph state and explicit transitions between workflow stages.
- State records the task identity, current stage, iteration, agent session references, review outcome, and references to relevant evidence.
- The current state and transition history can be inspected through the observability facilities.
- Session policies in FR-05 can change agent sessions without losing the task's requirements or unresolved findings.
- On process restart, the coordinator establishes current ownership and reconciles persisted checkpoints, command intent, volume/clone identity and observed external effects before dispatching affected work.
- An interrupted run resumes only from a verified workflow boundary with its existing volume and clone, and fresh role sessions. Unknown effects are not replayed, and an unfenced old executor cannot share its slot or volume with a new one.
- After host shutdown, startup remains paused while the coordinator reconstructs active tickets and run IDs from PostgreSQL, matches each run to its named Docker volume, and checks Docker metadata, the volume manifest, and the clone's Git state. Task containers must not auto-start before coordinator reconciliation. A matching interrupted run restarts in a replacement container using that same volume and ticket; a missing, duplicate, or mismatched volume blocks automatic resumption without deleting saved work.
- A later pickup after human review, dependency resolution, or a safely ended run creates a new run, volume, and role sessions. It may clone a verified saved remote branch. A host reboot alone does not create a new run for an interrupted task.

The [process-restart recovery diagram](diagrams/process-restart-recovery.puml) defines operation reconciliation, and the separate [host-restart recovery diagram](diagrams/host-restart-recovery.puml) defines shutdown/startup inventory and volume remounting. PostgreSQL is the selected authoritative event journal/outbox and checkpoint backend, on persistent storage with an off-host backup/restore path. The [durable recovery proposal](durable-recovery.md) describes synchronous, JSON-only LangGraph checkpoints and explicit reconciliation across the journal/checkpoint write boundary; its earlier SQLite assumption is superseded. Platform-specific executor fencing remains to be specified. An in-memory checkpointer does not survive a process restart, and checkpointing alone does not prevent duplicate external actions. [LangGraph persistence documentation](https://docs.langchain.com/oss/python/langgraph/persistence)

### NFR-02 — Use Ponytail to improve implementation efficiency

Status: required integration.

The system must integrate [Ponytail](https://github.com/DietrichGebert/ponytail) with the aim of conserving tokens and producing simpler, maintainable code. Ponytail supplies agent instructions that favor understanding the existing code and reusing appropriate solutions while preserving necessary validation. Its published results are evidence to evaluate, not guaranteed savings for this system.

Acceptance criteria:

- The selected Ponytail version and configuration are recorded with each task run and applied consistently to the configured agent roles, including fresh sessions.
- Its instructions do not override task requirements, project guidance, or required review and validation checks.
- Controlled comparisons can run with and without Ponytail to measure total task tokens, completion time, repair iterations, and correctness.
- Quality evaluation includes requirement satisfaction and review findings; fewer lines or tokens alone do not establish better code.

To decide: version, integration mechanism, per-role settings, and measurable adoption targets. Nothing is installed by this requirements document.

### NFR-03 — Evaluate Caveman as an optional further optimization

Status: optional candidate, not an adopted dependency.

Working interpretation: the user's “Cavement” refers to [Caveman](https://github.com/JuliusBrussee/caveman). Confirm the intended project before integration. Caveman offers distinct features for shorter output and reduced input, among others; selecting one feature must not implicitly enable the rest.

Acceptance criteria for a trial:

- Caveman can be enabled or disabled without changing the task workflow.
- Evaluation compares the baseline, Ponytail alone, and Ponytail with the selected Caveman features on equivalent tasks.
- Measurements include instruction overhead, repeated context, repair passes, and both input and output tokens.
- Shortening or compression preserves actionable review findings, evidence references, requirements, and required structured data.
- Observability retains the actual messages exchanged and, where content is transformed, distinguishes the original tool evidence from what the model received.
- Any automatic model routing is disabled unless separately configured by the user; optimization cannot silently override per-role model choices.

Adoption depends on measured additional savings without an unacceptable loss of correctness, review quality, or diagnosability. The evaluation thresholds and feature selection remain open.

### NFR-04 — Be portable across supported systems

Status: required.

The system must be downloadable and usable on different supported systems with documented setup and either API credentials or eligible OpenAI subscription access. It must not depend on the original developer's machine, personal paths, or copied authentication state. Jira, repository access, and target-project build tools remain separate prerequisites.

Acceptance criteria:

- Installation, prerequisites, configuration, authentication, and first-run instructions are documented.
- A clean machine on each declared supported platform can complete setup and a representative task without modifying application code.
- Project paths and configuration locations are configurable rather than hard-coded.
- Shared configuration can be transferred without embedding credentials; each machine can authenticate independently.
- Setup reports missing dependencies, unavailable model access, or unsupported platform features clearly.
- Dependency and integration versions are reproducible across installations.

First-release deployment is one server hosting the coordinator, Docker Engine, PostgreSQL, credential manager, artifacts, and per-run Docker volumes. Docker starts a separate container for each admitted task run, up to the configured limit of three. Splitting these components across servers is outside current scope. Portability therefore requires Docker Engine on each declared supported platform. To decide: supported operating systems, Docker packaging and setup, minimum prerequisites, and platform-specific execution isolation. This does not promise support for every operating system.

### NFR-05 — Make execution and agent communication observable

Status: required.

The user must be able to inspect a running or completed task and understand what the agents received, what they said, what actions they attempted, and how the workflow responded. Logs and metrics must support diagnosing an agent that pursues the wrong work, repeats itself, fails, or stalls. A durable, append-only event history records system-visible decisions, command intents, results, and state transitions. Correlated traces span agent, model, tool, Git, and Jira operations; aggregate metrics expose failures, latency, queueing, and usage. LangSmith may provide LangGraph tracing, but recovery must not depend on a tracing service as its authoritative record.

Acceptance criteria:

- Structured events include timestamps and identifiers for the project, ticket or task, run, role, session, and iteration as applicable.
- Events that affect workflow state or external operations are durably recorded in an ordered history with operation, causation, and schema identifiers. The task timeline and event-backed diagnostic views can be rebuilt from that history; large outputs are retained as referenced artifacts rather than embedded in every event.
- A chronological task view connects workflow transitions, agent requests and responses, tool calls and results, validation outcomes, retries, errors, and completion reasons.
- A user can follow a failure from the task and workflow decision to the relevant trace span, agent or tool input and result, error, code snapshot, and recorded reason for stopping or retrying. Traces share task/run/session/operation identifiers with the event history.
- Agent, model, tool, Git, and Jira spans expose duration, outcome, retries, and error details where the integration provides them. Missing or sampled trace data is identified rather than presented as a complete record.
- Inter-agent communication records the sender, recipient, handoff content, and associated evidence. This includes feedback routed through the orchestrator or shared state rather than direct agent chat.
- The effective instructions and context supplied to each session are inspectable, including enabled skills and references to the versions of supplied artifacts.
- Logs can be filtered by task, role, session, stage, and time, and exported for investigation.
- Activity is visible while a run executes and remains available afterward; monitoring exposes last activity and time spent in the current stage.
- Aggregate metrics show queue depth, stage duration, failure and retry rates, and the token and cost observations defined in NFR-06. A task view links its usage totals to the underlying model/session and evaluator usage records, including unknown or estimated values.
- Credentials and sensitive authentication material are redacted. Redaction, truncation, or unavailable provider events are identified explicitly rather than silently omitted.

The event history is the authoritative local recovery and audit record for system-observed business facts; current task and usage views can be rebuilt from it. [Durable recovery](durable-recovery.md) defines its ordering and checkpoint relationship. Traces and metrics provide diagnostic and aggregate views over correlated activity. An event records what the system observed, so uncertain external effects still require Git or Jira reconciliation. This covers observable messages and execution events exposed by the integration. It does not assume access to a model's private internal reasoning or invent token usage that a provider did not report. Retention limits, access controls, update latency, trace export, and the log viewing interface remain to be specified.

### NFR-06 — Measure usage to tune parallelism

Status: required.

The system must expose token usage and operational metrics per agent and task so the user can make informed decisions about the parallel task limit in FR-03. Usage monitoring must work across both access modes, with capability gaps made visible.

Acceptance criteria:

- Usage is attributed to run, role, session, iteration, configured model, and actual model where reported.
- Each agent-model and policy-evaluator call has a correlated usage observation linked to its event and, when available, trace span, so the task view can account for implementation, review, repair, and guardrail activity together.
- Input, output, cached input, and reasoning-token details are recorded when exposed, with their accounting semantics preserved to avoid double-counting subsets.
- Usage from repair attempts, failed calls, and replaced sessions is included when reported; missing usage is marked unknown rather than recorded as zero.
- Cumulative reports avoid counting the same usage event twice and distinguish provider-reported values from estimates.
- The user can compare token totals, duration, throughput, queue wait, repair counts, failures, and rate-limit events at concurrency levels 1, 2, and 3.
- Reports retain the relevant model, session policy, and optimization settings so comparisons can account for configuration differences and task difficulty.
- API cost estimates, if offered, are labeled with their pricing basis. Subscription usage and any exposed account limits are shown separately; token counts are not presented as an exact subscription quota balance.

The requirement is visibility for user-directed tuning. Automatic adjustment of concurrency is not selected. Metric availability for each provider and access mode must be verified during integration design.

### NFR-07 — Keep subsystems modular and independently testable

Status: required.

The system must separate subsystem responsibilities so one integration or implementation can be upgraded without rewriting unrelated workflow behavior.

Acceptance criteria:

- Ticket access, workflow state, agent execution and authentication, workspace and repository operations, policy enforcement, optimization integrations, and observability have explicit boundaries.
- Boundaries exchange documented, validated data rather than depending on another subsystem's private state or unstructured conversation parsing.
- Provider-specific authentication and response formats are handled at the integration boundary.
- A subsystem can be exercised with test substitutes for its external dependencies.
- Replacing a ticket adapter or upgrading an agent integration does not require changes to the implementer–reviewer process when its contract is unchanged.
- Contract and persisted-state changes have an explicit compatibility or migration strategy.

These are separation requirements, not a requirement for microservices, separate deployments, or a general-purpose plugin platform. The [functional architecture proposal](functional-architecture.md) groups the responsibilities into a modular coordinator and a required per-run container boundary; [high-priority gaps](design-gaps.md) record the remaining runtime and storage decisions. Concrete interfaces will follow those decisions.

### NFR-08 — Configure the model independently for each agent

Status: required configuration capability.

The user must be able to choose a model independently for the implementer and senior reviewer, and for any additional agent roles introduced later. Model selection and session-reuse policy are separate settings.

Acceptance criteria:

- The two roles can use different models, or the same model, within a task run.
- Effective model settings are visible before work starts and recorded with execution and usage events.
- Every newly created session receives the model configured for its role, including sessions replaced under FR-05.
- Unsupported or inaccessible model choices produce a clear error rather than a silent fallback.
- Neither an authentication adapter nor a token-optimization integration silently changes the selected model.
- Settings can be changed without editing application code.

To decide: defaults, configuration precedence across system/project/task scopes, optional model-specific settings, and whether model changes can apply to an active run. Supporting multiple model choices does not automatically require support for multiple model vendors.

### NFR-09 — Detect and contain drift through modular guardrail policies

Status: required capability and modularity constraint. NeMo Guardrails is the preferred framework, pending confirmation of integration fit; it is not installed or finalized as a dependency.

The system must use a framework-backed policy layer to detect and contain divergence from the task's established scope, requirements, design, workflow permissions, and execution limits. Policies must be easy to define, test, change, and version independently of ticket adapters, agent integrations, and core workflow code.

NeMo supports configurable rails at input, output, and tool-execution boundaries, with configuration, custom prompts, flows, and custom actions. These are a plausible basis for the required policy layer; task-specific scope judgments still need policies and evaluation. [NeMo rail types](https://docs.nvidia.com/nemo/guardrails/latest/about-nemo-guardrails-library/rail-types), [NeMo configuration](https://docs.nvidia.com/nemo/guardrails/latest/configure-guardrails/overview)

Acceptance criteria:

- Policies are defined as separately versioned JSON configuration and rule/evaluator modules behind a stable interface. Changing a rule, threshold, prompt, or project-specific policy does not require rewriting the workflow. New evaluator capabilities may require a modular extension.
- Each run records its effective policy versions, settings, evaluator models, and enforcement points. Policy updates have explicit activation and rollback behavior; an active run must not silently change policy mid-execution.
- The task specification established after preflight is the reference for scope checks. Agents may propose corrections or additional work, but cannot rewrite that reference or weaken policy to justify their own actions.
- Policy results identify the rule, decision, evidence, and required response. Evaluation errors or unavailable checks are distinguished from policy violations and never treated as implicit permission for a protected action.
- Protected actions are checked before execution where blocking is required. The architecture identifies which internal agent/tool actions can actually be intercepted; observing a whole session afterward is not represented as prevention of its internal actions.
- Objective constraints such as allowed transitions, task limits, publication permissions, and exact-code approval remain enforced by application/execution controls. Semantic checks assess scope and requirement alignment against evidence. A model's favorable assessment cannot bypass the objective constraints.
- Confirmed necessary work outside scope invokes FR-10. An uncertain scope decision halts for clarification. Containment stops implementation while allowing the narrowly defined preservation and ticket-handoff operations.
- The policy layer covers preflight decisions and significant execution boundaries, including plan/approach assessment, review, ticket mutations, handoffs, and publication. Exact evaluation frequency and role ownership remain design decisions.
- Policy inputs, decisions, failures, and resulting actions are correlated with task logs. Evaluator tokens, latency, and costs where available contribute to usage metrics.
- Evaluations include legitimate in-scope changes, unrelated edits, forgotten requirements, stale approvals, invalid ticket closures, injected instructions, repeated repair loops, and partial handoff failures. Report missed violations, false blocks, and overhead before enabling a policy as a blocking gate.
- Guardrail integrations support the selected agent access modes, or clearly identify unsupported enforcement. Any additional model credentials, runtime requirements, or costs are explicit rather than assumed to be covered by an existing subscription.

Design direction: keep scheduling, per-task state transitions, and policy enforcement as separate responsibilities. LangGraph owns workflow state; the guardrail layer returns policy decisions; adapters perform authorized external operations. These can remain modules in one application. No long-lived manager-agent conversation is required to hold all task state.

To decide: NeMo version and integration mechanism, initial JSON policy schema and rule library, evaluator selection, validation thresholds, configuration precedence, and policy-update behavior for active runs. Integration choices must respect the JSON-only configuration requirement rather than introduce another policy-authoring format. Llama Guard or another classifier may be used within a policy if justified, but a generic content-safety classifier alone does not satisfy task-drift detection.

### NFR-10 — Use JSON exclusively for structured data and configuration

Status: explicit required format constraint.

Messages, events, stored structured records, system/project/agent configuration, policy definitions, schemas, and structured examples must use JSON. Human-authored and generated configuration use the same format and validation rules.

Acceptance criteria:

- Configuration and structured messages are validated against their versioned JSON schemas.
- There is no YAML configuration path, alternate authoring format, or format-conversion layer.
- Third-party integrations do not require users to maintain a second configuration format.
- JSON parsing is strict and consistent across configuration and message handling.

Referenced source code, prose documents, and raw tool output retain their native content formats; their structured metadata and configuration are JSON.

## 6. Communication contracts

The [message-contract draft](message-contracts.md) defines a common envelope, typed commands/results/events, shared evidence and context records, agent handoffs, usage accounting, journal and delivery records, schema evolution, and delivery semantics. A [draft JSON envelope schema](contracts/message-envelope.schema.json) accompanies the document; payload and domain validation are specified in prose pending review.

JSON is required for all structured messages, records, configuration, policies, schemas, examples, and persisted graph state. PostgreSQL is selected for the journal/inbox/outbox and checkpoints on the same server as the coordinator and task containers. Coordinator-internal handlers may use in-process dispatch, while assignments and results cross an authenticated local task-container boundary whose exact transport remains open. The [durable recovery proposal](durable-recovery.md) retains the reconciliation rules. Module interfaces will be derived from the agreed contracts. No message delivery or runtime implementation exists yet.

## 7. State-machine design

The [PlantUML state-machine draft](state-machine.md) translates the requirements and message contracts into task and delivery-path overviews plus fourteen detailed sub-state machines. It covers task intake, scheduling, process restart and host-shutdown recovery, access readiness, project/workspace preparation, preflight, role sessions, implementation/review, dependency handoff, publication, guarded command delivery, agent-output/handoff policy, and final outcome tracking. The index identifies current and stale previews.

The user's sketch and later clarification require configurable delivery, optional human review, feedback handling when enabled, remote code publication, and ticket completion. The state-machine draft shows the human-review wait across runs, PR modes, guarded direct-main push, and Jira completion gate. Diagram state names are proposed vocabulary, not implemented behavior or frozen message-schema enums.

## 8. Remaining requirements and design decisions

The [high-priority design gaps](design-gaps.md) now build on the selected single-host, per-run-container deployment with Docker Engine. Its integration details, the coordinator/worker contract, persistent storage layout, and detailed durable recovery remain to be designed. Further requirements must define execution sandboxing and credential protection around the selected per-run Docker volume/full-clone mechanism, cancellation and pause behavior, usage and retry budgets, and measurable performance and retention targets. The requirements above establish direction without claiming those guarantees are already specified.

LangGraph and Ponytail are selected constraints; Caveman remains an optional evaluation. Drift detection and containment with modular policies are required, with NeMo Guardrails preferred pending integration design. Ticket preflight, dependency handoff, and system-created Docker volumes and full clones are required workflow stages. Initial durable recovery is proposed separately; volume naming and layout, platform-specific executor fencing, execution sandboxing, module interfaces, packaging, supported platforms, and the user interface still need design. No application code, dependencies, or implementation milestones are introduced by this draft.

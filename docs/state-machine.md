# Slop Machine state-machine design

Status: PlantUML design draft based on the requirements, communication contracts, and the user's state-machine sketch. No runtime is implemented. The names here propose workflow states; they are not yet frozen enum values in the message schemas.

## Overview

The overview describes a task across one or more runs. On process restart, recovery first fences prior owners, reconciles durable and observed effects, and either restores a passive task or resumes an interrupted run at a verified boundary. Its scheduler detail invokes task intake, which fetches an eligible Jira ticket and records its ID and revision or captures a user prompt and clarifications. The scheduler can admit one, two, or three concurrent active task runs according to configuration. It checks selected-mode compatibility and resolves the configured target repository before claim; after claim, the system creates and verifies a new Docker volume with an independent full clone on the task's branch. Each task container then fetches the current selected credential from the credential manager at startup and proves its runtime is ready. Only then does the system mark linked Jira work started and dispatch preflight. Agents and external adapters report outcomes; application workflow rules perform the transitions.

For a ticket that proves unneeded, follow **Fetch eligible Jira ticket → Queue → Selected-mode precheck → Resolve repository → Claim run → Create and verify new volume/full clone → Start container and fetch current credential → Verify Jira work started → Preflight → Mark Jira ticket unneeded → Close sessions → Retire volume → Verify final outcome → Closed without code**. Preflight re-fetches the ticket and compares it with the design and current code before deciding. The no-code outcome requires a recorded explanation, sufficient evidence, and confirmation that Jira was updated and closed. A prompt-only task uses the analogous internal task closure path. This run still receives a volume/full clone, but does not need coding sessions.

After senior-agent review and checks, the configured delivery mode selects one of three paths:

| Mode | Path to completion |
| --- | --- |
| `pr_human` | Push reviewed branch → create/update PR → request human review in PR or task chat → incorporate feedback or approve → verify merge → mark Jira Done when linked. Waiting for a human ends the active run; a response queues a new run. |
| `pr_automatic` | Push reviewed branch → create/update PR → apply current checks, policy, and any PR-host approval rules → verify merge → mark Jira Done when linked. No human-review wait is added by this mode. |
| `direct_main` | Check the expected main head and direct-push permission → integrate the reviewed snapshot → fast-forward push main without a PR → verify the remote main commit → mark Jira Done when linked. |

A prompt-only task without a Jira ticket verifies its own completion record after the target branch update. PR creation and a branch push do not complete a Jira ticket. The default mode remains a design decision.

[Delivery-path PlantUML source](diagrams/delivery-paths.puml)

[Current task-run lifecycle PlantUML](diagrams/agent-state-machine.puml). Its earlier PNG/SVG previews are stale after the source changes below.

## Detailed state machines

The design uses two overviews and fourteen detailed views. The task overview stays focused on task and run outcomes; startup recovery and other operational decisions live in their own detailed flows. A detail view's final marker returns an outcome to its caller; it does not necessarily end the task. Only `pr_human` enters the task overview's human-review state, which has no active run or slot; an authenticated response queues a new run. A dependency resolution never resumes the old agent loop.

| View | Responsibilities | Source | Render |
| --- | --- | --- | --- |
| Task lifecycle | Scheduler handoff, run boundaries, human-review wait, final outcomes | [PlantUML](diagrams/agent-state-machine.puml) | Stale; refresh pending |
| Delivery paths | Small decision view of the three configurable delivery modes | [PlantUML](diagrams/delivery-paths.puml) | Source only |
| Task intake | Jira ticket retrieval, prompt clarification, source preservation, tracking choice | [PlantUML](diagrams/task-intake.puml) | Stale; refresh pending |
| Task scheduler | Pre-claim access/project checks; claim, create volume/full clone, start credential-ready container, verify work-start update, dispatch | [PlantUML](diagrams/task-scheduler.puml) | Stale; refresh pending |
| Host-shutdown recovery | Reconstruct ticket/run-to-volume mapping, fence old containers, inspect saved clone, resume same run or quarantine | [PlantUML](diagrams/host-restart-recovery.puml) | Source only; render pending |
| Process restart recovery | Establish ownership, fence old executors, reconcile effects, fetch current credential in replacement container, resume or close | [PlantUML](diagrams/process-restart-recovery.puml) | Stale; refresh pending |
| Access readiness | Pre-claim mode check; task-container credential fetch at every start; selected runtime setup and visible failure | [PlantUML](diagrams/access-readiness.puml) | Stale; refresh pending |
| Project and volume/clone | Resolve repository before claim; create a new verified volume/full clone for every run; verify before code and retire safely at run end | [PlantUML](diagrams/project-workspace.puml) | Stale; refresh pending |
| Preflight | Reconcile ticket, design, current code, prerequisites and pending human review/delivery | [PlantUML](diagrams/preflight.puml) | Stale; refresh pending |
| Role sessions | Independent selection per role/pass, policy-checked handoffs, explicit context and run-end teardown | [PlantUML](diagrams/role-session.puml) | Stale; refresh pending |
| Implementation and review | Isolated workspace, output policy, snapshots, checks, bounded repairs and exact-code approval | [PlantUML](diagrams/implementation-review.puml) | Stale; refresh pending |
| Dependency handoff | Stop work; conditionally establish prompt's original Jira ticket; create/link prerequisite and preserve branch | [PlantUML](diagrams/dependency-handoff.puml) | Stale; refresh pending |
| Configured delivery | PR/human response or direct main push; renew approval binding before merge; verify target and Jira Done | [PlantUML](diagrams/publication.puml) | Stale; refresh pending |
| Guarded operations | Validate commands, enforce policy, persist intent, execute, reconcile, retry or cancel | [PlantUML](diagrams/operation-guardrails.puml) | Stale; refresh pending |
| Output and handoff policy | Check significant agent plans/results and role handoffs; contain necessary extra work and route uncertainty | [PlantUML](diagrams/output-handoff-policy.puml) | Source only; render pending |
| Final outcome tracking | Verify stage-owned Jira outcomes; publish attention/cancellation and expose pending sync | [PlantUML](diagrams/run-outcome-tracking.puml) | Source only; render pending |

All views share [the local PlantUML style file](diagrams/state-machine-style.iuml). Keep that file beside the `.puml` sources when rendering them. The lifecycle, scheduler, access, and restart sources have changed; their existing image previews are stale. No new renders are included in this pass.

### Preflight

[Current PlantUML source](diagrams/preflight.puml). The earlier PNG/SVG previews do not show resumed human-review checks.

### Implementation and review

[Current PlantUML source](diagrams/implementation-review.puml). The earlier PNG/SVG previews do not show agent-output policy transitions.

### Readiness and session details

[Access readiness](diagrams/access-readiness.puml) separates a secret-free pre-claim compatibility check from credential startup. Every new or replacement task container pulls the current credential for its explicitly selected mode from the credential manager, then verifies the selected runtime before Jira work-start or preflight. Unsupported tooling/models, missing or stale credentials, authentication failures, and limits return actionable outcomes. There is no subscription-to-API fallback. Later failures use bounded operation recovery under the same selected access; an unavailable quota measurement is recorded without inventing a balance.

[Project/volume lifecycle](diagrams/project-workspace.puml) has four system-owned entry purposes. `resolve_for_admission` supplies verified repository identity and a target snapshot before claim. `create_for_run` creates and verifies a new Docker volume with an independent full clone on a task-owned branch after claim, before Jira start and preflight. `verify_for_code` checks the existing run clone against the accepted specification before an agent assignment; a moved base repeats preflight. `end_run` removes a safe volume or retains a dirty one with a recovery record. A later new run gets a new volume and can clone a verified saved remote branch. Same-run recovery remounts the existing volume after fencing and reconciliation. Neither agent chooses a branch or isolation mechanism.

[Role sessions](diagrams/role-session.puml) also has two entry purposes. `select_for_pass` independently selects a fresh or reusable session for the requested role and delivers explicit current context. `end_run` stops/fences both roles, persists endings and evidence, and revokes session references before lifecycle cleanup releases the execution slot. Incomplete cleanup retains the slot and exposes recovery. A task's evidence survives session replacement.

### Process restart recovery

[Process-restart recovery](diagrams/process-restart-recovery.puml) governs coordinator restart and operation-level reconciliation. The separate [host-shutdown recovery](diagrams/host-restart-recovery.puml) inventories persistent services and Docker volumes, joins each ticket/run record to its saved volume, and inspects the full clone before a replacement container can remount it. The [durable recovery design](durable-recovery.md) defines the authoritative journal and checkpoint boundary. Startup pauses dispatch, establishes one current coordinator owner, and classifies passive tasks, pending Jira sync, active runs and orphan/mismatched volumes. The old executor must be stopped or fenced; in-flight operations are compared with observed Git, PR, Jira and provider effects before any retry. Only a consistent checkpoint and verified volume/clone, inputs, selected mode and budget can resume the **same run** at its saved stage. The replacement fetches the current selected credential and starts fresh role sessions. Otherwise the old run is safely closed or the affected task is quarantined. A later new run gets a new volume.

### Dependency handoff

[Current PlantUML source](diagrams/dependency-handoff.puml). The earlier PNG/SVG previews omit prompt-only ticket promotion.

### Publication

[Current PlantUML source](diagrams/publication.puml). The earlier PNG/SVG previews omit the renewed approval gate.

### Guarded operations

[Current PlantUML source](diagrams/operation-guardrails.puml). See also the new [output/handoff policy source](diagrams/output-handoff-policy.puml) and [final outcome tracking source](diagrams/run-outcome-tracking.puml).

### Cross-view acceptance walkthroughs

These paths name the expected handoff between diagram sources. They are design checks for development, not evidence of a running system.

| Case | Required path and observable result |
| --- | --- |
| Jira ticket with invalid access selection or repository mapping | Queue → selected-mode precheck → repository resolution. A failure returns attention without a run claim, execution slot, volume, or Jira work-start update. The user sees the selected mode, target project and reason for the block. |
| Container cannot fetch the current credential | Claim run → verify volume → start container → authenticate to credential manager and fetch selected credential. Missing/stale credentials or unsupported runtime access block agent work and the Jira work-start update; startup cleanup stops/fences the container and reconciles the volume before releasing the slot. No mode fallback occurs. |
| Claimed run whose volume cannot be verified | Claim run and reserve slot → system creates or reconciles Docker volume with full clone → verify volume identity, clone origin, branch, commit and task ownership. Failure enters startup cleanup; no agent runs and no Jira work-start update is issued. Confirmed cleanup releases the claim/slot; uncertain cleanup retains them for recovery. |
| Host restarts with active tickets | Keep dispatch paused → recover PostgreSQL task/ticket/run records → match each saved volume ID to Docker metadata and manifest → fence old containers → inspect clone and reconcile effects/checkpoint → remount the same volume in a replacement container and resume the same run at a verified stage. Missing or mismatched volume evidence quarantines the affected task without discarding its work. |
| Cancellation during claimed startup | Stop startup → reconcile any in-flight volume creation and Jira start update → retire or retain the volume safely → record cancellation → release the claim/slot only after effects and ownership are known. No agent begins preflight. |
| Process restarts mid-operation | Pause dispatch → establish coordinator ownership → inspect checkpoint and operation intent → fence old executor → reconcile observed effect. A confirmed effect advances the saved state; an absent effect may be retried only when safe. Unknown effects never trigger blind replay. Resume the same run only at a verified stage boundary; otherwise close or quarantine it while preserving partial work. |
| Process restarts after a verified PR merge but before Jira Done | Reconcile the saved merge operation against the remote target branch and merged commit → do not merge again → verify or safely complete the pending Jira transition → restore the recorded outcome without reviving an old agent session. |
| Old agent process survives coordinator restart | New coordinator acquires its epoch but keeps the affected run's claim and slot → revoke/stop/fence the old process and volume writer → no new agent or checkout for that task until fencing is confirmed. Other eligible tasks may proceed within remaining capacity. |
| Checkpoint is consistent but the container or an agent session is gone | Reconcile in-flight effects and verify volume, inputs, selected mode and budget → start replacement container and fetch current credential → resume the same run at its saved stage with fresh sessions and explicit findings/context → never restore saved token bytes, replay the old tool call, or create a second volume for that run. |
| Preflight closes a ticket without code | Claim run → system creates and verifies volume → Jira start update → preflight evidence-backed closure → stop zero coding sessions → inspect and retire the clean volume → confirm Jira closure. |
| Human requests in-scope changes | End old run and release its slot → authenticated response queues a new run → preflight refreshes ticket, design, target code, prerequisites and saved PR work → freeze current specification → preserve feedback as repair context → implementer and senior reviewer use new-run sessions → update PR and request another human review. |
| Human approves an unchanged PR | New run and preflight → reconcile current specification, PR head, prior senior approval and checks → publication authenticates the response again → renewed approval gate checks exact snapshot, spec, policy and base → merge/verify target branch → Jira Done when linked → final outcome tracking verifies the ticket. A mismatch returns to review/repair or attention, never directly to merge. |
| Internally tracked prompt needs a prerequisite | Stop/fence implementation → create or reconcile an original Jira ticket linked to the internal task and original prompt → create/find prerequisite → preserve current work on its dedicated remote branch → link dependency, mark original ticket blocked and sync internal record → end run. Failure to establish the original ticket leaves handoff incomplete; it does not create an unlinked prerequisite. |
| Run ends in attention or cancellation | Stop/fence role sessions → record requested outcome and partial effects → update linked Jira ticket with reason and recovery action → verify ticket result. If Jira cannot be confirmed, persist pending sync and return attention; release the stopped run's slot after recording that failure. |
| Agent proposes off-scope work in output or handoff | Capture exact proposal with task/spec/policy refs → evaluate versioned policies → allow in-scope work, reject a correctable proposal, seek clarification for uncertainty, or stop implementation and enter FR-10 for confirmed necessary extra work. Observed-only effects are recorded and reconciled before any retry. |

## How to read the transitions

Labels follow the conceptual form `event [guard] / effect` where useful. A guard must be satisfied before the transition; a label describing a successful external operation means its outcome was verified, not merely requested. Diamonds route on mutually exclusive outcomes. Green states denote successful outcomes, amber denotes waiting/attention/cancellation, and red denotes a dependency blocker.

State boxes describe a period in the task's lifecycle, not separate autonomous manager agents. Jira is an external system accessed by an adapter; it is not itself a workflow state. Likewise, the implementer and reviewer are roles invoked while the workflow is in implementation or review states.

## Important behavior preserved from the requirements

1. **Intake, concurrency and ownership:** Jira tickets and optional prompts enter through the same scheduler. Prompt clarification preserves the original request and replies. Selected-mode compatibility and target repository are checked before claim; the claim rechecks their freshness with capacity and ownership. The system creates and verifies a fresh Docker volume with full clone for the claimed run, then starts its container and fetches the current selected credential before Jira work-start and preflight. Agents receive the system-selected volume. The scheduler prevents duplicate active runs and enforces the configured 1–3 limit. Waiting for a prerequisite or retained old volume consumes no execution slot. Cleanup does not release a slot until the old executor is stopped or fenced against further effects; cleanup failure remains visible for recovery.
2. **Freshness:** preflight refreshes the ticket, design, and target branch. Missing required design or conflicting intent explicitly returns clarification; no no-design exception has been selected. A demonstrably stale ticket with unchanged intent can still be corrected and reassessed. Closure as unnecessary requires evidence and a confirmed Jira update; prior PR or direct-push effects awaiting finalization resume delivery instead of being closed as unneeded. Ambiguity never becomes permission to change scope or close a ticket.
3. **Role sessions:** the first invocation of each role in a run uses a fresh session. Subsequent passes use that role's independent reuse/fresh-per-pass setting; all four combinations are represented by a role-parameterized selector. A completed preflight that closes a ticket need not instantiate coding sessions. Each invocation records its model resolution, supplied context, and usage. Fresh sessions explicitly receive current work, validation and unresolved findings; run-end teardown closes both roles before slot release.
4. **Review evidence:** every review and validation result binds to an immutable code snapshot and specification. Repairs, rebases, or other changes invalidate the relevant old approval and require new checks/review. Review findings retain stable IDs through repair.
5. **Configurable delivery and completion:** all modes require senior-agent approval, current checks and policy, verified target-branch publication, and a confirmed completion status. `pr_human` requests an explicit human decision; a response starts a new run, whose preflight reconciles the current task, PR head, checks and old approval. In-scope feedback returns to the implementer; approval reaches a second current-spec/snapshot gate before merge. Final rejection needs attention. `pr_automatic` creates and merges a PR without an added human wait, unless the host requires approval. `direct_main` creates no PR and uses a fast-forward push guarded by the expected main head. A chat approval does not waive required PR-platform approval.
6. **Scope containment:** significant agent plans, results and role handoffs receive a separate policy check against the fixed task specification. Confirmed necessary work outside scope triggers the dependency handoff, including if discovered during publication review. The current run may propose that work but cannot implement it. Uncertain scope decisions halt for clarification. Only an intercepted boundary can prevent an action before execution; observed-only effects are recorded and reconciled.
7. **Durable handoff:** a prompt-only task that reaches FR-10 first acquires a verified original Jira ticket linked to its internal record and original request. Prerequisite creation, code preservation, dependency linking, ticket updates, and confirmation are separate recoverable operations. An ambiguous original/prerequisite ticket result is reconciled before retrying. No-change handoffs record the base rather than invent a commit.
8. **Fresh pickup:** after the prerequisite is resolved, the scheduler may queue a new run. It rechecks whether the fix is actually available in the target code, assesses the original ticket again, and reconciles saved work. Old agent sessions and old approval are never resumed.
9. **Guardrails and observability:** protected commands and significant agent output/handoffs pass through versioned policy checks and execution permissions. JSON commands, results, actual agent messages, tool activity, handoffs, policy decisions, usage, and transitions remain correlated in the journal. Logs and token accounting are continuous side effects rather than extra blocking workflow stages.
10. **Failure semantics:** a completed test that fails is different from a test runner that fails to start. A completed review asking for repairs is different from a malformed model response. Operational retries reuse operation identity; a new repair pass has a new operation and consumes the configured task budget.
11. **Stop and outcome semantics:** cancellation, exhausted budgets, unresolvable policy failures, and unrecoverable operations produce recorded outcomes and cleanup. After executors stop, the system inspects the run volume, removes it when safe or retains it with a recovery block. Final tracking then verifies stage-owned Jira states or posts attention/cancellation with a reason. Failed sync is recorded as pending and returns attention without retaining a stopped run's slot. Partial remote effects remain recorded. Late results cannot revive a stopped run. Deliberate requeue after attention starts a new run and first reconciles existing effects.
12. **Access and isolation:** selected-mode compatibility and repository identity are checked before claim, then revalidated at claim. Every claimed run receives a fresh, exclusive Docker volume with full clone. Its task container fetches the current selected credential from the manager on every start, including replacement startup; token bytes never come from a checkpoint. Credential/runtime readiness is verified before Jira work-start or preflight, even when preflight later closes the task without code. Coding uses the already-created volume after a specification/lease check. Pre-claim readiness is an observation, not a reservation or a promise of unlimited usage. Failures cannot silently change the selected mode/model or overwrite an occupied branch or directory.
13. **Process restart:** startup dispatch stays paused until a single current coordinator owns recovery and persisted records are classified. An interrupted run retains its slot until prior executors are fenced; a lease timeout alone is insufficient. Journal, checkpoint, delivery and observed external effects are reconciled before same-run resumption at a verified boundary. A replacement container fetches the current selected credential, and lost role sessions restart with explicit context; prior tool calls are never blindly replayed. A safely closed run releases its slot, and any later pickup gets a new run and volume. Incomplete ownership blocks the affected task, while a corrupt required store blocks all scheduling.

## Relationship to the sketch and unresolved choices

The sketch's implementation/review loop is retained and expanded with preflight, validation, policy enforcement, repair budgets, dependency preservation, configurable delivery, and ticket completion. Human review is optional. The question of who performs a PR merge remains a deployment-policy choice.

| Topic | Decision represented |
| --- | --- |
| PR publication | Both PR modes push the internally reviewed branch, verify its remote commit, create/update the PR, and link it from the task or ticket. |
| Human review | Only `pr_human` waits for a human decision in the PR or task chat. An absent response never implies approval. Waiting ends the active run; feedback or approval starts a new run after preflight. Current task/spec and approval binding are checked before merge. |
| No-human PR | `pr_automatic` can merge after current agent approval, checks, policy, and host rules; a host requirement for human approval stops this mode rather than bypassing it. |
| Direct main | `direct_main` pushes reviewed code to main without a PR. The expected-head fast-forward guard and integration serialization prevent one parallel task from overwriting another. A changed final snapshot needs renewed checks and senior review. |
| Ticket completion | Move Jira to Done only after the PR merge or direct main push is verified; then verify the Jira transition. |
| Rebase / base movement | Reconcile the branch before merging or pushing main, then validate and review the resulting snapshot. Never publish one snapshot on the approval of a different one. |
| Queue continuation | Whether completing a task also stops intake is independent of its merge and Jira outcome. The scheduler policy remains open. |
| Waiting for a human | In `pr_human`, no execution slot or model process is retained. Review timeout and escalation remain open. |
| Delivery default | No default among the three modes has been selected. A mode is explicit in each task specification. |
| Prompts | Prompts enter the same task flow. Conversion before ordinary execution remains a configuration choice. An internally tracked prompt that reaches FR-10 acquires a linked original Jira ticket before its prerequisite ticket is created. |
| Jira outcomes | Stage-owned updates for Done, no-work, blocked dependency and PR review are verified after run cleanup. Attention and cancellation publish their reasons to Jira when linked; a failed sync is visible and leaves the task in attention for recovery. |
| Role configuration | Per-role session/model defaults, configuration precedence and changes during active runs remain open. No new run can inherit an old session. |
| Access integration | Each task container must pull the current selected credential at startup, but supported subscription token production/consumption, manager reader identity, readiness signal freshness and evaluator credentials/costs still need integration design. No silent mode fallback is permitted. |
| Volume implementation | Named Docker volumes with full clones are selected and created by the system during run startup. Volume naming, branch naming, retention and backup details remain open. Exclusive ownership, safe teardown and a new volume for every new run are required. |
| Missing project design | Preflight explicitly requests clarification; authority selection and any permitted no-design policy still need a decision. |
| Limits | Loop, execution, evaluation, and recovery limits are required bounded policies; numeric defaults and reserved handoff capacity remain to be chosen. |
| Long waits and pauses | Operation backoff and restart recovery are shown. General user-directed pause/resume remains open. The initial checkpoint store and journal/reconciliation boundary are proposed in the durable recovery design. |

The human-review event, merge result, and direct-main publication contracts are sketched in the message-contract draft; their operation-specific JSON payload schemas and exact hosting-system semantics still need design before implementation.

The restart diagram establishes ownership and checkpoint transitions; the linked durable recovery design proposes the journal/outbox transaction and checkpoint reconciliation rules for the selected single-host container deployment. Those details remain [high-priority gaps](design-gaps.md) until the local coordinator/container ownership and persistent-storage layout are defined in the [functional architecture](functional-architecture.md). Migration compatibility and crash-window checks are proposed as acceptance requirements, while exact container/process fencing still needs implementation design. These sources do not settle Jira selection/claim policy, prompt-to-ticket conversion before ordinary execution, active-run limit changes, or Jira progress-update cadence. Output-policy activation thresholds and coverage for non-interceptable actions remain integration decisions.

## Mapping to communication contracts

| State-machine activity | Existing draft message contracts |
| --- | --- |
| Ticket discovery | `sm.backlog.query` / `sm.backlog.queried` |
| Work-start ticket update | `sm.ticket.mutate` / `sm.ticket.mutated` with the configured active transition or progress update |
| Attention/cancellation ticket update | `sm.ticket.mutate` / `sm.ticket.mutated`; final tracking records confirmed or pending sync |
| Queue and claim visibility | `sm.task.transitioned` with task/run IDs, revisions, reason, and decision references |
| Run volume startup/teardown | Proposed `sm.workspace.volume.create` / `sm.workspace.volume.created` and `sm.workspace.volume.finalize` / `sm.workspace.volume.finalized`; repository adapter owns these operations |
| Preflight decision | `sm.preflight.assess` / `sm.preflight.assessed` |
| Ticket correction, closure, linking | `sm.ticket.mutate` / `sm.ticket.mutated` |
| Role invocation | `sm.agent.assign` → `sm.implementation.completed` or `sm.review.completed` |
| Role session/context lifecycle | `sm.agent.session.started`, `sm.handoff.delivered`, `sm.agent.session.ended`; role assignments carry independent policy and explicit context/feedback references |
| Validation | `sm.validation.run` / `sm.validation.completed` |
| Policy evaluation | `sm.policy.evaluate` / `sm.policy.evaluated` |
| Significant output and handoff checks | `sm.policy.evaluate` / `sm.policy.evaluated`, plus correlated agent/tool and handoff observations |
| Dependency handoff | `sm.dependency.handoff` / `sm.dependency.handed_off`, with separate child operations |
| Preserving work | `sm.repository.preserve` / `sm.repository.preserved` |
| PR publication | `sm.publication.create_pr` / `sm.publication.completed` |
| Human decision | `sm.human_review.recorded` from the PR or task-chat adapter |
| PR merge | `sm.publication.merge_pr` / `sm.publication.merged` |
| Direct main push | `sm.publication.push_main` / `sm.publication.main_pushed` |
| Ticket completion | `sm.ticket.mutate` / `sm.ticket.mutated` with the configured Done transition |
| Run cancellation | `sm.run.cancel` / `sm.run.cancelled` |
| Restart recovery | Persisted task/checkpoint revisions, `DeliveryRecord` leases and operation IDs, journal evidence, `WorkspaceVolumeRef`, session references and observed external results; same-run resume creates new invocation IDs |
| Observable transitions | `sm.task.transitioned` with revisions, reason, and decision references |

Access-readiness results, repository resolution, volume reservation/verification and final outcome sync still need operation-specific payload schemas. The diagram entry purposes and outcome labels are proposed control flow, not additions to the draft JSON message schema.

The diagram's guards apply to message content and evidence, not keywords in free-form agent prose. The command-delivery view is a logical lifecycle; its preliminary policy evaluation is itself an observable bounded operation with deterministic permission checks, not an infinite recursive invocation of its own semantic policy gate.

## Rendering and verification

All seventeen PlantUML sources passed local 1.2026.9 syntax-only validation. This revision does not render the modified scheduler, access, or restart diagrams; their older previews are stale. The renderer lives in temporary storage and is not an application dependency or a repository artifact.

From the repository root, with a local PlantUML JAR available:

```sh
java -Djava.awt.headless=true -jar /path/to/plantuml.jar -checkonly docs/diagrams/*.puml
java -Djava.awt.headless=true -DPLANTUML_LIMIT_SIZE=16000 -jar /path/to/plantuml.jar -tsvg docs/diagrams/*.puml
java -Djava.awt.headless=true -DPLANTUML_LIMIT_SIZE=16000 -jar /path/to/plantuml.jar -tpng docs/diagrams/*.puml
```

Syntax reference: [PlantUML state diagrams](https://plantuml.com/state-diagram). The [Smetana renderer](https://plantuml.com/smetana02) avoids requiring a separate Graphviz installation. Rendering validates the diagram syntax and presentation; it does not prove an implemented workflow's correctness.

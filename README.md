# Slop Machine

A coding agent system intended to take tickets from a backlog and work through them using an implementer agent and a senior engineer reviewer.

The aim is to reduce the manual setup and coordination involved in assigning coding work, reviewing the result, and feeding review findings back into implementation.

## Project status

This repository is at the groundwork stage. It contains the project overview, a requirements draft, Git exclusions, and an MIT license. There is no application code or runnable agent yet.

The [design document](docs/design.md) records functional requirements, non-functional requirements, technology constraints, and usability proposals. The first release is a single-server coding assistant: the coordinator, Docker Engine, PostgreSQL, credential manager, artifacts, and per-run Docker volumes live on that server. It starts a separate task container for each admitted run, with at most three running at once. Deploying components across multiple servers is out of scope. The [functional architecture](docs/functional-architecture.md) and [high-priority design gaps](docs/design-gaps.md) describe the remaining local runtime, storage, and recovery decisions. The [message-contract draft](docs/message-contracts.md) defines communication and observable event formats before module interfaces, with a [draft JSON envelope schema](docs/contracts/message-envelope.schema.json). Delivery transport remains a design decision. Architecture will be developed before coding begins.

All structured messages, configuration, policy definitions, schemas, and examples use JSON exclusively.

The [PlantUML state-machine design](docs/state-machine.md) provides task and delivery-path overviews plus detailed views for intake, scheduling, process and host-shutdown recovery, access readiness, project/workspace preparation, preflight, role sessions, implementation/review, dependency handoff, publication, guarded operations, agent-output policy, and final outcome tracking. The [durable recovery proposal](docs/durable-recovery.md) records ordered event history, checkpoint reconciliation, and same-volume recovery after host shutdown. The index marks current and stale previews.

## Intended capabilities

- Read tickets from Jira and create new tickets when needed. Jira is the first planned backlog integration.
- Reconcile each ticket with the design and current code before implementation; correct stale tickets and close unnecessary ones with evidence.
- Work against configured project repositories, rather than being tied to this repository.
- Create a fresh Docker volume and full repository clone for every claimed run before preflight or agent code work; recover an interrupted run from its same volume.
- Use an implementer agent and a senior engineer reviewer in a bounded feedback loop.
- Run project checks and return actionable findings to the implementer.
- Configure delivery as a PR with human review, a PR without human review, or a guarded direct push to main with no PR.
- When human review is enabled, route PR or task-chat feedback through another bounded implementation and review pass.
- Verify the merge or direct main push, then mark the Jira ticket Done or complete the prompt-only task.
- Support at most three concurrent ticket workflows, with separate working directories.
- Use the existing Codex subscription through supported Codex tooling where available.
- Be portable across the machines used to run the system.
- Expose agent messages, inter-agent handoffs, logs, metrics, and token usage for diagnosis and concurrency tuning.
- Configure the model and session-reuse policy independently for each agent role.
- Keep subsystem boundaries modular to make development and upgrades easier.
- Detect and contain drift through modular, versioned guardrail policies.
- When additional work is required outside scope, create a prerequisite ticket, preserve current code on a dedicated remote branch, mark the original ticket blocked, and halt. A fresh agent loop can pick it up after the dependency is resolved.

In the human-review PR mode, the task waits for an explicit decision without holding an agent session or execution slot; the response starts a new run. In the no-human modes, the required senior-agent review and checks still apply. The system never marks Jira Done before verifying that the code reached the configured target branch.

## Questions for the design process

- How should Jira projects, ticket selection, ticket creation, and repository mappings be configured?
- What responsibilities and evidence should the implementer and senior engineer exchange?
- What checks must pass before work can be considered ready for a pull request?
- How should Docker volume naming, branch naming, credentials, concurrency, and conflicting changes be handled?
- How should platform-specific executor fencing, user-directed pauses, and usage or retry limits work alongside the proposed durable recovery design?
- How should LangGraph manage task state, and how will token optimizations be evaluated?

LangGraph is selected for workflow state management, and Ponytail is a required integration aimed at token efficiency and code quality. Caveman is an optional candidate for further token savings. NeMo Guardrails is the preferred framework for modular drift policies, pending integration design. A new task container for each admitted run is required for the first release, with Docker Engine as the selected container runtime. Terraform use remains a design decision. Direct merging has not been adopted.

## Repository hygiene

The `.gitignore` excludes credentials, local configuration, virtual environments, dependencies, generated output, runtime state, and agent workspaces. Keep real tokens and authentication caches out of Git. Use sanitized example files for configuration that needs to be shared.

Dependency lockfiles, source files, tests, project instructions, and documentation should remain trackable. Git exclusions do not protect secrets that have already been committed.

## License

[MIT](LICENSE)

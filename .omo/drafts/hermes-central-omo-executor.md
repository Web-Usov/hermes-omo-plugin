---
slug: hermes-central-omo-executor
status: planned
intent: clear
review_required: false
review_mode: self-review-and-independent-design-advice
review_gate: native-plan-review-agents-unavailable-in-this-session
plan_path: .omo/plans/hermes-central-omo-executor.md
plan_sha256: b800aa17423f55be3d12ad7556914d9208f934b33b2df0e67ce13137c4e632c5
pending-action: publish-plan
approach: Extend the existing plugin while separating task completion from unconfirmed Hermes dispatch.
---

# Draft: hermes-central-omo-executor

## Affected user and ideal state

Web-Usov wants Hermes as the central assistant and OmO Native as its autonomous executor for deep technical tasks. IS-1 through IS-6 and GAP-1 through GAP-6 are specified in the canonical plan; its success matrix maps all of them to implementation and failure QA.

## Components (topology ledger)

| Component | Outcome | Status | Evidence |
| --- | --- | --- | --- |
| RPC/lifecycle | Authoritative task completion and run identity | active | host_client.py; OmO 5.1.69 docs/rpc.md |
| Storage | Owned sessions, checkpoints and transactional outbox | active | sessions.py |
| Execution owner | One receiver and command owner per task | active | tools.py prompt/question ownership |
| Hermes dispatch | Original chat routing with explicit unconfirmed state | active | pinned Hermes run_plugin_injection.py |
| Controls | Scoped answers, steering, cancel and workspaces | active | schemas.py; questions.py; tools.py |
| Native transport | POSIX sockets and Windows authenticated named pipes | active | OmO 5.1.69 Windows RPC contract |

## Open assumptions (announced defaults)

- Single-user local deployment; co-located Hermes/OmO first. Cross-machine Linux-to-Windows deployment is outside this milestone.
- TDD with unittest and a package-name-independent test runner; both Windows and Linux/WSL integration are required for implementation completion.
- Preserve explicit permissions; require a workspace and prevent parallel editing of one canonical directory.
- Plugin-only dispatch cannot guarantee durable Hermes admission. Keep events locally and show dispatch_unconfirmed; stronger admission is an external public-API prerequisite.

## Findings (cited - path:lines)

- Baseline upstream/main: 7da797cf57eb23eb92f746f31de397f9ed157ce9, plugin manifest 0.2.8.
- host_client.py:740-778 uses agent_settled; tools.py:688-725 inspects assistant stop state. Neither proves root task completion.
- OmO Native 5.1.69 docs/rpc.md:134-257 supplies authoritative lifecycle and replay/completion rules; Windows endpoints use named pipes and a different auth handshake.
- Hermes 8b2fbfcfe526a79c556baecca8233ebd9f77152e, gateway/run_plugin_injection.py:191-233 confirms that inject_message=True only schedules work; lines 489-518 create a fresh admission UUID and discard its receipt.
- Same Hermes revision, hermes_cli/plugins_context.py:225-244 exposes on_unload/spawn_task. The parent directly checked these sources after independent advice.

## Decisions (with rationale)

- Extend this fork, keep OmO as the execution authority, and preserve its session for explicit follow-ups.
- Capture original routing inside the tool call; a background thread must not infer it from ambient context.
- Use transactionally stored local events and truthful dispatch uncertainty. Do not access private Hermes admission APIs or claim exactly-once effects.
- Keep implementation tasks unchecked. Publication is documentation only.

## Scope IN

Canonical plan, eight implementation tasks, dependency matrix, versioned evidence requirements, real behavior/failure QA and publication to the user's fork.

## Scope OUT (Must NOT have)

Product changes in this session, deployment, third-party messages/PRs, Hermes/OmO core changes, remote transport and automatic task-side privileged actions.

## Open questions

No blocking question for plan publication. The exact first production deployment topology remains an execution-session decision; both supported co-located options and their boundary are explicit.

## Approval gate

status: approved-for-plan-publication
Authorization: the current user message requests creating the fork, adding an OmO plan and pushing it to Web-Usov. It does not authorize implementation or deployment.

## Design advice and review record

- Independent read-only design advisor: st_01a1eb94, completed; no source files edited. Its durable-admission and conversation-reset findings were verified against official pinned Hermes source and incorporated.
- Native plan-consultant/reviewer workflow was unavailable because this session was not opened with that workflow. No high-accuracy plan-review approval is claimed and no gate bypass was attempted.
- Parent self-review checks canonical section order, eight tasks, dependency ordering, acceptance/failure cases, existing source references, truthful delivery guarantees and documentation-only scope.
- Product tests/integration were not run for this documentation-only publication. Their exact future commands and prerequisite behavior are part of the plan.

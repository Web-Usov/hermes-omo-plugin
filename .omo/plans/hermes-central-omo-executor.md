# Hermes as coordinator, OmO Native as deep-task executor

## TL;DR (For humans)

**Who this is for and what changes for them:** Web-Usov talks to Hermes as the central assistant. Hermes delegates a technical goal to OmO, remains available, and brings questions and completion back to the same conversation.

**What you'll get:** An extension of the existing plugin with background execution, local restart recovery, corrections during work, and native Windows plus Linux/WSL support. Automatic delivery exposes uncertainty instead of silently claiming success.

**Why this approach:** OmO keeps responsibility for its tools, agents, implementation and tests. Extend the existing RPC bridge and SQLite database rather than add another task scheduler or copy full agent histories.

**What it will NOT do:** Deploy infrastructure, introduce cross-machine transport, merge code automatically, or let parallel jobs edit one directory.

**Effort:** Large
**Risk:** High - ownership and delivery must remain correct across restarts, questions and background continuations.
**Decisions to sanity-check:** Single-user deployment; explicit workspaces; authenticated local transport; background gateway mode with a synchronous CLI path; durable local outbox with best-effort gateway dispatch. Durable destination admission needs a stronger public Hermes API and is an external prerequisite, not a guarantee this plugin alone can provide.

This publication contains a plan only. Implementation starts with an explicit execution request, for example /ulw-execute selecting this plan.

---

> TL;DR (machine): Large / High; extend plugin 0.2.8 with authoritative lifecycle, persistent background delivery, steering, Windows transport and regression tests.

## Scope

### Affected user and ideal state

**Affected user:** Web-Usov has OmO Native 5.1.69 on Windows. Hermes is the intended central assistant. The location of Hermes is not assumed: the first milestone supports co-located native Windows or co-located Linux/WSL. Linux Hermes driving a Windows worker across a machine/OS boundary requires a separate deployment contract.

| Row | Statement | Reason |
| --- | --- | --- |
| IS-1 | Hermes remains responsive during deep OmO work. | Delegation must not occupy model turns with repeated waits. |
| IS-2 | Completion means the intended root work actually finished. | Child agents, goals and background work can outlive a turn. |
| IS-3 | Results/questions and original routing survive locally after restart; delivery uncertainty is visible. | Public Hermes injection does not acknowledge durable destination admission. |
| IS-4 | Corrections, answers and cancellation affect the intended task. | New instructions must not create duplicate work or affect a newer turn. |
| IS-5 | Native Windows and Linux/WSL use authenticated local transports. | Windows support must not weaken authentication. |
| IS-6 | Workspace, permissions and verification evidence are inspectable. | Executor reports and independent verification are different evidence. |
| GAP-1 | The skill loops over bounded omo_wait calls. | Closed by 4, 5, 8. |
| GAP-2 | agent_settled and transcript heuristics decide completion. | Closed by 1, 2, 3, 4. |
| GAP-3 | No background owner, durable outbox or original-chat delivery exists. | Closed by 2, 4, 5. |
| GAP-4 | Resume rejects running tasks; stale age can imply cancellation. | Closed by 3, 4, 6. |
| GAP-5 | Transport assumes AF_UNIX, getuid, POSIX modes and /proc. | Closed by 1, 7. |
| GAP-6 | Defaults embed an author's cwd/full-access; explicit IDs bypass chat filtering. | Closed by 2, 6, 8. |

### Baseline and contract references

- Fork: Web-Usov/hermes-omo-plugin; upstream: yelsoft16/hermes-omo-plugin.
- Baseline commit: 7da797cf57eb23eb92f746f31de397f9ed157ce9. plugin.yaml:1-16 declares 0.2.8 and Hermes >=0.21.5.
- host_client.py:701-778: retained sessions, prompt acknowledgement and settled-event completion.
- tools.py:235-256,393-473,583-624,688-725: turn/session status, waits, resume and transcript fallback.
- tools.py:266-306: age-based cancellation; tools.py:114-126,332-386: cwd/permission defaults.
- sessions.py:11-40,110-117: stored identities and unscoped explicit-ID resolution.
- host_client.py:489-525,560-610: POSIX credential checks and Unix connection/authentication.
- tests/test_host_integration.py: three opt-in scenarios cover question roundtrip, pending-question reconnect and resume. They do not establish deep-task completion or Hermes wakeup.
- OmO Native 5.1.69 shipped docs/rpc.md: Session lifecycle (lines 134-257), Authentication (1368-1446), Windows endpoints (2551-2553), prompt/steer/follow_up and effect ledger (4220-4450). Capture versioned, redacted fixtures in task 1; the executor must not depend on an absolute workstation path.
- That OmO contract explicitly rejects agent_settled/agent_idle as root completion evidence and exposes waiting_children, waiting_background, waiting_human, restoring, persisted, completionId and completion.kind.
- Windows maps the logical socket address to a named pipe and uses the existing raw .secret handshake; it does not use POSIX endpoint-key proof. The unix:// CLI spelling remains a logical address on Windows.
- Hermes primary sources to pin in task 1: NousResearch/hermes-agent, hermes_cli/plugins_context.py and the actual plugin lifecycle/injection implementation. Verify APIs and routing fields before writing an adapter. Existing requires_hermes >=0.21.5 is not evidence that every new API exists there.

- Verified Hermes revision: 8b2fbfcfe526a79c556baecca8233ebd9f77152e (November 17, 2026). hermes_cli/plugins_context.py:225-244 exposes on_unload and spawn_task, with a running loop required for spawn_task. Initialize lightweight supervision from register(ctx); do not invent plugin startup/shutdown hooks.
- At that revision gateway/run_plugin_injection.py:191-233 returns True after scheduling and logs later failure. Lines 489-518 generate a fresh admission UUID on every call and discard the receipt. Thus True is not durable admission or idempotent retry acknowledgement.
- Capture owner context in the tool call with gateway.session_context.get_session_env. Persist profile, HERMES_SESSION_KEY, original session ID and SessionSource origin fields; pass session_key OR origin to injection, never both. A key can resolve to a new conversation after /new; a plugin-only preflight cannot make routing atomic.

### Must have

1. Preserve the seven existing tools and question/answer behavior; add explicit asynchronous operation and omo_send for corrections.
2. One background owner per plugin profile, owning execution connections, prompts and requester-scoped question frames. Tool handlers submit commands to that owner instead of racing socket readers.
3. Persist job ownership before a prompt, lifecycle revision/run identity, pending questions, input/effect IDs and an outbox. Store no credentials or secret records in reports/database payloads.
4. Reduce authoritative root lifecycle only. Publish terminal state only when restoring=false and persisted=true; deduplicate stable session + epoch/completion identity. Preserve raw state for inspection.
5. Separate ready-for-follow-up, root-task-complete, session-closed and transport-disconnected. A completion.kind=reply alone does not prove a delegated goal's acceptance criteria.
6. Reconnect using session path/durable identity, then reconcile lifecycle/questions. Host restart may interrupt work; do not automatically rerun a possibly applied prompt.
7. Capture the original profile, native session key, original session ID and origin/thread. Dispatch to one validated route; never another chat. Chat/thread routing may deliver into a replacement conversation after /new, which must be documented and labelled with the original session. A strict original-conversation policy disables automatic injection when the API cannot atomically pin that session and leaves the event retrievable.
8. Explicit workspace, configurable permissions and owner-scoped explicit IDs. Parallel jobs need distinct canonical editing directories.
9. Keep completed sessions available for explicit follow-ups. OmO chooses its own execution tools and subagents; Hermes sends a goal, scope, constraints and acceptance criteria.

### Must NOT have (guardrails, anti-slop, scope boundaries)

- This publication changes planning artifacts only; implementation/deployment is not authorized by it.
- No changes to Hermes/OmO core, remote daemon, cross-machine bridge, automatic merges or automatic approval of privileged actions.
- No exactly-once or durable destination-admission claim from SQLite or inject_message=True. A stronger guarantee requires a public Hermes API accepting stable delivery ID plus pinned target and returning a committed receipt; do not modify Hermes core or access private authority objects/databases.
- No cancellation due to time, silence, transport loss or a failed probe. No completion inference from assistant wording, stop reason or hook count.
- No permission escalation, expired approval replay, unscoped latest-session gateway fallback or another conversation's task access.
- No broad cleanup or new task scheduler. New modules separate real contracts: lifecycle reduction, execution ownership, delivery and byte transport.

## Verification strategy

Verification is agent-executed. No implemented product behavior is claimed by this plan's publication.

- TDD with existing Python unittest, temporary databases, fake transports/clocks and deterministic crash injection.
- Task 1 adds tests/run_suite.py to load this checkout as package omo regardless of its directory name. Canonical command: python tests/run_suite.py. Selected modules: python tests/run_suite.py tests.test_lifecycle.
- Explicit real integration command: python tests/run_suite.py --integration. It requires pinned test runtimes/profile, counts executed integration tests, and fails if required tests are skipped or prerequisites are missing. It never addresses a production/personal chat.
- Run on native Windows and Linux/WSL. Platform-specific POSIX tests stay qualified POSIX tests; do not weaken them to simulate Windows support.
- Record versions, exact command, exit code, test counts, redacted event trace and correlation IDs under .omo/evidence/hermes-central-omo-executor/.
- Record baseline failures before changes. Fix regressions; do not hide unrelated failing assertions.
- Happy path: deep task, independent Hermes request, same-origin question/answer, correction, child/background completion and explicit follow-up.
- Failure path: Hermes restart, reconnect after terminal event, host interruption, dispatch lost after True, duplicate injection, conversation reset and expired question. Verify local recoverability separately from destination admission.

## Execution strategy

### Parallel execution waves

File ownership determines parallelism. Small waves are intentional because protocol, tools and storage edits overlap.

- Wave A: task 1 establishes contract fixtures and executable baseline.
- Wave B: tasks 2 and 3, storage versus RPC/lifecycle.
- Wave C: tasks 4 and 7, background ownership versus platform transport.
- Wave D: tasks 5 and 6, Hermes delivery versus tool controls.
- Wave E: task 8 integrates metadata/docs and end-to-end QA, followed by final audits.

### Dependency matrix

| Todo | Depends on | Blocks | Can parallelize with |
| --- | --- | --- | --- |
| 1 | none | 2, 3 | none |
| 2 | 1 | 4, 6 | 3 |
| 3 | 1 | 4, 6, 7 | 2 |
| 4 | 2, 3 | 5, 6, 8 | 7 |
| 5 | 4 | 8 | 6 |
| 6 | 2, 3, 4 | 8 | 5 |
| 7 | 3 | 8 | 4 |
| 8 | 5, 6, 7 | final audits | none |

## Todos

Each task includes implementation and regression tests. Evidence filenames below are relative to .omo/evidence/hermes-central-omo-executor/.

- [ ] 1. Pin contracts and make the test suite reproducible.

  Recommended task executor category: deep-low.
  Files: tests/run_suite.py, tests/test_runner.py, tests/fixtures/rpc/, tests/fixtures/hermes/, contract evidence.
  Work: Capture redacted OmO 5.1.69 capabilities, lifecycle frames, reconnect snapshots, questions and Windows discovery/auth vectors. Pin the tested Hermes revision and actual lifecycle/injection APIs; document the minimum asynchronous version. Build a package-name-independent unittest runner with selected modules and a strict --integration mode. Record the upstream baseline without changing product behavior.
  Closes: GAP-2, GAP-5, GAP-6. Wave A; blocks 2 and 3.
  References: plugin.yaml:1-16; host_client.py:461-483; tests/test_host_integration.py; baseline contract references above.
  Acceptance: python tests/run_suite.py tests.test_runner exits 0; fixture metadata names source revision/version; baseline run records actual counts and pre-existing failures.
  QA happy: run from a checkout named hermes-omo-plugin, successfully importing omo. QA failure: a nonexistent selected module or missing required integration runtime exits nonzero, never an empty green run. Evidence: task-1-contracts.txt.
  Commit: Y | test: pin RPC contracts and add reproducible test runner.

- [ ] 2. Persist ownership, lifecycle and delivery state.

  Recommended task executor category: deep-low.
  Files: sessions.py, tests/test_sessions.py.
  Work: Add additive schema migrations for profile/native conversation routing, stable root identity, lifecycle seq/epoch/completion kind, input acceptance and delivery. Add an outbox with unique event key, origin, payload, lease/attempt/ack state. Update reduced session state and insert its event in one transaction. Use serialized DB transactions or per-thread connections, not an unlocked shared SQLite transaction.
  Contract: completion key = profile + stable plugin job ID + epoch + completion identity; question keys include request ID/revision. Tool session_id is not assumed to equal Hermes' native session key. Verify ownership on every explicit-ID read/write. Preserve uncertain prompt acceptance as distinct from running/done.
  Closes: GAP-2, GAP-3, GAP-6. Wave B; depends on 1; blocks 4 and 6.
  References: sessions.py:11-40,52-150; tools.py:258-264; task 1 routing fixtures.
  Acceptance: python tests/run_suite.py tests.test_sessions exits 0 with migration, duplicate insert, transaction rollback and cross-owner cases.
  QA happy: migrate populated 0.2.8 data and preserve resumable paths. QA failure: crash between state update/outbox insertion rolls both back; another chat cannot access an explicit ID. Evidence: task-2-storage.txt.
  Commit: Y | feat: persist task ownership and delivery outbox.

- [ ] 3. Use authoritative lifecycle for completion and recovery.

  Recommended task executor category: deep-low.
  Files: lifecycle.py, host_client.py, tests/test_lifecycle.py, tests/test_host_client_unit.py.
  Work: Add a pure reducer for authoritative root lifecycle snapshots/events. Ignore old seq values; equal-seq replay creates no effects. Require persisted=true and restoring=false; retain terminal state across disconnect. Associate session/run/epoch and preserve completion.kind. Expose readiness separately from completed delegated work. Extend asynchronous capability checks and retain typed correlated errors/acknowledgements.
  Work: Expose lifecycle, pending questions and input/effect receipts through HostClient. Get result text for the associated run after validating completion; transcript text formats the report but cannot decide terminal state. Task 4 removes handler completion heuristics when integrating this reducer.
  Closes: GAP-2, GAP-4. Wave B; depends on 1; blocks 4, 6 and 7.
  References: host_client.py:649-778; tools.py:688-725; OmO Session lifecycle and task 1 frames.
  Acceptance: python tests/run_suite.py tests.test_lifecycle tests.test_host_client_unit exits 0.
  QA happy: waiting_children -> waiting_background -> persisted root done creates one correct completion. QA failure: agent_settled, old epoch, disconnected transport, restoring or persisted=false creates no false done. Reconnect to done can recover one event. Evidence: task-3-lifecycle.txt.
  Commit: Y | fix: derive task completion from authoritative lifecycle.

- [ ] 4. Give background tasks one persistent execution owner.

  Recommended task executor category: deep-high.
  Files: worker.py, tools.py, tests/test_worker.py, tests/test_tools_unit.py.
  Work: Add one profile-scoped service with start/stop and command queue. Own attached execution connections, prompt admission and requester-scoped question frames. Persist starting state before sending a prompt; use stable input/effect IDs and resolve acknowledgement uncertainty through host receipts rather than a new prompt.
  Work: Async omo_start returns after acceptance/persistence. omo_wait observes stored service state without racing another reader. On startup, reattach active paths and reconcile lifecycle/questions. Distinguish Hermes restart from host restart: report interrupted host work and require explicit continuation. Replace age-based cancellation with informational stale/unknown status and reconciliation. Remove _transcript_settled_text as a completion authority. Retain synchronous CLI operation where supported.
  Closes: GAP-1, GAP-2, GAP-3, GAP-4. Wave C; depends on 2 and 3; blocks 5, 6, 8.
  References: tools.py:178-208,266-306,332-473,688-725; host_client.py:701-738; requester-scoped RPC visibility.
  Acceptance: python tests/run_suite.py tests.test_worker tests.test_tools_unit exits 0; background mode needs no model-generated wait loop.
  QA happy: disconnect the calling tool and later observe questions/completion through the owner. QA failure: crash after prompt acceptance before local ack, restart and prove no duplicate input; old timestamps alone never cancel. Evidence: task-4-background.txt.
  Commit: Y | feat: supervise delegated tasks in the background.

- [ ] 5. Dispatch events to the original Hermes chat with explicit uncertainty.

  Recommended task executor category: deep-high.
  Files: delivery.py, __init__.py, plugin.yaml, tests/test_delivery.py, tests/test_plugin_lifecycle.py.
  Work: Initialize lightweight profile supervision from register(ctx), stop/join via ctx.on_unload, and use ctx.spawn_task only with a verified running loop; otherwise use a managed supervisor thread without blocking registration. Probe gateway availability before dispatch; repeated initialization creates no second owner. Capture get_session_env in the original tool context, validate profile/injection permission, then pass one route (session_key OR origin). No latest-chat/home-channel fallback.
  Contract: pending -> sending -> dispatch_unconfirmed on the current API, with a recoverable sending lease. False/exception leaves pending. True means scheduled, not committed: retain the immutable event, expose delivery_confirmed=false and never mark accepted/delivered or delete it based on that Boolean. No automatic replay after a known True response. Send-at-crash uncertainty remains retrievable with an explicit retry because automatic retry can duplicate admissions. Envelope includes stable event ID and original session identity.
  Contract: a future confirmed path requires a public receipt API with stable ID replay and pinned target. Until present, report durable destination admission as unavailable; never call private authority internals. A /new does not reassign stored ownership. Strict conversation policy disables unpinnable injection; chat/thread policy documents that a late result can appear in that same chat's replacement conversation.
  Work: Persist question ID/revision/expiration; revalidate before presentation/answer, so stale approvals cannot be replayed. Coalesce progress in status. Status/wait always retrieves retained results regardless of injection outcome. Graceful unload disconnects receivers, stops dispatch and then closes storage; it never cancels OmO tasks or invokes omo_stop.
  Closes: GAP-1, GAP-3. Wave D; depends on 4; blocks 8.
  References: __init__.py; plugin.yaml; pinned Hermes source and task 1 fixtures; task 2 outbox.
  Acceptance: python tests/run_suite.py tests.test_delivery tests.test_plugin_lifecycle exits 0.
  QA happy: restart Hermes while OmO works, then dispatch the retained event to the original test chat/thread with visible unconfirmed state. QA failure: False, missing route, expired question, True followed by dispatch failure, crash during sending and /new retain evidence without false delivery success or changed ownership. Test strict-conversation refusal explicitly. Evidence: task-5-delivery.txt.
  Commit: Y | feat: dispatch background events with explicit delivery state.

- [ ] 6. Add scoped corrections and explicit controls.

  Recommended task executor category: deep-low.
  Files: schemas.py, tools.py, questions.py, tests/test_tools_unit.py, tests/test_questions.py.
  Work: Add omo_send(session_id, message, mode, expected_turn_id), with steer/follow_up modes. Use the owner, expectedTurnId and stable input identity; return acceptance disposition and propagate turn_conflict. Never create a new task to hide a conflict. Preserve resume for idle sessions and answer for pending questions.
  Work: Authorize explicit IDs against calling profile/conversation; require reliable gateway origin, retaining an explicitly CLI-only latest-session fallback. Distinguish single-task cancellation from global omo_stop. Do not auto-answer approvals or escalate permissions.
  Work: Remove the author's fixed cwd. Resolve explicit cwd, configured default or reliable CLI context, otherwise return missing-workspace. New installations default to workspace permission while preserving explicitly configured existing presets through migration. Reject parallel edits in the same canonical directory.
  Closes: GAP-4, GAP-6. Wave D; depends on 2, 3, 4; blocks 8.
  References: schemas.py; tools.py:114-126,258-264,332-386,480-632,815-843; questions.py; RPC prompt/steer/follow_up/effect ledger.
  Acceptance: python tests/run_suite.py tests.test_tools_unit tests.test_questions exits 0.
  QA happy: correct a running task and observe the accepted input in the same session. QA failure: stale turn, cross-chat ID, expired approval, shared parallel cwd or missing workspace fails explicitly without creating work. Evidence: task-6-controls.txt.
  Commit: Y | feat: support scoped task corrections and controls.

- [ ] 7. Support authenticated native Windows transport.

  Recommended task executor category: deep-low.
  Files: transport.py, host_client.py, tests/test_transport.py, tests/test_host_client_unit.py.
  Work: Separate byte-stream read/write/close/readiness from framing and lifecycle. Preserve Unix sockets and endpoint-key proof on POSIX. Add a Windows named-pipe backend using standard-library Windows facilities, exact host logical-path normalization/discovery and the raw .secret handshake, verified against task 1 vectors and pinned runtime source. Do not assume Windows AF_UNIX equals OmO's actual transport.
  Work: Read the host secret only from its credential location; never log it. Validate Windows-appropriate ownership/ACL rules. Keep getuid, POSIX chmod repair, /tmp, /proc and signals in POSIX branches. Start/stop via supported host CLI operations or an identity-validated platform process handle, never an arbitrary PID kill. Preserve deadlines and split UTF-8/JSONL framing. Unsupported/missing credentials fail before any prompt.
  Closes: GAP-5. Wave C; depends on 3; blocks 8.
  References: host_client.py:76-177,262-339,379-460,489-610,649-693; OmO Authentication and Windows logical endpoint sections.
  Acceptance: python tests/run_suite.py tests.test_transport tests.test_host_client_unit exits 0 on Windows and Linux; strict integration includes real platform auth/reconnect with no required skip.
  QA happy: authenticate, open, prompt, receive question and reconnect to a retained Windows session. QA failure: wrong secret/unsafe credential location fails closed; POSIX HMAC tests stay equally strong. Evidence: task-7-windows.txt, task-7-posix.txt.
  Commit: Y | feat: support authenticated native Windows RPC.

- [ ] 8. Integrate workflow documentation and end-to-end acceptance.

  Recommended task executor category: deep-low.
  Files: README.md, plugin.yaml, skills/omo-orchestration/SKILL.md, skills/omo-orchestration/references/task-prompt-template.md, docs/acceptance/, tests/test_host_integration.py, tests/test_gateway_integration.py.
  Work: Register final tools and verified unload/task APIs; set evidence-based minimum versions. Document migration, background/CLI behavior, workspace/permissions, co-location, Windows auth, local outbox guarantees and the public injection post-scheduling loss window. Expose deliberate retry/retrieval for unconfirmed events and explain duplicate risk. Do not advertise durable destination admission. Remove wait-loop instructions only from background mode. Keep Goal/Context/Scope/Acceptance/Verification briefs; OmO chooses agents/tools.
  Work: Return compact reports with changed files, commands/exit codes, skipped tests, artifacts and open issues; label OmO-reported versus independently verified evidence. Test deterministic local gateway routing, initialization and injection errors, plus real test-profile Hermes/OmO behavior on both platforms.
  Closes: GAP-1 through GAP-6. Wave E; depends on 5, 6, 7.
  References: README.md; bundled orchestration skill; tests/test_host_integration.py; evidence from tasks 1-7.
  Acceptance: python tests/run_suite.py and python tests/run_suite.py --integration exit 0 on both platforms, with pinned versions, nonzero executed integration counts and correlated events. Missing prerequisite is a failed prerequisite, not a green skip.
  QA happy: long child/background task, independent Hermes request, same-origin question/answer, correction, completion and same-session follow-up. QA failure: Hermes restart, missed terminal event, host interruption, duplicate event, rejected injection and cross-chat control yield no false done, lost local record, duplicate task or approval replay; show delivery uncertainty separately from task success. Evidence: task-8-e2e.txt.
  Commit: Y | docs: document and verify Hermes coordinator workflow.

## Final verification wave

After all implementation tasks, audit the final working tree. Every audit requires concrete evidence; unresolved failures remain open. This publication marks no implementation checkbox complete.

- [ ] F1. Plan compliance audit - match every behavior to tasks/evidence; validate migration and capability/version declarations.
- [ ] F2. Code quality review - transactions, thread ownership, scoped IDs, secrets, shutdown and transport failure paths.
- [ ] F3. Real manual QA - agent-executed Windows/Linux test-profile gateway and OmO sessions, including restart and deep agent work.
- [ ] F4. Ideal-state fidelity - prove all IS rows, delivery guarantees and ready/complete/closed distinctions against real behavior.

## Commit strategy

- This planning publication changes .omo/plans/ and .omo/drafts/ only, in one documentation commit on the fork's main.
- Future implementation uses a feature branch from updated main and one verified behavior-plus-tests commit per task, using the repository's test:/fix:/feat:/docs: style.
- Stage explicit paths; inspect staged diffs; include no unrelated work or credentials.
- Future push/merge/deploy actions require that execution session's authorization. This request authorizes the fork and plan publication only.

## Success criteria

| IS | Delivering todo(s) | Proving QA scenario | Evidence |
| --- | --- | --- | --- |
| IS-1 | 4, 5, 8 | Deep task plus independent Hermes request without wait-loop turns | task-4-background.txt, task-8-e2e.txt |
| IS-2 | 1, 2, 3, 4 | Child/background work, stale snapshots and reconnect after completion | task-3-lifecycle.txt |
| IS-3 | 2, 4, 5, 8 | Hermes restart, question recovery, dispatch loss after True, unconfirmed state and /new | task-5-delivery.txt, task-8-e2e.txt |
| IS-4 | 2, 3, 6, 8 | Same-session correction, stale-turn refusal, scoped cancel, expired approval | task-6-controls.txt |
| IS-5 | 1, 7, 8 | Authenticated Windows/POSIX question and reconnect roundtrips | task-7-windows.txt, task-7-posix.txt |
| IS-6 | 1, 2, 6, 8 | Workspace/permissions, cross-chat refusal and report evidence | task-2-storage.txt, task-8-e2e.txt |

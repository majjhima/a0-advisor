# Conditional implementation plan: Agent Zero Advisor MVP

Date: 2026-10-19
Status: **Draft for plan review. Product baseline approved; feasibility gate incomplete. No BUILD/install/provider authorization implied.**

## 1. Outcome and current evidence

Implement [specification revision 0.4](../specs/advisor-mvp.md): manual transcript-only `last`/`work` review, explicit global preset/slot and provider disclosure, bounded report-only findings, a dedicated per-chat report view, and durable per-chat latest-report storage. No main-agent continuation, core goal dependency, automatic monitoring, or investigative tools.

The [initial feasibility report](../feasibility/initial-boundaries.md) records source findings and three passed extracted-function probes. It does not close framework/UI/concurrency/provider integration gates. Two delegated workstreams failed with quota errors; their unfinished work is not counted as evidence. This conditional plan makes the unresolved proofs explicit instead of declaring implementation-ready status.

All tasks below are **unstarted**. Proposed test/source paths do not exist yet. Tests must use synthetic contexts, fake provider transport, and temporary directories, never real credentials or private chats. Do not install the repository into the live plugin loader merely to run fixtures.

## 2. Architecture decisions

| Boundary | Proposed implementation direction | Constraint/gate |
| --- | --- | --- |
| Ingress | One service reached by deterministic slash command and authenticated plugin API. WebUI resolver intercepts before normal message submission. | Never route Advisor through `context.communicate()`; its active-run branch creates an intervention. Prove active-run handling in F1. Raw slash text sent through arbitrary backends is not a safe promised integration. |
| Command output | Empty replacement text; supported informational effect or separate report response. | Nonempty command result text can re-enter the main model path. No `send_message` effect. Prove WebUI/backend rendering and no history mutation. |
| Job ownership | Plugin-owned background registry, cancellation handles, single-flight admission and generation identity per chat; runtime-wide capacity bound. | Must not use/replace the main context task. All scheduling, stop/reset/delete hooks and event-loop ownership require F1/F2 evidence. |
| Snapshots | Pure extractor produces immutable redacted evidence with request/answer/message IDs, revision inputs, capture time, and digest. | Exact answer fidelity and thread-safe copy cannot be inferred from a history counter. Reject missing anchors; no raw snapshots persisted. |
| Reviewer | Existing framework model wrapper with explicit preset inheritance; plugin-local validated overrides for retry/output limits. | Disable wrapper/SDK/transport retries and fallback. Refuse unsupported configurations. Destination acknowledgment must match resolved identity before dispatch. |
| Persistence | Small schema-versioned JSON sidecar resolved from chat-folder helper; separate latest completed report and minimal outcome/recovery marker. | Atomic write, explicit save failure, safe deletion/fork behavior, no chat export/log pollution, 24 KiB bound. Private core helpers are reference patterns, not assumed stable APIs. |
| UI | Native plugin settings and a dedicated modal/plugin report view, scoped notification, running/status/cancel controls. | Use supported surfaces, auth/CSRF, sanitization, no remote image/resource fetching. UI exact entry point is frozen after F1. |
| Packaging | Plugin payload at this repository root; runtime payload ultimately under `usr/plugins/a0_advisor`, using `usr.plugins.a0_advisor...` imports. | No install/symlink/sys.path mutation now. Establish offline import strategy before BUILD. Local/community choice stays open; no publication. |

No dependency addition is currently justified. Any required setup/update/cleanup belongs in root `hooks.py`; omit `execute.py`. No modifications to `agent.py`, `initialize.py`, command precedence, or core `/goal` are planned.

## 3. Dependency graph and gates

```text
F1 command/background proof ─┐
F2 evidence/lifecycle proof ─┼─ GATE F: evidence + reviewed plan ─ B1 contract
F3 model policy proof ──────┤                              ├─ B2 storage ─ B3 recovery
F4 storage policy proof ────┘                              │
                                                           ├─ B4 last evidence ─ B5 work
                                                           └─ B6 reviewer config ─ B7 model client
B2/B3 + B4/B5 + B6/B7 ─ B8 manual service ─ B9 lifecycle invalidation
B8/B9 ─ B10 commands/API ─ B11 settings ─ B12 report UI ─ V1 acceptance ─ R1 review ─ S1 release
```

After Gate F, independent pure storage/extraction/model work can proceed in parallel only against the frozen B1 contracts. Service, settings, and UI follow those shared interfaces. No parallel editing of shared contracts/state files.

## 4. Phase F — close feasibility before product BUILD

### F1: Prove command and background ingress

- Acceptance: synthetic active-run dispatch leaves main task, intervention, pause, and history unchanged; informational command output never becomes a new user task; background job survives request completion and responds to cancellation.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests/feasibility -p 'test_command_boundary.py' -v`; use fake main agent/provider and the real resolver/extension contract where imports can be isolated safely. Inspect effective namespace with a fixture catalog and later the authorized live catalog.
- Files (up to 3): `tests/feasibility/test_command_boundary.py`, `tests/feasibility/fixture_support.py`, `docs/feasibility/initial-boundaries.md`.
- Dependencies: none. Scope: M. Freeze command result format and UI entry mechanism only after evidence.

### F2: Prove last/work snapshot and lifecycle identity

- Acceptance: extract exact completed text and request pairing from legacy/native fixtures; copy active work without mutating/await-blocking its run; detect new instructions, stop/reset/fork/delete and compaction invalidation through plugin-available boundaries.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests/feasibility -p 'test_history_boundary.py' -v`; fixture replay must include reload, partial/tool-only output, user intervention and changing source identities. An extracted `ResponseTool.execute` test alone is insufficient.
- Files (up to 4): `tests/feasibility/test_history_boundary.py`, `tests/feasibility/history_fixtures.py`, `tests/feasibility/fixture_support.py`, `docs/feasibility/initial-boundaries.md`.
- Dependencies: F1 for event-loop/ingress findings. Scope: M. Failure blocks live `work`; it does not authorize idle-only scope.

### F3: Prove reviewer/provider policy boundaries

- Acceptance: synthetic preset/slot inheritance preserves auth and fails without fallback; instrumented transport proves no hidden retries/fallback and enforces budget/deadline; masking and destination mutation fail safely.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests/feasibility -p 'test_model_boundary.py' -v`; mock network transport at its lowest usable boundary and assert request counts for transient errors. Include token accounting, masking and destination mutation.
- Files (up to 4): `tests/feasibility/test_model_boundary.py`, `tests/feasibility/model_fixtures.py`, `tests/feasibility/fixture_support.py`, `docs/feasibility/initial-boundaries.md`.
- Dependencies: none; coordinate shared fixture changes sequentially with F1/F2. Scope: M. Keep provider-specific supported/unsupported outcomes explicit.

### F4: Prove durable storage boundaries

- Acceptance: filesystem fixtures establish atomic save/recovery and clear/delete behavior; fork/export do not copy the sidecar; write failures preserve a valid previous report or expose an explicitly reviewed commit-point limitation.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests/feasibility -p 'test_storage_boundary.py' -v`; use synthetic folders, interrupted markers, concurrent clear/delete and injected replace/fsync errors. No live chat directory writes.
- Files (up to 3): `tests/feasibility/test_storage_boundary.py`, `tests/feasibility/storage_fixtures.py`, `docs/feasibility/initial-boundaries.md`.
- Dependencies: F2 for deletion/fork hook findings; no shared fixture edits in parallel. Scope: M. Actual deployment durability remains a separate authorized runtime check.

### Gate F — stop and review

- [ ] F1–F4 passing evidence recorded with actual commands/counts; no source-only claim promoted to integration proof.
- [ ] Freeze source/test import strategy, exact plugin hooks/UI surfaces, supported framework revision and provider matrix.
- [ ] Resolve any contract conflict through a reviewed spec/plan update; no silent core edit or scope reduction.
- [ ] Human approves technical plan before B1. Failure of a proof is an architectural finding, not permission to bypass it.

## 5. Phase B — product slices after Gate F approval

Every task follows red/green/refactor: write the focused regression/acceptance test, observe its expected failure, make the smallest implementation, then rerun that test and the accumulated suite. Load incremental-implementation and test-driven-development before BUILD. Proposed filenames may be refined at Gate F without increasing task size.

### B1: Define validated review contracts and offline plugin shell

- Acceptance: bounded schema/enums, host-vs-model fields and reason codes match spec; empty setup produces no model call; runtime imports are testable without live installation.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_contract.py' -v`; validate manifest and defaults with installed YAML support.
- Files (5): `plugin.yaml`, `default_config.yaml`, `helpers/contract.py`, `tests/test_contract.py`, `tests/support.py`.
- Dependencies: Gate F. Scope: M. Covers schema/failure foundations for AC-01/10/19.

### B2: Save/read/clear bounded per-chat reports

- Acceptance: atomic schema-validated state save and authorized chat-bound lookup; corrupt/oversized/unwritable state fails explicitly; save failure preserves prior valid report and clear invalidates callbacks.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_storage.py' -v`; temporary directories and injected write/replace/fsync errors; no real chat folders.
- Files (3): `helpers/storage.py`, `tests/test_storage.py`, `helpers/contract.py`.
- Dependencies: B1, F4. Scope: M. AC-09/18/20/21/22; test directory recreation race before live use.

### B3: Recover interrupted outcomes without restarting jobs

- Acceptance: minimal admission marker is written before dispatch; restart reports interruption and preserves latest completed report; disable/uninstall do not erase saved reports or copy them to forks.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_recovery.py' -v`; reconstruct service state using fixture storage in a fresh object/process and assert zero provider dispatch.
- Files (3): `helpers/recovery.py`, `tests/test_recovery.py`, `helpers/storage.py`.
- Dependencies: B2. Scope: M. AC-12/17/23.

### Checkpoint B3

- [ ] Contract/storage/recovery suites pass; 24 KiB and export/log exclusions hold.
- [ ] No installed payload or real user state touched; failure semantics reviewed before proceeding.

### B4: Review-ready last-answer evidence

- Acceptance: correct answer/request/constraints selected, intermediate tool calls/reports excluded; redaction precedes provider request; required anchors never silently truncate.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_snapshot_last.py' -v`; native/legacy/reload/compaction/missing-answer and secret-shaped fixtures.
- Files (4): `helpers/snapshot.py`, `helpers/redaction.py`, `tests/test_snapshot_last.py`, `tests/history_fixtures.py`.
- Dependencies: B1, F2. Scope: M. AC-04/05/06/15/16.

### B5: Capture bounded active-work evidence

- Acceptance: immutable consistent snapshot during active run; ordinary progress is allowed but labeled newer; new instructions/source changes invalidate without mixing requests/chats.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_snapshot_work.py' -v`; barrier-controlled concurrent fixture mutations and no-answer/empty-work cases.
- Files (3): `helpers/snapshot.py`, `tests/test_snapshot_work.py`, `tests/history_fixtures.py`.
- Dependencies: B4, F1/F2. Scope: M. AC-03/08/16.

### B6: Explicit reviewer configuration and disclosure

- Acceptance: sanitized global catalog with Main/Utility labels; unset/invalid selection has no fallback; acknowledged target/provider/API-origin and policy must still match at dispatch after preset changes.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_reviewer_config.py' -v`; synthetic preset inheritance, immutability, credential isolation, mutation and unsupported endpoints.
- Files (3): `helpers/reviewer_config.py`, `tests/test_reviewer_config.py`, `default_config.yaml`.
- Dependencies: B1, F3. Scope: M. AC-01/11.

### Checkpoint B6

- [ ] Snapshot and config tests pass; no hidden reasoning/keys/raw settings in evidence or catalog.
- [ ] Active-work consistency and historical answer pairing are demonstrated, not inferred.

### B7: One bounded reviewer request

- Acceptance: framework wrapper call honors selected auth/kwargs except policy overrides, bounded input/output, all-layer no-retry policy, deadline/cancellation, schema validation and sanitized failures; no tools supplied.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_reviewer_client.py' -v`; fake transport counts underlying requests for success/transient/invalid/truncated/timeout paths and unsupported configs return before dispatch.
- Files (4): `helpers/reviewer_client.py`, `helpers/budget.py`, `prompts/reviewer.md`, `tests/test_reviewer_client.py`.
- Dependencies: B4/B6, F3. Scope: M. AC-02/05/06/10/11.

### B8: End-to-end manual review service with fake provider

- Acceptance: admission reserves one/chat and two/runtime without queue; orchestrates snapshot/call/save with host-owned IDs; immediate rejections preserve prior records and failed reviews preserve prior completed report.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_service.py' -v`; full last/work fake-provider flows, racing starts/cancel/clear and storage failures.
- Files (4): `helpers/service.py`, `helpers/jobs.py`, `tests/test_service.py`, `tests/support.py`.
- Dependencies: B2/B3/B5/B7. Scope: M. AC-02/03/07/09/10/19/20.

### B9: Stop/invalidation/lifecycle integration

- Acceptance: hook bindings proven in F1/F2 cancel only originating review; late completion cannot publish after stop/new instructions/reset/delete/disable; install/update cleanup is idempotent and preserves report data.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_lifecycle.py' -v`; fixture extension dispatch plus fake active main task, delete/fork and disable/uninstall scenarios.
- Files (up to 5): `hooks.py`, `helpers/lifecycle.py`, two narrow extension adapters at the exact paths frozen in F2, `tests/test_lifecycle.py`.
- Dependencies: B8. Scope: M. AC-08/09/12/17/21/23. If more adapters are needed, split by event family before coding.

### Checkpoint B9

- [ ] Fake-provider manual last/work flow completes and saves across synthetic restart.
- [ ] Cancellation never resumes main agent; quotas, no retries, stale suppression and clear/delete races pass.

### B10: Deterministic commands and authenticated API

- Acceptance: help/last/work/status/cancel/report/clear share service; auth/CSRF/context validation precede disclosure and side effects; no replacement text/intervention/history path.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_entrypoints.py' -v`; resolver/API fixture cases, disabled plugin, invalid context, command collision, and core goal coexistence.
- Files (4): `commands/advisor.command.yaml`, `commands/advisor.py`, `api/advisor.py`, `tests/test_entrypoints.py`.
- Dependencies: B9, F1. Scope: M. AC-01/13/14/19.

### B11: Native settings and disclosure form

- Acceptance: Config discovered; catalog shows selected Main/Utility names without keys; acknowledgment and lowered numeric limits save/reload through native settings and affect runtime.
- Verify: focused API/config tests plus browser save/reload after isolated installation is authorized; use native framework URL discovery rather than assuming a port.
- Files (up to 4): `webui/config.html`, `webui/advisor-store.js`, `api/advisor.py`, `tests/test_settings.py`.
- Dependencies: B10/B6. Scope: M. Load frontend-ui-engineering, browser-automation/form workflow and browser testing skills before UI implementation/verification.

### B12: Dedicated report view and scoped notifications

- Acceptance: readable saved report with evidence/limits/time/historical labels; running/cancel/clear controls; no cross-chat redirection, executable HTML or remote image fetches.
- Verify: browser fixtures cover switch/reload/report retrieval, adversarial output, missing source links and cancel/clear; repeat against the authorized target runtime. Status/report/clear must make zero model calls.
- Files (up to 4): `webui/report.html`, `webui/advisor-store.js`, one WebUI surface adapter at the exact path frozen in F1, `tests/test_report_view.py`.
- Dependencies: B10/B11. Scope: M. AC-12/13/16/18/21. Real browser execution is not replaced by static HTML tests.

## 6. VERIFY, REVIEW, SHIP gates

### V1: Prove the acceptance matrix

- Acceptance: all AC-01–AC-23 mapped to passing tests or explicit release blockers; actual isolated lifecycle/API/browser checks; no claims of provider-backed usefulness from mocks alone.
- Verify: `/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_*.py' -v`, `git diff --check`, named-runtime integration and approved small provider sample. Load debugging-and-error-recovery; invoke test-engineer for coverage/edge-case analysis at VERIFY.
- Files (up to 3): `tests/test_acceptance.py`, `docs/verification/advisor-mvp.md`, `docs/plans/advisor-mvp.md`. Dependencies: B12. Scope: M. Fixes go back to their small owning slices.
- Provider sample requires explicit test consent and budget/model selection. Installation/restarts must use isolated data and management APIs; never interrupt the user's ongoing work as a test.

### R1: Review implementation and security

- Acceptance: code/security findings resolved, no unsupported assertions or unresolved release-critical risk; plan/spec/docs reflect actual behavior.
- Verify: load code-review-and-quality; orchestrator invokes code-reviewer and security-auditor at REVIEW under persona rules. Any multi-persona run uses only the authorized parallel fan-out/merge pattern, not nested reviewer routing.
- Files (up to 2): `docs/verification/advisor-mvp.md`, `docs/plans/advisor-mvp.md`. Dependencies: V1. Scope: S. Review artifacts are evidence, not substitutes for tests.

### S1: Authorized installation/release and rollback

- Acceptance: target runtime and storage mapping verified; normal/repeat install, update, disable/uninstall readiness and cleanup checked; saved user reports preserved; rollback leaves main chat/core goal unchanged.
- Verify: load shipping-and-launch, a0-create-plugin review reference and appropriate distribution reference; install/update only through management APIs after authorization. Discover target URL; record exact payload revision and test results.
- Files (up to 4): `README.md`, `docs/verification/advisor-mvp.md`, `.gitignore`, `docs/plans/advisor-mvp.md`. Dependencies: R1, explicit install/publication decisions. Scope: M.
- No default commit/push/Index PR. Local vs community can remain unresolved until packaging/release; community publishing and license attribution need explicit review.

## 7. Risks and required decisions

| Risk | Mitigation / release block |
| --- | --- |
| Command becomes main-agent intervention | F1 negative test from actual `communicate` behavior; all supported entry points resolve before it. |
| Background job lost with API request, or lifecycle hook unavailable | Prove scheduling/cancellation ownership; stop at Gate F if plugin-only surfaces cannot meet contract. |
| Wrong answer/request or snapshot race | F2 transport fixtures, consistent capture and source identities; never guess or silently make work idle-only. |
| Hidden retries/fallback or unenforceable provider cap | Instrument underlying calls; explicit policy overrides and unsupported-config refusal. |
| Atomic replacement succeeds but later fsync fails | Document/test commit-point semantics; report durability uncertainty explicitly. If preserving the previous report in every stated failure requires stronger storage protocol or wording, resolve at F4/Gate F before BUILD. |
| Clear/delete/fork leaks or recreates reports | Generation invalidation plus source/context checks, race fixtures and lifecycle-owned cleanup. |
| Persistence/transcript evidence leaks | Pre-call masking, bounded normalized sidecar, sanitized rendering and synthetic content-marker log/export tests. |
| Claiming container replacement safety from a mount alone | Inspect actual host deployment configuration/volume lifecycle before that guarantee. |
| Failed delegated research mistaken for completion | Only recorded parent probes count; integration tasks remain unchecked. |

Product approval is settled. Remaining approval is the **technical plan/feasibility gate**, not a request to reapprove the same product choices. Distribution, live installation, provider evaluation and publication have separate authorization boundaries. No new dependency or core edit is planned. Before any staging, exclude `.a0proj`, credentials and generated state explicitly; never `git add .` blindly.

## 8. Current verification status

- Product baseline recorded as approved in specification revision 0.4.
- Three extracted-function probes passed; no live plugin/model/UI tests run.
- Framework source revision and metadata recorded in feasibility report; support matrix not certified.
- All F/B/V/R/S tasks above are unstarted, not completed by writing this plan.
- Plan is ready for review as a conditional roadmap. Gate F must close before product BUILD; unresolved proofs are not waived by approval of later slices.

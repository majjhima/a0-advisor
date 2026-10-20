# Draft specification: Agent Zero Advisor MVP

Date: 2026-10-19
Revision: 0.4 — product baseline approved; feasibility and implementation planning
Status: **Product baseline approved on 2026-10-19. Feasibility checks and technical planning authorized; implementation plan not yet approved.**

“Must” describes the accepted product contract, not an existing implementation. On 2026-10-19 the user approved durable retention, then the five product recommendations: MVP scope, report interface, reviewer setup/disclosure, initial limits, and global configuration. Proceed with offline feasibility checks and technical planning. Distribution remains open. Any material scope change requires approval; do not interpret product approval as authorization to install, make provider calls, commit, publish, or skip the implementation-plan gate.

## 1. Objective and assumptions

Build a standalone Agent Zero plugin that reviews the last completed answer or a bounded snapshot of current chat work against the user's requests and constraints. It should identify unsupported conclusions, missed requirements, and justified partial progress without inventing a new goal system or automatically creating more agent work.

Approved product baseline and architectural constraints:

1. The first release is manual-first, with one reviewer and report-only behavior.
2. Review is transcript-only; no independent repository inspection or external research in the MVP.
3. Reviewer selection uses a saved model preset and either Main (`chat`) or Utility (`utility`), not an agent profile or the active chat's override.
4. No core framework edits, Ghost Judge dependency, or replacement of `/goal` is needed.
5. This project remains the source/documentation repository. A future runtime installation belongs under `/a0/usr/plugins/a0_advisor/`.
6. Durable per-chat report retention and the initial defaults below are approved as the MVP baseline; limits may be revisited after testing through a specification update. Distribution remains undecided and does not block offline feasibility or planning.

This product approval does not authorize installation, provider calls, commits, publication, automatic monitoring, or corrective execution. The technical plan requires review before product implementation.

## 2. Document review

Reviewed project inputs:

| Input | Finding | Planning implication |
| --- | --- | --- |
| `README.md` | Project remains unimplemented; the product baseline is now approved. | Preserve the feasibility/plan gate before BUILD. |
| `docs/ghost_judge_handoff.md` | Substantial prior research, implementation history, concurrency risks, and proposed stages. Explicitly not a specification. | Use as background; revalidate framework interfaces and do not inherit all proposals. |
| `LICENSE` | MIT, copyright 2026 majjhima. | Retain this license; check upstream licenses and attribution before copying any code. |

The handoff is a strong research record, but did not settle report delivery, context/privacy budgets, last-answer extraction, defaults, or lifecycle/retention policy. This specification records the approved product baseline; exact integration mechanisms still require evidence. Recorded Ghost Judge test results apply to that earlier project, not Advisor. Its mocked model tests do not prove real review quality.

Repository state at review: `README.md` already modified; `docs/` and `.a0proj/` already untracked. Preserve existing work. Before any future staging, explicitly exclude local metadata/secrets; never use indiscriminate staging. No ignore-file change is made by this planning document.

## 3. Current framework findings and limits

A read-only developer feasibility review inspected current framework source. These are source findings, **not live end-to-end verification**.

| Concern | Source evidence | Consequence |
| --- | --- | --- |
| Deterministic commands | Command resolver supplies context identity and can return informational effects. However, `AgentContext.communicate` sends active-run input to intervention before the later `_process_chain` hook. | Resolve Advisor before normal message submission through the WebUI resolver/authenticated plugin API. A late chain short-circuit does not establish active-run isolation. |
| Report effects | Command WebUI handling renders `show_markdown` through a notification; backend handling differs. | A toast is not a durable report panel. Do not assume a custom effect is accepted. |
| Presets | `plugins/_model_config/helpers/model_config.py`: `get_presets`, `resolve_preset`, `preset_to_config`, `build_config_from_preset`, `build_model_config`. | Preserve explicit preset selection and inheritance; no fallback to the effective chat model. |
| Preset scope | Current named lookup is global-only; project scope returns no preset and the catalog reports no project presets. | Propose global saved presets for MVP; do not advertise project preset selection. |
| Model slots | `chat` maps to `chat_model`; `utility` to `utility_model`; both are chat-completion models. | Utility also uses `models.ModelType.CHAT`. Preserve provider/auth/kwargs/rate-limit behavior. |
| History identity | `helpers/history.py`: messages have persisted IDs and sequences; history persists a counter. | Useful snapshot inputs, but not a complete revision/staleness contract. Compression/reset/fork behavior still needs proof. |
| Answer fidelity | `agent.py`: durable AI history may contain wrapped tool calls, unlike the displayed final answer. | Never define “last answer” as simply the last `ai=True` message or use only transient `loop_data.last_response`. |
| Settings/lifecycle | `helpers/plugins.py` discovers `webui/config.html`; installer and plugin helpers dispatch lifecycle hooks. | A settings section alone is insufficient. Any required setup/cleanup belongs in plugin-root `hooks.py`, not `execute.py`. |
| Test environment | Latest framework metadata probe finds Python 3.12.4 and pytest available; earlier notes recorded pytest absent. | Keep the dependency-light framework-runtime `unittest` plan; recheck dependencies and do not install without approval. |

Source locations are a starting map, not a frozen compatibility promise. The [initial feasibility evidence](../feasibility/initial-boundaries.md) records three passed extracted-function probes, the active-run ingress risk, model default retries, mount classification, and limitations. These do not prove live command execution, settings persistence, history extraction, effective namespace availability, or provider behavior. Establish supported versions through the [conditional technical plan](../plans/advisor-mvp.md); its feasibility gate remains open.

## 4. Approved product contract

### 4.1 Users, scope, and non-goals

The primary user is someone working in an Agent Zero chat who wants an independent check without starting another task. The supported first-release interface is the authenticated Agent Zero WebUI. Shared command resolution should provide useful plain-text results to backend callers, but mobile/channel integrations and a host CLI are not release requirements.

The Advisor must work without a core goal, a Ghost Judge goal, or any other plugin-specific task store. It must not inspect files, browse, invoke tools, mutate the project, resume the main agent, or add review messages to the main conversation history. Transcript-only findings about code/tests/external facts must distinguish observed evidence from the primary agent's unsupported claims.

### 4.2 Commands and user journeys

Candidate namespace; collision testing is a release gate:

```text
/advisor
/advisor last [optional review criteria]
/advisor work [optional review criteria]
/advisor status
/advisor cancel
/advisor report
/advisor clear
```

| Invocation | Required behavior |
| --- | --- |
| Bare `/advisor` or an unknown subcommand | Return help/usage without a reviewer call; never silently submit text to the main model. |
| `last` | Review the most recent completed user-visible answer, paired with the request that produced it. A newer request already present at invocation does not block selection; its unfinished answer is never selected. |
| `work` | Review a fixed snapshot of the current request and visible work available at capture time. It may be captured during an active run if the feasibility proof establishes a consistent, non-mutating snapshot. |
| `status` | Report this chat's job state, selected preset/model labels, and latest report availability; do not return credentials or another chat's state. |
| `cancel` | Idempotently cancel only this chat's review. Do not stop or resume the main agent. Best-effort transport cancellation may not prevent provider billing. |
| `report` | Show the latest retained report in this chat, or a clear empty state; no reviewer call. |
| `clear` | Cancel this chat's review and drop its retained report. Late callbacks must not recreate cleared data. |

Review criteria are plain text, at most 2,000 Unicode characters. Reject oversized criteria and invalid/missing context before side effects. Criteria guide the review but cannot grant tools, override privacy, or redefine another request's requirements. For `last`, show the originating request identity explicitly when the chat has since moved on. Starting a review is an explicit manual action, never a task for the main model to interpret.

The WebUI must show a running indicator, selected preset/provider/model, review scope, and a Cancel control. Completion creates a short chat-scoped notification with access to a dedicated report view. Notifications must not change the active chat or expose report details while another chat is active. The report view may be modal or a plugin page; its exact mechanism must use supported plugin surfaces rather than inventing command effects.

For non-WebUI callers, `report` returns a bounded textual rendering of the same retained record. Live execution/scheduling and delivery parity remain integration proof obligations, not established capabilities. Core commands keep their ownership; a namespace collision requires an explicit spec revision, not precedence changes.

### 4.3 What the reviewer assesses

| Assessment | Intended meaning |
| --- | --- |
| `satisfactory` | No material issue identified within the supplied evidence; not a correctness certification. |
| `progressing` | Work is appropriately incomplete and appears consistent with the request. |
| `needs_attention` | At least one material, evidence-supported gap needs attention. |
| `blocked` | A supplied dependency, constraint, or missing user decision prevents reasonable progress. This alone is not an agent failure. |
| `insufficient_evidence` | The provided context cannot support a useful assessment. |

`nit` means a non-material issue; `concern` means a meaningful quality/requirement risk; `blocker` means an evidenced obstacle to the stated objective or a material constraint violation. No severity triggers execution or bypasses limits. Missing final polish during active work is not automatically a finding. A report may have no findings; it must not invent them to fill a quota.

### 4.4 Report and job contract

Service outcomes are distinct from the reviewer's assessment. Local validation errors never become an unfavorable assessment.

| Field | Required contract |
| --- | --- |
| `schema_version` | Integer `1` for this candidate contract. |
| `review_id`, `scope`, `trigger` | Host-generated identity, `last` or `work`, and `manual`; never model-selected. Rejected invocations have no admitted job ID. |
| `status` | Terminal outcome: `completed`, `unavailable`, `cancelled`, `stale`, `no_answer`, `insufficient_context`, `busy`, `invalid_request`, or `setup_required`. |
| `reason_code` | Host-validated, sanitized explanation category. Examples: `invalid_preset`, `timeout`, `invalid_output`, `context_too_large`, `source_unavailable`. No raw provider exception text. |
| `assessment` | Required only for `completed`; absent/null for all other outcomes. |
| `summary`, `findings`, `next_step` | Validated model content, advisory only. A finding has severity, description, source evidence IDs, and a suggested action. |
| `snapshot` | Host-generated context/request/answer IDs where applicable, history boundary, capture time, and a digest of the actual bounded/redacted evidence. |
| `reviewer` | Sanitized preset, slot, provider, and model labels captured at dispatch; no keys, headers, or credential-bearing URL parameters. |
| `verification_limits` | Explicit transcript-only limitation, omitted/truncated context, and evidence that was not independently checked. |

Presence and validation rules:

- Every admitted job outcome has `schema_version`, `review_id`, `scope`, `trigger`, and `status`. Immediate rejections use the same response envelope but `review_id` is null. Help/status/report/clear responses are command results, not new reviewer results.
- For `completed`, `reason_code` is null and assessment, summary, findings, next_step, snapshot, reviewer, and verification_limits are present. For other outcomes, assessment, summary, findings, and next_step are null; reason_code is non-null. Host snapshot/reviewer metadata may be retained only if that admitted job actually captured/resolved them; otherwise they are null. No unvalidated model content is retained on failure.
- Reviewer output contains only assessment, summary, findings, next_step, and verification_limits; host operational fields cannot be supplied or overridden by the model. Unknown/extra keys are rejected.
- Summary: at most 1,000 Unicode characters. Each finding: description at most 500, suggested action at most 300, and 1–4 evidence IDs. Next step: at most 300 characters (empty is allowed). Verification limits: at most 8 strings of 400 characters each, including host-added limitations. Findings may be empty. Evidence IDs must match the bounded bundle.
- Operational reason codes are a closed set for schema version 1: `not_configured`, `disclosure_required`, `invalid_preset`, `invalid_slot`, `invalid_arguments`, `context_unavailable`, `source_unavailable`, `context_too_large`, `unsafe_context`, `capacity_busy`, `capacity_unavailable`, `timeout`, `invalid_output`, `provider_error`, `unsupported_configuration`, `user_cancelled`, `source_changed`, `plugin_disabled`, `runtime_interrupted`, and `storage_error`. Authorization errors are returned through framework access controls without leaking context metadata. Adding a code requires a documented schema-contract update.

Progress state is separate: `idle -> preparing -> running -> terminal`. No queue is created. Admission creates a job only after basic input/context authorization and setup checks. Immediate `busy`, `invalid_request`, or `setup_required` rejections update neither the active job nor retained records. A failed/cancelled admitted job updates the latest job outcome but must not overwrite a prior completed report; show those records separately. Repeating report/status/help makes no admission or provider call.

Only the host creates identities and operational outcomes. Accept exactly one structured reviewer result with valid enums and bounded fields; do not execute embedded code or use evaluation-based parsers. Invalid schema, unknown evidence IDs, excess findings, truncated output, or unsupported model output yields `unavailable`/`invalid_output`; no repair-model call or automatic retry.

Store only the normalized, masked report, not raw model output/reasoning. Escape text and sanitize Markdown; disable executable HTML, remote images, and automatic resource fetching. Opening an external link must be a deliberate user action.

## 5. Approved privacy, limits, and state rules

### 5.1 Initial approved defaults

These are approved initial product defaults, not measured performance claims. The stated limits bound a request; they do not guarantee a monetary cost because provider pricing varies. Revisit them after testing through an explicit specification update.

| Setting/policy | Approved initial default | Runtime effect |
| --- | --- | --- |
| Reviewer preset | Unset | Return `setup_required` until the user selects an existing global saved preset. No implicit use of `Default` or the current chat model. |
| Model slot | `utility` | Use that slot of the explicitly selected preset; Main (`chat`) remains selectable. |
| Provider disclosure | Not acknowledged | Block dispatch until configuration acknowledges the resolved provider/model and applicable review-context policy. |
| Timeout | 60 seconds | End-to-end deadline from accepted invocation, covering preparation, rate-limit waiting, and provider response. |
| Input budget | At most 12,000 model tokens | Includes reviewer instructions, criteria, evidence labels, constraints, and transcript. Apply the lower supported model-context limit after reserving output and transport overhead. |
| Output budget | At most 2,000 model tokens | Apply through the framework's supported provider configuration; fail explicitly if the required limit cannot be honored. |
| Findings | At most 8 | Reject invalid/oversized reviewer results rather than silently hiding findings. |
| Attempts | One underlying provider request | At most one dispatch; disable framework/SDK/transport retries for this call. If a supported configuration cannot guarantee this, return `unavailable`/`unsupported_configuration` without dispatch. No fallback, screening, or schema-repair call. |
| Per-chat concurrency | One active review, no queue | Repeated starts return `busy`; cancel before starting another. |
| Plugin-wide concurrency | Two active reviews, no queue | A process-local capacity guard bounds concurrent work across chats, without mixing their content/state. It is not a global goal/report store. |
| Retention (approved) | Durable per-chat sidecar | Latest completed report plus minimal admitted-job outcome/recovery metadata. Immediate rejections update neither record. Active jobs, locks, and raw review snapshots remain memory-only. |

Only preset, slot, acknowledgment, timeout, input budget, and output budget need settings UI. Numeric settings may lower these MVP ceilings, not raise them. Invalid values fail validation; no silent widening. Findings, concurrency, attempt count, and retention are fixed MVP policies, not speculative settings.

“Active” means an admitted job in `preparing` or `running`, counted against both concurrency limits. Reserve capacity atomically before snapshot capture or dispatch; release it exactly once at termination. If concurrency cannot be bounded, return `unavailable`/`capacity_unavailable` rather than dispatching. Pure setup validation before admission must make no provider call. The deadline starts at admission. Timeout must also release capacity and discard late results.
These are local admitted-job limits, not a guarantee about remote provider processing after cancellation. Do not dispatch another request for the same job. The retained job outcome contains operational metadata only, not a duplicate of the completed report or raw snapshot. Clearing a chat removes both retained records.

Use global plugin configuration for the MVP; no project/profile configuration overrides. Framework plugin enable/disable scopes remain respected, but enabling the plugin never starts reviews. Configuration edits affect only future reviews except disabling the plugin, revoking acknowledgment, or changing the reviewer target, which cancel outstanding reviews. Snapshot all other effective configuration at job admission.

Preset names are references, not stored copies of secrets. Build model configuration with the framework inheritance/auth/kwargs behavior intact. Before every dispatch, resolve the selected preset again and verify its sanitized provider/model/API-origin identity against the acknowledgment. A changed destination requires renewed acknowledgment and no call occurs. The acknowledgment describes destination and context policy, not approval of any provider's privacy/retention practices. Do not claim local cancellation deletes provider-held data.

### 5.2 Evidence selection and masking

Construct a structured evidence bundle from allowlisted user-visible history. Never send hidden reasoning, raw system prompts, raw settings, credentials, private unrelated chats, or arbitrary workspace files. Relevant project constraints may come from applicable instruction text, but must pass the same minimization and masking policy; do not dump the project metadata directory.

Required anchors are the originating request, explicit applicable constraints/corrections, optional criteria, and (for `last`) the exact completed answer. Keep these intact; if they cannot fit safely, return `insufficient_context` without a provider call. Optional older context and visible tool evidence are added newest-relevant-first, retaining chronological order in the final bundle. Drop whole optional items to fit; list omissions and never truncate silently. Budget accounting must use a tested tokenizer/estimation path with a documented safety margin for supported providers; unsupported accounting is not permission to exceed the budget.

For `work`, the active request is required and the available work may be partial. No meaningful request/work yields `insufficient_context`. Never infer a deleted or compressed answer from an unverified summary; if `last` cannot recover an exact completed answer, return `no_answer` with `source_unavailable` and no provider call. Fixtures must cover legacy and native response-tool final-answer shapes.

Mask before token accounting and before transmission. Do not expand secret aliases. Redact known secret values available through an approved local masking facility and common credential-shaped patterns; omit known credential-bearing tool records. If safe handling cannot be established for a required item, fail closed. Pattern masking is best-effort and cannot guarantee recognition of every secret; the UI/docs must state this limitation. No model-driven masking pass is allowed.

Assign host-generated evidence IDs to each included item. Findings must refer to IDs present in the submitted bundle. Reports may retain short redacted excerpts (at most 300 characters per reference), not full transcript copies. Evidence links resolve only within the originating chat and must handle missing history with a clear unavailable state.

### 5.3 Snapshot and cancellation semantics

A snapshot is immutable. Capture context identity, originating request/answer IDs, included-message IDs/sequences, relevant constraint/config fingerprints, and a digest of bounded/redacted content. The exact concurrency-safe capture mechanism must pass an offline/live integration proof; a sequence counter or timestamp alone is insufficient.

| Event after capture | Required outcome |
| --- | --- |
| Main agent appends ordinary progress to the same request | The fixed snapshot remains reviewable. Label the report “as of” capture time and show that newer work exists; never present it as a judgment of the newer answer. |
| New user instructions, edited applicable constraints, or a different main request arriving after capture | Invalidate the job, cancel best-effort, and suppress any actionable completed result. This does not retroactively disqualify an answer selected when a newer request was already present before capture. |
| History reset/edit/compaction removes or changes required source identity | Return/mark `stale`; do not reconstruct missing anchors from guesswork. |
| Fork/clone | Do not copy jobs/reports into the new context. The original review remains bound to the original context if still valid. |
| User stops the main agent or cancels the Advisor | Cancel the originating chat's review; never resume either agent. |
| User switches/closes a view | Do not move the report or cancel solely because another chat is viewed. Delivery stays bound to the original chat. |
| Plugin disable or runtime restart | Cancel/drop active work, retain saved reports, and never resume an interrupted review. Disable is not a request to erase user data. |
| Chat deletion | Cancel active work and remove its saved Advisor state; late callbacks must not recreate the deleted directory. |
| Plugin uninstall | Cancel owned work and clean up runtime registrations/resources. Preserve saved user reports by default; document explicit deletion separately. |
| `clear` then a late provider completion | Discard it using an invalidated job/generation identity. |

Check validity immediately before dispatch, before publishing a completed result, and when displaying it. A new message arriving during preparation must prevent a call if it invalidated the request. Cancellation/clear must remain terminal even when a provider ignores cancellation. Admission, active-job ownership, and terminal transitions must be atomic enough to avoid duplicate calls or late report replacement.

A previously completed report can remain visible as **historical** after new work, with its original scope/time and an outdated/source-unavailable badge. It must not be relabeled as a fresh judgment. A completed result already known stale at publication must never be promoted to the retained completed report.

### 5.4 Retention and authorization

**Approved direction:** store the latest completed normalized report and minimal job outcome/recovery metadata in a plugin-owned per-chat JSON sidecar, proposed at `/a0/usr/chats/<chat_id>/advisor/state.json`. Resolve the chat folder through the framework helper, not an arbitrary caller-supplied path. No database or new dependency is required. Keep this data out of `context.data`, conversation history, and content logs.

Store only bounded, validated, masked report content, host identities, schema version, and minimal operational metadata. Never persist credentials, hidden reasoning, full transcript snapshots, raw reviewer output, provider-call objects, locks, or resumable execution instructions. Transient snapshots are released when a job terminates. The saved state envelope is bounded to 24 KiB of serialized UTF-8; overflowing results fail with `unavailable`/`invalid_output`, preserving the earlier valid report.

| Event | Approved retention behavior |
| --- | --- |
| Switch chats or refresh/close/reopen browser | Saved reports remain associated with the originating chat. |
| Backend/container service restart, crash, or update | Reload the saved report if the same chat storage survives. Never resume a provider call automatically. |
| Restart during review | Preserve the previous completed report. Use a minimal admission marker to identify an interrupted job on recovery and record `cancelled`/`runtime_interrupted`; the marker contains no raw evidence or execution state. |
| Disable then re-enable | Retain saved reports; cancel active reviews and make no calls while disabled. |
| `/advisor clear` | Cancel active work and delete the chat's Advisor sidecar/state. Late completions must not recreate it. Report deletion failures explicitly. |
| Delete chat | Remove its report with the chat folder and prevent late recreation. |
| Fork, clone, or JSON import/export | Do not copy reports into a different context. The separate sidecar is not automatically included in JSON chat export; export integration is deferred. |
| Plugin uninstall | Preserve saved reports as user data unless the user explicitly requests deletion. Required runtime cleanup still belongs to lifecycle hooks. |

Use schema-versioned JSON and atomic replacement in the same directory with flushed data/durability handling appropriate to the target filesystem. Validate file size, schema, and originating context on load. A malformed, incompatible, or unreadable sidecar must not wedge the chat, start a review, or be silently overwritten; show a sanitized storage error. An unwritable store prevents admission/provider dispatch. A completion-save failure must be visible, preserve the previous valid saved report, and must not label a new result as durably saved. Use `unavailable`/`storage_error` for storage failures; deletion and non-review commands return their equivalent command error. Never expose raw paths containing secrets or raw exception details.

Minimal admission markers allow recovery status, not job resumption. Prove atomic ordering of admission/completion writes, generation invalidation, clear/delete, and late callbacks so earlier or interrupted work cannot overwrite a newer valid result. Revalidate source identity on report access; restored reports can be historical or source-unavailable rather than current judgments. Restrict filesystem access to the framework service account; persistence does not imply encryption at rest.

“Restart” means replacement/loss of the owning framework Python process, including a container restart. A normal container restart retains its filesystem, but container recreation/replacement requires a persistent volume or bind mount covering chat storage. Read-only `findmnt` classifies the covering mount as `/a0/usr xfs`; host/Docker deployment configuration and recreation policy remain unverified. Verify those before claiming recreation durability. Backups covering the chat folder include the sidecar; ordinary JSON chat export does not. Local clear/delete does not erase external backups or provider-held records and is not forensic erasure.

Use existing authentication/CSRF and context authorization. Possession of a context ID is not permission to read a report. No unauthenticated report URLs, cross-chat global report list, automatic file export, external telemetry, or content-bearing debug logs. If reliable lifecycle cleanup or snapshot validity cannot be achieved through ordinary plugin surfaces, stop at the feasibility gate and revise the specification rather than patching core behavior silently.

Verify storage and export boundaries with code inspection **and** synthetic end-to-end tests: a nonsecret marker placed only in fixture reviewer output must appear in the approved sidecar and authorized report view, but not in logs, `chat.json`, or JSON chat export. Raw snapshots and credential fixtures must never appear in the sidecar. Original chat persistence is unchanged. Verify restart recovery, disable/re-enable, clear/delete races, save failures, and corrupt-state handling. Do not use real private content as test markers.

## 6. Technical shape, structure, and style

Proposed stack: existing Agent Zero plugin APIs, framework Python, framework model wrappers, native command infrastructure, and Alpine.js/native plugin settings. No additional dependency is currently justified.

Proposed source layout, **not yet created**:

```text
plugin.yaml                 Plugin identity and supported configuration scopes
hooks.py                    Only if required lifecycle work exists
commands/                   Unique command definitions and deterministic entrypoints
helpers/                    Review contract, snapshot extraction, preset client, orchestration
prompts/                    Transcript-only reviewer rubric and structured output contract
webui/config.html           Preset/slot selection and only settings that affect runtime
webui/                      Approved report view and minimal supporting UI
api/                        Only endpoints required by the approved report/settings design
tests/                      Framework-runtime unittest suite
docs/specs/                 Reviewed specifications
```

Plugin-local imports in an installed payload use `usr.plugins.a0_advisor...`; do not import installed Ghost Judge helpers or mutate `sys.path`. API handlers retain framework auth and CSRF. Keep pure contract/extraction logic testable separately from runtime imports. Validate input before side effects.

Style example, illustrative only:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SnapshotIdentity:
    context_id: str
    request_id: str
    answer_id: str | None
    history_sequence: int
```

This illustrates immutability/type hints, not a sufficient final staleness schema. Follow existing framework patterns and avoid speculative abstractions.

### Commands and verification environment

Read-only commands usable now, from the project root:

```bash
cd /a0/usr/projects/a0_advisor
git status --short
git diff --check
```

Future test command, only after test files exist:

```bash
cd /a0/usr/projects/a0_advisor
/opt/venv-a0/bin/python -m unittest discover -s tests -p 'test_*.py' -v
```

There is no plugin build/dev command or test suite today. Do not report a zero-test run as validation. Discover the actual live WebUI URL when integration testing; do not assume a port or start another server for this planning task.

## 7. Testing strategy and success criteria

Use deterministic unit tests first, then framework-boundary tests, then real UI/provider acceptance checks. Mocked transport tests do not establish useful reviewing behavior. No numeric coverage target is justified yet; cover the contracts and failure paths below.

MVP acceptance proposal:

1. `last` and `work` operate in the correct chat without a goal and without starting/resuming the main agent. Another chat's content never appears.
2. Last-answer extraction works for both supported transport/history shapes, including native response-tool calls; intermediate messages and reports are not selected. Reload/compaction limitations are tested or explicitly unsupported with a clear status.
3. Config selects an existing global preset and labels Main/Utility models. An unset preset yields `setup_required`; explicitly invalid/unknown/blank preset selections or invalid slots are rejected with `unavailable` and the appropriate reason code, never a fallback. Provider credentials and kwargs remain correct without appearing in reports, except the explicit reviewed overrides needed to enforce retry/output limits.
4. Context is bounded and masked before transmission. Prompt-injection examples cannot enable tools, execution, or continuation. Oversized, invalid, timed-out, and cancelled reviews terminate predictably.
5. In-progress work and missing evidence are reported fairly. Findings reference supplied evidence and identify unverified repository/test/external claims.
6. Duplicate clicks, concurrent chats, new messages, reset/fork, stop, and stale completion are covered. Core `/goal` remains independent and unmodified.
7. Config visibility, catalog labels, save/reload, command execution, and the chosen report surface pass real browser checks. Every exposed setting demonstrably changes behavior.
8. A small approved real-provider sample checks report parsing, evidence grounding, and the manual flow. Record cost/latency and all unverified limits separately from mocked results.

### Acceptance scenarios required before release

| ID | Given / when | Then |
| --- | --- | --- |
| AC-01 | No selected preset or no matching acknowledgment; user requests a review | `setup_required`, zero provider calls, no main-agent/history side effect. |
| AC-02 | A valid completed answer and configured reviewer; user invokes `last` | Exactly one dispatch and at most one underlying provider request use the selected slot and anchored request; a validated report is available in the originating chat. Inherited SDK retries are disabled or dispatch is refused. |
| AC-03 | An active run; user invokes `work`; more tool/model progress arrives | Consistent bounded snapshot, no blocked/mutated main run, “as of” report with newer-work indication. |
| AC-04 | No completed answer, only an intermediate tool call or compressed summary | `no_answer`, zero provider calls; no guessed answer. |
| AC-05 | Required anchors exceed the effective input limit | `insufficient_context`, zero provider calls; optional omissions alone are disclosed. |
| AC-06 | A known secret, alias, and credential-shaped value appear in visible evidence | Provider-bound fixture contains redaction/omission, never expanded credentials; unsafe required input fails closed. |
| AC-07 | Two starts race in one chat; a third chat exceeds plugin capacity | At most one admitted active job per chat and two globally; rejected starts return `busy` without changing another job/report. Each admitted job dispatches at most once. |
| AC-08 | New instructions/stop/reset invalidate an in-flight review; provider later completes | No fresh completed report is published, no main-agent continuation, and late callbacks cannot revive the job. |
| AC-09 | `clear` races with result publication or context deletion | No report reappears; cancellation affects only the originating chat. |
| AC-10 | Reviewer times out, emits malformed output, unknown evidence IDs, or exceeds limits | Sanitized `unavailable` outcome; no retry/fallback, assessment, or replacement of the previous valid report. |
| AC-11 | A valid preset changes provider/model/API origin after acknowledgment | Dispatch is blocked pending renewed acknowledgment; keys are never sent to the browser/report. |
| AC-12 | Browser reload or backend/container restart with retained chat storage; user requests `report` | Latest durably saved report reloads with original identity and historical/staleness labels. Interrupted work is marked `cancelled`/`runtime_interrupted`; no call resumes. |
| AC-13 | An authenticated request names an unauthorized context or malicious rendered content | Access is denied before disclosure; report rendering executes no HTML/script and fetches no remote image. |
| AC-14 | Core `/goal` is enabled and no goal exists | Manual review remains available and never creates/modifies a goal. |
| AC-15 | An earlier completed answer exists but a newer user request has started | `last` pairs the earlier answer with its actual request and labels the scope; it is not judged against invented requirements from the newer request. |
| AC-16 | A completed report is displayed after newer work or lost history anchors | It is visibly historical/outdated or source-unavailable, never presented as current validation. |
| AC-17 | Plugin is disabled then re-enabled | Active reviews are cancelled, saved reports remain, and no previous work resumes or makes calls while disabled. |
| AC-18 | A synthetic marker exists only in normalized fixture reviewer output; save/export and logging paths run | Marker is stored only in the approved sidecar and authorized report response, not `chat.json`, JSON chat export, or logs. Raw snapshots and credentials are absent from the sidecar. |
| AC-19 | Immediate invalid/busy/setup rejection follows a prior completed report | Neither retained record is overwritten. A later admitted failure updates only the job outcome, not the completed report. |
| AC-20 | Completion write is interrupted or fails; an earlier valid report exists | Atomic storage preserves the previous valid report; failure is explicit and no unsaved result is presented as durable. Admission write failure causes zero provider calls. |
| AC-21 | Clear/chat deletion races with a late result; storage is reloaded | Saved Advisor data stays deleted, no callback recreates it, and no cross-chat path can be written. |
| AC-22 | Sidecar is malformed, too large, incompatible, or unreadable | Sanitized storage error without chat failure, silent overwrite, or provider dispatch. Clear remains available subject to filesystem permission. |
| AC-23 | Chat is forked/cloned/exported/imported, or plugin uninstalled/reinstalled | No report is inherited by a different context; JSON export omits the sidecar. Uninstall stops runtime work but preserves user reports for the original chat unless deletion was explicitly requested. |

Future lifecycle tests must cover normal installation, repeat/update safety, disabling with active work, and cleanup through management APIs if the implementation owns resources. Release also requires an approved small real-provider evaluation: supported output parses, findings cite real included evidence, progress is not mistaken for failure, and transcript-only limits are displayed. A universal correctness/accuracy claim is not a release criterion. No provider-backed test, plugin installation, or browser workflow was run for this revision.

## 8. Initial sequencing and approval gates

The product baseline is approved. A [conditional technical plan](../plans/advisor-mvp.md) now defines proof tasks, implementation slices, dependencies and checkpoints; it is not approved for BUILD, and the feasibility gate remains incomplete. The roadmap below retains those boundaries.

| Gate | Next outcome | Required evidence |
| --- | --- | --- |
| A — Agree scope | Complete: manual/report-only product baseline and initial defaults approved. | User approval recorded in section 10 on 2026-10-19. |
| B — Prove boundaries | Validate command/report delivery without a reviewer model; preset resolution; history extraction/identity; settings discovery. | Offline synthetic fixtures show no main-agent invocation, no history mutation from reporting, correct identities, and no model fallback. Use `strip_api_key=True` for display/catalog data; retain runtime credentials securely for actual model construction. |
| C — Plan slices | Conditional draft exists; finalize against evidence after B. | Small tasks have acceptance, verification and dependencies. Proof closure and human plan approval precede BUILD. |
| D — Implement/verify | Tests-first manual `last` flow, then `work`, settings/report UX, and failure/concurrency hardening. | Unit and framework tests plus actual browser and approved provider checks. |
| E — Review/ship | Review code, security, and coverage at the appropriate lifecycle stages; choose installation/distribution and rollback. | Review findings resolved; lifecycle/runtime checks; explicit authorization for commits or publication. |

The risky command-delivery and last-answer boundaries come before provider integration. Once interfaces are approved, independent tests/docs can proceed separately; shared snapshot/report/state contracts must be settled before parallel implementation.

Later, separately approved stages: opt-in per-chat automatic advisory; isolated read-only investigation; bounded corrective work. Defer multi-reviewer rosters, screening models/Jev integration, tool-step monitoring, and pre-delivery gating. None is implied by approval of this MVP.

## 9. Boundaries

- **Always:** preserve user control, bind reviews to snapshots, minimize/mask input, use framework model/auth/config helpers, test error paths, distinguish evidence from inference, and disclose verification limits.
- **Ask first:** dependencies, provider disclosure/budgets, changes to approved report retention, automatic activation/correction, core-file changes, installation, commits, pushing, and community publication.
- **Never:** commit secrets/private transcripts or `.a0proj/`, silently substitute a reviewer model, override `/goal`, share global goal/review state across chats, execute reviewer advice automatically, or equate a model judgment with proven correctness.

## 10. Approval checklist and remaining proof obligations

### Approved decision

- [x] Durable per-chat storage for the latest completed report and minimal outcome/recovery metadata; retain across chat switching, browser reload, backend/container restart with surviving storage, and disable/re-enable. Keep live jobs temporary; never auto-resume. Clear/chat deletion removes saved state. See section 5.4 for storage, backup/export, and failure boundaries.

### Product recommendations approved on 2026-10-19

- [x] Manual-first, single-reviewer, transcript-only, report-only scope; live `work` is a snapshot, not monitoring.
- [x] Dedicated per-chat report view and short scoped notification. Exact native UI integration must be proved.
- [x] Explicit reviewer setup and destination/context-policy acknowledgment, with no implicit preset selection.
- [x] Initial ceilings: 60 seconds, 12,000 input tokens, 2,000 output tokens, 8 findings, one attempt, one job/chat and two jobs/runtime; revisit after testing.
- [x] Global-only configuration/preset selection for MVP; no project/profile overrides or automatic activation.

The local-versus-community distribution choice remains unanswered from the previous planning request. It stays open; no publication, repository restructuring, or Index submission is authorized. Functional specification work can continue independently of that choice.

### Proof obligations, not decisions to hide in code

1. Reserve/check the candidate command namespace and prove both WebUI/backend invocation avoid the main-agent task/history path.
2. Prove consistent non-mutating `work` capture during an active run, safe background scheduling, and cancellation/stop/cleanup using plugin-only surfaces. If impossible, present a proposed scope change for approval; do not silently restrict `work` to idle chats.
3. Prove exact completed-answer extraction across supported history transports, reload, and compaction; define any explicit unsupported cases.
4. Prove destination acknowledgment cannot be bypassed by preset mutation, and that supported providers honor the required context/output limits.
5. Establish the supported Agent Zero revision range and native report/settings integration; prove atomic sidecar writes/recovery, chat delete/fork isolation, disable/uninstall retention, and deployment storage durability. No live compatibility or deployment-mount claim exists yet.
6. After product approval and feasibility evidence, write and approve the technical implementation plan/tasks before BUILD.

Specification approval settles behavior, not implementation verification or authorization to install dependencies, make provider calls, deploy, commit, or publish.

## References

- [Project overview](../../README.md)
- [Conditional technical plan](../plans/advisor-mvp.md)
- [Initial feasibility evidence](../feasibility/initial-boundaries.md)
- [Ghost Judge planning handoff](../ghost_judge_handoff.md)
- `/a0/skills/a0-create-plugin/SKILL.md` and implementation/channel-command references
- `/a0/skills/a0-development/SKILL.md`
- Framework source owners listed in section 3

Original handoff history and upstream research are preserved unchanged. No upstream code was copied, no plugin was implemented/installed, and no runtime state was modified by this document.

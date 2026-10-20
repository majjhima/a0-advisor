# Advisor initial feasibility evidence

Date: 2026-10-19
Status: **Partial evidence; integration gate remains open. No Advisor implementation or installation.**

## Scope and provenance

Product baseline: [specification revision 0.4](../specs/advisor-mvp.md). Framework checkout inspected: `/a0`, HEAD `ff2a5b9c9c4e490ba38dd7c313e779510bb2071a`. This identifies a source checkout, not a certified supported release or proof that every working-tree file matches that commit.

Two delegated developer workstreams failed with provider quota errors before delivering their reports. No results from those failed workstreams are counted as verification. Parent fallback performed the source inspection and three extracted-function probes below. No live chats, stored presets, credentials, provider requests, management APIs, or lifecycle hooks were exercised by the fallback. No plugin payload was installed.

## Source findings

| Boundary | Evidence | Consequence |
| --- | --- | --- |
| Active-run command safety | `/a0/agent.py:269-286`: `communicate()` unpauses and assigns a message to `intervention` when its task is alive. The bundled resolver extension is at `_process_chain/start`. | A later chain short-circuit is not a safe active-run entry point. Invoke Advisor through the WebUI pre-send command resolver or authenticated plugin API before `communicate()`. Other backends must call the command resolver directly, not send raw slash text through the main agent. |
| Report-only command effects | `/a0/plugins/_commands/extensions/python/_functions/agent/AgentContext/_process_chain/start/_10_resolve_slash_command.py:35-59`: nonempty result text rewrites the agent message; empty text plus supported effects sets `data['result']`. | Do not return a nonempty replacement text or `send_message` effect for an advisory-only operation. Prove exact WebUI/backend handling with fixtures. |
| WebUI pre-send | `/a0/plugins/_commands/extensions/webui/send_message_before/_10_resolve_slash_command.js` calls `store.resolveBeforeSend`; `/a0/plugins/_commands/webui/commands-slash-store.js:540-553` parses slash input and sets `sendCtx.cancel`. | Supported source candidate for non-intervention dispatch. Actual composer/active-run behavior remains untested. |
| Command payload | `/a0/plugins/_commands/helpers/commands.py:1041-1095` provides `payload.context.context_id`, agent, optional history, and normalizes text/effects. `:1100-1110` checks context history attributes for lists. | Resolve the live context deliberately; do not assume `include_history` supplies the agent's structured history or meets privacy requirements. |
| Answer fidelity | `/a0/agent.py:747-798` templates user/AI history and stores function-call text with metadata. `/a0/tools/response.py:7-14` returns nonempty `text` or `message` with `break_loop=True`. | Exact response-tool text is a useful extraction target, but successful final delivery and originating-request identity must also be proven. Never choose the latest AI message blindly. |
| History identity | `/a0/helpers/history.py:405-443,523-537` assigns message sequences and serializes counter/IDs. | Snapshot IDs are available; consistent active-run copy, compaction handling, reset/fork semantics, and request invalidation remain open. |
| Explicit preset | `/a0/plugins/_model_config/helpers/model_config.py`: `get_presets` at 451, `resolve_preset` at 567, inheritance at 697, `build_model_config` at 882. Named project scope returns none at 574. | Global-only preset policy remains appropriate. Synthetic inheritance/auth/slot tests still required; no real saved preset was read. |
| Model retries | `/a0/models.py:574-580,591-651`: `unified_call` defaults `a0_retry_attempts` to 2 and retries transient errors. `/a0/helpers/litellm_transport.py` has Chat/Responses request branches and response-retry modes. | Disable wrapper retries explicitly; audit and test SDK/transport retries and transport fallback too. One wrapper invocation is not proof of one underlying request. Fail unsupported configurations without dispatch. |
| Atomic storage | `/a0/helpers/persist_chat.py:105-124` writes/flushed/fsyncs a temporary file, replaces the destination, then fsyncs its directory. | A source pattern exists. Do not depend on a private helper as a stable plugin API without a deliberate compatibility decision. Atomic replacement alone does not solve delete races or all filesystem durability failures. |
| Storage/export/delete | `/a0/helpers/persist_chat.py:35,161-174`: chat-folder helper, JSON export serializes context, chat deletion removes folder. | A separate per-chat sidecar fits the approved export/delete boundary. Fork/copy behavior and late directory recreation require dedicated tests. |
| Namespace | Text scan of `*.command.yaml`/`*.command.yml` under bundled/user plugins and this project found no exact top-level `name: advisor` match. | Preliminary only. Effective catalog, aliases, project/profile overrides, and availability must be checked before reservation. |

## Executed offline probes

Interpreter: `/opt/venv-a0/bin/python`, Python 3.12.4. Probe runner read source via `ast`, extracted the named function with decorators removed, and supplied synthetic dependencies. It did not import or initialize the framework. Probes ran as inline terminal code; no product/test implementation files were created. Promote them to reproducible test fixtures in plan tasks F1–F4 before counting them as integration coverage.

| ID | Actual execution | Result and boundary |
| --- | --- | --- |
| P1 | Extracted `AgentContext.communicate`; synthetic alive task, agent, message, and `Agent.DATA_NAME_SUPERIOR`. Assert message becomes agent intervention and paused is cleared. | PASS. Confirms why active-run ingress must avoid this function. Not a real extension/command-dispatch test. |
| P2 | Extracted `ResponseTool.execute`; synthetic `Response` and exception classes. Assert `text` and `message` return exact text and break-loop; empty text raises. | PASS for both accepted keys and the empty-answer case. Not transport/history extraction, completed-run validation, or rendered-output proof. |
| P3 | Extracted `_write_atomic`; actual temporary project-local directory; write v1; mock `os.replace` to raise on v2; assert v1 remains and temporary file is cleaned. | PASS. Directory removed on completion. Not a power-loss, post-replace fsync failure, concurrent writer, or clear/delete test. |

The initial P1 harness failed with `NameError: Agent` because a synthetic constant was omitted. The corrected harness reran all three probes successfully. The harness error is not an application defect and the failed attempt is not counted as a pass.

Framework metadata probe currently finds `pytest` available. Earlier planning notes recorded it absent; no dependency installation was performed by the fallback. Keep the dependency-light `unittest` command in the plan, and recheck runtime metadata when executing tests.

Read-only mount classification command: `findmnt -T /a0/usr/chats -n -o TARGET,FSTYPE` returned `/a0/usr xfs`. This shows a distinct mount covering chat storage in this container namespace. It does **not** establish Docker volume ownership, compose configuration, replacement policy, host backup behavior, or recreation durability. Host/deployment configuration remains unverified.

## Open integration gates

1. **Ingress/scheduling:** fixture-dispatch help/status/start/cancel while a main run is active; no `communicate`, intervention, unpause, history mutation, or task replacement. Confirm job survival beyond API request completion and plugin-owned cancellation/cleanup.
2. **Evidence:** both supported final-answer history shapes, originating request pairing, reload/compaction, consistent live-work snapshot, and user-stop/new-message/reset/fork/delete invalidation.
3. **Model/privacy:** explicit preset inheritance without fallback; redaction before request; safe destination acknowledgment fingerprint; token/output caps; zero SDK/transport retries or fallback; deadline/cancellation; zero content-bearing logs.
4. **Persistence:** schema validation, admission/recovery markers, failed writes including post-replace failures, stale-generation suppression, clear/delete races, fork/export isolation, disable/uninstall report retention.
5. **UI/auth/runtime:** real authorized command/API/settings/report flow, HTML/image sanitation, browser save/reload, and named-runtime verification. Live installation/provider testing requires separate authorization.

No scope reduction is approved. In particular, failure to prove active-run snapshot review is not permission to silently make `work` idle-only. No general provider/version compatibility claim is made. See the [conditional implementation plan](../plans/advisor-mvp.md).

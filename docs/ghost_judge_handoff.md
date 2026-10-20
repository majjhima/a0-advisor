# Planning handoff: Ghost Judge to Agent Zero Advisor

Date: 2026-10-18

**Status:** Chat summary and planning input, not an approved implementation specification. No Advisor plugin has been implemented or installed by this effort. The user requested that the findings be carried into the `a0_advisor` project for planning.

## 1. Purpose and direction

The work started as an improvement to a Ghost Judge fork: replace its direct OpenRouter model choice with any saved Agent Zero model preset's Main or Utility model. Testing exposed slash-command conflicts and limitations in Ghost Judge's goal-dependent feedback loop.

The user then asked for a reviewer independent of goal requests: automatic review, optional activation when requested, and an explicit command to assess the last response or ongoing chat work. The reference was oh-my-pi's advisor/watchdog design. A separate Advisor plugin was recommended rather than expanding the existing goal loop. The user has now created `a0_advisor` as the place to plan that plugin; specific commands, defaults, and implementation stages still require agreement.

**Working principle:** assess work against the conversation's requests and constraints; do not require another goal store, and do not automatically convert every criticism into further agent work.

## 2. What was implemented in Ghost Judge

### Model-preset integration

The user clarified that “profiles” meant **saved model presets**, not agent profiles such as `researcher` or `developer`.

```yaml
judge_preset: "Default"
judge_model_slot: "utility" # "chat" = Main Model
judge_timeout: 30
```

- Any named preset can select either slot; selection does not follow the active chat's model override or agent-profile identity.
- Provider, authentication, API base, model kwargs, context configuration, and rate limits come from Agent Zero.
- Missing new keys default to `Default`/`utility`; explicit unknown, blank, or invalid selections return `unavailable`, not another model.
- Removed runtime use of `judge_provider`, `judge_model`, `judge_reasoning_effort`, and `judge_max_tokens`; old keys are ignored and described in migration guidance.
- Replaced direct OpenRouter HTTP calls with the framework model wrapper; enforced asynchronous timeout, propagated cancellation, and returned fixed failure messages rather than raw provider exceptions.
- Added best-effort credential-shaped output scrubbing. This is NOT complete secret protection and does not replace masking input before model calls.
- Fixed an extension import from nonexistent `helpers.ghost_judge` to `helpers.quality_gate`.
- Added tests, specification/plan documents, and Git exclusions for local metadata and caches.

### Preset-resolution contract worth reusing

`plugins._model_config.helpers.model_config` owns preset definitions and inheritance:

1. `resolve_preset(name)` returns a raw preset or `None`; it does not apply inheritance.
2. `preset_to_config(Default)` provides the baseline.
3. `build_config_from_preset(selected, baseline, strip_api_key=False)` applies Default inheritance and per-slot kwargs replacement.
4. Select `chat_model` or `utility_model` and call `build_model_config(..., models.ModelType.CHAT)`.
5. Build with `models.get_chat_model(..., model_config=mc, **mc.build_kwargs())`.
6. `await wrapper.unified_call(system_message=..., user_message=...)` returns response/reasoning.

Preserve config immutability, authentication and provider-specific kwargs isolation. Do not silently substitute the current Utility Model if resolution fails. Recheck these APIs against the target framework before implementation.

### Local commit, installation, and verification

| Item | State |
| --- | --- |
| Ghost Judge source | `/a0/usr/projects/ghost_judge` |
| User fork | https://github.com/majjhima/ghost-judge |
| Original upstream | https://github.com/kironkeyz/ghost-judge |
| Related upstream PR | https://github.com/kironkeyz/ghost-judge/pull/2 |
| Local commit | `2eaac75` — `feat: select judge model via Agent Zero presets` |
| Publication | Commit created locally; no push performed in this chat |
| Installed runtime payload | `/a0/usr/plugins/ghost_judge` |
| Automated verification recorded | 49 unittest tests passed, 0 skipped; Python syntax checks and `git diff --check` passed |
| Independent review | Test, code, and security reviewers approved after targeted fixes |
| Not verified | Real provider-backed end-to-end judgment; actual Chat/Responses transport behavior |

Tests included synthetic preset/framework-boundary checks, an unpatched Default-resolution smoke check, kwargs/auth/rate-limit preservation, timeouts/cancellation, parser/error handling, forwarding, and extension import. Model calls were mocked; passing tests do not establish useful real-world reviewing behavior.

Subsequent changes remained uncommitted at handoff: `README.md`, `commands/goal.command.yaml`, `commands/subgoal.command.yaml`, and new `webui/config.html`. Git status was rechecked when preparing this handoff; do not assume the installation and commit are identical.

## 3. Configuration UI and testing lessons

Ghost Judge initially had no Config button. Agent Zero detects a settings screen from `webui/config.html`; `settings_sections: [agent]` alone does not create one.

A native settings form was added to the project and installed copy. It binds to `config.*` and relies on the framework settings modal for persistence. Discovery returned `has_config_screen: True`. It exposes enable, preset name, slot, turn limit, timeout, failure limit, and domain settings.

**Limits:** no browser save/reload round-trip was demonstrated. Architecture review also found `max_turns` in plugin settings is not wired into new goal creation; exposing a field is not evidence that it affects runtime behavior.

A recorded usability request was to populate the preset selector from existing presets and show Main/Utility model names, rather than require typing a preset name. Carry this into Advisor planning, but it has not been implemented.

Historical README attribution and benchmark-environment statements were restored at the user's request. Preserve original authorship and historical test claims when borrowing code; do not rewrite them as if the new preset path produced those results. Check applicable licenses before copying.

## 4. Slash-command problems and resolutions

### Outdated duplicate Commands plugin

Both `usr/plugins/commands` and bundled `plugins/_commands` were enabled. The older community plugin called `input.setSelectionRange(...)` on a contenteditable composer, causing `input.setSelectionRange is not a function`.

A local guard was applied and syntax checked; the user then removed the obsolete plugin. Removal and sole remaining bundled command provider were verified. The temporary patch was to that removed runtime plugin, not the Ghost Judge repository.

### Goal command collision

The bundled `_goal` plugin owns `/goal`, while Ghost Judge originally contributed a command with the same name. The effective command catalog uses first-wins merging; observed resolution was `/goal` -> `_goal` and `/subgoal` -> `ghost_judge`. They operated on different state stores, so core status and Ghost Judge criteria were disconnected.

Ghost Judge's definitions were renamed to `/ghost-goal` and `/ghost-subgoal`, retaining the existing YAML filenames and script paths. Project README and installed definitions were updated. Framework resolution tests confirmed both new names resolve to Ghost Judge and `/goal` remains core-owned. Temporary test state was backed up/restored and reproduction artifacts removed. No post-fix browser workflow was established.

**Design lesson:** use unique commands and context-bound state. Do not replace `/goal`, disable `_goal`, or patch command precedence for Advisor. `_goal` being always enabled does NOT prohibit a plugin from optionally reading goal context; independent operation simply must not depend on it.

## 5. Why a standalone Advisor was recommended

| Ghost Judge finding | Implication |
| --- | --- |
| One shared `/a0/data/ghost_goal_state.json` | Chats share state, failure counters, and refinement budget; unsuitable for a per-chat advisor |
| Activation requires its own nonempty goal | Removing this check alone does not provide suitable review semantics |
| Hook is `AgentContext.communicate/end` | `communicate` launches/returns a task; its background watcher judges after task completion and visible output |
| Feedback uses `context.communicate(UserMessage(...))` | Late feedback can intervene in a newer user turn instead of the one reviewed |
| Goal-specific `done/reason/missing` contract | Poor fit for exploratory work, clarification requests, and valid partial progress |
| Small trailing transcript excerpt | Cannot prove repository/test/external-fact correctness without actual investigation |
| Active-context JSON read without a writer found in inspected source | Do not rely on tab-based domain context |
| Configured `max_turns` is not consumed by goal creation | Do not carry over dead settings |

These are source-inspection findings from the chat, not newly run concurrency tests. The async staleness path needs reproduction and regression coverage in any successor.

Reuse the preset-based model client, useful domain criteria, and selected tests. Replace the global goal store, goal-specific prompts/verdicts, and auto-resubmission controller. A standalone plugin should not import runtime helpers from an installed Ghost Judge plugin or require it to be installed.

## 6. oh-my-pi research: relevant ideas and boundaries

Primary reference: https://github.com/can1357/oh-my-pi/blob/main/docs/advisor-watchdog.md

Source reviewed at commit `9d5d4ac5b50ac2bb5866e4202efc0cf785dfc57a`:
https://github.com/can1357/oh-my-pi/blob/9d5d4ac5b50ac2bb5866e4202efc0cf785dfc57a/docs/advisor-watchdog.md

- The advisor is an optional separate LLM agent consuming transcript deltas and able to investigate with its own tools; it emits findings through `advise`.
- `WATCHDOG.md` supplies advisor-only guidance; `WATCHDOG.yml` configures the reviewer roster and policy. They are not a separate goal system or necessarily a timer.
- Review cadence can follow model turns or completed runs. Findings have `nit`, `concern`, or `blocker` severity, with different delivery/continuation rules.
- The design includes deduplication, note budgets, cooldown, cancellation handling, bounded transcript management, and masking before reviewer calls.
- Its own advisory messages are excluded from subsequent deltas; advisor-initiated continuations do not schedule another review, avoiding feedback cascades.
- Deliberate user interruption suppresses automatic resumption. Investigative tools default to a restricted read/search set.
- Some oh-my-pi blocker paths bypass ordinary budgets. This is a reference behavior, NOT a recommended exception to hard user-control/cost limits for Agent Zero.

Borrow the separation of review, findings, and intervention—not the entire multi-advisor runtime in the MVP. Research was document/source inspection, not execution of oh-my-pi.

### Jev and decision models: separate from the advisor

At the inspected revision, no direct Jev/decision-model stage was found in the advisor path. Advisor model selection uses the `advisor` role; structured judgment uses a separate `judge` role via eval-kernel `judge()`/`judge_batch()` calls.

| Advisor LLM | Jev / structured judgment |
| --- | --- |
| Reviews transcript, investigates, explains findings | Evaluates supplied items against explicit questions/rubrics |
| Emits advice and severity | Returns structured choices, probabilities, or scores |
| Scheduled by advisor policy | Explicit judgment API calls |

An advisor granted `eval` could potentially compose these capabilities, but this was not verified as a default or dedicated integration. A cheap decision-model screening stage before an expensive advisor would be a NEW design choice for Agent Zero, not a proven feature copied from oh-my-pi. It risks false negatives; manual review should bypass any such screen. Judgment output is evidence, not proof.

Pinned sources:
- Judgment API: https://github.com/can1357/oh-my-pi/blob/9d5d4ac5b50ac2bb5866e4202efc0cf785dfc57a/docs/tools/eval.md
- System One adapter: https://github.com/can1357/oh-my-pi/blob/9d5d4ac5b50ac2bb5866e4202efc0cf785dfc57a/packages/ai/src/judgment/systemone.ts
- Jevify guidance: https://github.com/can1357/oh-my-pi/blob/9d5d4ac5b50ac2bb5866e4202efc0cf785dfc57a/packages/coding-agent/src/prompts/system/jevify-notice.md

## 7. Proposed Advisor MVP and later stages

**Recommendation, awaiting specification approval:** manual-first, one reviewer selected by named preset plus Main/Utility slot, visible reports, no required goal and no automatic corrective execution.

### Separate three concerns

1. **Activation:** manual; later opt-in per-chat automatic review; eventually a project/global default for new chats with a per-chat off switch.
2. **Scope:** latest completed response versus a versioned snapshot of ongoing work. Use the originating user request, relevant corrections, and project constraints as the baseline. Do not invent requirements or treat unfinished progress as failure.
3. **Action:** report only by default; optional bounded correction later; true pre-delivery gating is a different feature.

Illustrative commands, NOT reserved or implemented (verify namespace availability):

```text
/advisor last
/advisor work
/advisor last Check whether the conclusions are supported by sources
/advisor on
/advisor off
/advisor status
```

An optional uniquely named agent tool can invoke the same service when asked in normal language. Commands should provide deterministic access independent of the primary model deciding to cooperate. Report delivery must not accidentally send the review as a new task for the main agent.

Suggested report fields: assessment (`satisfactory`, `progressing`, `needs_attention`, `blocked`, `insufficient_evidence`), severity-ranked findings with evidence references, suggested next step, snapshot/turn identity, and verification limits. Do not represent a model's confidence as objective correctness.

### Staged work to plan

| Stage | Outcome / acceptance direction |
| --- | --- |
| 0. Specify | Agree commands, report contract, review-context/privacy policy, and activation defaults; prove hook/delivery feasibility |
| 1. Manual reviews | Last-response and current-work reports for the correct chat; no goal dependency, unwanted execution, or network call when idle |
| 2. Automatic advisory | Opt-in completed-response review with per-chat state, cadence, dedupe, timeouts, and stale-result handling |
| 3. Deeper investigation | Optional isolated read-only workspace tools and reviewer policy, clearly differentiated from transcript-only review |
| 4. Bounded correction | Separate explicit opt-in; hard retry/cost caps; respects stop, new messages, and other continuation mechanisms |

Defer multiple reviewers, decision-model screening, per-tool-step monitoring, and synchronous gating until the basic path is reliable. Do not silently combine manual-first review and automatic correction.

## 8. Architecture constraints and verification targets

- Key all state, reports, queues, and budgets by chat/context; never reuse a shared global goal file.
- Attach chat ID, originating request/turn ID, snapshot revision, and trigger origin to each review. Cancel or mark stale results after new instructions, switches, resets, forks, or transcript rewrites; never inject into a newer turn.
- Keep advisor reports separate from user instructions. Treat transcript, tool output, and reviewer advice as untrusted data. Mask/redact sensitive input before sending it to a potentially different provider.
- Never resume after deliberate user cancellation. Avoid reviewing the review or starting advisor -> primary -> advisor loops. Single-flight reviewer per chat; hard review, context, timeout, and optional corrective-pass limits.
- Reviewer failures must not wedge the main agent; distinguish unavailable reviewer from negative assessment.
- Prefer ordinary plugin hooks, APIs, tools, and native settings; preserve auth/CSRF. No planned core edits or `_goal` overrides. Optional read-only goal context can be evaluated later.
- Real UI tests must cover Config visibility, preset catalog/model labels, save/reload persistence, command execution, and report rendering. Check each exposed setting actually changes runtime behavior.

Source-inspected Agent Zero integration candidates (revalidate; not a tested Advisor implementation):
- `monologue_end` for completed-run observation; extensions execute sequentially/awaited, so inline review adds latency and background review requires staleness controls.
- `tool_execute_after` runs before the agent honors `break_loop`; possible control point for future bounded continuation, but NOT proof that already streamed output can be withheld. Test timing and interaction with `_goal` before attempting a true delivery gate.
- Commands receive `payload.context.context_id`; design report-only side effects intentionally. `show_markdown` differs between WebUI delivery and backend command resolution; do not assume it is a persistent report panel.
- Non-underscore `context.data` keys are persisted in the inspected framework. Underscore-prefixed transient lock/dedup state is not a durable activation preference.

Minimum tests to plan: per-chat isolation, correct last answer selection, no-answer case, ongoing-work snapshot races, manual review bypasses screening, invalid preset/slot without fallback, secret masking, parser/timeout/cancellation errors, duplicate triggers, user stop/new-message behavior, no recursive review, and optional core-goal coexistence. Validate real provider output separately from mocked boundary tests.

## 9. Open decisions and next planning session

1. Final public command/tool names and report destination: modal, chat-side note, persistent report, or a combination?
2. Exact review context, size budget, masking, retention, and handling of private/internal transcript content?
3. Transcript-only MVP versus isolated read-only investigation in the first release?
4. Default preset/slot, timeout, review cadence, activation persistence and override precedence?
5. How should explicit criteria be supplied without creating another competing goal system?
6. When, if ever, may findings trigger corrective work; how will user consent and hard limits be represented?
7. Local-only prototype or eventual community distribution; packaging, license/attribution, and minimum Agent Zero compatibility?

Suggested starting point: turn sections 7-8 into a short reviewed specification, prove command/report delivery without starting the main agent, then write tests before implementation. Do not interpret this handoff as approval to implement every proposal.

## 10. Local reference map and maintenance notes

| Reference | Location |
| --- | --- |
| Preset client to evaluate for reuse | `/a0/usr/projects/ghost_judge/helpers/judge_client.py` |
| Goal-coupled controller | `/a0/usr/projects/ghost_judge/helpers/quality_gate.py` |
| Global state design to avoid | `/a0/usr/projects/ghost_judge/helpers/goal_state.py` |
| Existing tests | `/a0/usr/projects/ghost_judge/tests/` |
| Previous approved preset specification | `/a0/usr/projects/ghost_judge/docs/specs/profile-based-judge-model.md` (filename predates preset clarification) |
| Agent Zero plugin skill | `/a0/skills/a0-create-plugin/SKILL.md` and its implementation/WebUI/review references |
| Framework hook owner | `/a0/agent.py`, `/a0/helpers/extension.py` |
| Command resolution owner | `/a0/plugins/_commands/helpers/commands.py` |
| Preset owner | `/a0/plugins/_model_config/helpers/model_config.py` |

Framework verification in this environment uses `/opt/venv-a0/bin/python`, not the separate `/opt/venv` agent runtime. The recorded Ghost Judge command was `/opt/venv-a0/bin/python -m unittest discover -s tests`; no new dependency installation was needed. Recheck available dependencies rather than assuming pytest exists.

This handoff adds documentation only. No plugin source has been copied into `a0_advisor`, no runtime behavior changed, no project metadata edited, and no new commit/push authorized. Target project `.a0proj/` was already untracked; add appropriate ignores before any future staging, and never commit local metadata, secrets, or chat transcripts.

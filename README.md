# a0-advisor
Agent Zero advisor to judge active model work for improved quality.

## Planning

This project is in planning; an Advisor plugin has not been implemented yet.

Start with the [Ghost Judge effort and Advisor planning handoff](docs/ghost_judge_handoff.md). It records the completed Ghost Judge work, testing limits, command conflicts, architecture findings, oh-my-pi/Jev research, proposed MVP, and open decisions.

The approved MVP baseline is goal-independent, manual, transcript-only review of the last response or a fixed snapshot of current work, using one explicitly selected saved model preset. It produces a dedicated per-chat report and short notification; it never resumes or corrects the main agent automatically. Per-chat automatic monitoring, investigative tools, and bounded correction remain deferred.

The [Advisor MVP specification](docs/specs/advisor-mvp.md), revision 0.4, records the approved product baseline, privacy/concurrency controls, initial limits, and durable-retention contract. Offline feasibility checks and technical planning are the next step; the implementation plan requires review before BUILD. Global-only configuration, explicit reviewer/provider disclosure, and the initial review ceilings are approved. Local-only versus community distribution remains open; no installation, provider calls, commits, or publication are authorized by this planning milestone.

**Approved retention direction:** save the latest completed Advisor report and minimal job outcome metadata alongside its chat, rather than only in process memory. Reports should survive browser reloads, backend restarts with retained storage, and plugin disable/re-enable; live reviews never resume automatically. Clearing Advisor data or deleting the chat removes saved state. Container replacement requires persistent chat storage, and ordinary JSON chat export does not include the separate report file. See specification section 5.4 for the full contract.

## Technical planning

- [Conditional implementation plan](docs/plans/advisor-mvp.md): four feasibility tasks, small tested product slices, and verification/review/release gates. Plan approval and proof closure are required before product BUILD.
- [Initial feasibility evidence](docs/feasibility/initial-boundaries.md): source findings and three passed isolated probes, not live integration certification. Active-run command isolation, final-answer extraction, provider request limits, and race-safe durable storage remain open proof obligations.

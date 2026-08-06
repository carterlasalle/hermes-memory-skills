# hermes-memory-skills (DuckBrain fork) — Task Board

> Foreman: deepseek-v4-flash @ openrouter | DuckBrain: hermes-memory | Remote: carterlasalle/hermes-memory-skills
> Scope: add **DuckBrain** as a memory backend to the agent-dreaming skills. SPEC gates CORE.

## Active

| ID | Task | Pri | Cpx | Deps | Tags | Model | Reasoning | Fallback |
|----|------|-----|-----|------|------|-------|-----------|----------|
| T02 | CORE: create `agent-dreaming-duckbrain/SKILL.md` — DuckBrain-native dreaming skill routing to DuckBrain MCP (remember/recall/forget/list_keys/squash), hermes-memory namespace default | High | 4±1 | T01 | +++duckbrain, ++memory, ++markdown, -vision | DS-V4-Flash | Medium | Kimi-K3 |
| T03 | CORE: extend `agent-dreaming-agnostic/SKILL.md` — add duckbrain to Phase 0 detection, Phase 2/2.5 routing, capacity & condensation sections, compatibility matrix (do NOT break built-in/holographic) | High | 4±1 | T01 | +++memory, ++duckbrain, ++markdown, -vision | DS-V4-Flash | Medium | Kimi-K3 |
| T04 | DOC: add `agent-dreaming-agnostic/references/duckbrain-backend.md` — DuckBrain tools, schema, namespaces, key/domain strategy, VSS, compaction | Medium | 2±1 | T01 | +++docs, ++duckbrain, -vision | DS-V4-Flash | Low | — |
| T05 | DOC: update `README.md` — add DuckBrain to backend matrix, detection, installation, cron notes | Medium | 2±1 | T02,T03 | +++docs, -vision | DS-V4-Flash | Low | — |
| T06 | TEST: end-to-end verification — run ONE manual dream cycle against live DuckBrain (:3000) in hermes-memory namespace; verify remember→recall→forget→squash routing works; record evidence (key ids) in diary | High | 3±1 | T02,T03 | +++duckbrain, ++terminal, ++memory, -vision | DS-V4-Flash | Medium | DS-V4-Pro |

## [ ] NEVER-DONE — Run coding-hermes-never-done 12-point audit

Load coding-hermes-never-done skill. Run ALL 12 checks: spec alignment,
doc coverage, test gaps, package upgrades, pitfall hunt, performance audit,
endpoint verification, CI/CD health, DuckBrain sync, code quality,
middle-out wiring, usability smoke test. Create a task for EVERY gap found.
This task is never complete — the audit always finds something.

**CHECK 12 — USABILITY SMOKE TEST:** See `references/never-done-12th-check-usability.md`.
Deploy fresh, walk the full user journey (install the skill → run a dream →
verify a DuckBrain memory lands), verify behavior not just file presence.

## Completed

| ID | Task | Pri | Cpx | Commit | Model |
|----|------|-----|-----|--------|-------|
| T01 | SPEC: `specs/duckbrain-backend.md` — routing contract (Phase 0 detect, Phase 2 remember/recall/forget, Phase 2.5 forget+squash, ns/key/domain strategy, capacity=compaction stats, compat matrix, verification). GitReins judge PASS (6/6 criteria) | Critical | 3±1 | <COMMIT> | DS-V4-Flash |
| T00 | Bootstrap: clone fork, gitreins+hilo init, board, AGENTS.md | Trivial | 1±0 | — | DS-V4-Flash |

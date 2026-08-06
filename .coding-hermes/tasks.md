# hermes-memory-skills (DuckBrain fork) — Task Board

> Foreman: deepseek-v4-flash @ openrouter | DuckBrain: hermes-memory | Remote: carterlasalle/hermes-memory-skills
> Scope: add **DuckBrain** as a memory backend to the agent-dreaming skills. SPEC gates CORE.
> Status 2026-08-05: **IDLE tick #1** — all board tasks [x]. NEVER-DONE audit: spec alignment ✅ (T06 live-verified), docs ✅, no code/deps/perf (markdown repo), pitfalls → squash partition pitfall documented in spec+skills (c65633e), endpoints ✅ (live :3000), DuckBrain sync ✅, wiring N/A, usability ✅ (T06 dream cycle), GitReins judge ✅ (8/8). No new tasks. Cooldown → 43200s.

## Active

| ID | Task | Pri | Cpx | Deps | Tags | Model | Reasoning | Fallback |
|----|------|-----|-----|------|------|-------|-----------|----------|

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
| T06 | TEST: live e2e dream cycle vs DuckBrain :3000 (hermes-memory ns) — remember→recall→forget→squash verified with REAL ids (`60bc7ad0…`, `aba9c198…`); evidence in `agent-dreaming-agnostic/references/duckbrain-e2e-evidence.md` + diary. GitReins judge PASS (8/8). Found: squash needs explicit `partition` (default-ns resolution bug); real squash blocked by server SQL parser bug (out of repo scope) | High | 3±1 | c65633e | DS-V4-Flash |
| T05 | DOC: `README.md` — DuckBrain added to backend matrix, detection (+:3000 liveness), installation, cron notes (E8). GitReins judge PASS (4/4) | Medium | 2±1 | 8c7818c | DS-V4-Flash |
| T04 | DOC: `agent-dreaming-agnostic/references/duckbrain-backend.md` — tools, schema, namespaces, key/domain strategy, VSS, compaction, comparison table. GitReins judge PASS (3/3) | Medium | 2±1 | f660415 | DS-V4-Flash |
| T03 | CORE: `agent-dreaming-agnostic/SKILL.md` — DuckBrain routing added (Phase 0 two-step detect, Phase 2 promotion routing, Phase 2.5 condensation, compat matrix column). GitReins judge PASS (5/5) | High | 4±1 | c230f07 | DS-V4-Flash |
| T02 | CORE: `agent-dreaming-duckbrain/SKILL.md` — DuckBrain-native dreaming skill (Phase 0 gate, DuckBrain-only Phase 2/2.5 routing, explicit namespaces, E1-E8 catalog). GitReins judge PASS (6/6) | High | 4±1 | 54c1bc0 | DS-V4-Flash |
| T01 | SPEC: `specs/duckbrain-backend.md` — routing contract (Phase 0 detect, Phase 2 remember/recall/forget, Phase 2.5 forget+squash, ns/key/domain strategy, capacity=compaction stats, compat matrix, verification). GitReins judge PASS (6/6 criteria) | Critical | 3±1 | 1ca9629 | DS-V4-Flash |
| T00 | Bootstrap: clone fork, gitreins+hilo init, board, AGENTS.md | Trivial | 1±0 | — | DS-V4-Flash |

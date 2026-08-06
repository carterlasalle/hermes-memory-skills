# hermes-memory-skills (DuckBrain fork) — Task Board

> Foreman: deepseek-v4-flash @ openrouter | DuckBrain: hermes-memory | Remote: carterlasalle/hermes-memory-skills
> Scope: add **DuckBrain** as a memory backend to the agent-dreaming skills. SPEC gates CORE.
> Status 2026-08-06: **IDLE tick #2** — audit found 2 gaps, BOTH fixed same tick (T07, T08). Checks: spec alignment → T07; docs ✅; test/deps/perf N/A (markdown repo); pitfalls → T08 (gitleaks allowlist whitelisted ALL .md — entire repo content — now narrowed, 40 commits re-scanned clean); endpoints ✅ (:3000/health, uptime 38.6h); CI/CD ✅ (gitreins guard PASS); DuckBrain sync ✅ (24 keys in hermes-memory, project+reflections current); wiring/usability ✅ (T06 evidence); GitReins judge ✅ (2/2 + 2/2). Also logged 3e4f6e9 (T06 finding #3 fix, landed post-idle-tick-1) on the board. Cooldown: 64800s (18h, owned via API PUT).

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
| T08 | PITFALL: `.gitleaks.toml` allowlist whitelisted `specs/`, `docs/`, `.*\.spec\.md`, AND `.*\.md` — the entire content of this markdown-only repo, disabling the secrets guard for real content. Narrowed to VCS/gitreins/log paths only (file is gitignored local state — fix lives in workdir; gitreins reads it from there). Verified: gitleaks detect 40 commits clean, no panic; gitreins guard secrets PASS. GitReins judge PASS (2/2, verdict 5e8bc470) | High | 1±0 | — (gitignored file; board+DuckBrain are the record) | DS-V4-Flash |
| T07 | SPEC: `specs/duckbrain-backend.md` §4.1 split into 4.1a (agnostic — config-gated two-step detection) and 4.1b (native — liveness + `mcp__duckbrain__*` tool presence, config informational), reconciling the spec with the 3e4f6e9 gate fix. GitReins judge PASS (2/2, verdict 5ce2fc90) | High | 2±1 | 4585a0e | DS-V4-Flash |
| T06b | FIX (logged post-idle-tick-1): `agent-dreaming-duckbrain/SKILL.md` Phase 0 gate — liveness + tool presence sufficient, config `memory.provider` optional (T06 finding #3; landed 2026-08-05 as 3e4f6e9) | High | 2±1 | 3e4f6e9 | DS-V4-Flash |
| T05 | DOC: `README.md` — DuckBrain added to backend matrix, detection (+:3000 liveness), installation, cron notes (E8). GitReins judge PASS (4/4) | Medium | 2±1 | 8c7818c | DS-V4-Flash |
| T04 | DOC: `agent-dreaming-agnostic/references/duckbrain-backend.md` — tools, schema, namespaces, key/domain strategy, VSS, compaction, comparison table. GitReins judge PASS (3/3) | Medium | 2±1 | f660415 | DS-V4-Flash |
| T03 | CORE: `agent-dreaming-agnostic/SKILL.md` — DuckBrain routing added (Phase 0 two-step detect, Phase 2 promotion routing, Phase 2.5 condensation, compat matrix column). GitReins judge PASS (5/5) | High | 4±1 | c230f07 | DS-V4-Flash |
| T02 | CORE: `agent-dreaming-duckbrain/SKILL.md` — DuckBrain-native dreaming skill (Phase 0 gate, DuckBrain-only Phase 2/2.5 routing, explicit namespaces, E1-E8 catalog). GitReins judge PASS (6/6) | High | 4±1 | 54c1bc0 | DS-V4-Flash |
| T01 | SPEC: `specs/duckbrain-backend.md` — routing contract (Phase 0 detect, Phase 2 remember/recall/forget, Phase 2.5 forget+squash, ns/key/domain strategy, capacity=compaction stats, compat matrix, verification). GitReins judge PASS (6/6 criteria) | Critical | 3±1 | 1ca9629 | DS-V4-Flash |
| T00 | Bootstrap: clone fork, gitreins+hilo init, board, AGENTS.md | Trivial | 1±0 | — | DS-V4-Flash |

# hermes-memory-skills (DuckBrain fork) — Task Board

> Foreman: deepseek-v4-flash @ openrouter | DuckBrain: hermes-memory | Remote: carterlasalle/hermes-memory-skills
> Scope: add **DuckBrain** as a memory backend to the agent-dreaming skills. SPEC gates CORE.
> Status 2026-08-07: **Tick — audit found 1 gap (T09), fixed + judged PASS**. Checks: spec alignment ✅ (found T09: native skill Phase 0 header still described old config-gated two-step check; §4.1b + result table said liveness+tool-presence — fixed 6564475, judge ea715153 2/2); docs ✅ (README matrix, agnostic refs, e2e evidence); test/deps/perf N/A (markdown-only repo); pitfalls ✅ (gitleaks allowlist narrowed — only VCS/gitreins/log paths, T08 held); endpoints ✅ (:3000/health HTTP 200); CI/CD ✅ (gitreins guard PASS — secrets/lint/tests; no GH Actions workflows on this repo — gitreins is the CI gate); DuckBrain sync ✅ (33 keys in hermes-memory, namespace alive, tick record current); wiring/usability ✅ (T06 live dream-cycle evidence, still valid); GitReins judge ✅ (evaluator configured + sized: 60 iter/20m/1M in/0.4M out). No gitreins tasks pending (0). Cooldown re-owned at 64800s (18h, PUT → GET verified).

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
| T09 | SPEC-ALIGN: `agent-dreaming-duckbrain/SKILL.md` Phase 0 header still described the OLD config-gated two-step check ("config value AND liveness", Step A/B) — contradicting SPEC-DB-001 §4.1b and the skill's own result-table note (operative gate = liveness + tool presence, config informational). Rewrote to Gate check 1 (liveness) + Gate check 2 (tool presence) + informational config read; removed redundant "Also verify tool availability" section. GitReins judge PASS (2/2, verdict ea715153) | High | 1±0 | 6564475 | DS-V4-Flash |
| T08 | PITFALL: `.gitleaks.toml` allowlist whitelisted `specs/`, `docs/`, `.*\.spec\.md`, AND `.*\.md` — the entire content of this markdown-only repo, disabling the secrets guard for real content. Narrowed to VCS/gitreins/log paths only (file is gitignored local state — fix lives in workdir; gitreins reads it from there). Verified: gitleaks detect 40 commits clean, no panic; gitreins guard secrets PASS. GitReins judge PASS (2/2, verdict 5e8bc470) | High | 1±0 | — (gitignored file; board+DuckBrain are the record) | DS-V4-Flash |
| T07 | SPEC: `specs/duckbrain-backend.md` §4.1 split into 4.1a (agnostic — config-gated two-step detection) and 4.1b (native — liveness + `mcp__duckbrain__*` tool presence, config informational), reconciling the spec with the 3e4f6e9 gate fix. GitReins judge PASS (2/2, verdict 5ce2fc90) | High | 2±1 | 4585a0e | DS-V4-Flash |
| T06b | FIX (logged post-idle-tick-1): `agent-dreaming-duckbrain/SKILL.md` Phase 0 gate — liveness + tool presence sufficient, config `memory.provider` optional (T06 finding #3; landed 2026-08-05 as 3e4f6e9) | High | 2±1 | 3e4f6e9 | DS-V4-Flash |
| T05 | DOC: `README.md` — DuckBrain added to backend matrix, detection (+:3000 liveness), installation, cron notes (E8). GitReins judge PASS (4/4) | Medium | 2±1 | 8c7818c | DS-V4-Flash |
| T04 | DOC: `agent-dreaming-agnostic/references/duckbrain-backend.md` — tools, schema, namespaces, key/domain strategy, VSS, compaction, comparison table. GitReins judge PASS (3/3) | Medium | 2±1 | f660415 | DS-V4-Flash |
| T03 | CORE: `agent-dreaming-agnostic/SKILL.md` — DuckBrain routing added (Phase 0 two-step detect, Phase 2 promotion routing, Phase 2.5 condensation, compat matrix column). GitReins judge PASS (5/5) | High | 4±1 | c230f07 | DS-V4-Flash |
| T02 | CORE: `agent-dreaming-duckbrain/SKILL.md` — DuckBrain-native dreaming skill (Phase 0 gate, DuckBrain-only Phase 2/2.5 routing, explicit namespaces, E1-E8 catalog). GitReins judge PASS (6/6) | High | 4±1 | 54c1bc0 | DS-V4-Flash |
| T01 | SPEC: `specs/duckbrain-backend.md` — routing contract (Phase 0 detect, Phase 2 remember/recall/forget, Phase 2.5 forget+squash, ns/key/domain strategy, capacity=compaction stats, compat matrix, verification). GitReins judge PASS (6/6 criteria) | Critical | 3±1 | 1ca9629 | DS-V4-Flash |
| T00 | Bootstrap: clone fork, gitreins+hilo init, board, AGENTS.md | Trivial | 1±0 | — | DS-V4-Flash |

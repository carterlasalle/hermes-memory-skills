# DuckBrain E2E Evidence — 2026-08-05 (T06)

Live end-to-end verification of SPEC-DB-001 §7 routing contract against
`http://127.0.0.1:3000` in namespace `hermes-memory`. Companion to the dream
diary entry (`$HERMES_HOME/dreams/diary.md`). All calls passed
`namespace="hermes-memory"` explicitly (SPEC §3.1 discipline).

## Environment

- Server: `GET /health` → `{"status":"healthy","uptime":73416,...}`
- Namespaces (list_namespaces): `hermes-memory` (default + current),
  `default`, `coding-hermes`, `livecostdashboard`
- Baseline keys (list_keys prefix="/"): `/hermes-memory/purpose`,
  `/person/carter-lasalle[/sources/*]`, `/reflections/2026-08-05/*`,
  `/system/test`, `/test/ollama-embed` (pre-existing test keys — not created
  by this cycle)
- Baseline compaction (get_compaction_stats): all zeros (no compaction yet)

## Scenario Results (T1–T7)

| # | Scenario | Call | Result | Evidence |
|---|----------|------|--------|----------|
| T1 | Phase 0 detects duckbrain | config read + :3000/health | ⚠️ PARTIAL — server healthy + ns present; `memory.provider` ABSENT in live config (skill table would classify built-in) | health JSON; config grep |
| T2 | Add → remember | `remember(key=/test/dream-cycle-2026-08-05-2140, domain=concept, attrs={source_session:t06-verify,...}, embedding_text=..., ns=hermes-memory)` | ✅ id `60bc7ad0-b46e-4544-8115-03fb32e714c0`; `recall(key=...)` returns it (E7) | ids below |
| T3 | Replace = remember + forget | remember same key → id `aba9c198-2dfd-4876-b72a-a004c980b485`; `forget(id=60bc7ad0..., reason="T06 replacement test — superseded by ...")` | ✅ recall(key) returns ONLY new id; old tombstoned | recall output |
| T4 | Remove → forget | `forget(id=aba9c198..., reason="T06 removal test")` | ✅ recall(key=...) → count 0 | recall output |
| T5 | Capacity = compaction stats | `get_compaction_stats()` | ✅ object w/ tombstonePercent, parquetRatio, partition counts (all 0 — honest, no compaction yet) | stats JSON |
| T6 | Condensation preview | `squash(dryRun=true)` (no partition) → ❌ "Default namespace not found"; `squash(dryRun=true, partition="concept/2026-08/")` → ✅ "Preview: Would compact 2 records, removing 2 tombstones" | ⚠️ works WITH explicit partition only | squash outputs |
| T7 | Cleanup | forget both test ids; `recall(keyPrefix="/test/dream-cycle")` → count 0 | ✅ records gone; key paths linger in list_keys (index pruned only by squash) | list_keys |

## Real Squash Attempt

`squash(partition="concept/2026-08/")` (dryRun=false) → **FAIL**:
`Parser Error: syntax error at or near ")" ...('namespaces/hermes-memory/concept/2026-08/current.jsonl'))`
— a DuckBrain **server-side** bug. DuckBrain source lives at
`/home/hermes/duckbrain` — **out of scope** for this repo (SPEC-DB-001
non-goals). Recorded for the DuckBrain maintainer; no repo change possible.

## Findings (spec-relevant)

1. **`squash` `partition` is effectively required on this deployment.**
   SPEC-DB-001 §2.6 marks `partition` optional; the server resolves a missing
   partition against a namespace literally named `default`, but this
   deployment's default namespace is `hermes-memory` →
   `"Default namespace not found"`. **Fix (this repo):** skill/spec should
   instruct passing `partition` explicitly (the real squash targets
   `namespaces/<ns>/<domain>/<yyyy-mm>/current.jsonl` partitions).
2. **T7 "zero /test/ keys" is only observable via `recall`, not `list_keys`.**
   `list_keys` still lists tombstoned key paths until squash physically
   removes them. Verification criteria should scope T7 to `recall(...)=0`.
3. **Live config lacks `memory.provider: duckbrain`.** On this host a strict
   skill-table Phase 0 classifies built-in despite a healthy server. Either
   set the provider (user's choice — this fork only documents) or treat
   liveness+namespace as the duckbrain signal per AGENTS.md routing contract.

## Verified Working (pass criteria T1–T6 with real tool output)

- `remember` → returns id; `recall(key=...)` confirms write (E7) ✅
- Replacement: write-new-then-forget-old; old id tombstoned ✅
- Removal: `forget` → `recall(key=...)` empty ✅
- `list_keys` / `get_compaction_stats` / `squash(dryRun, partition)` ✅
- Namespace discipline: every call explicit `namespace="hermes-memory"` ✅

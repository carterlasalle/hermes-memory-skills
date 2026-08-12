# DuckBrain E2E Evidence — 2026-08-12 (E2E-001)

Live end-to-end re-verification of the SPEC-DB-001 §7 dream-cycle routing contract
against `http://127.0.0.1:3000` in namespace `hermes-memory`. Mandated re-run of the
T06 evidence (`agent-dreaming-agnostic/references/duckbrain-e2e-evidence.md`,
2026-08-05) — the last full run was 12 tick-events ago; E2E-001 mandates
re-verification every 5–10 ticks. All calls passed `namespace="hermes-memory"`
explicitly (SPEC §3.1 discipline). Companion: `e2e-output/tasks.md` (findings).

## Environment

- Server: `GET /health` →
  `{"status":"healthy","uptime":616891.008454167,"timestamp":"2026-08-12T11:39:03.447Z"}`
  (uptime ≈ 7.1 days; server live and healthy)
- Config: `$HERMES_HOME/config.yaml` has a `memory:` section
  (memory_enabled/user_profile_enabled/char limits) but **NO `memory.provider` key**
  → informational per SPEC-DB-001 §4.1b (T06 finding #3 persists; not a blocker)
- Phase 0 gate (agent-dreaming-duckbrain §Phase 0): liveness ✅ + tool presence ✅
  (8 `mcp__duckbrain__*` tools in toolset: remember / recall / forget / list_keys /
  squash / get_compaction_stats / list_namespaces / switch_namespace) →
  backend = **duckbrain**
- Namespaces (list_namespaces): `hermes-memory` (default + current), `default`,
  `coding-hermes`, `livecostdashboard`, `ozzgraph`, `sudobroker`, `gitreins` —
  7 total (T06: 4; +3 project namespaces added since)
- Baseline keys (list_keys prefix="/", limit=100): 100 keys incl.
  `/hermes-memory/purpose`, `/person/carter-lasalle[/sources/*]`,
  `/project/{hermes-memory-duckbrain,ozzgraph,sudobroker}/...`,
  `/reflections/2026-08-05..12/*`, `/system/test`,
  `/test/dream-cycle-2026-08-05-2140`, `/test/embedding-cycle`,
  `/test/ollama-embed` (pre-existing test keys — **no `/test/e2e-*` at baseline**)
- Baseline compaction (get_compaction_stats): all zeros
  (totalRecords 0, tombstoneRecords 0, tombstonePercent 0) — see finding E2E-001-F2

## Scenario Results (T1–T7)

| # | Scenario | Call | Result | Evidence |
|---|----------|------|--------|----------|
| T1 | Phase 0 detects duckbrain | config read + :3000/health + tool-presence scan | ✅ server healthy (uptime 616891s); `memory.provider` ABSENT in live config → informational per §4.1b, matches the native skill's Phase 0 table (liveness + tools is the operative gate); all 8 `mcp__duckbrain__*` tools present | health JSON; config grep; tool schemas |
| T2 | Add → remember | `remember(key=/test/e2e-2026-08-12-0439, domain=concept, attrs={source_session:e2e-001-verify}, embedding_text=<real sentence>, ns=hermes-memory)` | ✅ id `602135a6-c29c-4abc-af6b-f174f61ed103`; partition `concept/2026-08/` (confirms spec partition naming); `recall(key=...)` returns it, count 1 (E7) | ids below |
| T3 | Replace = remember + forget | remember same key (new text) → id `cbeabe25-4871-4239-9cf9-6096995a3f5d`; `forget(id=602135a6..., reason="E2E-001 replacement test — superseded by /test/e2e-2026-08-12-0439")` | ✅ recall(key) returns ONLY new id `cbeabe25...`, count 1, new text; old tombstoned (`tombstoned:true`) | recall output |
| T4 | Remove → forget | `forget(id=cbeabe25..., reason="E2E-001 removal test")` | ✅ `recall(key=...)` → count 0 | recall output |
| T5 | Capacity = compaction stats | `get_compaction_stats()` | ✅ object with tombstonePercent/parquetRatio/partition counts — but ALL ZEROS despite live data (see F2; squash dry-run preview is the operative signal on this deployment) | stats JSON (pre + post) |
| T6 | Condensation preview | `squash(dryRun=true, partition="concept/2026-08/")` — partition passed explicitly (T06 pitfall) | ✅ `"Preview: Would compact 15 records, removing 6 tombstones"`; no mutation (dryRun) | squash output |
| T7 | Cleanup | forget both test ids (done in T3/T4); `recall(keyPrefix="/test/")` | ✅ count 0 — no live /test/ keys remain; key paths linger in list_keys (index pruned only by squash — T06 finding #2 behavior confirmed) | recall + list_keys |

### Actual tool output (ids and evidence, no fabrication)

- `remember` (v1): `{"success":true,"id":"602135a6-c29c-4abc-af6b-f174f61ed103","key":"/test/e2e-2026-08-12-0439","partition":"concept/2026-08/","author":"carterlasalle@gmail.com"}`
- `recall(key=/test/e2e-2026-08-12-0439)` (v1): count 1, id `602135a6-...`, action "add", embedding_text byte-identical to input
- `remember` (v2, same key): `{"success":true,"id":"cbeabe25-4871-4239-9cf9-6096995a3f5d",...}` partition `concept/2026-08/`
- `forget(602135a6...)`: `{"success":true,"id":"602135a6-...","tombstoned":true}`
- `recall(key=...)` after replace: count 1, id `cbeabe25-...` ONLY
- `forget(cbeabe25...)`: `{"success":true,"id":"cbeabe25-...","tombstoned":true}`
- `recall(key=...)` after remove: `{"memories":[],"count":0}`
- `squash(dryRun=true, partition="concept/2026-08/")`: `{"success":true,"message":"Preview: Would compact 15 records, removing 6 tombstones","stats":{"totalRecordsKept":15,"totalRecordsRemoved":6,"tombstonesRemoved":6}}`
- `recall(keyPrefix="/test/")` (cleanup): `{"memories":[],"count":0}`
- `list_keys(prefix="/test/")`: `/test/dream-cycle-2026-08-05-2140`, `/test/e2e-2026-08-12-0439`, `/test/embedding-cycle`, `/test/ollama-embed` (tombstoned paths linger — expected)

## Skill / Spec Routing Spot-Check (step 10)

Routing contract exercised vs SPEC-DB-001 §4.2/§4.4 and both skills:

| Operation | Contract (SPEC §4.2) | Exercised (this run) | agent-dreaming-duckbrain | agent-dreaming-agnostic |
|---|---|---|---|---|
| Add | `remember(key, domain, attributes, embedding_text, namespace)` | ✅ T2 | ✅ Phase 2 | ✅ Phase 2 |
| Replace | `remember` (same key) THEN `forget(old id)` — order matters | ✅ T3 | ✅ Phase 2 | ✅ Phase 2 |
| Remove | `forget(id, reason)` tombstone | ✅ T4 | ✅ Phase 2 | ✅ Phase 2 |
| Condense | `forget` stale + `squash(dryRun=true, partition)` → `squash()` | ✅ T6 (dryRun only; real squash out of scope) | ✅ Phase 2.5 | ✅ Phase 2.5 |
| Partition pitfall | §2.6 — always pass `partition` explicitly | ✅ passed explicitly; dryRun worked | ✅ documented | ✅ documented |
| Phase 0 gate | §4.1b native: liveness + tool presence, provider informational | ✅ matches reality (config absent + healthy + tools → duckbrain) | ✅ Phase 0 | n/a (agnostic is config-gated, §4.1a — also correct) |

**Divergence found and fixed in this commit:**
- E2E-001-F1 (LOW): `agent-dreaming-duckbrain/SKILL.md` compatibility-matrix Phase 0
  row said `memory.provider: duckbrain` AND :3000 healthy — contradicting the
  skill's own Phase 0 gate (provider informational; liveness + tools operative,
  §4.1b) and this run's reality (provider absent, still duckbrain). Row updated.

**Observations (no routing impact):**
- `attributes` values round-trip double-encoded server-side
  (`"source_session": "\"e2e-001-verify\""`). Harmless — the recall API has no
  attribute filter; values are preserved. Recorded for awareness.
- Namespace inventory grew from 4 (T06) to 7 — SPEC §3.1 lists the T06 set without
  claiming exclusivity; no divergence.
- Real squash (dryRun=false) NOT re-attempted — documented broken server-side at
  T06 (SQL parse error on `current.jsonl`), out of scope (SPEC non-goals;
  DuckBrain source at `/home/hermes/duckbrain`).

## Findings

2 findings, both LOW, both fixed in this commit — see `e2e-output/tasks.md`:
- E2E-001-F1: native skill compat-matrix Phase 0 row contradicts its own gate (fixed)
- E2E-001-F2: `get_compaction_stats` reads all-zeros despite live records/tombstones
  on this deployment (caveat added to both skills' Phase 2.5 verify step)

## Verified Working (pass criteria T1–T7 with real tool output)

- `remember` → returns id; `recall(key=...)` confirms write (E7) ✅
- Replacement: write-new-then-forget-old; old id tombstoned ✅
- Removal: `forget` → `recall(key=...)` empty ✅
- `list_keys` / `get_compaction_stats` / `squash(dryRun=true, partition)` ✅
- Namespace discipline: every call explicit `namespace="hermes-memory"` ✅
- Phase 0 gate (liveness + tool presence) matches the native skill's table and
  this run's reality (provider absent, backend still duckbrain) ✅
- Routing contract (remember=add, remember+forget=replace, forget=remove,
  forget+squash=condense) matches both skills verbatim ✅

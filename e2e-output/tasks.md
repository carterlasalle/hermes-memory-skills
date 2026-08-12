# E2E-001 — Findings

Status: **2 findings, both LOW severity, both fixed in this commit.**

Full evidence: `e2e-output/report.md`. Run: 2026-08-12 against live DuckBrain
`http://127.0.0.1:3000`, namespace `hermes-memory` (SPEC §3.1 — explicit on every call).

---

## E2E-001-F1 — agent-dreaming-duckbrain compat-matrix Phase 0 row contradicts its own gate

- **Severity:** LOW (doc inconsistency; the skill's authoritative Phase 0 section is correct)
- **File:** `agent-dreaming-duckbrain/SKILL.md` — "Compatibility Matrix — DuckBrain row"
- **Reproduction:**
  1. Read the skill's Phase 0 section: the operative gate is **server liveness +
     tool presence**, with config `memory.provider` "informational, NOT a blocker"
     (SPEC-DB-001 §4.1b; T06 finding #3 fix 3e4f6e9).
  2. Read the same file's compatibility matrix row: "Phase 0 detection |
     `memory.provider: duckbrain` AND :3000 healthy".
  3. This E2E-001 run: live config **omits** `memory.provider` yet the backend IS
     duckbrain (healthy :3000 + all 8 `mcp__duckbrain__*` tools present).
- **Expected:** matrix row matches the skill's own Phase 0 gate (liveness + tool
  presence; provider informational) and SPEC §4.1b.
- **Actual:** row requires `memory.provider: duckbrain`, which would classify this
  deployment (provider absent) as NOT duckbrain — contradicting the skill's own
  gate and this run's verified reality. The row was copied verbatim from SPEC §4.5
  (which is scoped to the **agnostic** skill's config-driven gate).
- **Fix (applied):** row now reads "` :3000 healthy AND `mcp__duckbrain__*` tools
  present (config provider informational — see Phase 0 / SPEC-DB-001 §4.1b)".

---

## E2E-001-F2 — get_compaction_stats reads all-zeros despite live records and tombstones

- **Severity:** LOW (operational doc gap; server-side behavior, DuckBrain source out of scope)
- **Files:** both skills' Phase 2.5 step 7 ("Verify" via `get_compaction_stats`);
  SPEC-DB-001 §2.5/§4.3 capacity model relies on this endpoint.
- **Reproduction:**
  1. Pre-cycle `get_compaction_stats()` → all zeros (totalRecords 0,
     tombstoneRecords 0, tombstonePercent 0, parquetRatio 0).
  2. This cycle wrote 2 records and created 2 tombstones in
     `concept/2026-08/current.jsonl` (ids `602135a6…`, `cbeabe25…`).
  3. Post-cycle `get_compaction_stats()` → **still all zeros**.
  4. Yet `squash(dryRun=true, partition="concept/2026-08/")` →
     "Would compact 15 records, removing 6 tombstones" — the data demonstrably
     exists and the tombstones are real.
- **Expected (per SPEC §2.5/§4.3):** the endpoint reports tombstone % /
  partition health reflecting stored memories, so Phase 2.5's "tombstone % high"
  trigger can fire.
- **Actual:** the endpoint counts compacted storage only on this deployment and
  reads all-zeros until a real squash has run — the Phase 2.5 trigger can never
  fire from stats alone. T06 recorded the same zeros as "honest, no compaction
  yet"; this run's squash preview proves that interpretation wrong. The T06
  pitfall note for squash already made `partition` effectively required; the
  stats gap was undocumented.
- **Fix (applied):** both skills' Phase 2.5 step 7 now carry a caveat — when stats
  read zero but writes/tombstones are known, treat the `squash(dryRun=true,
  partition=...)` preview ("Would compact N records, removing M tombstones") as
  the operative bloat signal.
- **Note:** this is DuckBrain server-side behavior (source `/home/hermes/duckbrain`),
  out of scope for this markdown repo per SPEC-DB-001 non-goals — no server change
  possible here; the doc caveat keeps dream cycles honest until the server is fixed.

---

All other scenarios (T1–T7) passed with real tool output; no other skill/spec
divergences found. Routing contract verified: remember=add,
remember+forget=replace, forget=remove, forget+squash=condense.

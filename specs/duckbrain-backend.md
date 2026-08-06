# SPEC-DB-001 — DuckBrain Backend Routing for agent-dreaming

## 1. Purpose

Add **DuckBrain** (git-backed, DuckDB/VSS persistent memory at `http://127.0.0.1:3000`)
as a first-class memory backend for the agent-dreaming skills. A worker reading
this spec must be able to:

1. Create `agent-dreaming-duckbrain/SKILL.md` (the DuckBrain-native dreaming skill)
2. Extend `agent-dreaming-agnostic/SKILL.md` Phase 0 / Phase 2 / Phase 2.5 /
   compatibility matrix to route to DuckBrain MCP tools
3. Write `agent-dreaming-agnostic/references/duckbrain-backend.md` (T04)
4. Verify the routing with a live end-to-end dream cycle (T06)

**without asking a single clarifying question.**

Non-goals (out of scope for this spec):
- Modifying DuckBrain itself (source lives at `/home/hermes/duckbrain`).
- Breaking the built-in MEMORY.md or Holographic routing — this fork ADDS a
  backend; the other two must keep working unchanged.
- Building a test suite — this is a markdown skills repo (see AGENTS.md).

## 2. Interface — DuckBrain MCP Tools (exact signatures)

The DuckBrain MCP server exposes these tools to an agent. **These are the ONLY
tools the duckbrain backend routes to. NEVER route to `memory()` or
`fact_store()` for the duckbrain backend.**

### 2.1 remember (write — add)

```
mcp__duckbrain__remember(
  key: string,            # REQUIRED — hierarchical key path, e.g. "/user/preference"
  domain: string,         # REQUIRED — enum: person|event|concept|message|config|raw_note
  attributes: object,     # REQUIRED — free-form object; may be {} but MUST be present
  embedding_text: string, # REQUIRED — text that gets embedded (VSS retrieval axis)
  namespace: string       # OPTIONAL — namespace to write to; default = current
)
```

Rules:
- `embedding_text` is REQUIRED and non-empty — without it the fact cannot be
  found by semantic `recall(query=...)`. This is the DuckBrain equivalent of
  "entities are mandatory" for holographic.
- `domain` must be one of the six enum values. Recommended mapping:
  - user preference → `person`
  - environment/service fact → `config`
  - project/workspace fact → `concept`
  - one-time occurrence → `event`
  - session exchange (traceability) → `message`
  - unstructured snippet → `raw_note`
- `attributes` may be empty (`{}`) but the parameter must be passed. Suggested
  attribute keys when used: `source_session`, `dream_phase`, `backend`.

### 2.2 recall (read — list / search / exact)

```
mcp__duckbrain__recall(
  key: string,            # OPTIONAL — exact key lookup
  id: string,             # OPTIONAL — exact UUID lookup
  keyPrefix: string,      # OPTIONAL — prefix glob, e.g. "/projects/"
  domain: string,         # OPTIONAL — domain filter (enum as above)
  query: string,          # OPTIONAL — semantic search (DuckDB VSS extension)
  limit: number,          # OPTIONAL — default 10, max results
  namespace: string       # OPTIONAL — namespace to query
)
```

Usage in the skill:
- **Baseline (Phase 1 step 3):** `recall(keyPrefix="/", limit=200)` — broad view
  of stored memories in the namespace.
- **Novelty check (Phase 2 step 2):** `recall(query="<candidate keywords>", limit=10)`
  — semantic similarity check.
- **Exact verification (post-promotion):** `recall(key="<the key just written>")`
  — confirm the write landed.
- At least one of `key | id | keyPrefix | query` must be provided; otherwise the
  call is a no-op and returns empty results.

### 2.3 list_keys (explore hierarchy)

```
mcp__duckbrain__list_keys(
  prefix: string,   # OPTIONAL — default "/"
  maxDepth: number, # OPTIONAL — default 3
  limit: number,    # OPTIONAL — default 50
  offset: number,   # OPTIONAL — default 0
  namespace: string # OPTIONAL
)
```

Used for: baseline key inventory, condensation candidate discovery, verifying
the key hierarchy is being respected.

### 2.4 forget (remove — tombstone)

```
mcp__duckbrain__forget(
  id: string,      # REQUIRED — UUID of the memory to tombstone
  reason: string,  # OPTIONAL — why (recorded in tombstone)
  namespace: string, # OPTIONAL — namespace to search
  domain: string   # OPTIONAL — domain filter (optimization)
)
```

- `forget` is a TOMBSTONE — the row is marked deleted, not physically removed.
- The `id` MUST come from a prior `recall`/`list_keys` result (never fabricated).
- `reason` should be set on every forget for auditability (e.g. "contradicted by
  session <id>", "stale — superseded by /project/foo/decision-3").

### 2.5 get_compaction_stats (capacity check — THE capacity model)

```
mcp__duckbrain__get_compaction_stats()   # no parameters
```

Returns repository compaction statistics: tombstone percentage, Parquet ratio,
partition health. This is the DuckBrain **capacity signal** — DuckBrain has no
char limit (unlike built-in MEMORY.md). Details of exact response fields are
dynamic; the skill MUST treat the returned object as authoritative and use the
fields that exist (tombstone %, partition count, parquet ratio when present).

### 2.6 squash (condensation — Phase 2.5)

```
mcp__duckbrain__squash(
  partition: string,  # OPTIONAL — specific partition to squash
  dryRun: boolean,    # OPTIONAL — default false; preview without changes
  aggressive: boolean # OPTIONAL — default false; squash git history aggressively
)
```

- `squash(dryRun=true)` is the preview — ALWAYS run preview first, inspect what
  would change, then run real squash.
- `aggressive=true` squashes git history — NEVER use aggressive by default;
  reserve for explicit user approval or extreme tombstone ratios (see 5.3).

### 2.7 namespace tools

```
mcp__duckbrain__list_namespaces()          # no parameters — all namespaces
mcp__duckbrain__switch_namespace(name: string)  # REQUIRED param — change current
```

## 3. Data Model — Namespace / Key / Domain Strategy

### 3.1 Namespaces (the routing axis for isolation)

Live namespaces (verified 2026-08-05): `hermes-memory` (default), `default`,
`coding-hermes`, `livecostdashboard`.

| Scope of the dream | namespace to use |
|---|---|
| General user memory (default) | `hermes-memory` |
| A specific project | that project's namespace (e.g. `coding-hermes`) |
| Anything else | `hermes-memory` |

Rules:
- **Always pass `namespace` explicitly** on every remember/recall/forget call.
  Never rely on the current-namespace default — a `remember` without explicit
  namespace lands in whatever namespace the session last switched to.
- **Never write cross-project fleet telemetry into `hermes-memory`** (user
  preference, AGENTS.md).
- Before a dream cycle targeting a non-default namespace, call
  `switch_namespace(name=<ns>)` AND pass `namespace=<ns>` explicitly on every
  write — belt and suspenders.

### 3.2 Keys (hierarchical retrieval axis)

Keys ARE a retrieval axis alongside VSS `query`. Use hierarchical, slash-
delimited, kebab-case keys:

```
/user/preference/<topic>            # user preferences, habits
/environment/<service>/<fact>       # ports, paths, service facts
/project/<name>/<decision-N>        # project decisions, numbered
/project/<name>/<component>/...     # project component facts
/tool/<name>/<fact>                 # tool quirks, workarounds
/session/<id>/...                   # (rare) session traceability
```

Rules:
- `list_keys(prefix="/project/foo/")` must return a meaningful subtree for
  condensation scans — keep keys granular enough to group, coarse enough to scan.
- A replacement of fact X at key K: write the new version at the SAME key K
  (new id), then `forget` the old id. Keys are not unique — ids are.
- Do NOT put free text in keys; free text goes in `embedding_text`.

### 3.3 Domains

Use the six-value enum (`person|event|concept|message|config|raw_note`) per the
mapping in 2.1. `domain` is a filter axis for recall and forget; inconsistent
domain usage silently fragments retrieval.

## 4. Wiring — Where the Routing Contract Lives

### 4.1 Phase 0 detection (in agent-dreaming-agnostic AND agent-dreaming-duckbrain)

Detection is a **two-step check** — config value AND liveness:

```
Step A — read $HERMES_HOME/config.yaml memory.provider:
  grep -A10 "^memory:" $HERMES_HOME/config.yaml | grep "provider:" | awk '{print $2}'
  → "duckbrain" → candidate duckbrain backend
  → "" | null | absent → built-in
  → "holographic" → holographic
  → anything else → unknown → fall back to built-in

Step B — if provider == "duckbrain", verify liveness:
  curl -s -m 5 http://127.0.0.1:3000/health
  → {"status":"healthy",...} → DUCKBRAIN ACTIVE
  → connection refused / timeout → log "duckbrain configured but server down —
     fallback to built-in" and route to built-in memory()
```

**Detection result table (must appear in BOTH skill files):**

| Config `memory.provider` | DuckBrain :3000 healthy? | Backend | Phase 2 tools |
|---|---|---|---|
| `duckbrain` | yes | **duckbrain** | remember / recall / forget / list_keys / squash |
| `duckbrain` | no | built-in (fallback) | `memory()` |
| `""` / null / absent | — | built-in | `memory()` |
| `holographic` | — | holographic | fact_store / fact_feedback |
| other | — | built-in (fallback) | `memory()` |

Also verify tool availability: if provider is `duckbrain` and the `mcp__duckbrain__*`
tools are NOT in the toolset (e.g. cron sessions where MCP tools may be absent),
fall back to built-in and log it.

### 4.2 Phase 2 promotion routing (duckbrain rows)

| Operation | DuckBrain call | Notes |
|---|---|---|
| New entry | `remember(key, domain, attributes, embedding_text, namespace)` | embedding_text = full candidate text; key per 3.2 |
| Replacement | `remember(...)` (same key, new text) THEN `forget(id=<old>, reason="superseded by <key>")` | two calls, order matters: write new, tombstone old |
| Removal | `forget(id=<id>, reason="<why>")` | id from prior recall/list_keys |
| Baseline/verify | `recall(keyPrefix="/", limit=200, namespace=...)` | Phase 1 baseline + post-promotion check |
| Novelty check | `recall(query="<keywords>", limit=10, namespace=...)` | semantic overlap test |
| Capacity check | `get_compaction_stats()` | see 4.3 |

### 4.3 Capacity = compaction stats (replaces char-limit model)

DuckBrain has **no char limit**. Capacity is **compaction health**:

| get_compaction_stats signal | Interpretation | Action |
|---|---|---|
| Low tombstone % | healthy | no action |
| Tombstone % high (exact threshold dynamic — use the report's own health field when present; otherwise treat "high" as a flagged/unhealthy partition) | bloat accumulating | run Phase 2.5 condensation |
| New write confirmed | — | `recall(key="<key>")` returns the entry |

Phase 2 post-promotion check for duckbrain:
1. `recall(key="<key written>", namespace=...)` — confirm the entry is retrievable.
2. `get_compaction_stats()` — record tombstone % in the dream diary.

### 4.4 Phase 2.5 condensation (duckbrain rows)

| Step | DuckBrain call |
|---|---|
| Inventory | `list_keys(prefix="/", maxDepth=3, limit=200, namespace=...)` |
| Candidate detail | `recall(keyPrefix="<subtree>", limit=100, namespace=...)` |
| Contradiction resolution | `recall(query="<claim>", limit=10)` + `session_search` to pick winner |
| Tombstone stale/contradicted | `forget(id=<id>, reason="<evidence>")` |
| Preview compaction | `squash(dryRun=true)` |
| Execute compaction | `squash()` (dryRun=false) — ONLY when tombstone % is high / health flag set |
| Verify | `get_compaction_stats()` — confirm tombstone % dropped |

Trigger table (Phase 2.5 when-to-run for duckbrain):

| Condition | Action |
|---|---|
| Every dreaming cycle | Light pass: review tombstone %, flag candidates |
| Tombstone % high (per health field) | Deep condensation: forget stale ids, then `squash(dryRun=true)` → `squash()` |
| User explicitly requests | Full condensation incl. `aggressive=true` only with explicit approval |

### 4.5 Compatibility matrix rows (must be added to agent-dreaming-agnostic)

| Feature | DuckBrain |
|---|---|
| Phase 0 detection | `memory.provider: duckbrain` AND :3000 healthy |
| Phase 1 session review | ✅ (same as other backends) |
| Phase 2: add | `remember(key, domain, attributes, embedding_text, namespace)` |
| Phase 2: replace | `remember` (same key) + `forget` old id |
| Phase 2: remove | `forget(id, reason)` tombstone |
| Phase 2: list/verify | `recall(keyPrefix="/")` + `list_keys` |
| Capacity check | `get_compaction_stats()` (tombstone %, parquet ratio) |
| Bloat remediation | `forget` + `squash(dryRun=true)` → `squash()` |
| Phase 3 REM | ✅ |
| Honcho/Mem0 fallback | N/A (falls back to built-in) |

## 5. Error Catalog

| # | Condition | What the skill must do |
|---|---|---|
| E1 | `memory.provider: duckbrain` but :3000 unreachable | Fall back to built-in `memory()`; log "duckbrain server down — fallback" in dream artifact backend section |
| E2 | `memory.provider: duckbrain` but `mcp__duckbrain__*` tools absent from toolset | Fall back to built-in; log reason (toolset gap) |
| E3 | `remember` called without `embedding_text` | NEVER happens — spec mandates embedding_text; if a worker omits it, treat as spec violation |
| E4 | `recall` called with no selector (no key/id/keyPrefix/query) | Returns empty; the skill must always pass at least one selector |
| E5 | `forget` called with a fabricated id | NEVER — id must come from a prior recall/list_keys result; document the source id in the diary |
| E6 | `squash` called without prior `dryRun=true` preview | NEVER — preview is mandatory before real squash |
| E7 | Write verification fails (`recall(key=...)` empty after remember) | Re-write once; if still empty, log failure in diary and DO NOT claim success (anti-fabrication rule) |
| E8 | Cron session (skip_memory=True): MCP tools may be unavailable | Detection still runs; if duckbrain tools absent, fall back to built-in MEMORY.md patching; note that scheduled cron cannot consolidate duckbrain memories — manual run (`hermes cron run <id>`) restores full access |

## 6. Edge Cases

- **Namespace drift:** current namespace changed by another tool mid-cycle → every
  call passes `namespace` explicitly, so drift cannot corrupt writes.
- **Duplicate keys:** replacement writes new id at same key — old id forgotten.
  Verify old id is tombstoned (list_keys/recall no longer returns it live).
- **Empty embedding_text:** never written (E3). If semantic recall of a candidate
  returns nothing, still write — the key/domain filters remain valid axes.
- **High tombstone ratio but user data critical:** run `squash(dryRun=true)`
  FIRST and inspect; never aggressive without explicit approval.
- **Multiple namespaces for one dream:** a dream about project X writes only to
  namespace X; general insights to `hermes-memory`. Never split one fact across
  two namespaces.
- **Recall returning no results for a legitimately new fact:** that IS the
  novelty signal — proceed with promotion (empty recall ≠ failure; failure is
  empty recall AFTER the write).
- **Concurrent cycles:** two dreaming runs overlapping → both pass namespace
  explicitly; keys are hierarchical so same-key writes are last-write-wins with
  the older id tombstoned; no lock needed.

## 7. Testing — Exact Scenarios (T06 end-to-end verification)

The test is a LIVE dream cycle against `http://127.0.0.1:3000` in the
`hermes-memory` namespace. Record evidence (key ids) in the dream diary.

| # | Scenario | Steps | Expected |
|---|---|---|---|
| T1 | Phase 0 detects duckbrain | read config `memory.provider` + curl :3000/health | backend == duckbrain, logged in artifact |
| T2 | Add routes to remember | `remember(key="/test/dream-cycle-<ts>", domain="concept", attributes={"source_session":"t06-verify"}, embedding_text="<test fact>", namespace="hermes-memory")` | returns id; recall(key=...) returns the entry |
| T3 | Replace = remember + forget | write same key new text; `forget(id=<old>, reason="T06 replacement test")` | new id live, old id tombstoned |
| T4 | Remove routes to forget | `forget(id=<test id>, reason="T06 removal test")` | recall(key=...) no longer returns it |
| T5 | Capacity = compaction stats | `get_compaction_stats()` | returns object with tombstone % / partition info; recorded in diary |
| T6 | Condensation preview | `squash(dryRun=true)` | returns preview without mutating |
| T7 | Cleanup | forget ALL test ids created this cycle | no test keys remain in hermes-memory namespace |

Verification pass criteria: T1–T6 all observed with real tool output; T7 leaves
zero `/test/` keys behind. The test diary entry must include the actual key ids,
not "simulated" or "assumed" results.

## 8. Hilo Impact

- **Depends on this spec:** T02 (`agent-dreaming-duckbrain/SKILL.md`), T03
  (agnostic extension), T04 (references doc), T05 (README), T06 (E2E test).
- **This spec depends on:** AGENTS.md routing contract (source of truth),
  live DuckBrain MCP tool surface (verified 2026-08-05), the existing
  `agent-dreaming-agnostic/SKILL.md` phase structure.
- **Files touched by downstream tasks:**
  - NEW: `agent-dreaming-duckbrain/SKILL.md`
  - EDIT: `agent-dreaming-agnostic/SKILL.md` (Phase 0 table, Phase 2 routing,
    Phase 2.5 table, compatibility matrix, pitfalls)
  - NEW: `agent-dreaming-agnostic/references/duckbrain-backend.md`
  - EDIT: `README.md` (backend matrix, detection, installation, cron notes)
- Hilo re-indexes automatically via post-commit hook; no manual action.

## 9. Anti-Fabrication Rules (inherited from AGENTS.md)

1. Never claim a DuckBrain write succeeded without `recall` confirming it (E7).
2. Never invent key ids — record the actual returned ids in the diary.
3. Never claim condensation happened without `get_compaction_stats()` before/after.
4. Cron limitation (E8) must be documented, not silently ignored.

# DuckBrain Memory Backend Reference

Reference for the DuckBrain memory provider — tools, schema, namespaces,
key/domain strategy, vector search (VSS), and compaction. Used by
`agent-dreaming-agnostic` Phase 0 detection and Phase 2 / Phase 2.5 routing.

DuckBrain is a git-backed persistent memory server at
`http://127.0.0.1:3000`. Storage is JSONL partitions + DuckDB (with the VSS
extension for semantic search). It is exposed to agents as `mcp__duckbrain__*`
MCP tools. Source lives at `/home/hermes/duckbrain` (not modified by this fork).

## Tools

All tool signatures below are the exact MCP surface (verified against the live
server 2026-08-05). These are the ONLY tools the duckbrain backend routes to —
never route to `memory()` or `fact_store()` when the duckbrain backend is active.

### `remember` — write (add / replace)

| Param | Required | Notes |
|-------|----------|-------|
| `key` | ✅ | Hierarchical key path, e.g. `/user/preference` |
| `domain` | ✅ | Enum: `person` \| `event` \| `concept` \| `message` \| `config` \| `raw_note` |
| `attributes` | ✅ | Free-form object; may be `{}` but MUST be passed |
| `embedding_text` | ✅ | Text that gets embedded — the VSS retrieval axis. Non-empty, always |
| `namespace` | — | Namespace to write to; ALWAYS pass explicitly |

Returns: `{success, id}` — the new memory's UUID. `embedding_text` is
REQUIRED and non-empty; without it the fact cannot be found by semantic
`recall(query=...)`. This is the DuckBrain equivalent of "entities are
mandatory" for holographic.

### `recall` — read (list / search / exact)

| Param | Required | Notes |
|-------|----------|-------|
| `key` | — | Exact key lookup |
| `id` | — | Exact UUID lookup |
| `keyPrefix` | — | Prefix glob, e.g. `/projects/` |
| `domain` | — | Domain filter (enum above) |
| `query` | — | Semantic search (DuckDB VSS extension) |
| `limit` | — | Max results (default 10) |
| `namespace` | — | Namespace to query |

At least one of `key | id | keyPrefix | query` MUST be provided; otherwise the
call is a no-op and returns empty results. Usage in the skill:

- **Baseline (Phase 1 step 3):** `recall(keyPrefix="/", limit=200)`
- **Novelty check (Phase 2 step 2):** `recall(query="<candidate keywords>", limit=10)`
- **Write verification (post-promotion):** `recall(key="<the key just written>")`

### `list_keys` — explore hierarchy

| Param | Required | Notes |
|-------|----------|-------|
| `prefix` | — | Default `/` |
| `maxDepth` | — | Default 3 |
| `limit` | — | Default 50 |
| `offset` | — | Default 0 |
| `namespace` | — | |

Used for: baseline key inventory, condensation candidate discovery, verifying
the key hierarchy is being respected.

### `forget` — remove (tombstone)

| Param | Required | Notes |
|-------|----------|-------|
| `id` | ✅ | UUID of the memory to tombstone |
| `reason` | — | Why (recorded in tombstone) — set it for auditability |
| `namespace` | — | Namespace to search |
| `domain` | — | Domain filter (optimization) |

`forget` is a TOMBSTONE — the row is marked deleted (action=`tombstone`), not
physically removed. The `id` MUST come from a prior `recall`/`list_keys`
result (never fabricated).

### `get_compaction_stats` — capacity check (THE capacity model)

No parameters. Returns repository compaction statistics. Live shape
(verified 2026-08-05):

```json
{
  "success": true,
  "stats": {
    "totalSize": 0, "totalPartitions": 0,
    "parquetPartitions": 0, "jsonlPartitions": 0,
    "totalRecords": 0, "tombstoneRecords": 0,
    "tombstonePercent": 0, "parquetRatio": 0,
    "oldPartitions": [], "largePartitions": []
  }
}
```

This is the DuckBrain **capacity signal** — DuckBrain has no char limit (unlike
built-in MEMORY.md). The skill MUST treat the returned object as authoritative
and use the fields that exist (tombstone %, partition count, parquet ratio when
present).

### `squash` — condensation (Phase 2.5)

| Param | Required | Notes |
|-------|----------|-------|
| `partition` | — | Specific partition to squash |
| `dryRun` | — | Default false; preview without changes |
| `aggressive` | — | Default false; squash git history aggressively |

- `squash(dryRun=true)` is the preview — ALWAYS run preview first, inspect what
  would change, then run real squash.
- `aggressive=true` squashes git history — NEVER use by default; reserve for
  explicit user approval or extreme tombstone ratios.

### Namespace tools

| Tool | Notes |
|------|-------|
| `list_namespaces()` | No parameters — all namespaces |
| `switch_namespace(name)` | REQUIRED param — change current namespace |

## Configuration

In `$HERMES_HOME/config.yaml`:

```yaml
memory:
  provider: duckbrain
```

Liveness check (Phase 0 two-step detection):

```bash
curl -s -m 5 http://127.0.0.1:3000/health
# → {"status":"healthy","uptime":...,"timestamp":"..."}
```

If the server is unreachable (connection refused / timeout), fall back to
built-in `memory()` and log `duckbrain server down — fallback` (E1). If the
`mcp__duckbrain__*` tools are absent from the toolset (e.g. cron sessions,
`skip_memory=True`), also fall back to built-in and log the reason (E2/E8).

## Data Model — Schema

Ground truth from `duckbrain/src/schema/memory.ts`. Hybrid schema: strict base
fields + flexible attributes.

| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID v4 | Unique identifier — ids are unique, keys are NOT |
| `key` | string | Filesystem-style hierarchical path, MUST start with `/` |
| `domain` | enum | `person` \| `event` \| `concept` \| `message` \| `config` \| `raw_note` |
| `timestamp` | ISO-8601 | Record creation time |
| `author` | git email | Git authorship attribution |
| `action` | enum | `add` \| `update` \| `tombstone` |
| `embedding_text` | string | Text content for vector embedding generation |
| `attributes` | JSON object | Flexible structured data, default `{}` |

Storage layout: JSONL files partitioned by
`namespace/domain/partition/chunk.jsonl`, with time-based (`YYYY-MM`) or
key-based partitioning. DuckDB (with VSS extension) is built from the JSONL at
query time and provides semantic search. The repository is git-backed — every
write is a git operation, which is why `squash(aggressive=true)` rewrites
history.

## Namespace Strategy

Live namespaces (verified 2026-08-05): `hermes-memory` (default + current),
`default`, `coding-hermes`, `livecostdashboard`.

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

## Key Strategy

Keys ARE a retrieval axis alongside VSS `query`. Use hierarchical,
slash-delimited, kebab-case keys:

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

## Domain Strategy

Use the six-value enum (`person|event|concept|message|config|raw_note`).
Recommended mapping:

| Kind of fact | domain |
|---|---|
| User preference | `person` |
| Environment/service fact | `config` |
| Project/workspace fact | `concept` |
| One-time occurrence | `event` |
| Session exchange (traceability) | `message` |
| Unstructured snippet | `raw_note` |

`domain` is a filter axis for recall and forget; inconsistent domain usage
silently fragments retrieval (domains are top-level partition folders on disk).

## VSS — Vector Similarity Search

DuckBrain's semantic retrieval uses DuckDB's VSS extension over embeddings
computed from `embedding_text`:

- **Write side:** `remember(embedding_text="<full candidate text>")` — the text
  is embedded at write time. Non-empty `embedding_text` is mandatory (E3).
- **Read side:** `recall(query="<keywords>", limit=10)` — semantic similarity
  ranking over stored embeddings.
- **Novelty check:** empty semantic results for a genuinely new fact IS the
  novelty signal — proceed with promotion. Empty recall AFTER a write is a
  failure (E7), not a novelty signal.
- VSS complements, not replaces, the key axis: `key`/`keyPrefix` are exact/glob
  lookups; `query` is ranked similarity. Use both for condensation scans.

## Compaction Model

DuckBrain has **no char limit** — capacity is **compaction health**:

| Signal | Interpretation | Action |
|---|---|---|
| Low tombstone % | healthy | no action |
| Tombstone % high (per health field) | bloat accumulating | run Phase 2.5 condensation: `forget` stale ids → `squash(dryRun=true)` → `squash()` |
| New write confirmed | — | `recall(key="<key>")` returns the entry |

Phase 2 post-promotion check for duckbrain:
1. `recall(key="<key written>", namespace=...)` — confirm the entry is retrievable.
2. `get_compaction_stats()` — record tombstone % in the dream diary.

Phase 2.5 condensation steps:

| Step | DuckBrain call |
|---|---|
| Inventory | `list_keys(prefix="/", maxDepth=3, limit=200, namespace=...)` |
| Candidate detail | `recall(keyPrefix="<subtree>", limit=100, namespace=...)` |
| Contradiction resolution | `recall(query="<claim>", limit=10)` + `session_search` to pick winner |
| Tombstone stale/contradicted | `forget(id=<id>, reason="<evidence>")` |
| Preview compaction | `squash(dryRun=true)` |
| Execute compaction | `squash()` (dryRun=false) — ONLY when tombstone % is high / health flag set |
| Verify | `get_compaction_stats()` — confirm tombstone % dropped |

## Comparison with Built-in MEMORY.md and Holographic

| Feature | Built-in (MEMORY.md) | Holographic | DuckBrain |
|---|---|---|---|
| Storage | Flat markdown file | SQLite database | Git-backed JSONL + DuckDB |
| Capacity | 2,200 chars hard limit | Unlimited (practical: thousands of facts) | Unlimited — capacity = compaction health (tombstone %) |
| Retrieval | Injected into system prompt (always visible) | On-demand via tool calls | On-demand via MCP tools; exact (key/prefix) + semantic (VSS query) |
| Compositional queries | No (grep at best) | Yes (HRR reason/probe/related) | Prefix globs + semantic similarity |
| Trust/decay | Manual (remove stale entries) | Automatic (trust scoring + feedback) | Manual — tombstone stale ids, then `squash` |
| Entry format | Free text, § delimited | Structured: content + category + entities + tags | Structured: key + domain + attributes + embedding_text |
| Bloat handling | Phase 2.5 Condensation (built into dreaming) | `fact_feedback(action='unhelpful')` decay | `forget` + `squash(dryRun=true)` → `squash()` |
| Offline/air-gapped | Yes (local file) | Yes (local SQLite) | Yes (local server + git) |
| Multi-profile safe | Via `$HERMES_HOME` | Via `$HERMES_HOME` | Via namespaces |

## Error Catalog (from SPEC-DB-001 §5)

| # | Condition | What the skill must do |
|---|---|---|
| E1 | `memory.provider: duckbrain` but :3000 unreachable | Fall back to built-in `memory()`; log in dream artifact backend section |
| E2 | Provider duckbrain but `mcp__duckbrain__*` tools absent | Fall back to built-in; log reason (toolset gap) |
| E3 | `remember` without `embedding_text` | NEVER happens — spec mandates embedding_text |
| E4 | `recall` with no selector | Returns empty; always pass at least one selector |
| E5 | `forget` with fabricated id | NEVER — id must come from a prior recall/list_keys result |
| E6 | `squash` without prior `dryRun=true` preview | NEVER — preview is mandatory |
| E7 | Write verification fails (`recall(key=...)` empty after remember) | Re-write once; if still empty, log failure — DO NOT claim success |
| E8 | Cron session (`skip_memory=True`): MCP tools unavailable | Fall back to built-in MEMORY.md patching; note cron cannot consolidate duckbrain memories — manual run (`hermes cron run <id>`) restores full access |

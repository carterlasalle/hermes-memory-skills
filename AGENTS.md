# AGENTS.md — hermes-memory-skills (DuckBrain fork)

Fork of [nexus9888/hermes-memory-skills](https://github.com/nexus9888/hermes-memory-skills)
adding **DuckBrain** as a first-class memory backend alongside built-in MEMORY.md
and Holographic. Keep `upstream` (nexus9888) as the sync remote; merge upstream
changes in rather than rewriting history.

## What the deliverable is

A set of Hermes Agent memory-consolidation **skills** (markdown, no code). The
key skill is `agent-dreaming-agnostic/SKILL.md` — three-phase background memory
consolidation (Light/Deep/Condensation/REM) that routes promotions to whichever
memory backend is active. The fork must add **DuckBrain** routing.

## DuckBrain — the user's live memory backend

DuckBrain is git-backed persistent memory backed by **DuckDB** (`duckdb` 1.4.4,
`src/duckdb/` VSS extension). Source lives at `/home/hermes/duckbrain`. The HTTP
server is live at `http://127.0.0.1:3000` (MCP over HTTP at `/mcp`; REST at
`/api/memories`, `/api/keys`, `/api/namespaces`, `/api/events`, `/api/search`).

Exposed MCP tools (what a foreman has in its toolset):
- `remember` — write: `{key, domain, attributes, embedding_text, namespace}`. Domain enum: `person|event|concept|message|config|raw_note`.
- `recall` — read: `{key | id | keyPrefix | domain | query, limit, namespace}`. `query` = semantic search (VSS).
- `list_keys` — explore hierarchy: `{prefix, maxDepth, limit, offset, namespace}`.
- `forget` — tombstone a memory: `{id, reason, namespace, domain}`.
- `list_namespaces` — all namespaces.
- `get_compaction_stats` — tombstone %, Parquet ratio, partition health.
- `squash` — compact partitions: JSONL→Parquet, remove tombstones, `{partition, dryRun, aggressive}`.

Live namespaces: `hermes-memory` (default), `default`, `coding-hermes`,
`livecostdashboard`. This fork's dreaming should target the `hermes-memory`
namespace (unless a project namespace is the subject).

## Backend routing contract (what the fork must implement)

The original `agent-dreaming-agnostic` routes Phase 0 detection → Phase 2
promotion → Phase 2.5 condensation per backend:

| Backend | Phase 0 detect | Phase 2 add | Phase 2 replace | Phase 2 remove | Phase 2.5 condense |
|---|---|---|---|---|---|
| built-in | `memory.provider` empty | `memory()` add | `memory()` replace | `memory()` remove | MEMORY.md char-limit trim |
| holographic | `memory.provider: holographic` | `fact_store` add | `fact_store` update | `fact_store` remove | `fact_feedback` decay |
| **duckbrain** | DuckBrain HTTP live (:3000) / namespace present | `remember` | `remember` (new key) + `forget` old | `forget` (tombstone) | `forget` + `squash` (compaction stats gate) |

DuckBrain mapping notes:
- **No char limit** — capacity = compaction health (tombstone %, Parquet ratio) via `get_compaction_stats`.
- Use hierarchical **keys** (e.g. `/user/preference`, `/project/<name>/...`, `/tool/<name>/...`) — keys ARE the retrieval axis alongside VSS `query`.
- Always set `domain` and provide `embedding_text` (the text that gets embedded) — without embedding_text the fact can't be found by semantic `recall`.
- **Namespace discipline:** default to `hermes-memory`; when a dream concerns a specific project, use that project's namespace. Never write cross-project fleet telemetry into `hermes-memory` (user preference).
- Condensation = `forget` stale/contradicted ids (tombstone) then `squash` when tombstone ratio is high; preview with `squash(dryRun=true)`.

## Foreman workflow (mandatory every tick)

1. Read `.coding-hermes/tasks.md` AND `.gitreins/tasks.yaml` (joint state).
2. SPEC tasks gate CORE: before touching CORE/verification, load `coding-hermes-specs` and ensure `specs/duckbrain-backend.md` exists with the exact routing contract above. No prose-level specs.
3. For each picked board task, ALSO create a matching GitReins task:
   `gitreins task create <ID> "<title>" "<criterion>"` → `task start` → work →
   `gitreins task complete <ID>` (fires LLM judge, writes verdict.json) → commit →
   `gitreins task delete <ID>`.
4. Commit via `gitreins commit "<msg>"` (message positional; runs Tier 1 guards) or
   plain `git commit` (pre-commit hook runs guards). Verify `gitreins guard` passes.
5. Hilo post-commit hook updates the hilo index automatically.
6. Push to `origin` (carterlasalle fork) after commits.

## Quality gates

- `.gitreins/` guards: secrets (gitleaks — fixed config), lint, tests (`true` no-op — docs repo, no test suite).
- LLM judge (Tier 2) arms on `gitreins task complete` (env: GITREINS_LLM_BASE_URL/API_KEY/MODEL).
- Verification for THIS fork = a real end-to-end dream-cycle test against live DuckBrain at :3000 (see T06), not just static files.

## Pitfalls

- This is a **markdown skills repo** — no Python tests. Don't invent a test suite; the `true` test_command is intentional.
- DuckBrain MCP tools are NOT `memory()`/`fact_store` — route to `remember`/`recall`/`forget`/`list_keys`/`squash`. Never call `fact_store` for the duckbrain backend.
- Keep the built-in MEMORY.md and Holographic routing intact — this fork ADDS DuckBrain; it must not break the other two backends.
- Don't rewrite git history; `upstream` must stay clean-syncable.
- gitignore local state: `.gitreins/tasks.yaml`, `.gitreins/history/`, `.gitleaks.toml`, `.hilo/`.

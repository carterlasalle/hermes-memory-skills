---
name: agent-dreaming-duckbrain
description: "DuckBrain-native background memory consolidation — three-phase dreaming (Light/Deep/Condensation/REM) that routes ALL promotions to the DuckBrain MCP tools (remember/recall/forget/list_keys/squash) in the hermes-memory namespace. Requires memory.provider: duckbrain and a live server at :3000."
tags: [memory, consolidation, dreaming, introspection, maintenance, duckbrain, duckdb]
triggers: ["dream", "consolidate memory", "run dreaming", "memory consolidation", "duckbrain dreaming"]
---

# Agent Dreaming (DuckBrain)

Three-phase background memory consolidation that routes Phase 2 promotions and
Phase 2.5 condensation **exclusively to DuckBrain** — the git-backed, DuckDB/VSS
persistent memory server at `http://127.0.0.1:3000`. This skill is the
DuckBrain-native counterpart of `agent-dreaming-agnostic`: it does NOT
auto-detect between backends — it IS the duckbrain backend. Use it when
`memory.provider: duckbrain` is configured and the server is live (see
[Phase 0](#phase-0-backend-detection--gate) for the exact gate).

**When to run:** Scheduled cron (recommended: every 6–8 hours) or manually after
a burst of activity. **Cron caveat:** scheduled cron runs with `skip_memory=True`
and may lack the `mcp__duckbrain__*` tools — see [E8](#error-catalog) and the
cron note at the bottom of this file. A manual `hermes cron run <id>` restores
full access.

**Autonomous vs interactive:** Light and Deep phases run autonomously in both
cron and manual modes. REM phase sends a message to the user with proposed
structural actions and waits for response — it never creates wiki pages or skills
without approval.

> **Never route to `memory()` or `fact_store()` in this skill.** DuckBrain has its
> own tool surface (`remember` / `recall` / `forget` / `list_keys` / `squash` /
> `get_compaction_stats` / namespace tools). The built-in and holographic
> backends keep their own skills; this fork ADDS DuckBrain without breaking them.

---

## Phase 0: Backend Detection (gate — run first)

Before any dreaming work, verify that the duckbrain backend is actually active.
This is the DuckBrain-NATIVE skill — it IS the duckbrain backend and does not
auto-detect between backends (SPEC-DB-001 §4.1b). The operative gate is
**server liveness + tool presence**; the config `memory.provider` value is
informational, NOT a blocker (live configs commonly omit the provider despite
a healthy `:3000` — T06 finding #3).

### Gate check 1 — server liveness

```bash
curl -s -m 5 http://127.0.0.1:3000/health
```

Expected: `{"status":"healthy",...}`. Connection refused / timeout → the server
is down → fall back to built-in `memory()` (E1) and log the reason.

### Gate check 2 — tool presence

The `mcp__duckbrain__*` tools (remember / recall / forget / list_keys / squash /
get_compaction_stats / namespace tools) must be present in the toolset. If they
are absent (e.g. cron sessions where MCP tools may not be loaded), fall back to
built-in `memory()` (E2/E8) and log the reason.

### Informational — read the active memory provider (not a gate)

```bash
grep -A10 "^memory:" $HERMES_HOME/config.yaml | grep "provider:" | awk '{print $2}'
```

Used only to distinguish an explicit `holographic` config (route via
`agent-dreaming-agnostic` instead). `duckbrain`, absent, or null with a live
`:3000` and tools present → duckbrain, proceed end-to-end with this skill.

### Detection result table

> **This is the DuckBrain-NATIVE skill — it IS the duckbrain backend.** Unlike
> `agent-dreaming-agnostic`, it does not auto-detect between backends. So the
> operative gate is **server liveness + tool presence**, NOT the config
> `memory.provider` value. A live `:3000` server with `mcp__duckbrain__*` tools
> in the toolset IS duckbrain, even if `config.yaml` omits `memory.provider`
> (common — see T06 finding #3). Only fall back when the server is down or the
> tools are absent.

| Config `memory.provider` | DuckBrain :3000 healthy? | `mcp__duckbrain__*` present? | Backend | Phase 2 tools |
|---|---|---|---|---|
| `duckbrain` (or absent/null) | yes | yes | **duckbrain** | remember / recall / forget / list_keys / squash |
| `duckbrain` (or absent/null) | yes | no | built-in (fallback, E2/E8) | `memory()` |
| any | no | — | built-in (fallback, E1) | `memory()` |
| `holographic` (explicit, server down) | — | — | holographic (use agnostic instead) | fact_store / fact_feedback |
| other | — | — | built-in (fallback) | `memory()` |

The primary signal is the **last two columns** (server healthy AND tools
present → duckbrain). The config `provider` value is informational; it is not
a blocker for this native skill when the server and tools are live.

### Gate

- Backend == **duckbrain** → proceed with THIS skill end-to-end.
- Backend != duckbrain → **STOP.** Do not run duckbrain phases against a
  different backend. Use `agent-dreaming-agnostic` (auto-detect) or
  `agent-dreaming` (built-in) instead, and log the mismatch in the dream
  artifact backend section.

### Record in the dream artifact

```markdown
## Backend
- Provider: duckbrain
- Server: http://127.0.0.1:3000 (healthy)
- Namespace: hermes-memory (default; override per-project)
- Compaction stats baseline: (from get_compaction_stats — see Phase 1 step 3)
```

---

## Phase 1: Light (Ingest + Stage)

**Goal:** Pull raw material from recent sessions, identify candidates for promotion.

### Steps

1. **Get recent sessions.** Call `session_search()` with no arguments. This
   returns recent session titles, previews, and timestamps. Note the count — if
   zero sessions, skip dreaming entirely and report "nothing to dream about."

2. **Resolve wiki path.** Read `wiki.path` from `$HERMES_HOME/config.yaml`
   (e.g. `grep wiki.path $HERMES_HOME/config.yaml`). Fall back to
   `$HERMES_HOME/wiki` if not configured. Never use `$HOME/wiki` — that's shared
   across profiles and breaks isolation. This path is needed for checking
   existing wiki pages when evaluating pointer candidates.

3. **Read current memory — DuckBrain baseline.** DuckBrain has no char limit, so
   the baseline is the key inventory + compaction health:

   - `mcp__duckbrain__list_keys(prefix="/", maxDepth=3, limit=200, namespace="<ns>")`
     — broad view of the key hierarchy (the "index" of what's stored).
   - `mcp__duckbrain__recall(keyPrefix="/", limit=200, namespace="<ns>")` —
     broad view of stored memories (SPEC-DB-001 §2.2 baseline usage).
   - `mcp__duckbrain__get_compaction_stats()` — record tombstone % / partition
     health. This is the capacity signal (see [Phase 2.5](#phase-25-condensation-memory-hygiene)).

   **Namespace:** default `hermes-memory`; if this dream concerns a specific
   project, use that project's namespace. ALWAYS pass `namespace` explicitly
   (SPEC-DB-001 §3.1 — never rely on the current-namespace default).

4. **Filter sessions.** Skip cron sessions (`session.source == "cron"`) — they're
   fully automated with no user interaction, unless they logged errors.

5. **Deep-dive sessions with signal.** For each remaining session that looks
   substantive, call `session_search(query="<session topic keywords>")` to get a
   full summary. **Important:** FTS5 queries match message *content*, not session
   titles — if `session_search(query="<exact session title>")` returns no results,
   try broader keyword queries using terms likely to appear in the messages
   (e.g. `"hello OR test OR bot"` rather than `"Testing the bot connection"`).
   Also try `session_search(query="...")` without a `session_id` constraint —
   discovery mode searches across all sessions and is more forgiving. Look for:
   - **User corrections** — "actually it's X not Y", "don't do that", "remember this"
   - **New preferences** — tool choices, style, workflow habits
   - **Environment discoveries** — ports, paths, service names, config quirks
   - **Recurring problems** — things that broke, workarounds found
   - **Resolved ambiguities** — cases where you had to ask clarifying questions
     and got definitive answers

6. **Stage candidates.** Write a dream artifact to
   `$HERMES_HOME/dreams/YYYY-MM-DD-HHMM.md` with this structure:

   ```markdown
   # Dream Artifact — YYYY-MM-DD HH:MM

   ## Backend
   - Provider: duckbrain
   - Namespace: [hermes-memory | <project ns>]
   - Key inventory: N keys (list_keys prefix="/")
   - Compaction: tombstone % = [from get_compaction_stats]

   ## Sessions Reviewed
   - [session_id] title (timestamp)
   - ...

   ## Candidates
   ### New Entries (genuinely new info not in current memory)
   - [candidate 1]: description + source session + proposed key (/user/preference/...)
   - ...

   ### Replacements (corrections to existing entries)
   - [existing entry text] → [proposed replacement] + source session + key
   - ...

   ### Removals (stale/broken info)
   - [entry text] — reason (contradicted by session X, expired, etc.) + source id
   ```

   **Do NOT write to memory yet.** This phase only stages candidates.

---

## Phase 2: Deep (Score + Promote)

**Goal:** Evaluate staged candidates against current memory, promote winners
**exclusively via DuckBrain MCP tools.**

### Steps

1. **Load the dream artifact** created in Phase 1. If it has no candidates, skip
   to Phase 3.

2. **Score each candidate.** For each candidate, assess four dimensions. A
   candidate must pass ALL four to be promoted — any single failure means skip:

   - **Novelty:** Is this genuinely new, or does it overlap with an existing
     entry? Run `mcp__duckbrain__recall(query="<candidate keywords>", limit=10,
     namespace="<ns>")` — semantic similarity check (SPEC-DB-001 §2.2). FAIL if
     a similar entry already exists. **Note:** empty recall for a legitimately
     new fact IS the novelty signal — proceed with promotion. Empty recall is
     only a failure AFTER the write (E7).
   - **Durability:** Will this still be true in 30 days? User preferences and
     environment facts score high. Task progress, TODOs, and session outcomes
     score low. → FAIL if the fact is temporary or likely to change.
   - **Specificity:** Is this precise enough to be actionable? "User prefers X"
     is good. "User might like X" is vague. → FAIL if the entry would require
     guessing to act on.
   - **Reduction:** Does promoting this let you *remove or shorten* an existing
     entry? → Not a hard fail, but candidates with reduction potential get priority.

   **For replacements:** Only assess Novelty (is the new version genuinely
   better?) and Durability (is the replacement still accurate?). Specificity
   and Reduction don't apply.

3. **Execute promotions — DUCKBRAIN-ROUTED:**

   #### New entries — `remember`

   ```python
   mcp__duckbrain__remember(
     key="/user/preference/<topic>",     # hierarchical, kebab-case (SPEC §3.2)
     domain="person",                    # person|event|concept|message|config|raw_note
     attributes={"source_session": "<id>", "dream_phase": "deep", "backend": "duckbrain"},
     embedding_text="<full candidate text>",   # REQUIRED — the VSS retrieval axis
     namespace="<ns>"                    # ALWAYS explicit
   )
   ```

   Domain mapping (SPEC-DB-001 §2.1):
   - user preference → `person`
   - environment/service fact → `config`
   - project/workspace fact → `concept`
   - one-time occurrence → `event`
   - session exchange (traceability) → `message`
   - unstructured snippet → `raw_note`

   Key strategy (SPEC-DB-001 §3.2):
   - `/user/preference/<topic>` — user preferences, habits
   - `/environment/<service>/<fact>` — ports, paths, service facts
   - `/project/<name>/<decision-N>` — project decisions, numbered
   - `/project/<name>/<component>/...` — project component facts
   - `/tool/<name>/<fact>` — tool quirks, workarounds
   - `/session/<id>/...` — (rare) session traceability
   - Do NOT put free text in keys; free text goes in `embedding_text`.

   #### Replacements — `remember` (same key, new text) THEN `forget` old id

   Order matters (SPEC-DB-001 §4.2): write the new version at the SAME key K
   (it gets a new id), then tombstone the old id:

   ```python
   mcp__duckbrain__remember(
     key="<same key K>", domain="<same domain>",
     attributes={"source_session": "<id>", "dream_phase": "deep", "backend": "duckbrain"},
     embedding_text="<new text>", namespace="<ns>")
   # id of old version came from a prior recall/list_keys result — never fabricated:
   mcp__duckbrain__forget(id="<old id>", reason="superseded by <key K>", namespace="<ns>")
   ```

   #### Removals — `forget` (tombstone)

   ```python
   mcp__duckbrain__forget(id="<id>", reason="<why>", namespace="<ns>")
   ```

   The `id` MUST come from a prior `recall`/`list_keys` result (SPEC-DB-001 §2.4).
   `forget` is a TOMBSTONE — the row is marked deleted, not physically removed.

4. **Log promotions to dream diary.** Append to `$HERMES_HOME/dreams/diary.md`:

   ```markdown
   ### YYYY-MM-DD HH:MM — Deep Phase (backend: duckbrain, ns: <ns>)
   - PROMOTED: [entry text] → key [/user/preference/x] id [<id>] (from: [session summary])
   - REPLACED: [old] → [new] (key: K; old id [id] tombstoned; reason: [correction source])
   - SKIPPED: [candidate] (reason: [novelty/durability/specificity failure])
   - REMOVED: [entry] id [<id>] (reason: [stale/contradicted])
   ```

   Record the ACTUAL key ids returned by remember — never invent or approximate
   them (anti-fabrication, SPEC-DB-001 §9.2).

5. **Post-promotion check — DUCKBRAIN-SPECIFIC (E7, mandatory):**

   1. `mcp__duckbrain__recall(key="<key written>", namespace="<ns>")` — confirm
      the entry is retrievable. If empty: re-write once; if still empty, log the
      failure in the diary and DO NOT claim success.
   2. `mcp__duckbrain__get_compaction_stats()` — record tombstone % in the dream
      diary (SPEC-DB-001 §4.3).

---

## Phase 2.5: Condensation (Memory Hygiene)

**Goal:** Trim bloat and decay stale entries without losing signal. DuckBrain
has NO char limit — capacity is **compaction health** (tombstone %, Parquet
ratio, partition health) from `get_compaction_stats`.

### When to run (trigger table)

| Condition | Action |
|---|---|
| Every dreaming cycle | Light pass: review tombstone %, flag candidates |
| Tombstone % high (per health field / flagged partition) | Deep condensation: forget stale ids, then `squash(dryRun=true)` → `squash()` |
| User explicitly requests | Full condensation incl. `aggressive=true` ONLY with explicit approval |

### Steps (DuckBrain Backend)

1. **Inventory:** `mcp__duckbrain__list_keys(prefix="/", maxDepth=3, limit=200,
   namespace="<ns>")` — discover condensation candidate subtrees.
2. **Candidate detail:** `mcp__duckbrain__recall(keyPrefix="<subtree>", limit=100,
   namespace="<ns>")` — pull the entries under each candidate subtree.
3. **Contradiction resolution:** `mcp__duckbrain__recall(query="<claim>", limit=10,
   namespace="<ns>")` + `session_search` to pick the winner. Never tombstone
   without evidence.
4. **Tombstone stale/contradicted:** `mcp__duckbrain__forget(id=<id>,
   reason="<evidence>", namespace="<ns>")` — one call per stale id.
5. **Preview compaction (MANDATORY before real squash — E6):**
   `mcp__duckbrain__squash(dryRun=true, partition="<domain>/<yyyy-mm>/")` —
   inspect what would change. **Always pass `partition` explicitly** — the
   server resolves a missing partition against a namespace literally named
   `default` and fails with "Default namespace not found" on deployments whose
   default is `hermes-memory` (proven T06, 2026-08-05). Real partition names
   follow `namespaces/<ns>/<domain>/<yyyy-mm>/current.jsonl` (e.g.
   `concept/2026-08/` for domain=concept data written 2026-08).
6. **Execute compaction:** `mcp__duckbrain__squash(partition="<domain>/<yyyy-mm>/")`
   (dryRun=false) — ONLY when tombstone % is high / the health flag is set.
   Never `aggressive=true` without explicit user approval.
7. **Verify:** `mcp__duckbrain__get_compaction_stats()` — confirm tombstone %
   dropped. Record before/after values in the diary (anti-fabrication,
   SPEC-DB-001 §9.3).

### Log to dream diary

```markdown
### YYYY-MM-DD HH:MM — Condensation (backend: duckbrain, ns: <ns>)
- TOMBSTONED: id [<id>] [content snippet] (reason: [evidence])
- SQUASHED: preview [N partitions/changes] → executed, tombstone % [before]→[after]
- HEALTHY: N keys, tombstone % [value]
```

### Rules

- **Preview before squash, always.** `squash(dryRun=true)` first, inspect, then
  real squash (E6). Never run `aggressive=true` without explicit user approval.
- **Never tombstone without evidence.** Verify staleness/contradiction against
  recent sessions before `forget`.
- **Condensation must not increase bloat.** Every action should reduce tombstone
  % or improve signal quality.
- **Wiki pointers are preferred over inline detail.** If a wiki page exists or
  should exist, point to it rather than embedding the detail in memory.

---

## Phase 3: REM (Pattern Extract)

**Goal:** Look across recent dream artifacts for recurring themes that warrant
structural action. Identical to the other backends (SPEC-DB-001 §4.5: Phase 3
REM ✅).

### Steps

1. **Read recent dream artifacts.** List files in `$HERMES_HOME/dreams/`
   (excluding `diary.md`). Read the last 3–5 artifacts. Look for:
   - **Repeated corrections** — Same thing corrected 2+ times → candidate for a
     wiki entry or skill update
   - **Repeated patterns** — Same problem solved 2+ times → candidate for a new
     skill
   - **Memory gaps** — Topics that came up frequently but have no memory entry →
     candidate for promotion in next cycle
   - **Skill staleness** — Skills referenced in sessions but with outdated info →
     candidate for `skill_manage(action='patch')`

2. **For patterns that warrant action, report — don't act.** REM patterns
   involve structural changes (creating wiki pages, skills, patching skills) that
   benefit from human review. Instead of executing directly:

   - **Log to dream artifact and diary** as usual (traceability)
   - **Send a message** to the user's home channel summarizing the patterns and
     proposed actions. Format:

   ```
   🌙 Dreaming REM (backend: duckbrain, ns: <ns>) — patterns found:

   • [PATTERN]: [description]
     → Proposed: [create wiki page / create skill / patch skill / queue for next cycle]

   • [PATTERN]: ...
     → Proposed: ...

   Reply with which to act on, or "skip all" to defer.
   ```

   - **Wait for user response** before executing any structural changes
   - If running as a cron job with no interactive session, send the message and
     stop — the user can trigger the actions manually or reply in chat

3. **Write REM summary** to the dream artifact (append):

   ```markdown
   ## REM Phase — Patterns
   - PATTERN: [description] → ACTION: [create wiki page / create skill / patch skill / queue for next cycle]
   - ...
   ```

4. **Update dream diary** with REM findings.

---

## Rules

- **Never fabricate session content.** If `session_search` doesn't return useful
  detail for a session, skip it. Do not infer or hallucinate what happened.
- **Never promote speculative entries.** Every promoted memory must trace to a
  specific session interaction. "I think the user might prefer X" is not
  promotion-worthy.
- **Namespace discipline (SPEC-DB-001 §3.1):**
  - General user memory → `hermes-memory` (default).
  - A dream about a specific project → that project's namespace.
  - ALWAYS pass `namespace` explicitly on every remember/recall/forget call —
    never rely on the current-namespace default (a `remember` without explicit
    namespace lands wherever the session last switched to).
  - Never write cross-project fleet telemetry into `hermes-memory`.
  - Never split one fact across two namespaces.
- **embedding_text is REQUIRED and non-empty.** Without it the fact cannot be
  found by semantic `recall(query=...)` — the DuckBrain equivalent of
  "entities are mandatory" for holographic (SPEC-DB-001 §2.1, E3).
- **Always set `domain`** (person|event|concept|message|config|raw_note).
  Inconsistent domain usage silently fragments retrieval (SPEC-DB-001 §3.3).
- **Keys are hierarchical, slash-delimited, kebab-case.** Keys ARE a retrieval
  axis alongside VSS query. A replacement of fact X at key K: write new version
  at same key K (new id), then forget the old id (SPEC-DB-001 §3.2).
- **Forget is a tombstone.** The id MUST come from a prior recall/list_keys
  result — never fabricated (E5).
- **Squash requires a dryRun preview first.** Real squash only when tombstone %
  is high; aggressive only with explicit approval (E6).
- **Pointer = entry.** If an entry has a wiki/skill pointer (`see wiki/...` or
  `see skill '...'`), the pointer IS the entry. Inline detail that duplicates
  what the pointer targets should be removed.
- **Keep dreams compact.** Dream artifacts should be under 200 lines. If a
  session generated 20+ candidates, pick the top 5 by durability score.
- **Dream diary is append-only.** Never overwrite previous diary entries. New
  entries go at the bottom (chronological order).
- **Session IDs in diary.** Always note which session(s) informed each promotion
  for traceability.

---

## Verification

After a dreaming run:
1. Phase 0 gate passed: provider == `duckbrain` AND :3000 healthy — logged
2. Phase 2 promotions used ONLY `mcp__duckbrain__*` tools (remember/recall/
   forget/list_keys) — never `memory()` or `fact_store()`
3. Every call passed `namespace` explicitly; default was `hermes-memory`
4. Post-promotion check passed: `recall(key=...)` confirmed each write, and
   `get_compaction_stats()` was recorded (E7)
5. Dream artifact exists with all phases documented
6. Dream diary has a new entry with real key ids
7. No entries were promoted without a source session reference
8. Phase 2.5 (if run): squash was previewed (dryRun=true) before execution, and
   tombstone % before/after was recorded

---

## Compatibility Matrix — DuckBrain row (SPEC-DB-001 §4.5)

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

---

## Error Catalog (SPEC-DB-001 §5)

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

---

## Pitfalls

- **Detection failure is silent.** If the Phase 0 gate doesn't pass, this skill
  falls back to built-in behavior. That's intentional — better to write to
  MEMORY.md than to nothing. But the user won't know unless you report the
  backend in the dream artifact and REM message.
- **Namespace drift.** The current namespace can be changed by another tool
  mid-cycle. Every call passes `namespace` explicitly, so drift cannot corrupt
  writes (SPEC-DB-001 §6).
- **Duplicate keys are legal.** Keys are not unique — ids are. Replacement =
  write new id at same key + forget old id. Verify the old id is tombstoned
  (list_keys/recall no longer returns it live).
- **Empty embedding_text is a silent orphan.** A `remember` without
  `embedding_text` succeeds but can never be found by semantic recall — it's
  invisible to VSS queries. Never omit it.
- **`squash(aggressive=true)` rewrites git history.** DuckBrain is git-backed.
  Aggressive squash is destructive to history — reserve for explicit user
  approval or extreme tombstone ratios.
- **Cron jobs cannot access duckbrain MCP tools.** Hermes core hardcodes
  `skip_memory=True` for all cron jobs (`cron/scheduler.py:1686`). Phase 0
  detection correctly identifies the backend, but the `mcp__duckbrain__*` tools
  are unavailable in scheduled cron. Running the job manually
  (`hermes cron run <id>`) restores full duckbrain access.
- **Dream diary patching requires extra context.** Diary entries accumulate
  repetitive closing phrases that appear in every cycle. When using `patch` to
  append, include the full pattern name and preceding lines in `old_string` to
  guarantee a unique match — a single-sentence `old_string` can hit multiple
  matches and fail.
- **Never call `fact_store` for the duckbrain backend.** This fork ADDS
  DuckBrain; the built-in MEMORY.md and Holographic routing must keep working
  unchanged. If you find yourself reaching for `memory()`/`fact_store()` while
  duckbrain is the detected backend, you're in the wrong skill.

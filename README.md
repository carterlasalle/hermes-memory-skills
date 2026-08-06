# Hermes Agent Memory Skills (Memory-Agnostic)

Memory hygiene skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent) that work with **any memory backend** — built-in MEMORY.md, Holographic, and DuckBrain.

These are the memory-agnostic versions of the original [hermes-memory-skills](https://github.com/nexus9888/hermes-memory-skills). They auto-detect which memory provider is active and route operations to the correct toolset. This fork additionally ships `agent-dreaming-duckbrain` — a DuckBrain-native dreaming skill for when the DuckBrain backend is configured.

## Why Memory-Agnostic?

The original skills assume the built-in `memory()` tool — they read/write `MEMORY.md` directly. That works great for the default backend, but Hermes Agent supports 8 external memory providers. Holographic, the most interesting one, stores facts in SQLite with HRR vector algebra and trust scoring. It has completely different tools (`fact_store`, `fact_feedback`), no character limit, and no `§` delimiter format.

Rather than making users choose between backends OR skills, these skills detect the active backend at runtime and adapt:

| Backend | Detection | Phase 2 tool | Capacity model | Bloat handling |
|---------|-----------|-------------|----------------|----------------|
| **Built-in** (default) | `memory.provider` empty/null | `memory()` | 2,200 char limit | Phase 2.5 Condensation (built into dreaming) |
| **Holographic** | `memory.provider: holographic` | `fact_store()` + `fact_feedback()` | Trust scores (0.0–1.0) | `fact_feedback(action='unhelpful')` decay |
| **DuckBrain** | `memory.provider: duckbrain` + `:3000` healthy | `mcp__duckbrain__*` (`remember`/`recall`/`forget`/`list_keys`/`squash`) | Compaction health (tombstone %, `get_compaction_stats`) | `forget` stale ids → `squash(dryRun=true)` preview → `squash()` |
| **Honcho, Mem0, etc.** | Detected but unsupported | Falls back to built-in `memory()` | Built-in limits apply | Built-in condensation |

## Skills

### 🌙 Agent Dreaming (Memory-Agnostic)

Three-phase background memory consolidation — same Light/Deep/REM structure as the original, but with backend detection as Phase 0, routing in Phase 2, and a built-in **Phase 2.5 Condensation** that replaces the deprecated `memory-lean-check` skill.

```
Phase 0:   Detect backend (built-in, holographic, or duckbrain)
Phase 1:   Light — review sessions, stage candidates
Phase 2:   Deep — score candidates, promote via correct backend tools
Phase 2.5: Condensation — trim bloat / decay stale facts (built into dreaming)
Phase 3:   REM — extract patterns, propose structural changes
```

**What's different from the original:**
- Phase 0 backend detection from `config.yaml`
- Phase 2 routes to `memory()`, `fact_store()`/`fact_feedback()`, or the `mcp__duckbrain__*` tools automatically
- Capacity check adapts: char limit for built-in, trust score distribution for holographic
- New entries in holographic always include `entities` for compositional recall
- **Phase 2.5 Condensation** built in — no separate `memory-lean-check` skill needed
- Honcho/Mem0/other backends gracefully fall back to built-in

### 🌙 Agent Dreaming (DuckBrain-Native)

`agent-dreaming-duckbrain` is the DuckBrain-only counterpart: same
Light/Deep/Condensation/REM structure, but **every** promotion routes to the
DuckBrain MCP tools (`remember` / `recall` / `forget` / `list_keys` / `squash`)
in the `hermes-memory` namespace. It does not auto-detect — it IS the duckbrain
backend. Use it when `memory.provider: duckbrain` is configured and the server
at `http://127.0.0.1:3000` is live.

Full details: `agent-dreaming-agnostic/references/duckbrain-backend.md` (tools,
schema, namespaces, key/domain strategy, VSS, compaction) and
`specs/duckbrain-backend.md` (the routing contract).

## Installation

```bash
# Clone the repo
git clone https://github.com/nexus9888/hermes-memory-skills.git
cd hermes-memory-skills

# Copy the agnostic dreaming skill into your Hermes skills directory
cp -r agent-dreaming-agnostic ~/.hermes/skills/management/

# If you use the DuckBrain backend, also copy the DuckBrain-native skill
cp -r agent-dreaming-duckbrain ~/.hermes/skills/management/

# Verify it's installed
hermes skills list | grep agnostic
```

Or install directly from the repo:

```bash
hermes skills tap add https://github.com/nexus9888/hermes-memory-skills
hermes skills install agent-dreaming-agnostic
hermes skills install agent-dreaming-duckbrain   # only if using DuckBrain
```

## Cron Setup

Recommended schedule (works regardless of backend):

```bash
# Dream — handles both promotion AND condensation
hermes cron create \
  --schedule "0 */6 * * *" \
  --name "agent-dreaming" \
  --skill agent-dreaming-agnostic \
  "Run memory consolidation with the active backend"
```

> **No separate lean-check cron needed.** Phase 2.5 Condensation runs as part of dreaming whenever capacity thresholds are breached (built-in >60%) or every cycle (holographic).
>
> ### ⚠️ Cron Limitation: Memory Providers Unavailable
>
> **Detection works, but promotion to holographic is blocked in cron.**
>
> Hermes Agent's cron scheduler hardcodes `skip_memory=True` for **all** cron jobs
> (`cron/scheduler.py:1686`). This disables the entire memory subsystem —
> `memory()`, `fact_store`, `fact_feedback`, and the `mcp__duckbrain__*` tools are
> all unavailable.
>
> | Context | Holographic tools | DuckBrain MCP tools | Built-in `memory()` | File patching |
> |---------|-------------------|---------------------|---------------------|---------------|
> | **Interactive session** | ✅ `fact_store` / `fact_feedback` | ✅ `mcp__duckbrain__*` | ✅ | ✅ |
> | **Manual run** (`hermes cron run`) | ✅ | ✅ | ✅ | ✅ |
> | **Scheduled cron** | ❌ blocked by `skip_memory=True` | ❌ blocked (E8) | ❌ blocked | ✅ fallback |
>
> **What this means in practice:**
> - Phase 0 detection correctly identifies holographic ✓
> - Phase 2 promotions degrade to direct `patch` on MEMORY.md (works, just not holographic)
> - Holographic facts from interactive sessions are **never condensed by cron runs**
> - DuckBrain memories are also never consolidated by cron runs — scheduled cron
>   lacks the `mcp__duckbrain__*` tools (E8), so Phase 2/2.5 degrade to MEMORY.md
>   patching. A manual `hermes cron run <job-id>` restores full DuckBrain access.
> - Built-in MEMORY.md stays healthy via file patching
>
> **Workarounds:**
> - **Increase cron frequency** — daily catches MEMORY.md bloat before it hits 80%+
> - **Run manually** — `hermes cron run <job-id>` uses the interactive toolset (holographic available)
> - **Switch to built-in** — `hermes memory off` eliminates the split-brain entirely
> - **Hybrid** — keep holographic for interactive sessions, accept MEMORY.md patching for cron
>
> This is a Hermes core limitation (tracked in [NousResearch/hermes-agent#34094](https://github.com/NousResearch/hermes-agent/issues/34094),
> [#18885](https://github.com/NousResearch/hermes-agent/issues/18885)), not a skill bug.

## Requirements

- Hermes Agent (any version with `memory` tool)
- `session_search` tool enabled
- `llm-wiki` skill installed (for wiki pointer creation in Phase 2)
- For Holographic backend: `memory.provider: holographic` in config.yaml + plugin installed
- For DuckBrain backend: `memory.provider: duckbrain` in config.yaml + DuckBrain server running at `http://127.0.0.1:3000`

## How It Detects the Backend

The skill reads `memory.provider` from `$HERMES_HOME/config.yaml`:

```yaml
# Built-in (default — no provider set):
memory:
  memory_enabled: true

# Holographic:
memory:
  provider: holographic

# DuckBrain (this fork — git-backed JSONL + DuckDB/VSS server at :3000):
memory:
  provider: duckbrain
```

The detection logic:

```
memory.provider == "" or null → built-in
memory.provider == "holographic" → holographic
memory.provider == "duckbrain" AND :3000 healthy → duckbrain
memory.provider == "duckbrain" AND :3000 down → built-in (fallback, E1)
memory.provider == "honcho" or "mem0" or ... → built-in (fallback)
```

Fallback is intentional — better to write to MEMORY.md than to nothing.

**DuckBrain liveness check (Phase 0 two-step):**

```bash
curl -s -m 5 http://127.0.0.1:3000/health
# → {"status":"healthy","uptime":...,"timestamp":"..."}
```

If the server is unreachable (connection refused / timeout) or the
`mcp__duckbrain__*` tools are absent from the toolset (cron sessions), the skill
falls back to built-in `memory()` and logs the reason in the dream artifact
(E1/E2/E8).

**DuckBrain specifics** (full reference:
`agent-dreaming-agnostic/references/duckbrain-backend.md`):
- No char limit — capacity = compaction health (`get_compaction_stats`, tombstone %).
- Every `remember`/`recall`/`forget` call passes `namespace` explicitly
  (default `hermes-memory`; project dreams use that project's namespace).
- `embedding_text` is REQUIRED on every `remember` — it is the VSS retrieval axis.
- `forget` is a tombstone; condensation = tombstone stale ids → `squash(dryRun=true)`
  preview → real `squash()` (never `aggressive=true` without explicit approval).

## Architecture

```
┌─────────────────────────────────────────────────────┐
│           agent-dreaming-agnostic                    │
│                                                      │
│  Phase 0: Backend Detection                          │
│    └─ read memory.provider from config.yaml          │
│                                                      │
│  Phase 1: Light (same regardless of backend)         │
│    ├─ session_search() → recent sessions             │
│    ├─ deep-dive signal sessions                      │
│    └─ stage candidates → dream artifact              │
│                                                      │
│  Phase 2: Deep (backend-routed)                      │
│    ├─ score candidates (4 dimensions)                │
│    ├─ built-in → memory() tool                       │
│    ├─ holographic → fact_store/fact_feedback         │
│    ├─ duckbrain → mcp__duckbrain__remember/forget    │
│    └─ post-promotion capacity check                  │
│                                                      │
│  Phase 2.5: Condensation (built in)                  │
│    ├─ built-in: merge/shorten/remove bloat           │
│    └─ holographic: trust decay + contradiction check │
│    ├─ duckbrain: forget stale ids → squash (dryRun)  │
│                                                      │
│  Phase 3: REM (pattern extraction)                   │
│    ├─ cross-dream pattern detection                  │
│    ├─ propose structural changes                     │
│    └─ user approval gate                             │
└─────────────────────────────────────────────────────┘
```

## License

MIT — same as the original hermes-memory-skills.

## Contributing

This is a shared workspace. If you add support for Honcho, Mem0, or another backend:

1. Update Phase 0 detection with the new provider
2. Add Phase 2 routing for the new backend's tools
3. Update the compatibility matrix in SKILL.md
4. Test with that backend active

PRs welcome.

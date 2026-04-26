# Memory System

How gosh.memory works: storage, retrieval, inference, and the decisions
an agent or operator needs to understand.

For installation and bootstrap, see [SETUP.md](SETUP.md).
For identity, tokens, and visibility rules, see
[PERMISSIONS-AND-ACL.md](PERMISSIONS-AND-ACL.md).

---

## MCP surface (full tool list)

gosh.memory exposes the following MCP tools. Names that reflect newer
capabilities (auth, MAL, write-log, secrets, versioning, prompts,
schema) are grouped at the bottom.

### Storage

| Tool | Purpose |
|------|---------|
| `memory_store` | Sync ingest. Extracts atomic facts from raw text/conversation |
| `memory_write` | Async write-log append. Returns `message_id`; extraction runs in the background |
| `memory_write_status` | Check the extraction state of a `memory_write` message |
| `memory_ingest` | Unified ingest; dispatches by `kind` (text / path / url) |
| `memory_ingest_document` | Document-flavored extraction (handles structural markers) |
| `memory_ingest_asserted_facts` | Ingest pre-extracted facts at any tier (granular, consolidated, cross-session) |
| `memory_import` | Import a corpus (`conversation_json`, `text`, `directory`, or `git`) |
| `memory_import_history` | Backward-compat alias for `memory_import` |

### Retrieval and inference

| Tool | Purpose |
|------|---------|
| `memory_recall` | Retrieve context, no LLM call |
| `memory_plan_inference` | Retrieve and return the executable inference plan (top-level `payload`, `payload_meta`, `secret_ref` — the latter is a sibling of `payload`, not nested). Used by agents that do their own LLM calls |
| `memory_ask` | Recall + LLM inference + answer |
| `memory_query` | Structured filter query (no vectors, no LLM) |
| `memory_list` | List facts with kind/scope/ACL filtering |
| `memory_get` | Get one fact by id |

### Tier control and indexing

| Tool | Purpose |
|------|---------|
| `memory_build_index` | Build/rebuild vector and BM25 indices. Conditionally rebuilds consolidated and cross-session tiers if dirty |
| `memory_flush` | Run consolidation and cross-session synthesis explicitly; returns counts |
| `memory_reextract` | Re-run the librarian on existing raw sessions |
| `memory_stats` | Counts, tier health, index status, process cost summary |
| `memory_migrate_jsonnpz` | Migrate legacy JSON/NPZ stores (operator tool) |

### Versioning and lifecycle

| Tool | Purpose |
|------|---------|
| `memory_edit` | Replace a fact with a new version; the prior version is marked superseded |
| `memory_retract` | Mark all versions of a fact invisible (soft delete) |
| `memory_purge` | Permanent delete |
| `memory_redact` | Mask specific fields (`fact`, `entities`, `content`) — content remains but is rendered as `[REDACTED]` |
| `memory_get_versions` | Return the version chain for an artifact |

### Configuration, schema, prompts

| Tool | Purpose |
|------|---------|
| `memory_set_config` / `memory_get_config` | Canonical runtime config (embedding model, profiles, profile configs, retrieval, metadata schema) |
| `memory_set_profiles` / `memory_get_profiles` | Inference profiles (legacy direct surface; `memory_set_config` covers the same surface) |
| `memory_set_schema` / `memory_get_schema` | Declared metadata schema (field types, required flags) |
| `memory_set_prompt` / `memory_get_prompt` / `memory_list_prompts` | Override extraction prompts per content type; `source` field reports `builtin` or `custom` |

### Authority / principals / membership

| Tool | Purpose |
|------|---------|
| `auth_bootstrap_admin` | One-time bootstrap of the first admin from `GOSH_MEMORY_ADMIN_TOKEN` |
| `auth_token_issue` / `auth_token_revoke` / `auth_token_list` | Token lifecycle |
| `principal_create` / `principal_get` / `principal_disable` | Principal lifecycle |
| `swarm_create` / `swarm_get` / `swarm_list` | Swarm CRUD |
| `membership_grant` / `membership_revoke` / `membership_list` | Persistent membership |
| `membership_register` / `membership_unregister` | Self-registration for the lifetime of the calling token |

### Secrets

| Tool | Purpose |
|------|---------|
| `memory_store_secret` | Store a provider key (write-only; X25519-sealed envelope delivered to agents at runtime) |
| `memory_list_secrets` | List metadata (no values) |
| `memory_delete_secret` | Delete a secret |

### Streaming / courier

| Tool | Purpose |
|------|---------|
| `courier_subscribe` | Subscribe to facts matching a filter; delivered over `/mcp/sse` |
| `courier_unsubscribe` | Cancel a subscription |

### Memory Adaptation Loop

| Tool | Purpose |
|------|---------|
| `memory_mal_configure` | Toggle MAL on/off; set tuning gates |
| `memory_mal_feedback` | Submit a query-failure signal |
| `memory_mal_trigger` | Run the optimization loop |
| `memory_mal_status` | Current MAL state and active artifacts |
| `memory_mal_list_feedback` | List feedback events with status (queued, applied, rejected) |
| `memory_mal_get_artifact` | Read a tuning artifact snapshot |
| `memory_mal_rollback` | Revert to a previous artifact |

### Operator tools

| Tool | Purpose |
|------|---------|
| `memory_admin_backfill_original_raw_sources` | Bulk source backfill for legacy data |

---

## Two ways to query memory

### `memory_recall` — retrieval without inference

```
query → detect_query_type → retrieve → route → build context → complexity hint
```

Returns context, retrieved fact ids, complexity hint, recommended
profile, retrieval-family info. No LLM call.

### `memory_plan_inference` — retrieve and plan, do not execute

Same retrieval, but the response also carries the **executable plan**
the caller needs to run inference: model id, `secret_ref`,
`payload_meta` (with provider, pricing, optional `local_cli` config).

This is what gosh.agent uses. It calls `recall` for context, then
`plan_inference` for the model and secret, then makes the API call
itself.

### `memory_ask` — retrieval + inference

`recall` → choose profile → build prompt → call provider → tool use up
to 3 rounds → answer. Use when you want a direct answer without
running your own LLM loop.

### Comparison

| | `recall` | `plan_inference` | `ask` |
|-|----------|------------------|-------|
| LLM call inside memory | No | No | Yes |
| Returns context | Yes | Yes | Yes (consumed internally) |
| Returns model + secret | No | Yes | Used internally |
| Cost | $0 | $0 | $0.02–0.20 |
| Caller picks model | Yes | Yes (via recommended) | No |

---

## Semantic extraction (Librarian)

The core differentiator. When content is stored, an LLM extracts
structured atomic facts — not chunks, not embeddings of raw text. Each
fact is a self-contained statement with entities, temporal links,
dependencies, and semantic metadata.

This extraction is **format-aware**: different content types get
specialized prompts that understand the structure of the input.

| Format | Detection | Strategy |
|--------|-----------|----------|
| Conversation | Speaker turns, Q&A patterns | Block segmentation → per-block extraction |
| Document | No speaker turns, structural markers | Document block segmentation |
| Code trace | Code blocks, stack traces | Single-pass domain-specific prompt |
| Agent trace | Tool call sequences, JSON DOM | Specialized prompt |
| Narrative | Dense prose | Single-pass extraction |
| Fact list | Pre-formatted facts | Light parsing |

Each extracted fact carries:

- `fact` — atomic statement text
- `kind` — `fact`, `preference`, `constraint`, `rule`, `decision`,
  `action_item`, `event`, `requirement`, `rejection`, `relationship`,
  `identity`, `location`, `possession`, `acquisition_event`, `plan`,
  `count_item`, `codebase_relation`
- `entities` — named entities mentioned
- `tags` — semantic tags
- `event_date` — when it happened (if temporal)
- `depends_on` — references to other facts
- `speaker` / `speaker_role`
- ACL: `owner_id`, `read`, `write`, `scope`
- `metadata` — declared schema fields plus extraction context
- `target` — delivery intent (e.g., `["agent:reviewer"]`)

This is not chunking. A single conversation turn might produce 5 atomic
facts. A 10-page document might produce 200+ facts with entity
cross-references.

### Custom extraction prompts

`memory_set_prompt` overrides the prompt for a given `content_type`.
`memory_list_prompts` reports `source: builtin | custom` so operators
can see what is overridden. Useful when ingesting domain content
(legal, medical, financial) where the default prompts under-extract.

---

## Fact organization

Extracted facts are organized into three levels, each built on the
previous:

### Granular facts

Atomic facts produced by extraction. Every query ultimately retrieves
from granular facts.

### Consolidated facts

Per-session summaries. After extraction, the system can summarize a
session's granular facts into higher-level statements that capture the
session's key points. Reduces noise during retrieval for broad queries.

### Cross-session facts

Per-entity synthesis across all sessions. The system identifies
entities that appear in multiple sessions and produces synthesized
facts that capture the entity's full history. Critical for queries
like "What do we know about X?" that span many sessions.

### How they are built

- **`memory_build_index`** conditionally rebuilds consolidated and
  cross-session facts if dirty (new granular facts added since the
  last build), then embeds all facts for retrieval.
- **`memory_flush`** runs consolidation and cross-session synthesis
  explicitly, returning real counts.
- **`memory_stats`** reports counts for all three levels: `granular`,
  `consolidated`, `cross_session`.

All three levels are embedded and searched. Retrieval uses RRF fusion
to combine signals across them.

---

## Fact lifecycle and versioning

Beyond store/retrieve, facts have a lifecycle:

| Operation | Effect |
|-----------|--------|
| `memory_edit(id, ...)` | Create a new version; old version marked `superseded`, new one becomes `active` |
| `memory_retract(id)` | Mark all versions invisible (soft delete) |
| `memory_purge(id)` | Permanent delete |
| `memory_redact(id, fields=[...])` | Mask specified fields. `fact`, `entities`, `content` are common targets — content remains but renders as `[REDACTED]` |
| `memory_get_versions(id)` | Return the full version chain |

Status values on a fact: `active`, `superseded`, `retracted`. Retrieval
hides non-active facts by default.

---

## Async write-log

`memory_write` exists for streaming and real-time ingest scenarios where
the caller cannot block on extraction. The flow:

```
memory_write(text) ─▶ message_id, status=queued
                       │
                       ▼ (async worker)
                     extract → write granular/consolidated/cross-session
                       │
                       ▼
memory_write_status(message_id) ─▶ {status, facts_extracted, attempts, error}
```

Used by `gosh-agent capture` (the hook persona) so prompt/response
captures from coding CLIs do not stall the user-facing CLI.

---

## Query type detection

Before retrieval, the query is classified. Each type triggers different
retrieval strategies and inference prompts.

| Type | Examples | Strategy |
|------|----------|----------|
| `lookup` | "What is X?" | Direct fact match |
| `temporal` | "When did X?" / "How long ago?" | Time-aware retrieval + `date_diff` tool |
| `counting` | "How many?" | Exhaustive scan + `count_items` tool |
| `current` | "Where does X live now?" | Prefer latest, supersession-aware |
| `synthesis` | "What does the user prefer?" | Multi-hop retrieval |
| `summarize` | "Write a summary" | Session round-robin |
| `icl` | In-context learning examples (`label:N` patterns) | Example-based retrieval |
| `rule` | "What's the policy on X?" | Prefer rule/constraint kinds |
| `causal` | "Why did X happen?" | Dependency chain traversal |
| `prospective` | "What's the next step?" | Pending/upcoming facts |
| `default` | Everything else | Standard hybrid retrieval |

Detection is regex-based (fast, no LLM call). The detected type
determines which retrieval pipeline fires, which tools are available
during inference, and which inference prompt template is used.

---

## Retrieval pipeline

### Step 1: Episode retrieval (when episode corpus exists)

For documents and structured conversations:
- BM25 + vector search across episodes
- Family routing: `conversation`, `document`, `codebase`, or `auto`
- RRF fusion to combine signals

### Step 2: Structural augmentation

If episode results are insufficient:
- Entity matching within conversations
- Section-based matching for documents

### Step 3: Semantic fact sweep (fallback)

Full hybrid fact search:
- BM25 keyword search on fact text
- Vector similarity on embeddings
- Named entity matching between query and facts
- RRF fusion across signals

### Output

A ranked list of facts filtered by ACL, formatted into a context
string with structured metadata.

### Configurable knobs

In the canonical config under `retrieval`:

```jsonc
{
  "retrieval": {
    "default_token_budget": 4000,
    "search_family": "auto",
    "rrf_k": 60,
    "bm25_k1": 1.5,
    "bm25_b": 0.75
  }
}
```

---

## Complexity hint and model selection

After retrieval, memory computes a complexity hint — a signal telling
the caller how hard this query is.

### Three axes

**Retrieval complexity** — structural signals from retrieval:
- Multi-hop (temporal, counting, current queries): +0.35
- Cross-scope (facts span multiple agents): +0.25
- Conflict found (supersession): +0.20
- High fact count (>50): +0.05

**Content complexity** — what the facts contain:
- Fact text length and entity density
- Structural complexity
- Temporal relationship density

**Query complexity** — how complex the question itself is:
- Multi-constraint decisions ("which option", "recommended")
- Strong constraint terms (latency, cost, budget, compliance)
- Decision-making markers

### Combined score

```
score = max(retrieval_complexity, content_complexity, query_complexity)
```

Score 0.0–1.0, mapped to levels 1–5.

`dominant` reports which axis drove the score: `retrieval`, `content`,
`query`, or `tie`.

### Level → profile

When profiles are configured in memory config:

| Level | Score range |
|-------|-------------|
| 1 | 0.0–0.2 |
| 2 | 0.2–0.4 |
| 3 | 0.4–0.6 |
| 4 | 0.6–0.8 |
| 5 | 0.8–1.0 |

The `profiles` config maps each level to a profile name. Profile names
are arbitrary strings. There are no hardcoded production defaults — the
operator picks the names and the models behind them.

`recommended_profile` in the recall/plan_inference output tells the
agent which profile memory recommends for this query.

---

## Prompt routing

After query type detection and retrieval, memory selects a prompt
strategy:

| Condition | Prompt type | Tools? |
|-----------|------------|--------|
| Query type = `summarize` | `summarize_with_metadata` | Yes |
| Query type = `icl` | `icl` | No |
| Low session coverage (<30%) with >20 sessions and retrieved facts | `tool` | Yes |
| Raw context present in hybrid packet | `hybrid` | No |
| Default | Same as query type | No |

This determines which inference prompt template is used and whether
tools are available.

---

## Payload building

When profiles are configured, `recall` and `plan_inference` build an
LLM-ready payload:

```jsonc
{
  "model": "anthropic/claude-sonnet-4-6",
  "messages": [{"role": "user", "content": "...context + query..."}],
  "max_tokens": 2048,
  "temperature": 0.3,
  "tools": [...]
}
```

Provider-specific (Anthropic, OpenAI, Google, Groq, Inception). The
prefix on `model` selects the provider.

### Truncation

Context is assembled by tier:
- Tier 1: active decisions, constraints, rejections
- Tier 2: action items, requirements, preferences
- Tier 3: all others
- Tier 4: codebase facts

If exceeded, lower tiers are dropped first. If everything is exhausted:
`payload_meta.budget_exceeded = true`.

### `payload_meta`

```jsonc
{
  "profile_used": "balanced",
  "profile_fallback": false,
  "context_tokens": 1245,
  "message_tokens_est": 456,
  "tool_tokens_est": 120,
  "memory_budget": 2000,
  "budget_exceeded": false,
  "prompt_type": "temporal",
  "prompt_key": "temporal",
  "use_tool": true,
  "provider": "anthropic",
  "provider_family": "anthropic",
  "backend": "api",
  "pricing": {"input_per_1k": 0.003, "output_per_1k": 0.015},
  "truncation": null
}
```

The full plan response also includes `secret_ref` at the **top level**,
alongside `payload` and `payload_meta`:

```jsonc
{
  "payload":      { ... },
  "payload_meta": { ..., "pricing": {...} },
  "secret_ref":   {"name": "anthropic", "scope": "system-wide"}
}
```

`pricing` lives inside `payload_meta`; `secret_ref` is a sibling of
`payload_meta`. The agent uses `secret_ref` to authenticate and
`pricing` to compute cost. Pre-v0.3.0 servers placed `secret_ref`
inside `payload_meta` — modern agents check the top-level location
first and fall back to the legacy nested form.

---

## Tool use in inference

For temporal, counting, and low-coverage queries, the LLM can call
tools:

| Tool | Purpose | When |
|------|---------|------|
| `date_diff` | Exact date arithmetic | Temporal queries |
| `count_items` / `sum_field` / `distinct_values` | Exact aggregation | Counting queries |
| `get_more_context` | Fetch full raw session text | Low coverage |

Tool execution is local. The LLM issues `tool_use` blocks, memory
executes them deterministically, and feeds results back for up to 3
rounds.

---

## Store pipeline

When content is stored via `memory_store`, `memory_write`, or
`memory_ingest_*`:

1. **Auth** — caller token resolved; scope must be passed explicitly.
2. **ACL gate** — caller must have write ACL for instance and scope.
3. **Dedup** — content hash and simhash compared against existing
   sessions (`NEAR_DUP_SIMHASH_THRESHOLD = 3`, min 200 chars).
4. **Raw session saved** — source of truth, written immediately.
5. **Format detection** — conversation / document / agent_trace /
   code_trace / fact_list / narrative / json_conv / web_dom /
   game_board.
6. **Semantic extraction** — format-specific librarian prompt produces
   atomic facts.
7. **Tagging** — ACL fields derived from caller + scope; metadata
   merged; `target` applied for delivery routing.
8. **Episode registration** — for document families, episode corpus
   updated.

---

## ACL and visibility (summary)

Each fact has:

```jsonc
{
  "owner_id": "agent:alice",
  "scope": "swarm-shared",
  "read":  ["swarm:alpha"],
  "write": ["agent:alice"]
}
```

Visibility:

- Owner always has access.
- `agent:PUBLIC` in `read` → everyone.
- Swarm membership checked at read time. Memberships can be passed
  per-call (`swarm_id` parameter) or persisted (`membership_grant`).
- Admin role bypasses fact-level ACL filtering.
- Superseded and retracted facts are hidden from default reads.
- Expired-TTL facts are hidden.

| Scope | Visible to |
|-------|-----------|
| `agent-private` | Only the owning agent (and admins) |
| `swarm-shared` | All members of the named swarm |
| `system-wide` | All authenticated principals |

For the full identity / token / membership model, see
[PERMISSIONS-AND-ACL.md](PERMISSIONS-AND-ACL.md).

---

## Embedding and index

`memory_build_index` must be called before the first `recall`.

1. Conditionally rebuilds consolidated and cross-session facts if dirty
2. Embeds all facts using the configured embedding model
3. Builds BM25 keyword indices
4. Builds entity lookup indices
5. Caches embeddings by content fingerprint

Embedding cache is fingerprint-based: if fact ids haven't changed,
embeddings are reused. Re-embedding is expensive and produces slightly
different vectors — never regenerate without reason.

---

## Cost model

| Operation | Estimated cost | Duration |
|-----------|---------------|----------|
| `store()` 1 session | $0.02–0.05 | 2–5 sec |
| `ingest_document()` full PDF | $0.50–2.00 | 30–120 sec |
| `build_index()` cold (500 facts) | $0.10–0.30 | 10–30 sec |
| `recall()` | $0.00 | <1 sec |
| `plan_inference()` | $0.00 | <1 sec |
| `ask()` with inference | $0.02–0.20 | 2–10 sec |

`process_cost_summary` in `memory_stats` reports LLM costs accumulated
during the current process lifetime, broken down by provider with
input/output/cache-read/cache-creation tokens. It is process-scoped, not
per-key — a process restart resets these counters.

---

## Schema and metadata

`memory_set_schema` declares which metadata fields are valid for facts
in a namespace. Validation rules:

- Fields are flat (no nested dicts)
- List fields contain only strings
- Type must be `string` or `number`
- `required: true` enforced at ingest

Example:

```json
{
  "source":   {"type": "string", "required": true},
  "severity": {"type": "number", "required": false},
  "tags":     {"type": "string", "required": false}
}
```

`memory_get_schema` returns the current declaration.

---

## Memory Adaptation Loop (MAL)

MAL is implemented and exposed through 7 MCP tools, but is **disabled
by default**. Enabling it unlocks feedback capture and allows
optimization runs to mutate retrieval/grouping/extraction parameters.

See [MEMORY-ADAPTATION-LOOP.md](MEMORY-ADAPTATION-LOOP.md) for the
design and current implementation status.

---

## Telemetry

For the full field list returned by `recall`, `ask`, `stats`, and
`agent_status`, see [TELEMETRY-CONTRACT.md](TELEMETRY-CONTRACT.md).

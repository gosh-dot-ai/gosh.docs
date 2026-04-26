# Telemetry Contract

End-to-end production telemetry surfaces across the GOSH system.

All telemetry fields documented here are available via `--json` output
on `gosh memory data ...` and `gosh agent task status` commands, and on
the corresponding MCP responses.

## Memory recall telemetry

Returned by `gosh memory data recall --json` / `memory_recall` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `context` | string | Retrieved facts formatted as a context string |
| `retrieved_count` | int | Number of facts retrieved |
| `query_type` | string | Detected query type (`lookup`, `temporal`, `counting`, `current`, `synthesis`, `summarize`, `icl`, `rule`, `causal`, `prospective`, `default`) |
| `token_estimate` | int | Estimated token count of context |
| `complexity_hint.score` | float | 0.0–1.0 complexity score |
| `complexity_hint.level` | int | 1–5 complexity level |
| `complexity_hint.signals` | list | Names of complexity signals that fired |
| `complexity_hint.retrieval_complexity` | float | Retrieval-side complexity |
| `complexity_hint.content_complexity` | float | Content-side complexity |
| `complexity_hint.query_complexity` | float | Query-side complexity |
| `complexity_hint.dominant` | string | `retrieval`, `content`, `query`, or `tie` |
| `sessions_in_context` | int | Sessions included in context |
| `total_sessions` | int | Total sessions available |
| `coverage_pct` | float | Session coverage percentage |
| `recommended_prompt_type` | string | Suggested prompt style for downstream LLM |
| `use_tool` | bool | Whether tools should be enabled for inference |
| `retrieval_families` | list | Retrieval routing families used |
| `search_family` | string | `conversation`, `document`, `codebase`, or `auto` |
| `recommended_profile` | string | Recommended inference profile (when profiles configured) |
| `payload` | object | LLM-ready payload (when profiles configured and not budget-blocked) |
| `payload_meta` | object | Payload metadata (token counts, truncation info, provider, pricing) |
| `secret_ref` | object | Secret reference for the chosen profile. **Top-level sibling of `payload` and `payload_meta`**, not nested. Older callers may also see it under `payload_meta.secret_ref` (legacy fallback). |
| `retrieved_episode_ids` | list | Episode ids considered |
| `actual_injected_episode_ids` | list | Episode ids actually injected into context |
| `selection_scores` | list | Per-fact retrieval scores (BM25, vector, entity, RRF) |
| `runtime_trace` | object | Performance timing trace |
| `process_cost_summary` | object | LLM cost breakdown for the calling process |

`payload_meta` fields used by callers:

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
  "tool_use_downgraded": false,
  "tool_use_downgrade_reason": null,
  "provider": "anthropic",
  "provider_family": "anthropic",
  "backend": "api",
  "pricing": {"input_per_1k": 0.003, "output_per_1k": 0.015},
  "terminal_render_candidate_available": false,
  "raw_text_exposed_to_model": null,
  "truncation": null
}
```

The full response also returns `secret_ref` at the **top level**
alongside `payload` and `payload_meta` — it is not nested inside
`payload_meta`. Pricing, by contrast, is nested inside `payload_meta`.

`memory_plan_inference` returns the same shape as `memory_recall`, with
the additional guarantee that `payload`, top-level `secret_ref`, and
`payload_meta.pricing` are populated (it is the executable plan
surface). Agents check the top-level location first and fall back to
`payload_meta.secret_ref` for compatibility with pre-v0.3.0 servers.

## Memory ask telemetry

Returned by `gosh memory data ask --json` / `memory_ask` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `answer` | string | LLM-generated answer |
| `profile_used` | string | Inference profile actually used |
| `profile_fallback` | bool | Whether a fallback profile was used |
| `recommended_profile` | string | Profile recommended by complexity routing |
| `tool_called` | string \| null | Name of tool invoked, if any (`date_diff`, `count_items`, `get_more_context`, …) |
| `tool_results` | list | Tool call results, when tools used |
| `budget_exceeded` | bool | Whether token budget was exceeded |
| `estimated_cost` | float | Estimated cost in USD |
| `inference_tokens` | object | `{input, output}` token counts |
| `retrieval_families` | list | Retrieval families used in the recall phase |
| `search_family` | string | Search family used |
| `retrieved_count` | int | Facts retrieved in the recall phase |
| `facts_used_count` | int | Facts actually included in inference context |
| `payload_meta` | object | Payload metadata from the recall phase |
| `process_cost_summary` | object | LLM cost breakdown |

## Memory stats telemetry

Returned by `gosh memory data stats` / `memory_stats` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `telemetry_version` | int | Schema version (currently 1) |
| `granular` | int | Atomic fact count |
| `consolidated` | int | Consolidated fact count |
| `cross_session` | int | Cross-session fact count |
| `secrets` | int | Stored secret count |
| `index_built` | bool | Vector index built? |
| `tiers_dirty` | bool | Consolidated/cross-session need recompute? |
| `index_status` | object | Detailed index state |
| `agent_id`, `swarm_id`, `scope` | string | Caller context as resolved on the server |
| `raw_sessions_count` | int | Number of raw sessions stored |
| `source_records_count` | int | Number of source records |
| `raw_session_status_counts` | object | Breakdown by status (`active`, `archived`, …) |
| `all_raw_sessions_active` | bool | Whether all raw sessions are active |
| `logical_source_count` | int | Distinct logical sources |
| `part_source_count` | int | Part/chunk source count |
| `embed_provider` | string | `openai` or `local` |
| `embed_model` | string | Embedding model id |
| `last_write_log_claim` | object | State of the async write-log worker |
| `process_cost_summary` | object | Cost breakdown for LLM calls |
| `process_cost_scope` | string | Always `process` — costs are process-scoped, not per-key |

`process_cost_summary` shape (flat, accumulated across all providers
the process talks to):

```jsonc
{
  "input_tokens": 12345,
  "output_tokens": 6789,
  "embed_tokens": 4567,
  "cost_usd": 0.42,
  "calls": 42
}
```

> `process_cost_summary` reflects costs accumulated during the current
> memory process lifetime, not per-key costs. A process restart resets
> these counters.

> The shape is intentionally flat — there is no per-provider breakdown
> in the current implementation, and prompt-cache token fields
> (`cache_read_tokens`, `cache_creation_tokens`) are not exposed.

## Agent telemetry

Returned by `gosh agent task status --json` / `agent_status` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `telemetry_version` | int | Schema version (currently 1) |
| `task_id` | string | External task id |
| `task_fact_id` | string | Internal memory fact id (authoritative task) |
| `status` | string | `pending`, `active`, `done`, `failed`, `partial_budget_overdraw`, `too_complex` |
| `phase` | string | `bootstrap`, `execution`, or `review` |
| `iteration` | int | Execution loop iteration count |
| `started_at` | string | ISO 8601 task start |
| `finished_at` | string \| null | ISO 8601 task end |
| `shell_spent` | float | SHELL budget consumed (1 SHELL = $0.01 at profile max rate) |
| `profile_used` | string \| null | Profile prefix derived from the model id (e.g., `anthropic`) |
| `backend_used` | string \| null | Backend type: `anthropic`, `openai`, `groq`, `google`, `inception`, `local_cli` |
| `tool_trace` | list[string] | Tool calls flattened into strings of the form `"<tool>:ok"` or `"<tool>:error"`. The agent emits a string list for compactness in fact metadata; it is not a list of objects. |
| `error` | string \| null | Error message when failed |
| `session` | string \| null | Prose summary of execution |
| `result` | string \| null | Final answer text |
| `session_fact` | object \| null | Wrapper assembled by the agent around the persisted `task_session` fact (id, kind, fact, metadata) |
| `result_fact` | object \| null | Wrapper assembled by the agent around the persisted `task_result` fact |
| `deliverable_fact` | object \| null | Wrapper assembled by the agent around the persisted `task_deliverable` fact, when present |
| `deliverable_fact_id` | string \| null | Convenience pointer to the deliverable fact id |
| `deliverable_kind` | string \| null | `document` or `code` |
| `created_at` | string | Task creation timestamp |

The `*_fact` keys are agent-side response wrappers. Memory itself
stores the underlying records under the kinds `task_session`,
`task_result`, and `task_deliverable` — see the table below. The agent
fetches those facts from memory and re-exposes them under the wrapper
keys so a single `agent_status` call returns the full task footprint
without round-tripping through `memory_get`.

### Task lifecycle artifacts in memory

The agent persists three canonical fact kinds per task:

| Kind | Purpose | Key metadata fields |
|------|---------|---------------------|
| `task` | Authoritative task assignment | `task_id`, `work_key`, `context_key`, `deliverable_kind`, `workspace_dir`, `target=["agent:X"]` |
| `task_session` | Runtime trace | `task_fact_id`, `status`, `phase`, `iteration`, `shell_spent`, `profile_used`, `backend_used`, `started_at`, `finished_at`, `tool_trace` |
| `task_result` | Final answer | (same as session) plus the answer text in `fact` |
| `task_deliverable` | External deliverable (document/code), when applicable | `task_fact_id`, `artifact_role: "terminal"`, `deliverable_kind`, `source` (`agent_result` or `external_cli`) |

`task_deliverable` is new: it captures the document or code artifact a
local-CLI execution produced, with provenance separating
agent-generated output from external-CLI output.

## Harness telemetry artifact

Per-case `telemetry.json` produced by production harness runs.

| Section | Contents |
|---------|----------|
| `validity` | Run validity classification (`valid`, `invalid`, `partial`) |
| `build_index` | Index build timing and stats |
| `recall` | Full recall telemetry (same fields as Memory Recall above) |
| `ask` | Full ask telemetry (same fields as Memory Ask above) |
| `judge` | Judge verdict, score, reasoning |

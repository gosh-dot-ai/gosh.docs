# Telemetry Contract

End-to-end production telemetry surfaces across the GOSH system.

All telemetry fields documented here are available via `--json` output or structured MCP responses on the `dev` branch.

## Memory Recall Telemetry

Returned by `gosh memory recall --json` / `memory_recall` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `context` | string | Retrieved facts formatted as context string |
| `retrieved_count` | int | Number of facts retrieved |
| `query_type` | string | Detected query type (lookup, temporal, aggregate, synthesize, etc.) |
| `token_estimate` | int | Estimated token count of context |
| `complexity_hint.score` | float | 0.0-1.0 complexity score |
| `complexity_hint.level` | int | 1-5 complexity level |
| `complexity_hint.signals` | list | Complexity signal names |
| `complexity_hint.retrieval_complexity` | float | Retrieval-side complexity |
| `complexity_hint.content_complexity` | float | Content-side complexity |
| `complexity_hint.query_complexity` | float | Query-side complexity (multi-constraint, decision-making, tradeoff signals) |
| `complexity_hint.dominant` | string | Dominant complexity axis: "retrieval", "content", "query", or "tie" |
| `sessions_in_context` | int | Sessions included in context |
| `total_sessions` | int | Total sessions available |
| `coverage_pct` | float | Session coverage percentage |
| `recommended_prompt_type` | string | Suggested prompt style for LLM |
| `use_tool` | bool | Whether tools should be used |
| `retrieval_families` | list | Retrieval routing families used |
| `search_family` | string | Search family (conversation, document, auto) |
| `recommended_profile` | string | Recommended inference profile (when profiles configured) |
| `payload` | object | LLM-ready payload (when profiles configured, not truncated) |
| `payload_meta` | object | Payload metadata (token counts, truncation info, provider) |
| `retrieved_episode_ids` | list | Episode IDs considered |
| `actual_injected_episode_ids` | list | Episode IDs actually injected into context |
| `selection_scores` | list | Per-fact retrieval scores |
| `runtime_trace` | object | Performance timing trace |

## Memory Ask Telemetry

Returned by `gosh memory ask --json` / `memory_ask` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `answer` | string | LLM-generated answer |
| `profile_used` | string | Inference profile actually used |
| `profile_fallback` | bool | Whether fallback profile was used |
| `recommended_profile` | string | Profile recommended by complexity routing |
| `tool_called` | bool | Whether inference invoked tools |
| `tool_results` | list | Tool call results (if tools used) |
| `budget_exceeded` | bool | Whether token budget was exceeded |
| `estimated_cost` | float | Estimated cost in USD |
| `retrieval_families` | list | Retrieval families used in recall phase |
| `search_family` | string | Search family used |
| `retrieved_count` | int | Facts retrieved in recall phase |
| `payload_meta` | object | Payload metadata from recall |

## Memory Stats Telemetry

Returned by `gosh memory stats` / `memory_stats` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `telemetry_version` | int | Schema version (currently 1) |
| `granular` | int | Extracted atomic fact count |
| `consolidated` | int | Consolidated fact count |
| `cross_session` | int | Cross-session fact count |
| `index_built` | bool | Whether vector index is built |
| `raw_sessions_count` | int | Number of raw sessions stored |
| `source_records_count` | int | Number of source records |
| `raw_session_status_counts` | object | Breakdown by status (active, etc.) |
| `all_raw_sessions_active` | bool | Whether all raw sessions are active |
| `logical_source_count` | int | Distinct logical sources |
| `part_source_count` | int | Part/chunk source count |
| `process_cost_summary` | object | Cost breakdown for LLM calls |
| `process_cost_scope` | string | Always `"process"` -- cost is process-scoped, not per-key |

**Important:** `process_cost_summary` reflects costs accumulated during the current process lifetime, not per-key costs. A process restart resets these counters.

## Agent Telemetry

Returned by `gosh agent <NAME> task status --json` / `agent_status` MCP tool.

| Field | Type | Description |
|-------|------|-------------|
| `telemetry_version` | int | Schema version (currently 1) |
| `task_id` | string | External task ID |
| `task_fact_id` | string | Internal memory fact ID |
| `status` | string | Effective status: pending, done, failed |
| `phase` | string | Current phase: bootstrap, execution, review |
| `iteration` | int | Loop iteration count |
| `started_at` | string | ISO 8601 task start timestamp |
| `finished_at` | string | ISO 8601 task end timestamp |
| `shell_spent` | float | SHELL budget consumed |
| `profile_used` | string | Model profile used for execution |
| `backend_used` | string | Backend type: `anthropic_api`, `claude_cli`, `codex_cli`, `gemini_cli` |
| `tool_trace` | list | Tool calls with success status |
| `session_fact` | object | Full structured session fact |
| `result_fact` | object | Full structured result fact |
| `created_at` | string | Task creation timestamp |

## Harness Telemetry Artifact

Per-case `telemetry.json` produced by production harness runs.

| Section | Contents |
|---------|----------|
| `validity` | Run validity classification (valid, invalid, partial) |
| `build_index` | Index build timing and stats |
| `recall` | Full recall telemetry (same fields as Memory Recall above) |
| `ask` | Full ask telemetry (same fields as Memory Ask above) |
| `judge` | Judge verdict, score, reasoning |

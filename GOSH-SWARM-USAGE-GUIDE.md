# GOSH Swarm Usage Guide

Practical guide for running multi-agent swarms with the current `gosh.memory` +
`gosh.agent` + `gosh.cli` stack.

For installation and configuration, see [SETUP.md](./SETUP.md).
For how memory retrieval and extraction work, see [MEMORY-SYSTEM.md](./MEMORY-SYSTEM.md).

Related docs:
- [ARCHITECTURE.md](./ARCHITECTURE.md) -- production architecture overview
- [TELEMETRY-CONTRACT.md](./TELEMETRY-CONTRACT.md) -- telemetry field reference
- [GOSH-SWARM-COORDINATION.md](./GOSH-SWARM-COORDINATION.md) -- coordination protocol

Copyright 2026 (c) Mitja Goroshevsky and GOSH Technology Ltd.
License: MIT

---

## 1. What Works Now

You can already run a directed swarm with targeted task delivery, courier
wake-up, exact task resolution, context-aware routing, and execution through
API profiles or official coding CLIs (`claude_code_cli`, `codex_cli`,
`gemini_cli`).

## 2. Canonical Fact Contract

The implemented task contract uses top-level `target` for delivery:

```json
{
  "kind": "task",
  "fact": "Planner tea preference",
  "target": ["agent:planner"],
  "metadata": {
    "task_id": "task-b227c463",
    "workflow_id": "wf_gate_global",
    "route": "fast_path",
    "priority": 1
  }
}
```

- `target` is delivery intent only, not ACL
- stable external task ID lives in `metadata.task_id`
- persisted memory fact ID is separate and used for exact execution

Canonical execution artifacts: `kind = "task_result"` and `kind = "task_session"`,
both linking back via `metadata.task_fact_id` and `metadata.task_id`.

## 3. Two Profile Systems

Memory-side and agent-side profiles are related but separate.

**Memory-side profiles** (per memory key) produce `recommended_profile`,
`payload`, and `payload_meta` from `memory recall`. If not configured,
`complexity_hint` is still returned but `payload` and `payload_meta` are absent.

**Agent-side profiles** (per watcher process) map `fast / balanced / strong /
review / extraction` to actual execution backends (API or CLI).

An agent may use both signals together, but they are not the same layer.

## 4. CLI Executor Safety Model

- Exactly one CLI execution may run at a time per agent process
- Cooldown is global across all CLI providers (minimum `600s`)
- A Claude Code task blocks Codex and Gemini in the same agent process (and vice versa)

This is intentional -- keeps consumer-login usage conservative.

## 5. Starting a Watcher Agent

Prerequisites: log into `claude`, `codex`, and `gemini` once manually on the
machine. See [SETUP.md](./SETUP.md) for building binaries and starting memory.

### Mixed routing example

```bash
cd /path/to/gosh.agent
ANTHROPIC_API_KEY=sk-ant-... ./target/debug/gosh-agent \
  --host 127.0.0.1 \
  --port 8877 \
  --memory-url http://127.0.0.1:8875/mcp \
  --memory-token YOUR_TOKEN \
  --watch \
  --watch-key planner-e2e \
  --watch-agent-id planner \
  --watch-swarm-id swarm-alpha \
  --watch-budget 25 \
  --poll-interval 5 \
  --extraction-profile anthropic_haiku_api \
  --fast-profile claude_code_cli \
  --balanced-profile codex_cli \
  --strong-profile gemini_cli \
  --review-profile anthropic_sonnet_api
```

> **Note:** `ANTHROPIC_API_KEY` is required for the agent's Anthropic API profiles
> (extraction, review, and any `anthropic_*` execution profiles).

### Single-provider smoke runs

```bash
# Codex-only:
./target/debug/gosh-agent ... --fast-profile codex_cli --balanced-profile codex_cli --strong-profile codex_cli

# Gemini-only:
./target/debug/gosh-agent ... --fast-profile gemini_cli --balanced-profile gemini_cli --strong-profile gemini_cli

# Claude-only:
./target/debug/gosh-agent ... --fast-profile claude_code_cli --balanced-profile claude_code_cli --strong-profile claude_code_cli
```

## 6. Creating and Dispatching Tasks

```bash
./target/debug/gosh --state-dir /tmp/gosh-swarm/state agent planner task create \
  --extract memory \
  --key planner-e2e \
  --swarm-id swarm-alpha \
  --workflow-id wf_gate_global \
  --route fast_path \
  --priority 1 \
  "Planner tea preference"
```

This writes an authoritative `kind=task` fact with top-level
`target=["agent:planner"]` and flat metadata (`task_id`, `workflow_id`,
`route`, `priority`). stdout prints the external `task_id`.

## 7. Execution Flow

1. Courier SSE delivers a targeted `task`
2. Agent exact-fetches the authoritative task fact
3. Agent asks memory for context and routing signal
4. Routing selects `fast`, `balanced`, or `strong`
5. Unified profile layer selects API or official CLI backend
6. Agent writes canonical `task_result` and `task_session`

## 8. Checking Status

```bash
./target/debug/gosh --state-dir /tmp/gosh-swarm/state agent planner task status task-12345678 \
  --key planner-e2e --swarm-id swarm-alpha

./target/debug/gosh --state-dir /tmp/gosh-swarm/state agent planner task list \
  --key planner-e2e --swarm-id swarm-alpha
```

## 9. Seeding Memory with Swarm Metadata

```bash
./target/debug/gosh --state-dir /tmp/gosh-swarm/state memory store \
  "The planner prefers tea during routine checks." \
  --key planner-e2e \
  --agent-id planner \
  --swarm-id swarm-alpha \
  --scope swarm-shared \
  --target agent:planner \
  --meta workflow_id=wf_simple \
  --meta route=preference \
  --meta priority=1
```

`--meta` is flat only. Values are parsed as JSON scalars (`1` -> number,
`true` -> boolean, `null` -> null). Arrays and objects are not supported.

## 10. Current Caveats

- **Transient poll warnings** may appear in watcher logs; they do not block
  courier delivery or task completion.
- **Memory context is key-wide** -- prior artifacts influence retrieval. Use
  fresh keys for stable routing experiments.
- **`memory_set_profiles` lacks a CLI command** -- memory-side profile
  bootstrap is still done through Python. This is a usability limitation, not
  a conceptual requirement.

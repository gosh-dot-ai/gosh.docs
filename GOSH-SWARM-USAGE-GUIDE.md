# GOSH Swarm Usage Guide

Practical guide for running multi-agent swarms with the current
`gosh.memory` + `gosh.agent` + `gosh.cli` stack.

For installation and configuration, see [SETUP.md](SETUP.md).
For how memory retrieval and extraction work, see
[MEMORY-SYSTEM.md](MEMORY-SYSTEM.md).
For identity, tokens, and visibility, see
[PERMISSIONS-AND-ACL.md](PERMISSIONS-AND-ACL.md).

Related docs:

- [ARCHITECTURE.md](ARCHITECTURE.md) — production architecture overview
- [TELEMETRY-CONTRACT.md](TELEMETRY-CONTRACT.md) — telemetry field reference
- [GOSH-SWARM-COORDINATION.md](GOSH-SWARM-COORDINATION.md) — fact-based coordination protocol

Copyright 2026 (c) Mitja Goroshevsky and GOSH Technology Ltd.
License: AGPL-3.0-only

---

## 1. What works now

You can already run a directed swarm with targeted task delivery,
courier wake-up, exact task resolution, context-aware routing, and
execution through API providers (Anthropic, OpenAI, Groq, Google) or
local coding CLIs (`claude`, `codex`, `gemini`).

Identity is real: every agent has a memory principal, an agent token,
and an X25519 keypair. Provider keys live inside memory and are
delivered to the agent at task time inside an X25519-sealed envelope
(X25519 + HKDF-SHA256 + AES-256-GCM) — they never sit on the agent
host as plaintext env vars.

## 2. Canonical fact contract

The implemented task contract uses top-level `target` for delivery:

```json
{
  "kind": "task",
  "fact": "Planner tea preference",
  "scope": "swarm-shared",
  "target": ["agent:planner"],
  "metadata": {
    "task_id": "task-b227c463",
    "workflow_id": "wf_gate_global",
    "route": "fast_path",
    "priority": 1
  }
}
```

- `target` is delivery intent only, not ACL. Visibility is controlled
  by `scope` and the read ACL.
- The stable external task id lives in `metadata.task_id`.
- The persisted memory fact id is separate and used for exact
  execution lookup.

Canonical execution artifacts written by the agent: `kind = "task_result"`,
`kind = "task_session"`, and (when applicable) `kind = "task_deliverable"`.
All link back via `metadata.task_fact_id` and `metadata.task_id`.

## 3. Two profile systems

Memory-side and agent-side profiles are related but separate.

**Memory-side profiles** (per memory namespace) produce
`recommended_profile`, `payload`, and `payload_meta` from
`memory_recall` and `memory_plan_inference`. Each profile names a
model, a `secret_ref`, and `pricing`. Memory selects the profile by
mapping the complexity hint level (1–5) to a profile name.

**Agent-side execution** consumes whatever memory returned: model id,
secret reference, pricing. The agent itself does not carry a profile
catalog. Memory is authoritative.

For local-CLI execution, the profile's `payload_meta.local_cli` block
points at one of `claude`, `codex`, or `gemini`, and the agent invokes
that binary instead of an API call.

## 4. CLI executor concurrency

- A single agent process tracks `in_flight_tasks` and caps itself at
  `max_parallel_tasks` (default 4).
- Watch dispatch deduplicates on `task_fact_id` — both courier delivery
  and poll fallback check `task_result` lookups before claiming.
- There is no global 600-second cooldown across coding CLIs in the
  current implementation. The single-flight constraint on the data
  plane is enforced at memory-write time.

## 5. Bringing up a watcher agent

Prerequisites:

- Memory running and bootstrapped (see [SETUP.md](SETUP.md) §1).
- A swarm created and the agent's principal granted membership.
- Provider keys stored as memory secrets (not as env vars on the agent
  host).

### 5.1 Create the agent

```bash
gosh agent create planner --memory local --swarm swarm-alpha
```

This:

1. Calls `principal_create` for `agent:planner`.
2. Issues an agent-kind token.
3. Generates an X25519 keypair, registers the public key with memory.
4. Grants membership in `swarm-alpha`.
5. Builds a join token, saves principal/join/secret keys to the OS
   keychain, writes `~/.gosh/agent/instances/planner.toml`.

### 5.2 Optionally hook coding CLIs

```bash
gosh agent setup --instance planner
```

By default writes hooks and MCP at **project scope** under `<cwd>/`.
Pass `--scope user` only if you really want all-host capture.
Restrict with `--platform claude` (repeatable). Override the auto-
detected memory key with `--key <KEY>`. Enable swarm capture with
`--swarm <SWARM>`.

### 5.3 Start in watch mode

```bash
gosh agent start --instance planner \
  --watch --watch-key planner-e2e --watch-swarm-id swarm-alpha \
  --watch-budget 25 --poll-interval 5
```

Watch flags can also be supplied via CLI args at start time:

| Flag | Purpose |
|------|---------|
| `--watch-key` | Namespace to watch for tasks |
| `--watch-context-key` | Separate retrieval namespace if different from work key |
| `--watch-agent-id` | Override agent id (rare; defaults to principal) |
| `--watch-swarm-id` (alias `--watch-swarm`) | Swarm subscription |
| `--watch-budget` | SHELL budget per task (default 10.0) |
| `--poll-interval` | Seconds between fallback polls (default 30) |

## 6. Creating and dispatching tasks

```bash
gosh agent task create --instance planner \
  --key planner-e2e \
  --swarm-id swarm-alpha \
  --scope swarm-shared \
  --workflow-id wf_gate_global \
  --route fast_path \
  --priority 1 \
  "Planner tea preference"
```

This writes an authoritative `kind=task` fact with top-level
`target=["agent:planner"]` and flat metadata. stdout prints the
external `task_id`.

Other useful flags on `task create`:

- `--context-key` — distinct retrieval namespace
- `--task-id` — pre-assign an external id
- `--metadata` — JSON object merged into task metadata
- `--target` — repeatable, overrides the inferred `agent:<instance>`

## 7. Execution flow

1. Courier SSE delivers a targeted `task` to the watcher.
2. Agent exact-fetches the authoritative task fact (poll fallback if
   courier dropped).
3. Agent calls `memory_recall` for context.
4. Agent calls `memory_plan_inference` for the executable plan
   (model id, `secret_ref`, pricing, optional `local_cli`).
5. Agent decrypts the sealed secret with its X25519 private key.
6. Bootstrap → execution → review phases run.
7. Agent persists `task_session` (canonical trace), `task_result`
   (final answer), and optionally `task_deliverable` (terminal
   document/code).

## 8. Checking status

```bash
gosh agent task status --instance planner task-12345678 \
  --key planner-e2e --swarm-id swarm-alpha --json

gosh agent task list --instance planner \
  --key planner-e2e --swarm-id swarm-alpha
```

The `--json` payload follows the schema in
[TELEMETRY-CONTRACT.md](TELEMETRY-CONTRACT.md).

## 9. Seeding memory with swarm metadata

```bash
gosh memory data store \
  --key planner-e2e --swarm swarm-alpha \
  --scope swarm-shared \
  --target agent:planner \
  --meta workflow_id=wf_simple \
  --meta route=preference \
  --meta priority=1 \
  "The planner prefers tea during routine checks."
```

`--meta` is flat. Values are parsed as JSON scalars (`1` → number,
`true` → boolean, `null` → null). Arrays and objects are not supported
on the CLI surface; for nested metadata, use the MCP API directly via a
proxy or import path.

If your CLI does not yet have a provisioned agent token, run
`gosh memory auth provision-cli` first.

## 10. Local-CLI execution

If the resolved profile carries a `payload_meta.local_cli` block, the
agent does not call a remote API. Instead it invokes the named coding
CLI (`claude`, `codex`, `gemini`) with a workspace directory the task
metadata defines.

When the local CLI completes, the agent validates the terminal
deliverable. If the task metadata declared
`deliverable_kind=document|code`, a `task_deliverable` fact is written
with `metadata.artifact_role="terminal"`,
`metadata.source="external_cli"`, and the deliverable content as the
fact body. Failed or invalid deliverables retry up to
`max_retries` (default 2) before the task transitions to `failed`.

## 11. Multi-agent example

Three agents on the same memory, three roles:

```bash
# coder (writes code in response to tasks)
gosh agent create coder-a --memory local --swarm feature-x
gosh agent setup --instance coder-a --platform claude
gosh agent start --instance coder-a --watch --watch-key feature-x \
  --watch-swarm-id feature-x

# coder (parallel)
gosh agent create coder-b --memory local --swarm feature-x --port 8768
gosh agent setup --instance coder-b --platform claude
gosh agent start --instance coder-b --watch --watch-key feature-x \
  --watch-swarm-id feature-x

# reviewer (review-only role)
gosh agent create reviewer --memory local --swarm feature-x --port 8769
gosh agent setup --instance reviewer --platform claude
gosh agent start --instance reviewer --watch --watch-key feature-x \
  --watch-swarm-id feature-x
```

Coordination between them is fact-based and follows the protocol in
[GOSH-SWARM-COORDINATION.md](GOSH-SWARM-COORDINATION.md).

## 12. Current caveats

- **Per-namespace context**: prior artifacts in the namespace influence
  retrieval. Use fresh `--key` values for stable routing experiments.
- **Memory-side profile bootstrap from the CLI** is via
  `gosh memory config set` (a JSON config blob). Per-profile
  `memory_set_profiles` is also exposed as a tool but the canonical
  config surface is `memory_set_config`.
- **Hooks / MCP scope is project by default** since gosh.cli v0.5.0 and
  gosh.agent v0.7.0. Re-run `gosh agent setup` from each project root
  where you want capture; `--scope user` opts back into the old
  global behavior.
- **The `--instance` flag is post-subcommand only.** Use
  `gosh memory data store ... --instance prod`, not
  `gosh memory --instance prod data store ...`.

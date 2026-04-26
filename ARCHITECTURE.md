# GOSH Architecture

Production architecture overview. For installation see [SETUP.md](SETUP.md).
For the permission model see [PERMISSIONS-AND-ACL.md](PERMISSIONS-AND-ACL.md).

## Components

### gosh.memory (Python)

Semantic long-term memory. MCP server speaking HTTP at `:8765` by
default, plus an SSE courier endpoint for real-time fact delivery.

- Format-aware semantic extraction (conversation, document, code trace,
  agent trace, narrative, fact list)
- Three-tier fact organization: granular → consolidated → cross-session
- Query-adaptive retrieval with RRF fusion over BM25, vector, and
  entity signals
- Complexity-aware routing: a three-axis score
  (retrieval / content / query) drives `recommended_profile`
- Canonical config per namespace: model profiles, profile pricing,
  retrieval knobs, metadata schema, extraction prompts
- Authority service: principals, tokens, swarms, memberships, instance
  ACL — backed by SQLite (`authority.db`), encrypted at rest with
  `GOSH_MEMORY_ENCRYPTION_KEY`
- X25519-sealed secrets: provider keys stored inside memory and
  delivered to the agent encrypted to its X25519 public key
  (X25519 + HKDF-SHA256 + AES-256-GCM, algorithm tag
  `x25519-hkdf-sha256-aes256gcm-v1`)
- Memory Adaptation Loop (MAL): production-feedback-driven self-tuning
  — implemented as 7 MCP tools, optional and disabled by default
- Async write-log (`memory_write` / `memory_write_status`) for
  streaming ingest without blocking on extraction
- Versioning surface: `memory_edit`, `memory_retract`, `memory_purge`,
  `memory_redact`, `memory_get_versions`

See [MEMORY-SYSTEM.md](MEMORY-SYSTEM.md) for the retrieval and
extraction pipelines, and [MEMORY-ADAPTATION-LOOP.md](MEMORY-ADAPTATION-LOOP.md)
for MAL.

### gosh.agent (Rust)

Autonomous task executor with five personas, all exposed by the same
`gosh-agent` binary:

- **Serve** — MCP server at `:8767` by default. Receives tasks, executes
  them through bootstrap → execution → review, writes back canonical
  `task_session`, `task_result`, and `task_deliverable` facts.
- **Capture** — invoked as a hook by Claude Code / Codex / Gemini.
  Buffers the captured prompt or response and writes it into memory
  via `memory_write`.
- **MCP-proxy** — stdio MCP server. Coding CLIs connect to this; it
  injects the agent's principal token, optional `default_key`, and
  optional `default_swarm` into outbound calls and forwards to the real
  memory authority.
- **Setup** — one-shot CLI that detects coding CLIs on the host and
  writes hooks + MCP entries at project or user scope.
- **Replay-buffer** — drains the per-instance capture buffer to
  authority via `memory_write`. Recovers captures that were buffered
  when memory was unreachable.

Highlights:

- Three execution phases: bootstrap (resolve task, fetch context, fetch
  plan), execution (loop up to 32 iterations against the LLM), review
  (verify result, optionally repair or retry)
- SHELL budget (1 SHELL = $0.01 at profile max rate). Twenty percent of
  the per-task budget is reserved for review by default
- Profiles, models, secret references, and pricing all come from memory
  via `memory_plan_inference` — the agent itself carries no hardcoded
  profile catalog
- Backends: Anthropic, OpenAI, Groq, Google, Inception (selected by
  the prefix on the model id), plus `local_cli` for invoking
  `claude` / `codex` / `gemini` directly with workspace handoff
- Watch mode: courier SSE subscription with poll fallback every
  `--poll-interval` seconds. Per-agent concurrency cap
  (`max_parallel_tasks`, default 4). Dedup via `task_fact_id` lookup
  before claim
- Setup persona writes hooks and MCP entries at project scope by
  default — a privacy-safe default that the v0.7.0 release introduced
  to stop cross-project prompt leakage

### gosh.cli (Rust, binary `gosh`)

Operator-facing orchestrator. Manages instances, secrets, and
lifecycle; forwards data and auth operations to memory and agent over
MCP.

- Per-instance config under `~/.gosh/{memory,agent}/instances/<name>.toml`
- OS keychain (or `--test-mode` file keychain) for tokens and X25519
  private keys
- Memory modes: `local` (managed by CLI), `remote` (external server),
  `ssh` (CLI ships memory to a remote host — partial)
- Memory runtimes: `binary` and `docker`
- Top-level groups:
  - `gosh memory`: setup / start / stop / status / logs / instance /
    init / data / auth / secret / config / prompt
  - `gosh agent`: create / import / setup / start / stop / status /
    logs / instance / bootstrap / task
  - `gosh setup`: download/install components (memory image + agent
    binary; CLI install is print-only for safety)
  - `gosh bundle`: build offline component bundle
  - `gosh status`: union view of all instances
- The `--instance` flag is post-subcommand only (since v0.5.0). Writers
  (`memory setup local`, `agent create`, `agent import`) take the name
  via `--name` or positional argument

### gosh.docs

This repository. Canonical cross-repo documentation. Component-local
docs (READMEs, CHANGELOGs, specs) live in each component's repo and
should be the source of truth for component internals; this repo
covers the system as a whole.

## Data flow

```
                      ┌────────────────┐
   Operator  ─────────┤   gosh (CLI)   ├──── Direct API client
                      └────┬───────┬───┘
                           │       │
       admin / agent token │       │ admin token, x-server-token
                           ▼       ▼
        ┌───────────────────────────────────────┐
        │   gosh.memory  (MCP at :8765)         │
        │     POST /mcp    tool calls           │
        │     GET  /mcp/sse   courier           │
        │     GET  /health    perimeter check   │
        │     authority.db (principals, tokens, │
        │       swarms, memberships, instances) │
        │     memory_<key>.db (facts + index)   │
        └────┬───────────────┬──────────────────┘
             │               │
             │ courier SSE   │ MCP (recall, plan_inference,
             │   (push)      │   ingest_asserted_facts,
             ▼               │   store, query, get, …)
        ┌─────────────┐      │
        │ gosh-agent  │◀─────┘
        │  (MCP :8767)│
        │  bootstrap →│      ┌─────────────────┐
        │  execution →│ ───▶ │  Coding CLI     │
        │  review     │      │  Claude / Codex │
        └─────┬───────┘      │  / Gemini       │
              │              └────────┬────────┘
              │  capture / mcp-proxy  │
              └───────────────────────┘
```

The arrows are loose: agents do not call other agents; they coordinate
exclusively through facts in memory (see
[GOSH-SWARM-COORDINATION.md](GOSH-SWARM-COORDINATION.md)). Coding CLIs
talk to memory only via the agent's stdio MCP proxy, which injects auth.

## Configuration

Models and provider keys live inside memory:

- `memory_set_config` — embedding, librarian, profile names, profile
  configs (model + secret_ref + pricing), retrieval tuning, metadata
  schema
- `memory_store_secret` — provider keys, encrypted at rest,
  X25519-sealed for delivery to agents

Agents do not carry model config or API keys. They:

1. Call `memory_recall` for context.
2. Call `memory_plan_inference` for the model id, `secret_ref`, and
   pricing.
3. Decrypt the sealed secret with their X25519 private key (held in
   the OS keychain).
4. Make the API call.

There is no `services.toml`. Instance configuration is per-instance
TOML under `~/.gosh/{memory,agent}/instances/`. See [SETUP.md](SETUP.md).

## Versioning and stability

- Storage format is in flux; data may need migration between releases.
  Keep source documents and re-ingest on schema bumps when possible.
- MCP tool names and signatures are evolving; recent releases expanded
  the surface (61 tools at v0.3.1 — auth, secrets, MAL, write-log,
  versioning, prompts, schema). See the gosh.memory README CHANGELOG
  for the current list.
- Every breaking change in gosh.cli is noted in its CHANGELOG with a
  migration step.

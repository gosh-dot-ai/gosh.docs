# Setup Guide

How to install gosh and bring up memory + agents in different
configurations. The canonical path uses `gosh.cli` to manage everything;
the lower-level modes show what `gosh.cli` is doing under the hood.

---

## Components and prerequisites

- `gosh.cli` (binary `gosh`) — orchestrator. Manages instances,
  secrets, lifecycle, and forwards commands to memory and agents.
- `gosh.memory` — Python service that ships as a Docker image (default)
  or a local Python install. Speaks MCP over HTTP.
- `gosh.agent` (binary `gosh-agent`) — Rust service. Speaks MCP over
  HTTP, can also run as a stdio MCP proxy to inject auth into coding
  CLIs.

Prerequisites:

- macOS or Linux (x86_64 / aarch64). Windows is in design.
- Docker (default memory runtime). Optional if you build memory from
  source and run as a local binary.
- Python 3.10+ (only if running memory from source).
- Rust 1.86+ (only if building gosh.cli or gosh.agent from source).
- An LLM provider key for memory's extraction and inference roles
  (OpenAI, Anthropic, Groq, Google, or Inception).
- For coding-CLI capture: at least one of Claude Code, Codex, or
  Gemini installed locally with credentials.

---

## Mode 1: Quickstart with `gosh.cli` (recommended)

This is the path the CLI's own quickstart and wizard prompt walk you
through. About ten minutes from a fresh machine to a working agent.

### 1.1 Install the CLI

```bash
curl -fsSL https://raw.githubusercontent.com/gosh-dot-ai/gosh.cli/main/install.sh | bash
```

The script downloads the latest `gosh` binary into `/usr/local/bin/`.
For private mirrors / forks, see the gosh.cli README.

### 1.2 Pull memory and agent components

```bash
gosh setup
```

Idempotent: downloads the `gosh-agent` binary and loads the
`gosh-memory` Docker image. Re-running skips components that are
already at the requested version. Pin a version with
`gosh setup --version vX.Y.Z`. Install only one component with
`--component agent` or `--component memory`.

### 1.3 Bring memory up

```bash
gosh memory setup local --data-dir ~/gosh-data --runtime docker
gosh memory start
curl -fsS http://localhost:8765/health   # {"status":"ok"}
```

`memory setup local` writes an instance config to
`~/.gosh/memory/instances/local.toml` and stores the encryption key,
bootstrap token, and server token in the OS keychain. `memory start`
exports them as env vars to the memory process and bootstraps the first
admin token on first run.

For an external memory image, use `--runtime binary --binary <PATH>`
instead of `--runtime docker`.

For a non-bind public address (so remote agents can reach it), pass
`--public-url https://memory.example.com:8765` to `memory setup local`.
Local admin commands keep using the bind URL; agents are issued bootstrap
files with the public URL.

### 1.4 Bootstrap the namespace

A namespace ("key") is the unit of isolation inside memory. Each key has
its own facts, embeddings, and config. Most operators run a single key
and put everything under it; multi-tenant deployments use one key per
tenant.

```bash
gosh memory init --key quickstart
```

This creates the `_instance_config` for the namespace with the current
admin as owner. Prints the owner principal — note it for the swarm step.

### 1.5 Store provider keys as memory secrets

Keys live inside memory, encrypted at rest, and delivered to the agent
at execution time inside an X25519-sealed envelope (X25519 +
HKDF-SHA256 + AES-256-GCM, algorithm tag
`x25519-hkdf-sha256-aes256gcm-v1`). They never sit in env vars on the
agent host.

```bash
export OPENAI_API_KEY=sk-...
gosh memory secret set-from-env OPENAI_API_KEY --name openai --key quickstart

# or paste the value directly:
gosh memory secret set anthropic sk-ant-... --key quickstart
```

`--scope` defaults to `system-wide`. Use `swarm-shared --swarm <ID>`
for a swarm-scoped secret, `agent-private --agent-id <NAME>` for an
agent-scoped one.

### 1.6 Configure extraction and inference profiles

Memory needs to know which model to use for each role. There are no
implicit defaults — operators set this explicitly.

```bash
gosh memory config set --key quickstart '{
  "schema_version": 1,
  "embedding_model": "text-embedding-3-large",
  "embedding_secret_ref": {"name": "openai", "scope": "system-wide"},
  "librarian_profile": "extraction",
  "profiles": {"1": "fast", "2": "fast", "3": "balanced", "4": "balanced", "5": "strong"},
  "profile_configs": {
    "extraction": {
      "model": "anthropic/claude-sonnet-4-6",
      "secret_ref": {"name": "anthropic", "scope": "system-wide"},
      "pricing": {"input_per_1k": 0.003, "output_per_1k": 0.015}
    },
    "fast": {
      "model": "anthropic/claude-haiku-4-5",
      "secret_ref": {"name": "anthropic", "scope": "system-wide"},
      "pricing": {"input_per_1k": 0.001, "output_per_1k": 0.005}
    },
    "balanced": {
      "model": "anthropic/claude-sonnet-4-6",
      "secret_ref": {"name": "anthropic", "scope": "system-wide"},
      "pricing": {"input_per_1k": 0.003, "output_per_1k": 0.015}
    },
    "strong": {
      "model": "anthropic/claude-opus-4-6",
      "secret_ref": {"name": "anthropic", "scope": "system-wide"},
      "pricing": {"input_per_1k": 0.015, "output_per_1k": 0.075}
    }
  }
}'
```

Provider routing is by the prefix on `model`:

| Prefix | Provider |
|--------|----------|
| `anthropic/` | Anthropic API |
| `qwen/`, `meta-llama/` | Groq API |
| `google/` | Google Gemini API |
| `inception/` | Inception Labs |
| (bare name like `gpt-4o`) | OpenAI |

`secret_ref.name` is a label that points at a stored secret, not a
routing key.

`profiles` maps complexity-hint levels (1–5) to profile names. Profile
names are arbitrary — `fast`, `balanced`, `strong` are conventional but
you can name them anything as long as `profile_configs` defines them.

### 1.7 Create a swarm and provision yourself

A swarm is the group through which agents share facts. Even a
single-agent setup wants one, since `swarm-shared` is the default scope
for things you want recallable from another session.

```bash
# from the output of `gosh memory init` above
OWNER_PRINCIPAL=service:<your-username>
gosh memory auth swarm create quickstart-swarm --owner "$OWNER_PRINCIPAL"

# provision the CLI as an agent identity so you can run data commands
gosh memory auth provision-cli
```

`provision-cli` creates `agent:cli-<your-username>`, an internal swarm
called `cli`, grants membership, and issues an agent token to the
keychain. After this, `gosh memory data ...` works.

### 1.8 First store and recall

```bash
gosh memory data store --key quickstart --scope swarm-shared --swarm quickstart-swarm \
  "Alice is a senior engineer at ACME Corp. She joined in 2024 and leads the platform team."

gosh memory data build-index --key quickstart

gosh memory data recall --key quickstart --swarm quickstart-swarm "Who is Alice?"
gosh memory data ask    --key quickstart --swarm quickstart-swarm "What does Alice do?"
```

`recall` returns retrieved facts and metadata, no LLM call. `ask` runs
recall then inference and returns a prose answer.

### 1.9 Create an agent

```bash
gosh agent create myagent --memory local --swarm quickstart-swarm
```

This:

1. Calls `principal_create` to register `agent:myagent`
2. Issues an `agent`-kind token for it
3. Generates an X25519 keypair and registers the public key with memory
4. Grants membership in `quickstart-swarm`
5. Builds a join token (memory URL + transport token + principal token + cert pin)
6. Saves principal token, join token, and X25519 secret key to the OS keychain
7. Writes the agent config to `~/.gosh/agent/instances/myagent.toml`

Pass `--binary <PATH>` to lock the agent to a specific `gosh-agent`
binary; otherwise it resolves at start time from `cfg.binary` or `PATH`.

### 1.10 Hook a coding CLI to the agent

```bash
gosh agent setup --instance myagent
```

By default this writes hooks and MCP registration **at project scope**:
`<cwd>/.claude/settings.json`, `<cwd>/.codex/hooks.json`,
`<cwd>/.gemini/settings.json`, and `<cwd>/.mcp.json` for the Claude
project. These fire only when the coding CLI is launched from this
directory.

To enroll across all projects on the host (rare, leaks prompts across
projects), pass `--scope user`. Codex MCP is always user-global because
upstream codex has no per-project mode; only Codex hooks honor `--scope`.

Filter by platform: `--platform claude` (repeatable) restricts setup to
specific CLIs.

Override the auto-detected memory key: `--key <KEY>`.

Enable swarm-shared capture: `--swarm <SWARM>`.

### 1.11 Start the agent (watch mode)

```bash
gosh agent start --instance myagent \
  --watch --watch-key quickstart --watch-swarm-id quickstart-swarm \
  --watch-budget 10
```

Watch mode subscribes to courier SSE for tasks targeted at this agent
and falls back to polling every `--poll-interval` seconds (default 30).
Each task gets up to `--watch-budget` SHELL (1 SHELL = $0.01 at the
profile's max rate) to complete.

CLI flags for `gosh agent start`:

| Flag | Default | Purpose |
|------|---------|---------|
| `--watch` | off | Enable watch mode |
| `--watch-key` | from saved cfg | Namespace to watch for tasks |
| `--watch-context-key` | watch-key | Separate namespace for retrieval |
| `--watch-agent-id` | derived from principal | Agent id to filter on |
| `--watch-swarm-id` (alias `--watch-swarm`) | from saved cfg | Swarm id to subscribe under |
| `--watch-budget` | 10.0 | SHELL budget per task |
| `--poll-interval` | 30 | Seconds between fallback polls |
| `--binary` | from saved cfg or `PATH` | Override `gosh-agent` path |

CLI args override saved config; saved config is the fallback. Either
must define `watch_key` and `watch_swarm_id` for watch to start.

### 1.12 Verify

```bash
gosh status                 # all instances
gosh memory status          # memory only
gosh agent status --instance myagent
gosh memory logs            # tail memory
gosh agent logs --instance myagent
```

Logs land at `~/.gosh/run/{memory,agent}_<name>.log`. PIDs at
`~/.gosh/run/{memory,agent}_<name>.pid`.

### 1.13 Run a task end-to-end

```bash
TASK_ID=$(gosh agent task create --instance myagent \
  --key quickstart --swarm-id quickstart-swarm \
  --scope swarm-shared --budget 10 \
  "Summarize what we know about Alice")

gosh agent task run    --instance myagent $TASK_ID --key quickstart --swarm-id quickstart-swarm
gosh agent task status --instance myagent $TASK_ID --key quickstart --swarm-id quickstart-swarm --json
gosh agent task list   --instance myagent --key quickstart --swarm-id quickstart-swarm
```

`task create` writes an authoritative `kind=task` fact with
`target=["agent:myagent"]`. The agent picks it up via courier (or poll
fallback) and runs it through bootstrap → execution → review.

`task create` accepts:

| Flag | Purpose |
|------|---------|
| `--key` | Memory namespace for the task |
| `--scope` | Default `agent-private`; use `swarm-shared` for visible-to-swarm tasks |
| `--swarm-id` (alias `--swarm`) | Swarm id |
| `--context-key` | Distinct retrieval namespace if different from work key |
| `--task-id` | External task id (otherwise auto-generated) |
| `--workflow-id` | Workflow id for grouping |
| `--metadata` | JSON object merged into task metadata |
| `--route` | Routing hint surfaced to memory's plan inference |
| `--target` | Override target principal(s) (repeatable) |
| `--priority` | Integer priority (default 0) |

### 1.14 Stop

```bash
gosh agent stop --instance myagent
gosh memory stop
```

Order does not strictly matter — agent can outlive memory and just stop
seeing tasks. Production deployments typically stop agents first.

---

## On-disk layout

```
~/.gosh/
├── memory/
│   ├── instances/
│   │   └── <name>.toml             # instance config (mode, runtime, url, paths)
│   └── current                     # name of the default instance
├── agent/
│   ├── instances/
│   │   └── <name>.toml             # instance config (memory, host, port, watch)
│   └── current
└── run/
    ├── memory_<name>.pid
    ├── memory_<name>.log
    ├── memory_<name>.container     # docker runtime only
    ├── agent_<name>.pid
    ├── agent_<name>.log
    └── agent_<name>_bootstrap.tmp  # temp bootstrap (deleted after start)
```

OS keychain entries:

```
gosh / memory/<instance>   →  encryption_key, bootstrap_token, server_token,
                              admin_token, agent_token  (JSON blob)
gosh / agent/<name>        →  principal_token, join_token, secret_key  (JSON blob)
```

---

## Mode 2: Direct MCP server (no `gosh.cli`)

Run gosh.memory standalone for integration with Claude Code, Codex,
Anthropic API, OpenAI Responses, Gemini, or any MCP-compatible client.
Skip if you are using `gosh.cli` — it does this for you.

### 2.1 Install from source

```bash
git clone https://github.com/gosh-dot-ai/gosh.memory.git
cd gosh.memory
python3 -m venv .venv
.venv/bin/pip install -e . -r requirements.txt

# optional extras:
.venv/bin/pip install -e ".[local-embed]"  # local embeddings, no OpenAI dep
.venv/bin/pip install cryptography         # TLS support
```

### 2.2 Configure provider (writes ~/.gosh-memory/config.json)

```bash
.venv/bin/python -m src.cli setup --provider anthropic --api-key sk-ant-...
# Providers: openai | groq | anthropic | google | inception
```

Or set env vars: `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, `OPENAI_API_KEY`,
`GOOGLE_API_KEY`. `GOSH_EXTRACTION_MODEL`, `GOSH_INFERENCE_MODEL`,
`GOSH_JUDGE_MODEL`, `GOSH_EMBED_MODEL` override per-role models.

### 2.3 Start the server

```bash
.venv/bin/python -m src.cli start \
  --host 127.0.0.1 \
  --port 8765 \
  --data-dir ./data \
  --extraction-model anthropic/claude-sonnet-4-6 \
  --inference-model anthropic/claude-sonnet-4-6
```

All flags:

```
--host ADDR              bind address (default 127.0.0.1)
--port PORT              port (default 8765)
--data-dir DIR           data directory (default ./data)
--model MODEL            shortcut for all stages
--extraction-model       override extraction
--inference-model        override inference
--judge-model            override judge (eval / MAL)
--embed-model            override embeddings (default text-embedding-3-large)
--server-token TOKEN     transport token; auto-generated if absent and
                         saved to ~/.gosh-memory/token (mode 0600)
--tls                    enable HTTPS, auto-generate self-signed cert
--tls-certfile PATH      custom certificate (implies --tls)
--tls-keyfile PATH       custom private key (implies --tls)
--advertise-host ADDR    host to embed in join token (default --host)
```

Env-var equivalents and cross-cutting config:

```
GOSH_MEMORY_TOKEN              alternative to --server-token (transport)
GOSH_MEMORY_ADMIN_TOKEN        bootstrap token (one-time admin pairing)
GOSH_MEMORY_ENCRYPTION_KEY     32-byte hex; encrypts authority.db at rest
```

The server prints:

```
gosh.memory MCP Server
  Listening: http://127.0.0.1:8765
  POST /mcp     — tool calls
  GET  /mcp/sse — Courier SSE
  Token: <auto-generated transport token>
  Saved to: ~/.gosh-memory/token
```

When `--tls` is enabled, it also prints a join token for remote agents:

```
gosh-agent --join gosh_join_eyJ1cmwiOi...
```

### 2.4 Bootstrap the first admin

```bash
# Pre-set a bootstrap token (or memory generates one):
export GOSH_MEMORY_ADMIN_TOKEN=$(openssl rand -hex 32)
# Then call auth_bootstrap_admin once via MCP. With gosh.cli this
# happens automatically on first `gosh memory start`.
```

### 2.5 Verify

```bash
curl http://localhost:8765/health
# {"status": "ok"}
```

MCP tool calls require an MCP session (`initialize` → tool call). Use
gosh.cli or any MCP client library; raw curl is awkward for the MCP
handshake.

---

## Mode 3: Connect Claude Code (or any MCP client) directly

If you have memory running standalone and want a coding CLI to talk to
it without `gosh-agent setup` doing the wiring:

```jsonc
// .mcp.json (project) or ~/.claude.json (user)
{
  "mcpServers": {
    "gosh-memory": {
      "type": "http",
      "url": "http://localhost:8765/mcp",
      "headers": {
        "x-server-token": "<contents of ~/.gosh-memory/token>",
        "Authorization": "Bearer <agent-kind principal token>"
      }
    }
  }
}
```

Two layers of auth, both required for the data plane:

- `x-server-token` — perimeter (server token). Sufficient only for `/health` and the MCP handshake.
- `Authorization: Bearer <token>` — principal token. Required for all data-plane tools.

For TLS / remote memory, swap `http://` for `https://` and embed the
issued join token's URL.

For OpenAI Responses and Anthropic MCP connector, use the same URL +
header pattern in their respective `tools[].headers` blocks. Public
HTTPS is required for both; use `--tls --advertise-host
your-host.example.com` when starting memory.

---

## Mode 4: Remote memory + local agent

A common production shape: memory runs on a server, agents run on
operator machines or build hosts.

### Server side

```bash
gosh memory setup local --runtime docker --public-url https://memory.example.com:8765
gosh memory start

# export an admin bundle for the operator(s)
gosh memory setup remote export --file admin-bundle.json
# JSON with { url, admin_token, server_token, tls_ca, schema_version }
```

> **TLS termination.** `gosh memory setup local` itself has no `--tls`
> flag — terminate TLS at a reverse proxy (Caddy, nginx, Traefik) and
> point `--public-url` at the proxy's HTTPS endpoint. The `--tls`
> flags described in Mode 2 below apply only to the bare
> `python -m src.cli start` invocation, where the Python server
> generates a self-signed cert directly.

### Operator side

```bash
gosh memory setup remote import --file admin-bundle.json --name prod
gosh memory data stats --instance prod --key default   # verify
```

### Provisioning a remote agent

On the memory server (admin context):

```bash
gosh agent create remote-coder --memory prod --swarm quickstart-swarm
gosh agent bootstrap export --instance remote-coder --file remote-coder.bootstrap.json
```

Move `remote-coder.bootstrap.json` to the agent host (it is mode 0600
and contains the principal token + X25519 private key — treat it as a
secret).

On the agent host:

```bash
gosh agent import remote-coder.bootstrap.json
gosh agent setup --instance remote-coder
gosh agent start --instance remote-coder --watch --watch-key default --watch-swarm-id quickstart-swarm
```

### Rotating credentials

```bash
gosh agent bootstrap rotate --instance <name>
```

Reissues the principal token, regenerates the X25519 keypair, rebuilds
the join token, writes them to the keychain. If the agent is currently
running, it is stopped, the bootstrap-temp file is rewritten, and the
agent is restarted with the same watch parameters.

---

## Agent profiles and the SHELL budget

`gosh-agent` does not carry a hardcoded profile list. The model, secret
reference, and pricing for each call are returned by memory's
`memory_plan_inference` tool. The agent extracts:

- `payload` — model id (e.g. `anthropic/claude-opus-4-6`)
- `secret_ref` — pointer to the secret to decrypt for the API call
- `payload_meta.local_cli` — when present, run a local coding CLI instead

The SHELL budget (1 SHELL = $0.01 at the profile's max rate) is enforced
per-task. Twenty percent is reserved for the review phase by default
(`review_budget_reserve = 0.2` in the agent's `AgentConfig`).

Concurrency: each agent process tracks `in_flight_tasks` in memory and
caps itself at `max_parallel_tasks` (default 4). There is **no** global
600-second cooldown across CLIs in the current implementation; the
memory side enforces single-flight on the data plane through
`memory_write` semantics.

---

## What gosh.cli is doing behind the scenes

If you need to integrate without `gosh.cli`, the dance is:

1. Start memory; export `GOSH_MEMORY_ADMIN_TOKEN` and
   `GOSH_MEMORY_ENCRYPTION_KEY`.
2. Call `auth_bootstrap_admin` to mint the first admin token.
3. Call `principal_create` for each agent identity.
4. Call `auth_token_issue --kind agent` to get an agent token.
5. Call `swarm_create` and `membership_grant` to set up groups.
6. Call `memory_set_config` and `memory_store_secret` to wire models
   and provider keys.
7. Build a join token JSON for each agent and ship it to the agent host.
8. On the agent host, run `gosh-agent serve --bootstrap-file <PATH>` (or
   pass `--join <TOKEN>` with `--allow-insecure-inline-join`).

See [PERMISSIONS-AND-ACL.md](PERMISSIONS-AND-ACL.md) for the principal /
token model in detail and [MEMORY-SYSTEM.md](MEMORY-SYSTEM.md) for the
fact and retrieval model.

---

## Inspecting and debugging

### Recall telemetry

```bash
gosh memory data recall --key <KEY> --swarm <SWARM> --json "<query>"
```

Key fields (full list in [TELEMETRY-CONTRACT.md](TELEMETRY-CONTRACT.md)):

- `retrieved_count` — facts found
- `query_type` — detected type (lookup, temporal, counting, …)
- `retrieval_families` — which retrieval paths fired
- `search_family` — `conversation`, `document`, `codebase`, or `auto`
- `complexity_hint.score` / `level` / `dominant`
- `recommended_profile` — what memory recommends for this query
- `payload_meta.budget_exceeded` — was context too large

### Ask telemetry

```bash
gosh memory data ask --key <KEY> --swarm <SWARM> --json "<question>"
```

- `profile_used` vs `recommended_profile`
- `profile_fallback` — was a fallback profile used
- `tool_called` — was a tool invoked (`date_diff`, `count_items`,
  `get_more_context`)
- `budget_exceeded`, `estimated_cost`

### Store health

```bash
gosh memory data stats --key <KEY>
```

- `granular`, `consolidated`, `cross_session` — fact tier counts
- `index_built` — vector index ready?
- `all_raw_sessions_active` — any stuck sessions?
- `process_cost_summary` — LLM cost breakdown for the current process
  (resets on memory restart; not per-key)

### Agent task

```bash
gosh agent task status --instance <NAME> <TASK_ID> --key <KEY> --json
```

- `status`, `phase`, `iteration`
- `shell_spent`, `profile_used`, `backend_used`
- `session_fact`, `result_fact`, `deliverable_fact` — agent-side wrappers
  around the persisted fact kinds `task_session`, `task_result`, and
  `task_deliverable` respectively (full id + kind + fact + metadata)

### Common failures

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| "data commands ... require an agent token" | Keychain has no agent token | `gosh memory auth provision-cli` |
| "scope must be provided explicitly" | Missing `--scope` on store/ingest | Pass `--scope agent-private\|swarm-shared\|system-wide` |
| `index_built: false` after store | First store, index not built yet | `gosh memory data build-index --key <KEY>` |
| `granular > 0` but `consolidated = 0` | Consolidation not yet run | `gosh memory data flush --key <KEY>` |
| Watch mode does not see tasks | `watch-key` mismatch with task key | Verify `gosh agent status` shows the right `watch_key` |
| `BOOTSTRAP_ALREADY_USED` | Bootstrap token consumed | Use the saved admin token (already in keychain) |
| Health check fails | Memory not running or wrong port | `gosh memory status`; check `~/.gosh/run/memory_<name>.log` |

For deeper inspection, the harness also produces a per-case
`telemetry.json` matching the shape in
[TELEMETRY-CONTRACT.md](TELEMETRY-CONTRACT.md).

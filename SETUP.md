# Setup Guide

How to install and run GOSH Memory in different configurations.

---

## Prerequisites

All modes require:
- Python 3.10+
- Git

For gosh.cli orchestration:
- Rust 1.86+ (for building gosh.cli and gosh.agent)

---

## Mode 1: Harness — full stack via gosh.cli

The recommended production setup. gosh.cli orchestrates memory and agent services,
manages secrets, and provides a unified CLI.

### 1.1 Clone repos

```bash
mkdir gosh && cd gosh
git clone https://github.com/Futurizt/gosh.cli.git
git clone https://github.com/Futurizt/gosh.memory.git
git clone https://github.com/Futurizt/gosh.agent.git
```

All three must be sibling directories.

### 1.2 Build CLI

```bash
cd gosh.cli
cargo build --release
export PATH="$PWD/target/release:$PATH"
```

### 1.3 Build Agent

```bash
cd ../gosh.agent
cargo build --release
```

The agent binary will be at `gosh.agent/target/release/gosh-agent`.
Set the path in `services.toml` after initialization (step 1.4).

### 1.4 Initialize

```bash
cd ../gosh.cli
gosh init          # creates services.toml from template
```

### 1.5 Configure paths

Edit `services.toml` and set absolute paths:

- `services.memory.path` — path to `gosh.memory` directory
- `services.alpha.binary` — path to agent binary (e.g. `/absolute/path/to/gosh.agent/target/release/gosh-agent`)

Then verify:

```bash
gosh doctor        # verify paths and ports
```

### 1.6 Set secrets

```bash
gosh secret set ANTHROPIC_API_KEY sk-ant-...
gosh secret set GROQ_API_KEY gsk_...
gosh secret set OPENAI_API_KEY sk-...
# Optional:
gosh secret set GOOGLE_API_KEY ...
```

Or export as environment variables — CLI falls back to env when a key is not in secrets.json.

### 1.7 Start services

```bash
gosh start         # starts memory first, then agent (dependency order)
gosh status        # verify both running
```

### 1.8 Configure memory models

Models must be configured explicitly. There are no implicit defaults for production.

```bash
cat > config.json << 'EOF'
{
  "schema_version": 1,
  "embedding_model": "text-embedding-3-large",
  "librarian_profile": "extraction",
  "profiles": {
    "1": "fast",
    "2": "balanced",
    "3": "strong",
    "4": "max",
    "5": "max"
  },
  "profile_configs": {
    "fast": {
      "model": "claude-haiku-4-5-20251001",
      "context_window": 200000,
      "max_output_tokens": 8192
    },
    "balanced": {
      "model": "claude-sonnet-4-6",
      "context_window": 200000,
      "max_output_tokens": 8192
    },
    "strong": {
      "model": "claude-opus-4-6",
      "context_window": 200000,
      "max_output_tokens": 8192
    },
    "max": {
      "model": "claude-opus-4-6",
      "context_window": 200000,
      "max_output_tokens": 8192
    },
    "extraction": {
      "model": "claude-sonnet-4-6",
      "context_window": 200000,
      "max_output_tokens": 8192
    }
  },
  "retrieval": {
    "default_token_budget": 4000,
    "search_family": "auto"
  }
}
EOF

gosh memory config set --file config.json --key my-namespace \
  --agent-id alpha --swarm-id swarm_alpha
gosh memory config get --key my-namespace \
  --agent-id alpha --swarm-id swarm_alpha --json   # verify
```

### 1.9 Extraction / inference / judge models

These are set via CLI args or services.toml:

```toml
[services.memory]
args = [
    "--extraction-model", "claude-sonnet-4-6",
    "--inference-model", "claude-sonnet-4-6",
]
```

Or via environment variables:
- `GOSH_EXTRACTION_MODEL`
- `GOSH_INFERENCE_MODEL`
- `GOSH_JUDGE_MODEL`
- `GOSH_EMBED_MODEL`

### 1.10 Agent config

Agent config is stored in memory as a `kind: agent_config` fact. The agent loads it at runtime -- there is no static TOML config for the agent itself.

Key fields:
- `execution_mode`: `context_only` (agent makes LLM calls) or `memory_payload` (memory provides ready payload)
- `fast_profile`, `balanced_profile`, `strong_profile`, `review_profile`, `extraction_profile`
- `max_parallel_tasks`, `global_cooldown_secs`

#### Builtin model profiles

The agent ships with 6 builtin profiles:

| Profile | Model | Backend |
|---------|-------|---------|
| `anthropic_haiku_api` | claude-haiku-4-5-20251001 | anthropic_api |
| `anthropic_sonnet_api` | claude-sonnet-4-6 | anthropic_api |
| `anthropic_opus_api` | claude-opus-4-6 | anthropic_api |
| `claude_code_cli` | claude-code | claude_cli |
| `codex_cli` | codex-cli | codex_cli |
| `gemini_cli` | gemini-cli | gemini_cli |

Each call tracks cost against the SHELL budget using the profile's `cost_per_1k` rate.

### 1.11 Important: no implicit defaults

- Memory extraction/inference models: must be set explicitly
- Agent profiles: must reference models defined in memory config
- API keys: must be provided via secrets or env vars
- Embedding model: defaults to `text-embedding-3-large` but should be set explicitly for production

### 1.12 First store and recall

```bash
# Store content (triggers extraction)
gosh memory store --key my-namespace \
  --agent-id alpha --swarm-id swarm_alpha \
  --scope swarm-shared \
  "Alice is a senior engineer at ACME Corp. She joined in 2024 and leads the platform team."

# Build vector index
gosh memory build-index --key my-namespace \
  --agent-id alpha

# Recall
gosh memory recall --key my-namespace \
  --agent-id alpha --swarm-id swarm_alpha \
  "Who is Alice?"

# Ask (recall + inference)
gosh memory ask --key my-namespace \
  --agent-id alpha --swarm-id swarm_alpha \
  "What does Alice do?"
```

### 1.13 With agent

```bash
# Create task
TASK_ID=$(gosh agent alpha task create \
  --extract memory \
  --key my-namespace \
  --swarm-id swarm_alpha \
  "Summarize what we know about Alice")

# Run task
gosh agent alpha task run $TASK_ID --key my-namespace --swarm-id swarm_alpha --budget 10

# Check status
gosh agent alpha task status $TASK_ID --key my-namespace --swarm-id swarm_alpha --json
```

> **Important:** The agent's `--watch-key` must match the `--key` used when creating tasks.
> Without `--watch-key`, the agent defaults to key `default` and won't see tasks in other namespaces.
>
> Uncomment `--watch-key` in `services.toml` and set it to your namespace:

```toml
[services.alpha]
type = "agent"
binary = "/path/to/gosh-agent"
port = 8767
args = [
    "--memory-url", "http://127.0.0.1:8765/mcp",
    "--memory-token", "${MEMORY_SERVER_TOKEN}",
    "--watch",
    "--watch-key", "my-namespace",
    "--watch-agent-id", "alpha",
    "--watch-swarm-id", "swarm_alpha",
]
depends_on = ["memory"]
health_endpoint = "/health"

[services.alpha.envs]
# configure ANTHROPIC_API_KEY in your local env
```

> Then restart the agent:

```bash
gosh restart alpha

# Create task
TASK_ID=$(gosh agent alpha task create \
  --extract memory \
  --key my-namespace \
  --swarm-id swarm_alpha \
  "Summarize what we know about Alice")

# Check status
gosh agent alpha task status $TASK_ID --key my-namespace --swarm-id swarm_alpha --json
```

### 1.14 Without agent (memory only)

Edit `services.toml` to remove or comment out the `[services.agent]` section.
`gosh start` will only start memory.

All memory commands (`store`, `recall`, `ask`, `list`, `stats`, `config`) work
without an agent.

### 1.15 Stop

```bash
gosh stop          # stops agent first, then memory
```

---

## Mode 2: Standalone — MCP server directly

Run gosh.memory as a standalone MCP server without gosh.cli or gosh.agent.
Useful for integration with external tools and AI assistants.

### 2.1 Install

```bash
git clone https://github.com/Futurizt/gosh.memory.git
cd gosh.memory
python3 -m venv .venv
.venv/bin/pip install -e . -r requirements.txt
```

For local embeddings (no OpenAI dependency):
```bash
.venv/bin/pip install -e ".[local-embed]"
```

For TLS support:
```bash
.venv/bin/pip install cryptography
```

### 2.2 Configure provider

```bash
.venv/bin/python -m src.cli setup --provider anthropic --api-key sk-ant-...
```

Or set environment variables:
```bash
export ANTHROPIC_API_KEY=<your-api-key>
# Or for Groq-hosted models:
export GROQ_API_KEY=gsk_...
# For embeddings (default --embed-model is text-embedding-3-large, which requires OpenAI):
export OPENAI_API_KEY=sk-...
```

### 2.3 Start server

```bash
.venv/bin/python -m src.mcp_server \
  --host 127.0.0.1 \
  --port 8765 \
  --data-dir ./data \
  --extraction-model claude-sonnet-4-6 \
  --inference-model claude-sonnet-4-6
```

Server prints:
```
gosh.memory MCP Server
  Listening: http://127.0.0.1:8765
  POST /mcp     — tool calls
  GET  /mcp/sse — Courier SSE
  Token: <auto-generated>
  Saved to: ~/.gosh-memory/token
```

Note the token — you need it for client connections.

### 2.4 With TLS (for remote access)

```bash
.venv/bin/python -m src.mcp_server \
  --host 0.0.0.0 \
  --port 8765 \
  --tls \
  --advertise-host your-server.example.com \
  --data-dir ./data \
  --server-token YOUR_SECRET_TOKEN
```

Server prints a join token for remote agents:
```
gosh-agent --join <join-token>...
```

### 2.5 Verify

```bash
# Health check (no auth needed)
curl http://localhost:8765/health
```

MCP tool calls require a session (initialize → tool call). Use `gosh.cli` or an MCP client library instead of raw curl.

---

## Mode 3: Connect to Claude Code / Claude Desktop

### 3.1 Start memory server (Mode 2)

Follow Mode 2 steps 2.1-2.3 to get the server running.

### 3.2 Get the token

```bash
cat ~/.gosh-memory/token
```

### 3.3 Add to Claude Code

**Option A: CLI**
```bash
claude mcp add gosh-memory \
  --transport http \
  http://localhost:8765/mcp
```

**Option B: Project config (`.mcp.json` in project root)**
```json
{
  "mcpServers": {
    "gosh-memory": {
      "type": "http",
      "url": "http://localhost:8765/mcp",
      "headers": {
        "x-server-token": "<token from ~/.gosh-memory/token>"
      }
    }
  }
}
```

**Option C: User config (`~/.claude.json`)**
Same format as Option B, applies to all projects.

### 3.4 Verify in Claude Code

Ask Claude: "Use the memory_stats tool with key 'test'"

Claude should call the tool and return memory statistics.

### 3.5 With TLS (remote server)

```json
{
  "mcpServers": {
    "gosh-memory": {
      "type": "http",
      "url": "https://your-server:8765/mcp",
      "headers": {
        "x-server-token": "<token>"
      }
    }
  }
}
```

---

## Mode 4: Connect to OpenAI Agents / Responses API

OpenAI supports MCP servers as tool providers in the Agents API.

### 4.1 Start memory server (Mode 2)

The server must be accessible from OpenAI's infrastructure (public URL or tunnel).

### 4.2 Configure in OpenAI

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-4.1",
    tools=[{
        "type": "mcp",
        "server_label": "gosh-memory",
        "server_url": "https://your-server:8765/mcp",
        "headers": {
            "x-server-token": "<token>"
        },
        "require_approval": "never",
    }],
    input="Use memory_recall with key 'default' to search for 'Alice'"
)
```

### 4.3 Notes

- Server must be publicly accessible (HTTPS required by OpenAI)
- Use `--tls` and `--advertise-host` when starting the server
- OpenAI will discover available tools via MCP protocol negotiation
- All memory tools (store, recall, ask, list, stats, config) are available

---

## Mode 5: Connect to Anthropic API (tool use)

Anthropic's API supports MCP tool servers directly.

### 5.1 Start memory server (Mode 2)

### 5.2 Configure in Anthropic SDK

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    mcp_servers=[{
        "type": "url",
        "url": "https://your-server:8765/mcp",
        "name": "gosh-memory",
        "authorization_token": "<token>",
    }],
    messages=[{
        "role": "user",
        "content": "Use memory_recall to search for Alice in key 'default'"
    }]
)
```

### 5.3 Notes

- Same requirements as OpenAI: public HTTPS endpoint
- The Anthropic MCP connector uses `mcp_servers` to declare servers, with optional per-server `tool_configuration`
- The connector API is evolving — check [Anthropic MCP connector docs](https://docs.anthropic.com/en/docs/agents-and-tools/mcp-connector) for the latest format
- All memory tools available via MCP protocol negotiation

---

## Mode 6: Connect to Google Gemini

### 6.1 Start memory server (Mode 2)

### 6.2 Configure in Google AI SDK

Google Gemini supports MCP servers through the SDK's tool integration.
The API is evolving — check [Gemini function calling docs](https://ai.google.dev/gemini-api/docs/function-calling) for the current MCP integration format.

The general pattern uses the SDK's MCP session/tool conversion:

```python
from google import genai
from google.genai import types

client = genai.Client()

# Connect to MCP server and convert tools
# (exact API depends on SDK version — see official docs)
```

### 6.3 Notes

- Requires public HTTPS endpoint
- Gemini MCP integration API is evolving — check official docs for the latest SDK format
- All memory tools available via MCP protocol negotiation

---

## Mode 7: Connect to any MCP-compatible client

gosh.memory speaks standard MCP over HTTP. Any client that supports
MCP streamable HTTP transport can connect.

### Connection details

| Setting | Value |
|---------|-------|
| Transport | HTTP (streamable) |
| Endpoint | `http(s)://HOST:PORT/mcp` |
| Auth header | `x-server-token: <token>` |
| Health check | `GET /health` (no auth) |
| SSE (Courier) | `GET /mcp/sse` |

### Available MCP tools

After connection, the client discovers these tools via MCP initialization:

| Tool | Description |
|------|-------------|
| `memory_store` | Store content, triggers extraction |
| `memory_recall` | Semantic search with complexity hints |
| `memory_ask` | Recall + LLM inference |
| `memory_list` | List facts with ACL filtering |
| `memory_get` | Get single fact by ID |
| `memory_stats` | Store health and telemetry |
| `memory_build_index` | Build/rebuild vector index |
| `memory_set_config` | Set canonical memory config |
| `memory_get_config` | Get current memory config |
| `memory_query` | Structured fact query with filters |
| `memory_ingest_asserted_facts` | Ingest pre-extracted facts |
| `memory_ingest_document` | Ingest a document |
| `memory_import` | Import from text/git/directory |
| `memory_flush` | Rebuild tiers without re-embedding |
| `memory_reextract` | Re-run extraction on raw sessions |

---

## First Steps After Setup (any mode)

### Store your first data

```
memory_store(key="demo", content="Alice is a senior engineer at ACME Corp.", session_num=1, session_date="2026-03-29")
```

### Build the index

```
memory_build_index(key="demo")
```

This embeds all facts and builds retrieval indices. Required before recall.

### Query

```
memory_recall(key="demo", query="Who is Alice?")
```

Returns: context with retrieved facts, complexity hint, query type classification.

```
memory_ask(key="demo", query="What does Alice do?")
```

Returns: LLM-generated answer based on retrieved facts.

### Check health

```
memory_stats(key="demo")
```

Returns: fact counts per tier, index status, session health, process costs.

---

## Troubleshooting

### "No granular facts" on recall
Index not built. Run `memory_build_index` after storing data.

### "secret not found: ANTHROPIC_API_KEY"
Set the key: `gosh secret set ANTHROPIC_API_KEY sk-ant-...` or export as env var.

### Health check fails
Check if server is running: `curl http://localhost:8765/health`
Check logs: `gosh logs memory` (harness mode) or server stdout (standalone).

### Token auth fails
Get the correct token: `cat ~/.gosh-memory/token`
Or set a known token at startup: `--server-token mytoken`

### TLS certificate errors
For self-signed certs, clients must either:
- Use the join token (contains CA cert for pinning)
- Accept invalid certs (development only)
- Provide a proper CA-signed certificate via `--tls-certfile`

---

## Inspecting and Debugging

### Inspecting a retrieval

```bash
# Full recall telemetry
gosh memory recall --key my-namespace --json "What is Project Alpha?"
```

Key fields to check:
- `retrieved_count` > 0 (facts were found)
- `query_type` -- was the query classified correctly?
- `retrieval_families` -- which retrieval paths fired?
- `search_family` -- conversation vs document routing
- `recommended_prompt_type` -- what prompt style was selected?
- `use_tool` -- should tools be used?
- `complexity_hint.score` -- how complex is this query?
- `recommended_profile` -- which model tier was recommended?
- `payload_meta.truncated` -- was context truncated?

### Inspecting routing and tool use

```bash
# Full ask telemetry
gosh memory ask --key my-namespace --json "Summarize Project Alpha"
```

Key fields to check:
- `profile_used` vs `recommended_profile` -- did the system use the recommended model?
- `profile_fallback` -- was a fallback needed?
- `tool_called` -- did inference use tools?
- `budget_exceeded` -- was the token budget hit?
- `estimated_cost` -- what did this cost?

### Inspecting store validity

```bash
# Memory stats
gosh memory stats --key my-namespace
```

Key fields to check:
- `granular` > 0 -- facts extracted successfully?
- `consolidated` > 0 -- per-session summaries built?
- `cross_session` > 0 -- per-entity synthesis built?
- `index_built` = true -- is the vector index ready?
- `all_raw_sessions_active` = true -- any stuck sessions?
- `raw_session_status_counts` -- breakdown of session states
- `process_cost_summary` -- LLM costs (note: process-scoped, resets on restart)

### Inspecting an agent run

```bash
# Agent task status with full telemetry
gosh agent alpha task status task-abc123 --key my-namespace --json
```

The `--json` flag returns structured telemetry (see [TELEMETRY-CONTRACT.md](TELEMETRY-CONTRACT.md) for schema). Without it, output is human-readable summary.

Key fields to check:
- `telemetry_version` -- schema version (currently 1)
- `status` -- done, failed, pending?
- `phase` -- bootstrap, execution, review
- `iteration` -- how many execution iterations ran?
- `started_at` / `finished_at` -- ISO 8601 timestamps for the task run
- `shell_spent` > 0 -- did the agent actually run?
- `profile_used` -- which model profile was used?
- `backend_used` -- `anthropic_api`, `claude_cli`, `codex_cli`, or `gemini_cli`
- `result_fact` -- is there a result?
- `session_fact` -- execution trace

### Inspecting harness artifacts

Per-case `telemetry.json` contains:

```bash
cat harness-output/case-001/telemetry.json | jq '.validity'
# -> "valid" or "invalid" or "partial"

cat harness-output/case-001/telemetry.json | jq '.recall.retrieved_count'
# -> number of facts retrieved

cat harness-output/case-001/telemetry.json | jq '.ask.profile_used'
# -> which model was used

cat harness-output/case-001/telemetry.json | jq '.judge'
# -> judge verdict and reasoning
```

### Failure classification

**Invalid run:**
- `validity: "invalid"` in telemetry
- Caused by: missing config, missing API keys, service not running

**Broken store:**
- `gosh memory stats` shows `index_built: false` or `all_raw_sessions_active: false`
- Fix: `gosh memory build-index`, check `raw_session_status_counts`

**Partial ingest:**
- `granular` = 0 despite data being stored -- extraction may have failed
- `consolidated` = 0 with `granular` > 0 -- consolidation not yet run
- `cross_session` = 0 with `granular` > 0 -- cross-session synthesis not yet run
- Fix missing tiers: `gosh memory build-index --key my-namespace` (rebuilds if dirty) or `gosh memory flush --key my-namespace` (forces consolidation + cross-session)
- If extraction itself failed, check logs and re-run: `gosh memory reextract --key my-namespace`

**Wrong family routing:**
- `search_family` in recall telemetry doesn't match expected (conversation vs document)
- Check `retrieval_families` for which paths fired
- May need to adjust `retrieval.search_family` in memory config

**Retrieval miss:**
- `retrieved_count` = 0 or very low
- Check `query_type` -- was the query misclassified?
- Check `coverage_pct` -- is enough data in the index?
- Check `selection_scores` for score distribution

**Packet miss:**
- `payload_meta.truncated` = true
- Context was too large for the selected profile's context window
- Increase `default_token_budget` or use a larger model

**Answer/judge miss:**
- `ask` telemetry shows answer but `judge.verdict` is negative
- Check `profile_used` -- was the right model used?
- Check `tool_called` -- were tools available when needed?
- Check `budget_exceeded` -- did the model run out of tokens?

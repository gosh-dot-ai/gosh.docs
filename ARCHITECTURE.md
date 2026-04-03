# GOSH Architecture

Production architecture overview for the GOSH AI system.

For installation and setup, see [SETUP.md](SETUP.md).

## Components

### gosh.memory (Python)
Semantic long-term memory for AI agents. MCP server exposing store, recall, ask, and management tools.

See [MEMORY-SYSTEM.md](MEMORY-SYSTEM.md) for the full system description and [MEMORY-ADAPTATION-LOOP.md](MEMORY-ADAPTATION-LOOP.md) for the autoresearch adaptation system.

- Format-aware semantic extraction (conversation, document, code trace, narrative)
- Query-adaptive retrieval with RRF fusion
- Complexity-driven model selection (three-axis scoring)
- Canonical memory config system (model profiles, retrieval knobs)
- Instance-level ACL with owner/swarm/admin scoping

### gosh.agent (Rust)
Autonomous task executor exposed via MCP. Receives tasks, retrieves context from memory, selects models via complexity routing, executes with budget control, writes results back.

- 3-phase execution: bootstrap → execution → review
- Budget controller (SHELL currency, 1 SHELL = $0.01 at max model rate)
- Config loaded from memory at runtime (profiles, concurrency, cooldown)
- 6 builtin model profiles (3 API + 3 CLI) with per-call cost tracking
- Watch mode with dispatch deduplication
- Structured task_result and task_session artifacts

### gosh.cli (Rust)
CLI orchestrator. Manages service lifecycle, secrets, and provides user-facing commands for memory and agent operations.

- Service management: start/stop/status/doctor
- Secret store with env var fallback (secrets.json -> env -> error)
- Memory commands: store, recall, ask, list, stats, config get/set, import, ingest
- Agent commands: start, stop, task create/run/status/list
- Cross-process registry locking (flock)

### gosh.docs
Canonical location for cross-repo system documentation. Component-local docs live in each repo's README.

## Data Flow

```
User / Operator
    |
    v
gosh.cli ──────────────────────────────────┐
    |  start/stop services                  |  task create/run/status
    |  secret management                    |  memory store/recall/ask
    v                                       v
gosh.memory (MCP :8765)            gosh.agent (MCP :8767)
    |                                       |
    |  store/recall/ask/config              |  agent_start/agent_status
    |  semantic extraction                  |  recall -> LLM -> tools -> write-back
    |  complexity hints                     |  budget tracking
    |  payload building                     |
    └───────────── memory MCP ──────────────┘
```

## Configuration

Models must be configured explicitly. There are no implicit production defaults.

See [SETUP.md](SETUP.md) for details.

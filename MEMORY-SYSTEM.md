# Memory System

How gosh.memory works: storage, retrieval, inference, and the decisions an agent
or operator needs to understand.

---

## Two Ways to Query Memory

An agent (or operator) can query memory via two MCP tools:

### `memory_recall` — retrieval without inference

Returns context, metadata, and complexity signals. No LLM call.

```
query → detect_query_type → retrieve → route → build context → complexity hint → payload (optional)
```

Use when: the caller makes its own LLM call (e.g., gosh.agent execution loop).

### `memory_ask` — retrieval + inference

Calls `recall()` internally, then runs LLM inference on the retrieved context.
Returns an answer.

```
query → recall() → select model via profiles → LLM call → tool use (optional) → answer
```

Use when: the caller wants a direct answer (e.g., operator inspection, simple Q&A).

### Key difference

| | `recall` | `ask` |
|-|----------|-------|
| LLM call | No | Yes |
| Returns | Context + metadata | Answer + metadata |
| Cost | $0 (retrieval only) | $0.02-0.20 (inference) |
| Caller decides model | Yes (via payload) | No (memory selects via profiles) |

---

## Semantic Extraction (Librarian)

The core differentiator. When content is stored, an LLM extracts structured
atomic facts — not chunks, not embeddings of raw text. Each fact is a
self-contained statement with entities, temporal links, dependencies, and
semantic metadata.

This extraction is **format-aware**: different content types get specialized
prompts that understand the structure of the input.

| Format | Detection | Strategy |
|--------|-----------|----------|
| Conversation | Speaker turns, Q&A patterns | Block segmentation → per-block extraction |
| Document | No speaker turns, structural markers | Document block segmentation |
| Code trace | Code blocks, stack traces | Single-pass domain-specific prompt |
| Narrative | Dense prose | Single-pass extraction |
| Fact list | Pre-formatted facts | Light parsing |

Each extracted fact carries:
- `fact` — atomic statement text
- `kind` — fact, preference, constraint, rule, decision, action_item, event
- `entities` — named entities mentioned
- `tags` — semantic tags
- `event_date` — when it happened (if temporal)
- `depends_on` — references to other facts
- `speaker` / `speaker_role` — who said it
- ACL fields (owner, read, write)
- Metadata and routing targets

This is not chunking. A single conversation turn might produce 5 atomic facts.
A 10-page document might produce 200+ facts with entity cross-references.

---

## Fact Organization

Extracted facts are organized into three levels, each built on the previous:

### Granular facts

The atomic facts produced by semantic extraction (above). Each is a single
self-contained statement with entities, temporal links, and metadata. These are
the base layer — every query ultimately retrieves from granular facts.

### Consolidated facts

Per-session summaries produced by the consolidation pass. After extraction,
the system can summarize a session's granular facts into higher-level
statements that capture the session's key points. These reduce noise during
retrieval for broad queries.

### Cross-session facts

Per-entity synthesis across all sessions. The system identifies entities that
appear in multiple sessions and produces synthesized facts that capture the
entity's full history. These are critical for questions like "What do we know
about X?" that span many sessions.

### How they are built

- **`build_index`** conditionally rebuilds consolidated and cross-session facts
  if dirty (i.e., new granular facts have been added since the last build),
  then embeds all facts for retrieval.
- **`flush`** runs consolidation and cross-session synthesis explicitly,
  returning real counts of facts produced at each level.
- **`memory_stats`** reports counts for all three levels: `granular`,
  `consolidated`, `cross_session`.

All three levels are embedded and available during retrieval. The retrieval
pipeline searches across all of them, using RRF fusion to combine signals.

---

## Query Type Detection

Before retrieval, the query is classified into a type. Each type triggers
different retrieval strategies and inference prompts.

| Type | Examples | Strategy |
|------|----------|----------|
| `lookup` | "What is X?" | Direct fact match |
| `temporal` | "When did X happen?" | Time-aware retrieval + date_diff tool |
| `counting` | "How many times did X?" | Exhaustive scan + count_items tool |
| `current` | "Where does X live now?" | Prefer latest facts, supersession-aware |
| `synthesis` | "What does user prefer?" | Multi-hop retrieval |
| `summarize` | "Write a summary" | Session round-robin |
| `icl` | In-context learning examples | Example-based retrieval |
| `rule` | "What's the policy on X?" | Prefer rule/constraint kind facts |
| `causal` | "Why did X happen?" | Dependency chain traversal |
| `prospective` | "What's the next step?" | Pending/upcoming facts |
| `default` | Everything else | Standard hybrid retrieval |

Detection is regex-based (fast, no LLM call). The detected type determines:
1. Which retrieval pipeline fires
2. Which tools are available during inference
3. Which inference prompt template is used

---

## Retrieval Pipeline

### Step 1: Episode retrieval (if episode corpus exists)

For documents and structured conversations:
- BM25 + vector search across episodes
- Family routing: conversation vs. document (or auto-detect)
- RRF fusion to combine signals

### Step 2: Structural augmentation

If episode results are insufficient:
- Entity matching within conversations
- Section-based matching for documents

### Step 3: Semantic fact sweep (fallback)

Full hybrid fact search:
- BM25 keyword search on fact text
- Vector similarity search on embeddings
- Named entity matching between query and facts
- RRF fusion across all signals

### Retrieval output

A ranked list of facts filtered by ACL, formatted into a context string.

---

## Complexity Hint and Model Selection

After retrieval, memory computes a complexity hint — a signal telling the
caller how hard this query is.

### Three axes

**Retrieval complexity** — how the retrieval behaved:
- Multi-hop (temporal, counting, current queries): +0.35
- Cross-scope (facts from multiple agents): +0.25
- Conflict detected (supersession): +0.20
- High fact count (>50 facts): +0.05

**Content complexity** — what the facts contain:
- Fact text length and entity density
- Structural complexity of the content
- Temporal relationship density

**Query complexity** — how complex the question itself is:
- Multi-constraint decisions (tradeoffs, "which option", "recommended")
- Strong constraint terms (latency, cost, budget, compliance)
- Decision-making markers

### Combined score

```
score = max(retrieval_complexity, content_complexity, query_complexity)
```

Score 0.0-1.0, mapped to levels 1-5.

The `dominant` field reports which axis drove the score: "retrieval", "content", "query", or "tie".

### Level → profile mapping

When profiles are configured in memory config:

| Level | Score range | Default profile name |
|-------|-------------|---------------------|
| 1 | 0.0-0.2 | fast |
| 2 | 0.2-0.4 | easy |
| 3 | 0.4-0.6 | balanced |
| 4 | 0.6-0.8 | complex |
| 5 | 0.8-1.0 | thorough |

Profile names and their backing models are configurable via `memory config set`.
There are no hardcoded production model defaults — profiles must be configured
explicitly.

The `recommended_profile` field in recall output tells the agent which model
tier memory recommends for this query.

---

## Prompt Routing

After query type detection and retrieval, memory selects a prompt strategy:

| Condition | Prompt type | Tools? |
|-----------|------------|--------|
| Query type = `summarize` | `summarize_with_metadata` | Yes |
| Query type = `icl` | `icl` | No |
| Low session coverage (<30%) with >20 sessions and retrieved facts | `tool` | Yes |
| Raw context present in hybrid packet | `hybrid` | No |
| Default | Same as query type | No |

This routing determines:
- Which inference prompt template is used
- Whether tools (date_diff, count_items, get_more_context) are available

---

## Payload Building

When profiles are configured, `recall()` builds an LLM-ready payload:

```json
{
  "model": "claude-sonnet-4-6",
  "messages": [{"role": "user", "content": "...context + query..."}],
  "max_tokens": 2048,
  "temperature": 0.3,
  "tools": [...]
}
```

The payload is provider-specific (Anthropic, OpenAI, Google formats).

### Truncation

If context exceeds the profile's context window, content is removed by priority
to fit within budget. If everything is exhausted: `payload_meta.budget_exceeded = true`.

### payload_meta

Accompanies the payload with metadata:

```json
{
  "profile_used": "balanced",
  "context_tokens": 1245,
  "message_tokens_est": 456,
  "tool_tokens_est": 120,
  "memory_budget": 2000,
  "budget_exceeded": false,
  "prompt_type": "temporal",
  "use_tool": true,
  "provider": "anthropic",
  "truncation": null
}
```

---

## Tool Use in Inference

For temporal, counting, and low-coverage queries, the LLM can call tools:

| Tool | Purpose | When |
|------|---------|------|
| `date_diff` | Exact date arithmetic | Temporal queries |
| `count_items` | Exact counting | Counting queries |
| `get_more_context` | Fetch full raw session text | Low coverage, need more detail |

Tool execution is local (no external calls). The LLM issues tool_use blocks,
memory executes them deterministically, and sends results back for up to 3 rounds.

---

## Store Pipeline

When content is stored via `memory_store` or `memory_ingest_document`:

1. **Dedup check** — content hash compared against existing sessions
2. **Raw session saved** — source of truth, written immediately
3. **Format detection** — conversation / document / code trace / narrative
4. **Semantic extraction** — format-specific LLM prompt produces atomic facts
5. **Tagging** — ACL (owner/read/write), metadata, target applied
6. **Episode registration** — for document families, episode corpus updated

---

## ACL and Visibility

Each fact has owner, read, and write ACLs:

```json
{
  "owner_id": "agent:alice",
  "read": ["agent:PUBLIC", "swarm:alpha"],
  "write": ["agent:alice"]
}
```

Visibility rules:
- Owner always has access
- `agent:PUBLIC` in read -- visible to everyone
- Swarm/group membership checked via membership registry
- Admin role bypasses all ACL
- Superseded/retracted facts are hidden
- Expired TTL facts are hidden

### Scope

The `scope` field on a fact controls its visibility tier:

| Scope | Visible to |
|-------|-----------|
| `agent-private` | Only the owning agent |
| `swarm-shared` | All agents in the same swarm |
| `system-wide` | All agents across all swarms |

### Swarm membership via swarm_id

When an agent passes `swarm_id="alpha"` on a memory call, it automatically
gets `swarm:alpha` in its memberships for that call. This means it can see
all `swarm-shared` facts that have `swarm:alpha` in their read ACL.

This is per-call -- the agent must pass `swarm_id` on every request to
maintain visibility.

### Persistent membership via membership_register

`membership_register` persists a membership so the agent does not need to
pass `swarm_id` on every call. Once registered, the membership is stored and
the agent sees swarm-scoped facts automatically.

Register also grants instance-level ACL access -- useful when you want an
agent to have persistent visibility without relying on per-call parameters.

### When to use register vs swarm_id parameter

| Approach | Persistence | Use when |
|----------|------------|----------|
| `swarm_id` parameter | Per-call only | Agent always knows its swarm, simple scripts |
| `membership_register` | Persistent | Long-running agents, watch mode, cross-session access |

With `swarm_id` parameter, the agent must remember to pass it on every
memory call. With `membership_register`, it works even without the parameter.

### Cross-key access

Each memory key is a separate namespace. Memberships are per-key -- an agent
registered in key A does not automatically have access to key B. Registration
must be done per key.

---

## Embedding and Index

`build_index()` must be called before the first `recall()`.

1. Conditionally rebuilds consolidated and cross-session facts if dirty
2. Embeds all facts (granular, consolidated, cross-session) using the configured embedding model
3. Builds BM25 keyword indices
4. Builds entity lookup indices
5. Caches embeddings by content fingerprint

Embedding cache is fingerprint-based: if fact IDs haven't changed, embeddings
are reused. Re-embedding is expensive and produces slightly different vectors —
never regenerate without reason.

---

## Cost Model

| Operation | Estimated cost | Duration |
|-----------|---------------|----------|
| `store()` 1 session | $0.02-0.05 | 2-5 sec |
| `ingest_document()` full PDF | $0.50-2.00 | 30-120 sec |
| `build_index()` cold (500 facts) | $0.10-0.30 | 10-30 sec |
| `recall()` | $0.00 | <1 sec |
| `ask()` with inference | $0.02-0.20 | 2-10 sec |

`process_cost_summary` in `memory_stats` tracks LLM costs during the current
process lifetime. It is process-scoped, not per-key — a process restart resets
these counters.

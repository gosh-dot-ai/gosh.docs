# Memory Adaptation Loop (MAL)

MAL is the autoresearch system that continuously improves gosh.memory
pipeline configuration using production feedback as signal. It adapts
retrieval parameters, grouping, and extraction prompts for each memory
namespace — without touching model weights, embeddings, or MCP tool
signatures.

Inspired by the autoresearch pattern (Karpathy, 2026), applied to a
memory pipeline.

## How it works

MAL operates on a simple loop:

1. **Collect feedback** — real production failures (user corrections,
   bad answers, agent verdicts) linked to canonical runtime traces
2. **Diagnose** — identify which pipeline stage broke first using the
   trace
3. **Propose** — generate one experiment atom (a single causal mutation)
4. **Evaluate** — test the atom against a frozen eval set
5. **Accept or reject** — if the primary metric improves and regression
   gates pass, apply the change; otherwise discard

Each iteration produces a versioned **tuning artifact** — a record of
what changed, why, and how to roll it back.

## Three modes

| Mode | What it tunes | Rollback |
|------|--------------|----------|
| `retrieval-only` | Selector parameters (weights, thresholds, top-K) | Instant config flip |
| `reprocessing` | Grouping parameters (how facts are organized) | Replay from raw sources |
| `extraction` | Extraction prompts (append-example only) | Re-extract from raw sessions |

`retrieval-only` is pure parameter search — fast, cheap, no data
replay. `reprocessing` and `extraction` mutate pipeline artifacts and
require replay to take effect.

## Tuning atoms in production

The implementation organizes mutations into **atom types**, each
consumed by a runtime stage:

| Atom | What it tunes |
|------|---------------|
| `lexical_signal_bundle` | Word/phrase/entity bonuses in retrieval |
| `locality_bundle` | Recency / current-state weighting |
| `window_bundle` | Supporting fact selection, token budget |
| `fusion_bundle` | RRF parameters, late-fusion settings |
| `inference_leaf_toggle` | Enable/disable specific inference prompt branches |
| `grouping_bundle` | Fact clustering thresholds, cluster size caps |
| `extraction_example_append` | Refinements appended to extraction prompts |

MAL never applies multiple atoms simultaneously. One variable per
experiment.

## Feedback-first design

The primary adaptation signal comes from real production failures, not
synthetic eval questions.

Allowed signal sources:

- Explicit user negative feedback
- User corrections ("the answer should be X, not Y")
- Agent post-answer verdict
- Operator-marked bad answers

Each feedback event links to the canonical runtime trace for that
answer, so MAL can diagnose which stage broke: retrieval miss, wrong
ranking, extraction gap, or inference failure.

Synthetic Q&A generation and frozen eval exist only as the
accept/reject gate for a candidate atom — not the primary signal source.

### Verdict and signal vocabulary

| Verdict | Meaning |
|---------|---------|
| `good_answer` | Query fully satisfied |
| `partial_answer` | Partial resolution |
| `bad_answer` | Wrong or missing |
| `hallucination_detected` | Model fabricated content |
| `missing_fact` | The fact exists but was not retrieved |
| `retrieval_gap` | Retrieval family was insufficient |

Signal source: `user`, `model`, or `system`.

## Autoresearch analogy

| autoresearch | MAL |
|---|---|
| Agent modifies `train.py` | Optimizer proposes one experiment atom |
| Fixed 5-minute training budget | Fixed `eval_top_k` frozen at run start |
| `val_bpb` — one scalar metric | `episode_hit_rate` — one primary metric |
| Keep or discard | Accept or discard |
| Accepted = new baseline | Accepted = auto-apply within budget |
| Iterate overnight | Iterate on schedule or trigger |

## Safety contract

MAL is **optional and disabled by default**.

- When disabled: no feedback capture, no adaptation runs, no mutations.
- Enabling MAL does not by itself change runtime behavior — it allows
  feedback capture and future runs.
- Disabling MAL stops future runs but does not roll back the current
  config.
- Rollback is always an explicit operator action via
  `memory_mal_rollback`.

If MAL is never enabled, the system behaves exactly like the non-MAL
runtime.

### Gates

| Gate | Default |
|------|---------|
| `min_signals` | ≥ 10 failures before proposing |
| Family support | `max(2, ceil(√N))` same-family failures before generalizing |
| A/B evaluation | Failure slice vs control group |
| Holdout split | Optional 70/30 to prevent overfitting |
| Convergence | 5 consecutive rejections halts further attempts |

## Metrics

**Primary metric**: `episode_hit_rate` — whether the correct episodes
(sources of ground truth) appear in the retrieval result.

**Regression gates**: secondary metrics that must not degrade when the
primary metric improves. A candidate atom that improves hit rate but
breaks temporal accuracy is rejected.

## Code-required path

When a failure is beyond what surface tuning can fix (e.g. a brand-new
extraction prompt is needed, or a new query executor), the MAL
scheduler returns the outcome `code_required` and emits a code-request
record:

```jsonc
{
  "task_type": "code_required",
  "agent_id": "coding",
  "analysis": "..."
}
```

The scheduler attempts to dispatch the request via courier when a
server is available; otherwise it persists the record and surfaces it
in `memory_mal_status`. If an `agent_id="coding"` exists in the swarm
and courier is reachable, the task is picked up and the fix is
implemented as code. This closes the loop from "the data tells us
something is wrong" to "the code is updated to handle it".

## MCP tools

| Tool | Purpose |
|------|---------|
| `memory_mal_configure` | Toggle on/off; set `auto_collect_feedback`, `auto_trigger`, `min_signals` |
| `memory_mal_feedback` | Submit a single failure signal with verdict + trace |
| `memory_mal_trigger` | Run the optimization loop; supports `estimate_only` and `force` |
| `memory_mal_status` | Current state, active artifacts, feedback counts |
| `memory_mal_list_feedback` | List feedback events filtered by status |
| `memory_mal_get_artifact` | Read a tuning artifact snapshot |
| `memory_mal_rollback` | Revert to a prior artifact |

## Artifact storage

Generated artifacts live under
`{data_dir}/mal/<key>/<agent_id>/`:

```
active_config.json          # currently active tuning
artifacts/<gen_id>/         # historical snapshots
feedback/<timestamp>        # feedback logs
```

## Current status

MAL is **implemented in production** as of memory v0.3.x. The 7 MCP
tools above are live; the optimization loop runs end-to-end with
retrieval-only, reprocessing, and extraction modes; the scheduler's
`code_required` outcome emits a code-request record targeted at
`agent_id="coding"` (delivered via courier when a server is
available).

It remains **disabled by default**. Operators opt in per namespace via
`memory_mal_configure`.

For benchmark results that exercise the production pipeline (the same
code path MAL adapts), see [BENCHMARKS.md](BENCHMARKS.md).

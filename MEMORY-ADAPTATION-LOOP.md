# Memory Adaptation Loop (MAL)

MAL is the autoresearch system that continuously improves gosh.memory pipeline
configuration using production feedback as signal. It adapts retrieval parameters,
grouping, and extraction prompts for each memory instance — without touching
model weights, embeddings, or MCP tool signatures.

Inspired by the autoresearch pattern (Karpathy, 2026), applied to a memory pipeline.

## How It Works

MAL operates on a simple loop:

1. **Collect feedback** — real production failures (user corrections, bad answers,
   agent verdicts) linked to canonical runtime traces
2. **Diagnose** — identify which pipeline stage broke first using the trace
3. **Propose** — generate one experiment atom (a single causal mutation)
4. **Evaluate** — test the atom against a frozen eval set
5. **Accept or reject** — if the primary metric improves and regression gates
   pass, apply the change; otherwise discard

Each iteration produces a versioned **tuning artifact** — a record of what
changed, why, and how to roll it back.

## Three Modes

| Mode | What it tunes | Rollback |
|------|--------------|----------|
| `retrieval-only` | Selector parameters (weights, thresholds, top-K) | Instant config flip |
| `reprocessing` | Grouping parameters (how facts are organized) | Replay from raw sources |
| `extraction` | Extraction prompts (append-example only) | Re-extract from raw sessions |

`retrieval-only` is pure parameter search — fast, cheap, no data replay needed.
`reprocessing` and `extraction` mutate pipeline artifacts and require replay
to take effect.

## The Experiment Atom

The central unit of MAL. One experiment atom is:
- One causally coherent mutation
- Proposed, evaluated, accepted or rejected as one indivisible unit
- Versioned with full rollback capability

Examples:
- Increase `entity_weight` from 0.3 to 0.5 (retrieval-only)
- Change grouping threshold for document blocks (reprocessing)
- Add one extraction example to the librarian prompt (extraction)

MAL never applies multiple atoms simultaneously. One variable per experiment.

## Feedback-First Design

The primary adaptation signal comes from real production failures, not
synthetic eval questions.

Allowed signal sources:
- Explicit user negative feedback
- User corrections ("the answer should be X, not Y")
- Agent post-answer verdict that the answer was wrong
- Operator-marked bad answers

Each feedback event links to the canonical runtime trace for that answer,
so MAL can diagnose which stage broke: retrieval miss, wrong ranking,
extraction gap, or inference failure.

Synthetic Q&A generation and frozen eval exist only as the accept/reject
gate for a candidate atom — they are not the primary signal source.

## Autoresearch Analogy

| autoresearch | MAL |
|---|---|
| Agent modifies `train.py` | Optimizer proposes one experiment atom |
| Fixed 5-minute training budget | Fixed `eval_top_k` frozen at run start |
| `val_bpb` — one scalar metric | `episode_hit_rate` — one primary metric |
| Keep or discard | Accept or discard |
| Accepted = new baseline | Accepted = auto-apply within budget |
| Iterate overnight | Iterate on schedule or trigger |

## Safety Contract

MAL is **optional and disabled by default**.

- When disabled: no feedback capture, no adaptation runs, no mutations
- Enabling MAL does not by itself change runtime behavior — it only allows
  feedback capture and future adaptation runs
- Disabling MAL stops future runs but does not roll back the current config
- Rollback is always an explicit operator action

If MAL is never enabled, the system behaves exactly like the non-MAL runtime.

## Metrics

**Primary metric**: `episode_hit_rate` — whether the correct episodes
(sources of ground truth) appear in the retrieval result.

**Regression gates**: secondary metrics that must not degrade when the
primary metric improves. A candidate atom that improves hit rate but
breaks temporal accuracy is rejected.

## Current Status

MAL is specified. The spec is at DRAFT v22. Implementation has not started —
there are no MAL-specific MCP tools, CLI commands, or runtime wiring in
the current codebase.

What exists today:
- The MAL spec (DRAFT v22) defines the full system
- The benchmark harness runs on the production pipeline (same code path MAL will adapt)

What does not exist yet:
- MAL runtime code
- `memory_adapt_feedback` / `memory_adapt_trigger` MCP tools
- Feedback capture wiring
- Automated diagnose → propose → evaluate → accept loop
- MAL-specific tests

Multibench tests exercise the production pipeline that MAL will adapt.

For benchmark results, see [BENCHMARKS.md](BENCHMARKS.md).

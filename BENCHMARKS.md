# Benchmarks

> Leader scores as of March 2026. Scores may change as new results are published.
> GOSH.AI results update frequently — benchmarks are on-going.
>
> We compare our Qwen 30B performance with comparable models (GPT-4o-mini class).

## Setup

- **Judge**: same prompt and model as most published references — GPT-4.1
- **Librarian (extraction)**: Qwen 30B via Groq API, or Mercury 2 via InceptionLabs API
- **Embeddings**: OpenAI text-embedding-3-large
- **Multibench mode**: all benchmarks run on the same production pipeline,
  harness, and code — most of the time in parallel

## Benchmark Suite

### gosh.memory — persistent agent memory system

| Benchmark | GOSH.AI | Current leader | Leader score |
|-----------|:-------:|---------------|-------------|
| AMA-Bench | **79.0% SOTA** | GPT-5.2 full-ctx | 72.3% |
| LoCoMo overall | **89.8% SOTA** | EverMemOS (GPT-4o-mini) | 86.76% |
| LoCoMo Cat 1 — single-hop | **90.0% SOTA** | Hippocampus (GPT-4o-mini) | 89.7% |
| LoCoMo Cat 2 — temporal | **100% SOTA** | Hippocampus (GPT-4o-mini) | ~88% |
| LoCoMo Cat 3 — multi-hop | 66.7% | Hippocampus (GPT-4o-mini) | 87.5% |
| LoCoMo Cat 4 — open-domain | **90.0% SOTA** | Zep (GPT-4o-mini) | 66.67% |
| LoCoMo Cat 5 — adversarial | **100% SOTA** | — | — |
| LongMemEval_S | IN PROGRESS... | Supermemory (GPT-4o) | 81.6% |
| Karnali T1 | IN PROGRESS... | no participants yet | — |
| MAB — Accurate Retrieval | IN PROGRESS... | dense RAG | ~70% |
| MAB — Test-Time Learning | IN PROGRESS... | long-context GPT-4 | ~55% |
| MAB — Long-Range Understanding | IN PROGRESS... | long-context GPT-4 | ~50% |
| MAB — Conflict Resolution | IN PROGRESS... | all competitors | ≤6% |
| RealMem | IN PROGRESS... | Graph Memory (NDCG) | 56.5% |
| MRCR 8-needle | IN PROGRESS... | Claude Opus 4.6 | 92% @ 256K |

### GOSH.AI Swarms — autonomous multi-agent runtime

| Benchmark | GOSH.AI | Current leader | Leader score |
|-----------|:-------:|---------------|-------------|
| SHADE-Arena | IN PROGRESS... | Claude 3.7 Sonnet | 27% |
| CUA-SHADE-Arena | IN PROGRESS... | hybrid monitor | AUC 90% |
| TheAgentCompany | IN PROGRESS... | Gemini 2.5 Pro | 30.3% |
| tau-bench pass^k | IN PROGRESS... | Claude 3.7 (pass^1) | ~80% |
| Remote Labor Index | IN PROGRESS... | Manus / Grok 4 | 2.5% |
| SWE-CI | IN PROGRESS... | Claude Opus 4.6 | 76% zero-regression |

## How We Benchmark

### Production pipeline

All benchmarks run through the same code path as production: `store` →
`build_index` → `recall` / `ask`. No special benchmark-only code paths.
The harness orchestrates test cases, collects telemetry, and runs the judge.

### Multibench mode

Multiple benchmarks run in parallel on the same memory server instance.
Each benchmark uses its own namespace (`key`), but they share the process,
embedding model, and extraction pipeline. This is deliberate — it matches
how the system runs in production with multiple agents and data sources.

### Cross-contamination control

With parallel benchmarks sharing a process, we verify that data from one
benchmark does not leak into another's results. We measure this by:
- Running recall queries from benchmark A against benchmark B's namespace
- Checking that zero facts from other namespaces appear in results
- Current measured cross-contamination: **< 1.5%**

### Judge protocol

- Model: GPT-4.1 (same as most published reference implementations)
- Prompt: standard benchmark-specific judge prompts (matching published baselines)
- Scoring: binary correct/incorrect for factual questions, rubric-based for synthesis

### Karnali

Our proprietary benchmark for evaluating semantic memory quality across
12 categories and 300 questions. Designed to test extraction fidelity,
retrieval precision, and inference accuracy on real-world conversation
and document data. More details to be published separately.

## Status

Multibench tests are on-going. Results and leaderboard positions are
frequently updated as the pipeline improves. The [Memory Adaptation Loop (MAL)](MEMORY-ADAPTATION-LOOP.md)
will automate this improvement cycle once implemented.

---
title: "Benchmarking Local LLMs for SRE Tasks"
date: 2026-06-15
tags: ["llm", "benchmarks", "sre", "ollama", "homelab"]
description: "I ran 7 models against real operational tasks on two GPUs. Here's what I found."
showToc: true
---

Public LLM leaderboards measure things that don't map to operational work. MMLU tests factual recall. HumanEval tests algorithm implementation. Neither tells you whether a model can generate a syntactically valid PromQL query on the first try, summarize a Loki log dump without hallucinating event timestamps, or synthesize a multi-source alert triage plan under a tight token budget.

I spent several weeks running a homelab benchmark suite against 7 models across two GPUs to answer a more specific question: **which local LLM should you run for SRE automation in 2025?**

Here's what I found.

---

## Why Local at All?

Three reasons:

1. **Cost.** SRE agents query models constantly — on a schedule, on every alert, on every pipeline event. Cloud API costs compound quickly when you're generating hundreds of completions per day.

2. **Latency.** An agent waiting 2-4 seconds on a cloud round-trip is noticeably slower for synchronous alerting paths. Sub-second inference on a local GPU changes what architectures are practical.

3. **Privacy.** Ops data — alert payloads, log excerpts, metric dumps — contains hostnames, IP addresses, and service internals you might not want leaving your network.

Local inference is not free. You pay upfront in GPU hardware and ongoing in power and maintenance. But for a homelab-scale SRE automation setup, the tradeoffs are favorable once you have the hardware.

---

## Hardware

| Node | GPU | VRAM | Role |
|---|---|---|---|
| Primary workstation | RTX 5080 | 16GB | Lead orchestrator, fast synchronous tasks |
| Secondary node | RTX 3070ti | 8GB | Subagents, background tasks |

Both nodes run Ollama behind a LiteLLM proxy, which handles load balancing and gives models a unified OpenAI-compatible endpoint.

---

## Methodology

I used [`agent-eval`](https://github.com/wcollani/homelab-tools), a declarative benchmarking framework built on DeepEval. You define tasks in YAML — expected inputs, expected outputs, evaluation criteria — and the framework runs each model against each task, then uses a local judge model to score the output.

**Three task types:**

**PromQL Generation** — Given a natural language description of a metric query (e.g., "P95 latency for the payments service over the last 15 minutes, grouped by pod"), generate a valid PromQL expression. Evaluated for syntactic correctness and label coverage. This is zero-shot — no examples in the prompt, no iterative correction.

**Loki Log Summarization** — Given a 2,000-token excerpt of structured log lines containing one anomaly (a spike in 5xx errors from a specific service), identify the anomaly, the affected service, and the approximate time window. Evaluated for accuracy and absence of hallucinated events.

**OpenTelemetry Instrumentation** — Given a Python function signature and a description of what it does, generate a working code snippet that adds OTEL spans and attributes. Evaluated for correctness and completeness of the tracing setup.

**Evaluation:** Outputs are scored 0.0–1.0 by a local judge model (`qwen2.5-coder:14b` in JSON mode). A score of 1.0 means the output fully satisfies the task criteria; 0.0 means it fails entirely.

**A note on the judge:** Using a local model as the judge means the evaluation is self-referential at the top end — `qwen2.5-coder:14b` judging itself. I treat this as "passing the bar" rather than an absolute quality ceiling. In practice, I also reviewed outputs manually and the scores correlated well with my assessment.

---

## Single-Agent Results

### The Full Table

| Model | VRAM Required | PromQL | Loki Summary | OTEL | Avg Latency |
|---|---|---|---|---|---|
| `qwen2.5-coder:14b` | 16GB | **1.0** | **1.0** | **1.0** | **<0.7s** |
| `qwen2.5-coder:7b` | 8GB | **1.0** | — | **1.0** | ~7s |
| `deepseek-r1:7b` | 8GB | 0.0 | **1.0** | 0.0 | 9.9s |
| `deepseek-r1:14b` | 16GB | 0.4 | 0.8 | 0.3 | ~18s |
| `llama3.1:8b` | 8GB | 0.3 | 0.4 | 0.2 | 10.6s |
| `gemma4:latest` | 16GB | 0.9 | 1.0 | 0.9 | ~20s |
| `qwen3.5:27b` | >16GB | — | — | — | >10 min (aborted) |

*`—` = not tested on that node due to hardware constraints. `qwen2.5-coder:7b` was only tested on the 3070ti (8GB). Latency figures are per-query on the respective GPU.*

### Key Findings

**`qwen2.5-coder:14b` is the clear winner for the 5080.** Perfect scores across all three task types, at under 0.7 seconds per query. It fits entirely in 16GB VRAM with no system RAM spill. For synchronous, latency-sensitive paths — agent loops that run on a schedule, alert summarizers, query generators — this is the model to run.

**`qwen2.5-coder:7b` is the right choice for the 3070ti.** On PromQL and OTEL it matches the 14B model's accuracy at the cost of 7–10x more latency. That's acceptable for background tasks. One critical constraint: capping `num_ctx` to 4096 tokens is mandatory on 8GB VRAM. Larger context windows cause OOM on large log dumps.

**Reasoning models are task-specific.** `deepseek-r1:7b` scored 1.0 on log summarization — a task that benefits from its chain-of-thought — but 0.0 on PromQL generation. Syntax generation is a pattern-matching task. Chain-of-thought doesn't help and may actively interfere by generating intermediate reasoning that ends up in the output.

**`llama3.1:8b` is not suitable for SRE tasks.** Scores across all three tasks were low enough that I wouldn't rely on it for any production path.

**26B+ models are a trap on 16GB VRAM.** `qwen3.5:27b` at 17GB+ causes the model weights to spill into system RAM. Under high-context load (generating a multi-section incident plan), the inference effectively stalls Ollama's request queue. I aborted the run after 10 minutes. These models are only viable for asynchronous overnight tasks where latency is irrelevant.

---

## Multi-Agent Orchestration Results

Single-agent benchmarks miss an important use case: an orchestrator model that has to synthesize outputs from multiple subagents. I ran a second benchmark suite for this.

**Scenario: Alert Triage & Remediation**

A mock high-CPU alert fires. The workflow:
1. **Subagent A** generates a PromQL query for the alert
2. **Subagent B** generates a LogQL query based on the metrics
3. The **orchestrator** receives the raw alert, the PromQL, mock metric data, the LogQL, and mock Loki logs — and produces a structured Markdown incident remediation plan

The "trick" in the payload: the high CPU was a symptom of database connection pool exhaustion visible in the Loki logs. A model that only looks at the CPU metric misses the root cause.

| Model | Hardware | Latency | GEval Score | Notes |
|---|---|---|---|---|
| `qwen2.5-coder:14b` | RTX 5080 | 19–25s | **1.0** | Identified root cause (DB pool exhaustion) |
| `gemma4:latest` | RTX 5080 | ~20s | **1.0** | Same accuracy as qwen2.5-coder:14b |
| `deepseek-r1:14b` | RTX 5080 | 26.1s | **0.8** | Good remediation plan; slight score penalty on structure |
| `llama3.1:8b` | Either | 10.6s | **0.2** | Context window overwhelmed; hallucinated key facts |
| `qwen3.5:27b` | RTX 5080 | >10 min | — | Aborted; system RAM swapping locked up Ollama queue |

The 14B models handled the full multi-source context without losing track of any input. The 8B model could not — it produced a plausible-looking but factually incorrect remediation plan that missed the database root cause entirely. This is the risk of using small models in orchestrator roles: they fail silently by generating confident-sounding nonsense.

---

## Concurrency: Can the Cluster Handle Agent Swarms?

SRE automation often involves multiple simultaneous subagent calls — checking ten services in parallel, spawning one PromQL generator per active alert. I ran a stress test using `asyncio` with 5, 10, 25, and 50 simultaneous requests against each node.

| Node | Model | Peak RPS | 50 Concurrent Requests | Failures |
|---|---|---|---|---|
| RTX 3070ti | `qwen2.5-coder:7b` | **7.77 RPS** | 6.43 seconds | 0 |
| RTX 5080 | `qwen2.5-coder:14b` | **4.55 RPS** | 11.22 seconds | 0 |

Zero failures under 50 concurrent requests. The cluster can sustain a single orchestrator spawning 50 simultaneous subagents and resolve the entire batch in under 12 seconds. For homelab-scale multi-agent workflows, this is more than sufficient headroom.

The 3070ti's higher RPS makes sense: the 7B model has less compute per token, so the GPU serializes requests faster. The 5080's lower RPS reflects the larger model — each request takes longer, but quality is higher.

---

## Framework Gotchas

Three issues hit me during the benchmarking run that are worth documenting:

**Strip `<think>` tags before scoring.** DeepSeek R1 models emit chain-of-thought as `<think>...</think>` blocks before the final answer. If you send the full output to DeepEval's judge, it tries to parse the entire thing as the answer and either scores it incorrectly or crashes on malformed JSON. A simple regex filter before evaluation fixed this: `re.sub(r"<think>.*?</think>", "", output, flags=re.DOTALL)`.

**Cap `num_ctx` on 8GB GPUs.** The default context window on some models is 8K or 16K tokens. On a 3070ti, loading a 4,000-token Loki log dump with a 16K context window causes OOM. Setting `num_ctx: 4096` in the Ollama model configuration is mandatory for stable operation on 8GB hardware.

**Enforce JSON mode on the judge model.** DeepEval expects structured JSON from the judge (score + reasoning). If the judge model emits a markdown code block around the JSON, DeepEval's parser breaks. Setting `response_format={"type": "json_object"}` via LiteLLM ensures clean output regardless of which judge model you use.

---

## Limitations

**These are homebrew tasks, not standard benchmarks.** The PromQL queries, Loki logs, and OTEL scenarios are synthetic — constructed to reflect real workflows, but not validated against a ground truth corpus. Results may not generalize to different task definitions.

**Single hardware configuration.** All tests ran on two specific GPUs. Results on different hardware (AMD, Apple Silicon, older NVIDIA) will vary, particularly for models near VRAM boundaries.

**No fine-tuned models.** Everything tested is a base chat/coder model. Fine-tuned SRE-specific models (if they exist) would likely perform better on these tasks.

**The judge model affects scores.** Using `qwen2.5-coder:14b` as the judge while also benchmarking it creates a conflict of interest at the top of the leaderboard. I mitigated this with manual review, but take the 1.0 self-score with appropriate skepticism.

---

## What to Run

**If you have 16GB VRAM:**
- Synchronous paths (alert summaries, real-time query generation): `qwen2.5-coder:14b`
- Multi-agent orchestrator: `qwen2.5-coder:14b` or `gemma4:latest`
- Async reasoning tasks (postmortems, incident write-ups): `deepseek-r1:14b`
- Avoid 26B+ models for anything synchronous

**If you have 8GB VRAM:**
- Subagent tasks (single-shot PromQL, OTEL snippets): `qwen2.5-coder:7b`
- Log summarization only: `deepseek-r1:7b`
- Always cap `num_ctx` to 4096
- Don't use these as orchestrators — the context window limitation shows at synthesis time

**The 14B sweet spot is real.** The jump from 7B to 14B is not incremental — it's the difference between a model that can hold multi-source context together and one that can't. For any workflow where the model has to synthesize inputs from multiple sources, 14B is the minimum that works reliably.

---

*Source: [`homelab-tools/agent-eval`](https://github.com/wcollani/homelab-tools) — benchmark framework and results. Hardware: RTX 5080 + RTX 3070ti running Ollama + LiteLLM.*

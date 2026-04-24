# Model Considerations

As of 2026-04-22 the repo has landed on **30–70B dense** as the model tier (see [README → Decisions landed](../README.md#decisions-landed)). The specific model within that tier is still open — this file covers the axes that informed the tier choice, the MoE-vs-dense math that ruled out the 200B+ path for this use case, and the 30–70B candidates.

## What matters for hardware sizing

### Dense vs. MoE (Mixture of Experts) architecture

Two independent axes:
- **Total params → storage.** The entire model must fit in memory (VRAM — the GPU's on-board memory — plus system RAM). This determines whether the hardware can run the model at all.
- **Active params → throughput.** Only the active slice is computed per token. This determines tokens/sec.

A 229B MoE with 10B active params needs ~130GB of storage but generates at roughly the speed of a 10B dense model. You get the knowledge capacity of a 229B model at the latency of a 10B model — as long as you have somewhere to park the weights.

The right mental model: **MoE trades storage for speed, not quality for speed.**

**Dense** (e.g. Llama, Qwen3, DeepSeek-R1): every parameter is active every forward pass. What you store is what you compute.

**MoE** (e.g. MiniMax M2.1, DeepSeek-V3, Mixtral): the network has many "expert" sub-networks and a learned router that picks 2–8 of them per token. Total parameter count is large (all experts combined), but active parameter count per token is small. Storage requirement is proportional to total params; compute per token is proportional to active params only.

#### Why MoE exists: more capacity for the same training compute budget
If you want to train a model that knows a lot about many topics, you can either make a larger dense model (expensive to train and run) or make an MoE model where different experts specialize in different domains. You get the breadth of a 229B model at the inference cost of a ~10B model. The tradeoff is that you need to store all 229B somewhere.

#### Where dense wins over MoE at the same total parameter count

This is the key question. If a 30B dense model and a 30B-total-params MoE model exist, which is better?

**Dense wins on:**
- **Coherence and depth on focused tasks.** All weights collaborate on every token. A dense model has richer internal connections for tasks that need sustained, integrated reasoning — like debugging a subtle bug across many files. MoE routing means some information literally never passes through certain experts.
- **Consistency.** The router can send similar inputs to different experts depending on small phrasing differences, which can cause slightly inconsistent outputs. Dense models are more deterministic in character.
- **Fine-tuning quality.** Dense models are easier to fine-tune with LoRA/QLoRA — every layer is always active, so gradient updates are straightforward. MoE fine-tuning requires careful balancing of expert load, and most heavily instruction-tuned or RLHF'd open models are dense.
- **Quantization resilience.** At aggressive quant levels (Q3/Q4), dense models tend to degrade more gracefully. Quantizing individual MoE experts can produce uneven quality drops.
- **Hardware efficiency when it fits in VRAM.** A 30B dense model that lives entirely in 48GB of VRAM runs with zero memory bandwidth overhead for expert loading. A large MoE requiring RAM offload adds latency on every expert swap.

**MoE wins on:**
- **Breadth across diverse domains.** Experts can specialize, so a 229B MoE has more total "knowledge" than a 30B dense model, at comparable inference speed.
- **Inference speed relative to total parameter count.** A 229B dense model would be very slow; a 229B MoE with 10B active params is fast.
- **CPU offload tolerance.** Because only a small set of experts is needed at a time, MoE models can tolerate having most weights in slower system RAM rather than fast VRAM, with less throughput penalty than a dense model in the same situation.

#### The practical upshot for this use case

For a **coding assistant specifically**, the MoE breadth advantage mostly doesn't apply — you're not asking it to switch between writing poetry and analyzing genome sequences. What matters is deep, consistent code understanding. Dense models in the 30–70B range, which have been heavily fine-tuned specifically for code, tend to be strong competitors to much larger MoE models on coding benchmarks, and they run faster on the available hardware.

The main reason to consider a large MoE (229B+) is if you want general capability across many domains from one model, or if you find benchmarks showing it clearly outperforms the dense alternatives on your specific tasks.

#### Breaking that down by actual task

The generic "dense vs MoE" comparison isn't evenly weighted across the different things Claude Code actually does. Split by task:

| Task Claude Code performs | Dense vs MoE | Why |
|---------------------------|--------------|-----|
| **Writing new code in a familiar stack** | Roughly tied | Either can generate idiomatic code in common languages. Dense has a slight edge on consistency. |
| **Writing code in an unfamiliar / rarely-used stack** | **MoE wins** | MoE's breadth of library and framework knowledge pays off when the task touches a niche ecosystem (obscure Go library, DSL, etc.). |
| **Debugging / tracing cross-file bugs** | **Dense wins decisively** | Holding a multi-file mental model across reasoning steps benefits from every parameter collaborating. MoE routing fragments sustained reasoning chains — the router may send "line 47 of module A" and "line 102 of module B" to different experts that never reconcile. |
| **Refactoring / multi-step architectural changes** | **Dense wins** | Consistency is the critical property. Dense responds the same way to similar prompts, so a refactor plan laid out on turn 1 is still recognizable on turn 15. MoE's router can shuffle "voice" and style between turns. |
| **Controlling / steering a project** (style rules, naming conventions, test patterns) | **Dense wins** | Same consistency argument. Dense models let you specify a convention once and have it stick. |
| **Agentic loops with tool calls** (Bash, Read, Edit, Grep) | **Dense wins slightly** | Tool-call JSON validity is tighter because the same attention heads see every token. MoE can produce inconsistent JSON formatting, especially under aggressive quantization — see [the quantization-for-coding section below](#how-quantization-affects-programming-outcomes). |
| **Broad recall of "how do I do X in library Y?"** | **MoE wins** | The single thing MoE is unambiguously best at — many small specialist experts > one generalist. |

**Net:** for a codebase-focused agent (Claude Code's actual use case), dense 30–70B is the right call for 4 out of 5 typical operations. The one vertical where MoE wins — broad library recall — is usually covered by a quick web search anyway. This is the reasoning behind the repo's [30–70B dense decision](../README.md#decisions-landed).

### Parameter count and quant
| Model size | Q4 quant | Q8 quant | Notes |
|-----------|---------|---------|-------|
| 30–32B | ~18GB | ~34GB | Fits in single 24GB GPU (Q4) |
| 70B | ~43GB | ~75GB | Needs 48GB+ GPU for Q4, 96GB for Q8 |
| 120–130B dense | ~75GB | ~130GB | Needs 96GB+ for comfortable Q4 |
| 200B+ MoE | ~108–130GB (Q4) | ~254GB (Q8) | Active params ~10B; rest in system RAM |

### Prompt processing vs. token generation
- Token gen speed (tok/s) is what people benchmark. Usually VRAM-bandwidth-bound.
- Prompt processing (prefill) matters for Claude Code with large repo contexts — CUDA has a real advantage here over Metal.

---

## Model comparison cheat sheet

Axes to look at when comparing any two open-weight candidates. Hardware-impacting items link back to the sections above.

### Quantization — the single most important knob
Reducing numeric precision of the weights so the model fits in less memory. Original training precision is usually FP16/BF16 (16-bit float). You choose a quant level at inference time.

| Level | Size vs FP16 | Quality loss | When to use |
|-------|--------------|--------------|-------------|
| FP16 / BF16 | 1.0× | none (reference) | You have the VRAM and want max fidelity |
| FP8 | 0.5× | ~negligible | Native on Blackwell/Hopper; DGX Spark default |
| Q8 / INT8 | 0.5× | ~1% | Near-lossless fallback |
| Q6_K | 0.38× | <2% | Best quality that still halves memory |
| **Q5_K_M** / **Q4_K_M** | 0.31× / 0.27× | 2–5% | **Standard sweet spot**; most GGUFs ship here |
| Q4 (INT4) | 0.25× | 3–7% | Tight-memory path; AutoRound/GPTQ/AWQ preserve quality better than naive round-to-nearest |
| Q3 | 0.2× | 5–12% | Only if you must (e.g. 200B MoE on 128GB Spark) |
| Q2 | 0.14× | major | Avoid; usually unusable |

**Why quantize.** A 70B model is ~140GB at FP16 but ~40GB at Q4 — the difference between "needs a mid-tier DIY" and "fits on DGX Spark with headroom." Every memory tier (Spark 128GB, single-3090 24GB, A6000 48GB) is really a "what quants of which model sizes fit here" question.

**Quant formats you'll see.**
- **GGUF** (llama.cpp native) — names like `Q4_K_M`: K = k-quants (newer, better), M/S/L = medium/small/large variant. Mixed precision per layer.
- **GPTQ / AWQ** — older INT4 quant methods, common on Hugging Face.
- **AutoRound (AR)** — Intel's 2025+ quant method; can mix INT4 + FP8 per layer for better quality retention (see [Spark_Resources](../Spark_Resources/README.md)).
- **MLX** — Apple's format, usually labelled `-4bit` / `-8bit`.

**Rule of thumb.** Start at Q4_K_M (or FP8 on Spark). If quality is insufficient, step up to Q5/Q6 before changing models. If quality is fine, don't bother going higher.

#### How quantization affects programming outcomes

The general "X% quality loss" numbers above are averaged across benchmarks and paper over which tasks actually break first. For a coding agent, the failure modes at each level are specific:

| Quant | Coding-specific failure modes |
|-------|-------------------------------|
| FP16 / BF16 | Reference baseline; nothing to report |
| FP8 | Indistinguishable from BF16 for code. Default on Blackwell / DGX Spark |
| Q8 | ~1% drop; safe for all Claude Code workflows |
| Q6_K | Subtle loss on long reasoning chains (40k+ tokens); imperceptible on short edits |
| Q5_K_M | Multi-file refactor plans start losing coherence after 4–5 back-and-forth turns; single-file edits still fine |
| **Q4_K_M (sweet spot)** | (a) Tool-call JSON occasionally malformed under streaming; (b) bug-traces lose architectural thread after 3–4 steps; (c) library-API hallucination rate ticks up (invented function names, wrong argument order) |
| Q4_0 / naive INT4 | All Q4_K_M issues amplified. AutoRound / GPTQ / AWQ preserve quality noticeably better than naive round-to-nearest at the same bitwidth |
| Q3 | Tool-call JSON frequently malformed → **breaks Claude Code's agent loop entirely** (the loop parses JSON and crashes on invalid) |
| Q2 | Unusable for coding; reasoning chain collapses on anything longer than a docstring |

Things to actually watch out for, in descending order of importance for an agentic coding setup:

- **Tool-call JSON validity is the agent-loop tripwire.** A model that emits perfectly valid JSON at Q6 can start dropping commas or closing brackets at Q4. Claude Code's loop doesn't recover — it sees a parse error and bails. This single failure mode is the biggest reason not to quant aggressively for an agent endpoint. If a local build is working at Q8 and you drop to Q4 for memory headroom and suddenly everything crashes mid-tool-use, this is why.
- **Long-context coherence degrades faster than short-prompt accuracy.** A Q4 model that scores 95% of its Q8 baseline on HumanEval (short, single-function tasks) can drop to 70% on SWE-Bench Verified (real multi-file bug fixes). The longer the context, the more per-token rounding errors compound.
- **Library-API hallucination rate rises.** At Q4 the model is more likely to confidently invent a function signature that doesn't exist, or use the wrong argument order. This shows up especially on less-common libraries that made up a smaller fraction of training data.
- **Code-style drift across turns.** A Q4 model is more likely to forget "we agreed to use `camelCase` on turn 3" by turn 8. Dense models drift less than MoE at the same quant level — another reason the repo landed on dense.
- **AutoRound / FP8-hybrid quants preserve Q4 memory savings with close to Q7 quality** when the model ships one. Qwen3.5-122B-A10B does (see [`Spark_Resources/README.md`](../Spark_Resources/README.md)). Worth preferring when available.
- **Default for a Claude Code endpoint:** stay at **Q4_K_M or higher** for anything that runs the agent loop. Use **FP8 on Blackwell / Spark** — on supported hardware it's essentially free quality over Q8. Drop below Q4_K_M only if you're experimenting with models that otherwise wouldn't fit at all.

### Total vs. active parameters (dense vs. MoE)
Covered [above](#dense-vs-moe-mixture-of-experts-architecture). Short version: **total params → storage**, **active params → speed**. For dense models they're the same number. Always check both when looking at an MoE.

### Context window (context length)
Max tokens the model can see at once (system prompt + conversation + generation). Commonly 8k / 32k / 128k / 1M. Claude Code sends 8–32k of repo context per request, so **anything under 32k is a hard blocker** for this use case; 128k gives comfortable headroom.

**KV cache cost scales with context.** A 32k context on a 70B model adds ~5–10GB of VRAM on top of the weights. A 128k context on the same model can add 20GB+. This is the number that turns "fits in VRAM at rest" into "OOMs on a long prompt." Budget for it.

### Base vs. Instruct vs. Coder variants
- **Base** — raw next-token predictor; not tuned to follow instructions. Skip for this use case.
- **Instruct** / **Chat** — RLHF or SFT-tuned to answer questions and follow commands. Minimum requirement for a Claude Code endpoint.
- **Coder** — additional code-corpus training on top. For a coding endpoint, always prefer the Coder variant if one exists (e.g. `Qwen3-Coder-30B-Instruct` over plain `Qwen3-30B-Instruct`).

### License
Matters if this ever serves anything beyond personal use.
- **Apache 2.0 / MIT** — full commercial use, no strings (Qwen, Mistral non-premium).
- **Llama community license** — commercial ok until ~700M MAU cap (Meta Llama family).
- **DeepSeek / Yi / Custom** — read the terms; most are permissive but explicit about allowed uses.
- **Gated** (request access on Hugging Face) — technical friction, not a usage restriction per se.

### Tokenizer efficiency
Different tokenizers encode the same text into different token counts. Code-trained tokenizers (Qwen-Coder, DeepSeek-Coder, Codestral) compress source code ~10–20% better than general-purpose tokenizers (Llama), which means more code fits per 32k context window *and* prefill is faster.

### Training cutoff / knowledge recency
An open-weight model trained in mid-2024 doesn't know React 19, Bun 1.2, or anything shipped since. For coding models this matters — many are refreshed on a 6–12 month cadence. Check the model card for the cutoff date before committing.

### Benchmarks to look at for coding
- **SWE-Bench Verified** — real bug-fix tasks across open-source repos. Most relevant for Claude-Code-style work. Treat as the primary signal.
- **LiveCodeBench** — time-windowed competitive-programming problems; less likely to be training-set contaminated than older benchmarks.
- **HumanEval / MBPP** — basic function-completion; **saturated** and near-useless for comparing modern frontier models, but still quoted everywhere. Disregard if they're the only numbers reported.
- **Aider polyglot** — multi-file edits across languages; a good stand-in for agentic coding.

Subjective eval on your own codebase beats any benchmark. The plan to test candidates on the learning build before buying Spark exists for exactly this reason.

### Speculative decoding (nice-to-have, not required)
A small "draft" model predicts several tokens ahead and the main model verifies them in one pass, boosting tok/s without changing quality. Examples: MTP-2 (used in the AutoRound Qwen3.5 repo, giving +80% on Spark), Medusa, EAGLE. Relevant if a chosen model already ships a draft head; not something to block on.

---

## Best-in-class open-weight coders (April 2026)

A quick survey beyond the four candidates listed below. Goal: know what's out there before committing. This is deliberately not exhaustive — only models with credible coding performance *and* working tool-calling through vLLM or Ollama are listed (Claude Code needs both).

### Top tier for agentic coding
- **Qwen3-Coder-30B-A3B-Instruct** — Alibaba's MoE-based coder (30B total, ~3B active). Strong on SWE-Bench Verified, excellent tool calling via vLLM's `qwen3_coder` parser. The single most common recommendation for local Claude Code as of April 2026.
- **Qwen3-Coder-72B** — larger dense sibling. Higher accuracy, slower (~150 tok/s on Spark). Use when you need the extra reasoning and can absorb the latency.
- **Qwen3-72B-Instruct** (general, not Coder) — still a strong coder because of Qwen's training mix. Pick this over Qwen3-Coder-72B only if you also want strong non-coding reasoning in the same endpoint.
- **DeepSeek-V3.x** (236B MoE, ~21B active) — stronger than the 30–70B tier on long-horizon complex problems; needs mid-tier DIY build, not Spark. Listed here for completeness as the Plan B escape hatch if 30–70B dense proves insufficient.
- **Llama-4 Scout / Maverick** — Meta's MoE line (17B active / 128B total for Scout). Decent coding but weaker than Qwen3-Coder at the same size class as of early 2026.

### Specialized
- **Devstral** (Mistral, May 2025+) — Mistral's agentic coder; the successor to Codestral. **Use this, not Codestral.** Codestral was explicitly shipped without tool calling, making it a dead end for Claude Code agent loops. Devstral fixes that.
- **StarCoder2 / StarCoder3** (BigCode) — open research models; solid completions but lighter tool-calling support. More useful as a comparison baseline than a production endpoint.
- **Yi-Coder** — capable smaller coder; English + Chinese. Niche choice.

### Leaderboards worth consulting
- **[Aider polyglot leaderboard](https://aider.chat/docs/leaderboards/)** — real multi-file edits across languages. The best public signal for agentic coding, closest analog to what Claude Code actually does.
- **[SWE-Bench Verified](https://www.swebench.com/)** — real-world bug fixes on open-source repos. Harder and more relevant than HumanEval.
- **[LiveCodeBench](https://livecodebench.github.io/)** — time-windowed competitive programming, resistant to training-set contamination.
- Disregard HumanEval / MBPP scores for model selection at this level — they're saturated and no longer discriminate between frontier candidates.

### Honest baseline
Frontier closed models (Claude 4.X, GPT-5, Gemini 3) remain the accuracy ceiling as of April 2026. A 30–70B dense open-weight coder running locally trades some accuracy for cost, latency, privacy, and control — and for most Claude Code sessions on a personal codebase, the trade is worthwhile. Don't expect parity; expect "good enough for 85% of sessions + hosted API for the hard 15%" as the practical pattern.

---

## Candidate Models

### Small / fast coding models (~30B)
Good fit for: DGX Spark (single unit), learning build with 24GB GPU, any hardware.

| Model | Params | Notes |
|-------|--------|-------|
| Qwen3 Coder 30B | 30B | [~483 tok/s on DGX Spark (FP8)](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks/) — very fast |
| DeepSeek-Coder-V2-Lite | 16B MoE (2.4B active) | Distilled from Coder V2; extremely efficient for coding; see full model in Large MoE section |
| Codestral 22B (Mistral) | 22B | Strong coding benchmark scores |

### Mid-size (~70B)
Good fit for: DGX Spark (single unit), mid-tier build. At Q4/FP8 a 70B model is ~43–70GB — well within the 128GB ceiling with room for KV cache. Only FP16 (~140GB) would exceed it.

| Model | Params | Q4 storage | Notes |
|-------|--------|-----------|-------|
| Qwen3 72B | 72B dense | ~43GB | Strong reasoning + coding |
| DeepSeek-R1 70B | 70B dense | ~43GB | Reasoning-focused, strong coder |
| Llama 4 Scout | 17B MoE (3B active) | ~10GB | Very efficient; fits on any hardware |

### Large MoE (~200B+)
Good fit for: mid-tier+ build with large system RAM. Not a good fit for DGX Spark single unit — all weights must be stored in memory; Q4 of a 229B MoE is ~130GB, which exceeds the 128GB unified memory ceiling after accounting for KV cache. ([NVIDIA's own spec](https://www.nvidia.com/en-us/products/workstations/dgx-spark/) says "up to 200B parameters" for inference.)

| Model | Params | Active params | Q4 storage | Notes |
|-------|--------|--------------|------------|-------|
| DeepSeek Coder V2 | 236B MoE | ~21B | ~140GB | Coding-focused; more active params than M2.1 so stronger on complex code; needs mid-tier build |
| MiniMax M2.1 | 229B MoE | ~10B | ~130GB | Original reference; general-purpose; needs Q3 (~108GB) on DGX Spark |
| DeepSeek-V3 | 671B MoE | ~37B | ~400GB | General-purpose; needs mid-tier build minimum |

---

## Key Insight: Model Choice Drives Hardware Choice

A DGX Spark at $4,699 running Qwen3 Coder 30B at ~480 tok/s is a fundamentally different proposition than a DIY EPYC rig running MiniMax M2.1 at 10 tok/s. Before sizing hardware, it's worth deciding whether the goal is:

1. **Best quality at any speed** → bigger MoE model, more RAM, learn the infrastructure
2. **Best usability for Claude Code** → fastest tok/s on a strong coding model, simplest setup
3. **Most control / learning** → DIY Linux server regardless of model

For mocking the Claude API in Claude Code, option 2 may be the right frame: a 30–70B coding model running at 100+ tok/s feels much more like the real API than a 200B model at 10 tok/s.

---

## Why the Linux/CUDA and Mac Studio paths target different models

The shopping list for the DIY learning build ([`shopping/learning-build.md`](../shopping/learning-build.md)) targets **MiniMax M2.1 (GGUF via llama.cpp)**, while the Mac Studio MLX docs ([`setup/mac-studio-mlx-remote.md`](../setup/mac-studio-mlx-remote.md)) target **Qwen2.5-Coder-32B-Instruct (MLX format)**. That is not an inconsistency — the two platforms have different native inference stacks:

- **CUDA / Linux**: llama.cpp and vLLM consume GGUF or safetensors. MiniMax M2.1 at Q4 was the original reference target because it stresses the DIY build's large-RAM + GPU-offload path. On DGX Spark (128GB ceiling), use Q3 instead — see [`builds.md`](builds.md) Option 0.
- **Apple Silicon**: MLX is Apple's native framework; MLX weights are the only format that fully uses the unified memory + Metal path. `mlx-community/Qwen2.5-Coder-32B-Instruct-8bit` (~35GB) is the sensible coding target at 128GB — leaves headroom for context and macOS, and runs at 15–30 tok/s on M4 Max.

Both paths are serving the same goal (Claude-Code-compatible endpoint), but each targets the model format native to its hardware. If the production endpoint ends up being the Linux/CUDA Spark (as currently recommended in the README), the Mac Studio remains a useful side-experiment for the colleague and is not on the critical path.

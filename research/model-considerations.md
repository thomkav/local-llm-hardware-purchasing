# Model Considerations

The model choice is open. This file covers the key axes that affect hardware sizing, plus candidate models.

## What matters for hardware sizing

### Dense vs. MoE architecture

Two independent axes:
- **Total params → storage.** The entire model must fit in memory (VRAM + system RAM). This determines whether the hardware can run the model at all.
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

## Candidate Models

### Small / fast coding models (~30B)
Good fit for: DGX Spark (single unit), learning build with 24GB GPU, any hardware.

| Model | Params | Notes |
|-------|--------|-------|
| Qwen3 Coder 30B | 30B | [~483 tok/s on DGX Spark (FP8)](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks/) — very fast |
| DeepSeek-Coder-V2-Lite | 16B MoE (2.4B active) | Distilled from Coder V2; extremely efficient for coding; see full model in Large MoE section |
| Codestral 22B (Mistral) | 22B | Strong coding benchmark scores |

### Mid-size (~70B)
Good fit for: mid-tier build (2× A6000), DGX Spark pair (256GB).

| Model | Params | Notes |
|-------|--------|-------|
| Qwen3 72B | 72B | Strong reasoning + coding |
| Llama 4 Scout | 17B MoE (active ~3B) | Very efficient, good instruction following |
| DeepSeek-R1 70B | 70B | Reasoning-focused, strong coder |

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

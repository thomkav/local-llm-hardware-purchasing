# Model Considerations

The model choice is open. This file covers the key axes that affect hardware sizing, plus candidate models.

## What matters for hardware sizing

### Dense vs. MoE architecture
- **Dense** (e.g. Llama, Qwen, DeepSeek-R1): every parameter is active every token. VRAM requirement = model size at chosen quant. Straightforward.
- **MoE** (e.g. MiniMax M2.1, DeepSeek-V3, Mixtral): only a fraction of params are active per token; the rest sit in RAM/disk. A 200B+ MoE can run on much less VRAM than a 200B dense model.

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
| Qwen3 Coder 30B | 30B | ~483 tok/s on DGX Spark (FP8) — very fast |
| DeepSeek-Coder-V2-Lite | 16B MoE (2.4B active) | Extremely efficient for coding |
| Codestral 22B (Mistral) | 22B | Strong coding benchmark scores |

### Mid-size (~70B)
Good fit for: mid-tier build (2× A6000), DGX Spark pair.

| Model | Params | Notes |
|-------|--------|-------|
| Qwen3 72B | 72B | Strong reasoning + coding |
| Llama 4 Scout | 17B MoE (active ~3B) | Very efficient, good instruction following |
| DeepSeek-R1 70B | 70B | Reasoning-focused, strong coder |

### Large MoE (~200B+)
Good fit for: mid-tier+ build, large system RAM + 24GB+ GPU.

| Model | Params | Active params | Notes |
|-------|--------|--------------|-------|
| MiniMax M2.1 | 229B MoE | ~10B | Original reference; Q4 ~130GB total |
| DeepSeek-V3 | 671B MoE | ~37B | Very capable; Q4 needs ~400GB total |

---

## Key Insight: Model Choice Drives Hardware Choice

A DGX Spark at $4,699 running Qwen3 Coder 30B at ~480 tok/s is a fundamentally different proposition than a DIY EPYC rig running MiniMax M2.1 at 10 tok/s. Before sizing hardware, it's worth deciding whether the goal is:

1. **Best quality at any speed** → bigger MoE model, more RAM, learn the infrastructure
2. **Best usability for Claude Code** → fastest tok/s on a strong coding model, simplest setup
3. **Most control / learning** → DIY Linux server regardless of model

For mocking the Claude API in Claude Code, option 2 may be the right frame: a 30–70B coding model running at 100+ tok/s feels much more like the real API than a 200B model at 10 tok/s.

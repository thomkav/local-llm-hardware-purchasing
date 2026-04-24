# MiniMax M2.1 Requirements

> **Scope note (2026-04-22):** MiniMax M2.1 is no longer the primary production target — the repo has landed on 30–70B dense on DGX Spark (see [README](../README.md#decisions-landed)). This file remains as reference for the MoE storage/compute math and as a sizing guide for the learning build when experimenting with large-MoE offload patterns.

## Model Specs
- 229B parameter MoE (Mixture of Experts)
- ~10B active parameters per token
- This matters: MoE means CPU offload works better than on dense models

## Quant Options (via Unsloth GGUFs)
| Quant | Size | Notes |
|-------|------|-------|
| UD-Q4_K_XL | ~130GB | Original reference quant (pre-2026-04-22 decision); fits 128GB RAM + small disk offload |
| Q3 (3-bit dynamic) | ~108GB | Fits in 128GB RAM fully; ~20–25 tok/s on M4 Max |
| Q4 (4-bit dynamic) | ~108GB | M3 Ultra 256GB confirmed working |
| Q8 | ~254GB | Needs 256GB RAM minimum |
| BF16 (unquantized) | ~458GB | Needs 512GB+ RAM |

## Performance Expectations (learning build)
- RTX 3090 24GB + 128GB DDR4 system RAM
- Run Q4_K_XL (~130GB): GPU handles active params, CPU/RAM handles experts
- Expected: ~8–15 tok/s on coding prompts
- Prompt ingestion (big repo context) will be the bottleneck — CUDA faster than Metal here

## Why MoE Matters for Hardware Sizing
- Only ~10B params are active per forward pass
- Remaining experts sit in RAM/disk, loaded on demand
- A single 24GB GPU is enough to keep active layers hot
- 192GB VRAM + 768GB RAM (Reddit build) is overbuilt for this model
- The learning build is NOT underpowered — it's appropriately sized

# Build Comparisons

Build choice is still open. See `model-considerations.md` — hardware sizing depends heavily on which model tier you target.

---

## Option 0 — DGX Spark (~$4,699)

NVIDIA GB10 Grace Blackwell Superchip. Compact desktop, 300W peak. ([Product page](https://www.nvidia.com/en-us/products/workstations/dgx-spark/), [LMSYS review](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark/))

- **Memory**: 128GB LPDDR5X unified — CPU and GPU share the same pool ([specs](https://www.aitooldiscovery.com/ai-infra/nvidia-dgx-spark-explained))
- **AI compute**: ~1 PFLOP FP4
- **Inference perf**: [~483 tok/s on Qwen3 Coder 30B (FP8), ~38 tok/s generation on GPT-OSS 120B](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks/)
- **Two-unit NVLink**: connect two for 256GB unified memory, ~$9,400
- **Management**: NVIDIA's own tooling; no traditional IPMI but decent remote management
- **Form factor**: Mac Mini-sized, quiet, 300W — no rack needed
- **Availability**: in stock via [NVIDIA](https://marketplace.nvidia.com/en-us/enterprise/personal-ai-supercomputers/dgx-spark/), [Amazon](https://www.amazon.com/NVIDIA-DGX-SparkTM-Supercomputer-Blackwell/dp/B0FWJ16CCH), Acer/ASUS/Dell/MSI OEM variants
- **Price note**: [increased from $3,999 → $4,699 in February 2026](https://forums.developer.nvidia.com/t/2-23-2026-price-change-announcement/361713) due to LPDDR5X supply constraints

**Best fit if**: model choice lands on 30–70B range; you want a clean turnkey box; you don't need IPMI-grade headless management.

**Memory ceiling caveat**: 128GB is not about compute — MoE models are indeed efficient per token. But *all* weights must be resident in memory so any expert can be called. MiniMax M2.1 at Q4 is ~130GB total, which exceeds 128GB once you account for KV cache. The Q3 quant (~108GB) fits. NVIDIA's own spec says ["up to 200B parameters"](https://www.nvidia.com/en-us/products/workstations/dgx-spark/) for inference, which aligns with this. For models under ~120B (or any 30–70B model), this isn't a constraint at all.

---

## Option 1 — Learning Build: Single RTX 3090 (~$2,250–2,680)

Single GPU, 128GB DDR4, EPYC Rome. Cheapest path to hands-on learning.

- CPU: EPYC 7402P (24c Rome, SP3, unlocked) ~$200–350
- Motherboard: Supermicro H12SSL-i (IPMI) ~$550–750
- RAM: 8× 16GB DDR4-3200 RDIMM ECC (128GB) ~$250–400 used
- GPU: Used RTX 3090 24GB ~$650–800 (EVGA FTW3 Ultra candidate)
- PSU/case/cooler/storage: ~$600
- **Total: ~$2,250–2,680**
- **Performance**: ~8–15 tok/s on a 200B MoE; much faster on 30–70B models

Full sourcing: `../shopping/learning-build.md`

**Best fit if**: primary goal is learning the stack; willing to upgrade GPU/RAM later; budget-constrained.

**Weakness**: 24GB GPU limits in-VRAM model size; slower prompt processing for large repo contexts.

---

## Option 2 — Mid-Tier DIY: Used Milan + 2× A6000 (~$8–11k)

- CPU: EPYC 7713P (64c Milan, SP3) used ~$600–900
- Motherboard: Supermicro H12SSL-i ~$700–900
- RAM: 12× 64GB DDR4-3200 RDIMM ECC (768GB) ~$2,400–4,800 used
- GPU: 2× used RTX A6000 48GB (~$6,000–9,000 pair)
- Chassis + PSU + storage ~$800–1,200
- **Total: ~$10–15k**
- **Performance**: Q4/Q5 of a 200B MoE fully in VRAM, ~30–50 tok/s; 70B models very fast

Sources: TheServerStore, UNIXSurplus, Bargain Hardware (UK), ServerMonkey, r/hardwareswap

**Best fit if**: want IPMI headless management + full VRAM for large models + room to grow.

---

## Option 3 — New Build: Genoa + Pro 6000 Blackwell (~$20–27k)

- CPU: EPYC 9354P (32c Genoa, SP5) ~$1,500–2,400
- Motherboard: Supermicro H13SSL-N ~$650–800
- RAM: 12× 32GB DDR5-4800 RDIMM ECC (384GB) — currently ~$15–18k (DRAM shortage)
- GPU: 1× RTX Pro 6000 Blackwell 96GB ~$8,900
- **Total: ~$20–27k** (DDR5 pricing is the main blocker right now)

**Best fit if**: all-new hardware required; DDR5 prices normalize; need maximum prompt processing speed.

---

## Mac Studio (not recommended for this use case)

- M3 Ultra 256GB (~$5.6k when available) — 819GB/s memory bandwidth
- M4 Max 128GB (~$3.5k) — caps at 128GB
- Issues: no IPMI, soldered RAM, slower CUDA-equivalent prefill on large contexts, Apple premium
- As of April 2026: both configs "currently unavailable," no ETA; M5 Ultra expected ~WWDC June 2026

---

## Market Timing Notes (as of April 2026)

**Reasons to wait:**
- DDR5 RDIMM prices should ease in 12–24 months
- M5 Ultra (~WWDC June 8) may reset used workstation GPU prices
- RTX Pro 6000 Blackwell prices will drift down as used units appear

**Reasons not to wait:**
- Used A6000/A100 prices are rising, not falling
- DRAM could stay tight if AI demand keeps scaling
- Tariffs/export controls — nothing gets cheaper after a round
- Learning time lost: 6 months hands-on > 20% hardware discount

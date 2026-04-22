# Local LLM Hardware Purchasing

Research and shopping notes for building a headless Linux server to run **MiniMax M2.1** locally, mocking the Anthropic/Claude API endpoint for use with Claude Code.

---

## Status

| Phase | Status |
|-------|--------|
| Research builds + tradeoffs | ✅ Done |
| Source components | 🔄 In progress |
| Purchase | ⬜ Not started |
| Build + burn-in | ⬜ Not started |
| Install software stack | ⬜ Not started |
| Test Claude Code integration | ⬜ Not started |

---

## Decision: Learning Build (~$2,250–2,680)

Single GPU, 128GB RAM, EPYC Rome SP3. Learn the stack before committing to bigger hardware.

### Why not the big rig?
MiniMax M2.1 is a 229B **MoE** model with only ~10B active params per token. The 192GB VRAM + 768GB RAM build people post on Reddit is overbuilt for this model. A single 24GB GPU keeps the active layers hot; the rest lives in system RAM. See [`research/model-requirements.md`](research/model-requirements.md).

### Why not Mac?
No IPMI (no out-of-band management for a headless server), soldered RAM, slower prompt ingestion vs. CUDA on large repo contexts, and the Apple premium isn't justified for a server. M3 Ultra 256GB was the best option but has been "currently unavailable" since April 2026 with no ETA. See [`research/builds.md`](research/builds.md).

---

## Component List

| Component | Choice | Target Price | Status |
|-----------|--------|-------------|--------|
| GPU | EVGA RTX 3090 FTW3 Ultra 24GB | $650–750 | Candidate identified |
| CPU | EPYC 7402P 24c (Rome, SP3, unlocked) | $200–350 | Not purchased |
| Motherboard | Supermicro H12SSL-i (IPMI) | $550–750 | Not purchased |
| RAM | 8× 16GB DDR4-3200 RDIMM ECC (128GB) | $250–400 | Not purchased |
| PSU | Corsair RM1000x 1000W | $180 | Not purchased |
| Case | Phanteks Enthoo Pro II Server Edition | $170 | Not purchased |
| CPU Cooler | Noctua NH-U14S TR4-SP3 | $100 | Not purchased |
| Storage | Samsung 990 Pro 2TB NVMe | $150 | Not purchased |
| **Total** | | **$2,250–2,680** | |

Full sourcing links and notes: [`shopping/learning-build.md`](shopping/learning-build.md)

---

## Planned Software Stack

| Layer | Choice |
|-------|--------|
| OS | Ubuntu 24.04 Server |
| Inference engine | llama.cpp (CUDA backend, best MoE support) |
| Model | MiniMax-M2.1 UD-Q4_K_XL GGUF (~130GB, via Unsloth) |
| API proxy | claude-code-router (OpenAI-compatible → Claude Code) |

Expected throughput: **~8–15 tok/s** on coding prompts with this hardware.

---

## Research Notes

- [`research/model-requirements.md`](research/model-requirements.md) — M2.1 specs, quant options, why MoE changes the hardware math
- [`research/builds.md`](research/builds.md) — all build tiers compared (learning → mid-tier → full), Mac analysis, market timing
- [`research/risks.md`](research/risks.md) — used hardware risk analysis and verification checklists
- [`shopping/learning-build.md`](shopping/learning-build.md) — active shopping list with per-component sourcing links

---

*Research started: 2026-04-17*

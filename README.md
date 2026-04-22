# Local LLM Hardware Purchasing

Research and shopping notes for running an open-weight coding model locally to mock the Anthropic/Claude API endpoint for Claude Code. Both the model and hardware are still being evaluated.

---

## Status

| Phase | Status |
|-------|--------|
| Research builds + tradeoffs | ✅ Done |
| **Choose a model tier** | 🔄 In progress |
| **Choose a build** | 🔄 In progress |
| Source components | ⬜ Not started |
| Purchase | ⬜ Not started |
| Build + burn-in | ⬜ Not started |
| Install software stack | ⬜ Not started |
| Test Claude Code integration | ⬜ Not started |

---

## Open Questions

**Model first, then hardware.** See [`research/model-considerations.md`](research/model-considerations.md) for the full breakdown. Key question:

> Is the goal **interactive coding feel** (30–70B dense, 100–500 tok/s, fits on a single DGX Spark), or **maximum capability** (200B+ MoE, ~10–40 tok/s, needs a DIY build with large RAM)?

For mocking the Claude API interactively, a fast 30–70B model likely feels better than a slow 200B one. 30–70B dense models fit comfortably on a single DGX Spark at Q4/FP8 (~43GB for a 70B model vs 128GB available). Large MoE models (229B+ at Q4 = 130GB+) exceed the DGX Spark's ceiling and require a DIY build.

### Factors still being weighed
- Whether 30–70B quality is sufficient, or 200B+ MoE is needed
- DIY server (IPMI, expandable, more complex) vs. DGX Spark (turnkey, quiet, $4,699)
- Used hardware risk tolerance — see [`research/risks.md`](research/risks.md)
- DDR5 RDIMM pricing (currently inflated ~2×; may normalize in 12–24 months)
- M5 Ultra Mac Studio expected ~WWDC June 2026

---

## Build Options

Full comparison in [`research/builds.md`](research/builds.md).

Throughput varies significantly by model size — numbers below are illustrative per tier.

| Build | Cost | Memory | Throughput by model tier | Best for |
|-------|------|--------|--------------------------|----------|
| [DGX Spark](research/builds.md#option-0--dgx-spark-4699) | ~$4,699 | 128GB unified | ~480 tok/s (30B), ~150 tok/s (70B) | Turnkey, up to 70B, quiet, no rack |
| [Learning (single 3090)](research/builds.md#option-1--learning-build-single-rtx-3090-2250-2680) | ~$2,250–2,680 | 24GB VRAM + 128GB RAM | ~60 tok/s (30B in VRAM), ~10 tok/s (200B MoE offloaded) | Start learning now; 30B sweet spot |
| [Mid-tier DIY (Milan + 2× A6000)](research/builds.md#option-2--mid-tier-diy-used-milan--2-a6000-8-11k) | ~$8–11k | 96GB VRAM + 768GB RAM | ~150 tok/s (70B), ~30–50 tok/s (200B MoE) | Large MoE fully in VRAM, IPMI, expandable |
| [New build (Genoa + Pro 6000)](research/builds.md#option-3--new-build-genoa--pro-6000-blackwell-20-27k) | ~$20–27k | 96GB VRAM + 384GB DDR5 | similar to mid-tier | All new; DDR5 currently overpriced |
| Mac Studio M3 Ultra 256GB | ~$5.6k | 256GB unified | 20–25 tok/s | Not recommended — no IPMI, unavailable |

---

## Planned Software Stack

| Layer | Choice |
|-------|--------|
| OS | Ubuntu 24.04 Server (or DGX OS for Spark) |
| Inference engine | llama.cpp (CUDA) or NVIDIA NIM (Spark) |
| Model | TBD — see `research/model-considerations.md` |
| API proxy | claude-code-router (OpenAI-compatible → Claude Code) |

---

## Research Notes

- [`research/model-considerations.md`](research/model-considerations.md) — model tiers, dense vs MoE tradeoffs, candidate models
- [`research/builds.md`](research/builds.md) — all build options compared, power costs, market timing
- [`research/risks.md`](research/risks.md) — used hardware risk analysis and verification checklists
- [`shopping/learning-build.md`](shopping/learning-build.md) — component sourcing for the learning build option

## Installation Notes

- [`setup/basement-install.md`](setup/basement-install.md) — humidity mitigation, rack mounting options, networking (switch, Tailscale), cabling

---

*Research started: 2026-04-17*

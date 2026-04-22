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

**Model first, then hardware.** The right build depends heavily on which model tier you target — see [`research/model-considerations.md`](research/model-considerations.md) for the tradeoffs. Key question:

> Is the goal **best quality at any speed** (large MoE, ~10 tok/s), or **best usability for Claude Code** (30–70B model, 100–500 tok/s)?

For mocking the Claude API interactively, a fast 30–70B coding model likely feels better than a slow 200B one.

### Factors still being weighed
- Model size tier (30B vs 70B vs 200B+ MoE)
- DIY server vs. turnkey appliance (DGX Spark)
- Used hardware risk tolerance (see [`research/risks.md`](research/risks.md))
- DDR5 RDIMM pricing (currently inflated ~2×; may normalize in 12–24 months)
- M5 Ultra Mac Studio expected ~WWDC June 2026

---

## Build Options

Full comparison in [`research/builds.md`](research/builds.md).

| Build | Cost | Memory | Est. Throughput | Best for |
|-------|------|--------|-----------------|----------|
| [DGX Spark](research/builds.md#option-0--dgx-spark-4699) | ~$4,699 | 128GB unified | ~480 tok/s (30B), ~38 tok/s (120B) | Turnkey, 30–70B models, quiet |
| [Learning (single 3090)](research/builds.md#option-1--learning-build-single-rtx-3090-2250-2680) | ~$2,250–2,680 | 24GB VRAM + 128GB RAM | 8–15 tok/s (200B MoE) | Start learning now, upgrade later |
| [Mid-tier DIY (Milan + 2× A6000)](research/builds.md#option-2--mid-tier-diy-used-milan--2-a6000-8-11k) | ~$8–11k | 96GB VRAM + 768GB RAM | 30–50 tok/s | Large models fully in VRAM, IPMI |
| [New build (Genoa + Pro 6000)](research/builds.md#option-3--new-build-genoa--pro-6000-blackwell-20-27k) | ~$20–27k | 96GB VRAM + 384GB DDR5 | 30–50 tok/s | All new; DDR5 currently overpriced |
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
- [`research/builds.md`](research/builds.md) — all build options compared, market timing
- [`research/risks.md`](research/risks.md) — used hardware risk analysis and verification checklists
- [`shopping/learning-build.md`](shopping/learning-build.md) — component sourcing for the learning build option

---

*Research started: 2026-04-17*

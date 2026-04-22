# Local LLM Hardware Purchasing

Research and shopping notes for building a headless Linux server to run **MiniMax M2.1** locally, mocking the Anthropic/Claude API endpoint for use with Claude Code.

---

## Status

| Phase | Status |
|-------|--------|
| Research builds + tradeoffs | ✅ Done |
| **Choose a build** | 🔄 In progress |
| Source components | ⬜ Not started |
| Purchase | ⬜ Not started |
| Build + burn-in | ⬜ Not started |
| Install software stack | ⬜ Not started |
| Test Claude Code integration | ⬜ Not started |

---

## Build Options

Full comparison in [`research/builds.md`](research/builds.md). Summary:

| Build | Cost | GPU VRAM | Est. Throughput | Notes |
|-------|------|----------|-----------------|-------|
| Learning (single 3090) | ~$2,250–2,680 | 24GB | 8–15 tok/s | Low risk, start learning now |
| Mid-tier (used Milan + 2× A6000) | ~$8–11k | 96GB | 30–50 tok/s | Q4/Q5 fully in VRAM |
| Full (new Genoa + Pro 6000 Blackwell) | ~$20–27k | 96GB | 30–50 tok/s | All new; DDR5 overpriced right now |
| Mac Studio M3 Ultra 256GB | ~$5.6k | 256GB unified | 20–25 tok/s | No IPMI, unavailable as of Apr 2026 |

**Key context:** MiniMax M2.1 is a 229B MoE model with ~10B active params/token, so it doesn't need as much VRAM as a dense 70B model. The learning build is not underpowered for this use case — it's a deliberate starting point. See [`research/model-requirements.md`](research/model-requirements.md) for the hardware math.

### Factors still being weighed
- Whether to start small and upgrade vs. buy once for the target capacity
- Used hardware risk tolerance (see [`research/risks.md`](research/risks.md))
- DDR5 RDIMM pricing (currently inflated ~2×; may normalize in 12–24 months)
- M5 Ultra Mac Studio expected ~WWDC June 2026 — could reset the used GPU market

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
- [`shopping/learning-build.md`](shopping/learning-build.md) — component sourcing links for the learning build option

---

*Research started: 2026-04-17*

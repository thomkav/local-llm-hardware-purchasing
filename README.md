# Local LLM Hardware Purchasing

Research and shopping notes for running an open-weight coding model locally to mock the Anthropic/Claude API endpoint for Claude Code. Model tier and build are **decided** as of 2026-04-22 — see [Decisions landed](#decisions-landed) below. Specific 30–70B model selection and purchase timing are still open.

## Intentions

Two users (one local in Portland, OR; one remote collaborator) both point their Claude Code installations at a shared inference server running in a basement rack. Goals: eliminate API costs for heavy Claude Code usage, keep code and context local, and share capacity without each needing separate hardware.

Claude Code's request pattern drives the hardware requirements more than raw tok/s: it sends large prompts (8k–32k tokens of repo context per request), which means **prefill speed (compute-bound)** matters as much as generation speed (bandwidth-bound). See [`setup/intentions.md`](setup/intentions.md) for full use case detail and [`setup/multi-user-access.md`](setup/multi-user-access.md) for concurrency patterns and the recommended vLLM + LiteLLM stack.

---

## Status

| Phase | Status |
|-------|--------|
| Research builds + tradeoffs | ✅ Done |
| Choose a model tier | ✅ 30–70B dense (2026-04-22) |
| Choose a build | ✅ DGX Spark primary + learning build on-ramp (2026-04-22) |
| **Pick a specific model** | 🔄 In progress — Qwen3 Coder 30B vs. Qwen3 72B vs. DeepSeek-R1 70B |
| Source components | ⬜ Not started |
| Purchase | ⬜ Not started |
| Build + burn-in | ⬜ Not started |
| Install software stack | ⬜ Not started |
| Test Claude Code integration | ⬜ Not started |

---

## Decisions landed

_Dated 2026-04-22. See the linked research files for the reasoning these summaries compress._

| Decision | Choice | One-line rationale |
|----------|--------|---------------------|
| **Model tier** | 30–70B dense | Claude Code is prefill-heavy on large repo contexts; MoE's breadth-across-domains advantage doesn't help a pure coding endpoint, and interactivity (100+ tok/s) beats capacity. See [`research/model-considerations.md`](research/model-considerations.md#the-practical-upshot-for-this-use-case). |
| **Production build** | DGX Spark ($4,699) | 128GB unified memory comfortably fits 30–70B at Q4–Q8, turnkey, no used-hardware risk, Blackwell compute wins on prefill for 8–32k-token prompts. See [`research/builds.md`](research/builds.md#option-0--dgx-spark-4699). |
| **Learning on-ramp** | Single-3090 DIY ($2,400–2,900) | Bought *first and separately* as a cheap hands-on rig for llama.cpp / vLLM / Ollama / LiteLLM familiarity. Not the production endpoint. See [`shopping/learning-build.md`](shopping/learning-build.md). |
| **Mid-tier DIY** | Plan B only | 2026 used-GPU prices ($10–12k for an A6000 pair alone) have narrowed the delta to Spark. Revisit only if a chosen 30–70B model proves inadequate and we commit to a 200B+ MoE. |
| **Mac Studio** | Side project, not production | Colleague's M4 Max via MLX ([`setup/mac-studio-mlx-remote.md`](setup/mac-studio-mlx-remote.md)) is a useful experiment but not on the purchase path. M5 Ultra slipped to October 2026 regardless. |

---

## Open Questions

The two big decisions (model tier, build) are resolved above. Remaining questions:

### Which specific 30–70B model
Candidates (all fit DGX Spark at Q4–Q8 — see [`research/model-considerations.md`](research/model-considerations.md#candidate-models)):
- **Qwen3 Coder 30B** — ~483 tok/s on Spark (FP8); fastest, most interactive.
- **Qwen3 72B** — stronger reasoning; ~150 tok/s on Spark; still well inside the 128GB ceiling at Q4.
- **DeepSeek-R1 70B** — reasoning-focused dense model.
- **Codestral 22B** — simpler, fast, fallback option.

Plan: run all four on the learning build before the Spark arrives to calibrate subjective preference on real Claude Code sessions.

### Less urgent / tracking only
- Used hardware risk tolerance for the learning build — see [`research/risks.md`](research/risks.md).
- DDR5 RDIMM pricing (moot unless we revisit the new-build Option 3).
- M5 Ultra Mac Studio slipped from WWDC June to ~October 2026 per [Bloomberg/Gurman, 2026-04-19](https://www.macworld.com/article/2973459/2026-mac-studio-m5-release-date-specs-price-rumors.html) (supply-chain snags). Does not affect the Spark plan.

---

## Build Options

Full comparison in [`research/builds.md`](research/builds.md).

Throughput varies significantly by model size — numbers below are illustrative per tier.

| Build | Cost | Memory | Throughput by model tier | Best for |
|-------|------|--------|--------------------------|----------|
| [DGX Spark](research/builds.md#option-0--dgx-spark-4699) | ~$4,699 | 128GB unified | ~480 tok/s (30B), ~150 tok/s (70B) | Turnkey, up to ~120B total (Q3 MiniMax M2.1 fits; Q4 does not) |
| [Learning (single 3090)](research/builds.md#option-1--learning-build-single-rtx-3090-2400-2900) | ~$2,400–2,900 | 24GB VRAM + 128GB RAM | ~60 tok/s (30B in VRAM), ~10 tok/s (200B MoE offloaded) | Start learning now; 30B sweet spot |
| [Mid-tier DIY (Milan + 2× A6000)](research/builds.md#option-2--mid-tier-diy-used-milan--2-a6000-14-18k) | ~$14–18k | 96GB VRAM + 768GB RAM | ~150 tok/s (70B), ~30–50 tok/s (200B MoE) | Large MoE fully in VRAM, IPMI, expandable |
| [New build (Genoa + Pro 6000)](research/builds.md#option-3--new-build-genoa--pro-6000-blackwell-20-27k) | ~$20–27k | 96GB VRAM + 384GB DDR5 | similar to mid-tier | All new; DDR5 currently overpriced |
| Mac Studio M3 Ultra 256GB | ~$5.6k | 256GB unified | 20–25 tok/s | Not recommended — no IPMI, unavailable |

_Prices last verified: 2026-04-22. Used GPU market has tightened sharply vs. 2025 estimates; A6000 pair alone now ~$10–12k used._

---

## Planned Software Stack

| Layer | Choice |
|-------|--------|
| OS | Ubuntu 24.04 Server (or DGX OS for Spark) |
| Inference engine | llama.cpp (CUDA) or NVIDIA NIM (Spark) |
| Model | TBD — see `research/model-considerations.md` |
| API proxy | Ollama's native `/v1/messages` (v0.14+) for the Ollama path, or LiteLLM's Anthropic-unified endpoint for the vLLM path. `claude-code-router` is no longer recommended (unmaintained). See [`setup/claude_code_any_model.md`](setup/claude_code_any_model.md). |

---

## Research Notes

- [`research/model-considerations.md`](research/model-considerations.md) — model tiers, dense vs MoE tradeoffs, candidate models
- [`research/model-requirements.md`](research/model-requirements.md) — VRAM/RAM requirements by model size and quantization (originally MiniMax-M2.1-focused; retained as MoE sizing reference)
- [`research/builds.md`](research/builds.md) — all build options compared, power costs, market timing
- [`research/risks.md`](research/risks.md) — used hardware risk analysis and verification checklists
- [`shopping/learning-build.md`](shopping/learning-build.md) — component sourcing for the learning build option

## Installation & Setup Notes

- [`setup/intentions.md`](setup/intentions.md) — full use case detail, what Claude Code actually does to an inference server
- [`setup/claude_code_any_model.md`](setup/claude_code_any_model.md) — step-by-step to point Claude Code at a local model on DGX Spark via Ollama (native Anthropic endpoint), vLLM + LiteLLM, or LM Studio. Triple-checked April 2026.
- [`setup/multi-user-access.md`](setup/multi-user-access.md) — concurrency patterns, throughput expectations, vLLM + LiteLLM stack for two users
- [`setup/basement-install.md`](setup/basement-install.md) — humidity mitigation, rack mounting options, networking (switch, Tailscale), cabling
- [`setup/mac-studio-mlx-remote.md`](setup/mac-studio-mlx-remote.md) — colleague's M4 Max Studio MLX + Tailscale setup (side project, not on the production path)
- [`setup/mac-studio-mlx-remote-for-colleague.md`](setup/mac-studio-mlx-remote-for-colleague.md) — same workflow in second person, shareable directly with the colleague

## External Spark Resources

- [`Spark_Resources/README.md`](Spark_Resources/README.md) — curated external GitHub projects relevant to the chosen DGX Spark production build (Qwen 3.5 122B-A10B INT4/AutoRound; vLLM-on-Spark Docker stack)

---

## Glossary

Domain-specific terms used across this repo. Universal acronyms (CPU, GPU, RAM, API, SSH, VPN, DNS) are omitted.

**LLM / inference**
- **LLM** — Large Language Model
- **MoE** — Mixture of Experts (model architecture with many sub-network "experts" + a learned router that picks a few per token; high total params, low active params per forward pass)
- **Dense model** — every parameter is active every forward pass (the non-MoE baseline)
- **KV cache** — Key-Value cache; per-request attention state stored while a prompt is being processed and a reply is being generated
- **Prefill** — the phase where the server reads the prompt (compute-bound)
- **Generation** — the phase where the server emits tokens one at a time (memory-bandwidth-bound)
- **tok/s** — tokens per second; the standard throughput unit
- **Q3 / Q4 / Q8** — 3-/4-/8-bit quantization of model weights (smaller = less memory, some quality loss)
- **FP4 / FP8 / FP16** — 4-/8-/16-bit floating-point precision
- **BF16** — Brain Floating-point 16-bit (ML-oriented FP16 variant with a wider exponent range)
- **GGUF** — GPT-Generated Unified Format; llama.cpp's on-disk model format
- **MLX** — Apple's machine-learning framework (native Apple Silicon inference path; not an acronym)
- **NIM** — NVIDIA Inference Microservices (NVIDIA's hosted inference runtime)

**Hardware**
- **VRAM** — Video RAM; the GPU's on-board memory
- **LPDDR5X** — Low-Power DDR5X; the soldered unified memory used in DGX Spark and Mac Studios
- **DDR4 / DDR5** — Double Data Rate 4th / 5th generation DIMM system memory
- **RDIMM** — Registered DIMM; buffered server-grade memory module
- **ECC** — Error-Correcting Code (memory with single-bit error detection + correction)
- **TDP** — Thermal Design Power; sustained watts the cooler must handle
- **PSU** — Power Supply Unit
- **NVMe** — Non-Volatile Memory Express; PCIe-attached SSD protocol
- **PCIe** — Peripheral Component Interconnect Express; the GPU/NVMe slot standard
- **SP3 / SP5** — AMD EPYC CPU sockets (Rome/Milan use SP3; Genoa uses SP5)
- **TR4-SP3** — Noctua cooler mount code compatible with both AMD Threadripper TR4 and EPYC SP3
- **PFLOP** — petaflop; 10¹⁵ floating-point operations per second
- **MSRP** — Manufacturer's Suggested Retail Price
- **DGX** — NVIDIA's data-center-class AI compute brand (DGX Spark is the desktop entry)

**Server management / networking**
- **IPMI** — Intelligent Platform Management Interface; out-of-band server management protocol (remote power cycle, console, sensors even when the OS is dead)
- **BMC** — Baseboard Management Controller; the chip on a server motherboard that implements IPMI
- **NIC** — Network Interface Card
- **VLAN** — Virtual LAN; switch feature to isolate traffic on a shared physical network

**Physical site (basement install)**
- **RH** — Relative Humidity
- **PGE** — Portland General Electric (the Portland, OR electric utility)
- **NEC** — National Electrical Code (US wiring standard; "80% continuous rule" is NEC 210.19)
- **U (as in 12U rack)** — rack unit; 1.75" of vertical rack height

**Miscellaneous**
- **WWDC** — Apple's Worldwide Developers Conference (annual June event)

---

*Research started: 2026-04-17*

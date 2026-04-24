# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo shape

This is a **documentation-only** repository — research notes and shopping lists in Markdown. There is no code, no build, no tests, no package manager. All work here is reading, editing, and cross-linking `.md` files. Don't search for `package.json`, `Makefile`, etc. — they don't exist.

See [`README.md`](README.md) for project overview, current status, and the top-level build comparison table.

## Key files

- [`README.md`](README.md) — status, open questions, build comparison table (human-readable overview, kept in sync with this file)
- [`research/model-considerations.md`](research/model-considerations.md) — model tiers, dense vs MoE tradeoffs, candidate models
- [`research/model-requirements.md`](research/model-requirements.md) — VRAM/RAM requirements by model size and quantization
- [`research/builds.md`](research/builds.md) — all build options compared (DGX Spark, learning, mid-tier, full)
- [`research/risks.md`](research/risks.md) — used hardware risk analysis and verification checklists
- [`shopping/learning-build.md`](shopping/learning-build.md) — component sourcing for the learning build option, with per-component Status fields
- [`setup/intentions.md`](setup/intentions.md) — full use case detail; what Claude Code actually does to an inference server
- [`setup/multi-user-access.md`](setup/multi-user-access.md) — concurrency patterns, vLLM + LiteLLM stack for two users
- [`setup/basement-install.md`](setup/basement-install.md) — humidity, rack mounting, networking, cabling for Portland unfinished basement
- [`setup/mac-studio-mlx-remote.md`](setup/mac-studio-mlx-remote.md) — colleague's M4 Max Studio: MLX inference + Tailscale remote access (side project, not part of Thomas's purchase)
- [`setup/mac-studio-mlx-remote-for-colleague.md`](setup/mac-studio-mlx-remote-for-colleague.md) — same workflow rewritten in second person, shareable directly with the colleague
- [`setup/claude_code_any_model.md`](setup/claude_code_any_model.md) — canonical step-by-step for pointing Claude Code at Ollama, vLLM, or LM Studio on DGX Spark (triple-checked April 2026)
- [`Spark_Resources/README.md`](Spark_Resources/README.md) — curated external GitHub projects for the chosen DGX Spark build (Qwen 3.5 122B-A10B INT4/AutoRound model, vLLM-on-Spark Docker stack)

## Project context

- **Goal:** run an open-weight coding model locally to serve as a Claude-Code-compatible endpoint, shared between two users (one local in Portland, one remote).
- **Request pattern drives hardware:** Claude Code sends large prompts (8k–32k tokens of repo context), so **prefill speed (compute-bound)** matters as much as generation speed (bandwidth-bound). Keep this tension visible when evaluating builds.
- **Model tier: decided (2026-04-22) — 30–70B dense.** MiniMax M2.1 was the original reference model but is no longer the target; the repo landed on interactive 30–70B dense over max-capability 200B+ MoE because Claude Code is prefill-heavy and code-focused. Specific model (Qwen3 Coder 30B vs. Qwen3 72B vs. DeepSeek-R1 70B) is still open. See `README.md` "Decisions landed."
- **Build: decided (2026-04-22) — DGX Spark primary, learning build as on-ramp.** Spark fits 30–70B at Q4–Q8 under its 128GB ceiling. Learning build (single RTX 3090 + EPYC, ~$2,400–2,900 as of April 2026) is bought first and separately for hands-on llama.cpp / vLLM / Ollama / LiteLLM familiarity. Mid-tier DIY is plan B only if a chosen model later proves inadequate and we commit to 200B+ MoE.
- **Current phase:** pick a specific 30–70B model, then purchase. Not yet purchasing.

## When updating status

Two places stay in sync as decisions land and components are purchased:

1. The status table and build comparison in [`README.md`](README.md) — update as phases complete or a build is chosen.
2. Per-component **Status** fields in [`shopping/learning-build.md`](shopping/learning-build.md) — update with actual prices paid when items are ordered.

If you add a new `.md` file under `research/`, `setup/`, or `shopping/`, add it to the Key files list above *and* link it from the relevant section of `README.md` so both entry points stay discoverable.

# Local LLM Hardware Purchasing

## Project Goal
Build a headless Linux server to run MiniMax M2.1 locally, virtualizing the Anthropic/Claude API endpoint for use with Claude Code.

## Current Decision
**Learning build** (~$2,200–2,750) — single GPU, 128GB RAM, EPYC Rome, learn the stack before committing to bigger hardware.

## Key Files
- `research/model-requirements.md` — MiniMax M2.1 specs and quant options
- `research/builds.md` — full build comparisons (learning, mid-tier, full)
- `research/components.md` — sourced component links and prices
- `research/risks.md` — used hardware risk analysis
- `shopping/learning-build.md` — active shopping list with current prices and decisions

## Context
- Date research started: 2026-04-17
- Primary model: MiniMax M2.1 (229B MoE, ~10B active params/token)
- Use case: mock Anthropic endpoint for Claude Code
- Headless server, so no Mac (no IPMI, locked RAM, Apple premium not justified)

## Software Stack (planned)
- Ubuntu 24.04 Server
- llama.cpp (best MoE support, runs Unsloth GGUFs)
- claude-code-router (proxy Claude Code → OpenAI-compatible endpoint)
- Start quant: UD-Q4_K_XL (~130GB)

## Current Status
- [ ] Finalize learning build component list
- [ ] Source / price-check all components
- [ ] Purchase
- [ ] Build and burn-in
- [ ] Install software stack
- [ ] Test Claude Code integration

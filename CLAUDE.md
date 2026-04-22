# Local LLM Hardware Purchasing — Claude Context

See `README.md` for project overview, current status, and component decisions.

## Key Files
- `README.md` — status, decisions, component table (human-readable overview)
- `research/model-requirements.md` — MiniMax M2.1 specs and quant options
- `research/builds.md` — full build comparisons (learning, mid-tier, full)
- `research/risks.md` — used hardware risk analysis and verification checklists
- `shopping/learning-build.md` — active shopping list with sourcing links and budget tracker

## Context for Claude
- Primary model: MiniMax M2.1 (229B MoE, ~10B active params/token)
- Use case: mock Anthropic endpoint for Claude Code via claude-code-router
- Headless Linux server (IPMI required — rules out Mac)
- Current phase: sourcing components for the learning build

## When updating status
Update the status table in `README.md` as components are purchased and build phases complete.
Update per-component "Status" fields in `shopping/learning-build.md` with actual prices paid.

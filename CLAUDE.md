# Local LLM Hardware Purchasing — Claude Context

See `README.md` for project overview, current status, and component decisions.

## Key Files
- `README.md` — status, open questions, build comparison table (human-readable overview)
- `research/model-considerations.md` — model tiers, dense vs MoE tradeoffs, candidate models
- `research/builds.md` — all build options compared (DGX Spark, learning, mid-tier, full)
- `research/risks.md` — used hardware risk analysis and verification checklists
- `shopping/learning-build.md` — component sourcing for the learning build option

## Context for Claude
- Goal: run an open-weight coding model locally to mock the Anthropic endpoint for Claude Code
- Model choice is open — see model-considerations.md
- Build choice is open — DGX Spark and DIY Linux server both in consideration
- MiniMax M2.1 was the original reference model but is not a firm target
- Current phase: deciding model tier + build simultaneously

## When updating status
Update the status table in `README.md` as components are purchased and build phases complete.
Update per-component "Status" fields in `shopping/learning-build.md` with actual prices paid.

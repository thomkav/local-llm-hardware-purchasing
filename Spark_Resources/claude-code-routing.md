# Routing Claude Code to a Local Model on DGX Spark

> **This page has moved.** The canonical step-by-step setup guide — for Ollama, vLLM, and LM Studio on DGX Spark — now lives at **[`setup/claude_code_any_model.md`](../setup/claude_code_any_model.md)**.
>
> The previous content at this path was superseded after a triple-check surfaced three corrections: (1) **Ollama v0.14+** ships a native Anthropic `/v1/messages` endpoint, so the Ollama path needs **no LiteLLM proxy**; (2) vLLM upstream does **not** officially support Spark's GB10 / SM_121 yet, so a community build is required; (3) a known Claude-Code-hangs-on-Ollama bug ([#51239](https://github.com/anthropics/claude-code/issues/51239)) needs to be flagged. All three are addressed in the canonical guide.

For the list of Spark-optimized external projects (the original purpose of this folder), see [`Spark_Resources/README.md`](README.md).

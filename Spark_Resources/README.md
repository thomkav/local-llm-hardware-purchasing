# Spark Resources

External projects and references relevant to running LLMs on NVIDIA DGX Spark. Curated after this repo's production build landed on Spark (see main [README](../README.md#decisions-landed)).

> Setup how-to lives in [`setup/claude_code_any_model.md`](../setup/claude_code_any_model.md). This folder is reserved for external/third-party projects we reference; in-repo setup guides belong in `setup/`.

---

## Models

### [albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4](https://github.com/albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4)

Qwen 3.5 122B-A10B (Mixture of Experts, ~10B active parameters per token) quantized and tuned specifically for DGX Spark. Uses **INT4 + FP8 hybrid layers** via Intel's **AutoRound** framework (the "AR" in the repo name), combined with **MTP-2 speculative decoding** and an **INT8 LM head**. Reports a **28.3 → 51 tok/s (+80%)** uplift over the baseline Spark inference path. Last updated 2026-04-14.

**Relevance to this repo.** Spark-compatible large MoE that slips under the 128GB ceiling thanks to aggressive quantization. Interesting **Plan B** if the chosen 30–70B dense model (see [open questions](../README.md#which-specific-3070b-model)) underwhelms on real Claude Code sessions — the A10B active-params count keeps latency roughly dense-30B-like while giving access to ~120B of total capacity.

### [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker)

Docker configuration and orchestration scripts for running **vLLM inference clusters on DGX Spark** (single- or multi-node). Ships pre-built wheels, model recipes, and deployment automation; supports **tensor, pipeline, and data parallelism** across multiple Sparks linked by the high-speed interconnect. Last commit 2026-04-23.

**Relevance to this repo.** Directly operationalizes the planned [vLLM + LiteLLM software stack](../setup/multi-user-access.md#inference-backend-vllm) on the chosen Spark build. Useful as (a) a ready-made Dockerfile + compose baseline when standing up the production endpoint, and (b) a starting template if a second Spark is ever added via NVLink to double memory to 256GB.

# Claude Code with Any Local Model on DGX Spark

Step-by-step setup for pointing **Claude Code** (Anthropic's CLI) at a local open-weight model running on **DGX Spark**, using either **Ollama** or **vLLM** as the inference backend. Content is triple-checked against the official Anthropic, Ollama, vLLM, and LiteLLM docs as of April 2026.

---

## TL;DR — three paths, pick one

| Path | Complexity | Concurrency | Best for |
|------|-----------|-------------|----------|
| **A. Ollama (native Anthropic endpoint)** | Lowest — 3 env vars, no proxy | Sequential only | **Default starting point on Spark** |
| **B. vLLM + LiteLLM proxy** | Higher — community Spark build required, proxy config | Continuous batching (true concurrency) | Production / multi-user once Path A proves the stack |
| **C. LM Studio (native Anthropic endpoint)** | Low — GUI-first | Sequential | GUI lovers; weaker Spark/ARM64 support |

**Recommendation for a fresh Spark**: start with **Path A (Ollama)**, validate Claude Code runs tool calls end-to-end, then optionally migrate to Path B if throughput or the known hang-bug forces the move.

---

## Critical context before you start (April 2026)

These are the facts that shape every step below. Don't skip them.

1. **Claude Code speaks the Anthropic Messages API only** — `POST /v1/messages`. It cannot call OpenAI `/v1/chat/completions` directly. ([Anthropic LLM Gateway docs](https://code.claude.com/docs/en/llm-gateway))
2. **Ollama v0.14.0+ ships a native Anthropic Messages endpoint** at `/v1/messages`. Claude Code can talk to it directly — **no proxy needed**. ([Ollama Anthropic compat docs](https://docs.ollama.com/api/anthropic-compatibility), [Ollama Claude Code guide](https://docs.ollama.com/integrations/claude-code))
3. **vLLM does NOT ship an Anthropic endpoint.** It only speaks OpenAI. You need LiteLLM (or equivalent) in front of vLLM to translate for Claude Code.
4. **vLLM upstream does NOT officially support DGX Spark's GB10 / SM_121 / ARM64** as of April 2026. You must build from a community fork or use a Spark-specific Docker image. ([vLLM issue #35519 re: NVFP4 on ARM64](https://github.com/vllm-project/vllm/issues/35519), [Spark install walkthrough](https://medium.com/@stablehigashi/vllm-installation-on-dgx-spark-gb10-sm-121-and-qwen-3-5-serving-guide-9eba91e448f8))
5. **Ollama on DGX Spark just works** — users report `gpt-oss:120b` at ~44 tok/s on Spark via Ollama. This is why Path A is the recommended starting point.
6. **Two known bugs to be aware of:**
   - **OAuth override gotcha** ([anthropics/claude-code#23022](https://github.com/anthropics/claude-code/issues/23022)): if you have a Claude Max/Pro subscription, setting `ANTHROPIC_BASE_URL` silently overrides OAuth. You'll see "404 model not found." **Fix: always set `ANTHROPIC_API_KEY=""` (empty) alongside `ANTHROPIC_BASE_URL`** — forces a clean switch.
   - **Claude Code hangs on Ollama Anthropic endpoint** ([anthropics/claude-code#51239](https://github.com/anthropics/claude-code/issues/51239)): trivial prompts can wedge Claude Code's loop even when raw `curl /v1/messages` works fine. If you hit this, drop to Path B.
7. **Claude Code needs ≥ 64k context** per Ollama's guidance. Most modern coding models (Qwen3-Coder-30B, etc.) support this by default; don't override lower.

---

## Prerequisites

- **DGX Spark** running DGX OS (Ubuntu-based ARM64 variant)
- SSH and sudo access
- **Claude Code** installed — confirm with `claude --version`. If not installed, follow the [official install guide](https://docs.claude.com/en/docs/claude-code/overview)
- 60+ GB free disk (models are 18–80 GB each)
- Spark on wired ethernet with stable internet (model pulls run 10–45 min)
- A scratch directory, e.g. `~/llm/`

---

## Path A — Ollama (recommended starting point)

### A1. Install Ollama on Spark

- [ ] Run the one-line installer:
  ```bash
  curl -fsSL https://ollama.com/install.sh | sh
  ```
- [ ] Confirm version is **≥ 0.14.0** (native Anthropic support landed in 0.14):
  ```bash
  ollama --version
  ```
  If older, rerun the installer — older Ollama versions only expose `/v1/chat/completions` and won't work with Claude Code's Anthropic-only client.
- [ ] Verify the daemon is listening:
  ```bash
  systemctl status ollama
  curl http://127.0.0.1:11434/api/version
  ```

### A2. Pull and warm up a model

- [ ] Pull Qwen3-Coder-30B (strong tool calling, well-supported on Spark):
  ```bash
  ollama pull qwen3-coder:30b
  ```
  Expect 10–20 minutes depending on bandwidth. About 18–22 GB on disk at default quant.
- [ ] Warmup run — loads weights into memory and confirms generation works:
  ```bash
  ollama run qwen3-coder:30b "Write a Python function that reverses a string."
  ```
  Watch the tok/s line that Ollama prints on exit. Expect 40–80 tok/s on Spark for a 30B model.

**Alternate coding models** recommended by Ollama for Claude Code:
- `qwen3-coder:30b` — default pick
- `glm-4.7` — strong general coder
- `minimax-m2.1` — larger MoE; needs Q3 to fit Spark's 128GB (see [`research/model-requirements.md`](../research/model-requirements.md))

### A3. Verify the Anthropic `/v1/messages` endpoint works

This step catches 90% of setup problems before you drag Claude Code into the loop.

- [ ] Direct curl against Ollama's Anthropic endpoint:
  ```bash
  curl http://127.0.0.1:11434/v1/messages \
    -H "Content-Type: application/json" \
    -H "x-api-key: ollama" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "qwen3-coder:30b",
      "max_tokens": 256,
      "messages": [{"role":"user","content":"Say hello in one sentence."}]
    }'
  ```
- [ ] Expected response: a JSON object with `"type": "message"`, `"role": "assistant"`, and a `content` array. Anthropic-shaped, not OpenAI-shaped.
- [ ] If this returns 404 or an OpenAI-shaped response, your Ollama is too old — see A1.

### A4. Configure Claude Code

Pick one scope.

**Global (all projects on this machine)** — add to `~/.bashrc` or `~/.zshrc`:
```bash
export ANTHROPIC_BASE_URL=http://localhost:11434
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=""    # CRITICAL — clears OAuth; see bug #23022
```
Reload: `source ~/.bashrc`.

**Persistent via Claude Code settings** — create or edit `~/.claude/settings.json`:
```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:11434",
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_API_KEY": ""
  }
}
```

**Per-project (mixed use: keep hosted Claude for most projects, local for one)** — in the project root create `.claude/settings.local.json` with the same JSON as above, then add `.claude/settings.local.json` to `.gitignore`.

### A5. End-to-end smoke test

- [ ] In a separate terminal, go to any project directory:
  ```bash
  cd ~/some-project
  claude
  ```
- [ ] Type a prompt that exercises tool calls:
  ```
  Read README.md and list the top-level headings.
  ```
- [ ] Expected flow:
  1. Claude Code sends an Anthropic request to Ollama
  2. Ollama responds with a tool-call to `Read`
  3. Claude Code executes `Read`
  4. The model summarizes the headings
  5. Loop ends, prompt returns
- [ ] Success = headings appear correctly.

**If Claude Code hangs** (no output, no progress, no error): you've hit [bug #51239](https://github.com/anthropics/claude-code/issues/51239). Ollama's Anthropic layer occasionally doesn't interact cleanly with Claude Code's streaming parser. Proceed to **Path B**.

---

## Path B — vLLM + LiteLLM proxy (higher throughput, Spark-specific build)

Use this when:
- Path A hangs or produces JSON errors
- You need true concurrent serving for two or more Claude Code users
- You want maximum tok/s

### B1. Install vLLM for Spark's SM_121 architecture

**This is the step that trips people up.** Upstream `pip install vllm` gives you wheels built for SM_80/90 (Ampere/Hopper) but **not** SM_121 (Spark's GB10 Blackwell). You need a Spark-compatible build.

**Option 1 (recommended) — `eugr/spark-vllm-docker`:**
Pre-built wheels and Dockerfile tuned for Spark, plus model recipes. See the repo's README:
```bash
cd ~
git clone https://github.com/eugr/spark-vllm-docker
cd spark-vllm-docker
# follow the repo README for docker compose up
```

**Option 2 — manual build from source:**
Follow [Stablehigashi's Spark install walkthrough](https://medium.com/@stablehigashi/vllm-installation-on-dgx-spark-gb10-sm-121-and-qwen-3-5-serving-guide-9eba91e448f8). Expect 30–60 min of compile time.

**Option 3 — NVIDIA's Spark vLLM page:**
Check [NVIDIA Build → Spark → vLLM](https://build.nvidia.com/spark/vllm) for any officially-supported container that matches your DGX OS version.

- [ ] Confirm vLLM can see Spark's GPU:
  ```bash
  python -c "import torch; print(torch.cuda.get_device_name(0))"
  # Expected output mentions "GB10" or "Blackwell"
  ```

### B2. Launch vLLM with tool calling

- [ ] Start the server. The flags matter — especially `--tool-call-parser qwen3_coder`, which is what makes tool-call output match Claude Code's expectations.
  ```bash
  vllm serve Qwen/Qwen3-Coder-30B-A3B-Instruct \
    --port 8000 \
    --host 127.0.0.1 \
    --gpu-memory-utilization 0.85 \
    --max-model-len 65536 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder
  ```
  - `--enable-auto-tool-choice` — lets the model decide when to call tools (required for Claude Code's agent loop)
  - `--tool-call-parser qwen3_coder` — exact parser for Qwen3-Coder family. For other model families: `hermes` (Qwen2.5 and Mistral variants), `llama3_json` (Llama 3.x), `mistral` (Mistral). See the [vLLM tool-calling docs](https://docs.vllm.ai/en/stable/features/tool_calling/).
  - `--max-model-len 65536` — 64k context, meets Claude Code's minimum. Raising to 128k roughly doubles KV-cache RAM.

- [ ] Run in `tmux` so it survives SSH drops:
  ```bash
  tmux new -s vllm
  # paste the vllm serve command, then Ctrl-b d to detach
  ```

- [ ] Smoke test the OpenAI endpoint:
  ```bash
  curl http://127.0.0.1:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model":"Qwen/Qwen3-Coder-30B-A3B-Instruct",
      "messages":[{"role":"user","content":"Hi"}]
    }'
  ```

### B3. Install LiteLLM proxy

- [ ] Install in a venv:
  ```bash
  python3 -m venv ~/llm/litellm-venv
  source ~/llm/litellm-venv/bin/activate
  pip install "litellm[proxy]"
  ```

- [ ] Create `~/llm/litellm-config.yaml`. Note: **use `hosted_vllm/` prefix, not `openai/`** — `hosted_vllm/` is LiteLLM's dedicated vLLM provider and handles parameter mapping better.
  ```yaml
  model_list:
    - model_name: "claude-*"
      litellm_params:
        model: "hosted_vllm/Qwen/Qwen3-Coder-30B-A3B-Instruct"
        api_base: "http://127.0.0.1:8000/v1"

  general_settings:
    master_key: "sk-REPLACE-WITH-KEY"
  ```
  The `"claude-*"` wildcard catches every Claude model string Claude Code sends (`claude-sonnet-4-5-20250929`, `claude-opus-4-7`, etc.) and routes all of them to your local vLLM.

- [ ] Generate a master key and paste it into the config above:
  ```bash
  openssl rand -hex 24
  ```

### B4. Launch LiteLLM

- [ ] Start in another `tmux` session:
  ```bash
  tmux new -s litellm
  source ~/llm/litellm-venv/bin/activate
  litellm --config ~/llm/litellm-config.yaml --port 4000 --host 127.0.0.1
  # Ctrl-b d to detach
  ```

- [ ] Verify the Anthropic-unified endpoint works:
  ```bash
  curl http://127.0.0.1:4000/v1/messages \
    -H "Content-Type: application/json" \
    -H "x-api-key: sk-YOUR-MASTER-KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model":"claude-sonnet-4-5-20250929",
      "max_tokens":256,
      "messages":[{"role":"user","content":"Hi"}]
    }'
  ```
  Anthropic-shaped response = the translation chain works. See [LiteLLM Anthropic-unified docs](https://docs.litellm.ai/docs/anthropic_unified/).

### B5. Configure Claude Code

Same env vars as Path A, but pointing at LiteLLM (port 4000) with your generated key:
```bash
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_AUTH_TOKEN=sk-YOUR-MASTER-KEY
export ANTHROPIC_API_KEY=""
```

Or in `~/.claude/settings.json`:
```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:4000",
    "ANTHROPIC_AUTH_TOKEN": "sk-YOUR-MASTER-KEY",
    "ANTHROPIC_API_KEY": ""
  }
}
```

### B6. Smoke test

Identical to Path A Step A5. If tool calls complete cleanly you're done. If you see JSON parse errors mid-loop, see the quantization note in [`research/model-considerations.md`](../research/model-considerations.md#how-quantization-affects-programming-outcomes) — tool-call JSON validity is the agent-loop tripwire.

---

## Path C — LM Studio (GUI-first alternative)

LM Studio v0.4.1+ ships a native `/v1/messages` endpoint, so no proxy is needed. Tradeoff: LM Studio's Spark/ARM64 support trails x86 Linux and macOS, so confirm your LM Studio build matches DGX OS before committing.

Quick sketch (full walkthrough: [LM Studio blog post](https://lmstudio.ai/blog/claudecode)):
1. Install LM Studio, download a model (e.g. Qwen3-Coder-30B)
2. Server tab → "Enable CORS" + "Start server" (binds `0.0.0.0:1234` by default)
3. Set Claude Code env vars:
   ```bash
   export ANTHROPIC_BASE_URL=http://localhost:1234
   export ANTHROPIC_AUTH_TOKEN=lm-studio
   export ANTHROPIC_API_KEY=""
   ```
4. Run `claude` — done.

---

## Multi-user / remote access

Layer Tailscale + LiteLLM virtual keys per [`multi-user-access.md`](multi-user-access.md) once single-user works. That doc assumes LiteLLM is already running (Path B). If you're on Path A (Ollama alone), you'd need to add LiteLLM in front of Ollama to get per-user keys and rate-limits — Ollama's native Anthropic endpoint has no auth beyond the literal `ANTHROPIC_AUTH_TOKEN=ollama` shared string.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Claude Code: `404 model not found` | OAuth conflict with base URL override | Set `ANTHROPIC_API_KEY=""` explicitly alongside `ANTHROPIC_BASE_URL`. [#23022](https://github.com/anthropics/claude-code/issues/23022) |
| Claude Code hangs on trivial prompt (Path A) | Ollama/Anthropic interaction bug | Move to Path B. [#51239](https://github.com/anthropics/claude-code/issues/51239) |
| "Tool call JSON parse error" mid-loop | Quant too aggressive OR tool parser missing/wrong | Raise quant to Q5/Q6 *or* ensure `--tool-call-parser qwen3_coder` on vLLM |
| `CUDA illegal instruction` on model load (vLLM Spark) | NVFP4 kernel + ARM64 incompatibility | Use a non-NVFP4 quant (Q4_K_M, Q8, FP16/BF16). [vLLM #35519](https://github.com/vllm-project/vllm/issues/35519) |
| `ImportError` / vLLM refuses to start on Spark | SM_121 not in upstream wheels | Use `eugr/spark-vllm-docker` or build from source |
| OOM at long context | KV cache > free memory | Lower `--max-model-len` (start at 32k, not 128k) |
| Ollama slow / stuck at load | Model not fully downloaded | `ollama pull <model>` to re-verify; check `~/.ollama/models` disk space |
| LiteLLM returns 500 on tool calls | Version mismatch with vLLM parser | Pin LiteLLM ≥ latest stable; confirm `hosted_vllm/` prefix (not `openai/`) in config |

---

## Verification checklist (done when all boxes ticked)

- [ ] Raw `curl /v1/messages` against the backend returns an Anthropic-shaped response
- [ ] `claude` completes a round-trip prompt that includes at least one tool call (Read, Grep, or Edit) without JSON errors
- [ ] Observed tok/s matches expectation for the chosen model + backend (Qwen3-Coder-30B ≈ 40–80 tok/s on Spark)
- [ ] The backend survives a `tmux` detach / SSH drop
- [ ] `~/.claude/settings.json` (or project `.claude/settings.local.json`) explicitly sets all three env vars including empty `ANTHROPIC_API_KEY`
- [ ] Both Claude Code sessions on Spark *and* any separate laptop (if using Tailscale) hit the same endpoint successfully

---

## References (all triple-checked 2026-04-23)

Official docs:
- [Claude Code LLM Gateway Configuration](https://code.claude.com/docs/en/llm-gateway) — env vars
- [Claude Code + vLLM integration](https://docs.vllm.ai/en/stable/serving/integrations/claude_code/)
- [Ollama Claude Code integration](https://docs.ollama.com/integrations/claude-code)
- [Ollama Anthropic compatibility](https://docs.ollama.com/api/anthropic-compatibility) — native `/v1/messages`
- [vLLM Tool Calling](https://docs.vllm.ai/en/stable/features/tool_calling/) — parser flags
- [vLLM OpenAI-compatible server](https://docs.vllm.ai/en/stable/serving/openai_compatible_server/)
- [Qwen3-Coder vLLM recipe](https://docs.vllm.ai/projects/recipes/en/latest/Qwen/Qwen3-Coder-480B-A35B.html)
- [LiteLLM Anthropic Unified endpoint](https://docs.litellm.ai/docs/anthropic_unified/)
- [LiteLLM vLLM provider](https://docs.litellm.ai/docs/providers/vllm)
- [LM Studio Claude Code guide](https://lmstudio.ai/blog/claudecode)

Spark-specific:
- [NVIDIA DGX Spark + vLLM](https://build.nvidia.com/spark/vllm)
- [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker) — recommended Docker image
- [Stablehigashi vLLM SM_121 install guide](https://medium.com/@stablehigashi/vllm-installation-on-dgx-spark-gb10-sm-121-and-qwen-3-5-serving-guide-9eba91e448f8)
- [DGX Spark LLM-stack thread (LiteLLM + llama-swap + vLLM + Ollama)](https://forums.developer.nvidia.com/t/running-a-full-llm-stack-on-dgx-spark-gb10-your-application-litellm-llama-swap-vllm-llama-cpp-ollama/367580)

Known bugs:
- [anthropics/claude-code#23022](https://github.com/anthropics/claude-code/issues/23022) — OAuth override
- [anthropics/claude-code#51239](https://github.com/anthropics/claude-code/issues/51239) — hang with Ollama Anthropic endpoint
- [vllm-project/vllm#35519](https://github.com/vllm-project/vllm/issues/35519) — NVFP4 crash on ARM64 Spark

Related in-repo:
- [`research/model-considerations.md`](../research/model-considerations.md) — model choice + quantization tradeoffs for coding
- [`setup/multi-user-access.md`](multi-user-access.md) — two-user extension with Tailscale + LiteLLM virtual keys
- [`Spark_Resources/README.md`](../Spark_Resources/README.md) — external Spark-optimized projects

# Mac Studio M4 Max — MLX Inference + Remote Access

> **Paired doc:** [`mac-studio-mlx-remote-for-colleague.md`](mac-studio-mlx-remote-for-colleague.md) is the same workflow rewritten in second person for the colleague to follow directly. Keep edits to commands, models, or energy/Tailscale settings in sync between the two.

Setup workflow for a colleague's new Mac Studio (M4 Max, 128GB unified memory). Goal: stand up an OpenAI-compatible inference endpoint on MLX that both of us can reach remotely via Tailscale, so we can collaborate on prompt/model experiments from two machines.

**Why MLX, not vLLM.** vLLM's wins (PagedAttention, CUDA graphs, continuous batching) are NVIDIA-specific. On Apple Silicon it either falls back to CPU or hits experimental MPS paths that are dramatically slower than native alternatives. MLX is Apple's own ML framework — it treats the unified memory as a first-class resource, uses the GPU through Metal, and ships `mlx_lm.server` with an OpenAI-compatible HTTP API. For single/low-concurrency use this is the right tool.

---

## Phase 1 — First boot of the Mac Studio (≈30 min)

- [ ] macOS setup assistant: Apple ID, Wi-Fi, time zone, FileVault on
- [ ] Run all pending updates: System Settings → General → Software Update (loop until clean)
- [ ] System Settings → Lock Screen: "Start Screen Saver when inactive" = Never; "Turn display off on power adapter when inactive" = Never
- [ ] System Settings → Energy: "Prevent automatic sleeping when display is off" = On; "Wake for network access" = On; "Start up automatically after a power failure" = On
- [ ] System Settings → General → Sharing: enable Remote Login (SSH). Note the `.local` hostname shown
- [ ] System Settings → General → Sharing → File Sharing: leave off unless needed
- [ ] Name the machine something predictable (e.g. `studio-<colleague>`) in Sharing → Computer Name

## Phase 2 — Dev tools (≈20 min)

- [ ] Xcode Command Line Tools:
  ```
  xcode-select --install
  ```
- [ ] Homebrew — run the one-liner from https://brew.sh, then follow its post-install notice to add `brew` to `PATH` (on Apple Silicon it's `/opt/homebrew/bin`)
- [ ] `uv` for Python env management:
  ```
  brew install uv
  ```
- [ ] (Optional) `git`, `gh`, `htop`, `tmux`:
  ```
  brew install git gh htop tmux
  ```

## Phase 3 — MLX-LM install (≈10 min)

- [ ] Create a project directory and virtualenv:
  ```
  mkdir -p ~/llm && cd ~/llm
  uv venv --python 3.12
  source .venv/bin/activate
  ```
- [ ] Install MLX-LM:
  ```
  uv pip install mlx-lm
  ```
- [ ] Sanity check:
  ```
  python -c "import mlx_lm; print(mlx_lm.__version__)"
  ```

## Phase 4 — Pull a model (≈15–45 min)

Recommended starting model for 128GB: **`mlx-community/Qwen2.5-Coder-32B-Instruct-8bit`**. ~35GB on disk / in RAM, current top-tier open-weight coding model, leaves plenty of headroom for context and other apps.

- [ ] Download via a warmup prompt (MLX auto-fetches to `~/.cache/huggingface/hub/`):
  ```
  mlx_lm.generate \
    --model mlx-community/Qwen2.5-Coder-32B-Instruct-8bit \
    --prompt "Write a Python function that reverses a string." \
    --max-tokens 256
  ```
- [ ] Check tokens/sec printed at the end — expect 15–30 tok/s on M4 Max for 32B Q8

Alternate models worth keeping around:
- `mlx-community/Qwen2.5-Coder-32B-Instruct-4bit` — ~18GB, a touch lower quality, noticeably faster
- `mlx-community/Meta-Llama-3.3-70B-Instruct-4bit` — ~40GB, different family for comparison
- `mlx-community/DeepSeek-Coder-V2-Lite-Instruct-8bit` — ~17GB, 16B params, fast iteration

## Phase 5 — Run the OpenAI-compatible server

- [ ] Start in a `tmux` session so it survives SSH drops:
  ```
  tmux new -s mlx
  cd ~/llm && source .venv/bin/activate
  mlx_lm.server \
    --model mlx-community/Qwen2.5-Coder-32B-Instruct-8bit \
    --host 0.0.0.0 \
    --port 8080
  ```
  Detach with `Ctrl-b d`. Reattach with `tmux attach -t mlx`.
- [ ] Local smoke test from another terminal on the Mac Studio:
  ```
  curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "qwen",
      "messages": [{"role": "user", "content": "Hello, who are you?"}]
    }'
  ```
- [ ] Bind to `0.0.0.0` (not `127.0.0.1`) is deliberate — needed so Tailscale peers can hit it. Keep the machine firewalled at the OS level; Tailscale handles the encrypted transport.

## Phase 6 — Tailscale for remote access (≈10 min)

- [ ] On Mac Studio:
  ```
  brew install --cask tailscale
  ```
  Launch it, sign in. Use either:
  - **Same tailnet** — colleague joins your existing Tailscale account (simplest)
  - **Node share** — colleague has their own tailnet, uses Tailscale's "Share node" feature to invite your account to a specific machine
- [ ] Grab the Tailscale IP from the Mac Studio: `tailscale ip -4` → note the `100.x.y.z`
- [ ] In Tailscale admin console, enable **MagicDNS** so you can use `http://studio-<colleague>:8080` instead of the raw IP
- [ ] From your laptop:
  ```
  curl http://100.x.y.z:8080/v1/models
  curl http://100.x.y.z:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen","messages":[{"role":"user","content":"ping"}]}'
  ```

## Phase 7 — Lock it down a little (≈5 min)

- [ ] Add a bearer token so the endpoint isn't wide-open to anyone else on the tailnet:
  ```
  mlx_lm.server --model ... --host 0.0.0.0 --port 8080 --api-key sk-<random>
  ```
  Generate with `openssl rand -hex 24`. Store in 1Password / shared vault.
- [ ] Clients pass it as `Authorization: Bearer sk-<random>`
- [ ] Tailscale ACLs: in the admin console, restrict which tailnet members can reach the Mac Studio on port 8080. Default-open is fine for a 2-person tailnet; tighten as others join.

## Phase 8 — Using it from your end

Short-term, just talk OpenAI directly:

```python
from openai import OpenAI
client = OpenAI(
    base_url="http://studio-<colleague>:8080/v1",
    api_key="sk-<random>",
)
resp = client.chat.completions.create(
    model="qwen",
    messages=[{"role": "user", "content": "Refactor this function: ..."}],
)
print(resp.choices[0].message.content)
```

To actually wire it to Claude Code (the stated project goal): MLX-LM speaks OpenAI, not Anthropic. You'll need a translation shim — **LiteLLM proxy** running on your laptop is the usual path. That's a follow-up, not part of this setup.

---

## Notes and gotchas

- **Concurrency.** `mlx_lm.server` processes requests serially. Two simultaneous callers queue. Fine for 2 devs collaborating; not a shared team service. If you outgrow it, look at `mlx-omni-server` or batching forks — or accept that the Mac Studio has a hard ceiling here (this is the real reason vLLM exists on NVIDIA).
- **First-token latency.** Model load happens on first request after server start — expect a 10–30s wait for a 32B. After that the weights stay resident and responses are fast.
- **Memory pressure.** macOS will start swapping if the colleague runs Chrome + Slack + Zoom on top of a 32B model loaded. Encourage closing heavy apps before serving, or watch Activity Monitor → Memory Pressure graph.
- **Sleep is the enemy.** Verify the energy settings in Phase 1 actually stuck. A Studio that sleeps mid-request drops the HTTP connection and forces another model reload on wake.
- **Thermals.** M4 Max Studio handles sustained inference well, but it will spin the fan audibly under load. Place it somewhere that's OK with that.
- **Persistent service (later).** When you're past "testing in tmux," wrap the server in a `launchd` agent at `~/Library/LaunchAgents/llm.mlx.plist` so it restarts automatically.

## Success criteria

You're done when:
1. Colleague has `mlx_lm.server` running on the Mac Studio, bound to `0.0.0.0:8080`, behind an API key
2. Tailscale is installed on both machines with MagicDNS enabled
3. From your laptop, `curl http://studio-<colleague>:8080/v1/chat/completions` returns a coding completion in under ~10s for a short prompt
4. Both of you can independently hit the endpoint and get matching outputs for the same prompt

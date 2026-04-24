# Mac Studio LLM Setup

> **Paired doc:** [`mac-studio-mlx-remote.md`](mac-studio-mlx-remote.md) is the same workflow written in first person for internal notes. Keep edits to commands, models, or energy/Tailscale settings in sync between the two.

A step-by-step for getting your new Mac Studio (M4 Max, 128GB) running as a local LLM inference server that Thomas can also reach over Tailscale. Plan on 1.5–2 hours end-to-end, most of it model download time you can walk away from.

**What you're building.** An HTTP endpoint on your machine that speaks the OpenAI API. Any tool that talks to OpenAI — Cursor, Continue.dev, raw `curl`, the OpenAI Python SDK — can point at it and run completions on a locally-hosted model. Thomas will reach the same endpoint remotely via Tailscale.

**Why MLX and not vLLM or Ollama.** MLX is Apple's native ML framework. It uses the Mac's unified memory properly and goes through Metal to the GPU cores. vLLM is NVIDIA-oriented and much slower on Apple Silicon. Ollama works fine but MLX is faster on your hardware and ships an OpenAI-compatible server out of the box.

---

## Phase 1 — First boot (≈30 min)

- [ ] Run through macOS setup: Apple ID, Wi-Fi, time zone, enable FileVault
- [ ] System Settings → General → Software Update: run updates until nothing's pending
- [ ] System Settings → Lock Screen:
  - "Start Screen Saver when inactive" = Never
  - "Turn display off on power adapter when inactive" = Never
- [ ] System Settings → Energy:
  - "Prevent automatic sleeping when display is off" = On
  - "Wake for network access" = On
  - "Start up automatically after a power failure" = On
- [ ] System Settings → General → Sharing:
  - Enable **Remote Login** (SSH)
  - Set **Computer Name** to something memorable like `studio-<yourname>` — this becomes your hostname on the network
- [ ] Confirm the machine stays awake: close the lid / walk away for 15 min, come back, make sure it hasn't slept

## Phase 2 — Dev tools (≈20 min)

- [ ] Install Xcode Command Line Tools (needed for lots of Python packages):
  ```
  xcode-select --install
  ```
- [ ] Install Homebrew: copy the one-liner from https://brew.sh and run it. When it finishes, it prints two commands to add `brew` to your `PATH` — run those.
- [ ] Confirm brew works:
  ```
  brew --version
  ```
- [ ] Install `uv` (fast Python environment manager) and a few conveniences:
  ```
  brew install uv git tmux htop
  ```

## Phase 3 — MLX-LM install (≈10 min)

- [ ] Set up a dedicated project directory with its own Python environment:
  ```
  mkdir -p ~/llm && cd ~/llm
  uv venv --python 3.12
  source .venv/bin/activate
  ```
- [ ] Install MLX-LM:
  ```
  uv pip install mlx-lm
  ```
- [ ] Verify the install:
  ```
  python -c "import mlx_lm; print(mlx_lm.__version__)"
  ```

## Phase 4 — Download a model (≈15–45 min, mostly waiting)

Start with **`mlx-community/Qwen2.5-Coder-32B-Instruct-8bit`** — a strong open-weight coding model, ~35GB on disk, fits comfortably in 128GB with plenty of room for context and the rest of macOS.

- [ ] Trigger the download by running a test generation. MLX pulls the weights to `~/.cache/huggingface/hub/` on first use:
  ```
  mlx_lm.generate \
    --model mlx-community/Qwen2.5-Coder-32B-Instruct-8bit \
    --prompt "Write a Python function that reverses a string." \
    --max-tokens 256
  ```
  When it finishes, note the tokens/sec printed at the end — expect 15–30 tok/s.

Other models you might want later (all from `mlx-community/` on Hugging Face):
- `Qwen2.5-Coder-32B-Instruct-4bit` — half the RAM (~18GB), slightly lower quality, faster
- `Meta-Llama-3.3-70B-Instruct-4bit` — different family, bigger (~40GB), good general-purpose
- `DeepSeek-Coder-V2-Lite-Instruct-8bit` — 16B params, fast, ~17GB

## Phase 5 — Run the inference server

The server keeps the model loaded in memory and exposes an OpenAI-compatible HTTP API.

- [ ] Start it in a `tmux` session so it survives you closing the terminal:
  ```
  tmux new -s mlx
  cd ~/llm && source .venv/bin/activate
  mlx_lm.server \
    --model mlx-community/Qwen2.5-Coder-32B-Instruct-8bit \
    --host 0.0.0.0 \
    --port 8080
  ```
  Detach from tmux: `Ctrl-b` then `d`. Reattach later: `tmux attach -t mlx`.
- [ ] In a **new** terminal window on the Mac Studio, test it:
  ```
  curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "qwen",
      "messages": [{"role": "user", "content": "Hello, who are you?"}]
    }'
  ```
  You should get back a JSON response with a completion. If you do — the server works.

Note: binding to `0.0.0.0` instead of `127.0.0.1` is intentional — it's what lets Tailscale peers reach the port. The macOS firewall + Tailscale's encrypted transport keep this safe.

## Phase 6 — Tailscale for remote access (≈10 min)

Tailscale creates an encrypted private network between your machines. Once set up, Thomas's laptop and your Mac Studio can reach each other by name, wherever either of you is.

- [ ] Install Tailscale:
  ```
  brew install --cask tailscale
  ```
- [ ] Launch the Tailscale app, sign in. Thomas will tell you whether to:
  - **Join his existing tailnet** (he sends an invite to your email), or
  - **Create your own tailnet** and share your Mac Studio node to him
- [ ] In Terminal, grab your Tailscale IP:
  ```
  tailscale ip -4
  ```
  Send Thomas the IP (looks like `100.x.y.z`). Also send him the hostname (`studio-<yourname>`).
- [ ] In the Tailscale admin console (https://login.tailscale.com/admin), enable **MagicDNS** — this lets Thomas reach you as `http://studio-<yourname>:8080` instead of using the raw IP.

## Phase 7 — Add an API key (≈5 min)

Right now anyone on the tailnet can hit your endpoint. Add a bearer token so only callers with the key can use it.

- [ ] Generate a random token:
  ```
  openssl rand -hex 24
  ```
- [ ] Restart the server with the key:
  ```
  tmux attach -t mlx
  # Ctrl-C to stop the running server, then:
  mlx_lm.server \
    --model mlx-community/Qwen2.5-Coder-32B-Instruct-8bit \
    --host 0.0.0.0 \
    --port 8080 \
    --api-key sk-<paste-the-random-hex-here>
  ```
- [ ] Send the key to Thomas via 1Password / Signal / whatever's secure — not email or Slack DM

## Phase 8 — Let Thomas test from his end

Ping Thomas with:
- Your Mac Studio's Tailscale hostname (`studio-<yourname>`) or IP
- The port (`8080`)
- The API key
- Model name (just `qwen` works — MLX-LM doesn't validate it strictly)

He'll run a test request. If it comes back with a completion, you're done with setup.

---

## Things to know

- **One request at a time.** MLX-LM processes requests serially. If you and Thomas both send at the same moment, the second one queues. This is fine for two people collaborating; not a team service.
- **First request is slow.** After starting the server, the first request takes 10–30s while the model loads into memory. Subsequent requests are fast — the model stays resident.
- **Memory pressure matters.** If you're running Chrome + Slack + Zoom + a 32B model, macOS will start swapping and everything gets slow. Check **Activity Monitor → Memory → Memory Pressure** — you want it green. If it's yellow/red, quit heavy apps.
- **The Studio stays awake, right?** Sanity-check the energy settings from Phase 1 actually stuck. A Studio that sleeps mid-request drops Thomas's HTTP connection and forces another model reload.
- **Fan noise under load.** Sustained inference will spin up the fan audibly. Place the Studio somewhere that's OK with that.

## When something breaks

- **Server won't start / out of memory**: try the 4-bit variant of the model instead (`Qwen2.5-Coder-32B-Instruct-4bit`). Or quit other apps and retry.
- **Thomas can't reach you**: have him run `tailscale ping studio-<yourname>` — if that fails, Tailscale isn't connected. If ping works but HTTP doesn't, the server isn't bound to `0.0.0.0` or isn't running.
- **Model download stalls**: it's just Hugging Face. Ctrl-C and rerun; it resumes.
- **tmux confusion**: `tmux ls` lists sessions. `tmux attach -t mlx` reattaches. `tmux kill-session -t mlx` nukes it.

## Done when

- `mlx_lm.server` is running in tmux, bound to `0.0.0.0:8080`, with an API key
- Tailscale is connected, MagicDNS works, Thomas is in the tailnet
- Thomas successfully hits the endpoint from his laptop and gets a completion back

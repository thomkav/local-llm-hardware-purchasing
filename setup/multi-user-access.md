# Multi-User Access Patterns

Two users, one inference server. This doc covers concurrency models, throughput expectations, and the recommended software stack.

> **Scope:** the vLLM + LiteLLM recommendation below applies to the **Linux/CUDA** production path (DGX Spark or DIY server). The Mac Studio / MLX path in [`mac-studio-mlx-remote.md`](mac-studio-mlx-remote.md) runs `mlx_lm.server`, which processes requests **sequentially** — not continuously batched. Two concurrent users on MLX will queue, not overlap. That is fine for a 2-person side project with bursty, non-overlapping use; it is not equivalent to the vLLM story below.

---

## How concurrent LLM requests work

An inference server processes requests as a sequence of **prefill** (reading the prompt) then **generation** (producing tokens). Two requests arriving simultaneously have a few possible outcomes depending on the server:

### Sequential queuing (naive)
Requests are processed one at a time. User B waits until User A's response is fully generated before their request starts. Simple but means one user can block the other for 10–60 seconds during a long generation.

### Continuous batching (preferred)
Modern inference servers (vLLM, TensorRT-LLM) process multiple requests concurrently by grouping token generation steps across requests into a single GPU pass. Both users are being served simultaneously — each at reduced tok/s — rather than one waiting. GPU utilisation is higher and overall system throughput is better.

llama.cpp supports `--parallel N` slots but splits the KV cache between them, which is less efficient than vLLM's PagedAttention approach. For two users, vLLM is the better choice.

---

## Throughput expectations: two concurrent users

Total system throughput is roughly fixed. With continuous batching, two concurrent users each get approximately half the single-user tok/s. In practice, because Claude Code requests are **bursty and not perfectly simultaneous**, contention is lighter than the worst case — requests often naturally interleave.

Rough estimates (generation tok/s per user, two concurrent):

| Build | Model | Single user | Two concurrent (each) | Notes |
|-------|-------|------------|----------------------|-------|
| DGX Spark | 30B dense | ~480 tok/s | ~240 tok/s | Plenty of headroom |
| DGX Spark | 70B dense | ~150 tok/s | ~75 tok/s | Still comfortable |
| Learning build | 30B dense | ~60 tok/s | ~30 tok/s | Getting slow for long responses |
| Mid-tier DIY | 70B dense | ~150 tok/s | ~75 tok/s | Comparable to DGX Spark |
| Mid-tier DIY | 200B MoE | ~40 tok/s | ~20 tok/s | Noticeable latency on long outputs |

**Prefill (time to first token) is the more likely bottleneck.** A 16k-token Claude Code prompt at prefill may take 5–15 seconds even on fast hardware. Two users prefilling simultaneously can double that wait. This is where the DGX Spark's Blackwell compute advantage over the learning build matters most.

---

## Recommended software stack for multi-user

### Inference backend: vLLM
[vLLM](https://github.com/vllm-project/vllm) is the right choice over llama.cpp for multi-user:
- Continuous batching with PagedAttention — efficient KV cache sharing across requests
- OpenAI-compatible API out of the box (what LiteLLM's Anthropic-unified endpoint wraps for Claude Code; `claude-code-router` is deprecated)
- Better throughput under concurrent load than llama.cpp's parallel slots

llama.cpp is fine for single-user or experimentation. Switch to vLLM when serving two users seriously.

### Proxy layer: LiteLLM
[LiteLLM](https://github.com/BerriAI/litellm) sits in front of vLLM and adds:
- **Per-user API keys** — each collaborator gets their own key; Claude Code is configured with their key
- **Rate limiting** — cap one user's requests/min so they can't monopolise the server
- **Usage tracking** — see who is generating how many tokens
- **Single endpoint** — one URL for both users regardless of backend changes

Setup: LiteLLM runs on the inference server, listens on a port (e.g. 4000), proxies to vLLM on localhost. Both users configure Claude Code to point at `http://<tailscale-ip>:4000`.

### Network: Tailscale
Both users install Tailscale. The server gets a stable `100.x.x.x` address. Remote collaborator adds themselves to the same Tailscale network (invite via the Tailscale admin console). No port-forwarding, no public IP exposure needed.

Access control: Tailscale ACLs (Access Control Lists) can restrict which devices can reach port 4000 on the server, so only your two machines can hit the inference endpoint.

### Summary stack

```
Claude Code (local)  ─┐
                       ├──► LiteLLM proxy :4000  ──►  vLLM :8000  ──►  GPU
Claude Code (remote) ─┘         (Tailscale)
```

---

## Practical capacity planning

For two Claude Code users doing active coding sessions:

- **DGX Spark + 70B model**: comfortable for two users. ~75 tok/s each during contention, fast prefill from Blackwell compute. Recommended if budget allows.
- **Learning build + 30B model**: workable but you'll feel contention during simultaneous long requests. Fine as a starting point; upgrade the GPU if it becomes frustrating.
- **Mid-tier DIY + 70B**: matches DGX Spark throughput with more flexibility and IPMI management, at higher cost and complexity.

If both users are frequently hitting the server simultaneously with large contexts (e.g. both running long agentic tasks at the same time), a 70B model on the learning build will feel slow. A DGX Spark or mid-tier build with a 70B model is the comfortable floor for two serious concurrent users.

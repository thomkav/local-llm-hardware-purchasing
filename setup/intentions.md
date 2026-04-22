# Intentions & Use Case

## What this system is for

Run an open-weight coding model on local hardware to serve as a drop-in replacement for the Anthropic Claude API, primarily for use with **Claude Code** (the CLI coding agent). Two users — a local user in Portland, OR and a remote collaborator — both point their Claude Code installations at the same inference endpoint.

Goals:
- Eliminate per-token API costs for heavy Claude Code usage
- Keep code and context local (no data sent to Anthropic)
- Learn the local inference stack end-to-end
- Share capacity between two collaborators without each needing their own hardware

## What Claude Code actually does to an inference server

Claude Code is an agentic coding tool. Its request pattern is different from a chat UI:

- **Large prompt prefill**: Claude Code sends the full contents of files, directory trees, and conversation history as a prompt on every request. A typical coding session with a medium-sized repo can be 8k–32k tokens of input per request.
- **Moderate output length**: Responses are usually 200–2000 tokens (code edits, explanations, shell commands). Rarely more than 4k.
- **Bursty, not streaming**: Requests come in clusters as the agent executes steps, then go quiet. Not a continuous stream.
- **Latency-sensitive prefill**: The time-to-first-token (TTFT) — driven by prefill speed — matters more than raw tok/s for interactive feel. A 10-second wait before the first token appears feels broken even if generation is fast after that.

This means **prefill throughput (compute-bound) matters as much as generation throughput (bandwidth-bound)** for this use case. It's one reason the Mac Studio's bandwidth advantage is partially offset — big contexts hammer the compute, not just the memory bus.

## Two-user access

Both users run Claude Code locally on their own machines. Both point at the same inference server endpoint (reachable via Tailscale). The server handles requests from both.

The remote collaborator accesses via Tailscale — same network, different physical location. From the server's perspective there is no meaningful difference between local and remote requests.

See [`setup/multi-user-access.md`](multi-user-access.md) for concurrency patterns, throughput expectations, and the recommended software stack for multi-user serving.

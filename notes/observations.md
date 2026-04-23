# Miscellaneous Observations

Anecdotes and informal reports that don't directly drive hardware decisions but are worth noting.

---

## DGX Spark — anecdotal early-adopter report (April 2026)

Source: secondhand via Yusuf, from a contact at Curtis Wright who owns a Spark unit.

- Running Qwen3 ~27B, observed ~10 tokens/sec generation.
- Contact noted the Spark feels "better built for MoE models."
- Power anomaly: unit draws ~20W under what should be load, then requires a reboot to recover normal behavior.
- Yusuf's read: setup is likely not optimal.

**Why this is interesting but not alarming:**
Our research benchmarks show ~480 tok/s on Qwen3 Coder 30B (FP8) from NVIDIA's own numbers. 10 tok/s on a similar-sized model is ~50x slower — almost certainly the model is not running on the GPU, or the inference backend isn't using cuBLAS/TensorRT-LLM. This is a setup problem, not a hardware ceiling. Consistent with early-adopter friction on a new architecture (GB10 Grace Blackwell) where third-party tools like Ollama weren't mature at launch.

The "better for MoE" observation is partly right in a narrow sense: 128GB unified memory lets large MoE models fit entirely in-pool. But the Spark's Blackwell compute advantage is most pronounced on prefill (compute-bound), which benefits dense and MoE equally.

The 20W / reboot issue is a separate concern — sounds like early firmware causing the unit to drop to a low-power idle it can't exit. Worth watching as a reliability signal on early units.

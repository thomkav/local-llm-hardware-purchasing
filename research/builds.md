# Build Comparisons

## Decision: Learning Build (chosen)
~$2,200–2,750. Single GPU, 128GB RAM, EPYC Rome. Learn the stack; upgrade later.

See `../shopping/learning-build.md` for active component list.

---

## Build A — Mid-Tier: Used Milan + 2× A6000 (~$8–11k)
Recommended if you need more capacity later.

- CPU: EPYC 7713P (64c Milan, SP3), used ~$600–900
- Motherboard: Supermicro H12SSL-i ~$700–900
- RAM: 12× 64GB DDR4-3200 RDIMM ECC (768GB) ~$2,400–4,800 used
- GPU: 2× used RTX A6000 48GB (~$6,000–9,000 pair)
- Chassis + PSU + storage ~$800–1,200
- **Total: ~$10–15k**
- Performance: Q4/Q5 fully in VRAM + offload, ~30–50 tok/s

Sources: TheServerStore, UNIXSurplus, Bargain Hardware (UK), ServerMonkey, r/hardwareswap

---

## Build B — New: Genoa + Pro 6000 Blackwell (~$20–27k)
Only justified if buying all new and needing maximum throughput.

- CPU: EPYC 9354P (32c Genoa, SP5) ~$1,500–2,400
- Motherboard: Supermicro H13SSL-N ~$650–800
- RAM: 12× 32GB DDR5-4800 RDIMM ECC (384GB) — currently ~$15–18k (DRAM shortage)
- GPU: 1× RTX Pro 6000 Blackwell 96GB ~$8,900
- **Total: ~$20–27k** (mostly RAM right now)
- Note: DDR5 RDIMM prices may normalize in 12–24 months

---

## Mac Studio (rejected for this use case)
- M3 Ultra 256GB (~$5.6k when available) — 819GB/s bandwidth
- M4 Max 128GB (~$3.5k) — caps at 128GB, less headroom
- Rejected reasons: no IPMI, soldered RAM (not expandable), slower prompt ingestion vs CUDA, Apple premium
- Note: M4 Ultra skipped by Apple. M5 Ultra expected ~WWDC June 2026.
- As of April 2026: both 128GB and 256GB configs "currently unavailable," 4–5 month wait

---

## Market Timing Notes (as of April 2026)
**Reasons to wait:**
- DDR5 RDIMM prices should ease in 12–24 months
- M5 Ultra (WWDC ~June 8) may reset used workstation GPU prices
- RTX Pro 6000 Blackwell prices will drift down as used units appear
- MoE architectures getting more efficient (less hardware per quality tier)

**Reasons not to wait:**
- Used A6000/A100 prices rising (datacenter gear being cannibalized for inference)
- DRAM could stay tight if AI demand keeps scaling
- Tariffs/export controls — nothing gets cheaper after a round
- Learning time lost: 6 months hands-on > 20% hardware discount

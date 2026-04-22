# Learning Build — Active Shopping List

Target: ~$2,200–2,750 | Single GPU | 128GB RAM | EPYC Rome | Headless Linux

---

## GPU — Used RTX 3090 24GB

**Status: CANDIDATE IDENTIFIED**

Candidate: EVGA GeForce RTX 3090 FTW3 Ultra 24GB GDDR6X (24G-P5-3987-KR)
- Triple-fan open-air, 2.75-slot, ~300mm length
- 3× 8-pin PCIe connectors (not 12VHPWR — confirmed compatible with Corsair RM1000x)
- 350W TDP, up to ~450W transient
- Confirmed: fits Phanteks Enthoo Pro II Server Edition
- **Notes**: EVGA no longer honors warranty transfers; buy gamer-sold, not miner

**Target price**: under $750 shipped for verified good condition

**Sources**:
- eBay RTX 3090 listings: https://www.ebay.com/sch/i.html?_nkw=rtx+3090+24gb
- r/hardwareswap — preferred (gamer-verified history)

**Before buying checklist**:
- [ ] GPU-Z or HWiNFO screenshot: VRAM junction temp <95°C under load
- [ ] Ask: "Was this used for mining?" — evasive = walk away
- [ ] Original box/accessories present
- [ ] Three separate 8-pin PCIe cables confirmed on PSU

---

## CPU — EPYC 7402P (24c/48t Rome, SP3, unlocked)

**Status: Not purchased**

Target price: ~$200–350

**Must say "unlocked" or "no vendor lock"** — Dell/HPE-locked chips won't POST on Supermicro board.

Alternatives:
- EPYC 7443P (24c Milan, faster) ~$400 — better if budget allows
- EPYC 7F32 (8c, high clock) ~$300 — better single-thread, fewer cores

Sources:
- eBay: https://www.ebay.com/sch/i.html?_nkw=epyc+7402p+unlocked

---

## Motherboard — Supermicro H12SSL-i (single SP3, IPMI)

**Status: Not purchased**

Target price: ~$550–750

This is the headless server key piece — IPMI/BMC gives real out-of-band management.

Sources:
- Newegg: https://www.newegg.com/supermicro-mbd-h12ssl-i-o/p/N82E16813183734
- eBay (CPU+board bundles often cheaper)

---

## RAM — 8× 16GB DDR4-3200 RDIMM ECC (128GB total)

**Status: Not purchased**

Target price: ~$250–400 used

Buy matched kit: same brand, same part number (e.g., Hynix HMAA2GR7AJR8N-XN or equivalent).

Can expand to 256GB later (8 slots × 32GB) without other changes.

Sources:
- eBay: https://www.ebay.com/sch/i.html?_nkw=16gb+ddr4+3200+rdimm+ecc

---

## PSU — Corsair RM1000x (1000W, 80+ Gold)

**Status: Not purchased**

Target price: ~$180

Handles 1× 3090 (350W) + EPYC (180W) + overhead. Has three separate 8-pin PCIe cables needed for FTW3 Ultra. Room to add second GPU later (up to ~900W load).

Sources:
- Amazon: https://www.amazon.com/s?k=corsair+rm1000x
- Newegg: https://www.newegg.com

---

## Case — Phanteks Enthoo Pro II Server Edition

**Status: Not purchased**

Target price: ~$170

SSI-EEB support (fits Supermicro H12SSL-i). Good airflow. Confirmed: fits FTW3 Ultra length.

Alternative: Fractal Design Meshify 2 XL (~$210)

Sources:
- Newegg: https://www.newegg.com/phanteks-ph-es719psg/p/N82E16811854104

---

## CPU Cooler — Noctua NH-U14S TR4-SP3

**Status: Not purchased**

Target price: ~$100

SP3 socket requires this specific model — the standard NH-U14S will not fit.

Sources:
- Amazon: https://www.amazon.com/s?k=noctua+nh-u14s+tr4-sp3

---

## Storage — 2TB Samsung 990 Pro NVMe

**Status: Not purchased**

Target price: ~$150

Models + quants take 100–200GB each. 2TB gives room to experiment with multiple quants.

Sources:
- Amazon: https://www.amazon.com/s?k=samsung+990+pro+2tb

---

## Budget Tracker

| Component | Target | Actual | Status |
|-----------|--------|--------|--------|
| GPU (RTX 3090 FTW3 Ultra) | $650–750 | — | Candidate identified |
| CPU (EPYC 7402P) | $200–350 | — | Not purchased |
| Motherboard (H12SSL-i) | $550–750 | — | Not purchased |
| RAM (128GB DDR4 RDIMM) | $250–400 | — | Not purchased |
| PSU (Corsair RM1000x) | $180 | — | Not purchased |
| Case (Enthoo Pro II) | $170 | — | Not purchased |
| CPU Cooler (NH-U14S SP3) | $100 | — | Not purchased |
| Storage (990 Pro 2TB) | $150 | — | Not purchased |
| **Total** | **$2,250–2,680** | — | |

---

## Cheaper Alternative (~$800–1,200)

If you want to start even smaller:
- Used HP Z840 / Z8 G4 on eBay: $400–700 (often includes 128GB RAM)
- Add used RTX 3090: ~$700
- Trade-off: Intel Xeon 6-channel DDR4 has less bandwidth than EPYC 8-channel; CPU offload for MoE is slower but functional

---

## Post-Build Software Checklist

- [ ] Install Ubuntu 24.04 Server
- [ ] Install NVIDIA drivers + CUDA
- [ ] Build llama.cpp from source (CUDA backend)
- [ ] Download MiniMax-M2.1 UD-Q4_K_XL GGUF (from Unsloth)
- [ ] Start llama.cpp server on OpenAI-compatible port (8080)
- [ ] Install claude-code-router
- [ ] Configure Claude Code to point at local endpoint
- [ ] Test coding prompt throughput (target: 8–15 tok/s)

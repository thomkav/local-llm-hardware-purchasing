# Used Hardware Risks

## GPU Risks (most common issues)

### Ex-mining cards
- RTX 3090s especially; fans and thermal pads often toast from 24/7 duty
- VRAM can be degraded from sustained high temps (>100°C junction)
- Look for gamer-sold cards, not "batch of 8 from datacenter"

### Ex-datacenter (A100/H100)
- Often fine electrically but no warranty, no NVIDIA support
- May have run at 100% for 2–3 years

### Fake/relabeled cards
- Common on eBay for A100 40GB, V100, even 3090s
- Chinese gray market repairs/rebuilds
- Verify: `nvidia-smi` for correct VBIOS, run `gpu-burn` for 24 hours

### Mining-modified BIOS (CMP, P102-100)
- Limited PCIe lanes, broken FMA math
- Not usable for 200B+ MoE models — avoid entirely

## Server/CPU/RAM Risks

### Vendor-locked BIOS
- Dell PowerEdge and HP ProLiant often refuse non-OEM parts
- Supermicro and Gigabyte are DIY-friendly — stick to these

### RAM mismatches
- Mixed speeds/vendors downclock to slowest stick
- MoE inference is memory-bandwidth-bound — this hurts throughput
- Buy matched kits (same brand, same part number)

### Dead DIMMs / bad slots
- Run memtest86 for 24+ hours before trusting
- Single flaky DIMM in 24 slots is miserable to diagnose

### PSU age
- Capacitors dry out; 5+ year old 2kW PSUs fail under sustained GPU load
- Budget for a new PSU even if "included"

### Dead-end platforms
- EPYC Rome (7002) and Naples (7001) are EOL — no BIOS updates
- Fine for inference, no upgrade path

## Logistics Risks

### Noise
- 2U/4U server fans at full tilt: 70–80 dB
- Not livable outside garage/basement/rack room
- Ask about fan profiles before buying

### Power delivery
- 8× 3090 or dual EPYC + 4 GPUs: 3–4 kW
- Standard US 15A/120V: 1.8 kW before breaker trips
- May need dedicated 20A circuit or 240V

### Shipping damage
- Heavy servers shipped poorly = bent chassis, broken PCIe slots
- Buy local (Craigslist, FB Marketplace, r/homelabsales) when possible
- Or buy from reputable refurbishers who provide real packaging photos

## De-Risking Strategy

1. **Buy chassis/CPU/RAM from refurbishers with warranty** (TheServerStore, UNIXSurplus, Bargain Hardware, ServerMonkey — 30–90 day warranty for ~10–20% premium, usually worth it)
2. **Buy GPU separately** from r/hardwareswap gamer sales — different source, isolated failure
3. **Burn-in week**: memtest86 → gpu-burn each GPU → 48-hour full-load inference run
4. **Budget 10–15% reserve** — one component will die; that's when, not if
5. **Never buy a "turnkey" used 8-GPU rig from one seller** — can't isolate which component failed

## GPU Verification Checklist (before purchase)
- [ ] Ask for GPU-Z or HWiNFO screenshot showing VRAM junction temp under load (<95°C)
- [ ] Ask directly: "Was this used for mining?" (evasive = yes)
- [ ] Check original box/accessories (signals gamer vs. stripped miner)
- [ ] Verify correct VBIOS via `nvidia-smi` after arrival
- [ ] Run `gpu-burn` for 24 hours on arrival

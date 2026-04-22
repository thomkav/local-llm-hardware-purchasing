# Basement Installation Notes

Unfinished basement in Portland, OR. Server + networking gear.

---

## Humidity

This is the real concern for Portland. Outdoor RH in Portland averages ~80% year-round, and unfinished basements without active dehumidification routinely sit at 65–75% RH, higher after rain. Electronics are typically rated for 20–80% RH non-condensing, but sustained operation near the top of that range accelerates corrosion on PCB traces and connectors over years.

The bigger risk is **condensation**: if the server is off and the basement cools overnight, then the server warms up quickly when powered on, you can get moisture on cold surfaces. This is especially risky on a GPU or motherboard coming out of storage.

**Practical steps:**
- Run a dehumidifier. A 50-pint unit (~$200–250, e.g. hOmeLabs or Frigidaire) will keep a typical Portland basement under 50% RH. This is non-optional for any electronics you care about.
- Put a cheap hygrometer/thermometer combo (~$10–15) near the rack and check it occasionally. Target: 40–55% RH.
- Keep equipment off the concrete floor — concrete wicks moisture. Even a few inches of elevation matters.
- Don't power on a server that's been in a cold humid space for hours. Let it equilibrate to room temperature first.
- Once running, the server's own heat output helps a lot — a machine that's always on stays above the dew point.

---

## Rack / Mounting

Options for an unfinished basement, roughly in order of effort:

### Option A — Wall-mount open frame (recommended for 1–2 servers)
A 12U or 18U wall-mount bracket attached to exposed studs. Keeps everything off the floor, looks clean, uses no floor space. Most unfinished basements have 2×4 or 2×6 stud walls somewhere.

- Navepoint 12U wall-mount open frame: ~$60–80
- StarTech 12U wall-mount bracket: ~$80–100
- Attach to at least 2 studs with lag bolts. A loaded 12U rack can weigh 100–200 lbs.
- If the only walls are poured concrete or CMU block, use concrete sleeve anchors (Tapcon or equivalent).

### Option B — Free-standing open-frame floor rack
A 4-post open-frame rack sitting on the floor. No wall attachment needed. Good if you might expand later.

- APC AR3100 (42U) or Navepoint 12U/18U floor unit: $150–400
- Put it on rubber anti-vibration pads or a small wooden platform (not directly on bare concrete).
- Leave 24–30 inches clearance at front and rear for airflow and cable management.

### Option C — Shelf or workbench
For the DGX Spark specifically, it's just a box. A sturdy shelf on the wall or an existing workbench works fine.

---

## Networking

You don't need a dedicated router in the basement — what you need is reliable ethernet from your main router down to the basement, and a small switch to fan out to the server's ports.

### Minimum setup
1. **Run a Cat6 cable** from your main router/switch upstairs down to the basement. Cat6 (not Cat5e) is the right call for new runs — better noise rejection, handles 10GbE if you ever upgrade NICs. Staple along joists with cable staples.
2. **Small unmanaged switch** in the basement: TP-Link TL-SG108 (8-port, ~$20) is fine if you don't need VLANs.
3. Plug the server's main NIC and its IPMI/BMC port into the switch. Assign the IPMI port a static IP in your router's DHCP reservation table so it's always reachable.

### Better: managed switch with VLAN
A managed switch (TP-Link TL-SG108E, ~$30) lets you put IPMI traffic on a separate VLAN from inference traffic. Not required to start, but good practice so IPMI management isn't on the same network segment as your regular home traffic.

### Remote access: Tailscale
Install [Tailscale](https://tailscale.com) on the server and your laptop. Zero-config WireGuard VPN — the server gets a stable `100.x.x.x` address reachable from anywhere, including when you're not home. Free for personal use. This replaces any need for port-forwarding or a VPN router.

For IPMI specifically: Tailscale on the server's main OS gives you SSH from anywhere. For true out-of-band IPMI access (when the OS is down), you'd need Tailscale running on a separate device on the same network — a Raspberry Pi on the basement switch works well for this.

### Approximate networking equipment
| Item | Est. cost |
|------|-----------|
| Cat6 cable (50–100ft bulk) | $15–25 |
| TP-Link TL-SG108E managed switch | $30 |
| Patch cables (short, for within rack) | $15 |
| Raspberry Pi 4 (for out-of-band IPMI access, optional) | $45–60 |
| **Total** | **~$60–100** |

---

## Cable Notes

- **Run Cat6, not Cat5e**, for any new cable you're pulling. The cost difference is negligible and it future-proofs for 10GbE.
- In an unfinished basement, you can staple cable runs along joists with insulated cable staples. Don't use regular staples — they can nick the jacket.
- Keep power cables and ethernet separated by a few inches where they run parallel to avoid interference.
- Label both ends of every run. A $15 label maker saves hours of future confusion.
- The IPMI port on a Supermicro board uses a standard RJ45 — treat it like any other ethernet port, just put it on a separate switch port and note its IP.

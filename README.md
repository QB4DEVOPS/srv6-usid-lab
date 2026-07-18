# Quisted SRv6 uSID Lab (CML)

Standalone lab package — **Quisted MPLS-SR topology**, adapted for **IPv6-only SRv6 uSID + IS-IS** on CML 2.10.

**Data plane:** IPv6 only (no IPv4 loopbacks, links, or IS-IS IPv4). **Mgmt (`Mgmt-intf` VRF):** IPv4 DHCP for OOB SSH — XR uses `ssh server vrf Mgmt-intf`; XE needs RSA key, `ip ssh source-interface GigabitEthernet3`, and `line vty` → `login local` (see **Management access** in the lab guide).

Proof model: **before/after ping + PCAP** (+ optional debug).

## Quick start

1. **Import** → `quisted-srv6-baseline.yaml` — IS-IS + IPv6 running, no SRv6 configured (student starting point)
2. **Lab guide** → [`srv6-usid-isis-lab.html`](srv6-usid-isis-lab.html) — configure SRv6 uSID step by step, verify as you go
3. **Reference** → `quisted-srv6-solution.yaml` — full SRv6 uSID configured (answer key)

Credentials:

| Nodes | Username | Password |
|-------|----------|----------|
| P1–P4 (IOS XR) | `cisco` | `Cisc01@3` |
| PE5–PE8 (IOS XE) | `cisco` | `cisco` |

## What's in this repo

| File | Purpose |
|------|---------|
| `quisted-srv6-baseline.yaml` | **Import this** — IS-IS + IPv6 baseline, no SRv6 |
| `quisted-srv6-solution.yaml` | Full SRv6 uSID solution (reference / answer key) |

## uSID proof (announcement runbook)

**Run order:** (1) before ping CLI → (2) arm debug → (3) then ping uSID.

| Step | Action |
|------|--------|
| 1 | PE5: `ping fc00:0:7::1` — save CLI output |
| 2 | CML PCAP on P1–P4; PE7: arm `debug ipv6 icmp` + `debug ipv6 packet detail` |
| 3 | PE5: `ping fcbb:bb00:7::` — save CLI + PCAP (Dst = uN SID) |

## Reference

- [Quisted MPLS-SR lab](https://www.quisted.net/index.php/2024/11/11/mpls-segment-routing-mpls-sr-lab/) — original topology inspiration

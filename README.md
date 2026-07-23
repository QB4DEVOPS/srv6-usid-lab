# Quisted SRv6 uSID Lab Series (CML)

A five-part lab curriculum that teaches SRv6 micro-SID (uSID) by re-implementing
[Quisted's MPLS Segment Routing lab](https://www.quisted.net/index.php/2024/11/11/mpls-segment-routing-mpls-sr-lab/)
on an IPv6-only data plane. The parts follow Quisted's proof arc — neighbour →
segment → FIB → easy ping → endpoint-SID ping → path walk → capture. Part 1 is the
foundation; Parts 2 and 3 are parallel branches off it, and Parts 4 and 5 build on
Part 2 (see the Prerequisite column below).

---

## Lab Series

| Part | Topic | Student Guide | YAML to Load | CML Lab Title | Prerequisite |
|------|-------|--------------|-------------|---------------|-------------|
| **1** | SRv6 uSID IS-IS underlay | [srv6-usid-part1-isis-lab.html](srv6-usid-part1-isis-lab.html) | `quisted-srv6-part1-baseline.yaml` | Quisted SRv6 uSID — Part 1 Baseline (IS-IS only) | None |
| **2** | SRv6 L3VPN — VRFs, VPNv4, End.DT4 SID | [srv6-usid-part2-l3vpn-lab.html](srv6-usid-part2-l3vpn-lab.html) | `quisted-srv6-part2-solution.yaml` | Quisted SRv6 uSID — Part 2 L3VPN Solution | Part 1 |
| **3** | SR-TE uSID Traffic Engineering | [srv6-usid-part3-srte-lab.html](srv6-usid-part3-srte-lab.html) | `quisted-srv6-part3-solution.yaml` | Quisted SRv6 uSID — Part 3 SR-TE Solution | Part 1 |
| **4** | CE Routers — dual-homed eBGP vs static non-speaker | [srv6-usid-part4-ce-lab.html](srv6-usid-part4-ce-lab.html) | `quisted-srv6-part4-solution.yaml` | Quisted SRv6 uSID — Part 4 CE Solution | Part 2 |
| **5** | SRv6 ↔ SR-MPLS Interworking Gateway (IGW) | [srv6-usid-part5-igw-lab.html](srv6-usid-part5-igw-lab.html) | `quisted-srv6-part5-solution.yaml` | Quisted SRv6 uSID — Part 5 IGW Solution | Part 2 |

Parts 3 and 2 are parallel extensions of Part 1 — either can be done after Part 1 without the other.

---

## Quick Start

1. Import the YAML for the part you want into CML (Workbench → Import Lab).
2. Start the lab and wait for all nodes to reach their prompts. The first cold boot of the IOS XRv 9000 nodes (P1–P4, plus IGW/MP1/MPE1 in Part 5) can take ~15–20 min each; later boots are faster.
3. Open the corresponding HTML student guide in a browser and follow the steps.
4. The solution YAML for each part is the answer key — load it if you get stuck.

---

## What's in This Repo

| File | CML Lab Title | Purpose |
|------|---------------|---------|
| `quisted-srv6-part1-baseline.yaml` | Quisted SRv6 uSID — Part 1 Baseline (IS-IS only) | Part 1 student start — IS-IS pre-configured, no SRv6 |
| `quisted-srv6-part1-blank-routing.yaml` | Quisted SRv6 uSID — Part 1 Blank Routing | Alt Part 1 start — addressing only, build IS-IS from scratch |
| `quisted-srv6-part1-solution.yaml` | Quisted SRv6 uSID — Part 1 Solution | Part 1 answer key — IS-IS + SRv6 uSID complete |
| `quisted-srv6-part2-solution.yaml` | Quisted SRv6 uSID — Part 2 L3VPN Solution | Part 2 answer key — L3VPN added |
| `quisted-srv6-part3-solution.yaml` | Quisted SRv6 uSID — Part 3 SR-TE Solution | Part 3 answer key — SR-TE policy on P1 |
| `quisted-srv6-part4-solution.yaml` | Quisted SRv6 uSID — Part 4 CE Solution | Part 4 answer key — CE routers added |
| `quisted-srv6-part5-solution.yaml` | Quisted SRv6 uSID — Part 5 IGW Solution | Part 5 answer key — IGW + MPLS domain added |
| `srv6-usid-part1-isis-lab.html` | — | Part 1 student guide |
| `srv6-usid-part2-l3vpn-lab.html` | — | Part 2 student guide |
| `srv6-usid-part3-srte-lab.html` | — | Part 3 student guide |
| `srv6-usid-part4-ce-lab.html` | — | Part 4 student guide |
| `srv6-usid-part5-igw-lab.html` | — | Part 5 student guide |

---

## CML Requirements

- **CML version:** 2.9 or 2.10
- **Node definitions required:**

| Node definition | Used by | Parts |
|---|---|---|
| `iosxrv9000` | P1–P4 (core P routers), IGW, MP1, MPE1 | 1–5 |
| `cat8000v` | PE5–PE8 (provider edge) | 1–5 |
| `iosv` | CE9–CE12 (customer edge) | 4 |

- CAT 8000V nodes require `license boot level network-advantage addon dna-advantage` and a reload before SRv6 locators become active (included in the day-0 config in every YAML).
- All nodes boot in parallel; no node staging required.

---

## Topology

8-router diamond mesh (Quisted's original topology, adapted for SRv6):

```
 [PE5]──────[P1]──────[P3]──────[PE7]
  │          │          │          │
 [PE6]──────[P2]──────[P4]──────[PE8]
```

- **P1–P4**: IOS XRv 9000 — SRv6 P routers, IS-IS 100, uSID locators `fcbb:bb00:000N::/48`
- **PE5–PE8**: CAT 8000V (IOS-XE 17.18) — SRv6 PE routers, IS-IS 100, VRF CUST-A / CUST-B
- **Data plane**: IPv6-only — no IPv4 on data links or loopbacks
- **Management**: OOB IPv4 DHCP via `Mgmt-intf` VRF on all nodes

---

## Credentials

> ⚠️ Lab-only credentials — do not reuse in any production environment.

| Nodes | Username | Password |
|-------|----------|----------|
| P1–P4 (IOS-XR) | `cisco` | `Cisc01@3` |
| PE5–PE8 (IOS-XE) | `cisco` | `cisco` |
| CE9–CE12, IGW, MP1, MPE1 | `cisco` | `cisco` |

---

## Proof Model (Part 1)

| Step | Action |
|------|--------|
| 1 | PE5: `ping fc00:0:7::1 source fc00:0:5::1` — IS-IS loopback, proves underlay only |
| 2 | Arm PCAP on a P1–P4 link; PE7: `debug ipv6 icmp` + `debug ipv6 packet detail` |
| 3 | PE5: `ping fcbb:bb00:7:: source fc00:0:5::1` — uN SID; PCAP destination = the SID |

---

## Reference

Original MPLS-SR lab: <https://www.quisted.net/index.php/2024/11/11/mpls-segment-routing-mpls-sr-lab/>

# SRv6 uSID Lab (CML)

**Quisted MPLS-SR topology**, adapted for **IPv6-only SRv6 uSID + IS-IS** on Cisco Modeling Labs 2.10.

Inspired by [Quisted's MPLS Segment Routing lab](https://www.quisted.net/index.php/2024/11/11/mpls-segment-routing-mpls-sr-lab/) — same 8-router topology, same verification discipline, IPv6/SRv6 data plane instead of MPLS labels.

**Prerequisites:** IPv6 addressing and IS-IS are assumed knowledge. This lab focuses on adding SRv6 uSID on top of a working IS-IS domain.

---

## Lab files

| File | Use |
|------|-----|
| `quisted-srv6-baseline.yaml` | **Import this.** IS-IS + IPv6 running, no SRv6 — student starting point. |
| `quisted-srv6-solution.yaml` | Full SRv6 uSID configured — reference / answer key. |

---

## Topology

8-router full-mesh core. All links and loopbacks are IPv6-only. Management is IPv4 DHCP via CML's external bridge (`ext-conn`).

| Role | Nodes | Platform | Image |
|------|-------|----------|-------|
| P (core) | P1–P4 | IOS XRv 9000 | `iosxrv9000-25-4-1` |
| PE (edge) | PE5–PE8 | Catalyst 8000V | `cat8000v-17-18-02` |

```
      PE5   PE6
       |  X  |
       P1 ── P2
       |      |
       P3 ── P4
       |  X  |
      PE7   PE8
```

### Addressing

| Node | Loopback | SRv6 locator (task 3) |
|------|----------|-----------------------|
| P1 | `fc00:0:1::1/128` | `fcbb:bb00:0001::/48` |
| P2 | `fc00:0:2::1/128` | `fcbb:bb00:0002::/48` |
| P3 | `fc00:0:3::1/128` | `fcbb:bb00:0003::/48` |
| P4 | `fc00:0:4::1/128` | `fcbb:bb00:0004::/48` |
| PE5 | `fc00:0:5::1/128` | `fcbb:bb00:0005::/48` |
| PE6 | `fc00:0:6::1/128` | `fcbb:bb00:0006::/48` |
| PE7 | `fc00:0:7::1/128` | `fcbb:bb00:0007::/48` |
| PE8 | `fc00:0:8::1/128` | `fcbb:bb00:0008::/48` |

Point-to-point links: `2001:<LowID><HighID>::<RtrID>/64`

---

## Credentials

| Nodes | Username | Password |
|-------|----------|----------|
| P1–P4 (IOS XR) | `cisco` | `Cisc01@3` |
| PE5–PE8 (IOS XE) | `cisco` | `cisco` |

Management interfaces use DHCP from CML's external bridge.

---

## Lab tasks

### Task 1 — Verify the baseline

Import `quisted-srv6-baseline.yaml`, start the lab, wait for IS-IS to converge (~3 min).

```
P1# show isis neighbors
PE5# ping fc00:0:7::1       ! loopback reachability via IS-IS
PE5# traceroute fc00:0:7::1 ! confirm path through P routers
```

All loopbacks should be reachable. No SRv6 forwarding yet.

### Task 2 — Configure SRv6 on IOS XR (P1–P4)

On each P router, add the SRv6 locator and enable it under IS-IS:

```
segment-routing
 srv6
  encapsulation
   source-address <loopback>
  !
  locators
   locator MAIN
    prefix fcbb:bb00:00NN::/48
    micro-segment behavior unode psp-usd
   !
  !
 !
!
router isis 100
 address-family ipv6 unicast
  segment-routing srv6
   locator MAIN
  !
 !
!
```

Verify: `show segment-routing srv6 sid` — you should see `uN` and `uA` entries.

### Task 3 — Configure SRv6 on IOS XE (PE5–PE8)

On each PE router, add the SRv6 locator (F3216 format) and enable it under IS-IS:

```
segment-routing srv6
 encapsulation
  source-address <loopback>
 !
 locators
  locator MAIN
   format usid-f3216
   prefix fcbb:bb00:00NN::/48
 !
!
router isis 100
 address-family ipv6 unicast
  segment-routing srv6
   locator MAIN
  !
 !
!
```

### Task 4 — Verify SRv6 uSID forwarding

```
! SRv6 SID table on P and PE nodes
P1# show segment-routing srv6 sid
PE5# show segment-routing srv6 sid

! Reachability — still works via IS-IS
PE5# ping fc00:0:7::1

! uSID forwarding — ping the uN SID directly
PE5# ping fcbb:bb00:7::

! Traceroute — should show P-router hops with SRv6 encapsulation
PE5# traceroute fcbb:bb00:7::
```

---

## Key concepts

**SRv6 uSID (micro-SID, F3216)** — IOS XE `usid-f3216` encodes multiple 16-bit compressed SIDs into one 128-bit IPv6 destination. Each router advertises a `/48` locator; the `uN` SID is the first address in that block (e.g. `fcbb:bb00:7::`). No MPLS, no LDP — the IPv6 FIB is the forwarding plane.

If you completed Quisted's MPLS-SR lab, the mental mapping is:

| MPLS-SR | SRv6 (this lab) |
|---------|----------------|
| SRGB / prefix-SID index | Locator block `fcbb:bb00:00NN::/48` |
| Label (e.g. 16007) | uN SID (`fcbb:bb00:7::`) |
| `show mpls forwarding` | `show segment-routing srv6 sid` |
| Label stack | SID list (compressed uSID in IPv6 dst) |

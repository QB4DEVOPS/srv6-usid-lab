# SRv6 uSID IS-IS Lab — Quisted Topology on CML 2.10

> Adapted from [Quisted's MPLS Segment Routing lab](https://www.quisted.net/index.php/2024/11/11/mpls-segment-routing-mpls-sr-lab/)

---

## What is SRv6 uSID?

### What is Segment Routing (SR)?

In short, **Segment Routing** is a modern approach to steering traffic through a network using an ordered list of **segments** instead of per-flow signaling state on every hop. Each segment is a simple forwarding instruction — typically “forward to this node” (node segment) or “forward to this neighbor” (adjacency segment). Link-state IGPs (IS-IS, OSPFv2, OSPFv3) advertise these segments in their topology database. The ingress router imposes the segment list; transit routers execute one segment at a time without maintaining RSVP-TE tunnels or LDP bindings for each path.

That model simplifies traffic engineering, improves scale, and keeps the control plane familiar: you still run an IGP, but you replace auxiliary label-distribution protocols with SIDs advertised by the IGP itself.

### SR-MPLS (what Quisted's lab teaches)

In **SR-MPLS** (MPLS Segment Routing), segments are encoded as **MPLS labels** from the Segment Routing Global Block (SRGB). Each loopback gets a **prefix-SID index** (P1=1, P2=2, …) that maps to a label (e.g. index 2 → label 16002). Quisted's post walks through migrating a traditional MPLS network — OSPF area 0 plus LDP — to **MPLS-SR** using OSPF prefix-SIDs, first on IOS-XR and IOS-XE, then adding a **Segment Routing Mapping Server (SRMS)** so legacy **IOSv** PE routers can participate even though they do not speak MPLS-SR themselves.

### What is SRv6?

**Segment Routing over IPv6 (SRv6)** applies the same segment-list idea to the **IPv6 data plane**. Instead of pushing MPLS labels, the ingress router encodes segments in the IPv6 destination address (and, when needed, SRv6 extension headers). Each router advertises an IPv6 **locator** (in this lab, a `/48`). Individual **SIDs** are IPv6 addresses or sub-blocks within that locator; the router performs an IPv6 FIB lookup and executes an SRv6 **behavior** (End, End.X, uN, uA, and others) rather than a label swap.

### Why SRv6 instead of SR-MPLS?

Both implement the *same* Segment Routing control-plane idea; the difference is the **data plane**. SR-MPLS carries segments as **MPLS labels**; SRv6 carries them as **IPv6 addresses** (SIDs) with defined forwarding behaviors. Neither needs LDP or RSVP-TE for basic segment forwarding — the IGP advertises the segments in both cases.

| | SR-MPLS | SRv6 |
|---|---------|------|
| **Forwarding lookup** | MPLS label swap / push / pop | IPv6 destination lookup + SRv6 behavior (End, End.X, uN, uA, …) |
| **Segment identity** | Label from SRGB + prefix-SID index | IPv6 address or sub-block inside a locator |
| **Underlay required** | MPLS enabled on every interface in the path | IPv6-only IGP (this lab) |
| **Typical IGP extensions** | OSPF/OSPFv3 or IS-IS prefix-SID sub-TLVs | IS-IS or OSPFv3 SRv6 locator + SID advertisements |
| **Service examples** | L3VPN over MPLS, TE via label stacks | L3VPN via End.DT46, EVPN, traffic engineering in IPv6 header |

#### Coming from VXLAN?

> **If you know VXLAN:** MPLS labels are **tag-like** — compact forwarding identifiers that can be pushed, swapped, popped, and stacked hop-by-hop. **SRv6 is different:** a segment is an **IPv6 destination** in the node's locator block (e.g. uN `fcbb:bb00:7::`). That is closer to **VTEP forwarding** — you steer by setting **destination IP = remote endpoint** (the node's SID), not by swapping a label.
>
> - **Loopback** (`fc00:0:7::1`) — the host prefix you ping; the destination you're trying to reach.
> - **uN / locator SID** — transport target: "deliver SRv6 processing to this node" (VTEP-like: new outer IP dst aimed at the endpoint).
> - **VNI** — not in this lab; in EVPN-SRv6 it maps to a **service SID** (e.g. End.DT2), not a loopback.

Operators choose **SRv6** in general when they want:

- **One forwarding paradigm** — services and underlay both expressed as IPv6 SIDs, not a parallel MPLS control and data plane on top of IP.
- **No MPLS label logistics** — no per-router label ranges, SRGB planning, or `show mpls forwarding` troubleshooting for the SR underlay.
- **IPv6-native designs** — greenfield cores, 5G transport, and cloud/DCI where IPv6 is the long-term substrate.
- **Rich endpoint behaviors** — one SID can mean route-to-node, lookup-in-VRF, or cross-connect (End, End.DT, End.DX) without extra label stacks.
- **Micro-SID (μSID) efficiency** — μSID can encode multiple 16-bit compressed segment identifiers in one 128-bit IPv6 destination-address container, reducing the number of full-length SIDs required in an SRH (this lab’s **F3216** format: 32-bit block + 16-bit micro-segments). Micro-SID is commonly written **uSID**; **uN**, **uA**, and `usid-f3216` are the ASCII forms used in standards, documentation, and CLI syntax.

**SR-MPLS** remains common in **brownfield** networks that already run MPLS VPNs, have ASICs optimized for label switching, or are mid-migration from LDP. Many operators run **SR-MPLS today and SRv6 tomorrow**; the high-level Segment Routing model transfers, but SID representation, forwarding behavior, packet overhead, hardware requirements, steering, and troubleshooting differ.

#### Why SRv6 in this CML lab?

This lab keeps Quisted’s topology and verification flow but implements SRv6 uSID instead of MPLS-SR. CML-specific reasons:

- **Platform reality** — Quisted’s IOSv PEs cannot run MPLS-SR or SRv6. SRv6 uSID requires **CAT 8000V** on PEs and **IOS XRv 9000** on P routers. There is no SRMS workaround for SRv6; all nodes must support it natively.
- **Images validated on CML 2.10** — `iosxrv9000-25-4-1` and `cat8000v-17-18-02` with SRv6 uSID features enabled out of the box in the imported configs.
- **Locator replaces label range** — Quisted’s device table uses **Label Ranges** (P1 = 24100–24199). Our table uses **SRv6 locators** in the same column (`fcbb:bb00:00NN::/48`).
- **IPv6 verification targets** — ping/traceroute `fc00:0:7::1` instead of `7.7.7.7`; commands are `show isis ipv6 route` and `show segment-routing srv6 sid` instead of `show mpls forwarding`.

If you completed Quisted’s MPLS-SR lab first, use this **approximate mental mapping** (not direct equivalence): **SRGB or label-allocation plan → locator and SID-allocation plan**; **prefix-SID → node or endpoint SID**; **MPLS label → SID or compressed SID instruction**; **label stack → SID list or compressed SID containers**; **`show mpls forwarding` → `show isis ipv6 route`**. If you have not, you do not need LDP or OSPF-SR experience — this guide is self-contained on IS-IS + SRv6.

### What is SRv6 uSID (micro-SID)?

> Micro-SID is commonly written **uSID**; **uN**, **uA**, and **`usid-f3216`** are the ASCII forms used in standards, documentation, and CLI syntax.

A full 128-bit SID per hop can be heavy. **Micro-SID (uSID)** can encode multiple 16-bit compressed segment identifiers in one 128-bit IPv6 destination-address container using a fixed format — on IOS-XE, **usid-f3216** (format **F3216**). Each node still advertises a `/48` locator; the router programs a **uN** (node) SID and **uA** (adjacency) SIDs from that block. You will see them with `show segment-routing srv6 sid` on P1, analogous to Quisted's `show mpls label table detail`.

**This lab uses uSID only.** Do not confuse it with **G-SRv6 / G-SID** (Generalized Segment Identifier) — a **competing** compression design (Replace-C-SID, historically China Mobile / Huawei). Both are standardized in RFC 9800, but they use **different SRH processing** (shift-and-lookup vs replace-and-lookup). This CML lab does **not** implement G-SRv6.

#### F3216 addressing in this lab (correct model)

| Layer | Purpose | This lab |
|-------|---------|----------|
| **Infrastructure** | IS-IS loopbacks, ping “easy” target | `fc00:0:<N>::1/128` — **not** locators |
| **Locator-Block** | Shared 32-bit uSID domain (GIB prefix) | `fcbb:bb00` (`/32` conceptually) |
| **Locator-Node** | 16-bit node ID (uSID slot 0 = uN) | `0001` … `0008` → `/48` per router |
| **Per-node locator** | Advertised in IS-IS | `fcbb:bb00:00NN::/48` for **all** P and PE nodes |
| **uN endpoint** | Endpoint-SID ping/traceroute target | `fcbb:bb00:<N>::` (e.g. PE7 → `fcbb:bb00:7::`) |

**Wrong (do not use):** per-node `/48` blocks like `fc00:0:1::/48`, `fc00:0:2::/48` on P routers — that is **not** F3216 (each router gets its own “block” instead of one shared Locator-Block + Node CSID). **Wrong:** splitting P routers on `fc00:` and PE routers on `fcbb:` — one uSID domain needs **one** Locator-Block.

```
128-bit F3216 locator (example PE7):
|←—— 32-bit block fcbb:bb00 ——→|← 16-bit node 0007 →|← uA/LIB space →|
 fcbb:bb00:0007:0000:0000:0000:0000:0000:0000:0000
 uN at fcbb:bb00:7::  (abbreviated uSID-0)
```

Each node advertises `/48` locator; the router programs **uN** (node) and **uA** (adjacency) SIDs from that block — visible with `show segment-routing srv6 sid` on P1, analogous to Quisted's `show mpls label table detail`.

**Cisco and Juniper both implement uSID (NEXT-C-SID)** against the same IETF F3216 baseline. The *concept* is vendor-neutral; the *configuration* differs:

| | Cisco (this lab) | Juniper (Junos) |
|---|------------------|-----------------|
| **Locator config** | IOS-XR: `micro-segment behavior unode` under locator; IOS-XE: `format usid-f3216` | `routing-options source-packet-routing srv6` → locator + `micro-sid { block-name }` |
| **IGP** | IS-IS SRv6 locator under IPv6 AF | IS-IS advertises micro-SID locators from SPR block |
| **TE segment list** | SRv6 TE policies (platform-dependent) | `micro-srv6-sid` only — cannot mix with classic `srv6-sid` in same list |
| **Verify** | `show segment-routing srv6 locator/sid` | `show isis database` SRv6 TLVs, SPR show commands |

This CML lab is **Cisco-only**. A Juniper uSID domain uses the same F3216 addressing math but different config hierarchy.

### What this lab demonstrates

This lab uses the same 8-router diamond as Quisted's MPLS-SR post, but with an **SRv6 uSID underlay** instead of OSPF, LDP, and MPLS labels. You will build **IPv6-only IS-IS** adjacencies, enable uSID locators on every router, and verify end-to-end IPv6 reachability from PE5 to PE7 through the P core. There is no SR mapping server step — every platform supports SRv6 natively.

### Addressing model

| Plane | What runs here | Addressing |
|-------|----------------|------------|
| **Lab data plane** | IS-IS + SRv6 uSID forwarding | **IPv6 only** — loopbacks `fc00:0:N::1`, links `2001:XY::/64`, locators `fcbb:bb00:00NN::/48` (shared F3216 block) |
| **Management** | OOB SSH via CML bridge (`Mgmt-intf` VRF) | **May be dual-stacked** — IPv4 DHCP is configured today; IPv6 on mgmt is optional and stays in the VRF (not in IS-IS) |

---

## Quisted's method — how we teach, not what {#quisted-method}

Fidelity to [Quisted's MPLS-SR lab](https://www.quisted.net/index.php/2024/11/11/mpls-segment-routing-mpls-sr-lab/) is **not** “same commands, different address family.” It is the **proof arc** he uses: build belief layer by layer until the student *sees* forwarding work, then challenge it with a harder destination.

| Quisted's move | What the student learns | This lab's equivalent |
|----------------|-------------------------|------------------------|
| **Same canvas** | Topology is familiar; only the technology changes | Identical 8-router diamond; P1 still has 5 peers; ECMP to far PE via P3/P4 |
| **Underlay first** | “Do neighbors exist?” before any segment talk | Step 2 — `show isis neighbors` on every node |
| **Segments second** | “Are SIDs programmed?” | Step 3 — `show segment-routing srv6 locator` / `sid` |
| **FIB on P1** | “Where would *this* router send traffic?” | Step 4 + [P1 MPLS FIB ↔ SRv6](#p1-mpls-fib--srv6-verification) — `show isis ipv6 route`, CEF |
| **Easy destination first** | Loopback ping — IGP reachability only | Step 5.1 — `ping fc00:0:7::1` (not uSID yet) |
| **Endpoint-SID ping second** | Segment semantics matter | Step 5.3 — `ping fcbb:bb00:7::` (uN endpoint, not loopback) |
| **Walk the path** | Forwarding table at each hop matches intuition | Step 5.4 — [SRv6 show commands hop-by-hop](#srv6-path-walk) |
| **Wire proof** | Don't trust CLI alone | Step 5.2–3 — PCAP + optional debug on PE7 |
| **Save artifacts** | Evidence you can revisit | Named `.txt` / `.pcap`; `verification/results.json` |
| **Side-by-side (optional)** | Alumni map old reflexes to new | [Command translation table](#what-changed-from-the-original-mpls-sr-lab) below |

**What we deliberately swapped (the *what*):** OSPF/LDP/MPLS labels → IS-IS/SRv6 uSID. **What we kept (the *how*):** neighbor → segment → FIB → easy ping → endpoint-SID ping → path walk → capture.

**Endpoint-SID ping** (Step 5.3) proves PE7 uN reachability and local endpoint behavior; it does **not** demonstrate a multi-uSID segment list or explicit traffic engineering.

If you only read the translation table, you miss the point. If you follow Steps 2 → 5 in order, you get the same *confidence curve* Quisted builds — “I can explain every hop.”

---

## What changed from the original MPLS-SR lab {#what-changed}

| Original (Quisted MPLS-SR) | This lab (SRv6 uSID) |
|----------------------------|----------------------|
| OSPF area 0 underlay | IS-IS 100, `level-2-only` |
| LDP + MPLS forwarding | No LDP; SRv6 uSID |
| MPLS prefix-SID indexes | Per-node SRv6 locators |
| IOS XRv core + CSR1000v / **IOSv** PEs | IOS XRv 9000 core + **CAT 8000V** on all PEs |
| SR mapping server (SRMS) for IOSv PEs | **Not needed** — no SRv6 mapping server; PE7/PE8 upgraded to **CAT 8000V** so every router originates its own locator via IS-IS |
| `show ospf database` | `show isis database` |
| `show mpls ldp neighbor` | `show isis neighbors` |
| `show mpls forwarding` | `show isis ipv6 route`, `show segment-routing srv6 locator` |
| `traceroute mpls ipv4` | IPv6 `ping` / `traceroute` |
| CML 2.7 | CML **2.10** (parallel boot) |

**Topology parity is preserved:** P1 still has **5** IS-IS peers (P2, P3, P4, PE5, PE6). ECMP to the far-side PE (PE7) still goes via P3 and P4.

**What about non-SRv6 speakers?** Quisted teaches attaching legacy IOSv PEs via **SRMS** (a P router advertises prefix-SID mappings for routers that cannot speak MPLS-SR). **This lab has no non-speakers** — PE7/PE8 were upgraded to CAT 8000V so every node originates its own SRv6 locator. There is **no SRv6 SRMS**. In production, legacy edges are handled by **platform upgrade** (this lab), **SRv6↔SR-MPLS interworking gateway (IWG)**, **dual-connected PE** (MPLS + SRv6 on one box), or leaving them on a separate MPLS/LDP island — not by mapping their locators from a server.

---

## Lab download & import

### Labs download

| Option | File | Notes |
|--------|------|-------|
| **Mgmt (recommended)** | `quisted-srv6-mgmt-initial.yaml` | ext-conn + mgmt-sw OOB bridge |
| Data-plane only | `quisted-srv6-initial.yaml` | 8 routers, 14 links |
| Router configs | `configs/*.cfg` | Paste or import via YAML |

## SRv6 Lab Setup (Baseline)

Using Cisco's Modeling Labs (CML) I build the following SRv6 lab using **IS-IS** neighbor relationships.

- 2 × **PE router ( Left )** (PE5, PE6) running **CAT 8000V** with **IOS-XE**.
- 4 × **P router ( Center )** (P1, P2, P3, P4) running **IOS XRv 9000** with **IOS-XR**.
- 2 × **PE router ( Right )** (PE7, PE8) running **CAT 8000V** with **IOS-XE**.

### Interfaces

***Logical View:***

Eight routers in four columns × two rows inside a rounded frame. Node fill by zone: **light blue** (PE5, PE6), **dark grey** (#4a4558, P1–P4), **purple** (PE7, PE8). Each circle contains an asterisk/snowflake router glyph. Links are a **full mesh between adjacent columns** (PE↔P, P↔P, P↔PE), colored to match the zone: blue on the left, grey in the center (including vertical P1–P2 and P3–P4), purple on the right.

```
        ┌──────────────────────────────────────────────────┐
        │                                                  │
        │   (PE5)══════(P1)══════(P3)══════(PE7)           │
        │    ║ ╲        ║ ╲    ╱ ║ ╲        ║ ╲            │
        │    ║  ╲       ║  ║   ║  ║  ╲       ║  ╲           │
        │    ║   ╲      ║  ║   ║  ║   ╲      ║   ╲          │
        │   (PE6)══════(P2)══════(P4)══════(PE8)           │
        │                                                  │
        └──────────────────────────────────────────────────┘

  blue links: PE5/PE6 ↔ P1/P2          grey: P1–P4 full mesh (+ P1|P2, P3|P4)
  purple links: P3/P4 ↔ PE7/PE8
```

(Open `srv6-usid-isis-lab.html` for the color-coded SVG mesh diagram with interface labels on the wiring view below it.)

### Firmware

- **IOS-XE (left)** — PE5, PE6 on **CAT 8000V**
- **IOS-XR (center)** — P1, P2, P3, P4 on **IOS XRv 9000**
- **IOS-XE (right)** — PE7, PE8 on **CAT 8000V**

> **Note:** Quisted's original used **IOSv** on the right PEs; this lab uses **CAT 8000V IOS-XE** on all PE nodes for SRv6 uSID feature parity. Zone colors match Quisted's teaching layout (left = blue/teal, center = grey, right = purple) even though both PE zones run IOS-XE here.

### Prerequisites

CML **2.10.0+** (validated on build.7).

| Platform | CML image (validated) | Earliest documented support—platform dependent |
|----------|----------------------|------------------------------------------------|
| IOS XRv 9000 | `iosxrv9000-25-4-1` | IOS XR 7.3.1+ |
| CAT 8000V | `cat8000v-17-18-02` | IOS XE 17.12.1a+ |

> Listed CML images are versions validated for this lab; minimum release columns are vendor-documented baselines—confirm against your platform's SRv6 uSID feature matrix before production use.

### Import steps

1. **Import Lab** → `quisted-srv6-mgmt-initial.yaml`
2. **Start** lab (all 8 routers boot in parallel)
3. Wait **BOOTED** on all nodes (XR ~15–20 min each on cold start)
4. PE first boot: license + `reload`; XR: run `verification/complete_xr_setup.py` if prompted

---

## Node table

| Node | Role | Platform | CML image | Loopback IPv6 | SRv6 locator |
|------|------|----------|-----------|---------------|--------------|
| P1 | Core P | IOS-XR | iosxrv9000 | fc00:0:1::1/128 | fcbb:bb00:0001::/48 |
| P2 | Core P | IOS-XR | iosxrv9000 | fc00:0:2::1/128 | fcbb:bb00:0002::/48 |
| P3 | Core P | IOS-XR | iosxrv9000 | fc00:0:3::1/128 | fcbb:bb00:0003::/48 |
| P4 | Core P | IOS-XR | iosxrv9000 | fc00:0:4::1/128 | fcbb:bb00:0004::/48 |
| PE5 | Left PE | IOS-XE | cat8000v | fc00:0:5::1/128 | fcbb:bb00:0005::/48 |
| PE6 | Left PE | IOS-XE | cat8000v | fc00:0:6::1/128 | fcbb:bb00:0006::/48 |
| PE7 | Right PE | IOS-XE | cat8000v | fc00:0:7::1/128 | fcbb:bb00:0007::/48 |
| PE8 | Right PE | IOS-XE | cat8000v | fc00:0:8::1/128 | fcbb:bb00:0008::/48 |

**Addressing (live lab):** **Loopbacks** `fc00:0:N::1/128` for IS-IS reachability (Step 5.1 “easy” ping). **SRv6 locators:** all routers use F3216 block `fcbb:bb00:000N::/48` (`usid-f3216` on IOS-XE; IOS-XR may display `fcbb:bb00:N::/48`). PE uSID ping targets (Step 5.3) are **`fcbb:bb00:N::`** (e.g. `fcbb:bb00:7::`).

---

## IP addressing scheme

**IPv6-only data plane.** Quisted's original lab used IPv4 `10.x` links and `x.x.x.x/32` loopbacks for MPLS-SR; this lab uses IPv6 only on loopbacks and point-to-point links. The **management interface** (`Mgmt-intf` VRF) is separate — it **may be dual-stacked**; day-0 configs use IPv4 DHCP on XR `MgmtEth` / PE `GigabitEthernet3` for OOB SSH via the CML bridge.

The point-to-point links use this IPv6 addressing scheme:

- **Links:** `2001:**X****Y**::/64` where **X** and **Y** are the two endpoint router IDs (lowest first, highest second in the prefix)
- **Host on each side:** `2001:**X****Y**::**&lt;Router Id&gt;**/64`

> In the HTML guide, **X** = blue (low ID), **Y** = purple (high ID), and the host nibble = magenta (own router ID).

For example the link between P1 and P2 uses prefix `2001:**1****2**::/64`; on P1: `2001:**1****2**::**1**/64` and on P2: `2001:**1****2**::**2**/64`. The link between P1 and PE5 uses `2001:**1****5**::/64`; on P1: `2001:**1****5**::**1**/64` and on PE5: `2001:**1****5**::**5**/64`.

### Link inventory

| Link | XR/XE interface (P1 side) | IPv6 (P1 / peer) |
|------|---------------------------|------------------|
| P1–P2 | Gi0/0/0/0 | `2001:**1****2**::**1**` / `2001:**1****2**::**2**` |
| P1–P3 | Gi0/0/0/1 | `2001:**1****3**::**1**` / `2001:**1****3**::**3**` |
| P1–P4 | Gi0/0/0/2 | `2001:**1****4**::**1**` / `2001:**1****4**::**4**` |
| P1–PE5 | Gi0/0/0/3 | `2001:**1****5**::**1**` / `2001:**1****5**::**5**` |
| P1–PE6 | Gi0/0/0/4 | `2001:**1****6**::**1**` / `2001:**1****6**::**6**` |
| P2–P3 | Gi0/0/0/2 | `2001:**2****3**::**2**` / `2001:**2****3**::**3**` |
| P2–P4 | Gi0/0/0/1 | `2001:**2****4**::**2**` / `2001:**2****4**::**4**` |
| P2–PE5 | Gi0/0/0/4 | `2001:**2****5**::**2**` / `2001:**2****5**::**5**` |
| P2–PE6 | Gi0/0/0/3 | `2001:**2****6**::**2**` / `2001:**2****6**::**6**` |
| P3–P4 | Gi0/0/0/0 | `2001:**3****4**::**3**` / `2001:**3****4**::**4**` |
| P3–PE7 | Gi0/0/0/3 | `2001:**3****7**::**3**` / `2001:**3****7**::**7**` |
| P3–PE8 | Gi0/0/0/4 | `2001:**3****8**::**3**` / `2001:**3****8**::**8**` |
| P4–PE7 | Gi0/0/0/4 | `2001:**4****7**::**4**` / `2001:**4****7**::**7**` |
| P4–PE8 | Gi0/0/0/3 | `2001:**4****8**::**4**` / `2001:**4****8**::**8**` |

### Quisted IPv4 link addressing (pedagogical map)

> If you completed [Quisted's MPLS-SR lab](https://www.quisted.net/index.php/2024/11/11/mpls-segment-routing-mpls-sr-lab/), point-to-point links used **10.`<Lowest Router Id>`.`<Highest Router Id>`.`<Router Id>`./24`** — same diamond wiring, IPv4 address family. The **lowest** and **highest** router IDs on the link define the subnet; each router's own ID is the host octet.

| Link | Quisted IPv4 subnet | Lower-ID router | Higher-ID router |
|------|---------------------|-----------------|------------------|
| P1–P2 | `10.1.2.0/24` | P1 → `10.1.2.1/24` | P2 → `10.1.2.2/24` |
| P1–PE5 | `10.1.5.0/24` | P1 → `10.1.5.1/24` | PE5 → `10.1.5.5/24` |
| P3–PE7 | `10.3.7.0/24` | P3 → `10.3.7.3/24` | PE7 → `10.3.7.7/24` |

This mapping is **pedagogical only** — it helps alumni of Quisted's post recognize subnets on the same diamond topology. The data plane remains **IPv6-only**; day-0 configs do not add Quisted's `10.x` on data interfaces.

### Interface naming

| Platform | Data interfaces | Management |
|----------|-----------------|------------|
| IOS XRv 9000 | `GigabitEthernet0/0/0/0` – `/4` | `MgmtEth0/RP0/CPU0/0` |
| CAT 8000V | `GigabitEthernet1`, `GigabitEthernet2` | `GigabitEthernet3` |

> **Note:** Quisted's original XR post used `Gi0/0/0/1`–`/5` for the same links. This lab uses slot base `/0`–`/4` per the lab YAML topology wiring.

**PE8 wiring (critical):** Gi1 → P3 (`2001:**3****8**::**8**`), Gi2 → P4 (`2001:**4****8**::**8**`). A Gi1/Gi2 swap breaks IS-IS on PE8.

---

## Management access

Path: `ext-conn` (System Bridge) → `mgmt-sw` → each node's management interface (DHCP on your management network).

| Node | Mgmt interface | SSH user | Password |
|------|----------------|----------|----------|
| P1–P4 | `MgmtEth0/RP0/CPU0/0` | `cisco` | `Cisc01@3` |
| PE5–PE8 | `GigabitEthernet3` | `cisco` | `cisco` |

Mgmt IPs are **DHCP leases** — they change when you redeploy the lab. Look them up in `verification/mgmt_hosts.json`, or query live from CML (PyATS `send_cli_command`):

- **XR:** `show ip interface brief | include MgmtEth`
- **XE:** `show ip interface brief | include GigabitEthernet3`

```bash
python verification/discover_mgmt.py      # refresh mgmt_hosts.json from CML PyATS
python verification/ssh_check_mgmt.py       # expect 8/8 reachable
python verification/ssh_check_mgmt.py --refresh   # discover from CML, then SSH check
```

SSH from your workstation:

```bash
ssh cisco@<mgmt-ip>
```

**SecureCRT (Windows) — all 8 routers in one window** — `verification/connect_securecrt.ps1` discovers mgmt IPs (optional `-Refresh`) and opens tabs with passwords from env (not saved sessions):

```powershell
$env:LAB_XR_SSH_PASSWORD = 'Cisc01@3'   # P1–P4 (R1–R4)
$env:LAB_XE_SSH_PASSWORD = 'cisco'       # PE5–PE8
$env:CML_PASSWORD = '<cml-admin-password>'   # only if using -Refresh
.\connect-lab.ps1 -Refresh
# or from anywhere in repo: .\verification\connect_securecrt.ps1 -Refresh
# standalone package: quisted-srv6-usid-lab\connect-lab.ps1
```

Lab alias: **P1 = R1**, P2 = R2, P3 = R3, P4 = R4.

### OOB SSH day-0 config

**IOS-XRv (P1–P4)** — SSH listens on the mgmt VRF:

```text
ssh server v2
ssh server vrf Mgmt-intf
interface MgmtEth0/RP0/CPU0/0
 vrf Mgmt-intf
 ipv4 address dhcp
```

**IOS-XE (PE5–PE8)** — requires an RSA key and mgmt source-interface (Gi3 is in `Mgmt-intf` VRF):

```text
ip domain name lab.local
interface GigabitEthernet3
 vrf forwarding Mgmt-intf
 ip address dhcp
!
crypto key generate rsa general-keys modulus 3072
ip ssh version 2
ip ssh source-interface GigabitEthernet3
line vty 0 15
 exec-timeout 0 0
 login local
 transport input ssh
```

Without the RSA key, `show ip ssh` reports **SSH Disabled** and port 22 on the PE mgmt IP stays closed.

---

## Lab configs (reference)

Configs are **included in the CML import** (`quisted-srv6-mgmt-initial.yaml`). You can also paste from `configs/*.cfg` or regenerate:

```bash
python generate-configs.py
python build-quisted-srv6-yaml.py
```

### Per-node variables

Substitute these values on each router (interfaces are in `topology.py` / `configs/`):

| Node | NET (`49.0001.0000.0000.NNNN.00`) | Loopback IPv6 | SRv6 locator | Config file |
|------|-----------------------------------|---------------|--------------|-------------|
| P1 | 0001 | fc00:0:1::1/128 | fcbb:bb00:0001::/48 | `configs/p1.cfg` |
| P2 | 0002 | fc00:0:2::1/128 | fcbb:bb00:0002::/48 | `configs/p2.cfg` |
| P3 | 0003 | fc00:0:3::1/128 | fcbb:bb00:0003::/48 | `configs/p3.cfg` |
| P4 | 0004 | fc00:0:4::1/128 | fcbb:bb00:0004::/48 | `configs/p4.cfg` |
| PE5 | 0005 | fc00:0:5::1/128 | fcbb:bb00:0005::/48 | `configs/pe5.cfg` |
| PE6 | 0006 | fc00:0:6::1/128 | fcbb:bb00:0006::/48 | `configs/pe6.cfg` |
| PE7 | 0007 | fc00:0:7::1/128 | fcbb:bb00:0007::/48 | `configs/pe7.cfg` |
| PE8 | 0008 | fc00:0:8::1/128 | fcbb:bb00:0008::/48 | `configs/pe8.cfg` |

**PE8 wiring:** Gi1 → P3 (`2001:38::8`), Gi2 → P4 (`2001:48::8`).

### Full config — P1 (IOS-XR)

IPv6-only data plane; mgmt VRF retains IPv4 DHCP for OOB SSH. Authoritative copy: `configs/p1.cfg`.

```text
hostname P1
logging console disable
service timestamps log datetime msec
service timestamps debug datetime msec
domain name lab.local
domain lookup disable
ssh server v2
username cisco
 group root-lr
 group cisco-support
 secret Cisc01@3
!
line console
 exec-timeout 0 0
 login authentication default
!
line default
 exec-timeout 0 0
 login authentication default
!
aaa authentication login default local
!
vrf Mgmt-intf
 address-family ipv4 unicast
 !
!
interface MgmtEth0/RP0/CPU0/0
 vrf Mgmt-intf
 ipv4 address dhcp
 no shutdown
!
ssh server vrf Mgmt-intf
!
interface Loopback0
 ipv6 address fc00:0:1::1/128
!
interface GigabitEthernet0/0/0/0
 description to P2
 ipv6 address 2001:12::1/64
 no shutdown
!
interface GigabitEthernet0/0/0/1
 description to P3
 ipv6 address 2001:13::1/64
 no shutdown
!
interface GigabitEthernet0/0/0/2
 description to P4
 ipv6 address 2001:14::1/64
 no shutdown
!
interface GigabitEthernet0/0/0/3
 description to PE5
 ipv6 address 2001:15::1/64
 no shutdown
!
interface GigabitEthernet0/0/0/4
 description to PE6
 ipv6 address 2001:16::1/64
 no shutdown
!
segment-routing
 srv6
  encapsulation
   source-address fc00:0:1::1
  !
  locators
   locator MAIN
    prefix fcbb:bb00:0001::/48
    micro-segment behavior unode psp-usd
   !
  !
 !
!
router isis 100
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 address-family ipv6 unicast
  metric-style wide
  segment-routing srv6
   locator MAIN
  !
 !
 interface Loopback0
  passive
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/4
  point-to-point
  address-family ipv6 unicast
  !
 !
!
end
```

### Full config — PE5 (IOS-XE)

```text
hostname PE5
no service config
no logging console
license boot level network-advantage addon dna-advantage
ip domain name lab.local
username cisco privilege 15 secret cisco
ipv6 unicast-routing
!
vrf definition Mgmt-intf
 address-family ipv4
 exit-address-family
!
interface Loopback0
 ipv6 address fc00:0:5::1/128
 ipv6 router isis 100
!
interface GigabitEthernet1
 description to P1
 ipv6 address 2001:15::5/64
 isis network point-to-point
 ipv6 router isis 100
 no shutdown
!
interface GigabitEthernet2
 description to P2
 ipv6 address 2001:25::5/64
 isis network point-to-point
 ipv6 router isis 100
 no shutdown
!
interface GigabitEthernet3
 description management (external bridge)
 vrf forwarding Mgmt-intf
 ip address dhcp
 no shutdown
!
crypto key generate rsa general-keys modulus 3072
ip ssh version 2
ip ssh source-interface GigabitEthernet3
line vty 0 15
 exec-timeout 0 0
 login local
 transport input ssh
!
segment-routing srv6
 encapsulation
  source-address fc00:0:5::1
 !
 locators
  locator MAIN
   format usid-f3216
   prefix fcbb:bb00:0005::/48
 !
!
router isis 100
 net 49.0001.0000.0000.0005.00
 is-type level-2-only
 metric-style wide
 address-family ipv6 unicast
  multi-topology
  segment-routing srv6
   locator MAIN
  !
 !
end
```

P2–P4 and PE6–PE8: use the matching file under `configs/` (same structure, different addresses/NET/locator).

**First boot on PE:** after license line, `reload` once. If SRv6 commands are missing, run `python verification/push_pe_srv6.py`.

---

## Lab procedure — configure and verify

Work through these steps on a **booted** lab. SSH via mgmt (`verification/mgmt_hosts.json`).

### How we prove this lab — run order

Quisted did not stop at “ping works.” He built **evidence stacks**: IGP → label/SID table → forwarding lookup on P1 → ping → traceroute → (in this lab) wire capture. **This lab uses the same teaching order** with SRv6 commands — see [Quisted's method](#quisted-method).

| # | What | Where | Artifact |
|---|------|-------|----------|
| **1** | **Before ping — CLI output** | PE5 | Loopback ping to `fc00:0:7::1` — screenshot/save terminal |
| **2** | **Arm debug** | CML + PE7 | Start PCAP (P1–P4); on PE7 (IOS-XE): `terminal monitor`, `logging monitor debugging`, `debug ipv6 icmp`, `debug ipv6 packet detail` |
| **3** | **Then ping** | PE5 | uSID ping to `fcbb:bb00:7::` while capture + debug run — save CLI + PCAP |
| **4** | **Walk the path** | PE5, P1, P3, PE7 | SRv6 / IS-IS show commands at each hop (control-plane receipt for step 3) |

**Before** = loopback destination in CLI only. **After** = uSID destination on the wire (PCAP Dst `fcbb:bb00:7::`) plus optional debug on PE7. **Step 4** ties the traceroute hops to locator routes and micro-SIDs — see [Walk the path with SRv6 show commands](#srv6-path-walk).

On **PE7** before step 2: `show segment-routing srv6 sid` — expect `FCBB:BB00:7::` **uN (PSP/USD)**.

**Save artifacts:** `pe5-pe7-loopback-cli.txt`, `pe5-pe7-usid-cli.txt`, `pe5-pe7-usid.pcap`, PE7 debug log.

### Step 1 — Mgmt reachability (all nodes)

```bash
python verification/ssh_check_mgmt.py
```

| Node | Command | Pass |
|------|---------|------|
| All 8 | SSH login | 8/8 reachable |

### Step 2 — IS-IS underlay

**IPv6-only on the data plane.** `router isis 100` uses `address-family ipv6 unicast` only — no IPv4 AF, no `ip router isis` on PE data interfaces. (Mgmt VRF IPv4 DHCP is OOB only and is not in IS-IS.)

**Quisted equivalent:** `show ospf database` (8 routers in area 0) + `show mpls interfaces` (LDP Yes on links).  
**This lab:** `show isis database` on **P1** + `show isis neighbors` on **every** router.

On **each** router:

```
show isis neighbors
```

| Node | Expected neighbors Up | Peers |
|------|----------------------|-------|
| P1 | 5 | P2, P3, P4, PE5, PE6 |
| P2 | 5 | P1, P3, P4, PE5, PE6 |
| P3 | 5 | P1, P2, P4, PE7, PE8 |
| P4 | 5 | P1, P2, P3, PE7, PE8 |
| PE5 | 2 | P1, P2 |
| PE6 | 2 | P1, P2 |
| PE7 | 2 | P3, P4 |
| PE8 | 2 | P3, P4 |

On **P1**:

```
show isis database
```

**Pass:** 8 LSP IDs (P1–P4, PE5–PE8).

**P1 — `show isis database`:**

```
RP/0/RP0/CPU0:P1# show isis database

IS-IS 100 (Level-2) Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime  ATT/P/OL
P1.00-00              0x00000042   0x8a3f        892           0/0/0
P2.00-00              0x00000038   0xb21c        845           0/0/0
P3.00-00              0x00000031   0xc104        901           0/0/0
P4.00-00              0x00000029   0xd5e8        867           0/0/0
PE5.00-00             0x0000001a   0xe6b2        823           0/0/0
PE6.00-00             0x00000015   0xf104        856           0/0/0
PE7.00-00             0x00000012   0xa8c3        889           0/0/0
PE8.00-00             0x0000000f   0xb9d1        834           0/0/0

    Total LSP count: 8
```

**What to check:** Same eight router IDs as Quisted’s `show ospf database` — proof the **whole mesh** is in one IS-IS level-2 domain before any SRv6 talk.

**P1 — `show isis neighbors`:**

```
IS-IS 100 neighbors:
System Id      Interface        SNPA           State Holdtime Type IETF-NSF
P2             Gi0/0/0/0        *PtoP*         Up    22       L2   Capable
P3             Gi0/0/0/1        *PtoP*         Up    24       L2   Capable
P4             Gi0/0/0/2        *PtoP*         Up    27       L2   Capable
PE5            Gi0/0/0/3        *PtoP*         Up    27       L2   Capable
PE6            Gi0/0/0/4        *PtoP*         Up    24       L2   Capable

Total neighbor count: 5
```

**PE5 live output:**

```
Tag 100:
System Id       Type Interface     IP Address      State Holdtime Circuit Id
P1              L2   Gi1           2001:15::1      UP    21       00
P2              L2   Gi2           2001:25::2      UP    29       00
```

**What to check:** Both adjacencies **UP**, **L2**, IPv6 link addresses on Gi1/Gi2 (not `10.x`). Replaces Quisted’s `show mpls ldp neighbor` on PE5.

### Step 3 — SRv6 uSID locators

**Quisted equivalent:** per-router **Label Ranges** (e.g. P1 = `24100–24199`) + `show mpls label range` / prefix-SID blocks.  
**This lab:** each router owns a **locator** `/48` plus local **uN/uA SIDs**.

> **Live-validated (Jul 15–16, SSH on CML):** All nodes use F3216 locator **`fcbb:bb00:000N::/48`** (`usid-f3216` on IOS-XE). IOS-XR may display `fcbb:bb00:N::/48` (leading zeros omitted). Re-collected Jul 16–17 on lab `eeee6ffa`; full artifact: `verification/quisted-srv6/proof_locator_fcbb.txt`.

#### Live proof: unified fcbb locators on P and PE

P routers and PEs share the same **`fcbb:bb00`** uSID block by design in `topology.py`. Loopback0 remains **`fc00:0:N::1/128`** for IGP reachability — that prefix is **not** the SRv6 locator. Seeing `fc00` on Loopback0 alongside `fcbb` locators is intentional: infrastructure addresses and segment IDs live in separate namespaces (analogous to Quisted loopbacks vs per-router label ranges).

| Node | Locator prefix | Status | uN SID | Loopback (fc00) |
|------|----------------|--------|--------|-----------------|
| P1 | `fcbb:bb00:1::/48` | Up (intended fcbb) | `fcbb:bb00:1::` | `fc00:0:1::1` |
| P2 | `fcbb:bb00:2::/48` | Up (intended fcbb) | `fcbb:bb00:2::` | `fc00:0:2::1` |
| P3 | `fcbb:bb00:3::/48` | Up (intended fcbb) | `fcbb:bb00:3::` | `fc00:0:3::1` |
| P4 | `fcbb:bb00:4::/48` | Up (intended fcbb) | `fcbb:bb00:4::` | `fc00:0:4::1` |
| PE5 | `FCBB:BB00:5::/48` | Up, `usid-f3216` | `FCBB:BB00:5::` | `fc00:0:5::1` |
| PE7 | `FCBB:BB00:7::/48` | Up, `usid-f3216` | `FCBB:BB00:7::` | `fc00:0:7::1` |

**P1 — locator + uN SID (Jul 17, lab eeee6ffa):**

```
RP/0/RP0/CPU0:P1# show segment-routing srv6 locator

Name                  ID       Algo  Prefix                    Status   Flags
MAIN                  1        0     fcbb:bb00:1::/48          Up       U

RP/0/RP0/CPU0:P1# show segment-routing srv6 sid

*** Locator: 'MAIN' ***

SID                         Behavior          Identification                    Owner               State  RW
fcbb:bb00:1::               uN (PSP/USD)      'default':1                       sidmgr              InUse  Y
fcbb:bb00:1:e000::          uA (PSP/USD)      [Gi0/0/0/3, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e001::          uA (PSP/USD)      [Gi0/0/0/4, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e002::          uA (PSP/USD)      [Gi0/0/0/0, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e003::          uA (PSP/USD)      [Gi0/0/0/2, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e004::          uA (PSP/USD)      [Gi0/0/0/1, Link-Local]:0         isis-100            InUse  Y

RP/0/RP0/CPU0:P1# show ipv6 interface loopback0 brief

Loopback0              [Up/Up]
    fc00:0:1::1
```

**PE5 — uSID ping to PE7 uN (5/5):**

```
PE5# ping fcbb:bb00:7:: source fc00:0:5::1 repeat 5

Sending 5, 100-byte ICMP Echos to FCBB:BB00:7::, timeout is 2 seconds:
Packet sent with a source address of FC00:0:5::1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/4/6 ms
```

**Pass:** P1–P4 locators Up under `fcbb`; loopbacks stay `fc00`; PE5 uSID ping 5/5.

On **every** router:

```
show segment-routing srv6 locator
```

| Node | Expected prefix (live) | Status |
|------|------------------------|--------|
| P1–P4 | `fcbb:bb00:000N::/48` | Up |
| PE5–PE8 | `fcbb:bb00:000N::/48` | Up, `usid-f3216` (XE) |

**Pass:** locator **Up** on all 8 nodes; IOS-XE shows format **`usid-f3216`**.

On **P1** and **PE5** (representative XR + XE):

```
show segment-routing srv6 sid
```

**Pass (P1):** local uN `fcbb:bb00:1::` + uA SIDs `fcbb:bb00:1:e005::` … `e009::` on Gi0/0/0/0–/4.

**P1 — `show segment-routing srv6 locator`:**

```
Name                  ID       Algo  Prefix                    Status   Flags
MAIN                  1        0     fcbb:bb00:1::/48             Up       U
```

**P1 — `show segment-routing srv6 sid` (live):**

```
*** Locator: 'MAIN' ***

SID                         Behavior          Identification                    Owner               State  RW
fcbb:bb00:1::                  uN (PSP/USD)      'default':1                       sidmgr              InUse  Y
fcbb:bb00:1:e005::             uA (PSP/USD)      [Gi0/0/0/0, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e006::             uA (PSP/USD)      [Gi0/0/0/3, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e007::             uA (PSP/USD)      [Gi0/0/0/4, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e008::             uA (PSP/USD)      [Gi0/0/0/1, Link-Local]:0         isis-100            InUse  Y
fcbb:bb00:1:e009::             uA (PSP/USD)      [Gi0/0/0/2, Link-Local]:0         isis-100            InUse  Y
```

**PE5 — `show segment-routing srv6 locator`:**

```
Name            Algo Prefix                                   Format               Status
MAIN            0    FCBB:BB00:5::/48                         usid-f3216           Up
```

**PE5 — `show segment-routing srv6 sid` (live):**

```
FCBB:BB00:5::          uN (PSP/USD)                              SID-MGR
FCBB:BB00:5:E002::     uA (PSP/USD)   Gi1 FE80:0:1::1 → P1      router isis 100
FCBB:BB00:5:E003::     uA (PSP/USD)   Gi2 FE80:0:2::1 → P2      router isis 100
```

**What to check:** Unified F3216 locators on **`fcbb:bb00`** for P and PE. uA SIDs bind to IS-IS adjacencies (`isis-100` / `router isis 100`).

After locators are Up on all nodes, continue to [Step 5 §4 — Walk the path](#srv6-path-walk) to trace PE7’s locator hop-by-hop through the core.

### Step 4 — Core forwarding (P1)

**Quisted equivalent:** `show mpls forwarding` + `show cef` for `2.2.2.2/32`, `5.5.5.5/32`, `7.7.7.7/32`.  
**This lab:** IS-IS IPv6 routes + SRv6 SID table on **P1** (core lookup point).

On **P1**:

```
show isis ipv6 route fc00:0:5::1/128
show isis ipv6 route fc00:0:7::1/128
show segment-routing srv6 sid
show cef ipv6 fc00:0:2::1/128
```

**Pass:** PE5 via Gi0/0/0/3; PE7 **ECMP** via Gi0/0/0/1 (P3) and Gi0/0/0/2 (P4). See also [P1 MPLS FIB ↔ SRv6 verification](#p1-mpls-fib--srv6-verification).

**P1 — route to PE5 (direct attachment):**

```
L2 fc00:0:5::1/128 [20/115]
     via fe80:0:5::1, GigabitEthernet0/0/0/3, PE5, Weight: 0
```

**P1 — route to PE7 (ECMP via P3 and P4):**

```
L2 fc00:0:7::1/128 [30/115]
     via fe80:0:4::1, GigabitEthernet0/0/0/2, P4, Weight: 0
     via fe80:0:3::1, GigabitEthernet0/0/0/1, P3, Weight: 0
```

**P1 — CEF for P2 loopback (adjacency FIB):**

```
fc00:0:2::1/128 ...
 remote adjacency to GigabitEthernet0/0/0/0
   via fe80:0:2::1/128, GigabitEthernet0/0/0/0
```

**What to check:** No MPLS labels; next-hops are **IPv6 link-local** on point-to-point IS-IS interfaces. ECMP to PE7 must show **two** outgoing interfaces.

### Step 5 — Proof run (PE5 → PE7) {#proof-run}

| Address | Role |
|---------|------|
| `fc00:0:7::1` | PE7 loopback — normal IS-IS IPv6 route (**before** CLI ping) |
| `fcbb:bb00:7::` | PE7 **uN** micro-SID from locator `fcbb:bb00:0007::/48` (**then ping** target) |

#### 1. Before ping — CLI output (PE5)

Ping the **IS-IS loopback** first. Save the terminal output — this is your **before** evidence (no PCAP yet).

```
ping fc00:0:7::1 source fc00:0:5::1 repeat 5
traceroute fc00:0:7::1 source fc00:0:5::1
```

**Pass:** `Success rate is 100 percent (5/5)`; traceroute shows core hops (e.g. P1 → P3 → PE7).

Example output to capture:

```
Sending 5, 100-byte ICMP Echos to FC00:0:7::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms
```

This proves IS-IS IPv6 underlay — **not** uSID steering.

#### 2. Arm debug

**CML (same link for every capture):**

1. Lab **BOOTED** on all nodes.
2. **Right-click** link on PE5→PE7 path (recommended: **P1 ↔ P4**, P1 `GigabitEthernet0/0/0/2`).
3. **Packet capture** → **Start** (up to 5000 packets). Leave running.

**PE7 (uN endpoint — cat8000v IOS-XE):**

> **Platform note:** PE nodes run **cat8000v IOS-XE**; P nodes run **IOS-XR**. Do not mix debug syntax — `debug segment-routing srv6` is valid on IOS-XR but returns `% Unrecognized command` on PE7.

```
show segment-routing srv6 sid
terminal monitor
logging monitor debugging
debug ipv6 icmp
debug ipv6 packet detail
```

Expect `FCBB:BB00:7::` with behavior **uN (PSP/USD)** before you ping. Confirm with `show debug` — expect ICMPv6 and IPv6 unicast packet debugging **on**.

#### 3. Then ping — Endpoint-SID (PE5)

With capture **still running** and debug **still on** PE7. This **endpoint-SID ping** proves PE7 uN reachability and local endpoint behavior; it does **not** demonstrate a multi-uSID segment list or explicit traffic engineering.

```
ping fcbb:bb00:7:: source fc00:0:5::1 repeat 10
traceroute fcbb:bb00:7:: source fc00:0:5::1
```

**Pass:** ping 100%; traceroute shows core hops. Save CLI output again.

**Example output:**

```
Sending 10, 100-byte ICMP Echos to FCBB:BB00:7::, timeout is 2 seconds:
!!!!!!!!!!
Success rate is 100 percent (10/10)

Tracing the route to FCBB:BB00:7::
  1 2001:15::1 4 msec 3 msec 3 msec
  2 2001:13::3 5 msec
    2001:14::4 5 msec 4 msec
  3 2001:37::7 4 msec 4 msec 4 msec
```

**What to look for** (on PE7 while debug is running, or immediately after ping):

| Check | Command / log line | Proof |
|-------|-------------------|-------|
| uN route via SR0 | `show ipv6 route fcbb:bb00:7::` | `receive via SR0` (not Loopback0) |
| Endpoint processing | `show logging` (last 30) | `Dst=FCBB:BB00:7::`, `if_output=SR0`; reply `Src=FCBB:BB00:7::` |
| SID programmed | `show segment-routing srv6 sid` | `FCBB:BB00:7::` **uN (PSP/USD)** |

Example `show logging` lines from a live capture:

```
*Jul 15 17:54:18.913: IPv6-Fwd: Destination lookup for FCBB:BB00:7:: : Local, i/f=SR0, nexthop=FCBB:BB00:7::
*Jul 15 17:54:18.914: ICMPv6: Echo request received,  Src: FC00:0:5::1 Dst: FCBB:BB00:7:: if_input: GigabitEthernet2 if_output: SR0
*Jul 15 17:54:18.914: ICMPv6: Sent echo reply, Src=FCBB:BB00:7::, Dst=FC00:0:5::1
```

Example `show ipv6 route` on PE7:

```
Routing entry for FCBB:BB00:7::/128
  Known via "connected", distance 0, metric 0, type receive
  Routing paths:
    receive via SR0
```

**Stop instruments:**

1. CML → stop capture → select ICMPv6 Echo request → export/screenshot.
2. PE7 → `undebug all`
3. PE7 → `no terminal monitor`

| PCAP field (step 3 traffic) | Expected |
|-----------------------------|----------|
| **Source** | `fc00:0:5::1` |
| **Destination** | `fcbb:bb00:7::` (uN micro-SID — **not** `fc00:0:7::1`) |
| **Next Header** | ICMPv6 (58) — no SRH on IGP shortest path |
| **Protocol** | ICMPv6 — **no MPLS** |

Echo **reply** swaps addresses: Src `fcbb:bb00:7::`, Dst `fc00:0:5::1`.

**Where is the micro-SID?** It **is** the IPv6 destination `fcbb:bb00:7::` — no separate uSID header in baseline mode. Transit routers match locator `/48`; PE7 performs uN endpoint processing.

**Contrast for announcements:** Step 1 CLI shows Dst `fc00:0:7::1`. Step 3 PCAP shows Dst `fcbb:bb00:7::` on the same path.

This proves the packet destination matches an **instantiated SID** (uN endpoint processing). It does not impose an explicit compressed segment list — that requires SR-TE policy (see [Optional: SR-TE uSID policy](#optional-sr-te-usid-policy-path-steering)).

#### 4. Walk the path with SRv6 show commands {#srv6-path-walk}

After step 3, **walk the same path on the control plane**. The uSID traceroute shows *where* packets go; these commands show *why* each hop can forward to `fcbb:bb00:7::`.

**Path:** PE5 → P1 → P3 (or P4, ECMP) → PE7  
**Destination under test:** `fcbb:bb00:7::/48` (PE7 uSID locator; micro-SID `fcbb:bb00:7::` is the uN endpoint inside that block)

##### Command cheat sheet (platform differences)

| Hop | Platform | Route to PE7 locator | IS-IS SRv6 locator | Local SID table |
|-----|----------|----------------------|--------------------|-----------------|
| PE5 | IOS-XE | `show ipv6 route fcbb:bb00:7::` | — (use IPv6 route) | `show segment-routing srv6 sid` |
| P1, P3, P4 | IOS-XR | `show isis ipv6 route fcbb:bb00:7::/48` | `show isis segment-routing srv6 locators` | `show segment-routing srv6 sid` |
| PE7 | IOS-XE | `show ipv6 route fcbb:bb00:7::` → **receive via SR0** | — | `show segment-routing srv6 sid` |

**CLI quirk (IOS-XRv):** `show isis segment-routing srv6` alone returns `% Incomplete command.` — append **`locators`** (or **`locator`**; same table on this image).

##### Step 0 — Origin: PE5

```
show ipv6 route fcbb:bb00:7::
show segment-routing srv6 locator
show segment-routing srv6 sid
```

**Pass — IS-IS learned PE7’s locator; ECMP toward P1 and P2:**

```
Routing entry for FCBB:BB00:7::/48
  Known via "isis 100", distance 115, metric 30, type level-2
  Routing paths:
    FE80:0:1::1, GigabitEthernet1
    FE80:0:2::1, GigabitEthernet2

MAIN   FCBB:BB00:5::/48   usid-f3216   Up

FCBB:BB00:5::          uN (PSP/USD)                              SID-MGR
FCBB:BB00:5:E002::     uA (PSP/USD)   Gi1 FE80:0:1::1 → P1      router isis 100
FCBB:BB00:5:E003::     uA (PSP/USD)   Gi2 FE80:0:2::1 → P2      router isis 100
```

**Read:** PE5 forwards to PE7’s **locator prefix** via link-local next-hops on Gi1/Gi2. Adjacency uA SIDs are owned by `router isis 100`.

##### Step 1 — Transit: P1

```
show isis ipv6 route fcbb:bb00:7::/48
show isis segment-routing srv6 locators
show segment-routing srv6 sid
```

**Pass — ECMP toward P3 and P4 (matches traceroute hop 2):**

```
L2 fcbb:bb00:7::/48 [20/115]
     via fe80:0:4::1, GigabitEthernet0/0/0/2, P4, Weight: 0
     via fe80:0:3::1, GigabitEthernet0/0/0/1, P3, Weight: 0

IS-IS 100 SRv6 Locators
MAIN   fcbb:bb00:1::/48   Active

fcbb:bb00:1::           uN (PSP/USD)   sidmgr
fcbb:bb00:1:e008::      uA (PSP/USD)   [Gi0/0/0/1] → P3    isis-100
fcbb:bb00:1:e009::      uA (PSP/USD)   [Gi0/0/0/2] → P4    isis-100
```

**Read:** P1 is a **transit** node — local locator **`fcbb:bb00:1::/48`**, IS-IS route to PE7’s **`fcbb:bb00:7::/48`**, uA SIDs on links toward P3/P4.

##### Step 2 — Transit: P3 (ECMP leg; repeat on P4 for the alternate path)

```
show isis ipv6 route fcbb:bb00:7::/48
show isis segment-routing srv6 locators
show segment-routing srv6 sid
```

**Pass — one hop from PE7:**

```
L2 fcbb:bb00:7::/48 [10/115]
     via fe80:0:7::1, GigabitEthernet0/0/0/3, PE7, Weight: 0

fcbb:bb00:3::           uN (PSP/USD)   sidmgr
fc00:0:3:e008::      uA (PSP/USD)   [Gi0/0/0/3] → PE7   isis-100
```

**Read:** P3 forwards `fcbb:bb00:7::/48` directly to PE7. Baseline uSID has no SRH — IPv6 destination = segment; transit routers match the locator `/48` and forward.

##### Step 3 — Destination: PE7

```
show segment-routing srv6 locator
show segment-routing srv6 sid
show ipv6 route fcbb:bb00:7::
```

**Pass — local uN endpoint (packet stops here):**

```
MAIN   FCBB:BB00:7::/48   usid-f3216   Up

FCBB:BB00:7::          uN (PSP/USD)                              SID-MGR
FCBB:BB00:7:E002::     uA (PSP/USD)   Gi1 FE80:0:3::1 → P3
FCBB:BB00:7:E003::     uA (PSP/USD)   Gi2 FE80:0:4::1 → P4

Routing entry for FCBB:BB00:7::/128
  Known via "connected", type receive
  receive via SR0
```

**Read:** `fcbb:bb00:7::` is the **local uN** on PE7 (`receive via SR0`) — uSID endpoint processing, not another forward hop.

##### Control-plane path diagram

```
PE5                    P1                     P3                     PE7
fcbb:bb00:0005::/48    fcbb:bb00:1::/48          fcbb:bb00:3::/48          fcbb:bb00:0007::/48
     |                      |                      |                      |
     | isis → fcbb:bb00:7::/48                      |                      | connected uN
     v                      v                      v                      v
 Gi1 fe80:0:1::1 ──► Gi0/0/0/3 (PE5)    Gi0/0/0/1 fe80:0:3::1 ──► Gi0/0/0/3 fe80:0:7::1
```

##### Tie to data plane (step 3 traceroute)

From PE5, the same path in the data plane:

```
traceroute fcbb:bb00:7:: source fc00:0:5::1
  1 2001:15::1                      ← P1
  2 2001:13::3 / 2001:14::4         ← P3 / P4 ECMP
  3 2001:37::7 / 2001:47::7         ← PE7 uSID endpoint
```

Control plane (IS-IS locator routes + uSIDs) and data plane (traceroute) must tell the same story. If they diverge, check IS-IS adjacencies (Step 2) before debugging SRv6.

**Per-node automation:** `python verification/node_swarm.py P3` (single router) or `python verification/run_swarms.py` (all 8 in parallel).

#### Optional: PCAP with SR-TE (SRH visible)

After applying `configs/snippets/p1-srte-usid-policy.cfg` on P1, repeat **steps 2–3**. Look for **Next Header = 43** (SRH) and compressed uSID slots (F3216).

See [Optional: SR-TE uSID policy](#optional-sr-te-usid-policy-path-steering).

### Step 5 reference — loopback vs uSID {#usid-forwarding}

Quick reference for automated tests and reviewers:

| Test | Target | Proves |
|------|--------|--------|
| T6 / Step 5.1 | `fc00:0:7::1` | IGP loopback reachability |
| T11 / Step 5.3 | `fcbb:bb00:7::` | uSID uN endpoint |

Legacy section anchors: [PCAP details](#pcap-proof) · [Debug details](#debug-proof)

### PCAP details {#pcap-proof}

See **Step 5 §2–3** — capture runs during the uSID ping only. Tap **P1 ↔ P4** (or P1 Gi0/0/0/3). Pass: Dst `fcbb:bb00:7::`, ICMPv6, no MPLS.

### Debug details {#debug-proof}

See **Step 5 §2–3** — IOS-XE debug stack on **PE7** (`terminal monitor`, `logging monitor debugging`, `debug ipv6 icmp`, `debug ipv6 packet detail`) before the uSID ping; `undebug all` + `no terminal monitor` after. Transit (P1) debug is optional and often quiet; endpoint + PCAP carry the proof.

### Step 6 — Automated validation (optional)

```powershell
$env:VERIFY_SSH_ONLY = '1'
python verification/run_live.py
```

**Pass:** `overall: PASS` in `verification/results.json` (tests T1–T12).

---

## P1 MPLS FIB ↔ SRv6 verification

After enabling MPLS-SR, Quisted verified the data plane on P-1 with `show mpls forwarding`, `show cef`, `show ospf sid-database`, and `show mpls label table detail`. In this lab there is **no MPLS/LDP FIB** — use the commands below on **P1** instead.

**Prefix mapping:** `2.2.2.2/32` → `fc00:0:2::1/128` · `5.5.5.5/32` → `fc00:0:5::1/128` · `7.7.7.7/32` → `fc00:0:7::1/128`

| Quisted MPLS-SR on P-1 | Lab equivalent on P1 | What it proves |
|------------------------|----------------------|----------------|
| `show mpls forwarding` | `show isis ipv6 route <prefix>` | Per-prefix next-hop / ECMP |
| | `show segment-routing srv6 sid` | Local uN + uA SIDs installed |
| `show cef 2.2.2.2/32` | `show cef ipv6 fc00:0:2::1/128` | FIB entry for P2 loopback |
| | `show route ipv6 fc00:0:2::1/128` | RIB + outgoing interface |
| `show ospf sid-database` | `show isis database verbose` | Remote locators + END/END.X SIDs in LSPs |
| `show mpls label table detail` | `show segment-routing srv6 sid` | SID ownership, behavior, interface binding |

### 1. `show mpls forwarding` → `show isis ipv6 route`

**Route to P2** (was `2.2.2.2/32` via Gi0/0/0/1):

```
RP/0/RP0/CPU0:P1# show isis ipv6 route fc00:0:2::1/128

L2 fc00:0:2::1/128 [10/115]
     via fe80::5054:ff:fe09:a39e, GigabitEthernet0/0/0/0, P2, SRGB Base: 16000, Weight: 0
```

**Pass:** direct adjacency to P2 on Gi0/0/0/0

**Route to PE5** (was `24100 Pop 5.5.5.5/32 Gi… 10.1.5.5`):

```
RP/0/RP0/CPU0:P1# show isis ipv6 route fc00:0:5::1/128

L2 fc00:0:5::1/128 [20/115]
     via fe80::5054:ff:fe88:a50b, GigabitEthernet0/0/0/3, PE5, Weight: 0
```

**Pass:** direct PE5 attachment on Gi0/0/0/3

**ECMP to PE7** (was dual labels to `7.7.7.7/32` via P3/P4):

```
RP/0/RP0/CPU0:P1# show isis ipv6 route fc00:0:7::1/128

L2 fc00:0:7::1/128 [30/115]
     via fe80::5054:ff:fe58:612a, GigabitEthernet0/0/0/2, P4, SRGB Base: 16000, Weight: 0
     via fe80::5054:ff:fef5:c8d4, GigabitEthernet0/0/0/1, P3, SRGB Base: 16000, Weight: 0
```

**Pass:** two next-hops (ECMP) via Gi0/0/0/1 and Gi0/0/0/2

> P1 may still show a slim `show mpls forwarding` with IS-IS SR Adj labels (24000–24009) only — not the full MPLS-SR FIB from Quisted.

### 2. `show cef 2.2.2.2/32` → `show cef ipv6`

```
RP/0/RP0/CPU0:P1# show cef ipv6 fc00:0:2::1/128

fc00:0:2::1/128, version 34, internal ...
 remote adjacency to GigabitEthernet0/0/0/0
   via fe80::5054:ff:fe09:a39e/128, GigabitEthernet0/0/0/0
    next hop fe80::5054:ff:fe09:a39e/128
    remote adjacency
    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/0    remote
```

```
RP/0/RP0/CPU0:P1# show route ipv6 fc00:0:2::1/128

Routing entry for fc00:0:2::1/128
  Known via "isis 100", distance 115, metric 10, type level-2
    fe80::5054:ff:fe09:a39e, from fc00:0:2::1, via GigabitEthernet0/0/0/0
```

### 3. `show ospf sid-database` → `show isis database verbose`

**Local P1 LSP** (locator + node SID + adjacency SIDs):

```
  SRv6 Locator:   MT (IPv6 Unicast) fcbb:bb00:1::/48 ...
    END SID: fcbb:bb00:1:: uN (PSP/USD)
    END.X SID: fcbb:bb00:1:e002:: ... uA (PSP/USD)   ! Gi0/0/0/0 toward P2
    END.X SID: fcbb:bb00:1:e001:: ... uA (PSP/USD)   ! Gi0/0/0/1 toward P3
```

**Remote P2 LSP** (like prefix-SID index 2 for `2.2.2.2`):

```
RP/0/RP0/CPU0:P1# show isis database P2.00-00 detail
  IPv6 Address:   fc00:0:2::1
  SRv6 Locator:   MT (IPv6 Unicast) fcbb:bb00:2::/48 ...
  Metric: 0  MT (IPv6 Unicast) IPv6 fc00:0:2::1/128
```

### 4. `show mpls label table detail` → `show segment-routing srv6 sid`

```
RP/0/RP0/CPU0:P1# show segment-routing srv6 sid

SID                         Behavior          Identification                    Owner
fcbb:bb00:1::                  uN (PSP/USD)      'default':1                       sidmgr
fcbb:bb00:1:e000::             uA (PSP/USD)      [Gi0/0/0/2, Link-Local]:0         isis-100
fcbb:bb00:1:e001::             uA (PSP/USD)      [Gi0/0/0/1, Link-Local]:0         isis-100
fcbb:bb00:1:e002::             uA (PSP/USD)      [Gi0/0/0/0, Link-Local]:0         isis-100
fcbb:bb00:1:e003::             uA (PSP/USD)      [Gi0/0/0/3, Link-Local]:0         isis-100
fcbb:bb00:1:e004::             uA (PSP/USD)      [Gi0/0/0/4, Link-Local]:0         isis-100
```

| Quisted MPLS label table | Lab SRv6 SID table |
|--------------------------|-------------------|
| `SR Adj Segment … Gi0/0/0/1, nh=10.1.2.2` | `uA … [Gi0/0/0/0]` toward P2 |
| `SR Pfx (idx 2)` → label 16002 | `uN` at remote `fcbb:bb00:2::` (in P2 LSP) |
| SRGB 16000–23999 | Still in LSP (`SRGB Base: 16000`) for adj-SID remnants |

### Quick checklist on P1

```
! "mpls forwarding" equivalent
show isis ipv6 route fc00:0:2::1/128    ! P2  (was 2.2.2.2)
show isis ipv6 route fc00:0:5::1/128    ! PE5 (was 5.5.5.5)
show isis ipv6 route fc00:0:7::1/128    ! PE7 ECMP (was 7.7.7.7)

! "cef" equivalent
show cef ipv6 fc00:0:2::1/128

! "ospf sid-database" equivalent
show isis database verbose
show isis database P2.00-00 detail

! "mpls label table detail" equivalent
show segment-routing srv6 sid
show segment-routing srv6 locator
```

---

## Side-by-side: Quisted MPLS commands vs this lab

| Original on P-1 (MPLS-SR) | This lab on P1 (SRv6 uSID) | Expected parity |
|---------------------------|----------------------------|-----------------|
| `show ospf database` (8 routers) | `show isis database` (8 LSPs) | Same 8 router IDs |
| 5 LDP neighbors | 5 IS-IS neighbors Up | Same peer set: P2,P3,P4,PE5,PE6 |
| `show mpls forwarding` → `24100 Pop 5.5.5.5/32` via Gi… `10.1.5.5` | `show isis ipv6 route fc00:0:5::1/128` via Gi0/0/0/3 | Direct PE5 attachment |
| ECMP to 7.7.7.7 via P3/P4 | ECMP `fc00:0:7::1/128` via Gi0/0/0/1 and Gi0/0/0/2 | Same ECMP pattern |
| `traceroute mpls ipv4 7.7.7.7` | `traceroute fc00:0:7::1 source fc00:0:1::1` | Multi-hop through core |

| Original (MPLS-SR) | This lab |
|--------------------|----------|
| `show mpls ldp neighbor` | `show isis neighbors` |
| `show mpls forwarding` | `show isis ipv6 route`, `show route ipv6 summary` |
| `show segment-routing mpls connected-prefix-sid` | `show segment-routing srv6 locator` |
| `ping 7.7.7.7 source 5.5.5.5` | `ping fc00:0:7::1 source fc00:0:5::1` |

---

## Troubleshooting / gotchas

### Platform & CML

1. **XR cold boot** — ~15–20 min per IOS XRv 9000 node on first start. Factory YAML boots all nodes in parallel; use node staging only if your host is RAM-constrained.
2. **CAT 8000V license** — `license boot level network-advantage addon dna-advantage` + reload once; SRv6 commands appear after reload.
3. **XR setup wizard** — blocks SSH until dismissed; run `verification/complete_xr_setup.py`.
4. **PE SSH disabled** — day-0 XE config must generate an RSA key, set `ip ssh source-interface GigabitEthernet3`, and under `line vty`: `login local` + `transport input ssh`. If `show ip ssh` says **SSH Disabled**, port 22 on the PE mgmt IP stays closed (P routers may still work). See **Management access → OOB SSH day-0 config**.

### IS-IS

5. **Unique NET per node** — duplicate NET prevents adjacency formation.
6. **Level mismatch** — all nodes must be `level-2-only`.
7. **IOS XE multi-topology** — required for IPv6 IS-IS on PE routers.
8. **Point-to-point** — XR uses `point-to-point` under interface; XE uses `isis network point-to-point`.

### SRv6

9. **Locator overlap** — all SRv6 locators must use the `fcbb:bb00:…` block, not `fc00:0:N::/48` (loopback space only).
10. **Reachability vs locator** — ping/traceroute targets are `fc00:0:N::1` loopbacks, not fcbb locator prefixes.

### Wiring

11. **PE8 Gi1/Gi2** — Gi1→P3, Gi2→P4 per `topology.py`. Swapped descriptions/addresses = broken adjacencies.
12. **XR interface indices** — `Gi0/0/0/0`–`/4` in this lab (not `/1`–`/5` from Quisted's post).

### Canvas & automation

13. **Mgmt link clutter** — factory YAML sets `hide_links: true` on ext-conn and mgmt-sw (node property, links still work). If an older import shows mgmt spaghetti, right-click **mgmt-sw** → **Edit** → enable **Hide links** (same for ext-conn). Toggle **Show Link Labels** in CML if P-core labels are dense.
14. **Layout script** — `verification/apply_lab_layout.py` repositions nodes and replaces annotations. If you manually adjust the canvas, re-running the script will overwrite your changes unless you skip it or add a `--force` guard (see `README.md`).
15. **Traceroute timeouts** — automation uses ~120s timeout for traceroute over SSH.

9. **Reachability vs locator** — IGP loopback tests use `fc00:0:N::1`. uSID endpoint tests use PE uN SIDs (e.g. `fcbb:bb00:7::`). Do not ping fcbb locator `/48` prefixes as ordinary hosts.

---

## Optional: SR-TE uSID policy (path steering)

Baseline import stops at uSID endpoint ping (**T11**). For wire proof, see [Step 5 — Proof run](#proof-run). For explicit segment-list steering (force via P4, policy counters, SRH capture):

- Head-end: **P1 (IOS-XR)** — PE5 cat8000v image does not accept `segment-routing traffic-eng policy` config
- Snippet: `configs/snippets/p1-srte-usid-policy.cfg`
- Commit segment-list and policy in **separate** commits; add `srv6 locator MAIN` under the policy

---

## Appendix A — Automation (`run_live.py`, T1–T12)

```powershell
$env:CML_HOST = '<your-cml-host>'
$env:CML_LAB_ID = '<your-lab-id>'
$env:CML_PASSWORD = '...'          # or VERIFY_SSH_ONLY=1 to skip CML API
python verification/run_live.py
```

| Test | What it checks | Pass criteria |
|------|----------------|---------------|
| T1 | Topology YAML | 8 nodes, 14 links |
| T2 | Addressing model | Derived from `topology.py` DATA_LINKS |
| T3 | Lab on CML | Lab ID present / joined |
| T4 | IS-IS adjacencies | P1 ≥5 neighbors, PE5 ≥2 neighbors |
| T5 | SRv6 locators | P1 `fcbb:bb00:0001::/48` Up; PE5 `fcbb:bb00:0005::/48` Up |
| T6 | IGP loopback ping | `fc00:0:7::1` 100% (IS-IS route) |
| T11 | uSID endpoint | PE7 uN `fcbb:bb00:7::` ping 100% |
| T12 | uSID traceroute | Traceroute to `fcbb:bb00:7::` has core hops |
| T7 | Traceroute | Multi-hop paths on PE5 and P1 |
| T8 | Schema | results.json self-check |
| T9 | Mgmt SSH | 8/8 nodes reachable |
| T10 | IS-IS route | P1 has route to `fc00:0:7::1/128` |

Artifacts: `verification/quisted-srv6/*.txt`, summary in `verification/results.json`.

---

## Appendix B — CML canvas layout

Coordinates and zone annotations are defined in `layout.py`.

| Priority | Nodes booted first |
|----------|-------------------|
| 40 | ext-conn, mgmt-sw |
| 30 | P1 |
| 29–27 | P2, P3, P4 |
| 20 | PE5–PE8 (parallel) |

Apply layout to a running lab:

```bash
export CML_PASSWORD=...
python verification/apply_lab_layout.py
```

---

## Appendix C — Future: L3VPN over SRv6 (out of scope)

Quisted's original lab progressed to L3VPN / BGP over MPLS-SR. **This lab stops at SRv6 uSID underlay reachability.** A future Part 3 could add:

- VRF definitions on PE5/PE7
- BGP VPNv4/VPNv6 with SRv6 encapsulation
- CE connectivity and end-to-end customer prefix tests

That work is intentionally **not required** here (constitution audit point #13 = OUT_OF_SCOPE).

---

## Reference files

| File | Purpose |
|------|---------|
| `quisted-srv6-mgmt-initial.yaml` | Recommended CML import |
| `configs/*.cfg` | Deployed router configs |
| `topology.py` | Links, addressing, interface wiring |
| `layout.py` | Canvas coordinates and annotations |
| `verification/mgmt_hosts.json` | Current mgmt DHCP IPs |
| `verification/results.json` | Last validation run |

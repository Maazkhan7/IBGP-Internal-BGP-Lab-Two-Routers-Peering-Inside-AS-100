<div align="center">

# 🌐 IBGP (Internal BGP) Lab — Two Routers Peering Inside AS 100

### Border Gateway Protocol | Single Autonomous System | GNS3 + Cisco IOS

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Protocol](https://img.shields.io/badge/Protocol-BGP-orange?style=for-the-badge)
![Type](https://img.shields.io/badge/Peering-IBGP-blueviolet?style=for-the-badge)
![AD](https://img.shields.io/badge/Admin%20Distance-200-red?style=for-the-badge)
![LocPrf](https://img.shields.io/badge/Local%20Pref-100-yellow?style=for-the-badge)
![Transport](https://img.shields.io/badge/TCP-179-blue?style=for-the-badge)
![Lab](https://img.shields.io/badge/Lab-GNS3-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA%20%E2%86%92%20CCNP-purple?style=for-the-badge)

*A hands-on lab that builds an Internal BGP session between two routers in the same AS, advertises each LAN, and verifies the BGP table and the routing table — including how IBGP differs from EBGP.*

</div>

---

## 📑 Table of Contents

1. [Project Overview](#-project-overview)
2. [BGP Introduction](#-bgp-introduction)
3. [Types of BGP](#-types-of-bgp)
4. [BGP Administrative Distance](#-bgp-administrative-distance-ad)
5. [How BGP / IBGP Behaves](#-how-bgp--ibgp-behaves)
6. [Lab Topology](#-lab-topology)
7. [Addressing Table](#-addressing-table)
8. [Configuration](#-configuration)
9. [Verification (Stage by Stage)](#-verification-stage-by-stage)
10. [EBGP Lab vs IBGP Lab](#-ebgp-lab-vs-ibgp-lab)
11. [Command Cheat Sheet](#-command-cheat-sheet)
12. [Key Takeaways](#-key-takeaways)
13. [Troubleshooting Checklist](#-troubleshooting-checklist)
14. [Skills Demonstrated](#-skills-demonstrated)
15. [Author](#-author)

---

## 🎯 Project Overview

| Item | Details |
|------|---------|
| **Lab Name** | IBGP — Internal BGP inside one Autonomous System |
| **Platform** | GNS3 (Cisco IOS routers, switches, VPCs), accessed over Telnet |
| **Autonomous System** | **AS 100** (both Router1 and Router2) |
| **Peering Link** | FastEthernet0/0 — `1.1.1.0/24` |
| **Advertised Networks** | `192.168.1.0/24` (R1 LAN) and `192.168.2.0/24` (R2 LAN) |
| **Goal** | Establish an IBGP neighbor relationship (same ASN on both sides) and exchange LAN prefixes |
| **Verified With** | `%BGP-5-ADJCHANGE` log, `show ip bgp`, `show ip route` |

### ✅ What this lab proves

- When `remote-as` **equals** the local AS number, the session is **IBGP**.
- IBGP routes are installed in the routing table with **Administrative Distance 200**.
- IBGP-learned routes carry **Local Preference 100** and an **empty AS-Path** (the AS number is not prepended inside the AS).

---

## 📘 BGP Introduction

**BGP (Border Gateway Protocol)** is the routing protocol that runs the Internet. It is the only **EGP (Exterior Gateway Protocol)** in use today and is defined in RFC 4271 (BGP-4).

| Property | Description |
|----------|-------------|
| **Category** | Exterior Gateway Protocol (routing *between* autonomous systems), but it can also run *inside* an AS as IBGP |
| **Algorithm** | **Path Vector** (tracks the list of ASNs a route passed through) |
| **Transport** | **TCP port 179** — reliable, unicast, session-based |
| **Neighbors** | **Manually configured**, no automatic discovery |
| **Updates** | Incremental and triggered; no periodic full-table refresh |
| **Metric** | No single metric — uses **path attributes** (Weight, Local Pref, AS-Path, Origin, MED, etc.) |
| **Loop Prevention** | EBGP: reject a route if own ASN is in the AS-Path. IBGP: split-horizon rule |
| **Scalability** | Built for very large networks (the global Internet table) |
| **Focus** | **Policy and control**, not fastest convergence |

### 🆚 IGP vs EGP

| | IGP (OSPF, EIGRP, RIP) | EGP (BGP) |
|---|---|---|
| Scope | Inside **one** autonomous system | **Between** autonomous systems |
| Goal | Fast, optimal paths | Policy-based, scalable path selection |
| Neighbor discovery | Automatic (hello) | Manual `neighbor` statement |
| Transport | Own protocol / IP | TCP 179 |
| Metric | Cost / composite | Path attributes |

### 🔢 What is an Autonomous System (AS)?

A collection of networks under **one administrative control** that presents a **common routing policy** to the outside world. Each AS is identified by an **ASN**:

| Range | Type |
|-------|------|
| 1 – 64495 | Public 16-bit ASNs |
| 64512 – 65534 | **Private** 16-bit ASNs |
| 4-byte ASNs | Extended range (RFC 6793) |

> In this lab, **both routers belong to AS 100**.

---

## 🔀 Types of BGP

| | **EBGP (External BGP)** | **IBGP (Internal BGP)** |
|---|---|---|
| **Neighbors are in** | **Different** AS numbers | **Same** AS number ⭐ |
| **Config rule** | `remote-as` ≠ local AS | `remote-as` = local AS ⭐ |
| **Administrative Distance** | **20** | **200** ⭐ |
| **Default TTL** | **1** (usually directly connected) | 255 (can be multiple hops away) |
| **AS-Path** | Local ASN is **prepended** when advertising | AS-Path **not** changed ⭐ |
| **Local Preference** | Not carried across ASes | **Carried** inside the AS (default 100) ⭐ |
| **Next-Hop** | Changed to the advertising router's own IP | **Not** changed by default |
| **Loop prevention** | AS-Path check | **Split-horizon rule**: a route learned from an IBGP peer is not passed to another IBGP peer |
| **Typical use** | ISP ⇄ customer, ISP ⇄ ISP | Distributing BGP information inside one AS |

> ⭐ **This lab is IBGP** — R1 and R2 are both in AS 100, so every ⭐ item above is visible in the outputs.

---

## 📏 BGP Administrative Distance (AD)

Administrative Distance decides **which routing source is trusted** when multiple sources know the same prefix. Lower = more trusted.

| Route Source | AD |
|--------------|----|
| Connected | 0 |
| Static | 1 |
| **EBGP** | **20** |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 170 |
| **IBGP** | **200** ⭐ |
| Unknown / unreachable | 255 |

### 🔎 Where AD 200 is visible in this lab

```
B     192.168.2.0/24 [200/0] via 1.1.1.2, 00:01:58
                      │   │
                      │   └── Metric (MED) = 0
                      └────── Administrative Distance = 200  → IBGP
```

> 💡 **Why is IBGP AD so high?** IBGP is meant to **carry external routes across an AS**, while the AS's own IGP (OSPF/EIGRP) should handle internal routes. A value of **200** ensures an IGP route to the same prefix always **wins** over an IBGP route, which avoids routing loops inside the AS.
>
> The second number `0` is **not** a "BGP metric" used to choose paths. It is the **MED/IGP-cost value** carried with the route. BGP selects paths using **attributes**.

---

## ⚙️ How BGP / IBGP Behaves

### 1️⃣ Neighbor (Session) Formation

BGP builds a **TCP session on port 179**, then exchanges messages:

| Message | Purpose |
|---------|---------|
| **OPEN** | Starts the session; carries ASN, hold time, router ID, capabilities |
| **KEEPALIVE** | Confirms the neighbor is alive (default every 60 s, hold time 180 s) |
| **UPDATE** | Advertises new prefixes and path attributes, or withdraws routes |
| **NOTIFICATION** | Reports an error and **closes** the session |

### 2️⃣ BGP Neighbor States (FSM)

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

| State | Meaning |
|-------|---------|
| Idle | Waiting to start; no resources allocated |
| Connect | Waiting for the TCP 3-way handshake |
| Active | TCP attempt failed or retrying (often a reachability or config issue) |
| OpenSent | OPEN sent, waiting for the peer's OPEN |
| OpenConfirm | OPEN received, waiting for KEEPALIVE |
| **Established** | ✅ Session up — UPDATEs are exchanged |

The log `%BGP-5-ADJCHANGE: neighbor ... Up` on both routers confirms **Established**.

### 3️⃣ Router ID

BGP needs a **Router ID**. If it is not set manually, IOS chooses the **highest loopback IP**, otherwise the **highest IP among active interfaces** when the BGP process starts.
In this lab: **R1 = 192.168.1.1** and **R2 = 192.168.2.1** (both visible in `show ip bgp`).

### 4️⃣ The `network` Command in BGP

| In IGPs (OSPF/EIGRP) | In BGP |
|---|---|
| Enables the protocol on matching interfaces | **Does not enable anything on an interface** |
| Wildcard / classful matching | Advertises a prefix **only if an exact match exists in the routing table** |

`network 192.168.1.0 mask 255.255.255.0` works on R1 because `192.168.1.0/24` is a **connected route** there.

### 5️⃣ IBGP-Specific Behavior

| Behavior | Explanation |
|----------|-------------|
| **AS-Path stays empty** | Inside the same AS the local ASN is **not prepended**, so locally learned IBGP routes show a blank Path |
| **Local Preference = 100** | LocPrf is shared inside the AS; the default value is **100** |
| **Next-hop is not changed** | The IBGP peer keeps the **original** next hop (here it is simply the neighbor, as the routers are directly connected) |
| **Split-horizon rule** | A route learned from one IBGP peer is **never advertised to another IBGP peer** |
| **Full mesh requirement** | Because of split horizon, IBGP needs a full mesh of **n(n-1)/2** sessions, or **route reflectors / confederations** to scale |
| **TTL 255** | IBGP neighbors can be several hops away (usually peered via loopbacks with `update-source`) |
| **IGP underlay** | In real networks IBGP depends on an IGP (OSPF/EIGRP) to reach neighbor loopbacks and next hops |

> 📌 With only **two routers**, the split-horizon rule has no visible effect — which makes this the perfect first IBGP lab.

### 6️⃣ Path Selection — Important Attributes

| Order | Attribute | Prefer |
|-------|-----------|--------|
| 1 | **Weight** (Cisco-only, local) | Highest |
| 2 | **Local Preference** | Highest |
| 3 | **Locally originated** | Network/aggregate on this router |
| 4 | **AS-Path** length | Shortest |
| 5 | **Origin** | IGP (`i`) < EGP (`e`) < Incomplete (`?`) |
| 6 | **MED** | Lowest |
| 7 | **EBGP over IBGP** | EBGP |
| 8 | **IGP metric to next hop** | Lowest |
| 9 | **Router ID** | Lowest |

### 7️⃣ Reading the BGP Table Columns

| Column | Meaning |
|--------|---------|
| `*` | Valid route |
| `>` | **Best** path (installed in the routing table) |
| `i` (after `>`) | Route was learned from an **IBGP** neighbor ⭐ |
| Network | Destination prefix |
| Next Hop | IP used to reach the prefix (`0.0.0.0` = locally originated) |
| Metric | MED |
| LocPrf | Local Preference (**100** default for IBGP) |
| Weight | **32768** = locally originated, **0** = learned from a neighbor |
| Path | AS-Path (**blank** here — same AS) |
| Origin code (last `i`) | `i` = IGP (injected by `network`), `e` = EGP, `?` = incomplete |

---

## 🗺️ Lab Topology

```
 ╔═══════════════════════════════════ AS 100 (IBGP) ═══════════════════════════════════╗
 ║                                                                                     ║
 ║   ┌──────────┐   Fa0/0 ──────────── IBGP session (TCP 179) ──────────── Fa0/0   ┌──────────┐
 ║   │ Router1  │  1.1.1.1/24                                          1.1.1.2/24  │ Router2  │
 ║   │  (R1)    │                                                                  │  (R2)    │
 ║   └────┬─────┘                                                                  └────┬─────┘
 ║      Fa1/0                                                                        Fa1/0
 ║  192.168.1.1/24                                                              192.168.2.1/24
 ║        │                                                                           │
 ║   ┌────┴─────┐                                                                ┌────┴─────┐
 ║   │ Switch3  │                                                                │ Switch4  │
 ║   └┬───┬───┬─┘                                                                └┬───┬───┬─┘
 ║  VPC5 VPC6 VPC7                                                             VPC8 VPC9 VPC10
 ╚═════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 📋 Addressing Table

| Device | Interface | IP Address | Subnet Mask | Connected To |
|--------|-----------|------------|-------------|--------------|
| **R1** (AS 100) | Fa0/0 | 1.1.1.1 | 255.255.255.0 | R2 Fa0/0 |
| **R1** (AS 100) | Fa1/0 | 192.168.1.1 | 255.255.255.0 | Switch3 |
| **R2** (AS 100) | Fa0/0 | 1.1.1.2 | 255.255.255.0 | R1 Fa0/0 |
| **R2** (AS 100) | Fa1/0 | 192.168.2.1 | 255.255.255.0 | Switch4 |
| VPC5 – VPC7 | eth0 | 192.168.1.x | 255.255.255.0 | Switch3 |
| VPC8 – VPC10 | eth0 | 192.168.2.x | 255.255.255.0 | Switch4 |

### BGP Parameters

| Parameter | R1 | R2 |
|-----------|----|----|
| Local AS | 100 | 100 |
| Neighbor | 1.1.1.2 | 1.1.1.1 |
| Remote AS | **100** | **100** |
| Router ID | 192.168.1.1 | 192.168.2.1 |
| Advertised Network | 192.168.1.0/24 | 192.168.2.0/24 |
| Peering Type | **IBGP** | **IBGP** |

---

## 🛠️ Configuration

### 🔹 Router1 (AS 100)

```bash
R1(config)# router bgp 100
R1(config-router)# network 192.168.1.0 mask 255.255.255.0
R1(config-router)# neighbor 1.1.1.2 remote-as 100
R1(config-router)# do write
Building configuration...
[OK]
```

### 🔹 Router2 (AS 100)

```bash
R2(config)# router bgp 100
R2(config-router)# network 192.168.2.0 mask 255.255.255.0
R2(config-router)# neighbor 1.1.1.1 remote-as 100
```

### 📝 Command Explanation

| Command | What it does |
|---------|--------------|
| `router bgp 100` | Starts BGP with local AS 100 (same on both routers) |
| `network <prefix> mask <mask>` | Originates the prefix into BGP — **must match a route in the routing table exactly** |
| `neighbor <ip> remote-as 100` | Defines the peer. Because `remote-as` = local AS, the session is **IBGP** |

> 🔑 **The only difference from the EBGP lab:** the `remote-as` value. EBGP used `remote-as 200` / `remote-as 100`; IBGP uses **100 on both sides**.

---

## 🔍 Verification (Stage by Stage)

### 🟢 Stage 1 — IBGP Neighbors Came Up

```
R1(config-router)# neighbor 1.1.1.2 remote-as 100
*Nov  1 14:43:24.535: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 Up

R2(config-router)# neighbor 1.1.1.1 remote-as 100
*Nov  1 14:43:25.027: %BGP-5-ADJCHANGE: neighbor 1.1.1.1 Up
```

| Annotation | Meaning |
|------------|---------|
| `%BGP-5-ADJCHANGE` | BGP adjacency state-change message |
| `neighbor ... Up` | TCP/179 session reached **Established** |
| Time difference ≈ 0.5 s | The session came up as soon as **both** sides had a matching neighbor statement |

✅ **Result:** IBGP peering inside AS 100 is established on both routers.

---

### 🟢 Stage 2 — BGP Table on R1 (`show ip bgp`)

```
R1# show ip bgp
BGP table version is 3, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  192.168.1.0      0.0.0.0                  0         32768 i
 *>i 192.168.2.0      1.1.1.2                  0    100      0 i
```

#### 📌 Annotated Reading

| Marker | Where | Explanation |
|--------|-------|-------------|
| `*>` | `192.168.1.0` | Valid + best. Next hop `0.0.0.0` and Weight **32768** → **locally originated** on R1 |
| `*>i` | `192.168.2.0` | Valid + best + **`i` = learned from an IBGP neighbor** |
| `1.1.1.2` | Next Hop | R2's Fa0/0 address |
| `100` | LocPrf | **Default Local Preference**, shared inside the AS |
| `0` | Weight | Learned from a neighbor (not local) |
| Blank | **Path** | **No AS number** — IBGP does not prepend the local ASN |
| `i` (last column) | Origin | Prefix was injected with the `network` command (IGP origin) |
| `192.168.1.1` | Router ID | Highest active interface IP on R1 |

---

### 🟢 Stage 3 — BGP Table on R2 (`show ip bgp`)

```
R2# show ip bgp
BGP table version is 3, local router ID is 192.168.2.1
...
     Network          Next Hop            Metric LocPrf Weight Path
 *>i 192.168.1.0      1.1.1.1                  0    100      0 i
 *>  192.168.2.0      0.0.0.0                  0         32768 i
```

| Marker | Explanation |
|--------|-------------|
| `*>i 192.168.1.0` | R2 learned R1's LAN through **IBGP**, next hop `1.1.1.1`, LocPrf 100 |
| `*> 192.168.2.0` | R2's own LAN — local, Weight 32768 |
| `192.168.2.1` | R2's Router ID |

✅ **Result:** Both routers have a mirrored BGP table: one local prefix and one IBGP-learned prefix.

---

### 🟢 Stage 4 — Routing Table on R1 (`show ip route`)

```
R1# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       ...
Gateway of last resort is not set

      1.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        1.1.1.0/24 is directly connected, FastEthernet0/0
L        1.1.1.1/32 is directly connected, FastEthernet0/0
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, FastEthernet1/0
L        192.168.1.1/32 is directly connected, FastEthernet1/0
B     192.168.2.0/24 [200/0] via 1.1.1.2, 00:01:58
```

| Marker | Explanation |
|--------|-------------|
| `B` | Route learned by **BGP** |
| **`[200/0]`** | **200 = Administrative Distance of IBGP** ⭐ ; `0` = MED value |
| `via 1.1.1.2` | Next hop = R2 |
| `00:01:58` | Age of the route |

### 🟢 Stage 5 — Routing Table on R2 (`show ip route`)

```
R2# show ip route
...
      1.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        1.1.1.0/24 is directly connected, FastEthernet0/0
L        1.1.1.2/32 is directly connected, FastEthernet0/0
B     192.168.1.0/24 [200/0] via 1.1.1.1, 00:03:07
      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, FastEthernet1/0
L        192.168.2.1/32 is directly connected, FastEthernet1/0
```

| Marker | Explanation |
|--------|-------------|
| `B 192.168.1.0/24 [200/0]` | R2 learned R1's LAN via IBGP with **AD 200** |
| `via 1.1.1.1` | Next hop = R1 |
| `00:03:07` | Route age |

✅ **Result:** Each router installed the other's LAN as an **IBGP route with AD 200**.

---

### 🔄 Stage Summary

| Stage | Check | Command | Result |
|-------|-------|---------|--------|
| 1 | Neighbor adjacency | Log `%BGP-5-ADJCHANGE` | ✅ Both neighbors Up |
| 2 | BGP table (R1) | `show ip bgp` | ✅ `*>i`, LocPrf 100, blank Path |
| 3 | BGP table (R2) | `show ip bgp` | ✅ Mirrored result |
| 4 | Routing table (R1) | `show ip route` | ✅ `B 192.168.2.0/24 [200/0]` |
| 5 | Routing table (R2) | `show ip route` | ✅ `B 192.168.1.0/24 [200/0]` |

---

## ⚖️ EBGP Lab vs IBGP Lab

Same topology, same LANs — only the `remote-as` changed. Compare what changed in the outputs:

| Item | **EBGP Lab** (AS 100 ⇄ AS 200) | **IBGP Lab** (AS 100 ⇄ AS 100) |
|------|-------------------------------|-------------------------------|
| `remote-as` | Different from local AS | **Same** as local AS |
| Route in routing table | `B 192.168.2.0/24 [`**`20`**`/0]` | `B 192.168.2.0/24 [`**`200`**`/0]` |
| BGP table status | `*>` | `*>i` |
| AS-Path | **`100`** (ASN prepended) | **Blank** (not prepended) |
| Local Preference | Blank (not carried) | **100** |
| Weight (learned) | 0 | 0 |
| Weight (local) | 32768 | 32768 |

> 🧠 **Memory trick:** EBGP = **E**xternal = **2**0. IBGP = **I**nternal = **2**00 (the "bigger" distance, because an IGP should win inside the AS).

---

## 🧾 Command Cheat Sheet

| Purpose | Command |
|---------|---------|
| Start BGP | `router bgp <asn>` |
| Advertise a network | `network <prefix> mask <mask>` |
| Create IBGP neighbor | `neighbor <ip> remote-as <same-asn>` |
| View BGP table | `show ip bgp` |
| View neighbor state | `show ip bgp summary` |
| View neighbor details | `show ip bgp neighbors` |
| View BGP routes in RIB | `show ip route bgp` |
| Check one prefix | `show ip bgp 192.168.2.0` |
| Reset a session | `clear ip bgp *` |
| Save config | `write memory` |

### 🔧 Real-world IBGP commands (for extension labs)

| Purpose | Command |
|---------|---------|
| Peer using loopbacks | `neighbor <lo-ip> update-source Loopback0` |
| Fix next hop for external routes | `neighbor <ip> next-hop-self` |
| Scale IBGP | `neighbor <ip> route-reflector-client` |

---

## 🧠 Key Takeaways

1. 🏠 **IBGP = same ASN on both sides.** Only the `remote-as` value turns an EBGP session into IBGP.
2. 📏 **IBGP AD = 200** (seen as `[200/0]`), compared with **EBGP AD = 20**.
3. 🚫 **No AS number is prepended** inside the AS, so the AS-Path is **blank** for IBGP-learned routes.
4. 🎚️ **Local Preference 100** is carried between IBGP peers (default value).
5. 🔤 In `show ip bgp`, `*>i` means **valid, best, learned from IBGP**.
6. ⚖️ **Weight 32768** = locally originated; **0** = learned from a neighbor.
7. 🔄 **Split horizon:** IBGP never re-advertises an IBGP-learned route to another IBGP peer — hence full mesh, route reflectors or confederations in larger networks.
8. 🎯 The BGP `network` command needs an **exact match** in the routing table.
9. 🧱 A strong IGP underlay (OSPF/EIGRP) is normally used with IBGP so neighbors and next hops stay reachable.
10. 🔔 `%BGP-5-ADJCHANGE ... Up` on both routers is the fastest confirmation of an Established session.

---

## 🩺 Troubleshooting Checklist

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Neighbor stuck in **Idle / Active** | No IP reachability to the neighbor | Check interface status and `ping` the neighbor |
| Session never forms | Wrong neighbor IP or `remote-as` not equal to local AS | For IBGP, use the **same** ASN |
| Prefix not in BGP table | `network` mask doesn't match the RIB exactly | Verify with `show ip route` |
| IBGP route in BGP table but not in RIB | Next hop unreachable, or an IGP route with lower AD exists | Check reachability to next hop; compare ADs |
| Route learned from IBGP not passed on | Split-horizon rule | Use full mesh or route reflectors |
| Loopback peering fails | Missing `update-source` or no IGP route to the loopback | Add `update-source Loopback0` and advertise loopbacks in the IGP |

---

## 💡 Ideas to Extend This Lab

- 🔁 Peer using **loopbacks** with `update-source`
- 🧭 Add an **IGP (OSPF/EIGRP)** as the underlay
- ➕ Add a **third router** and observe the split-horizon rule and the need for a full mesh
- 🪞 Configure a **route reflector**
- 🔗 Combine **EBGP + IBGP** in one topology and use `next-hop-self`
- 🎛️ Change **Local Preference** to influence exit-path selection
- 🧪 Test end-to-end connectivity from VPC5 to VPC8 with `ping` and `trace`

---

## 🧰 Skills Demonstrated

`BGP` · `IBGP` · `EBGP vs IBGP` · `Autonomous Systems` · `Administrative Distance` · `Local Preference` · `AS-Path` · `Cisco IOS` · `GNS3` · `Routing Table Analysis` · `Network Troubleshooting` · `Network Documentation`

---

## 👤 Author

<div align="center">

**Maaz Khan**
CCNA-Certified Network & NOC Engineer
📍 Lower Dir, KPK, Pakistan

![CCNA](https://img.shields.io/badge/CCNA-Certified-blue?style=flat-square&logo=cisco)
![NOC](https://img.shields.io/badge/NOC-Engineer-orange?style=flat-square)

⭐ If this lab helped you, give the repo a star!

</div>

---

<div align="center">

**#BGP #IBGP #EBGP #CCNA #CCNP #Cisco #GNS3 #Networking #NetworkEngineer #NOC**

</div>

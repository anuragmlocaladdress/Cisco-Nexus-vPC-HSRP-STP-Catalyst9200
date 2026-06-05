# 🔀 Cisco Nexus vPC — Dual-Peer HA Lab

### vPC | HSRP Active/Standby | STP Root Election | MC-LAG Downlink

### with Cisco Catalyst 9200 Access Switch

![NX-OS](https://img.shields.io/badge/Cisco_NX--OS-9.3%2B-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Catalyst](https://img.shields.io/badge/Cisco_Catalyst-9200-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![EVE-NG](https://img.shields.io/badge/Lab-EVE--NG-333333?style=for-the-badge)
![vPC](https://img.shields.io/badge/HA-vPC_Active%2FActive-FF8300?style=for-the-badge&logo=cisco&logoColor=white)
![HSRP](https://img.shields.io/badge/Gateway-HSRP_v2-0F6E56?style=for-the-badge)

**Author:** Anurag Mishra — Solution Architect - IT & Security  
📍 Hyderabad, India | [LinkedIn](https://www.linkedin.com/in/anuragmishra6) | [GitHub](https://github.com/anuragmlocaladdress)

---

## 🗺️ Lab Topology

```
          ┌─────────────────────────────────────────────┐
          │          vPC Peer-Link — Po10                │
          │   (Eth1/49 + Eth1/50 — LACP Active)         │
          │                                              │
   ┌──────┴──────┐                           ┌──────────┴────┐
   │   N9K-1     │  Keepalive mgmt0 /24      │    N9K-2      │
   │  PRIMARY    │◄─────────────────────────►│  SECONDARY    │
   │ HSRP Active │  192.0.2.1 ↔ 192.0.2.2   │ HSRP Standby  │
   │  STP Root   │                           │               │
   └──────┬──────┘                           └───────┬───────┘
          │ Eth1/1 → Po30 (vpc 30)                   │ Eth1/1 → Po30 (vpc 30)
          └──────────────────┬───────────────────────┘
                             │
                      ┌──────┴──────┐
                      │   C9200     │
                      │ Access SW   │
                      │  Po1 (LAG)  │
                      └─────────────┘
```

---

## 📋 Table of Contents

1. [Lab Overview](#1--lab-overview)
2. [Device & IP Addressing](#2--device--ip-addressing)
3. [VLAN Design](#3--vlan-design)
4. [Key Concepts](#4--key-concepts)
5. [Phase 1 — N9K-1 vPC Primary](#5--phase-1--n9k-1-vpc-primary-configuration)
6. [Phase 2 — N9K-2 vPC Secondary](#6--phase-2--n9k-2-vpc-secondary-configuration)
7. [Phase 3 — Cisco C9200 Access Switch](#7--phase-3--cisco-c9200-access-switch-configuration)
8. [Phase 4 — Verification](#8--phase-4--verification-commands)
9. [How It All Works Together](#9--how-it-all-works-together)
10. [Lessons Learned & Common Issues](#10--lessons-learned--common-issues)
11. [References](#-references)

---

## 1. 🧠 Lab Overview

This lab demonstrates a **production-grade Cisco Nexus vPC pair** with HSRP gateway redundancy and STP root alignment, connected to a Cisco Catalyst 9200 access switch via MC-LAG — built and validated in EVE-NG.

| Feature             | Technology Used                                      |
|---------------------|------------------------------------------------------|
| Distribution HA     | Cisco vPC — Active/Active forwarding                 |
| Access Uplink       | MC-LAG via vPC member port (Po30)                    |
| Gateway Redundancy  | HSRP v2 — Active on N9K-1, Standby on N9K-2         |
| Loop Prevention     | RSTP — N9K-1 as Root (priority 4096)                 |
| Peer Detection      | vPC Keepalive via Management VRF                     |
| STP Consistency     | `peer-switch` — single bridge ID presented to C9200  |
| Peer Sync           | vPC Peer-Link — LAG 256 (Eth1/49–50)                 |

---

## 2. 📡 Device & IP Addressing

| Device  | Role                        | Interface | IP Address    |
|---------|-----------------------------|-----------|---------------|
| N9K-1   | vPC Primary / HSRP Active   | mgmt0     | 192.0.2.1/24  |
| N9K-1   | SVI VLAN 10                 | Vlan10    | 10.10.10.2/24 |
| N9K-1   | SVI VLAN 20                 | Vlan20    | 10.20.20.2/24 |
| N9K-2   | vPC Secondary / HSRP Standby| mgmt0     | 192.0.2.2/24  |
| N9K-2   | SVI VLAN 10                 | Vlan10    | 10.10.10.3/24 |
| N9K-2   | SVI VLAN 20                 | Vlan20    | 10.20.20.3/24 |
| C9200   | Access Switch               | VLAN 10   | DHCP client   |

---

## 3. 🔌 VLAN Design

| VLAN | Name         | HSRP VIP     | N9K-1 IP     | N9K-2 IP     | Purpose        |
|------|--------------|--------------|--------------|--------------|----------------|
| 10   | USERS_VLAN10 | 10.10.10.1   | 10.10.10.2   | 10.10.10.3   | User segment   |
| 20   | USERS_VLAN20 | 10.20.20.1   | 10.20.20.2   | 10.20.20.3   | User segment   |

---

## 4. 💡 Key Concepts

**vPC — Virtual Port Channel**

vPC allows a downstream device to dual-home into two separate Nexus switches using a single port-channel. Both uplinks are active simultaneously — no STP blocking on the access side. The two Nexus peers sync state over the **Peer-Link (Po10)** and use a dedicated **keepalive** over mgmt VRF to detect peer failures.

**peer-switch**

With `peer-switch` enabled, both N9K-1 and N9K-2 present the **same STP bridge ID** to downstream devices. From C9200's perspective, it sees one switch — no topology change when a peer fails or reloads.

**HSRP v2**

Both Nexus SVIs have individual IPs but share a **Virtual IP (VIP)** as the default gateway for end clients. N9K-1 wins active election (priority 110 > 90). `preempt` ensures N9K-1 reclaims active if it recovers from a failure.

**STP Root Alignment**

N9K-1 is manually set as STP Root (priority 4096) and HSRP Active — both are intentionally aligned. N9K-2 has STP priority 8192 (secondary). This ensures consistent traffic paths.

**vPC Role Priority**

Lower `role priority` wins Primary. N9K-1 is set to 100, N9K-2 to 200. Primary is the tie-breaker for split-brain decisions and controls vPC operational state.

---

## 5. ⚙️ Phase 1 — N9K-1 vPC Primary Configuration

> N9K-1 is the vPC Primary, HSRP Active, and STP Root for all VLANs.

### Step 1.1 — Enable Features

```text
hostname N9K-1

feature vpc
feature lacp
feature interface-vlan
feature hsrp
feature spanning-tree
```

### Step 1.2 — STP Root Election

```text
! Low priority = Root Bridge for VLANs 10 and 20
spanning-tree vlan 10,20 priority 4096
```

### Step 1.3 — vPC Keepalive Interface

```text
interface mgmt0
  ip address 192.0.2.1/24
  no shutdown
```

### Step 1.4 — VLANs

```text
vlan 10
  name USERS_VLAN10
vlan 20
  name USERS_VLAN20
```

### Step 1.5 — SVIs with HSRP

```text
interface Vlan10
  ip address 10.10.10.2/24
  hsrp version 2
  hsrp 10
    priority 110
    preempt
    ip 10.10.10.1
  no shutdown

interface Vlan20
  ip address 10.20.20.2/24
  hsrp version 2
  hsrp 20
    priority 110
    preempt
    ip 10.20.20.1
  no shutdown
```

### Step 1.6 — vPC Domain

```text
vpc domain 1
  role priority 100
  peer-switch
  peer-keepalive destination 192.0.2.2 source 192.0.2.1 vrf management
  system-priority 4000
```

### Step 1.7 — vPC Peer-Link (Po10)

```text
interface Ethernet1/49
  switchport
  channel-group 10 mode active

interface Ethernet1/50
  switchport
  channel-group 10 mode active

interface Port-channel10
  description vPC-PEER-LINK
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20
  spanning-tree port type network
  vpc peer-link
  no shutdown
```

### Step 1.8 — vPC Downlink to C9200 (Po30)

```text
interface Ethernet1/1
  switchport
  channel-group 30 mode active

interface Port-channel30
  description vPC-TO-C9200
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20
  vpc 30
  spanning-tree port type edge trunk
  no shutdown
```

> ✅ N9K-1 fully configured — vPC Primary, HSRP Active, STP Root

---

## 6. ⚙️ Phase 2 — N9K-2 vPC Secondary Configuration

> N9K-2 mirrors N9K-1 with higher role/STP/HSRP priorities — it is the standby peer.

### Step 2.1 — Enable Features

```text
hostname N9K-2

feature vpc
feature lacp
feature interface-vlan
feature hsrp
feature spanning-tree
```

### Step 2.2 — STP Secondary (Not Root)

```text
! Higher priority = N9K-2 will NOT win root election
spanning-tree vlan 10,20 priority 8192
```

### Step 2.3 — vPC Keepalive Interface

```text
interface mgmt0
  ip address 192.0.2.2/24
  no shutdown
```

### Step 2.4 — VLANs

```text
vlan 10
vlan 20
```

### Step 2.5 — SVIs with HSRP

```text
interface Vlan10
  ip address 10.10.10.3/24
  hsrp version 2
  hsrp 10
    priority 90
    preempt
    ip 10.10.10.1
  no shutdown

interface Vlan20
  ip address 10.20.20.3/24
  hsrp version 2
  hsrp 20
    priority 90
    preempt
    ip 10.20.20.1
  no shutdown
```

### Step 2.6 — vPC Domain

```text
vpc domain 1
  role priority 200
  peer-switch
  peer-keepalive destination 192.0.2.1 source 192.0.2.2 vrf management
  system-priority 4000
```

### Step 2.7 — vPC Peer-Link (Po10)

```text
interface Ethernet1/49
  switchport
  channel-group 10 mode active

interface Ethernet1/50
  switchport
  channel-group 10 mode active

interface Port-channel10
  description vPC-PEER-LINK
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20
  spanning-tree port type network
  vpc peer-link
  no shutdown
```

### Step 2.8 — vPC Downlink to C9200 (Po30)

```text
interface Ethernet1/1
  switchport
  channel-group 30 mode active

interface Port-channel30
  description vPC-TO-C9200
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20
  vpc 30
  spanning-tree port type edge trunk
  no shutdown
```

> ✅ N9K-2 fully configured — vPC Secondary, HSRP Standby

---

## 7. ⚙️ Phase 3 — Cisco C9200 Access Switch Configuration

> C9200 dual-homes into both Nexus peers via a single LACP port-channel (Po1). From C9200's view, it connects to one switch.

### Step 3.1 — Uplink Port-Channel to Nexus vPC Pair

```text
interface GigabitEthernet1/0/1
  channel-group 1 mode active

interface GigabitEthernet1/0/2
  channel-group 1 mode active

interface Port-channel1
  description UPLINK-TO-NEXUS-vPC
  switchport mode trunk
  switchport trunk allowed vlan 10,20
  no shutdown
```

### Step 3.2 — Access Ports

```text
interface GigabitEthernet1/0/10
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
  no shutdown

interface GigabitEthernet1/0/11
  switchport mode access
  switchport access vlan 20
  spanning-tree portfast
  no shutdown
```

### Step 3.3 — Management SVI

```text
interface Vlan10
  ip address 10.10.10.100 255.255.255.0
  no shutdown

ip default-gateway 10.10.10.1
```

> ✅ C9200 dual-homed to both Nexus peers — active uplinks on both paths, no STP blocking

---

## 8. 🔍 Phase 4 — Verification Commands

### vPC Health

```text
show vpc brief
show vpc role
show vpc peer-keepalive
show lacp interface port-channel10
```

| Check              | Expected Output          |
|--------------------|--------------------------|
| N9K-1 vPC Role     | `primary`                |
| N9K-2 vPC Role     | `secondary`              |
| Peer-keepalive     | `peer is alive`          |
| Peer-link state    | `peer link is up`        |
| vPC member 30      | `Up` on both peers       |

---

### HSRP

```text
show hsrp brief
show hsrp group 10
show hsrp group 20
```

| Check              | Expected Output          |
|--------------------|--------------------------|
| N9K-1 VLAN 10      | `Active`                 |
| N9K-2 VLAN 10      | `Standby`                |
| N9K-1 VLAN 20      | `Active`                 |
| Virtual IP VLAN 10 | `10.10.10.1`             |
| Virtual IP VLAN 20 | `10.20.20.1`             |

---

### STP

```text
show spanning-tree vlan 10
show spanning-tree vlan 20
show spanning-tree summary
```

| Check              | Expected Output          |
|--------------------|--------------------------|
| Root Bridge        | N9K-1                    |
| N9K-1 Priority     | `4096`                   |
| N9K-2 Priority     | `8192`                   |
| C9200 uplink       | `FWD` (no blocked port)  |

---

### Port-Channel / MC-LAG

```text
! On Nexus
show port-channel summary
show interface port-channel30

! On C9200
show etherchannel summary
show lacp neighbor
```

---

### End-to-End Ping

```text
! From N9K-1 — ping across vPC domain
ping 10.10.10.1 vrf default
ping 10.20.20.1 vrf default

! From client in VLAN 10
ping 10.10.10.1     ! VLAN 10 gateway
ping 10.20.20.1     ! VLAN 20 gateway (cross-VLAN via SVI)
```

---

## 9. 🔄 How It All Works Together

```
Client (VLAN 10)
      |
  C9200 Access Switch
    Po1 → active LACP to both Nexus peers (no STP blocking)
      |
  ┌───┴───────────────────────────────┐
  │                                   │
N9K-1 (Po30 vpc 30)           N9K-2 (Po30 vpc 30)
 HSRP Active → 10.10.10.1      HSRP Standby
 STP Root (4096)                STP Secondary (8192)
  │                                   │
  └───────── vPC Peer-Link ───────────┘
              (Po10 — sync, BUM traffic)

Client sends frame to gateway MAC → HSRP VIP 10.10.10.1
Both Nexus peers can forward return traffic simultaneously
If N9K-1 fails → N9K-2 takes HSRP Active within dead interval
vPC keepalive detects failure → secondary does not isolate
```

---

## 10. ⚠️ Lessons Learned & Common Issues

| # | Symptom                               | Root Cause                                     | Fix                                                              |
|---|---------------------------------------|------------------------------------------------|------------------------------------------------------------------|
| 1 | vPC not forming — keepalive failing   | mgmt0 IP not configured or VRF mismatch        | Verify `show vpc peer-keepalive` — check source/dest/VRF         |
| 2 | Peer-link shows inconsistent VLANs   | VLAN allowed list mismatch on Po10             | Match `switchport trunk allowed vlan` on both peers              |
| 3 | C9200 uplink shows STP `BLK`         | `vpc peer-link` or `vpc 30` not configured     | Verify `vpc 30` on both Po30 interfaces                          |
| 4 | HSRP not electing N9K-1 as Active    | Priority not set or `preempt` missing          | Add `priority 110` + `preempt` on N9K-1 HSRP group              |
| 5 | STP Root not N9K-1                   | Priority not manually set                      | `spanning-tree vlan 10,20 priority 4096` on N9K-1               |
| 6 | vPC secondary suspends all ports     | Keepalive down + peer-link down simultaneously | Do not lose both — always recover keepalive first                |
| 7 | Traffic blackhole after N9K-1 reload | `preempt` missing on HSRP                      | Add `preempt` so N9K-1 reclaims Active after it comes back up    |
| 8 | `system-priority` mismatch warning   | Different values on each peer                  | Set identical `system-priority 4000` under `vpc domain` on both  |

---

## 📚 References

- [Cisco NX-OS vPC Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/vpc/configuration/guide/b-cisco-nexus-9000-nx-os-vpc-configuration-guide-93x.html)
- [Cisco NX-OS HSRP Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/unicast/configuration/guide/b-cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-93x/b-cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-93x_chapter_010111.html)
- [Cisco Catalyst 9200 Software Configuration Guide](https://www.cisco.com/c/en/us/support/switches/catalyst-9200-series-switches/products-installation-and-configuration-guides-list.html)
- [Cisco NX-OS Spanning Tree Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/stp/configuration/guide/b-cisco-nexus-9000-series-nx-os-spanning-tree-configuration-guide-93x.html)

---

**Anurag Mishra** — Technology Specialist | Network Implementation Engineer  
📍 Hyderabad, India

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/anuragmishra6)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-333?style=for-the-badge&logo=github&logoColor=white)](https://github.com/anuragmlocaladdress)

*All configurations are from a lab environment built in EVE-NG. Sanitize IPs and credentials before production use.*

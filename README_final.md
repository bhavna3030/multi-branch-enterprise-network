# Multi-Branch Enterprise Network (Cisco Packet Tracer)

A segmented enterprise network with one headquarters (HQ) and two branch offices, connected through an ISP. It uses VLANs, router-on-a-stick inter-VLAN routing, VLSM addressing, OSPF, DHCP, NAT (PAT) and ACLs. Connectivity, access restrictions and link failover were verified with `ping`, `tracert` and `show` commands.

**Tool:** Cisco Packet Tracer | **Devices:** 4 routers (2911), 3 switches (2960), PCs and 1 server

---

## Contents
1. [Topology](#topology)
2. [Technologies used](#technologies-used)
3. [Addressing plan (VLSM)](#addressing-plan-vlsm)
4. [VLANs](#vlans)
5. [Port and cable map](#port-and-cable-map)
6. [Configuration summary](#configuration-summary)
7. [Security policy (ACLs)](#security-policy-acls)
8. [Verification and expected output](#verification-and-expected-output)
9. [Test results](#test-results)
10. [Failover test](#failover-test)
11. [Issues faced and how I fixed them](#issues-faced-and-how-i-fixed-them)
12. [Limitations and future improvements](#limitations-and-future-improvements)
13. [Repository structure](#repository-structure)
14. [How to run](#how-to-run)

---

## Topology

```
                          ISP-R (Loopback0 = 8.8.8.8, "the internet")
                             |
                             | s0/0/0   203.0.113.0/30
                             |
                           HQ-R  (NAT, ACLs, DHCP, inter-VLAN routing)
              g0/0 (trunk)  / |  \
                           /  |   \
       HQ-SW  <-----------'   |    '-----------------.
   (VLAN 10/20/30/99)         |                       |
    |      |      |           | g0/1                  | g0/2
  Sales   IT +   Guest        | 172.16.3.0/30         | 172.16.3.4/30
  PCs    Server  PCs          |                       |
                            BR1-R ------------------ BR2-R
                         g0/1 | g0/2   172.16.3.8/30  g0/2 | g0/1
                              |         (backup path)      |
                           g0/0                          g0/0
                            BR1-SW                       BR2-SW
                              |                            |
                           BR1 PC                       BR2 PC
```

Normal path: each branch reaches HQ over its direct link.
Backup path: if a branch-to-HQ link fails, traffic goes through the other branch over the BR1-BR2 link.

> Add your own topology screenshot here: `![Topology](screenshots/01-topology.png)`

---

## Technologies used

| Area | Technology |
|---|---|
| Layer 2 | VLANs, 802.1Q trunking |
| Inter-VLAN routing | Router-on-a-stick (subinterfaces) |
| Addressing | VLSM subnetting of 172.16.0.0/16 |
| Routing | OSPF (single area 0), passive interfaces, default route propagation |
| Services | DHCP (router-based pools), NAT/PAT overload |
| Security | Named extended ACLs |
| Verification | ping, tracert, show commands |

---

## Addressing plan (VLSM)

Base network: `172.16.0.0/16`. Subnets were allocated from largest to smallest so they do not overlap.

| Network | Subnet | Mask | Usable hosts | Gateway / ends |
|---|---|---|---|---|
| HQ Sales (VLAN 10) | 172.16.0.0/24 | 255.255.255.0 | 254 | 172.16.0.1 |
| HQ IT (VLAN 20) | 172.16.1.0/25 | 255.255.255.128 | 126 | 172.16.1.1 |
| HQ Guest (VLAN 30) | 172.16.1.128/26 | 255.255.255.192 | 62 | 172.16.1.129 |
| HQ Mgmt (VLAN 99) | 172.16.1.192/27 | 255.255.255.224 | 30 | 172.16.1.193 |
| Branch 1 LAN | 172.16.2.0/26 | 255.255.255.192 | 62 | 172.16.2.1 |
| Branch 2 LAN | 172.16.2.64/26 | 255.255.255.192 | 62 | 172.16.2.65 |
| HQ-BR1 link | 172.16.3.0/30 | 255.255.255.252 | 2 | HQ .1, BR1 .2 |
| HQ-BR2 link | 172.16.3.4/30 | 255.255.255.252 | 2 | HQ .5, BR2 .6 |
| BR1-BR2 link | 172.16.3.8/30 | 255.255.255.252 | 2 | BR1 .9, BR2 .10 |
| HQ-ISP link | 203.0.113.0/30 | 255.255.255.252 | 2 | HQ .1, ISP .2 |

Static address: HQ server `172.16.1.10/25`, gateway `172.16.1.1`. All other hosts use DHCP.

Router IDs: HQ-R `1.1.1.1`, BR1-R `2.2.2.2`, BR2-R `3.3.3.3`.

---

## VLANs

| VLAN | Name | Switch ports (HQ-SW) |
|---|---|---|
| 10 | SALES | Fa0/1-8 |
| 20 | IT | Fa0/9-16 (server on Fa0/11) |
| 30 | GUEST | Fa0/17-24 |
| 99 | MGMT | none (reserved for management) |

HQ-SW Gi0/1 is an 802.1Q trunk to HQ-R Gi0/0 carrying VLANs 10, 20, 30 and 99.

---

## Port and cable map

| From | To | Cable |
|---|---|---|
| HQ-R g0/0 | HQ-SW g0/1 | Straight-through (trunk) |
| HQ-R g0/1 | BR1-R g0/1 | Cross-over |
| HQ-R g0/2 | BR2-R g0/1 | Cross-over |
| BR1-R g0/2 | BR2-R g0/2 | Cross-over |
| HQ-R s0/0/0 | ISP-R s0/0/0 | Serial DCE (HQ-R is DCE, clock rate 64000) |
| BR1-R g0/0 | BR1-SW g0/1 | Straight-through |
| BR2-R g0/0 | BR2-SW g0/1 | Straight-through |

---

## Configuration summary

Full configs are in the [`configs/`](configs/) folder. Key points:

- **HQ-SW:** VLANs 10/20/30/99, access ports per VLAN, trunk on Gi0/1.
- **HQ-R:** subinterfaces `g0/0.10`, `.20`, `.30`, `.99` (dot1Q) as VLAN gateways; DHCP pools for Sales, IT and Guest; NAT overload on `s0/0/0`; three ACLs; OSPF with LAN subinterfaces set passive; static default route to the ISP, advertised into OSPF with `default-information originate`.
- **BR1-R / BR2-R:** LAN interface (passive in OSPF), two point-to-point links, local DHCP pool for the branch LAN.
- **ISP-R:** serial link to HQ-R and `Loopback0 8.8.8.8/32` to simulate the internet.

---

## Security policy (ACLs)

| ACL | Applied on | Direction | Policy |
|---|---|---|---|
| `GUEST-IN` | HQ-R g0/0.30 | in | Guests cannot reach any internal network (172.16.0.0/20). They can still reach the internet. |
| `SALES-IN` | HQ-R g0/0.10 | in | Sales cannot reach the IT VLAN (172.16.1.0/25), which includes the server. |
| `BRANCH-IN` | HQ-R g0/1 and g0/2 | in | Branch LANs cannot reach the Mgmt VLAN (172.16.1.192/27). |

Each ACL ends with `permit ip any any`, because every ACL has an implicit `deny any` at the end.

---

## Verification and expected output

### OSPF neighbors
```
HQ-R# show ip ospf neighbor
Neighbor ID   Pri  State       Dead Time  Address       Interface
2.2.2.2       1    FULL/BDR    00:00:3x   172.16.3.2    GigabitEthernet0/1
3.3.3.3       1    FULL/BDR    00:00:3x   172.16.3.6    GigabitEthernet0/2
```
(`FULL/DR` or `FULL/BDR` are both fine.) Dead time counts down from 40 seconds.

### Routing table on HQ-R
```
HQ-R# show ip route
S*    0.0.0.0/0 is directly connected, Serial0/0/0
      172.16.0.0/16 is variably subnetted
C        172.16.0.0/24    is directly connected, GigabitEthernet0/0.10
C        172.16.1.0/25    is directly connected, GigabitEthernet0/0.20
C        172.16.1.128/26  is directly connected, GigabitEthernet0/0.30
C        172.16.1.192/27  is directly connected, GigabitEthernet0/0.99
O        172.16.2.0/26    [110/2] via 172.16.3.2, GigabitEthernet0/1
O        172.16.2.64/26   [110/2] via 172.16.3.6, GigabitEthernet0/2
C        172.16.3.0/30    is directly connected, GigabitEthernet0/1
C        172.16.3.4/30    is directly connected, GigabitEthernet0/2
O        172.16.3.8/30    [110/2] via 172.16.3.2 and via 172.16.3.6
C     203.0.113.0/30      is directly connected, Serial0/0/0
```
(Exact metrics can differ slightly by Packet Tracer version.)

### Routing table on a branch router
```
BR1-R# show ip route ospf
O*E2  0.0.0.0/0     [110/1]  via 172.16.3.1, GigabitEthernet0/1
O     172.16.0.0/24 [110/2]  via 172.16.3.1, GigabitEthernet0/1
O     172.16.1.0/25 [110/2]  via 172.16.3.1, GigabitEthernet0/1
O     172.16.2.64/26 [110/2] via 172.16.3.10, GigabitEthernet0/2
```

### NAT
```
HQ-R# show ip nat translations
Pro   Inside global       Inside local        Outside local    Outside global
icmp  203.0.113.1:1       172.16.0.11:1       8.8.8.8:1        8.8.8.8:1
```
All internal hosts appear as `203.0.113.1` (PAT).

### ACL match counters
```
HQ-R# show access-lists
Extended IP access list GUEST-IN
    10 deny ip 172.16.1.128 0.0.0.63 172.16.0.0 0.0.15.255 (4 match(es))
    20 permit ip any any (8 match(es))
Extended IP access list SALES-IN
    10 deny ip 172.16.0.0 0.0.0.255 172.16.1.0 0.0.0.127 (4 match(es))
    20 permit ip any any
Extended IP access list BRANCH-IN
    10 deny ip 172.16.2.0 0.0.0.127 172.16.1.192 0.0.0.31 (4 match(es))
    20 permit ip any any
```
The match counts rise as you run the blocked tests.

### DHCP
```
HQ-R# show ip dhcp binding
IP address      Client-ID/Hardware address   Lease expiration   Type
172.16.0.11     0001.xxxx.xxxx               ...                Automatic
172.16.1.21     0002.xxxx.xxxx               ...                Automatic
```

### Interfaces
`show ip interface brief` should show every used interface as `up/up`, including all four `g0/0.x` subinterfaces on HQ-R.

---

## Test results

| # | Test | From -> To | Expected | Result |
|---|---|---|---|---|
| 1 | Inter-VLAN blocked | Sales PC -> IT server (172.16.1.10) | Fail (SALES-IN) | |
| 2 | Guest isolation | Guest PC -> Sales PC | Fail (GUEST-IN) | |
| 3 | Guest internet | Guest PC -> 8.8.8.8 | Success (NAT) | |
| 4 | Sales internet | Sales PC -> 8.8.8.8 | Success (NAT) | |
| 5 | Branch to HQ server | BR1 PC -> 172.16.1.10 | Success | |
| 6 | Branch to Mgmt blocked | BR1 PC -> 172.16.1.193 | Fail (BRANCH-IN) | |
| 7 | Branch to branch | BR1 PC -> BR2 PC | Success | |
| 8 | Branch internet | BR2 PC -> 8.8.8.8 | Success (default route + NAT) | |
| 9 | Same VLAN | IT PC -> IT server | Success | |
| 10 | DHCP | All PCs set to DHCP | Address from the correct pool | |

Fill the **Result** column with Pass/Fail after you run each test, and link the screenshot.

> Note: an IT PC pinging a Sales PC also fails. The request gets through, but the reply is dropped by `SALES-IN` when it re-enters HQ-R. This is expected with these ACLs, because they are not stateful.

---

## Failover test

| Step | Action | Observed path from BR1 PC to 172.16.1.10 |
|---|---|---|
| 1 | Normal state, `tracert 172.16.1.10` | 172.16.2.1 -> 172.16.3.1 -> 172.16.1.10 |
| 2 | `shutdown` on HQ-R g0/1 | OSPF removes the direct route |
| 3 | `tracert 172.16.1.10` again | 172.16.2.1 -> 172.16.3.10 -> 172.16.3.5 -> 172.16.1.10 |
| 4 | `no shutdown` on HQ-R g0/1 | Direct path returns |

**Why it works:** OSPF cost via the direct link is 1, and via the other branch it is 2, so the direct path is preferred. When the direct link fails, OSPF recalculates and installs the longer path automatically.

> Screenshots: `screenshots/06-traceroute-before.png`, `screenshots/07-traceroute-after.png`

---

## Issues faced and how I fixed them

| Problem | Cause | Fix |
|---|---|---|
| Router-to-router links stayed red | Used a straight-through cable and interfaces were shut down | Used cross-over cables and `no shutdown` on both ends |
| Serial link down | No clock rate on the DCE side, module missing | Installed HWIC-2T, set `clock rate 64000` on HQ-R |
| Branch PCs got `169.254.x.x` | Branch LAN interface had no IP | Configured g0/0 on branch routers |
| OSPF neighbor stuck in EXSTART | Two routers had the same router ID (config pasted on the wrong router) | Corrected the router ID and IP addresses, ran `clear ip ospf process` |
| Commands rejected with `% Invalid input` | Typed config commands in user mode | Used `enable` then `configure terminal` |
| Missing OSPF neighbor | An interface was down/down because the other end was shut down or miscabled | Checked both ends with `show ip interface brief` |
| Trace stopped at the first hop | No OSPF routes learned | Fixed links and OSPF `network` statements |

---

## Limitations and future improvements

- HQ-R is a single point of failure, and so is the single ISP link. A second HQ router with HSRP and a backup ISP would fix this.
- Router-on-a-stick uses one physical link for all VLAN traffic. A Layer 3 switch with SVIs would scale better.
- ACLs are stateless. A firewall or reflexive ACLs would be used in production.
- Not yet implemented: port security, SSH-only management, STP/RSTP tuning, EtherChannel, OSPF authentication, DHCP snooping, a DMZ, and a site-to-site VPN.

---

## Repository structure

```
multi-branch-enterprise-network/
├── README.md
├── multi-branch-network.pkt
├── configs/
│   ├── HQ-R.txt
│   ├── BR1-R.txt
│   ├── BR2-R.txt
│   ├── ISP-R.txt
│   └── HQ-SW.txt
└── screenshots/
    ├── 01-topology.png
    ├── 02-ospf-neighbors.png
    ├── 03-routing-table.png
    ├── 04-nat-translations.png
    ├── 05-access-lists.png
    ├── 06-traceroute-before.png
    ├── 07-traceroute-after.png
    └── 08-dhcp-binding.png
```

---

## How to run

1. Install Cisco Packet Tracer (free with a Cisco Networking Academy account).
2. Open `multi-branch-network.pkt`.
3. Wait about 30-40 seconds for links to turn green and OSPF to converge.
4. Use the commands in [Verification](#verification-and-expected-output) and the [test table](#test-results) to check the network.

---

## Author

**Bhavna** | [GitHub](https://github.com/bhavna3030) | [LinkedIn](https://www.linkedin.com/in/bhavna-r-150119281)

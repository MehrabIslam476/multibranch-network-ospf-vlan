# Enterprise Multi-Branch Campus Network Design & Implementation

[![Cisco Packet Tracer](https://img.shields.io/badge/Cisco%20Packet%20Tracer-v8.x-005073?logo=cisco&logoColor=white)](#)
[![Routing Protocol](https://img.shields.io/badge/Routing-OSPF%20Area%200-orange)](#)
[![Security](https://img.shields.io/badge/Security-Extended%20ACLs%20%7C%20AAA-blue)](#)
[![Switching](https://img.shields.io/badge/Switching-802.1Q%20Inter--VLAN-green)](#)

A production-style multi-branch enterprise campus network designed and simulated in **Cisco Packet Tracer**. The project implements a scalable hierarchical architecture interconnecting two departmental branch LANs with a centralized administrative server infrastructure across high-speed serial WAN links.

---

## Table of Contents
- [Architecture & Topology](#architecture--topology)
- [Key Features & Implemented Technologies](#key-features--implemented-technologies)
- [IP Addressing Schema](#ip-addressing-schema)
- [Network Segmentation & VLANs](#network-segmentation--vlans)
- [Routing & WAN Architecture](#routing--wan-architecture)
- [Access Control & Security Policies](#access-control--security-policies)
- [Centralized Network Services](#centralized-network-services)
- [Device Configuration Highlights](#device-configuration-highlights)
- [Verification & Testing](#verification--testing)
- [How to Run the Simulation](#how-to-run-the-simulation)

---

## Architecture & Topology

```text
               +-----------------------+
               |  Admin Services LAN   |
               |  - Web: .10           |
               |  - DNS: .11           |
               +-----------+-----------+
                           | (192.168.100.0/24)
                   +-------+-------+
                   |  R3 / Admin   |
                   +-------+-------+
                           |
                     [Serial Link] (10.0.23.0/30)
                           |
                   +-------+-------+
                   |   R2 / Core   |----+ (192.168.30.0/24 - Dept-B LAN w/ DHCP)
                   +-------+-------+
                           |
                     [Serial Link] (10.0.12.0/30)
                           |
                   +-------+-------+
                   |   R1 / ROAS   |
                   +-------+-------+
                           | (Trunk 802.1Q)
                   +-------+-------+
                   |   S1 Switch   |
                   +-------+-------+
                      /         \
        (VLAN 10 / Students)   (VLAN 20 / Faculty)
```

The network consists of three primary domains:
1. **Department A (Branch 1 - R1 & S1):** Uses **Router-on-a-Stick (ROAS)** with IEEE 802.1Q encapsulation to segment traffic between `STUDENTS` (VLAN 10) and `FACULTY` (VLAN 20).
2. **Department B (Branch 2 - R2):** Configured with centralized **Cisco IOS DHCP Services** and default gateway assignment.
3. **Administrative Server Farm (Core - R3):** Hosts enterprise **DNS** and **Web (HTTP/S)** application services protected by state-aware extended access control lists.

---

## Key Features & Implemented Technologies

- **Dynamic Routing:** Single-Area **OSPF (Area 0)** across all core routers with explicit router IDs and optimized $/30$ point-to-point subnets.
- **Inter-VLAN Routing:** 802.1Q trunking on Cisco Catalyst 2960 switches with sub-interface routing on Cisco 2911 ISRs.
- **Automated IP Management:** Cisco IOS DHCP Pool on R2 with DNS pointers and static exclusions for infrastructure nodes.
- **Role-Based Access Control:** Inbound Extended ACLs on WAN boundaries filtering HTTP/HTTPS web portal access based on source subnets and IP roles.
- **Core Application Infrastructure:** Configured DNS records (`www.mehrab.ac.bd`) and custom HTTP landing pages.
- **Device Hardening & Baseline Security:** Encrypted privilege passwords (`enable secret`), secured console and auxiliary lines, disabled unused services (`no ip domain-lookup`), and warning login banners.

---

## IP Addressing Schema

| Subnet / Domain | Network Address | Default Gateway | Subnet Mask | Description / Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Dept-A VLAN 10** | `192.168.10.0/24` | `192.168.10.1` | `255.255.255.0` | Students LAN (Sub-interface `g0/0.10`) |
| **Dept-A VLAN 20** | `192.168.20.0/24` | `192.168.20.1` | `255.255.255.0` | Faculty LAN (Sub-interface `g0/0.20`) |
| **Dept-B LAN** | `192.168.30.0/24` | `192.168.30.1` | `255.255.255.0` | Dynamic DHCP Client Subnet |
| **Admin Server LAN**| `192.168.100.0/24` | `192.168.100.1` | `255.255.255.0` | DNS (`.11`) and Web Server (`.10`) Farm |
| **WAN Link (R1 - R2)**| `10.0.12.0/30` | N/A | `255.255.255.252` | Serial Backbone (`R1: .1`, `R2: .2`) |
| **WAN Link (R2 - R3)**| `10.0.23.0/30` | N/A | `255.255.255.252` | Serial Backbone (`R2: .1`, `R3: .2`) |

---

## Network Segmentation & VLANs

On Catalyst Switch **S1**, switchports are partitioned to eliminate broadcast bleed across departments:
- **VLAN 10 (`STUDENTS`):** Ports `Fa0/2` – `Fa0/10` (Access Mode)
- **VLAN 20 (`FACULTY`):** Ports `Fa0/11` – `Fa0/19` (Access Mode)
- **Trunk Uplink (`Fa0/1`):** 802.1Q trunk carrying tagged traffic for VLANs 10 and 20 directly to `R1 Gig0/0`.

```cisco
! Switch S1 Configuration
vlan 10
 name STUDENTS
vlan 20
 name FACULTY
!
interface range fa0/2 - 10
 switchport mode access
 switchport access vlan 10
!
interface range fa0/11 - 19
 switchport mode access
 switchport access vlan 20
!
interface fa0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

---

## Routing & WAN Architecture

Dynamic routing is established using **Open Shortest Path First (OSPF Process ID 1)** within **Area 0 (Backbone)**.

```cisco
! Router R1 OSPF Configuration
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0

! Router R2 OSPF Configuration
router ospf 1
 router-id 2.2.2.2
 network 192.168.30.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0

! Router R3 OSPF Configuration
router ospf 1
 router-id 3.3.3.3
 network 192.168.100.0 0.0.0.255 area 0
 network 10.0.23.0 0.0.0.3 area 0
```

---

## Access Control & Security Policies

An extended access control list (`BLOCK_STUDENTS_ALL`) is enforced inbound on **R3 Serial 0/0/0** to enforce strict departmental privilege boundaries:
- **Restricted Access:** Blocks all Students (`192.168.10.0/24`) and designated restricted hosts in Dept-B (`192.168.30.12`, `192.168.30.14`) from accessing HTTP (Port 80) and HTTPS (Port 443) on the Admin Web Server (`192.168.100.10`).
- **Permitted Access:** Allows unrestricted access for Faculty (`192.168.20.0/24`), general ICMP diagnostics, and DNS queries.

```cisco
! Router R3 Extended Access-List
ip access-list extended BLOCK_STUDENTS_ALL
 deny tcp 192.168.10.0 0.0.0.255 host 192.168.100.10 eq 80
 deny tcp 192.168.10.0 0.0.0.255 host 192.168.100.10 eq 443
 deny tcp host 192.168.30.12 host 192.168.100.10 eq 80
 deny tcp host 192.168.30.12 host 192.168.100.10 eq 443
 deny tcp host 192.168.30.14 host 192.168.100.10 eq 80
 deny tcp host 192.168.30.14 host 192.168.100.10 eq 443
 permit ip any any
!
interface s0/0/0
 ip access-group BLOCK_STUDENTS_ALL in
```

---

## Centralized Network Services

### 1. Dynamic Host Configuration Protocol (DHCP)
Configured on router **R2** for Department B:
- **Pool Name:** `DEPTB`
- **Network Scope:** `192.168.30.0/24`
- **Reserved / Excluded Range:** `192.168.30.1` - `192.168.30.10`
- **DNS Server:** `192.168.100.11`

### 2. Domain Name System (DNS) & HTTP Server
- **DNS Server (`192.168.100.11`):** Authoritative A-record mapping `www.mehrab.ac.bd` $\rightarrow$ `192.168.100.10`.
- **Web Server (`192.168.100.10`):** Hosts the administrative intranet portal for faculty verification.

---

## Verification & Testing

Execute the following commands in Cisco IOS privileged EXEC mode to inspect states and troubleshoot:

```cisco
! Check Layer 2 VLAN assignments and trunk links
S1# show vlan brief
S1# show interfaces trunk

! Verify dynamic routing convergence and neighbor adjacencies
R1# show ip ospf neighbor
R1# show ip route ospf

! Inspect active DHCP leases
R2# show ip dhcp binding

! Verify security rules and packet match counters
R3# show access-lists BLOCK_STUDENTS_ALL
R3# show ip interface s0/0/0 | include access
```

### Connectivity Matrix Summary

| Originating Node | Destination | Protocol / Port | Expected Result |
| :--- | :--- | :--- | :--- |
| **Faculty PC (VLAN 20)** | `www.mehrab.ac.bd` | HTTP (Port 80) | **SUCCESS (200 OK)** |
| **Student PC (VLAN 10)** | `www.mehrab.ac.bd` | HTTP (Port 80) | **DENIED (Timeout by ACL)** |
| **Student PC (VLAN 10)** | DNS Server (`192.168.100.11`) | UDP Port 53 | **SUCCESS** |
| **Dept-B Host** | Gateway (`192.168.30.1`) | ICMP Echo | **SUCCESS (Dynamic Lease)** |

---

## How to Run the Simulation

1. Clone this repository:
   ```bash
   git clone https://github.com/<your-username>/enterprise-campus-network-ospf.git
   ```
2. Open **Cisco Packet Tracer** (version 8.0 or higher).
3. Open the topology file: `enterprise_campus_network.pkt`.
4. Allow 30–45 seconds for Spanning Tree Protocol (STP) and OSPF neighbor convergence (or press **Fast Forward Time**).
5. Open any client desktop browser and navigate to `http://www.mehrab.ac.bd` to verify policy-based ACL filtering.


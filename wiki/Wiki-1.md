# 🏛️ Wiki 1: Network Architecture, Topology & Routing Engineering

Designing a secure, high-availability financial network requires a holistic understanding of how traffic flows from the physical port on an employee's desk to the global internet. This document outlines the architectural paradigms, layer-by-layer segmentation, and dynamic routing trade-offs applied in this enterprise topology.

---

## 1. Architectural Paradigm: Collapsed Core vs. Full Three-Tier

The banking infrastructure was rigorously evaluated against the classical Cisco Three-Tier model (Access, Distribution, Core). 

### The Real-World Headquarters vs. Branch Deployment
In a full-scale Headquarters environment, the Core layer demands absolute maximum throughput, zero-downtime hardware redundancy, and advanced chassis-based switching (typically achieved using modular platforms like the **Cisco Catalyst 6500-E series** or modern Nexus equivalents). However, replicating this massive hardware footprint across regional branches (such as the simulated San Francisco `SFO-MARKET50` or Chicago `CHI-MAIN200` offices) introduces unjustifiable CAPEX (capital expenditure) and OPEX (operational overhead).

### The Collapsed Core Compromise
For the primary New York site (`NYC-WALL11`), I utilized a **Collapsed Core** architecture.
* **Engineering Rationale:** By consolidating the Core and Distribution functions into a single high-performance Layer 3 routing switch, the network achieves wire-speed Inter-VLAN routing via Switched Virtual Interfaces (SVIs) while maintaining a robust aggregation point for the Access layer. 
* **Business Benefit:** It delivers the segmentation and routing benefits of a traditional Three-Tier model without the financial burden of dedicated, standalone core routers at every site.

---

## 2. Layer 2 Segmentation & Blast Radius Reduction

A "flat network" is a significant security and performance liability. To minimize broadcast storms and limit the "blast radius" of any potential internal compromise or malware outbreak, the Access Layer is heavily segmented using **802.1Q VLANs**.

### VLAN Schema & Logical Air-Gapping
Departments are logically air-gapped at Layer 2. Any cross-VLAN communication must be hardware-routed through the L3 Core, providing a centralized chokepoint where future Access Control Lists (VACLs/RACLs) can be strictly enforced.

| VLAN ID | Subnet | Zone / Assignment | Layer 2 Security & Design Notes |
| :--- | :--- | :--- | :--- |
| **10** | `10.0.10.0/24` | **IT Staff** | General IT operations. Ports hardened with Port Security. |
| **20** | `10.0.20.0/24` | **Accounting** | Financial staff. Highly restricted future access to Core Servers. |
| **30** | `10.0.30.0/24` | **Directors** | VIP user segment. Prioritized for bandwidth and QoS. |
| **40** | `10.0.40.0/24` | **Voice (VoIP)** | Dynamically tagged via Cisco Discovery Protocol (CDP). |
| **50** | `10.0.50.0/24` | **Server Core** | Houses internal Databases & AD DS. Strict topological isolation. |
| **100** | `10.0.100.0/24` | **Management** | Out-of-band (OOB) management network for network engineers. |
| **150** | `192.168.150.0/24` | **DMZ** | Public-facing services hosted securely behind the Edge Router. |
| **999** | N/A | **Native VLAN** | Dead-end blackhole VLAN to mitigate VLAN Hopping / Double-Tagging. |

### Unified Communications & IP Telephony Integration
A modern bank does not run separate physical cabling for data and voice. 
* **The PC-to-Phone Cascade:** To minimize physical cabling costs to employee desks, end devices are connected in a cascaded topology (`Access Switch -> IP Phone -> Employee PC`).
* **CDP Tagging & QoS:** The access switch instructs the IP Phone via CDP to tag its voice frames with `VLAN 40`, separating it from the PC's untagged data traffic (which falls into the access VLAN). Logically separating the voice traffic is the critical first step for implementing strict **Quality of Service (QoS)**, ensuring latency-sensitive voice packets are prioritized across all trunk links up to the WAN edge.

---

## 3. Internal Routing Strategy (IGP): OSPFv2

To ensure seamless, fault-tolerant connectivity between the NY HQ and regional branches, **Open Shortest Path First (OSPFv2)** was selected as the Interior Gateway Protocol.

### Core OSPF Engineering Decisions:
1. **Single-Area Design (Area 0):** Given the scope of this deployment, partitioning the network into multiple OSPF areas would overcomplicate the design. A single Backbone Area (`Area 0`) ensures rapid convergence and keeps the Link-State Database (LSDB) manageable.
2. **Transit Networks:** Inter-router communication between the HQ and branches utilizes dedicated transit blocks from the `10.255.255.0/24` range (e.g., `/30` point-to-point links) to prevent IP address waste and simplify route summarization.
3. **Dynamic Route Injection:** Relying on static default routes across a distributed enterprise cannot adapt to link failures. Instead, the HQ WAN Edge Router (`NYC-WALL11-F1-GW1`) acts as the gateway of last resort. By utilizing the `default-information originate` mechanism, it dynamically propagates the `0.0.0.0/0` route to all downstream branch routers as an `O*E2` (External Type 2) route. If the primary internet link drops, the routing table converges automatically.
4. **DMZ Reachability & Redistribution:** The DMZ subnet is directly connected to the Edge Router. To provide internal staff with controlled access to the proxy interface, the connected DMZ subnet is explicitly redistributed into the OSPF process.

---

## 4. Edge Routing & Autonomous Systems (EGP): eBGP

The network perimeter must interact with volatile and unpredictable external environments—namely, Internet Service Providers (ISPs) and the Global Internet. To manage this strict boundary, **Border Gateway Protocol (eBGP)** is deployed.

### Abstracting the Internal Topology
* **Routing Isolation:** The bank operates within its own **Private Autonomous System (`AS 64500`)**. By running BGP exclusively at the external edge, the internal routing protocol (OSPF) is completely isolated from the massive, volatile global routing tables of the simulated ISPs (`AS 100`, `AS 200`, `AS 300`).
* **Route Advertisement:** The HQ Edge Router establishes an eBGP peering session with the primary ISP router. This allows the bank to selectively and dynamically advertise its public-facing Edge IP (`20.20.20.14`), which hosts the NAT pool and the Banking Web Service, to the global routing table.

### Multi-Homing Readiness
Deploying eBGP is a forward-looking architectural decision. While the current simulation may rely on a primary provider, BGP inherently prepares the bank's infrastructure for **Multi-Homing** (connecting to multiple ISPs simultaneously). This allows for enterprise-grade redundancy, seamless traffic failover, and outbound path manipulation via BGP path attributes (such as Local Preference, MED, or AS-Path prepending) in future iterations of the network.
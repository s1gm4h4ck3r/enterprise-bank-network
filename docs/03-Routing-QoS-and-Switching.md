# 🛣️ 3. Routing, Quality of Service (QoS), and Switching

> **Simulation Scope Note:** This document describes the actual **Cisco Packet Tracer Implementation** alongside the intended **Target Enterprise Architecture**. Theoretical enterprise features (such as Cisco Viptela SD-WAN, advanced LLQ/WRED QoS queuing, and dual-core VRRP) represent the production standard and are clearly marked. Within the simulation, routing relies on native Cisco IOS OSPF/BGP capabilities, and Layer 2 protection utilizes standard Rapid-PVST+ edge features without redundant core switches.

---

## 🔄 3.1 Dynamic Routing Architecture (OSPF & eBGP)

To guarantee fault tolerance and rapid adaptation to link failures, static routing is entirely deprecated in favor of dynamic routing protocols.

### Interior Gateway Protocol: OSPFv2 (Internal LAN/WAN)

* **Design Choice:** Open Shortest Path First (OSPFv2) is deployed as the IGP.
* **Area Layout:** A single backbone area (Area 0) is utilized.
* **Transit Links:** Inter-router communication utilizes dedicated 10.255.255.0/24 transit blocks.
* **Default Route Injection:** The HQ Edge Router (`NYC-WALL11-F1-GW1`) acts as the gateway of last resort. By utilizing the `default-information originate` command, it dynamically propagates the 0.0.0.0/0 route to downstream branch routers.

### Exterior Gateway Protocol: eBGP (Internet Edge)

* **Design Choice:** The bank operates within a Private Autonomous System (AS 64500).
* **Architectural Benefit:** BGP completely abstracts the internal OSPF topology from the public internet. The Edge Router establishes eBGP peering sessions with the upstream ISP (AS 100).

---

## 🧠 3.2 Design Rationale: Routing & Switching Architecture

### 1. Collapsed Core vs. Traditional Three-Tier Architecture

* **Engineering Trade-Off:** Collapsing Distribution and Core into Layer 3 multi-layer switches centralizes Switched Virtual Interfaces (SVIs) and routing tables on high-performance backplanes, enabling wire-speed inter-VLAN routing.

### 2. Single-Area OSPF vs. Multi-Area OSPF

* **Rationale:** With a limited number of routers and subnets in the current scope, maintaining a single Link-State Database (LSDB) minimizes Dijkstra SPF computation overhead without introducing the administrative complexity of Area Border Routers (ABRs).

### 3. Private Autonomous System (AS 64500) Isolation

* **Rationale:** Configuring eBGP enforces a clean separation of trust. Upstream ISPs learn only the bank's exposed public boundary via explicit network statements, preventing unintended route leaks.

---

## 🌐 3.3 Branch Connectivity (SD-WAN vs. Standard Routing)

### Target Architecture (Cisco Viptela SD-WAN)

In the intended production environment, Cisco Viptela vEdge routers establish a secure, encrypted IPsec overlay fabric across hybrid underlay transports (Broadband + MPLS). The centralized vManage controller dynamically steers traffic based on application profiles.

### Packet Tracer Implementation (OSPF Transit)

Within the simulation, regional branches (`CHI-MAIN200` and `SFO-MARKET50`) are integrated using standard Cisco ISR 2911 routers. They establish OSPF Area 0 adjacencies over transit interfaces (`10.255.255.0/30` subnets) connecting back to the HQ Edge Router, ensuring seamless internal corporate reachability.

---

## 🚦 3.4 Quality of Service (QoS) Design

### Target Architecture (Advanced Queuing)

The production standard applies strict traffic classification (DSCP) and queuing logic:

* **Voice (VoIP):** Expedited Forwarding (EF) via Low Latency Queuing (LLQ).
* **Banking Data:** Assured Forwarding (AF31) via Class-Based Weighted Fair Queuing (CBWFQ).
* **Bulk / Backups:** Scavenger class via Weighted Random Early Detection (WRED).

### Packet Tracer Implementation (Traffic Separation)

To emulate foundational QoS within the simulation, traffic separation is applied at the Access edge. Access switches utilize the Cisco Discovery Protocol (CDP) to dynamically tag Voice traffic into a dedicated VLAN. Switchports connected to IP Phones are explicitly configured with `switchport voice vlan 40` to isolate latency-sensitive VoIP frames from standard PC data.

---

## 🔀 3.5 Switching & First Hop Redundancy (FHRP)

### Target Architecture (VRRP & Root Bridge Tuning)

To ensure high availability across the enterprise, the target architecture dictates:

* **VRRP:** Virtual Router Redundancy Protocol is configured across dual Core Switches sharing a single Virtual IP (VIP).
* **Root Bridge Hardcoding:** The unified logical Core is hardcoded as the Spanning Tree Root Bridge using the `priority 4096` command.

### Packet Tracer Implementation (Rapid-PVST+ Edge Protection)

Layer 2 resilience is actively configured in the simulation without the need for redundant cores:

* **Protocol:** Rapid-PVST+ (`spanning-tree mode rapid-pvst`) is enforced globally for sub-second convergence.
* **Edge Protection:** All user-facing access ports are explicitly configured with `spanning-tree portfast` to bypass listening/learning states, and `spanning-tree bpduguard enable` to immediately disable the port if an unauthorized switch is connected.
# 🗂️ 2. IP Addressing, VLAN Schema & Network Access Control

> **Simulation Scope Note:** This document outlines both the intended **Target Enterprise Architecture** and the actual **Cisco Packet Tracer Implementation**. Theoretical enterprise features (such as Cisco ISE, 802.1X, Infoblox DDI, and Dynamic ARP Inspection) represent the production standard and are clearly marked. Within the actual simulation, Network Access Control is functionally emulated using strict MAC-based Port Security and DHCP Snooping.

---

## 🌍 2.1 Global Addressing Plan & IPAM Strategy

To conserve routable public IPv4 space and maintain internal topology privacy, the corporate infrastructure utilizes RFC 1918 private addressing.

* **Corporate LAN (Trusted Zone):** Allocated from the 10.0.0.0/8 block. Each regional branch is assigned a dedicated /16 supernet (e.g., HQ is 10.0.0.0/16, San Francisco is 192.168.1.0/24) to facilitate clean route summarization.
* **Demilitarized Zone (DMZ):** Strictly segregated into the 192.168.150.0/24 space to provide immediate visual identification of DMZ assets in routing tables and firewall logs.
* **Transit Links:** Point-to-point OSPF links utilize 10.255.255.0/24, carved into /30 subnets to prevent IP waste.

### DDI Integration (Target Architecture)

In the intended production environment, static IP management via spreadsheets is prohibited. All subnets, DNS records, and DHCP scopes are centrally orchestrated via Infoblox Grid DDI appliances located in the Core Server VLAN (VLAN 50).

### DHCP Relay (Packet Tracer Implementation)

Within the simulation, IP management is handled via centralized DHCP servers. The Core Switch (`NYC-WALL11-F1-CSW1`) acts as a DHCP Relay. The `ip helper-address 10.0.50.101` command is explicitly configured on all user-facing Switched Virtual Interfaces (SVIs), seamlessly forwarding broadcast DHCP requests as unicast packets to the centralized server.

---

## 🏗️ 2.2 VLAN & Subnet Inventory (HQ: NYC-WALL11)

The Access Layer is heavily segmented using 802.1Q VLANs. Cross-VLAN communication is hardware-routed exclusively by the unified Core, where routing boundaries can be strictly enforced.

| VLAN ID | Subnet Allocation | Gateway (SVI) | Zone | Purpose & Applied Security Controls |
| --- | --- | --- | --- | --- |
| **10** | 10.0.10.0/24 | 10.0.10.1 | Trusted | **IT Staff / Engineering.** Standard data network. Target: 802.1X. PT: Port Security & DHCP Snooping. |
| **20** | 10.0.20.0/24 | 10.0.20.1 | Trusted | **Accounting / Finance.** Strictly filtered data network. Target: 802.1X. PT: Port Security & DHCP Snooping. |
| **30** | 10.0.30.0/24 | 10.0.30.1 | Trusted | **Directors / VIP.** Dedicated bandwidth allocation. Target: 802.1X. PT: Port Security. |
| **40** | 10.0.40.0/24 | 10.0.40.1 | Trusted | **Voice (VoIP).** Dedicated to Cisco IP Phones. Traffic dynamically tagged via CDP. |
| **50** | 10.0.50.0/24 | 10.0.50.1 | Trusted | **Data Center Core.** Houses centralized servers, databases, and Active Directory. |
| **100** | 10.0.100.0/24 | 10.0.100.1 | Mgmt | **In-Band Management.** Dedicated segment for network administrators and telemetry. |
| **150** | 192.168.150.0/24 | 192.168.150.1 | DMZ | **Web Services.** Terminated on the Edge Router. Completely isolated from internal SVIs. |
| **200** | 192.168.200.0/24 | Physically Isolated | OOB | **Out-of-Band Mgmt (Target Architecture).** Separate network for LTE Console Servers. |
| **999** | N/A | N/A | Native | **Blackhole VLAN.** The default native VLAN is removed from all 802.1Q trunks. |

---

## 📌 2.3 Critical Host IP Bindings

Certain infrastructure components require statically bound IPs outside of the DHCP scopes.

### Packet Tracer Implementation (Active Bindings)

These IP addresses are explicitly routed and configured in the simulation files (e.g., in NAT rules and ACLs):

* **WAN Edge Public IP (NAT Overload/PAT):** 20.20.20.14
* **Nginx Reverse Proxy Backend (DMZ Web Server):** 192.168.150.101
* **Core Database / DHCP Server:** 10.0.50.101

### Target Architecture Allocations

These addresses represent theoretical enterprise appliances integrated into the network design:

* **F5 BIG-IP WAF Virtual IP (VIP):** 192.168.150.100
* **Zabbix / Graylog Monitoring Stack:** 10.0.100.10 / 10.0.100.11
* **Active Directory / DNS Primary:** 10.0.50.11

---

## 🔒 2.4 End-User Port Configuration & NAC

To minimize structured cabling costs to individual employee desks while maintaining strict security, access layer ports utilize a cascaded topology and dynamic Network Access Control (NAC).

### Physical Path (The Cascade)

* **Flow:** Access Switch Port (RJ45) ➔ Cisco IP Phone (Internal Switch) ➔ User PC (RJ45).
* **Voice VLAN Tagging:** The access switch utilizes the Cisco Discovery Protocol (CDP) to instruct the IP Phone to tag its voice frames with 802.1Q VLAN 40. The PC's data traffic passes through untagged and falls into the configured access VLAN (e.g., VLAN 10 or 20).

### Advanced NAC (Target Architecture)

In the intended production environment, the edge switchport acts as a RADIUS authenticator querying the Cisco Identity Services Engine (ISE):

* Corporate laptops authenticate via machine certificates (802.1X).
* IP Phones and legacy printers authenticate via MAC Authentication Bypass (MAB).
* Dynamic ARP Inspection (DAI) prevents Man-in-the-Middle (MitM) ARP poisoning.

### Layer 2 Security Mechanisms (Packet Tracer Implementation)

Without an ISE server in the simulation, strict explicit trust is emulated directly on the access switchports before any device is allowed to transmit data payloads:

1. **Strict Port Security:** Access ports are hardcoded with MAC-based security. The switch limits the port to a maximum of 2 MAC addresses (`switchport port-security maximum 2`) to accommodate the PC and IP Phone cascade. Addresses are dynamically learned (`mac-address sticky`), and violations restrict the port (`violation restrict`) without bringing the entire link down.
2. **DHCP Snooping:** Access ports are configured as untrusted. The switch intercepts DHCP offers to prevent Rogue DHCP server attacks (`ip dhcp snooping`). Trunk uplinks to the Core are explicitly trusted (`ip dhcp snooping trust`).
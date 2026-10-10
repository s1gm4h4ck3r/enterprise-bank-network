# 🛡️ Wiki 2: Security Architecture, Edge Services & Enterprise Infrastructure

A robust enterprise perimeter is not defined solely by routing protocols and Access Control Lists. This document expands on the **Defense-in-Depth** strategy, detailing the physical layer (L1) deployment, compute/storage fabrics, and the operational observability tools used to secure and monitor the Bank's infrastructure.

---

## 1. Threat Modeling & The DMZ Concept

### The "Flat Network" Vulnerability
In legacy or poorly designed networks, public-facing services (like a Banking Web Service) are often placed in the same routing domain as internal corporate assets. 
* **The Risk:** If an attacker exploits an Application Layer (L7) vulnerability on the Web Server, they gain an immediate foothold in the internal network, allowing for lateral movement (Pivot Attacks) directly to critical databases and management subnets.

### The Segmented Solution (Demilitarized Zone)
To mitigate this, a **Demilitarized Zone (DMZ)** was engineered. The DMZ is a strictly controlled, isolated subnet (`192.168.150.0/24`) residing on a dedicated Edge Router sub-interface. 
* **Architectural Benefit:** Even if the DMZ proxy is fully compromised, the attacker is trapped. L3 routing from the DMZ to the Core Corporate Networks (`10.0.0.0/8` block) is hardware-blocked by the Edge Router.

---

## 2. Physical Layer (L1) & Compute Fabric

Network segmentation (VLAN 50) protects the core logically, but the actual workloads demand enterprise-grade hardware to ensure uninterrupted, high-speed banking operations.

* **Optical Distribution (MMF vs. SMF):** The building's structured cabling converges at the Optical Distribution Frames (ODF). Internal connections between the Core, Distribution, and Server Room utilize **Multi-Mode Fiber (MMF)** to provide massive East-West bandwidth (e.g., for vMotion or database synchronization). Conversely, external Edge connections to the ISPs utilize **Single-Mode Fiber (SMF)** to eliminate modal dispersion and ensure reliable long-haul transmission to the provider's PoP.
* **Virtualization & Compute:** Critical workloads (Core Banking Systems, Processing) are virtualized using **VMware ESXi** hypervisors running on high-density **HPE ProLiant** bare-metal servers.
* **Storage Area Network (SAN):** To support high-IOPS transactional databases (OLTP, Oracle/MS SQL), the compute nodes are integrated with an **HP 3PAR Storage System**, guaranteeing minimal latency for financial transactions.

---

## 3. Security Boundaries: NAT & Traffic Filtering

With the physical and compute layers established, the network boundary is strictly controlled using Cisco ISR capabilities to mitigate external threats and hide the internal topology.

### Dynamic Overload & Static PAT
* **Outbound Privacy (NAT Overload):** Internal staff and management endpoints are masked behind a single public IP via PAT (Overload). This entirely hides the internal IP schema (RFC 1918) from external reconnaissance tools.
* **Inbound Control (Static PAT):** The Banking Service is published using strict Port Forwarding. Instead of a vulnerable 1:1 NAT mapping (which exposes all ports of a server to the internet), only specific TCP ports (`Public IP : 443` -> `Internal IP : 443`) are forwarded. The NAT engine silently drops all other port requests before they even hit the ACL.

### Stateful Emulation via Extended ACLs
To secure the transit path, the `SECURE_DMZ` Extended ACL was engineered to emulate stateful firewall behavior.
* By utilizing the `established` keyword, the router inspects TCP headers (ACK/RST flags). It permits inbound return traffic for sessions initiated from the corporate network, but explicitly drops any unsolicited `SYN` packets from the internet attempting to penetrate the trusted zones.

---

## 4. Layer 2 Security (Access Switches)

Security must be enforced at the edge, where end-user devices connect. The following L2 security features are globally deployed across all floor access switches:
1. **Port Security:** Configured on all user-facing ports. Limits are set to `maximum 2` MAC addresses (to accommodate the PC + IP Phone cascade). The violation mode is set to `restrict` to drop unauthorized frames and generate Syslog traps without physically disabling the port.
2. **DHCP Snooping:** Prevents Rogue DHCP server attacks. Uplinks to the Core Switch are configured as `trust` ports, ensuring clients only receive IP assignments from the legitimate server in VLAN 50.

---

## 5. Observability, Logging & Automation (DevOps Integration)

A secure network must be measurable and auditable. Modern enterprise operations require shifting from manual CLI administration to centralized observability.
* **Monitoring (Zabbix):** Network health, interface utilization, and server uptime are monitored via Zabbix. ICMP (`echo-reply`) and SNMP traffic are explicitly permitted through the ACLs to allow polling without triggering security alerts.
* **Centralized Logging (Graylog):** Security violations (e.g., Port Security `restrict` traps) and ACL deny-logs are aggregated into Graylog. This provides SIEM-like capabilities to detect brute-force attempts or probing in real-time.
* **Infrastructure Automation (Ansible):** Routine configuration backups, VLAN provisioning, and baseline security hardening are automated using Ansible playbooks, ensuring configuration drift is minimized across the Cisco switch fleet.

---

## 6. Architectural Scalability: The NGFW Gap

While the current Cisco IOS L3/L4 filtering provides a highly cost-effective (CAPEX-optimized) security boundary, future scalability in a production financial environment demands Next-Generation solutions.

**Gap Analysis:** Upgrading the perimeter with a dedicated **Check Point NGFW** (e.g., Quantum Series) would introduce:
1. **Deep Packet Inspection (DPI) & App-ID:** Moving beyond port-based ACLs to actual application-layer traffic identification (distinguishing a legitimate HTTPS web request from malicious tunneling over port 443).
2. **Threat Prevention:** Active Intrusion Prevention Systems (IPS) and Sandboxing (Zero-Day threat emulation) directly at the WAN edge.
3. **SSL/TLS Inspection:** Allowing the firewall to decrypt and inspect inbound encrypted traffic for hidden payloads before re-encrypting it for the internal servers.
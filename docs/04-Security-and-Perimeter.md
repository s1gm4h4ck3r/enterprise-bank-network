# 🛡️ 4. Security Architecture, Firewalls, and Perimeter Defense

This document details the Defense-in-Depth strategy deployed across the banking infrastructure. It outlines threat modeling comparisons, firewall enforcement, NAT translation logic, and Application Layer (L7) reverse proxy isolation.

> **Simulation Scope Note:** Advanced technologies such as Cloud DDoS scrubbing, F5 WAF, Check Point NGFW, VMware NSX-T, and VPNs represent the **Target Enterprise Architecture**. They are mentioned here to demonstrate comprehensive perimeter design but are not physically present in the Packet Tracer simulation. Within the actual simulation, these security boundaries are functionally emulated using stateful-like Extended ACLs (`SECURE_DMZ`, `DMZ_OUT_FILTER`) and NAT boundaries on the Cisco ISR Edge Router (`NYC-WALL11-F1-GW1`).

---

## 🎯 4.1 Threat Modeling: Flat Network vs. Segmented DMZ

To understand why this architecture is structured with strict perimeter boundaries, consider the threat vectors present in legacy flat topologies versus the engineered To-Be model:

```text
[Legacy Flaw: Flat Network]
Internet ---> [Edge Router] ---> [Server Subnet (Core Banking + Web + DB in one L2/L3 domain)]
                              * Threat: Compromise of public Web Server yields direct root access to DB.

[Engineered Solution: Defense-in-Depth]
Internet ---> [Cloud DDoS] ---> [Edge Router (NAT)] ---> [DMZ (VLAN 150) / WAF / Nginx Proxy]
                                                                  |
                                                         (NGFW / Core SVI Boundary)
                                                                  v
                                                    [Trusted Zone (VLAN 50: DB / Core)]


```

### Attack Vectors Addressed:

1. **Direct Database Enumeration:** In a flat model, port 1521 (Oracle DB) is directly addressable if routing permits. In this architecture, all incoming traffic terminates at the DMZ Nginx reverse proxy.
2. **Blast-Radius Containment:** Segregating the DMZ onto an isolated sub-interface (`Gi0/0.150`) ensures that perimeter web exploits cannot bypass stateful inspection points to pivot into employee data segments.

---

## ⚖️ 4.2 Architectural Trade-Off: Static PAT vs. 1:1 Static NAT

A critical perimeter decision was whether to deploy **1:1 Full Static NAT** or **Static Port Address Translation (PAT)** for publishing the Banking Web Service.

| Criteria | 1:1 Static NAT | Static PAT (Port Forwarding) | Architectural Verdict |
| --- | --- | --- | --- |
| **Port Exposure** | Exposes all 65,535 TCP/UDP ports to the host unless blocked by ACL. |  |  |

| Exposes **only** explicitly mapped ports (TCP 80, 443).

| **Static PAT Chosen.** Minimizes attack surface at the translation layer.

|
| **IP Consumption** | Consumes one dedicated public IPv4 address per internal server.

| Multiple services can share a single public IP via distinct ports.

| **Static PAT Chosen.** Conserves scarce IPv4 addresses assigned by ISP.

|
| **Misconfiguration Risk** | High. An ACL misconfiguration could expose management ports (e.g., SSH 22).

| Low. Unmapped packets are hardware-dropped by NAT.

| **Static PAT Chosen.** Provides defense-in-depth redundancy.

|

---

## 🌩️ 4.3 Advanced Perimeter Security (Target Architecture)

The following technologies represent the intended production standards for advanced threat mitigation. They are conceptually designed as part of the overall architecture but excluded from the Packet Tracer implementation.

### Edge Security (DDoS & WAF)

* **Volumetric DDoS Protection (Arbor/Akamai):** By advertising the bank's public prefixes (via BGP) through the cloud scrubbing center, malicious traffic is absorbed and dropped in the cloud.
* **Web Application Firewall (F5 BIG-IP):** Positioned inline within the DMZ to terminate client SSL sessions, inspect decrypted L7 payloads for OWASP vulnerabilities, and drop illicit requests.

### Next-Gen Firewalls & Microsegmentation

* **NGFW Perimeter (Check Point Quantum):** Designed to replace legacy stateless packet filters with Deep Packet Inspection (DPI), Application Control, and Threat Emulation.
* **Microsegmentation (VMware NSX-T):** Hypervisor-level rules restrict VM-to-VM flows to prevent lateral movement within the Data Center (e.g., Application Server VM can communicate with Database VM only on TCP port 1521).

---

## 🧱 4.4 Firewall and ACL Emulation (Packet Tracer Implementation)

In the simulation, the Cisco ISR Edge Router (`NYC-WALL11-F1-GW1`) enforces the security boundaries using native IOS features to emulate an NGFW.

### Extended ACL Stateful Emulation (`SECURE_DMZ`)

Stateful firewall behavior is emulated via the `established` keyword on incoming TCP streams:

* Inbound TCP packets on port 80 (HTTP) and 443 (HTTPS) destined for `192.168.150.101` are explicitly permitted.
* Inbound TCP return packets for sessions initiated by internal hosts are permitted (flag checks for ACK/RST control bits via the `established` rule).
* Unsolicited inbound TCP SYN packets targeting internal subnets are unconditionally dropped.

### Strict DMZ Isolation (`DMZ_OUT_FILTER`)

A dedicated Extended ACL applied to the `Gi0/0.150` subinterface explicitly drops any unsolicited traffic originating from the DMZ (`192.168.150.0/24`) toward the corporate LAN (`10.0.0.0/8`). This guarantees that even if a DMZ web server is compromised, it cannot initiate connections to the internal databases.

---

## 💻 4.5 Application Layer (L7) Isolation: The Reverse Proxy

**Network segmentation without application layer termination is incomplete**.

### Engineering Functions of the DMZ Reverse Proxy (Target Architecture):

1. **SSL/TLS Cryptographic Offloading:** Absorbs CPU-intensive TLS handshakes at the edge, relieving internal transaction processors.
2. **HTTP Header Sanitization:** Removes server signatures to prevent targeted reconnaissance.
3. **Strict HTTP-to-HTTPS Redirection:** Unencrypted port 80 requests are automatically rewritten with HTTP 301 redirects to HTTPS on port 443.
4. **Backend Topology Shielding:** Clients connect only to the reverse proxy IP (`192.168.150.101`). The backend application and database architecture are completely shielded from external view.

**💡 Hybrid Simulation Note:** Cisco Packet Tracer cannot perform real TLS cryptographic handshakes, HTTP header parsing, or reverse proxy buffering. To validate this design end-to-end, the L2/L3 routing and NAT were simulated in Packet Tracer, while the Application Layer (Ubuntu Server, Docker containerized Nginx, and TLS certificates) was implemented and benchmarked in a parallel **VirtualBox** environment.

---

## 🔄 4.6 Network Address Translation (NAT) Matrix

The following NAT rules are actively configured on the `NYC-WALL11-F1-GW1` Edge Router to manage translation boundaries.

| NAT Type | Direction | Configuration / Rule | Purpose & Business Logic |
| --- | --- | --- | --- |
| **NAT Overload (PAT)** | Outbound | 10.0.0.0/16 -> 20.20.20.14 | **Privacy:** Internal endpoints are masqueraded behind the Edge Router's public IP using `access-list 1`. Entirely hides internal RFC 1918 space from external reconnaissance. |

|
| **Static PAT** | Inbound | 20.20.20.14:80/443 -> 192.168.150.101:80/443 | **Strict Forwarding:** Public access to the Banking Service is restricted explicitly to mapped ports (TCP 80 and 443). Nemapirani portovi se odbacuju od strane hardverskog NAT mehanizma.

|

---

## 🔐 4.7 Remote Access & VPN Connectivity (Target Architecture)

* **Remote Access VPN (Client-to-Site):** Check Point Mobile Access VPN terminating on the external perimeter for remote workers. Requires Multi-Factor Authentication (MFA).
* **B2B IPsec VPN (Site-to-Site):** Persistent site-to-site IPsec tunnels connecting to financial clearinghouses, enforcing IKEv2 with AES-256-GCM encryption and SHA-384 data integrity verification.
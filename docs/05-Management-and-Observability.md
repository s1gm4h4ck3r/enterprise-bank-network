# 👁️ 5. Management, Observability, and Automation

To ensure network reliability, security auditing, and configuration consistency, all management infrastructure is designed for high availability and forensic integrity.

> **Simulation Scope Note:** Within the Packet Tracer simulation, management is practically implemented via In-Band routing (VLAN 100) and secure SSH access. The external telemetry collectors (SNMP/Syslog), NTP hierarchy, Out-of-Band (OOB) networks, and automation nodes described below represent the **Target Enterprise Architecture** meant to integrate with this network foundation, but are not physically configured in the `.pkt` file.

---

## 🚨 5.1 Out-of-Band (OOB) Management (Target Architecture)

Relying purely on in-band management (VLAN 100) is a critical risk during catastrophic network failures (e.g., routing loops, ISP outages, or misconfigured ACLs locking out administrators).

* **Infrastructure:** Opengear Network Resilience platform console servers are deployed in the HQ MDF and Branch racks.
* **Physical Connectivity:** These devices connect physically (via RJ45 Rollover serial cables) to the console ports of all core routers, switches, NGFWs, and WAFs.
* **Cellular Backup:** The console servers feature dedicated LTE/4G cellular modems and distinct Out-of-Band VPNs. This allows network engineers to securely access CLI interfaces and recover equipment even if the primary ISPs go completely offline.

---

## ⏱️ 5.2 Network Time Protocol (NTP) Infrastructure (Target Architecture)

Financial regulations (MiFID II, PCI DSS) require strictly synchronized timestamps for transaction logging and SIEM event correlation. Even a few milliseconds of drift can invalidate an audit trail.

* **Hierarchy (Stratum Design):**
* **Stratum 1:** The bank's edge routers sync securely to external Stratum 1 atomic clock sources (e.g., NIST, regional ISP time servers).
* **Stratum 2:** The Core Switches sync to the edge routers and act as the internal Stratum 2 NTP servers for the entire corporate network.


* **Distribution & Security:** All internal edge devices, hypervisors, and end-user endpoints sync exclusively to the internal Core Switches via VLAN SVIs. No internal device is permitted to query internet time servers directly (blocked by NGFW policy).

---

## 📊 5.3 Monitoring, SIEM & Automation Design (Target Architecture)

Modern enterprise operations require shifting from manual CLI administration to centralized observability and Infrastructure as Code (IaC).

### Network Monitoring (Zabbix & Grafana)

* **Data Collection:** The target architecture dictates a Zabbix server (10.0.100.10) utilizing SNMPv3 with authPriv (SHA-256/AES-256 encryption) to poll network infrastructure devices, and Zabbix Agents for ESXi hosts and DMZ proxies.
* **Metrics Visualization:** Zabbix data is exported to Grafana to provide Network Operations Center (NOC) engineers with real-time dashboards for bandwidth utilization, interface errors, and latency metrics.

### Centralized Logging (Graylog / Splunk)

* **Log Aggregation:** Routine syslog data (interface flaps, BGP adjacency changes, port security traps, VRRP state transitions) are forwarded via UDP 514 to a Graylog ingestor (10.0.100.11) for fast indexing and operational troubleshooting.
* **Security Event Correlation (SIEM):** Critical security events are forwarded to a heavy enterprise SIEM, Splunk Enterprise Security, for SOC analysis and incident response.

### Infrastructure as Code (Ansible & HashiCorp Vault)

* **Configuration Management:** An Ansible control node (10.0.100.12) automates baseline security standards, routine VLAN provisioning, and firmware compliance checks across the Cisco fleet using SSH.
* **Secrets Management:** Passwords, API keys, and SNMP strings used by Ansible are never hardcoded in playbooks. They are dynamically retrieved at runtime from HashiCorp Vault.

---

## ⚙️ 5.4 In-Band Management & Access (Packet Tracer Implementation)

While advanced telemetry is planned for the target architecture, the core administrative access and management isolation are actively configured in the simulation.

### Dedicated Management VLAN (VLAN 100)

* **Network Isolation:** A dedicated In-Band management network (`10.0.100.0/24`) is strictly isolated from user traffic.
* **Switching Infrastructure:** The SVI for VLAN 100 is configured on the `NYC-WALL11-F1-CSW1` Core Switch (`10.0.100.1`), which acts as the gateway. Access-layer connectivity for administrative laptops is provisioned on the dedicated management switch `NYC-WALL11-F4-ASW1`.

### Secure CLI Access (SSHv2)

* **Encrypted Sessions:** Telnet is disabled. The Edge Router (`NYC-WALL11-F1-GW1`) is configured with a domain name (`wall11.corp`), RSA crypto keys, and enforces secure access on virtual terminal lines using `transport input ssh`.
* **Local Authentication:** Device access requires explicit credentials authenticated against the local user database (e.g., `username netadmin privilege 15 secret...`).
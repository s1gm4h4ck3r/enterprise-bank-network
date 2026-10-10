# 🌐 Wiki 3: Web Services, SSL & Application Isolation

While Network Address Translation (NAT) and Access Control Lists (ACLs) provide robust Layer 3/Layer 4 boundary security, relying solely on them to publish a bare web application is an architectural anti-pattern. 

This section details the Application Layer (L7) defense strategy for the **Banking Service**, utilizing the concept of a Reverse Proxy and centralized SSL/TLS termination within the Demilitarized Zone (DMZ).

---

## 1. The Problem with Direct Application Exposure

In a standard, unprotected deployment, an external client connects directly to the backend application server (e.g., an internal Java, Node.js, or Apache Tomcat service) hosted in the corporate network. 

### The Security Risk
* **L7 Vulnerabilities:** Direct exposure leaves the application's native vulnerabilities, HTTP header leaks, and potential memory overflow exploits completely accessible to the global internet. 
* **Stack Reconnaissance:** Attackers can easily fingerprint the backend technology stack (OS version, framework versions) and execute targeted zero-day attacks. If the application crashes or is exploited, the core server itself is compromised, granting the attacker a direct pivot point into the internal network.

---

## 2. The Reverse Proxy Architecture (The Buffer Zone)

To mitigate L7 threats and abstract the internal topology, the DMZ Server (`192.168.150.101`) does not host the actual banking application. Instead, it acts as an **Nginx Reverse Proxy**, forming a secure abstraction layer between external users and the internal backend services.

### Traffic Interception & Sanitization
1. External requests hitting the public Edge IP (`20.20.20.14:443`) are hardware-forwarded by the Cisco ISR router to the DMZ proxy.
2. The Nginx proxy accepts the TCP connection, drops malformed packets, and sanitizes the HTTP headers.
3. It then initiates a *new*, separate, and clean connection to the actual backend application.

### Architectural Benefit
The external attacker never interacts with the actual Banking Application directly. They only interact with the hardened, containerized proxy daemon. This drastically reduces the attack surface and completely hides the internal technological stack from automated reconnaissance tools (like Shodan or Nmap).

---

## 3. SSL/TLS Termination & Edge Encryption

Serving financial, corporate, or authenticated services over unencrypted HTTP (Port 80) exposes user session cookies and credentials to Man-in-the-Middle (MitM) attacks and packet sniffing.

### Enforcement of Secure Connections
The DMZ proxy enforces strict edge encryption using the following methodologies:
1. **HTTP to HTTPS Redirection:** Any legacy or accidental traffic arriving on Port 80 is immediately intercepted by the proxy and answered with an HTTP `301 Moved Permanently` redirect. This forces the client browser to upgrade and reconnect over Port 443 (HTTPS).
2. **Centralized Edge Encryption (SSL Termination):** Instead of managing distributing SSL certificates across dozens of backend application servers, **SSL Termination** occurs entirely on the Reverse Proxy. The Nginx server decrypts the incoming HTTPS traffic using its private key (RSA-2048) and passes the sanitized, plain-text requests to the protected internal backend.
3. **Cryptographic Offloading (Performance Optimization):** TLS handshakes and cipher negotiations are extremely CPU-intensive. Offloading this cryptographic processing to the dedicated Nginx proxy frees up valuable CPU cycles on the backend servers, allowing them to focus 100% of their compute resources on processing banking business logic and database queries.

---

## 4. Defense-in-Depth Integration (Summary)

This project demonstrates a true multi-layered security approach (Defense-in-Depth), moving seamlessly from the physical wire up to the application level. The final inbound traffic flow is strictly controlled across multiple OSI layers:

1. **L3/L4 Perimeter (Edge Router):** Static PAT drops any non-80/443 traffic at the hardware ASIC level. The `SECURE_DMZ` Extended ACL ensures that even if the DMZ proxy is compromised, it cannot initiate L3 connections into the Corporate Core VLANs.
2. **L7 Proxy Buffer (DMZ):** Nginx absorbs the external connections, terminates SSL/TLS, sanitizes headers, and completely masks the backend architecture.
3. **Application Layer (Core):** The internal Banking Service only communicates with the local proxy, remaining completely blind and inaccessible to the outside world.
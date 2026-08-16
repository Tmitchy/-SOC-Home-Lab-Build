<h1 align="center">🖥️ SOC Home Lab - Build & Log Forwarding Plan</h1>
<h3 align="center">Phase 1: Environment Setup- Getting logs flowing before the SIEM goes in</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Stage-Environment_Build-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/SIEM-To_Be_Added-lightgrey?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Platform-VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white"/>
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white"/>
  <img src="https://img.shields.io/badge/pfSense-212121?style=for-the-badge"/>
</p>  
     
---

## 📋 Purpose

To ensure a SIEM generates meaningful alerts, it must first receive consistent, well-configured log data. This document addresses **Stage 1** of my SOC lab build: setting up the operating systems and configuring each one to send logs for the SIEM in the next phase.

---
## Stage 1 - Complete

Four virtual machines have been deployed and configured on an isolated VirtualBox network. The environment is now prepared for centralized log collection during the next phase.

---
### 🧱 Lab Inventory - Virtual Machines

| # | OS | Role in Lab | Purpose |
|---|---|---|---|
| 1 | **pfSense Firewall** | Perimeter / Gateway | Sits at the edge of the lab network, filtering and is the first log source that shows *what tried to get in* before it ever reaches an endpoint |
| 2 | **Windows 11** | Endpoint / Workstation | Simulates a user endpoint where phishing, malware execution, and user-behavior events originate |
| 3 | **Windows Server** | Domain Controller / Server | Active Directory, DNS, authentication logs, the "crown jewel" logs in most real SOCs |
| 4 | **Ubuntu Server** | Linux Server | Simulates backend/infra services - SSH, sudo, and service logs | "Where the Elastic Stack SIEM is located and where Kibana and Logstash are configured."
| 5 | **Ubuntu (Desktop Image)** | Linux Endpoint | Comparison point to the Windows endpoint cross-platform log normalization practice |

> All VMs run on **VirtualBox**, networked on an internal/host-only adapter so traffic stays isolated from my home network.

---

### 🌐 Planned Network Layout

```mermaid
graph TD
    F[pfSense Firewall - Perimeter] -->|Firewall/Traffic Logs| S[SIEM - TBD]
    F --> A[Windows 11 Endpoint]
    F --> B[Windows Server - AD/DNS]
    F --> C[Ubuntu Server]
    F --> D[Ubuntu Desktop]
    A -->|Event Logs / Sysmon| S
    B -->|Security & AD Logs| S
    C -->|Syslog / auditd| S
    D -->|Syslog| S
    S --> E[Dashboards & Detection Rules]
```

The SIEM box itself isn't built in this phase yet; it's about making sure every VM is *ready to forward* the moment it's introduced.

---

### ⚙️ Per-OS Configuration Plan

#### 🔥 Firewall - Perimeter Configuration & Logging

**Goal:** Every packet that reaches an endpoint passes through here first - this is the earliest point in the lab where I can see attempted access, not just what already landed on a host.

- Deploying **pfSense** as a virtual appliance, sitting between the lab's internal network and the outside (host-only/NAT boundary)
- Configuring baseline **firewall rules**:
  - Default-deny inbound, explicit allow rules per VM/service
  - Segmenting the Windows Server (DC) and Ubuntu Server onto a restricted internal subnet, separate from the two endpoints
- Enabling **logging on both blocked and allowed traffic** - blocked traffic shows attempted access; allowed traffic gives a baseline of "normal" to compare against later
- Turning on the **pfSense package for log export** (planning to use the built-in syslog forwarding) so logs can ship out the same way the OS logs do
- Log shipper: forwarding via **syslog** to the SIEM once it's introduced - no separate agent needed, since pfSense speaks syslog natively

![](https://raw.githubusercontent.com/Tmitchy/-SOC-Home-Lab-Build/main/images/pfsense.png)

---
#### 🪟 Windows 11 (Endpoint)
**Goal:** Capture process creation, network connections, and user activity at the endpoint level.

- Install **Elastic Agent/Defend (Native EDR Telemetry)** with a solid config to log:
  - Process creation (Event ID 1)
  - Network connections (Event ID 3)
  - Image/DLL loads (Event ID 7)
  - Security log
  - Sysmon operational log
  - PowerShell operational log

- Enable **PowerShell Script Block Logging** (catches obfuscated/malicious scripts)

![](https://raw.githubusercontent.com/Tmitchy/-SOC-Home-Lab-Build/main/images/Windows%20-%2011.png)
  
---

#### 🪟 Windows Server (Domain Controller)
**Goal:** Capture authentication and directory-service events, the backbone of most SOC detections.

- Enable **Advanced Audit Policy** (not just legacy auditing):
  - Logon/Logoff events (4624, 4625, 4634)
  - Account management (4720, 4726, etc.)
  - Kerberos ticket events (4768, 4769) - useful for later detecting things like Kerberoasting
- Enable **DNS debug/analytic logging** for visibility into resolution requests
- Same **Elastic Agent/Defend (Native EDR Telemetry)** plan as the Windows 11 box, pointed at:
  - Security log
  - DNS Server log
  - Directory Service log

![](https://raw.githubusercontent.com/Tmitchy/-SOC-Home-Lab-Build/main/images/Windows-Server%20(DC).png)

---

#### 🐧 Ubuntu Server
**Goal:** Capture authentication, privilege escalation, and service-level activity.

- Configure **rsyslog** to centralize local logs (`/var/log/auth.log`, `/var/log/syslog`)
- Install **auditd** for deeper visibility:
  - `sudo`/`su` usage
  - File integrity on sensitive paths (`/etc/passwd`, `/etc/shadow`)
- **Elastic Agent/Defend (Native EDR Telemetry)**, forwarding:
  - `auth.log`
  - `syslog`
  - `audit.log`

![](https://raw.githubusercontent.com/Tmitchy/-SOC-Home-Lab-Build/main/images/Ubuntu%20Server.png)

---

#### 🐧 Ubuntu Desktop (Image)
**Goal:** A lighter-weight Linux endpoint for comparison against the Windows 11 box.

- Same **rsyslog** baseline as the server
- **Elastic Agent/Defend (Native EDR Telemetry)** forwarding auth
- Used mainly to practice normalizing Linux vs. Windows log formats once they hit the SIEM

![](https://raw.githubusercontent.com/Tmitchy/-SOC-Home-Lab-Build/main/images/Ubuntu-Image.png)

---

### ✅ Progress... - Stage 1 (before SIEM is introduced)

- [x] Elastic Agent/Defend (Native EDR Telemetry) installed & configured on Windows 11
- [ ] Sysmon installed & configured on Windows 11
- [ ] Advanced Audit Policy enabled on Windows Server
- [x] Elastic Agent/Defend (Native EDR Telemetry) installed on both Windows boxes
- [ ] auditd installed & rules applied on Ubuntu Server
- [x] Elastic Agent/Defend (Native EDR Telemetry) installed on both Ubuntu boxes
- [x] All 5 VMs confirmed reachable on the internal lab network
- [x] Static IPs assigned to each VM for consistent log source identification

---

### 📚 Resources I'm Using

#### Windows Logging
- [Microsoft — Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations)
- [Windows Security Event Log Encyclopedia (Ultimate Windows Security)](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)

#### Linux Logging
- [rsyslog Documentation](https://www.rsyslog.com/doc/)
- [Linux auditd Guide (Red Hat)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/security_hardening/auditing-the-system_security-hardening)

#### Log Shipping (Elastic Stack)
- [Elastic — Ingesting Windows Event Logs](https://www.elastic.co/guide/en/beats/winlogbeat/current/how-winlogbeat-works.html)

#### Networking & Lab Setup
- [VirtualBox Networking Modes Explained](https://www.virtualbox.org/manual/ch06.html)
- [Active Directory Home Lab Setup Guide](https://adsecurity.org/?page_id=41)

#### Detection Engineering (for later phases)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Sigma Rules Repository](https://github.com/SigmaHQ/sigma)

> This list will keep growing as the lab progresses - new resources get added as I hit new problems.

---

### 🔜 Next Document: 
[**SIEM Introduction (Stage-2)**](https://github.com/Tmitchy/SIEM-Introduction)

The next write-up in this series will focus on setting up the SIEM (Elasticsearch), linking each Elastic Agent/Defend (Native EDR Telemetry) to it, and creating the initial dashboards and detection rules based on this log data.

---

<p align="center"><i>Part 1 of a multi-part SOC home lab build series.</i></p>

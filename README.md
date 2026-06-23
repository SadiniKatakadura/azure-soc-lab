# Azure SOC Home Lab — Threat Detection with Microsoft Sentinel

![Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?style=flat&logo=microsoft-azure&logoColor=white)
![Sentinel](https://img.shields.io/badge/Microsoft_Sentinel-SIEM-blue?style=flat)
![Ubuntu](https://img.shields.io/badge/Ubuntu_24.04_LTS-E95420?style=flat&logo=ubuntu&logoColor=white)
![KQL](https://img.shields.io/badge/KQL-Detection_Engineering-green?style=flat)
![MITRE](https://img.shields.io/badge/MITRE_ATT%26CK-Mapped-red?style=flat)

## Overview

Designed and deployed a cloud-native Security Operations Centre (SOC) lab integrating Microsoft Sentinel with a local Ubuntu Server 24.04 honeypot VM to simulate, detect, and investigate real attack scenarios. The lab demonstrates end-to-end threat detection engineering: from attack simulation using Kali Linux, through log ingestion via Azure Arc and Azure Monitor Agent, to custom KQL analytic rules mapped to the MITRE ATT&CK framework.

This project was built to replicate the core workflows of an entry-level SOC analyst role — threat detection, alert triage, and incident documentation.

---

## Architecture

```
┌─────────────────┐     Host-Only Network      ┌──────────────────────┐
│   Kali Linux VM │ ──────────────────────────► │ Ubuntu Server 24.04  │
│   (Attacker)    │     192.168.56.0/24         │ (Honeypot / Target)  │
└─────────────────┘                             └──────────┬───────────┘
                                                           │
                                                 Azure Monitor Agent
                                                 (via Azure Arc)
                                                           │
                                                           ▼
                                              ┌────────────────────────┐
                                              │  Log Analytics         │
                                              │  Workspace (soc-lab-law)│
                                              └──────────┬─────────────┘
                                                         │
                                                         ▼
                                              ┌────────────────────────┐
                                              │  Microsoft Sentinel     │
                                              │  5x KQL Analytic Rules  │
                                              │  Incident Generation    │
                                              └────────────────────────┘
```

**Key design decision:** Using Kali Linux for controlled attack simulation rather than random internet noise allows precise demonstration of specific MITRE ATT&CK techniques and validates each detection rule against known attack patterns.

---

## Technologies Used

| Component | Technology |
|---|---|
| SIEM Platform | Microsoft Sentinel |
| Log Storage | Azure Log Analytics Workspace |
| VM Management | Azure Arc (Connected Machine Agent v1.65) |
| Log Shipping | Azure Monitor Agent (AMA) v1.42 |
| Honeypot OS | Ubuntu Server 24.04 LTS |
| Attack Platform | Kali Linux |
| Virtualisation | Oracle VirtualBox 7.2 |
| Query Language | KQL (Kusto Query Language) |
| Framework | MITRE ATT&CK |

---

## Detection Rules

Five custom scheduled analytic rules were authored in KQL, each targeting a distinct attack technique and mapped to the MITRE ATT&CK framework.

| # | Rule Name | Severity | MITRE Tactic | MITRE Technique |
|---|---|---|---|---|
| 1 | SSH Brute Force Detection | High | Credential Access | T1110.001 |
| 2 | Successful SSH Login After Brute Force | High | Initial Access | T1078 |
| 3 | New Local User Account Created | High | Persistence | T1136.001 |
| 4 | Sudo Privilege Escalation | High | Privilege Escalation | T1548.003 |
| 5 | SSH Login from New Source IP | High | Initial Access | T1078 |

All rule `.kql` files are located in [`/detection-rules`](./detection-rules/).

---

## Attack Simulations Performed

The following attacks were executed from Kali Linux against the Ubuntu honeypot over the Host-Only network (192.168.56.0/24):

**1. SSH Brute Force (T1110.001)**
Tool: Hydra
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.101 -t 4 -V
```
Generated hundreds of failed SSH authentication attempts, triggering Rule 01 and Rule 02.

**2. Backdoor User Creation (T1136.001)**
```bash
sudo useradd -m backdoor-user
sudo passwd backdoor-user
```
Simulates post-compromise persistence by creating an unauthorised local account.

**3. Sudo Privilege Escalation (T1548.003)**
```bash
sudo useradd testuser
echo "testuser:Test@1234" | sudo chpasswd
su - testuser
sudo id
```
Simulates an attacker escalating from a low-privilege shell to root using sudo.

---

## Incidents Generated

Microsoft Sentinel generated the following incidents from the detection rules:

| Incident ID | Name | Severity | Status |
|---|---|---|---|
| 1 | SSH Brute Force Detection | High | Active |
| 2 | Sudo Privilege Escalation | High | Active |


---

## Infrastructure Setup Summary

### Azure Arc Connection
The Ubuntu VM was registered with Azure Arc using the Connected Machine Agent, enabling Azure-native management and monitoring of the local VirtualBox VM without requiring it to be hosted in Azure.

```bash
# Verification command
sudo azcmagent show | grep "Agent Status"
# Output: Agent Status : Connected
```

### Azure Monitor Agent
AMA v1.42 was deployed via Azure CLI to ship Ubuntu syslog data to the Log Analytics Workspace:

```bash
az connectedmachine extension create \
  --resource-group soc-lab-rg \
  --machine-name soc-honeypot \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --type AzureMonitorLinuxAgent \
  --location westus2
```

### Data Collection Rule
A DCR was configured to collect `auth` and `authpriv` syslog facilities at `LOG_DEBUG` level, routing data to the `soc-lab-law` Log Analytics Workspace. This captures all SSH authentication events, sudo usage, and user management activity.

### Verification KQL Query
```kql
Syslog
| where Computer == "soc-honeypot"
| take 10
```

---

## Incident Response Walkthrough

A documented IR walkthrough for the SSH Brute Force incident is available in [`/docs/incident-response.md`](./docs/incident-response.md).

---

## Repository Structure

```
azure-soc-lab/
├── README.md
├── detection-rules/
│   ├── 01-ssh-brute-force.kql
│   ├── 02-ssh-success-after-bruteforce.kql
│   ├── 03-new-local-user.kql
│   ├── 04-sudo-privilege-escalation.kql
│   └── 05-ssh-new-source-ip.kql
├── docs/
│   └── incident-response.md
└── screenshots/
    ├── 01-sentinel-analytics-rules.png
    ├── 02-incidents-dashboard.png
    ├── 03-ssh-bruteforce-incident-detail.png
    ├── 04-syslog-kql-results.png
    └── 05-azure-arc-connected.png
```

---

## Key Skills Demonstrated

- **Detection Engineering** — Authored 5 production-style KQL analytic rules from scratch
- **SIEM Administration** — Deployed and configured Microsoft Sentinel on Azure
- **Cloud Infrastructure** — Connected on-premises VM to Azure using Arc and AMA
- **Offensive Security Knowledge** — Simulated brute force, persistence, and privilege escalation attacks
- **Incident Response** — Triaged and documented real alerts generated by the lab
- **Linux Administration** — Configured Ubuntu Server, auditd, SSH hardening, and syslog
- **MITRE ATT&CK Framework** — Mapped all detections to specific tactics and techniques

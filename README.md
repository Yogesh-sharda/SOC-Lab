# 🛡️ Wazuh SOC Lab — Active Directory Threat Detection Environment

> A complete enterprise-style Security Operations Center (SOC) lab built for educational purposes —
> simulating insider threats, lateral movement, and attack detection using open-source tools.

---

## 📌 Project Overview

Built during **Semester 6 — B.Tech Cybersecurity Specialization**.

The scenario: *An attacker has gained access to one internal workstation. Enumerate the internal network,
move laterally across systems, and demonstrate how critical systems can be compromised — while a SIEM
watches and detects everything in real time.*

This lab covers:
- Internal network enumeration from a compromised workstation
- Lateral movement simulation across domain-joined systems
- Real-time threat detection using **Wazuh SIEM**
- MITRE ATT&CK technique mapping
- Windows Security Event log analysis (Event IDs 4625, 4720, 4732, 4104, 4672)

> ⚠️ **Disclaimer:** All attack simulations were performed exclusively inside a privately owned, isolated
> virtual lab environment for educational and defensive cybersecurity research only.
> Never perform these techniques against systems you do not own or have explicit written authorization to test.

---

## 🏗️ Lab Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          HOST MACHINE                                │
│                     Fedora Linux (QEMU/KVM)                          │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Windows Server  │  │   Windows 10     │  │   Kali Linux     │   │
│  │      2019        │  │   (Victim /      │  │   (Attacker)     │   │
│  │  Domain          │  │    Endpoint)     │  │                  │   │
│  │  Controller      │  │                 │  │  Nmap, CME,      │   │
│  │  DNS + AD DS     │  │  Wazuh Agent    │  │  Enum4Linux,     │   │
│  │  192.168.122.10  │  │  192.168.122.20 │  │  SMBClient       │   │
│  └────────┬─────────┘  └────────┬────────┘  └──────────────────┘   │
│           │                     │                                    │
│           └──────────┬──────────┘                                   │
│                 virbr0 NAT Network (192.168.122.0/24)                │
└──────────────────────────────────────────────────────────────────────┘
                               │
                     Internet (AWS EC2)
                               │
                               ▼
                  ┌────────────────────────────┐
                  │       AWS EC2 Ubuntu       │
                  │       Wazuh v4.7.5         │
                  │  Manager + Dashboard       │
                  │       + Indexer            │
                  └────────────────────────────┘
```

See full topology → [`architecture/lab-topology.md`](architecture/lab-topology.md)

---

## 🖥️ Virtual Machines

| Machine | Role | OS | RAM | Storage |
|---|---|---|---|---|
| Windows Server 2019 | Active Directory DC + DNS | Windows Server 2019 | 4 GB | 50 GB |
| Windows 10 | Victim Endpoint (Wazuh Agent) | Windows 10 | 4 GB | 40 GB |
| Kali Linux | Attacker Machine | Kali Linux 2024 | 3 GB | 30 GB |
| AWS EC2 | Wazuh SIEM Server | Ubuntu 22.04 | 4 GB | 50 GB |

---

## 🛠️ Technologies Used

| Category | Tools |
|---|---|
| Virtualization | QEMU/KVM, Virtual Machine Manager (Fedora) |
| SIEM | Wazuh v4.7.5 (Manager + Dashboard + Indexer) |
| Domain Services | Active Directory Domain Services (AD DS) |
| Attacker Tools | Nmap, CrackMapExec, Enum4Linux, SMBClient, PowerShell |
| Endpoint Logging | Windows Security Events, Wazuh Agent |
| Cloud | AWS EC2 (Ubuntu 22.04) |
| Framework | MITRE ATT&CK |

---

## 📋 Project Phases

| Phase | Description | Status |
|---|---|---|
| 1 | Fedora Host + VM Setup | ✅ Complete |
| 2 | Active Directory + Domain Controller | ✅ Complete |
| 3 | Windows 10 Domain Join | ✅ Complete |
| 4 | Wazuh SIEM on AWS EC2 | ✅ Complete |
| 5 | Wazuh Agent on Windows 10 | ✅ Complete |
| 6 | Attack Simulation from Kali | ✅ Complete |
| 7 | Detection & Analysis in Wazuh | ✅ Complete |

---

## 🎯 Detection Summary

| Attack | Tool Used | Event ID | Wazuh Alert | MITRE Technique |
|---|---|---|---|---|
| Network host discovery | Nmap | — | Connection attempts | T1046 |
| SMB share enumeration | crackmapexec, smbclient | 4624 / 4625 | SMB Auth Events | T1135, T1021.002 |
| Brute force simulation | crackmapexec | 4625 | ✅ DETECTED | T1110 |
| Backdoor account creation | net user | 4720 | ✅ DETECTED | T1136.001 |
| Privilege escalation | net localgroup | 4732 | ✅ DETECTED | T1078 |
| PowerShell abuse | PowerShell | 4104 | ✅ DETECTED | T1059.001 |
| Defender disable attempt | Set-MpPreference | 5001 | ✅ DETECTED | T1562.001 |

---

## 📂 Repository Structure

```
wazuh-soc-lab/
│
├── README.md                        ← You are here
│
├── screenshots/
│   └── README.md                    ← Screenshot index & descriptions
│
├── architecture/
│   └── lab-topology.md              ← Full network diagram & IP table
│
├── configs/
│   ├── ossec-agent.conf             ← Sanitized Wazuh agent config
│   └── sysmon-config.xml            ← Sysmon config (future)
│
├── reports/
│   └── SOC-Lab-Report.md            ← Full project report
│
├── attack-simulations/
│   ├── 01-network-recon.md          ← Nmap reconnaissance
│   ├── 02-smb-enumeration.md        ← SMB enumeration
│   ├── 03-brute-force.md            ← Failed login simulation
│   ├── 04-privilege-escalation.md   ← Account creation & escalation
│   └── 05-powershell-abuse.md       ← PowerShell attack detection
│
└── docs/
    ├── setup-guide.md               ← Full beginner setup guide
    ├── troubleshooting.md           ← Every error & fix documented
    └── event-id-reference.md        ← Windows Event ID cheat sheet
```

---

## 🚧 Problems Faced & Solutions

| Problem | Root Cause | Fix |
|---|---|---|
| VM ping blocked (100% packet loss) | Windows Firewall blocks ICMP by default | Enabled Echo Request (ICMPv4-In) rule in Windows Defender Firewall |
| Domain login failed after promotion | Account scope changed to `YOG\Administrator` | Used domain-prefixed format at login |
| No internet after domain join | DC DNS doesn't forward internet queries | Added `8.8.8.8` as alternate DNS on Windows 10 |
| Wazuh agent "Never Connected" | Ports 1514/1515 closed in AWS Security Group | Added inbound rules for TCP 1514 and TCP 1515 in EC2 |
| Agent version mismatch error | Agent v4.13/4.14 newer than manager v4.7.5 | Installed exact matching agent version 4.7.5 |
| KVM network confusion | Expected VMware-style networking; KVM uses `virbr0` | Used default KVM NAT network |
| Lost Wazuh dashboard password | Credentials only shown once during install | Reset via `wazuh-passwords-tool.sh` on EC2 |
| `agent-auth.exe` PowerShell error | Quoted paths with parameters need `&` operator | Used `& "path\agent-auth.exe" -m <ip> -A <name>` |

Full troubleshooting log → [`docs/troubleshooting.md`](docs/troubleshooting.md)

---

## 🔮 Future Improvements

- [ ] Sysmon integration for richer process telemetry
- [ ] Mimikatz credential dumping detection
- [ ] Custom Sigma rules in Wazuh
- [ ] Pass-the-Hash simulation
- [ ] BloodHound Active Directory enumeration
- [ ] SOAR playbooks for automated response
- [ ] Honeypot SMB share for deception
- [ ] Wazuh email alerting

---

## 🎓 Skills Demonstrated

`SIEM Engineering` `Active Directory` `Threat Detection` `Windows Security` `Log Analysis`
`Linux Administration` `Cloud Deployment (AWS)` `Network Security` `Endpoint Monitoring`
`MITRE ATT&CK` `Blue Team Operations` `PowerShell` `Nmap` `CrackMapExec`

---

## 📰 Full Write-Up

Read the complete Medium article:
[I Built a Home SOC Lab from Scratch — Here's Everything That Went Wrong (and Right)](https://medium.com/@yogeshsharda)

---

## 👤 Author

**Yogesh Sharda** — B.Tech, Cybersecurity Specialization (Semester 6)

> *"Theory without practice is just trivia."*

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

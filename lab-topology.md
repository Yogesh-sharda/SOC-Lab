# 🗺️ Lab Topology & Network Architecture

## Network Overview

All virtual machines run on a single Fedora Linux host using QEMU/KVM.
The default KVM NAT network (`virbr0`) provides the internal subnet.
Wazuh SIEM runs on a separate AWS EC2 instance — isolated from the monitored machines.

> Running the SIEM on a separate cloud instance reflects real enterprise architecture:
> the monitoring plane should never sit on the same network as the systems it watches.

---

## IP Address Table

| Machine | Role | IP Address | Network |
|---|---|---|---|
| Fedora Host | Hypervisor | Real Wi-Fi IP (redacted) | Physical |
| Windows Server 2019 | Domain Controller + DNS | 192.168.122.10 | virbr0 (NAT) |
| Windows 10 | Victim Endpoint | 192.168.122.20 | virbr0 (NAT) |
| Kali Linux | Attacker | 192.168.122.x (DHCP) | virbr0 (NAT) |
| KVM Gateway | virbr0 Gateway | 192.168.122.1 | virbr0 (NAT) |
| AWS EC2 Ubuntu | Wazuh SIEM | Public IP (redacted) | Internet |

---

## Network Diagram

```
FEDORA HOST (Physical Machine)
└── QEMU/KVM Hypervisor
    └── virbr0 NAT Bridge (192.168.122.0/24)
        │
        ├── Windows Server 2019 [192.168.122.10]
        │     ├── Active Directory Domain Services
        │     ├── DNS Server (yog.local)
        │     └── Domain: YOG
        │
        ├── Windows 10 [192.168.122.20]
        │     ├── Domain joined: yog.local
        │     └── Wazuh Agent v4.7.5 → reports to AWS
        │
        └── Kali Linux [192.168.122.x — DHCP]
              ├── Attacker / Red Team machine
              └── Tools: Nmap, CrackMapExec, SMBClient, Enum4Linux

                              │
                    virbr0 → Internet
                              │
                              ▼
                    AWS EC2 — Ubuntu 22.04
                    ┌──────────────────────┐
                    │   Wazuh v4.7.5       │
                    │   ─────────────────  │
                    │   Wazuh Manager      │ ← receives logs from agents
                    │   Wazuh Indexer      │ ← stores and indexes events
                    │   Wazuh Dashboard    │ ← web UI for SOC analysis
                    └──────────────────────┘
```

---

## Port Requirements (AWS Security Group)

| Port | Protocol | Purpose |
|---|---|---|
| 443 | TCP | Wazuh Dashboard (HTTPS web UI) |
| 1514 | TCP/UDP | Agent log forwarding to manager |
| 1515 | TCP | Agent registration |
| 22 | TCP | SSH access to EC2 instance |

---

## Domain Configuration

| Setting | Value |
|---|---|
| Forest | yog.local |
| Domain | yog.local |
| NetBIOS Name | YOG |
| Functional Level | Windows Server 2016 |
| Domain Controller | DC01 (192.168.122.10) |
| DNS Authority | DC01 (itself) |
| Domain Admin | YOG\Administrator |

---

## Network Design Decisions

**Why KVM's default virbr0 network?**
Initially planned to use a custom Host-Only subnet matching VMware conventions.
KVM enforces its own networking model. Fighting the defaults caused hours of confusion.
Solution: use virbr0 as-is. It provides NAT (internet access) and internal VM communication simultaneously.

**Why AWS EC2 for Wazuh instead of a local VM?**
A local Wazuh VM would have required 4+ GB extra RAM on the host.
AWS EC2 Free Tier provides sufficient resources. More importantly, it separates the monitoring
plane from the monitored machines — which is correct architectural practice.

**Why static IPs for the Domain Controller?**
The Domain Controller is also the DNS server for the domain.
If its IP changes, every domain-joined machine loses DNS resolution and authentication breaks.
Static IP on the DC is non-negotiable in any AD environment.

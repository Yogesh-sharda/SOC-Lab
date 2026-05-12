# SOC Lab Report
## Detection of Insider Threats and Lateral Movement Using Wazuh SIEM in an Active Directory Environment

**Author:** Yogesh Sharda
**Semester:** 6th — B.Tech Cybersecurity Specialization
**Lab Environment:** Fedora + QEMU/KVM + AWS EC2
**Wazuh Version:** v4.7.5
**Domain:** yog.local

---

## 1. Abstract

This report documents the design, deployment, and testing of a home Security Operations Center (SOC) lab
built to simulate a real-world insider threat and lateral movement scenario. The lab consists of an
Active Directory environment with a domain controller, a victim endpoint monitored by Wazuh SIEM,
and a Kali Linux attacker machine. Attack simulations include network reconnaissance, SMB enumeration,
brute force authentication attacks, unauthorized account creation, privilege escalation, and PowerShell
abuse. All attacks were detected and logged by Wazuh, with automatic MITRE ATT&CK technique mapping.

---

## 2. Objectives

1. Build an enterprise-style internal network using virtual machines
2. Configure Active Directory Domain Services with a domain controller and joined endpoint
3. Deploy Wazuh SIEM on a cloud-isolated monitoring plane (AWS EC2)
4. Simulate attacker behavior from a Kali Linux machine
5. Detect and analyze attack indicators in the Wazuh dashboard
6. Map detected techniques to the MITRE ATT&CK framework
7. Document the full attack-detection lifecycle for SOC analysis

---

## 3. Lab Architecture

### 3.1 Network Topology

All VMs run on a Fedora Linux host via QEMU/KVM using the default virbr0 NAT network (192.168.122.0/24).
Wazuh runs on a separate AWS EC2 instance to isolate the monitoring plane.

### 3.2 Machine Inventory

| Machine | Role | IP |
|---|---|---|
| Windows Server 2019 | Active Directory DC + DNS | 192.168.122.10 |
| Windows 10 | Victim Endpoint | 192.168.122.20 |
| Kali Linux | Attacker | 192.168.122.x (DHCP) |
| AWS EC2 Ubuntu | Wazuh SIEM | Redacted |

### 3.3 Domain Configuration

| Setting | Value |
|---|---|
| Domain | yog.local |
| NetBIOS | YOG |
| Functional Level | Windows Server 2016 |
| Domain Admin | YOG\Administrator |

---

## 4. Setup Summary

### Phase 1 — Fedora Host & Virtualization

```bash
sudo dnf install @virtualization
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER
```

### Phase 2 — Active Directory

- Installed AD DS role via Server Manager
- Promoted Windows Server to Domain Controller
- Forest and domain: `yog.local`
- DSRM recovery password set
- DNS delegation warning ignored (expected in isolated labs)

### Phase 3 — Windows 10 Domain Join

- Configured static IP: 192.168.122.20
- DNS set to DC: 192.168.122.10 / Alternate: 8.8.8.8
- Joined domain via `sysdm.cpl → Computer Name → Change → Domain: yog.local`
- Post-join login: `YOG\Administrator`

### Phase 4 — Wazuh on AWS EC2

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
chmod +x wazuh-install.sh
sudo ./wazuh-install.sh -a
```

AWS Security Group — Required Inbound Rules:

| Port | Protocol | Purpose |
|---|---|---|
| 443 | TCP | Dashboard |
| 1514 | TCP/UDP | Agent logs |
| 1515 | TCP | Agent registration |

### Phase 5 — Wazuh Agent on Windows 10

```powershell
# Verify connectivity
Test-NetConnection <WAZUH_IP> -Port 1514
# TcpTestSucceeded : True

# Stop service
Stop-Service Wazuh

# Register agent
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m <WAZUH_IP> -A Win10-Client

# Start service
Start-Service Wazuh
```

Key lesson: **Agent version must exactly match manager version.**
Manager was v4.7.5. Agents v4.13 and v4.14 both failed with version mismatch.
Installing agent v4.7.5 resolved the issue immediately.

---

## 5. Attack Simulations

### 5.1 Network Reconnaissance

**MITRE:** T1046 (Network Service Discovery), T1018 (Remote System Discovery)

```bash
# Host discovery
nmap -sn 192.168.122.0/24

# Full port scan
nmap -sS 192.168.122.20

# Service + OS detection
nmap -A 192.168.122.20
```

**Findings:**
- DC01 at 192.168.122.10 — Windows Server 2019, ports 53 (DNS), 88 (Kerberos), 389 (LDAP), 445 (SMB)
- WIN10 at 192.168.122.20 — Windows 10, port 445 (SMB) open

---

### 5.2 SMB Enumeration

**MITRE:** T1135 (Network Share Discovery), T1021.002 (SMB/Windows Admin Shares)
**Event IDs:** 4624, 4625

```bash
# Anonymous share listing
smbclient -L //192.168.122.20 -N

# Subnet-wide SMB enumeration
crackmapexec smb 192.168.122.0/24

# Null session attempt
crackmapexec smb 192.168.122.20 -u '' -p ''

# Full domain enumeration
enum4linux -a 192.168.122.10
```

**Output:**
```
SMB  192.168.122.10  445  DC01   [*] Windows Server 2019 x64
SMB  192.168.122.20  445  WIN10  [*] Windows 10.0 Build 19041
```

---

### 5.3 Brute Force / Failed Login Simulation

**MITRE:** T1110 (Brute Force), T1110.001 (Password Guessing)
**Event ID:** 4625 — An account failed to log on

```bash
crackmapexec smb 192.168.122.20 -u fakeuser -p fakepass
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass1
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass2
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass3
```

**Output:**
```
SMB  WIN10  [-] WIN10\fakeuser:fakepass STATUS_LOGON_FAILURE
SMB  WIN10  [-] WIN10\administrator:wrongpass1 STATUS_LOGON_FAILURE
```

**Wazuh Detection:** Search `4625` in Security Events. Spike of failures from attacker IP visible.
MITRE tag: T1110 applied automatically.

---

### 5.4 Unauthorized Account Creation (Persistence)

**MITRE:** T1136.001 (Create Local Account), T1078 (Valid Accounts)
**Event IDs:** 4720 (Account Created), 4732 (User Added to Group)

```cmd
net user hacker Password123! /add
net localgroup administrators hacker /add
net user hacker
```

**Wazuh Detection:**
- Event 4720 → "A user account was created" → tagged T1136
- Event 4732 → "A member was added to a security-enabled local group" → tagged T1078

---

### 5.5 PowerShell Abuse

**MITRE:** T1059.001 (PowerShell), T1105 (Ingress Tool Transfer), T1562.001 (Disable Defender)
**Event ID:** 4104 (Script Block Logging)

```powershell
# Enable Script Block Logging first
reg add "HKLM\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" `
  /v EnableScriptBlockLogging /t REG_DWORD /d 1

# Suspicious outbound request
Invoke-WebRequest http://example.com

# Process enumeration
Get-Process

# Disable Windows Defender
Set-MpPreference -DisableRealtimeMonitoring $true

# Obfuscated encoded command (attacker evasion technique)
powershell -enc SQBFAFgA
```

**Wazuh Detection:** Every command captured as Event ID 4104, forwarded and indexed.
Search `powershell` in Security Events. Obfuscated commands appear as-is.

---

## 6. Detection Results

| Attack | Tool | Event ID | Detected | MITRE |
|---|---|---|---|---|
| Network reconnaissance | Nmap | — | Partial | T1046 |
| SMB enumeration | crackmapexec, smbclient | 4624/4625 | ✅ Yes | T1135 |
| Brute force login | crackmapexec | 4625 | ✅ Yes | T1110 |
| Account creation | net user | 4720 | ✅ Yes | T1136.001 |
| Privilege escalation | net localgroup | 4732 | ✅ Yes | T1078 |
| PowerShell abuse | PowerShell | 4104 | ✅ Yes | T1059.001 |
| Defender disabled | Set-MpPreference | 5001 | ✅ Yes | T1562.001 |

### Wazuh Dashboard — First 24 Hours

| Metric | Value | Meaning |
|---|---|---|
| Total Events | 480+ | Agent actively shipping Windows event logs |
| Critical Alerts (Level 12+) | 0 at baseline | Clean environment before attack phase |
| Authentication Failures | 2 at baseline → spike during attack | 4625 brute force visible as anomaly |
| Successful Authentications | 37 | Domain auth, services, user sessions |

---

## 7. Key Findings

1. **Brute force is immediately visible** — even a small number of failed logins (3–5) from the same IP
   creates a detectable spike in Event ID 4625 logs. A real SOC analyst would investigate within minutes.

2. **Account creation and privilege escalation leave permanent audit trails** — Events 4720 and 4732
   are logged whether or not the attacker cleans up. Wazuh captures these in real time.

3. **PowerShell logging catches obfuscated commands** — Script Block Logging decodes and records
   base64-encoded PowerShell before execution. There is no evasion once logging is enabled.

4. **Network reconnaissance has limited detection without Sysmon** — Nmap scans generate connection
   attempts but no high-fidelity events without Sysmon's network telemetry. This is a gap.

5. **Version matching is critical in SIEM deployment** — A single version mismatch between agent and
   manager causes silent connection failures. Always verify `wazuh-control info` first.

---

## 8. Mitigation Recommendations

| Finding | Recommendation |
|---|---|
| Brute force possible via SMB | Implement account lockout policy (5 attempts → 30 min lockout) |
| No MFA on domain accounts | Enable MFA for all privileged accounts |
| Flat network allows lateral movement | Implement VLAN segmentation between endpoint and server |
| PowerShell abuse detected | Enforce Constrained Language Mode + AppLocker |
| Defender can be disabled by admin | Deploy EDR with tamper protection enabled |
| Local admin accounts used | Implement LAPS (Local Administrator Password Solution) |
| No network segmentation | Deploy firewall rules between endpoint subnet and DC subnet |

---

## 9. Troubleshooting Log

| Problem | Root Cause | Fix Applied |
|---|---|---|
| VM ping blocked (100% loss) | Windows Firewall blocks ICMP | Enabled Echo Request (ICMPv4-In) inbound rule |
| Domain login failed | Account became domain-scoped after promotion | Used `YOG\Administrator` format |
| No internet after domain join | DC DNS has no internet forwarder | Added `8.8.8.8` as alternate DNS |
| Agent "Never Connected" | AWS ports 1514/1515 closed | Added inbound rules in EC2 Security Group |
| Agent version mismatch | Agent v4.14 > Manager v4.7.5 | Installed matching agent v4.7.5 |
| KVM network confusion | Expected VMware networking model | Used default virbr0 NAT network |
| Lost Wazuh password | Credentials shown only once at install | Reset via `wazuh-passwords-tool.sh` |
| agent-auth.exe syntax error | PowerShell requires `&` for quoted paths | Used `& "path\agent-auth.exe" -m <ip>` |

---

## 10. Conclusion

This lab successfully demonstrated the complete attack-detection lifecycle for an insider threat and
lateral movement scenario. Starting from a single compromised workstation on the internal network,
an attacker can enumerate hosts, enumerate shares, attempt credential brute force, create persistence
accounts, escalate privileges, and abuse PowerShell — all within minutes.

Every one of these techniques left detectable traces in Windows Security Event Logs, which Wazuh
collected, indexed, and automatically mapped to MITRE ATT&CK techniques.

The most important lesson: **attacks are just logs.** Every action an attacker takes generates
event data. The question is always whether that data is collected, indexed, and watched.
Wazuh answers that question for free, in a home lab, on open-source software.

---

## 11. References

- [Wazuh Documentation](https://documentation.wazuh.com)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Windows Security Event IDs](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)
- [CrackMapExec Documentation](https://github.com/byt3bl33d3r/CrackMapExec)
- [Nmap Reference Guide](https://nmap.org/book/man.html)

# 📋 Windows Security Event ID Reference

Quick reference for all Event IDs relevant to this SOC lab.
Use these as search terms in the Wazuh Security Events dashboard.

---

## Authentication Events

| Event ID | Description | When It Fires | SOC Relevance |
|---|---|---|---|
| **4624** | Successful logon | User/service logs in successfully | Baseline — monitor for unusual times/sources |
| **4625** | Failed logon | Wrong password or unknown username | Brute force detection (T1110) |
| **4634** | Account logoff | User session ends | Session tracking |
| **4648** | Logon using explicit credentials | `runas` or network auth with alt creds | Lateral movement indicator |
| **4672** | Special privileges assigned to new logon | Admin-level session started | Privileged access monitoring |
| **4776** | Domain controller validated credentials | NTLM authentication attempt | Pass-the-hash detection |

---

## Account Management Events

| Event ID | Description | When It Fires | SOC Relevance |
|---|---|---|---|
| **4720** | User account was created | `net user /add` executed | Persistence detection (T1136) |
| **4722** | User account was enabled | Disabled account re-enabled | Suspicious if unexpected |
| **4723** | Password change attempt | User tries to change their password | Normal — monitor for policy violations |
| **4724** | Password reset attempt | Admin resets a user's password | Account manipulation |
| **4725** | User account disabled | Account deactivated | Could be attacker disabling detection tools |
| **4726** | User account deleted | `net user /delete` | Account cleanup or tampering |
| **4728** | Member added to global security group | Added to Domain Admins etc | Privilege escalation (T1078) |
| **4732** | Member added to local security group | Added to Administrators group | Privilege escalation (T1078) |
| **4738** | User account changed | Account properties modified | Account manipulation |
| **4756** | Member added to universal security group | Added to enterprise-wide group | Privilege escalation |

---

## Process & Execution Events

| Event ID | Description | When It Fires | SOC Relevance |
|---|---|---|---|
| **4688** | New process created | Any program starts | Malware execution, lateral tools (T1059) |
| **4689** | Process exited | Program terminates | Paired with 4688 for full lifecycle |
| **4104** | PowerShell Script Block logged | Any PowerShell command runs | PowerShell abuse (T1059.001) |
| **4103** | PowerShell module logging | PowerShell module loaded | PowerShell abuse |

> To get 4688, enable: `Computer Configuration → Audit Policy → Audit Process Creation`

---

## System & Service Events

| Event ID | Description | When It Fires | SOC Relevance |
|---|---|---|---|
| **7045** | New service installed | Service created | Persistence (T1543.003) |
| **7040** | Service start type changed | Service startup mode modified | Evasion technique |
| **4697** | Service was installed | Audit log for service install | Persistence |

---

## Windows Defender Events

| Event ID | Description | When It Fires | SOC Relevance |
|---|---|---|---|
| **5001** | Real-time protection disabled | `Set-MpPreference -Disable...` | Defense evasion (T1562.001) |
| **5004** | Real-time monitoring config changed | Defender settings modified | Evasion |
| **1116** | Malware detected | Defender finds a threat | Malware incident |
| **1117** | Action taken on malware | Defender quarantined/removed | Incident response |

---

## Network & Firewall Events

| Event ID | Description | When It Fires | SOC Relevance |
|---|---|---|---|
| **5156** | Network connection allowed | Outbound connection permitted | Traffic monitoring |
| **5157** | Network connection blocked | Firewall blocked a connection | Anomaly detection |
| **4946** | Firewall rule added | New firewall exception created | Persistence (T1562) |

---

## Scheduled Task Events

| Event ID | Description | When It Fires | SOC Relevance |
|---|---|---|---|
| **4698** | Scheduled task created | New task registered | Persistence (T1053) |
| **4702** | Scheduled task updated | Task modified | Persistence |
| **4699** | Scheduled task deleted | Task removed | Cleanup after attack |

---

## How to Search in Wazuh

### By Event ID
```
data.win.system.eventID:4625
```

### By Event ID (simple search bar)
Just type: `4625`

### By Source IP (find attacker)
```
data.srcip:192.168.122.67
```

### By Username
```
data.win.eventdata.targetUserName:hacker
```

### By MITRE Technique
```
rule.mitre.id:T1110
```

---

## Quick SOC Investigation Workflow

1. **Alert fires** → note Event ID and timestamp
2. **Search Event ID** in Wazuh Security Events
3. **Expand the alert** → check source IP, username, hostname
4. **Check surrounding events** → what happened before and after?
5. **Correlate across machines** → did the same source IP touch other hosts?
6. **Map to MITRE ATT&CK** → what tactic/technique does this align with?
7. **Determine severity** → isolated event or part of a chain?
8. **Document findings** → alert timeline, affected accounts, remediation

---

## Most Important Event IDs for This Lab

| Priority | Event ID | Why |
|---|---|---|
| 🔴 Critical | 4625 | Brute force / failed logins |
| 🔴 Critical | 4720 | New account created — persistence |
| 🔴 Critical | 4732 | Added to Administrators — escalation |
| 🔴 Critical | 5001 | Defender disabled — evasion |
| 🟡 High | 4104 | PowerShell executed — execution |
| 🟡 High | 4688 | New process created — execution |
| 🟡 High | 4698 | Scheduled task created — persistence |
| 🟢 Medium | 4624 | Successful logon — baseline/anomaly |
| 🟢 Medium | 4648 | Explicit credential use — lateral movement |

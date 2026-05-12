# Attack 02 — SMB Enumeration

**MITRE ATT&CK:** T1135 (Network Share Discovery) | T1021.002 (SMB/Windows Admin Shares)
**Tools:** CrackMapExec, SMBClient, Enum4Linux
**Event IDs:** 4624 (Successful Logon), 4625 (Failed Logon)
**Performed from:** Kali Linux

---

## Objective

Enumerate SMB shares, users, and domain information from the internal network.
SMB (Server Message Block) is one of the most common lateral movement vectors in Windows environments.

---

## Commands Used

### Step 1 — List SMB Shares Anonymously

```bash
smbclient -L //192.168.122.20 -N
```

**Flags:**
- `-L` — list shares on target
- `-N` — no password (anonymous / null session)

**Expected output:**
```
Sharename    Type    Comment
---------    ----    -------
ADMIN$       Disk    Remote Admin
C$           Disk    Default Share
IPC$         IPC     Remote IPC
```

---

### Step 2 — Enumerate SMB Across Full Subnet

```bash
crackmapexec smb 192.168.122.0/24
```

**What this does:** Probes every host on the subnet for SMB. Reveals:
- Hostname
- OS version
- Domain membership
- SMB signing status

**Output:**
```
SMB  192.168.122.10  445  DC01   [*] Windows Server 2019 x64 (name:DC01) (domain:yog.local) (signing:True)
SMB  192.168.122.20  445  WIN10  [*] Windows 10.0 Build 19041 x64 (name:WIN10) (domain:yog.local) (signing:False)
```

> **Key finding:** WIN10 has SMB signing: False — this machine is vulnerable to SMB relay attacks.

---

### Step 3 — Null Session Attempt

```bash
crackmapexec smb 192.168.122.20 -u '' -p ''
```

Tests whether the machine allows unauthenticated SMB access.

---

### Step 4 — Full Domain Enumeration (Enum4Linux)

```bash
enum4linux -a 192.168.122.10
```

**What this does:** Comprehensive SMB enumeration including:
- Domain users list
- Password policy
- Shares
- Groups
- OS information

---

### Step 5 — Enumerate Shares with Valid Credentials

```bash
crackmapexec smb 192.168.122.20 -u administrator -p 'Password123!' --shares
```

---

## Wazuh Detection

Each SMB connection attempt generates Windows authentication events.

**In Wazuh Dashboard:**
1. Security Events → Search: `4625`
2. Filter by source IP (Kali)
3. Look for: `data.win.system.eventID:4625`

**What the alert shows:**
- Source IP of enumeration
- Target account name
- Failure reason (e.g., `STATUS_LOGON_FAILURE`)
- Timestamp

---

## Why SMB Matters for Lateral Movement

SMB is the primary Windows file sharing and remote management protocol.
Once an attacker finds open SMB shares or valid credentials:
- They can read/write files remotely
- Execute commands via PsExec or WMI
- Move files (tools, payloads) between machines
- Access sensitive data without additional exploits

---

## Defensive Notes

| Control | Effect |
|---|---|
| Disable SMB v1 | Removes legacy, unauthenticated access |
| Enable SMB signing on all hosts | Prevents SMB relay attacks |
| Block SMB (445) between endpoints | Stops lateral movement via SMB |
| Restrict null sessions | `RestrictAnonymous = 2` in Group Policy |
| Monitor Event ID 4625 spikes | Detect enumeration/brute force attempts |

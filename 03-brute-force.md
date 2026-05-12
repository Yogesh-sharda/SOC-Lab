# Attack 03 — Brute Force / Failed Login Simulation

**MITRE ATT&CK:** T1110 (Brute Force) | T1110.001 (Password Guessing)
**Tool:** CrackMapExec
**Event ID:** 4625 — An account failed to log on
**Performed from:** Kali Linux

---

## Objective

Simulate a credential brute force attack over SMB to trigger failed login detection in Wazuh.
This is one of the most common and most detectable attack patterns in enterprise environments.

---

## Commands Used

### Step 1 — Single Failed Login Test

```bash
crackmapexec smb 192.168.122.20 -u fakeuser -p fakepass
```

**Output:**
```
SMB  192.168.122.20  445  WIN10  [-] WIN10\fakeuser:fakepass STATUS_LOGON_FAILURE
```

---

### Step 2 — Simulate Multiple Failed Attempts

```bash
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass1
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass2
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass3
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass4
crackmapexec smb 192.168.122.20 -u administrator -p wrongpass5
```

Each of these generates one **Windows Event ID 4625** on the target machine.
Multiple rapid failures from the same source IP is the brute force detection pattern.

---

### Step 3 — Username Spraying (Multiple Accounts, Same Password)

```bash
crackmapexec smb 192.168.122.20 -u administrator -u guest -u fakeuser -p Password123
```

**Why spraying?**
Password spraying tries one common password across many accounts.
It avoids account lockout policies triggered by many attempts on a single account.

---

## What Happens on the Target (Windows 10)

Every failed SMB login generates Event ID 4625 in the Windows Security log:

```
Event ID: 4625
Description: An account failed to log on.

Subject:
  Security ID:    SYSTEM
  Account Name:   WIN10$
  Logon Type:     3 (Network)

Failure Information:
  Failure Reason: Unknown user name or bad password
  Status:         0xC000006D
  Sub Status:     0xC000006A

Network Information:
  Source IP:      192.168.122.67  ← Kali attacker
  Source Port:    [random high port]
```

---

## Wazuh Detection

**Where to look:**
1. Wazuh Dashboard → Security Events
2. Search: `4625`
3. Or search: `T1110` for MITRE-tagged brute force alerts

**What Wazuh shows:**
- Source IP (attacker's IP)
- Target hostname and account
- Number of failures over time (spike visible in timeline)
- MITRE ATT&CK tag: T1110 — Brute Force

**Detection pattern:**
A single 4625 is normal. 5+ failures from the same source IP in under 60 seconds
is the brute force detection threshold for most SIEM rules.

---

## Creating a Custom Wazuh Rule (Optional)

Add to `/var/ossec/etc/rules/local_rules.xml` on the Wazuh manager:

```xml
<group name="windows,authentication_failed,">

  <rule id="100002" level="10" frequency="5" timeframe="60">
    <if_matched_sid>60122</if_matched_sid>
    <description>Brute force attack detected: 5+ failed logins in 60 seconds</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>

</group>
```

---

## Defensive Notes

| Control | Effect |
|---|---|
| Account lockout policy (5 attempts) | Stops brute force by locking the account |
| MFA on all domain accounts | Password alone is not enough even if guessed |
| Disable NTLM authentication | Forces Kerberos — harder to brute force |
| Monitor 4625 event spikes | Automated alert on >5 failures from one IP |
| Geo-block / IP allowlisting | Restricts login sources to trusted IPs |
| Network segmentation | Limits which hosts can reach SMB port 445 |

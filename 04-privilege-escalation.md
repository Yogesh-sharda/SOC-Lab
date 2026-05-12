# Attack 04 — Unauthorized Account Creation & Privilege Escalation

**MITRE ATT&CK:** T1136.001 (Create Local Account) | T1078 (Valid Accounts)
**Tool:** Windows CMD (net user, net localgroup)
**Event IDs:** 4720 (Account Created) | 4732 (User Added to Group)
**Performed on:** Windows 10 Victim Machine

---

## Objective

Simulate an insider threat or post-compromise persistence technique:
create a hidden backdoor account and escalate it to Administrator-level access.
This is one of the most common post-exploitation steps in real breaches.

---

## Commands Used

### Step 1 — Create a New Local User Account

```cmd
net user hacker Password123! /add
```

**What this does:** Creates a new local user account named `hacker` with the specified password.

**Expected output:**
```
The command completed successfully.
```

**This generates:**
- **Windows Event ID 4720** — A user account was created

---

### Step 2 — Add User to Administrators Group

```cmd
net localgroup administrators hacker /add
```

**What this does:** Grants the `hacker` account full local Administrator privileges.

**Expected output:**
```
The command completed successfully.
```

**This generates:**
- **Windows Event ID 4732** — A member was added to a security-enabled local group

---

### Step 3 — Verify the Account

```cmd
net user hacker
net localgroup administrators
```

**Expected output of `net localgroup administrators`:**
```
Alias name     administrators
Members
-------------------------------------------------------------------------------
Administrator
hacker           ← backdoor account now visible
YOG\Domain Admins
```

---

## What Happens in Windows Event Logs

### Event ID 4720 — Account Created

```
Event ID: 4720
Description: A user account was created.

New Account:
  Security ID:   WIN10\hacker
  Account Name:  hacker

Subject (who created it):
  Security ID:   WIN10\Administrator
  Account Name:  Administrator
  Logon ID:      0x3E7
```

### Event ID 4732 — User Added to Privileged Group

```
Event ID: 4732
Description: A member was added to a security-enabled local group.

Member:
  Security ID:  WIN10\hacker
  Account Name: hacker

Group:
  Group Name:   Administrators
  Group Domain: WIN10

Subject:
  Account Name: Administrator
```

---

## Wazuh Detection

**Where to look:**
1. Wazuh Dashboard → Security Events
2. Search: `4720` → finds account creation
3. Search: `4732` → finds group modification
4. Search: `hacker` → finds all events involving the new account

**MITRE tags automatically applied by Wazuh:**
- T1136 — Create Account (Persistence)
- T1078 — Valid Accounts (Defense Evasion, Persistence, Privilege Escalation)

**Why these MITRE techniques?**
- T1136: Attackers create accounts to maintain access even after their initial entry point is removed
- T1078: Using a legitimate account blends in with normal activity and evades many detection rules

---

## Why This Is Dangerous

1. **Persistence** — even if the initial exploit is patched, the backdoor account remains
2. **Stealth** — after creation, the `hacker` account looks like a legitimate local user
3. **Privilege** — with Administrator access, the attacker can disable security software, dump credentials, move laterally
4. **Hard to find without SIEM** — without centralized logging, this account could go unnoticed for months

---

## Cleanup (After Lab — Remove the Test Account)

```cmd
net user hacker /delete
```

**Verify removal:**
```cmd
net user
```

---

## Defensive Notes

| Control | Effect |
|---|---|
| Monitor Event ID 4720 + 4732 | Immediate alert on any new account creation |
| Principle of Least Privilege | Standard users cannot run `net user /add` |
| Windows Privileged Access Workstations (PAWs) | Limits admin commands to designated machines |
| LAPS (Local Admin Password Solution) | Randomizes local admin passwords — limits lateral movement |
| Audit account management policy | Ensures 4720/4732 events are actually logged |
| Regular account audits | Periodic review of all local accounts on endpoints |

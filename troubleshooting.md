# 🔧 Troubleshooting Log

Every real problem encountered during this lab build, documented exactly as it happened.

---

## Problem 1 — VM Ping Blocked (100% Packet Loss)

**Symptom:**
```bash
ping 192.168.122.10
# Request timeout for icmp_seq 0
# 100% packet loss
```

**Root cause:**
Windows Firewall blocks ICMP (ping) by default. This is a Windows security default, not a network issue.

**Fix:**
On Windows Server and Windows 10:
```
Control Panel → Windows Defender Firewall →
Advanced Settings → Inbound Rules →
Find: "File and Printer Sharing (Echo Request - ICMPv4-In)" →
Right-click → Enable Rule
```

Or via PowerShell:
```powershell
Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)"
```

**Result:** Ping replies from Kali within seconds.

---

## Problem 2 — Domain Login Failed After Promotion

**Symptom:**
After the server rebooted as a domain controller, `Administrator` / `[password]` was rejected.

**Root cause:**
Promoting a Windows Server to a domain controller changes the identity model entirely.
The local `Administrator` account becomes the domain administrator.
The correct login format is `DOMAIN\Username`.

**Fix:**
```
Username: YOG\Administrator
Password: [your password]
```

Or click **Other User** at the login screen and type the full domain-prefixed username.

**Lesson:** This is fundamental Active Directory behavior. Domain accounts and local accounts are
different identity contexts. Every enterprise attack involving AD depends on understanding this distinction.

---

## Problem 3 — No Internet After Domain Join on Windows 10

**Symptom:**
Browser showed `DNS_PROBE_FINISHED_BAD_CONFIG`. Internet completely stopped after joining `yog.local`.

**Root cause:**
After domain join, Windows 10 uses the Domain Controller (192.168.122.10) as its only DNS server.
The DC only knows how to resolve internal domain names (`yog.local`, `DC01`, etc.).
It has no configuration to forward external DNS queries to the internet.

**Fix:**
Open IPv4 Properties on Windows 10 and add an alternate DNS server:

| Field | Value |
|---|---|
| Preferred DNS | 192.168.122.10 (DC — keep this for domain) |
| Alternate DNS | 8.8.8.8 (Google — handles internet queries) |

**Result:** Domain authentication works AND internet access works simultaneously.

---

## Problem 4 — Wazuh Agent Showed "Never Connected"

**Symptom:**
Dashboard showed `Win10-Client` registered but status permanently: `Never connected`.

**Root cause:**
AWS Security Group did not have inbound rules for Wazuh's communication ports.
Wazuh agents use port 1514 (log forwarding) and 1515 (registration).
Without these ports open, the agent could register but never send data.

**Diagnosis:**
```powershell
Test-NetConnection <WAZUH_IP> -Port 1514
# TcpTestSucceeded : False  ← confirmed firewall block
```

**Fix:**
AWS Console → EC2 → Security Groups → Edit Inbound Rules → Add:

| Port | Protocol | Source |
|---|---|---|
| 1514 | TCP | 0.0.0.0/0 |
| 1515 | TCP | 0.0.0.0/0 |

After adding rules, restart Wazuh agent on Windows:
```powershell
Restart-Service Wazuh
```

**Verification:**
```powershell
Test-NetConnection <WAZUH_IP> -Port 1514
# TcpTestSucceeded : True  ← fixed
```

---

## Problem 5 — Agent Version Mismatch Error

**Symptom:**
```
ERROR: Agent version must be lower or equal to manager version
```

Tried agent versions 4.14.5 and 4.13.0. Both failed.

**Root cause:**
Wazuh strictly enforces version compatibility. Any agent version newer than the manager is rejected.
The manager was running v4.7.5. Agents 4.13 and 4.14 are newer — both rejected.

**Diagnosis:**
```bash
sudo /var/ossec/bin/wazuh-control info
# WAZUH_VERSION="v4.7.5"
```

**Fix:**
Download the exact matching agent version:
```
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi
```

Install → register → agent connected within seconds.

**Rule:** Always run `wazuh-control info` on the server **before** downloading any agent.
Match the version exactly. Not approximately. Exactly.

---

## Problem 6 — KVM Network Confusion

**Symptom:**
Trying to create a Host-Only network like VMware caused endless confusion.
Custom subnets conflicted with KVM defaults. VMs couldn't communicate.

**Root cause:**
KVM uses `virbr0` (NAT bridge) by default on `192.168.122.0/24`.
This is different from VMware's networking model.

**Fix:**
Stop trying to customize it. Use the default `virbr0` network for all VMs.
KVM's default NAT provides:
- VM-to-VM communication ✅
- Internet access for downloads ✅
- Connectivity to AWS Wazuh server ✅

**Lesson:** Learn your tools' defaults before trying to customize them.

---

## Problem 7 — Lost Wazuh Dashboard Password

**Symptom:**
Cannot login to `https://<EC2-IP>`. Password not saved after installation.

**Root cause:**
Wazuh prints credentials only once at the end of installation. If you don't save them, they're gone.

**Fix:**
SSH into the EC2 instance and reset:
```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  -u admin -p 'YourNewPassword123!'
```

**Prevention:** The first thing you do after Wazuh installs → copy those credentials into a notes file.

---

## Problem 8 — agent-auth.exe PowerShell Syntax Error

**Symptom:**
```powershell
"C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m <IP> -A Win10-Client
# At line:1 char:53
# Unexpected token '-m' in expression or statement.
```

**Root cause:**
In PowerShell, when you quote an executable path, PowerShell treats it as a string expression,
not a command to run. The `-m` flag is then an unexpected token.

**Fix:**
Prefix with `&` (the call operator) which tells PowerShell to execute the string as a command:
```powershell
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m <WAZUH_IP> -A Win10-Client
```

**Result:** Agent registered successfully. `INFO: Valid key received` printed.

---

## Summary Table

| # | Problem | Root Cause | Fix |
|---|---|---|---|
| 1 | Ping blocked | Windows Firewall blocks ICMP | Enable ICMPv4-In firewall rule |
| 2 | Domain login failed | Must use `DOMAIN\Username` format | Use `YOG\Administrator` |
| 3 | No internet after domain join | DC DNS doesn't forward external queries | Add 8.8.8.8 as alternate DNS |
| 4 | Agent never connected | AWS ports 1514/1515 closed | Add inbound rules to EC2 Security Group |
| 5 | Agent version mismatch | Agent newer than manager | Install matching agent v4.7.5 |
| 6 | KVM networking confusion | Different model than VMware | Use default virbr0 network |
| 7 | Lost Wazuh password | Shown only once at install | Reset via wazuh-passwords-tool.sh |
| 8 | agent-auth.exe syntax error | PowerShell needs `&` for quoted exe paths | Use `& "path\exe" -flags` |

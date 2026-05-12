# 📖 Setup Guide — Complete Beginner Walkthrough

This guide documents every step taken to build this lab, including all problems faced and how they were solved.

---

## Prerequisites

| Requirement | Minimum |
|---|---|
| Host RAM | 16 GB (8 GB will struggle) |
| Host Storage | 200 GB free |
| Host OS | Fedora Linux (or any Linux with KVM support) |
| Internet | Required for downloads and AWS connectivity |

---

## Phase 1 — Fedora Host Setup

### Install Virtualization Stack

```bash
sudo dnf install @virtualization
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER
```

> ⚠️ You must **reboot** after adding yourself to the libvirt group — logging out is not enough.

### Verify KVM is Working

```bash
virsh list --all
```

Should return an empty list without errors.

---

## Phase 2 — Create Virtual Machines

Use **Virtual Machine Manager** (virt-manager) as the GUI.

### Download ISOs

| OS | Where to get it |
|---|---|
| Windows Server 2019 | [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/) |
| Windows 10 | [Microsoft Download Page](https://www.microsoft.com/en-us/software-download/windows10) |
| Kali Linux | [kali.org/get-kali](https://www.kali.org/get-kali/) |

### VM Specs

| Machine | RAM | CPU | Storage |
|---|---|---|---|
| Windows Server 2019 | 4 GB | 2 cores | 50 GB |
| Windows 10 | 4 GB | 2 cores | 40 GB |
| Kali Linux | 3 GB | 2 cores | 30 GB |

### Networking

Use the **default** KVM NAT network (`virbr0`) for all VMs.

> Do NOT try to create a custom subnet. KVM's default network (192.168.122.0/24) works perfectly.
> Fighting the defaults costs hours. Working with them costs nothing.

---

## Phase 3 — Windows Server 2019 Setup

### Set Static IP

`Control Panel → Network and Internet → Network Connections → Ethernet → IPv4 Properties`

| Field | Value |
|---|---|
| IP Address | 192.168.122.10 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 192.168.122.1 |
| Preferred DNS | 192.168.122.10 (itself) |

### Install Active Directory Domain Services

1. Open **Server Manager**
2. Click **Add Roles and Features**
3. Select: **Active Directory Domain Services**
4. Click through → **Install**
5. Wait for completion

### Promote to Domain Controller

After installation, click the **notification flag** → **Promote this server to a domain controller**

| Setting | Value |
|---|---|
| Deployment | Add a new forest |
| Root domain name | yog.local |
| DSRM password | Set something you'll remember |
| DNS Server | ✅ Enabled (leave checked) |
| Global Catalog | ✅ Enabled (leave checked) |

Click through → **Install** → Server will reboot automatically.

### Post-Reboot Login

After reboot, login with domain-prefixed credentials:

```
Username: YOG\Administrator
Password: [your password]
```

> ❌ Common mistake: typing just `Administrator` fails after domain promotion.
> The account is now domain-scoped: `YOG\Administrator`.

---

## Phase 4 — Windows 10 Setup

### Set Network Settings

| Field | Value |
|---|---|
| IP Address | 192.168.122.20 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 192.168.122.1 |
| Preferred DNS | 192.168.122.10 |
| Alternate DNS | 8.8.8.8 |

> Without the alternate DNS (8.8.8.8), internet will stop working after joining the domain
> because the DC's DNS only resolves internal names.

### Join Domain

```
Win + R → sysdm.cpl → Computer Name → Change → Domain: yog.local
```

Enter domain admin credentials when prompted. Restart.

### Verify Connectivity from Kali

```bash
ping 192.168.122.10   # DC01
ping 192.168.122.20   # WIN10
```

If ping fails → disable ICMP block in Windows Firewall:
`Windows Defender Firewall → Allow an app → File and Printer Sharing (Echo Request - ICMPv4-In)`

---

## Phase 5 — Wazuh on AWS EC2

### Launch EC2 Instance

- OS: Ubuntu 22.04
- Instance type: t3.medium (minimum) or t2.medium
- Storage: 50 GB
- Security Group: allow ports 22, 443, 1514, 1515

### Install Wazuh

```bash
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>

sudo apt update && sudo apt upgrade -y

curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
chmod +x wazuh-install.sh
sudo ./wazuh-install.sh -a
```

> ⚠️ **Save the credentials printed at the end of installation.** They are shown exactly once.

### Access Dashboard

Open browser: `https://<EC2-PUBLIC-IP>`
Accept the SSL certificate warning → login with saved credentials.

### If You Lost the Password

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  -u admin -p 'YourNewPassword123!'
```

### Check Manager Version

```bash
sudo /var/ossec/bin/wazuh-control info
# WAZUH_VERSION="v4.7.5"
```

> ⚠️ **Note this version.** Your Windows agent must match it exactly.

---

## Phase 6 — Wazuh Agent on Windows 10

### Download the Right Version

Download agent version **4.7.5** (must match your manager):
`https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi`

### Verify Port Connectivity First

```powershell
Test-NetConnection <WAZUH_EC2_IP> -Port 1514
# TcpTestSucceeded : True
```

If False → check AWS Security Group inbound rules.

### Install Agent

Run the MSI → set Manager Address to your EC2 public IP → install.

### Register Agent via PowerShell (Admin)

```powershell
Stop-Service Wazuh

# Clear stale key if re-registering
Remove-Item "C:\Program Files (x86)\ossec-agent\client.keys" -ErrorAction SilentlyContinue

# Register
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m <WAZUH_EC2_IP> -A Win10-Client

Start-Service Wazuh
```

### Verify in Dashboard

Open Wazuh → **Agents** → `Win10-Client` should show **Active**.

---

## Phase 7 — Verify Log Collection

On Windows 10 PowerShell:

```powershell
# Create a test event
net user testuser Pass123! /add
net user testuser /delete
```

In Wazuh Security Events, search `4720` — the account creation event should appear within 30–60 seconds.

Your lab is ready.

# 📸 Screenshots

Add your screenshots to this folder and update the table below.

> Tip: Before uploading — blur or redact all public IPs, AWS identifiers,
> passwords, and account names in your screenshots.

---

## Screenshot Index

| Filename | Description | Phase |
|---|---|---|
| `01-vm-topology.png` | Virtual Machine Manager showing all 3 VMs running | Setup |
| `02-ad-install.png` | Server Manager — AD DS role installation | Phase 2 |
| `03-domain-promotion.png` | Promote server to domain controller wizard | Phase 2 |
| `04-domain-join-success.png` | "Welcome to the yog.local domain" confirmation | Phase 3 |
| `05-kali-ping-dc.png` | Kali successfully pinging Domain Controller | Phase 3 |
| `06-kali-ping-win10.png` | Kali successfully pinging Windows 10 victim | Phase 3 |
| `07-wazuh-install.png` | Wazuh all-in-one installer running on EC2 | Phase 4 |
| `08-wazuh-dashboard.png` | Wazuh dashboard home after first login | Phase 4 |
| `09-agent-active.png` | Win10-Client showing Active status in Wazuh | Phase 5 |
| `10-security-events.png` | Wazuh Security Events stream — 480+ events | Phase 5 |
| `11-nmap-scan.png` | Nmap host discovery from Kali | Phase 6 |
| `12-smb-enum.png` | CrackMapExec SMB enumeration output | Phase 6 |
| `13-brute-force-kali.png` | crackmapexec brute force command from Kali | Phase 6 |
| `14-failed-login-alert.png` | Wazuh alert — Event ID 4625 failed logon | Phase 6 |
| `15-user-creation.png` | net user hacker command on Windows 10 | Phase 6 |
| `16-privilege-escalation-alert.png` | Wazuh alert — Event ID 4720 + 4732 | Phase 6 |
| `17-powershell-alert.png` | Wazuh alert — Event ID 4104 PowerShell | Phase 6 |
| `18-mitre-mapping.png` | MITRE ATT&CK technique tags in Wazuh dashboard | Phase 7 |

---

## How to Add Screenshots

1. Take screenshot inside your VM (use `Print Screen` or `scrot` on Linux)
2. Crop to show only the relevant area
3. **Redact sensitive information** (blur IPs, passwords, AWS account ID)
4. Save as `.png` with the filename from the table above
5. Drop the file into this `screenshots/` folder

---

## Recommended Tool for Redacting

On Linux:
```bash
# Install GIMP for blurring sensitive areas
sudo dnf install gimp
```

On Windows: use **Paint 3D** or **Snipping Tool** to draw rectangles over sensitive data.

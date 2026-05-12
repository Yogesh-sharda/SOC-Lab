# Attack 01 — Network Reconnaissance

**MITRE ATT&CK:** T1046 (Network Service Discovery) | T1018 (Remote System Discovery)
**Tool:** Nmap
**Performed from:** Kali Linux

---

## Objective

Simulate an attacker enumerating the internal network from a compromised workstation
to identify live hosts, open ports, and running services.

---

## Commands Used

### Step 1 — Discover All Live Hosts on Subnet

```bash
nmap -sn 192.168.122.0/24
```

**What this does:** Sends ICMP ping and ARP probes across the entire /24 subnet
to find which machines are online.

**Output:**
```
Nmap scan report for 192.168.122.1   → KVM Gateway
Nmap scan report for 192.168.122.10  → Windows Server 2019 (DC01)
Nmap scan report for 192.168.122.20  → Windows 10 (WIN10)
Nmap scan report for 192.168.122.67  → Kali Linux (self)
```

---

### Step 2 — Full Port Scan on Victim Endpoint

```bash
nmap -sS 192.168.122.20
```

**What this does:** SYN stealth scan — sends SYN packets without completing the TCP handshake.
Faster and less noisy than a full connect scan.

**Output (expected):**
```
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

---

### Step 3 — Service Version + OS Detection

```bash
nmap -sV -sC 192.168.122.20
```

**What this does:**
- `-sV` — detects software versions running on open ports
- `-sC` — runs default Nmap scripts for additional enumeration

---

### Step 4 — Aggressive Scan on Domain Controller

```bash
nmap -A 192.168.122.10
```

**What this does:** Combines OS detection, service detection, script scanning, and traceroute.
Useful for fingerprinting the domain controller.

**Key ports found on DC:**
```
PORT    STATE  SERVICE
53/tcp  open   domain     (DNS)
88/tcp  open   kerberos   (Kerberos authentication)
135/tcp open   msrpc
139/tcp open   netbios-ssn
389/tcp open   ldap       (Active Directory LDAP)
445/tcp open   microsoft-ds (SMB)
636/tcp open   tcpwrapped (LDAPS)
3268/tcp open  ldap       (Global Catalog)
```

---

## Wazuh Detection

Wazuh detects connection sweeps as multiple authentication/connection attempts from the same source.
To investigate in Wazuh:

1. Open Wazuh Dashboard → Security Events
2. Search for: `data.srcip:192.168.122.67` (Kali IP)
3. Look for rapid sequential connection events
4. Check timestamps — scan generates many events in a short window

> For deeper detection, Sysmon's **Event ID 3** (Network connection) provides process-level
> context — which process made the connection, to which IP and port.

---

## What an Attacker Learns

From this scan, an attacker now knows:
- DC01 (192.168.122.10) is a domain controller with Kerberos, LDAP, and SMB
- WIN10 (192.168.122.20) is an endpoint with SMB open (potential lateral movement target)
- No firewall is blocking internal reconnaissance

---

## Defensive Notes

| Control | Effect |
|---|---|
| Host-based firewall rules | Limits which ports are reachable from internal hosts |
| Network segmentation (VLANs) | Prevents attacker from scanning entire subnet |
| Sysmon Event ID 3 monitoring | Provides process-level network connection logging |
| IDS/IPS (Snort, Suricata) | Can detect nmap scan patterns via signature matching |

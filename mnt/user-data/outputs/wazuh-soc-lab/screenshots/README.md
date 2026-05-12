<img width="1497" height="902" alt="Screenshot_20260513_010751" src="https://github.com/user-attachments/assets/18b08d5a-7137-4d82-946c-7b7784fb4f91" />
## Screenshot Index

| Filename | Description | Phase |
|---|---|---|
| <img width="1497" height="902" alt="Screenshot_20260513_010637" src="https://github.com/user-attachments/assets/b8943526-db82-47c7-8918-bdae872745c3" /> | Kali pinging DC (192.168.122.10) — 8 packets, 0% loss, avg 0.480ms; and pinging Win10 (192.168.122.20) — 6 packets, 0% loss | Phase 3 |
| <img width="1497" height="902" alt="Screenshot_20260513_010751" src="https://github.com/user-attachments/assets/176f0728-a98b-4c54-9aa3-237d240041f5" />
 | Nmap `-sV` scan of 192.168.122.20 — open: 135/tcp msrpc, 139/tcp netbios-ssn, 445/tcp microsoft-ds; OS: Windows; MAC: 52:54:00:2F:23:AB | Phase 6 |
| <img width="1497" height="902" alt="Screenshot_20260513_010822" src="https://github.com/user-attachments/assets/0ceef91f-babf-48fb-8d41-6731a72f9c9c" />
 | CrackMapExec SMB enum — DESKTOP-1KOOU1K, Win10/Server 2019 Build 19041 x64, domain: yog.local, SMBv1: False, signing: False | Phase 6 |
| <img width="1497" height="902" alt="Screenshot_20260513_010915" src="https://github.com/user-attachments/assets/dc41f4ba-f505-4e91-94cc-dc6a97eaae76" />
 | CrackMapExec brute force with `-u users.txt -p passwords.txt` — result: STATUS_LOGON_FAILURE | Phase 6 |
| <img width="1023" height="763" alt="Screenshot_20260513_011117" src="https://github.com/user-attachments/assets/9e95ddd9-ce3d-4b35-9cc1-a6aa8434182b" />
 | PowerShell (Admin): `Invoke-WebRequest http://example.com` — StatusCode 200, Content-Type text/html, CF-RAY header visible | Phase 6 |
| <img width="1015" height="770" alt="Screenshot_20260513_011246" src="https://github.com/user-attachments/assets/b5b34c58-d5a6-4461-9855-a4003796d127" />
 | PowerShell (Admin): `net user hacker Password123! /add` — "The command completed successfully" | Phase 6 |
| `<img width="1029" height="769" alt="Screenshot_20260513_011401" src="https://github.com/user-attachments/assets/0306516c-5f9d-44b1-a604-c242783a46b7" />
 | PowerShell (Admin): `net localgroup administrations hacker /add` — System error 1376, group does not exist (typo: should be `administrators`) | Phase 6 |
| <img width="1910" height="943" alt="Screenshot_20260513_011553" src="https://github.com/user-attachments/assets/f7741424-cfa7-4578-a136-645f64d91a7a" />
 | Wazuh Security Events list — T1078 logon successes/failures, T1098 user account created (rule 60109), T1484 users group changed (rule 60170), T1531 logon failures (rule 60122) | Phase 6–7 |
| <img width="1905" height="947" alt="Screenshot_20260513_011619" src="https://github.com/user-attachments/assets/127c6332-eaea-4bfa-80fb-302423c017d5" />
 | Wazuh Security Events dashboard — 68 total alerts, 0 critical, 4 auth failures, 24 auth successes; alert evolution and top 5 pie charts | Phase 5–7 |

---

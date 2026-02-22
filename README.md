# 🧠 HackTheBox Writeups

---

## 📖 About

This repository documents my journey through HackTheBox machines. Each writeup follows a structured format:

- **TL;DR** — A one-paragraph chained summary of the full exploit path
- **Enumeration** — Nmap results and service discovery
- **Exploitation** — Step-by-step exploitation with annotated commands
- **Privilege Escalation** — Local privesc to root/SYSTEM with full explanation
- **Key Takeaways** — Core security lessons from each machine

These aren't just "here's the command, type it" writeups — each step explains **why** the technique works, not just **what** to type.

---

## 🗺️ Machine Index

| # | Machine | OS | Difficulty | Concepts Covered |
|---|---------|-----|------------|-----------------|
| 1 | [Active](./Active/README.md) | 🪟 Windows | 🟢 Easy | GPP cpassword decryption, Kerberoasting |
| 2 | [Administrator](./Administrator/README.md) | 🪟 Windows | 🟡 Medium | BloodHound ACL abuse, Password Safe cracking, DCSync |
| 3 | [Artic](./Artic/README.md) | 🪟 Windows | 🟢 Easy | ColdFusion file upload, JuicyPotato |
| 4 | [Blackfield](./Blackfield/README.md) | 🪟 Windows | 🔴 Hard | AS-REP Roasting, ForceChangePassword, SeBackupPrivilege, NTDS.dit |
| 5 | [Cascade](./Cascade/README.md) | 🪟 Windows | 🟡 Medium | LDAP enumeration, AD Recycle Bin abuse, .NET reverse engineering |
| 6 | [Codify](./Codify/README.md) | 🐧 Linux | 🟢 Easy | Node.js vm2 sandbox escape, bash script timing attack |
| 7 | [Flight](./Flight/README.md) | 🪟 Windows | 🔴 Hard | UNC path injection, NTLM theft, SeImpersonatePrivilege, GodPotato |
| 8 | [Forest](./Forest/README.md) | 🪟 Windows | 🟢 Easy | AS-REP Roasting, Account Operators, WriteDacl, DCSync |
| 9 | [Jeevs](./Jeevs/README.md) | 🪟 Windows | 🟡 Medium | Jenkins Groovy script RCE, JuicyPotato |
| 10 | [Keeper](./Keeper/README.md) | 🐧 Linux | 🟢 Easy | Request Tracker default creds, KeePass memory dump extraction |
| 11 | [Knife](./Knife/README.md) | 🐧 Linux | 🟢 Easy | PHP 8.1.0-dev backdoor RCE, knife sudo GTFOBins |
| 12 | [Lame](./Lame/README.md) | 🐧 Linux | 🟢 Easy | Samba username map script (CVE-2007-2447) |
| 13 | [Love](./Love/README.md) | 🪟 Windows | 🟢 Easy | SQL injection auth bypass, AlwaysInstallElevated |
| 14 | [Mailing](./Mailing/README.md) | 🪟 Windows | 🟢 Easy | Directory traversal, CVE-2024-21413 Outlook Moniker Link RCE |
| 15 | [Mounteverde](./Mounteverde/README.md) | 🪟 Windows | 🟡 Medium | Password spraying, Azure AD Connect credential extraction |
| 16 | [NIbbles](./NIbbles/README.md) | 🐧 Linux | 🟢 Easy | Nibbleblog file upload RCE, sudo cron script abuse |
| 17 | [Netmon](./Netmon/README.md) | 🪟 Windows | 🟢 Easy | Anonymous FTP, PRTG Network Monitor RCE |
| 18 | [Pilgrimage](./Pilgrimage/README.md) | 🐧 Linux | 🟢 Easy | Exposed .git, ImageMagick LFI CVE-2022-44268, binwalk RCE CVE-2022-4510 |
| 19 | [Querier](./Querier/README.md) | 🪟 Windows | 🟡 Medium | MSSQL NTLM hash capture via Responder, GPP password extraction |
| 20 | [Return](./Return/README.md) | 🪟 Windows | 🟢 Easy | Printer admin LDAP credential capture, Server Operators privesc |
| 21 | [Sauna](./Sauna/README.md) | 🪟 Windows | 🟢 Easy | Username enumeration, AS-REP Roasting, DCSync |
| 22 | [ServMon](./ServMon/README.md) | 🪟 Windows | 🟢 Easy | Anonymous FTP, NVMS-1000 directory traversal, NSClient++ RCE |
| 23 | [Sniper](./Sniper/README.md) | 🪟 Windows | 🟡 Medium | PHP LFI via SMB UNC path, RFI to RCE, CHM file malware |
| 24 | [Solidstate](./Solidstate/README.md) | 🐧 Linux | 🟡 Medium | JAMES admin console default creds, POP3 email dump, writable cronjob |
| 25 | [Sunday](./Sunday/README.md) | 🐧 Linux | 🟢 Easy | Finger service user enum, shadow.backup hash, sudo wget overwrite |
| 26 | [Timelapse](./Timelapse/README.md) | 🪟 Windows | 🟢 Easy | SMB ZIP cracking, PFX certificate abuse, LAPS password extraction |
| 27 | [cozyhosting](./cozyhosting/README.md) | 🐧 Linux | 🟢 Easy | Spring Boot Actuator session hijack, command injection, GTFOBins SSH |
| 28 | [remote](./remote/README.md) | 🪟 Windows | 🟢 Easy | NFS mount, Umbraco CMS credential extraction, authenticated RCE |

---

## 🧰 Tools & Techniques Featured

`nmap` · `gobuster` · `kerbrute` · `impacket` · `evil-winrm` · `bloodhound` · `responder` · `hashcat` · `john` · `crackmapexec` · `smbclient` · `pypykatz` · `pspy` · `msfvenom` · `chisel` · `ligolo-ng` · `netcat`

---

## ⚠️ Disclaimer

All machines documented here are retired HackTheBox labs. This content is purely for **educational purposes** to help others learn penetration testing concepts and defensive security thinking.

---

## 🔗 Connect

If you find these helpful, feel free to ⭐ the repository!


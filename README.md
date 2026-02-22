<h1 align="center">
  <br>
  🧠 HackTheBox Writeups 🧠
  <br>
</h1>

<h4 align="center">A growing collection of detailed walkthroughs for HackTheBox machines.</h4>

<p align="center">
  <a href="#about">About</a> •
  <a href="#contents">Contents</a> •
  <a href="#highlights">Highlights</a> •
  <a href="#disclaimer">Disclaimer</a>
</p>

---

## 📖 About

This repository is a personal knowledge base documenting my progress through **HackTheBox** machines. Each writeup walks through the full attack chain — initial enumeration, exploitation, and privilege escalation — with real commands, inline screenshots, and explanations of *why* each technique works, not just what to type.

Whether you're studying for the OSCP, preparing for a CTF, or just looking to sharpen your pentesting methodology, these writeups cover practical techniques across Linux, Windows, and Active Directory environments.

---

## 📂 Contents

Writeups are organized by operating system:

### 🐧 Linux
From exploiting vulnerable web applications and outdated services to abusing misconfigurations like world-writable cron jobs, SUID binaries, and weak sudo rules.

### 🪟 Windows
Exploiting IIS servers, abusing service account privileges, leveraging SMB misconfigurations, and classic Windows privilege escalation via `SeImpersonatePrivilege` (PrintSpoofer/GodPotato/JuicyPotato) and `AlwaysInstallElevated`.

### 🏢 Active Directory
Domain enumeration with BloodHound, AS-REP Roasting, Kerberoasting, DCSync, ACL abuse (GenericAll, WriteDacl, ForceChangePassword), and credential extraction from Azure AD Connect and LAPS.

---

## 🎯 Highlights & Methodologies

Key techniques and vulnerabilities covered across these machines:

- **Web Exploitation:** LFI/RFI, SQL injection auth bypass, SSRF, file upload bypasses, CMS vulnerabilities (Umbraco, Nibbleblog, ColdFusion).
- **Network Attacks:** Anonymous FTP/SMB enumeration, UNC path injection, NTLM hash capture via Responder.
- **Credential Attacks:** GPP cpassword decryption, Kerberoasting, AS-REP Roasting, hash cracking with Hashcat/John, Pass-the-Hash.
- **Privilege Escalation (Linux):** Writable cronjobs, SUID abuse, sudo GTFOBins, pspy process monitoring.
- **Privilege Escalation (Windows):** Token impersonation, SeBackupPrivilege + NTDS.dit extraction, AlwaysInstallElevated, password manager cracking.
- **Active Directory:** BloodHound ACL chaining, DCSync, Azure AD Connect credential extraction, Password Safe abuse.

---

## ⚠️ Disclaimer

All information and writeups in this repository are for **educational purposes only**. All machines documented here are **retired** HackTheBox labs. Do not use these techniques on systems you do not own or do not have explicit permission to test.

---

<p align="center">
  <i>"The only truly secure system is one that is powered off, cast in a block of concrete and sealed in a lead-lined room with armed guards."</i><br>— Eugene H. Spafford
</p>

<h1 align="center">
  <br>
  🧠 HackTheBox Writeups 🧠
  <br>
</h1>

<h4 align="center">A growing collection of detailed walkthroughs for HackTheBox machines.</h4>

<p align="center">
  <a href="#about">About</a> •
  <a href="#contents">Contents</a> •
  <a href="#highlights">Highlights</a>
</p>

---

## 📖 About

This repository is a personal knowledge base documenting my progress through **HackTheBox** machines. Each writeup walks through the full attack chain — initial enumeration, exploitation, and privilege escalation — with real commands, inline screenshots, and explanations of *why* each technique works, not just what to type.

Whether you're studying for the OSCP, preparing for a CTF, or just looking to sharpen your pentesting methodology, these writeups cover practical techniques across Linux, Windows, and Active Directory environments.

---

## 📂 Contents

Each machine has its own folder containing a full `README.md` writeup and an `images/` directory with referenced screenshots. Every writeup follows a consistent structure:

- **TL;DR** — One-paragraph summary of the entire attack path
- **Enumeration** — Port scans, service fingerprinting, and discovery
- **Exploitation** — Step-by-step with commands and explanations
- **Privilege Escalation** — Local privesc to root or SYSTEM
- **Key Takeaways** — Security lessons from the box

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

<p align="center">
  <i>"The only truly secure system is one that is powered off, cast in a block of concrete and sealed in a lead-lined room with armed guards."</i><br>— Eugene H. Spafford
</p>

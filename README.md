# 🧠 HackTheBox Writeups

---

## 📖 About

This repo is a growing collection of my HackTheBox machine walkthroughs. Each writeup breaks down the full attack chain — from the initial nmap scan all the way to root — with real commands, annotated screenshots, and explanations of *why* each technique works, not just *what* to type.

The goal is to document what I learn in a way that's useful to review later and actually readable for anyone following along.

---

## 📂 Structure

Each machine gets its own folder with a `README.md` writeup:

```
/MachineName
  └── README.md     ← full walkthrough
  └── images/       ← screenshots referenced inline
```

Every writeup follows the same layout:
- **TL;DR** — one-paragraph chained summary of the entire attack path
- **Enumeration** — port scan results and service fingerprinting
- **Exploitation** — step-by-step with commands and explanations
- **Privilege Escalation** — local privesc to root or SYSTEM
- **Key Takeaways** — the security lessons from the box

---

## 🧰 Common Tools & Techniques

`nmap` · `gobuster` · `kerbrute` · `impacket` · `evil-winrm` · `bloodhound` · `responder` · `hashcat` · `john` · `crackmapexec` · `smbclient` · `pypykatz` · `pspy` · `msfvenom` · `chisel` · `ligolo-ng` · `netcat` · and more

---

## ⚠️ Disclaimer

All machines documented here are **retired** HackTheBox labs. This content is purely for educational purposes to help others understand penetration testing concepts and defensive security thinking.

---

*More writeups added regularly as I work through the platform.* ⚡

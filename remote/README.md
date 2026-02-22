# remote — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Easy
**OS:** Windows

---

## TL;DR

Enumerating the NFS service (Port 2049) reveals an exposed `site_backups` share → Mounting the share locally reveals a backup of an Umbraco CMS installation → Running `strings` on the Umbraco database file (`Umbraco.sdf`) extracts a SHA1 password hash for the `admin` user → Cracking the hash yields the `admin` password (`baconandcheese`) → Accessing the Umbraco CMS (v7.12.4) dashboard allows executing an authenticated Remote Code Execution (RCE) exploit to gain an initial foothold.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=9018 10.10.10.180
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | Microsoft ftpd (Anonymous Login Allowed) |
| 80 | HTTP | Microsoft HTTPAPI httpd 2.0 |
| 111 | rpcbind| 2-4 (RPC) |
| 135 | RPC | Microsoft Windows RPC |
| 139 | NetBIOS| Microsoft Windows netbios-ssn |
| 445 | SMB | microsoft-ds |
| 2049 | nfs | nlockmgr |
| 5985 | WinRM | Microsoft HTTPAPI httpd 2.0 |

The target is a Windows machine running several services, notably including NFS (Network File System) on port 2049 and an IIS web server on port 80 hosting a site for "Acme Widgets".

We check the anonymous FTP service but find nothing immediately actionable beyond a generic banner. 

We shift our focus to the Network File System (NFS) service. NFS is typically associated with Linux, but it can be enabled on Windows Server. We use `nfs-ls` or `showmount` to list available NFS shares on the target:

```bash
showmount -e 10.10.10.180
# OR
nfs-ls -D nfs://10.10.10.180
```

The output reveals an exported share named: `/site_backups`.

---

## Exploitation — NFS Mounting & Umbraco CMS RCE

We mount the exposed `/site_backups` share to our local Kali Linux machine to explore its contents:

```bash
mkdir /mnt/nfs
sudo mount -t nfs 10.10.10.180:/site_backups /mnt/nfs
```

Inside the mounted directory, we find a complete backup of a web application. Inspecting the files reveals the application is built on **Umbraco CMS**, an open-source .NET content management system.

Umbraco stores its configuration and credentials in a local SQL Server Compact database file named `Umbraco.sdf`, typically located in the `App_Data` directory.

We navigate to `/mnt/nfs/App_Data/` and inspect `Umbraco.sdf`. Since it's a binary database file, simply `cat`ing it will output gibberish. Instead, we use the `strings` utility and `grep` for keywords like "password", "admin", or look for hash patterns:

```bash
strings Umbraco.sdf | grep -i admin
```

Among the output, we discover two blocks of credential information. One contains a SHA1 hash for the `admin@htb.local` user:

```json
{"hashAlgorithm":"SHA1"}
admin baconandcheese b8be16afba8c314ad33d812f22a04991b90e2aaa
```

We extract the SHA1 string (`b8be16afba8c314ad33d812f22a04991b90e2aaa`) and crack it offline using Hashcat with the RockYou wordlist:

```bash
hashcat -m 100 admin.hash /usr/share/wordlists/rockyou.txt
```

The hash cracks successfully, yielding the password: `baconandcheese`.

We now have valid credentials for the Umbraco admin portal:
`admin@htb.local : baconandcheese`

We navigate to the Umbraco administrative login page on the live web server (port 80), typically located at `http://10.10.10.180/umbraco/`.

We authenticate using the recovered credentials.
Once logged in, we check the Umbraco version number in the dashboard (usually available in the "Help" or "About" menus). The version is **Umbraco CMS 7.12.4**.

A quick search on Exploit-DB reveals that Umbraco CMS versions prior to 7.12.4 are vulnerable to an authenticated Remote Code Execution exploit (CVE-2018-16763). This vulnerability exists in the back-office API and allows manipulating the `template` system to execute arbitrary C# code.

We can use a publicly available Python exploit script for CVE-2018-16763, providing it with our target URL, email, and password. The script automates logging in and injecting a payload (like downloading and executing a Netcat binary or spawning an interactive command shell).

```bash
python3 exploit.py -u admin@htb.local -p baconandcheese -i http://10.10.10.180 -c "powershell.exe -e <base64_reverse_shell>"
```

The exploit triggers successfully, executing our payload and granting us an initial reverse shell on the target system.

We have user access.

---

## Key Takeaways

- **Exposed Backups:** Leaving live backups on exposed, anonymous NFS or SMB shares is a critical vulnerability. Even if the live site is secure, backups provide a full blueprint of the application and often contain live credentials or keys.
- **Embedded Databases:** File-based databases like SQLite or SQL Server Compact (`.sdf`) are highly susceptible to simple string extraction attacks if physical access (or backup access) is gained.
- **CMS RCE via Templates:** Content Management Systems (like Umbraco, WordPress, or Joomla) inherently possess functionality to modify templates (which are often executable code). If an administrative account is compromised, pivoting from CMS Admin to full RCE is usually guaranteed.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
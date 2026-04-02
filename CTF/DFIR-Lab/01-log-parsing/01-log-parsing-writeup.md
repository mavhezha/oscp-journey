# Challenge 01 — Log Parsing

## Overview

| Field | Details |
|-------|---------|
| Challenge | 01 — Log Parsing |
| Category | Digital Forensics & Incident Response |
| Difficulty | Easy |
| Date | 2026-03-25 |
| Status | ✅ Completed — 6/6 flags captured |

## Scenario

A Linux web server was compromised on **November 14, 2025**. Multiple log files were preserved for analysis. The goal was to reconstruct the full attack timeline by examining six log files:

| File | Description |
|------|-------------|
| `auth.log` | SSH authentication and authorization events |
| `apache-access.log` | Apache HTTP server access log |
| `apache-error.log` | Apache HTTP server error log |
| `syslog` | General system log |
| `kern.log` | Kernel messages |
| `suspicious-cron.log` | Exported crontab entries |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `grep` | Search for patterns in log files |
| `awk` | Extract specific fields from log lines |
| `sort` | Sort output alphabetically or numerically |
| `uniq -c` | Count unique occurrences |
| `cat` | Read file contents |
| `tail` | Show last N lines of output |

---

## Methodology

The hints in the challenge README guided the approach:
- Start with `auth.log` to find the initial access vector
- Use `grep`, `awk`, `sort` and `uniq` to filter noise
- Look for patterns: brute-force = many failures then success
- Cross-reference timestamps across different log files
- Check for unusual user creation and privilege escalation

---

## Flag 1.1 — Brute Force IP

**Question:** What IP address conducted the brute-force attack?

**Command:**
```bash
sudo grep "Failed password" auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head
```

**Explanation:**
- `grep "Failed password"` — extracts all failed SSH login attempts
- `awk '{print $11}'` — extracts the 11th field (the IP address)
- `sort | uniq -c` — counts occurrences of each unique IP
- `sort -rn` — sorts by count, highest first
- `head` — shows top 10 results

**Output:**
```
83 198.51.100.47
 1 94.238.212.16
 1 94.194.162.161
...
```

**Finding:** `198.51.100.47` had **83 failed attempts** — a clear brute-force pattern. All other IPs had only 1 failure which is normal noise.

**Flag:** `FLAG{198.51.100.47}`

---

## Flag 1.2 — Web Shell Filename

**Question:** What was the filename of the web shell uploaded?

**Command:**
```bash
sudo grep -i "\.php" apache-access.log | grep "POST"
```

**Explanation:**
- `grep -i "\.php"` — finds all requests to PHP files (`-i` = case insensitive, `\.` escapes the dot)
- `| grep "POST"` — filters to POST requests only (file uploads use POST)

**Output:**
```
198.51.100.47 - - [14/Nov/2025:03:30:00 +0000] "POST /uploads/shell.php HTTP/1.1" 200 1337 "-" "Mozilla/5.0 (compatible; beacon/2.0)"
```

**Finding:** The attacker uploaded `shell.php` to `/uploads/` at `03:30:00`. The user agent `beacon/2.0` indicates a **Cobalt Strike beacon**. Response code `200` confirms the upload succeeded.

**Flag:** `FLAG{shell.php}`

---

## Flag 1.3 — Persistence Username

**Question:** What username did the attacker create for persistence?

**Command:**
```bash
sudo grep -i "useradd\|adduser\|new user" auth.log
```

**Explanation:**
- Searches `auth.log` for user creation events
- `\|` means OR — searches for any of the three patterns simultaneously

**Output:**
```
Nov 14 03:33:00 webserver useradd[20100]: new user: name=svc-backup, UID=1002, GID=1002, home=/home/svc-backup, shell=/bin/bash, from=pts/0
```

**Finding:** User `svc-backup` was created at `03:33:00` — just 3 minutes after the web shell upload. Key observations:
- Named to look like a legitimate service account
- Has `/bin/bash` shell — not a real service account
- Created from `pts/0` — an interactive terminal session

**Flag:** `FLAG{svc-backup}`

---

## Flag 1.4 — C2 Callback IP and Port

**Question:** What is the C2 callback IP and port?

**Approach:** This required multiple steps and pivoting between log files.

**Step 1 — Find svc-backup activity:**
```bash
sudo grep -i "svc-backup\|beacon\|connect\|curl\|wget" auth.log syslog
```

This revealed crontab activity at `03:45:00`.

**Step 2 — Search syslog during the crontab edit window:**
```bash
sudo grep "Nov 14 03:4" syslog
```

**Output:**
```
Nov 14 03:45:00 webserver crontab[21000]: (svc-backup) BEGIN EDIT (svc-backup)
Nov 14 03:45:01 webserver crontab[21000]: (svc-backup) REPLACE (svc-backup)
Nov 14 03:45:01 webserver crontab[21000]: (svc-backup) END EDIT (svc-backup)
Nov 14 03:47:00 webserver kernel: [UFW ALLOW] IN= OUT=eth0 SRC=10.0.1.50 DST=203.0.113.99 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=54321 DF PROTO=TCP SPT=48912 DPT=4444 WINDOW=64240 RES=0x00 SYN URGP=0
```

**Finding:** The UFW firewall log shows an outbound TCP connection:
- `SRC=10.0.1.50` — the compromised server
- `DST=203.0.113.99` — the attacker's C2 server
- `DPT=4444` — destination port 4444 (classic reverse shell port)

The crontab edit triggered a beacon.sh script which called back to the C2 server.

**Flag:** `FLAG{203.0.113.99:4444}`

---

## Flag 1.5 — Kernel Module

**Question:** What kernel module was loaded by the attacker?

**Command:**
```bash
sudo grep -i "module\|insmod\|modprobe" kern.log
```

**Explanation:**
- Searches `kern.log` for kernel module loading events
- `insmod` and `modprobe` are commands used to load kernel modules

**Output:**
```
Nov 14 03:50:00 webserver kernel: [98765.432100] rootkit_mod: loading out-of-tree module taints kernel.
Nov 14 03:50:00 webserver kernel: [98765.432200] rootkit_mod: module verification failed: signature and/or required key missing - tainting kernel
Nov 14 03:50:01 webserver kernel: [98765.433000] rootkit_mod: module loaded successfully, hiding PID 31337
```

**Finding:** The attacker loaded `rootkit_mod` — a malicious kernel module. Key observations:
- **"out-of-tree"** — not an official Linux kernel module
- **"verification failed"** — unsigned, not from a trusted source
- **"hiding PID 31337"** — actively hiding the attacker's process from process listings

**Flag:** `FLAG{rootkit_mod}`

---

## Flag 1.6 — Cron Persistence Path

**Question:** What is the full path of the persistence script in cron?

**Command:**
```bash
sudo cat suspicious-cron.log
```

**Explanation:**
- Simply reads the exported crontab file preserved as evidence

**Output:**
```
# Crontab export for user svc-backup
# Generated: Nov 14 03:45:01 2025
# Host: webserver

# m h  dom mon dow   command
*/5 * * * * /tmp/.hidden/beacon.sh
```

**Finding:** The attacker added `/tmp/.hidden/beacon.sh` to run every 5 minutes. Key observations:
- Stored in `/tmp` — writable by all users, often overlooked
- `.hidden` folder — dotfiles are hidden from normal `ls` output
- `beacon.sh` — matches the Cobalt Strike beacon user agent seen earlier
- `*/5 * * * *` — runs every 5 minutes for persistent C2 access

**Flag:** `FLAG{/tmp/.hidden/beacon.sh}`

---

## Full Attack Timeline

```
02:55:00 → Attacker scans with DirBuster (web directory brute force)
03:30:00 → Web shell uploaded → /uploads/shell.php
03:31:00 → Web shell tested (id, whoami, cat /etc/passwd)
03:33:00 → Backdoor user created → svc-backup
03:34:00 → SSH key added to /home/svc-backup/.ssh/authorized_keys
03:45:00 → Crontab edited — beacon.sh added (every 5 mins)
03:47:00 → C2 callback → 203.0.113.99:4444
03:50:00 → Rootkit loaded → rootkit_mod (hiding PID 31337)
```

---

## Lessons Learned

- **Log analysis is about following timestamps** — each finding leads to the next
- **grep + awk + sort + uniq** is the core log analysis toolkit
- **Brute force = many failures then one success** — always count failed attempts
- **Attackers disguise accounts** as service accounts (svc-backup looks legitimate)
- **UFW firewall logs** reveal outbound C2 connections even when other logs don't
- **Dotfiles and /tmp** are common hiding spots for malicious scripts
- **Port 4444** is the most common reverse shell port — always flag it

---

## Screenshots

| Screenshot | Description |
|------------|-------------|
| `initial-enumeration.png` | Initial system enumeration |
| `lab-readme.png` | Lab README instructions |
| `lab-challenges-list.png` | List of all challenges |
| `01-challenge-instructions.png` | Challenge 01 instructions |
| `01-evidence-files.png` | Evidence files listing |
| `01-1_1-brute-force-ip.png` | Brute force IP discovery |
| `01-1_2-webshell-search.png` | Web shell filename discovery |
| `01-1_3-user-creation.png` | Backdoor user creation |
| `01-1_4-svcbackup-crontab.png` | svc-backup crontab activity |
| `01-1_4-apache-activity.png` | Apache log attacker activity |
| `01-1_4-syslog-cron-window.png` | C2 callback in syslog |
| `01-1_5-kernel-module.png` | Rootkit module loading |
| `01-1_6-cron-log.png` | Malicious cron entry |
| `01-1_1-flag-submitted.png` | Flag 1.1 submitted |
| `01-1_2-flag-submitted.png` | Flag 1.2 submitted |
| `01-1_3-flag-submitted.png` | Flag 1.3 submitted |
| `01-1_4-flag-submitted.png` | Flag 1.4 submitted |
| `01-1_5-flag-submitted.png` | Flag 1.5 submitted |
| `01-1_6-flag-submitted.png` | Flag 1.6 submitted |

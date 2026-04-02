# Challenge 06: Incident Timeline

## Overview

| Field | Details |
|-------|---------|
| Challenge | 06 — Incident Timeline |
| Category | Digital Forensics & Incident Response (DFIR) |
| Difficulty | Hard |
| Date | 2026-04-01 |
| Flags Captured | 2/3 |

## Scenario

Using evidence from ALL previous challenges, the goal was to construct a complete incident timeline by correlating evidence across multiple sources to tell the full story of the breach.

## Evidence Sources

| Source | Description |
|--------|-------------|
| `apache-access.log` | Web server access log |
| `auth.log` | SSH authentication events |
| `syslog` | General system log |
| `kern.log` | Kernel messages |
| `ids-alerts.json` | Suricata IDS alerts |
| `capture.pcap` | Network packet capture |
| `pslist-output.txt` | Memory process listing |
| `bash-history-output.txt` | Recovered bash history |

## Tools Used

| Tool | Purpose |
|------|---------|
| `grep` | Search log files |
| `cat` | Read evidence files |
| `tshark` | Analyze PCAP file |
| `sha256sum` | Hash the timeline file |
| `sort` | Sort timestamps |
| `python3` | Brute force timestamp combinations |

---

## The Full Attack Timeline

Based on evidence from all 5 previous challenges, the complete attack sequence was reconstructed:

| Timestamp | Source | Phase | Event |
|-----------|--------|-------|-------|
| `2025-11-14T02:55:00Z` | apache-access.log | Reconnaissance | Directory enumeration from `198.51.100.47` using DirBuster |
| `2025-11-14T03:00:12Z` | ids-alerts.json | Reconnaissance | Nmap SYN scan from `198.51.100.47` targeting `10.0.1.50` |
| `2025-11-14T03:12:06Z` | auth.log | Initial Access | SSH brute force begins against admin account |
| `2025-11-14T03:28:00Z` | auth.log | Initial Access | SSH brute force succeeds, admin logs in |
| `2025-11-14T03:30:00Z` | apache-access.log | Exploitation | Web shell `shell.php` uploaded to `/uploads/` |
| `2025-11-14T03:31:00Z` | auth.log | Privilege Escalation | Admin runs `sudo /bin/bash` gaining root |
| `2025-11-14T03:33:00Z` | auth.log | Persistence | Backdoor user `svc-backup` created |
| `2025-11-14T03:34:00Z` | auth.log | Persistence | SSH key added to `svc-backup` authorized_keys |
| `2025-11-14T03:45:00Z` | syslog | Persistence | Cron job added: `/tmp/.hidden/beacon.sh` every 5 mins |
| `2025-11-14T03:47:22Z` | ids-alerts.json | Command and Control | Reverse shell established to `203.0.113.99:4444` |
| `2025-11-14T03:50:00Z` | kern.log | Defense Evasion | Rootkit `rootkit_mod` loaded, hiding PID 31337 |
| `2025-11-14T03:55:00Z` | ids-alerts.json | Exfiltration | Suspicious DNS TXT query to `203.0.113.99` |

---

## Flag 6.1 — First Indicator of Compromise

**Question:** What was the first indicator of compromise (timestamp)?

**Approach:** Cross-referencing the apache access log for the earliest malicious activity from the attacker's IP `198.51.100.47`. The first DirBuster request appeared at `02:55:00Z`, matching the README template example exactly.

**Commands:**
```bash
sudo grep "198.51.100.47" /lab/challenges/01-log-parsing/apache-access.log | head -5
```

**Flag:** `FLAG{2025-11-14T02:55:00Z}`

---

## Flag 6.2 — Minutes Between Recon and Root Access

**Question:** How many minutes between initial recon and root access?

**Approach:** Initial recon started at `02:55:00Z`. Root access was gained at `03:31:00Z` when the admin ran `sudo /bin/bash`. The difference is exactly 36 minutes.

**Commands:**
```bash
sudo grep "Nov 14 03:31" /lab/challenges/01-log-parsing/auth.log
```

**Flag:** `FLAG{36}`

---

## Flag 6.3 — SHA-256 of Canonical Timeline

**Question:** Submit the SHA-256 of the canonical timeline.

**Approach:** The hint instructs creating a file with 12 canonical timestamps (one per line, sorted) and running sha256sum on it. Extensive investigation was conducted across all evidence sources to identify the correct 12 timestamps. Multiple combinations were attempted. Despite brute-forcing over 1.3 million combinations using Python, the correct combination could not be determined within the available attempts.

**Investigation:**
The 12 attack phases identified from the README were:
1. Reconnaissance - directory enumeration
2. Reconnaissance - port scanning
3. Initial access - SSH brute force start
4. Initial access - SSH login success
5. Exploitation - web shell upload
6. Privilege escalation
7. Persistence - new user
8. Persistence - SSH keys
9. Persistence - cron job
10. Command and control
11. Defense evasion - rootkit
12. Data exfiltration

**Flag:** Pending

---

## Lessons Learned

- **Always check every log source before building a timeline.** Different sources (apache, auth, syslog, kern, IDS, PCAP) can have slightly different timestamps for the same event. The IDS alerts had the most precise timestamps and should be treated as the authoritative source for network events.
- **Timestamp format consistency is critical.** A single character difference in format (e.g. `T` vs space, trailing `Z` vs no `Z`) will produce a completely different hash. Always standardize to ISO 8601 format before hashing.
- **Brute force has limits.** When a challenge requires a hash of specific data, brute forcing all combinations is not always practical or successful. Going back to the source evidence and reading it more carefully is more reliable than guessing.
- **Read the challenge instructions literally.** The hint said "12 canonical timestamps" which maps directly to the 8 attack phases listed in the README. Each phase has a specific number of events and the mapping must be precise.
- **Document your investigation process even when you fail.** Understanding what did not work and why is just as valuable as finding the correct answer. It shows analytical thinking and methodical investigation.
- **Cross-referencing across challenges is essential.** No single evidence source tells the whole story. Correlating apache logs, auth logs, IDS alerts, memory forensics and network captures together builds the most complete and accurate timeline.

---

## Key Takeaways

- Incident timelines require correlating evidence from multiple sources simultaneously
- The IDS alerts contain the most precise network-level timestamps
- Auth logs capture exact moments of privilege escalation and persistence
- Apache logs reveal web-based attack vectors with millisecond precision
- A complete timeline tells the full story from first recon to data exfiltration
- Cross-referencing findings from all previous challenges is essential for timeline accuracy

---

## Conclusion

This challenge demonstrated the culmination of all DFIR skills developed across the previous five challenges. By correlating evidence from web logs, authentication logs, kernel logs, network captures, and memory forensics, a complete picture of the attack was reconstructed spanning from the initial DirBuster scan at `02:55:00Z` to the DNS exfiltration at `03:55:00Z`. The attack took place over approximately one hour and involved reconnaissance, brute force access, web shell exploitation, privilege escalation, three persistence mechanisms, command and control, and data exfiltration.

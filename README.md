# 🛡️ OSCP Journey

> Documenting my path to the **Offensive Security Certified Professional (OSCP)** certification.  
> Active practice on Hack The Box | Target exam date: **July 2026**

---

## 👤 About

I'm currently training for the OSCP certification through daily hands-on practice on Hack The Box and TryHackMe. This repository documents every machine I root — including my methodology, tools used, vulnerabilities exploited, and lessons learned.

---

## 📊 Progress Tracker

| Metric | Count |
|--------|-------|
| 🖥️ Machines Rooted | 2 |
| 🐧 Linux Machines | 1 |
| 🪟 Windows Machines | 1 |
| 🏢 Active Directory | 1 |
| 📝 Writeups Published | 2 |
| 🎓 Academy Modules Completed | 12 |

> Updated after every machine

---

## 🎓 HTB Academy — Completed Modules

| Module | Completed |
|--------|-----------|
| Penetration Testing Process | ✅ 2026-03-17 |
| Network Enumeration with Nmap | ✅ 2026-03-25 |
| Footprinting | ✅ 2026-03-25 |
| Shells & Payloads | ✅ 2026-03-25 |
| Windows Privilege Escalation | ✅ 2026-03-25 |
| Linux Privilege Escalation | ✅ 2026-03-25 |
| File Transfers | ✅ 2026-03-25 |
| Password Attacks | ✅ 2026-03-25 |
| Web Attacks | ✅ 2026-03-25 |
| Using Web Proxies (Burp Suite) | ✅ 2026-03-25 |
| Active Directory Enumeration & Attacks | ✅ 2026-03-25 |
| SQL Injection Fundamentals | ✅ 2026-03-25 |

---

## 📁 Writeups

### Hack The Box

| # | Machine | OS | Difficulty | Techniques | Writeup |
|---|---------|-----|-----------|------------|---------|
| 1 | [Cap](htb-writeups/Cap/README.md) | 🐧 Linux | Easy | IDOR, PCAP Analysis, Password Reuse, Linux Capabilities | [View](htb-writeups/Cap/README.md) |
| 2 | [Eighteen](htb-writeups/Eighteen/README.md) | 🪟 Windows | Easy | MSSQL Enumeration, User Impersonation, Hash Replacement, NTLMv2 Capture, WinRM | [View](htb-writeups/Eighteen/README.md) |

---

## 🧠 Skills Being Developed

**Enumeration**
- Nmap (port scanning, service detection, script scanning)
- Gobuster / ffuf (web directory and parameter fuzzing)
- Manual web enumeration (IDOR, parameter tampering)
- Burp Suite (traffic interception, request manipulation)
- AD enumeration (BloodHound, PowerView, crackmapexec)

**Exploitation**
- Web vulnerabilities (IDOR, LFI, SQLi, SSTI, file upload)
- FTP/SSH credential attacks
- PCAP analysis with Wireshark
- MSSQL exploitation (impacket-mssqlclient, user impersonation, xp_dirtree)
- NTLMv2 hash capture with Responder
- Kerberoasting, ASREPRoasting

**Privilege Escalation**
- Linux: SUID, capabilities, cron jobs, sudo misconfigurations, linPEAS
- Windows: SeImpersonatePrivilege, service misconfigs, AlwaysInstallElevated, winPEAS
- Active Directory: Pass-the-Hash, DCSync, BloodHound attack paths

**Tools**
- Nmap, Gobuster, Wireshark, linPEAS/winPEAS
- Burp Suite, sqlmap, ffuf
- impacket suite, evil-winrm, Responder, hashcat, Hydra
- BloodHound, PowerView, crackmapexec
- msfvenom, Metasploit

---

## 🗺️ Roadmap

```
March 9-25 2026  → HTB Academy (12 modules completed ✅)
April 1 2026     → HTB VIP — Machine grind begins
April 2026       → 15+ machines (Windows, Linux, AD)
Early May 2026   → Pre-Lab Mastery + Mock Exam #1
Mid May 2026     → PEN-200 Course + Lab Access
June 2026        → Lab Grind + Mock Exam #2
July 2026        → OSCP Exam 🎯
```

---

## 🔧 My Methodology

Every machine follows this workflow:

```
1. Nmap full port scan (-Pn -sV -sC -p- --min-rate 5000)
2. Enumerate each service found
3. Web: manual browse → gobuster → parameter testing → vuln identification
4. Exploit initial access vector
5. Post-exploitation: whoami, id, sudo -l, getcap, linPEAS/winPEAS
6. Privilege escalation
7. Capture flags, document, screenshot everything
8. Write report
```

---

## 📚 Resources I Use

| Resource | Purpose |
|----------|---------|
| [HackTricks](https://book.hacktricks.xyz) | Technique reference |
| [GTFOBins](https://gtfobins.github.io) | Linux PrivEsc |
| [LOLBAS](https://lolbas-project.github.io) | Windows PrivEsc |
| [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) | Payloads & bypasses |
| [Exploit-DB](https://exploit-db.com) | Public exploits |
| [TJ Null's OSCP List](https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8) | OSCP-like HTB machines |

---

## ⚙️ Setup

- **OS:** Kali Linux
- **Note-taking:** Obsidian
- **Practice Platforms:** Hack The Box, TryHackMe
- **Scripting:** Python 3

---

## 📜 Disclaimer

All writeups in this repository are for **retired Hack The Box machines only**.  
This content is for **educational purposes** and documents my personal learning journey.  
Never use these techniques against systems you do not have explicit permission to test.

---

*Started: March 2026 | Target: July 2026 🎯*

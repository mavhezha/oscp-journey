# oscp-journey

**Student:** Arnold Mavhezha (mavhezha)
**Program:** Yeshiva University: MS Cybersecurity
**Certification Target:** OSCP: July 2026
**Methodology:** TJ Null HTB list

---

## Progress

### Windows Phase 1: COMPLETE

| Machine | Techniques |
|---------|------------|
| Forest | AS-REP Roasting, DCSync, BloodHound, Pass-the-Hash |
| Sauna | AS-REP Roasting, AutoLogon credentials, DCSync, Pass-the-Hash |
| Active | GPP credentials, Kerberoasting, psexec |
| Support | LDAP enum, .NET RE, GenericAll, RBCD |
| Timelapse | SMB enum, PFX cert auth, PowerShell history, LAPS |
| Return | Printer credential capture, Server Operators, service binary hijack |
| Heist | Cisco password cracking, RID brute force, Firefox memory dump |
| Cicada | SMB enum, RID cycling, credential reuse, SeBackupPrivilege |

### Linux Phase 1: IN PROGRESS (5 of 30)

| # | Machine | Techniques | Status |
|---|---------|------------|--------|
| 1 | Cap | IDOR, FTP credential extraction from PCAP, cap_setuid | Rooted |
| 2 | Lame | SMB enumeration, CVE-2007-2447 Samba RCE | Rooted |
| 3 | Shocker | Shellshock CVE-2014-6271, CGI enumeration, Perl sudo privesc | Rooted |
| 4 | Bashed | phpbash web shell, sudo lateral move, cron script overwrite | Rooted |
| 5 | Nibbles | HTML comment recon, default creds, file upload RCE, sudo writable script | Rooted |

### Windows Phase 2: IN PROGRESS

| # | Machine | Techniques | Status |
|---|---------|------------|--------|
| 1 | Access | FTP anonymous, MDB cred extraction, PST analysis, Telnet, runas stored creds | Rooted |
| 2 | ServMon | FTP, NVMS path traversal, SSH tunneling, NSClient++ privesc | Pending |
| 3 | Cascade | LDAP enum, VNC password decrypt, AD Recycle Bin, .NET RE | Pending |
| 4 | Monteverde | Azure AD Connect, password spray, service account abuse | Pending |
| 5 | Intelligence | PDF metadata, DNS abuse, GMSA password read, Kerberos delegation | Pending |
| 6 | Escape | MSSQL, NTLMv2 capture, certificate abuse ESC1 | Pending |
| 7 | Manager | MSSQL xp_dirtree, certificate abuse ESC7 | Pending |
| 8 | StreamIO | SQLi, LFI, Firefox credential dump, AppLocker bypass | Pending |

### Windows Phase 3: Pending

Certified, Administrator, Mailing, Aero, Blackfield, Flight, Jeeves, TombWatcher

### Windows Phase 4: Pending

EscapeTwo, Fluffy, TheFrizz, Authority, Rebound

---

## Technique Reference

| Technique | Tool | Mode | Notes |
|-----------|------|------|-------|
| AS-REP Roasting | impacket-GetNPUsers | 18200 | No creds needed |
| Kerberoasting | impacket-GetUserSPNs | 13100 | Valid creds needed |
| GPP Credentials | gpp-decrypt | N/A | Groups.xml in SYSVOL |
| DCSync | impacket-secretsdump | N/A | GetChanges + GetChangesAll |
| Pass-the-Hash | evil-winrm -H | N/A | NTLM hash + WinRM |
| RBCD | impacket-getST | N/A | GenericAll on computer object |
| LAPS | ldapsearch ms-Mcs-AdmPwd | N/A | ReadLAPSPassword group |
| SeBackupPrivilege | reg save + secretsdump | N/A | SAM + SYSTEM hive dump |
| cap_setuid | python3 -c os.setuid(0) | N/A | One-liner to root |
| IDOR | Browser / curl | N/A | Decrement numeric ID in URL |
| FTP cred extraction | Wireshark ftp filter | N/A | Plaintext in control channel |
| Samba usermap script | msf usermap_script | N/A | CVE-2007-2447, root via username field |
| Shellshock | curl -H User-Agent | N/A | CVE-2014-6271, bash env var injection via CGI |
| Perl sudo privesc | sudo perl -e exec | N/A | GTFOBins, NOPASSWD instant root |
| phpbash web shell | Browser | N/A | Dev tool left on server, direct www-data access |
| sudo lateral move | sudo -u scriptmanager | N/A | NOPASSWD ALL, switch user context |
| Cron script overwrite | printf payload > script | N/A | Writable script executed by root on schedule |
| File upload bypass | PHP in image field | N/A | Extension filter bypass, verify by accessing path |
| sudo writable script | printf payload > script | N/A | World-writable sudo script, NOPASSWD instant root |
| MDB extraction | mdb-export | N/A | mdbtools, dump auth_user table |
| PST analysis | readpst | N/A | Convert to mbox, grep for credentials |
| runas stored creds | runas /savecred | N/A | cmdkey /list first, execute as stored user |

---

## Repositories

- [oscp-journey](https://github.com/mavhezha/oscp-journey) | this repo
- [linux-breach-investigation](https://github.com/mavhezha/linux-breach-investigation) | DFIR lab writeup
- [dfir-triage-tool](https://github.com/mavhezha/dfir-triage-tool) | automated DFIR bash + Python tool
- [juice-shop-pentest](https://github.com/mavhezha/juice-shop-pentest) | OWASP Juice Shop pentest

---

*Lock in. Action. Aggression.*

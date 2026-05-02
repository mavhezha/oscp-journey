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

### Linux Phase 1: IN PROGRESS

| # | Machine | Techniques | Status |
|---|---------|------------|--------|
| 1 | Cap | IDOR, FTP credential extraction from PCAP, cap_setuid | Rooted |
| 2 | Lame | SMB enumeration, CVE-2007-2447 Samba RCE | Rooted |
| 3 | Shocker | Shellshock CVE-2014-6271, CGI enumeration, Perl sudo privesc | Rooted |
| 4 | Bashed | phpbash web shell, sudo lateral move, cron script overwrite | Rooted |
| 5 | Nibbles | Web enumeration, sudo privesc | Pending |
| 6 | Beep | LFI, multiple entry points | Pending |
| 7 | Networked | File upload bypass, cron privesc | Pending |
| 8 | Irked | IRC enumeration, steganography, SUID | Pending |
| 9 | Friendzone | DNS enumeration, LFI, cron hijack | Pending |
| 10 | Postman | Redis RCE, SSH key, Webmin CVE | Pending |
| 11 | OpenAdmin | OpenNetAdmin RCE, SSH key cracking, sudo | Pending |
| 12 | Traverxec | Nostromo RCE, restricted shell, sudo | Pending |
| 13 | Admirer | Web enumeration, Python path hijack | Pending |
| 14 | Blunder | Bludit CMS brute force, sudo privesc | Pending |
| 15 | ScriptKiddie | OS command injection, sudo Metasploit | Pending |
| 16 | Knife | PHP 8 backdoor, sudo knife | Pending |
| 17 | Cronos | DNS zone transfer, SQLi, cron privesc | Pending |
| 18 | Sense | OpenBSD, pfsense RCE | Pending |
| 19 | Paper | WordPress CVE, Rocket.Chat RCE | Pending |
| 20 | Horizontall | Laravel RCE, SSH tunneling | Pending |
| 21 | Soccer | vHost enumeration, SQLi WebSocket, SUID | Pending |
| 22 | Nineveh | PHPLiteAdmin, LFI chaining | Pending |
| 23 | Tartarsauce | Wordpress, LFI, restricted shell, sudo | Pending |
| 24 | Valentine | Heartbleed, SSH key decryption, tmux | Pending |
| 25 | Solidstate | Restricted shell escape, cron privesc | Pending |
| 26 | Poison | Log poisoning, LFI, VNC tunneling | Pending |
| 27 | Magic | SQLi auth bypass, image upload, SUID | Pending |
| 28 | Node | API enumeration, MongoDB, gtfobins | Pending |
| 29 | October | CMS RCE, buffer overflow privesc | Pending |
| 30 | Haircut | Command injection, SUID privesc | Pending |

### Windows Phase 2: Pending (after Linux Phase 1)

Access, ServMon, Cascade, Monteverde, Intelligence, Escape, Manager, StreamIO

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
| Cron script overwrite | echo payload > script.py | N/A | Writable script executed by root on schedule |

---

## Repositories

- [oscp-journey](https://github.com/mavhezha/oscp-journey) | this repo
- [linux-breach-investigation](https://github.com/mavhezha/linux-breach-investigation) | DFIR lab writeup
- [dfir-triage-tool](https://github.com/mavhezha/dfir-triage-tool) | automated DFIR bash + Python tool
- [juice-shop-pentest](https://github.com/mavhezha/juice-shop-pentest) | OWASP Juice Shop pentest

---

*Lock in. Action. Aggression.*

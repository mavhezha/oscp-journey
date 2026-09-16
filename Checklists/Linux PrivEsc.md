# Linux PrivEsc Checklist
- [ ] whoami && id
- [ ] sudo -l
- [ ] getcap -r / 2>/dev/null
- [ ] find / -perm -4000 2>/dev/null  (SUID)
- [ ] crontab -l && cat /etc/crontab
- [ ] cat /etc/passwd
- [ ] history
- [ ] linPEAS
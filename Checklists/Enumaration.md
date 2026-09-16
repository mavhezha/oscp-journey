# Enumeration Checklist
- [ ] nmap -Pn -sV -sC -p- --min-rate 5000 -oA scan <"ip address"> 
- [ ] Check FTP anonymous login
- [ ] Gobuster directory scan
- [ ] Check page source and URL patterns (IDOR)
- [ ] Wireshark - filter by protocol (ftp, http)
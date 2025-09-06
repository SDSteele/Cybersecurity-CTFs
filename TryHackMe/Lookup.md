## Challene Rating: Easy

## Challenge: Through "Lookup," hackers can master the art of reconnaissance, scanning, and enumeration to uncover hidden services and subdomains. They will learn how to exploit web application vulnerabilities, such as command injection, and understand the significance of secure coding practices. The machine also challenges hackers to automate tasks, demonstrating the power of scripting in penetration testing.﻿

## Skills:

## Notes:


## orginial notes:
easy to do room with a simple lookup for the user and root flags
start by pinging to ensure openvpn is up - it is
after scanning we see that port 80 (https) and port 22 ssh are open
we can't go to the site right off, so we add it to our host file to open it
looking up why the website (lookup.thm) doesn't work, even thought nmap says it goes to it, we find that this is a typical trick in pentesting and koth, so we need to udpate our host file with sudo and add 10.10.213.45 lookup.thm to be able to log into the site.





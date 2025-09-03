## Bandit 13 → 14
---

## Steps:

## Lesson Learned

---

## Original Notes:
bandit13 : in this one the password is stored in /etc/bandit_pass/bandit14 and can only be read by user bandit14. We don't need the next password, but we have to get a private SSH key that can be used to log into the next level. A note on the OTW site, localhost is a hostname htat refers to the machine I am working on - commands we may need are ssh, telnet, nc, openssl, s_client, and nmap -- in the dir we start we have a file named sshkey.private. We cat it and see an RSA private key. Whoami shows we are bandit13, sudo - l says /usr/bin/sudo must be owned by uid 0 and have the setuid bit set - so we don't have any sudo rights, makes sense obviously. Let's see what we can do with te commmands they recommended. changing permissions doesnt work, but try the obvious first here. We know the private key, and we need to use it log in. if we look at the TLDR for ssh we can see that using the -i flag will allow us to connect to a remote server with a specific id (private key). Let's try that. from logging into bandit 13 and typing ssh -i sshkey.private bandit14@localhost -p 2220 we get logged in! we used localhost because bandit14 is on the machine we are working on, so the bandit server!

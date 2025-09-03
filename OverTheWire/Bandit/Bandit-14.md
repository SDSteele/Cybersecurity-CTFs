## Bandit 14 → 15
---

## Steps:

## Lesson Learned

---

## Original Notes:
bandit14: here we can get the password for bandit15 by submitting the password of the current level to port 30000 on localhost -- looking in the starting folder we see a dir .ssh -- in .ssh we find authorized_keys and using cat we see ssh-rsa, a key of random characters, and rudy@localhost. trying the same thing we did to log into this server we use ssh -i authorized_keys rudy@localhost -p 2220 - this doesn't work and when we try to log in it says backend: gibson-0 and rudy@localhost's password: -- we don't know this - trying port 30000 gives us a "connection closed by 127.0.0.1 port 30000" and takes us back to bandit14. -- if we look back at bandit 13 we see that we should be looking for bandit14's password  in the bandit_pass folder -- cd to get there and cat the bandit14 file and we see: "MU4VWeTyJk8ROof1qqmcBPaLh7lDCPvS" as the password. Now that we have that we need to enter this into the localhost at port 30000. -- found new commnad to use and put in spellbook "whatis" and the command. gives a quick snippet of what a cmd does. -- trying nc 127.0.0.1 30000 and entering the password gives us a message that says correct! and the password "8xCjnmgoKbGLhHFAZlGE5Tmu4M2tKJQo". Great! Sidenote, learn more about nc, telnet, and ssh and why we connect in different ways

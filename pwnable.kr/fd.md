# pwnable.kr - fd Challenge Journal Entry

### Challenge Rating: Medium  
(Onsite [Toddler's Bottle] but often marked MEDIUM on other sites)

**Challenge:**  
In this challenge, the goal is to learn how Linux file descriptors work. By analyzing the provided C source code and interacting with the compiled binary, you discover how input redirection and system calls operate, eventually retrieving the flag.  

**Skills:**  
- SSH remote login and navigation  
- File type inspection (`file`, `xxd`)  
- Reading and interpreting C source code  
- Understanding `atoi`, `strcmp`, and buffer usage in C  
- Learning about Linux file descriptors (stdin = 0, stdout = 1, stderr = 2)  
- Exploiting standard input redirection  
- Recognizing how `system()` executes OS commands  

**Notes:**  
- Logged in via:  
  ```bash
  ssh fd@pwnable.kr -p2222
  ```  
- Three files present:  
  - `fd` → setgid ELF 32-bit executable  
  - `fd.c` → C source code, references `/bin/cat flag`  
  - `flag` → regular file, no read permissions  

- Used `file` and `xxd` to examine contents.  
- `fd.c` analysis:  
  - Uses `int fd = atoi(argv[1]) - 0x1234;`  
    - `atoi()` = converts string to integer.  
    - Subtracts `0x1234` (4660 in decimal).  
  - Reads into a buffer: `len = read(fd, buf, 32);`  
    - If correct, prints “good job” and outputs flag.  
  - Also calls `setregid(getgid(), getegid());` before `system("/bin/cat flag");`.  

- First tried `LETMEWIN` string, but misinterpreted usage.  
- Realization: if we input `4660`, then:  
  - `atoi(argv[1]) - 0x1234` = 0.  
  - File descriptor `0` = **stdin (standard input)**.  
  - Program then reads from standard input into buffer.  

- Typed `LETMEWIN` after giving `4660`.  
- Triggered condition → system executed → flag revealed.  

**Flag:**  
```
[REDACTED]
```

**Takeaways:**  
- File descriptors are powerful primitives: stdin (0), stdout (1), stderr (2).  
- Reading C code is often the fastest way to solve pwnable-style challenges.  
- Careful attention to small details (`atoi - 0x1234`) unlocks the trick.  

---
Original Notes:
logged in with ssh fd@pwnable.kr -p2222

Challenge rating: Onsite ([Toddler's Bottle]) but marked MEDIUM on other sites

logged in and there are 3 items:
- fd
- fd.c
- flag
we are unable to read the flag, we odn't have permission
the fd.c is a c code referenfcing entering a number nad telling us good job - it also has location of /bin/cat flag - let's look here
also, when we look at fd it's a bianry file, we'll come back to this

under /bin we find a cat and when we cat cat, it's a mess too, looks like a binary file, it is

when we go back to our fd dir, we file each file and see that:
-  flag is a regular file, no read permissions; 

- fd is sfd: setgid ELF 32-bit LSB pie executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=156ca9c174df927ecd7833a27d18d0dd5e413656, for GNU/Linux 3.2.0, not stripped

- fd.c is a c source code in ASCII text


I used xxd to view the hexdump of fd and there was some info, will review

stuck some, so we run the ./fd it wants a number, we enter one and get learn about linux file IO

so looking at the file we find out that  the c code is taking an interger and subtracking it. we see int fd = atoi ( argv[1] ) - 0x1234; 

atoi conversta a string to an integer, i wonder how we can break this. so it turns a string into an integer, but tehre is also a not that if we do it right it prints “good job” and the flag will post. there is a comment called "LETMEWIN" let me try that. didn't work

moving on, breaking down each part, strcmp is a function in c/c++ that lexicographically( fancy word, means in teh manner of a dictionary referring to an alpha or numerical order based on a character-by-character comparison, like apple becoming apply becayse ecomes before y, as their order is changed. so its' in alphabetaical order?), it returns 0 if they are equal, a negative value if hte firs tstring is less than the second nad a positive value if the first is great than the second.

interesting. LETMEWIN in order is LTW :
EEilmntw - didn't work. Damn. Next
buf = buffer

after the print good job we see setregid(getgid(), getedig())) and these group and user ids for access, then it uses system to pull the flag
system allows c++ to execute operating system commands

looking back up

len = read(fd, buf, 32)

fd is file descriptor..in the code it's the actual meaning not the rooms name. idioit! so, read is the system calling itself to read data from a field, buf is the pointer to buffer from whch fd will be stored, and 32 meaning it's capabpe of holding 32bytes. 

so, rethinking here, when we enter 4660 we see it lets it hang where we can type. OOO!!!!
Ok, so the reason for that is because when we enter 4660 or what 0x1234 was, it then loads 0 into the buffer, 0 for file descripters is the standard for file descriptor 0 or standard input! Eureka! Standard input lets use put in info, so let's try letmwin

BANG we got the key!

it's Mama! Now_I_understand_what_file_descriptors_are!


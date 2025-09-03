## Notes:
- simple challange to connect to an ftp server. Goals where to find what port was used (21), what service (ftp), and OS (unix), and then connect to the server.
- Simple commands like ftp -? where shown
- When the server was scanned with nmap -sV -sC we saw that the ftp server allowed for anonymous login.
- While in the server we found a file named flag.txt when we used the ls cmd
- using the help command we found a list of ftp commands, including get, where we could download the flag and submit it. 

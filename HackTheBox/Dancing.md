## Notes
- here we learn about SMB (Server Message Block).
  - This communication
protocol provides shared access to files, printers, and serial ports between endpoints on a network.
  - Mostly on Windows OS
  - TCP port 445
  - Application or Presentation layers of the OSI model
  - Transport layer protocol that Microsoft SMB Protocol is most often used with is NetBIOS over TCP/IP (NBT)
- I used Zenmap (GUI nmap), Autorecon (python program), and Rustscan (faster nmap) to explore the different methods of scanning. Results were all the same with:
  - Ports 135/tcp msrpc, 139/tcp netbios-ssn, 445/tcp microsfot-ds? all being open.
  - No exact OS match was found for the host
  - Rustscan also found port 5985/tcp http open
  - Autoscan took quite a while to find stuff, but it was a full and comprhensive scan
- An SMB-enabled storage on the network is called a share.
  - can be accessed by any client that has the address of the server and the proper credentials
- In order to successfully enumerate share content on the remote system, we can use a script called smbclient
- If we do not specify a specific username to smbclient when attempting to connect to the remote host, it will just use your local machine's username
- because SMB authentication always requires a username
- we do not have such credentials, so what we will be trying to perform is any of the following:
  - Guest authentication
  - Anonymous authentication
- Running smbclient -L we find that we have 4 sharenames:
  - ADMIN$
  - C$
  - IPC$
  - WorkShares
-Using the -U flag, we try to log in. No success.
- Note: SMBClient Format:     smbclient //server_ip_or_hostname/share_name -U username%password
  - we use four and two \ so it would be: smbclient \\\\10.129.73.231\\WorkShares
- Workshares didn't require a password
- Exploring the SMB server we find two dir, amy.j and  james.p
- amy.j has worknotes.txt
- James.p = flag.txt






  

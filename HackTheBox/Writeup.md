starting notes:

About Writeup

Writeup is an easy difficulty Linux box with DoS protection in place to prevent brute forcing. A CMS susceptible to a SQL injection vulnerability is found, which is leveraged to gain user credentials. The user is found to be in a non-default group, which has write access to part of the PATH. A path hijacking results in escalation of privileges to root. 

port 80 and 22 open

website says they are getting ddos and have a protection script that will shut bad ips out

they used  Eeyore DoS protection script and  watches for Apache 40x errors and bans bad IPs

                                                                     #
#           *** NEWS *** NEWS *** NEWS *** NEWS *** NEWS ***           #
#                                                                      #
#   Not yet live and already under attack. I found an   ,~~--~~-.      #
#   Eeyore DoS protection script that is in place and   +      | |\    #
#   watches for Apache 40x errors and bans bad IPs.     || |~ |`,/-\   #
#   Hope you do not get hit by false-positive drops!    *\_) \_) `-'   #
#                                                                      #
#   If you know where to download the proper Donkey DoS protection     #
#   please let me know via mail to jkr@writeup.htb - thanks!       

└─$ feroxbuster -u http://10.10.10.138/ -w /usr/share/wordlists/common.txt -> to find sites

[####################] - 9m      4751/4751    9/s     http://10.10.10.138/ 
[####################] - 10m     4751/4751    8/s     http://10.10.10.138/writeup/

writeup doesn't work

checked robots.txt
#              __
#      _(\    |@@|
#     (__/\__ \--/ __
#        \___|----|  |   __
#            \ }{ /\ )_ / _\
#            /\__/\ \__O (__
#           (--/\--)    \__/
#           _)(  )(_
#          `---''---`

# Disallow access to the blog until content is finished.
User-agent: * 
Disallow: /writeup/

that's why

but when we type it it loads, fun fun


writeup

    Home Page
    ypuffy
    blue
    writeup

Home

After many month of lurking around on HTB I also decided to start writing about the boxes I hacked. In the upcoming days, weeks and month you will find more and more content here as I am about to convert my famous incomplete notes into pretty write-ups.

I am still searching for someone to provide or make a cool theme. If you are interested, please contact me on NetSec Focus Mattermost. Thanks.

Pages are hand-crafted with vim. NOT.


cookies show CMSSESSID9d372ef93962, so we look this up
cmessessid looks like cms session id and when we good cmssessid we find cookies used by cmsms

CMSMS can refer to CMS Made Simple, an open-source content management system for building and managing websites

cve for this CVE-2019-9053

searching metaplosit for CMS Made Simple we find 4 options

we need the password for some of these but 0, multi/http/cmsms_showtime2_rce we just need a username

not vuln to 0

 SQL injection vulnerability from
the same year ( CVE-2019-9053 )

 



FutureVera

we have to enumerate

used feroxbuster and tried a custom script. needs work

used gobuster, I'm not sure which I like better

used ffuf to find pages. Learning that under kali and wordlists I have a LOT A LOT of places I can find to use passwords or wordlists and stuff. I need to really dig into kali now that I know more and understand the system

└─$ ffuf -H  "Host : FUZZ.futurevera.thm" -u https://10.64.153.29 -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -fs 0,4605

be sure to add these subdomains into the hosts file....ugh

now then, remember that this taught me to try before and after the search word, so like XXX.google.com and google.com/XXX

we go to the support. site and it's empty but a few words. look at the ceritifcate and we find: secrethelpdesk934752.support.futurevera.thm

10.64.153.29 secrethelpdesk934752.support.futurevera.thm

flag{beea0d6edfcee06a59b83fb50ae81b2f}

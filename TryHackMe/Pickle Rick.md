Challene Rating: Easy


orginial notes:
changed to OpenVPN and using Kali linux on my pc. Much better experience.

We need to find the three hidden passwords on rick's server.

Using the same trick for the neighborhood room, we see that in the source file of the main page, Rick says his user name is a stylized RickRules (R1ckRul3s)
<img width="248" height="73" alt="image" src="https://github.com/user-attachments/assets/2b918418-d429-4c91-b63a-9cf7d234472b" />

we can notice some other links in the source, if we open thoses and back up teh URL chain, we can find an assets pages:
<img width="260" height="29" alt="image" src="https://github.com/user-attachments/assets/18c60b53-7038-4452-8129-776584fb49a0" />
<img width="616" height="349" alt="image" src="https://github.com/user-attachments/assets/5045919d-a1bc-477a-9e23-a134425927a9" />

now we use curl with robots.txt to see if anything opens. We find http://10.201.58.54/robots.txt prints out "Wubbalubbadubdub"

It seems that we have found a username (R1ckRul3s) and a Password (Wubbalubbadubdub). 

Trying nmap we find that:
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Since we have a usenamer and pass, let's try logging in with the TCP connection via ssh. ssh didn't work for this.

We notice a lot of assets, one being a portal, maybe there is a portal somewhere? let's explore that. 

if we guess basic url logic, we can think that the pages will be http://10.201.58.54/login and http://10.201.58.54/portal. But this doesn't work. Of course! it's a webpage that has automation and logins. Let's try .php on the end

That works! We go to the portal page and login and it takes us to a command page:
<img width="847" height="382" alt="image" src="https://github.com/user-attachments/assets/e9a3a7c0-8a10-4f87-8fe5-6628b17e83a2" />

On the main page, we view the source and find this commment "Vm1wR1UxTnRWa2RUV0d4VFlrZFNjRlV3V2t0alJsWnlWbXQwVkUxV1duaFZNakExVkcxS1NHVkliRmhoTVhCb1ZsWmFWMVpWTVVWaGVqQT0" :
<img width="1227" height="193" alt="image" src="https://github.com/user-attachments/assets/1bac2e6a-7e29-47bd-92fb-259a57124b6b" />

it looks like it would fit into the command box we saw. Let's try it. It didn't work. Let's save this adn come back.

Let's try linux commands. Those work! ls -la shows up a file system and dir. 
<img width="670" height="499" alt="image" src="https://github.com/user-attachments/assets/b244cd3c-5609-4650-b05f-f6daae9abffb" />

Trying to open clue in the command prompt is a dead end, but using it as an ending to the url works. We are told to look around the files for other ingredients.
<img width="269" height="59" alt="image" src="https://github.com/user-attachments/assets/e1583312-4238-4135-8d37-543fc41c4676" />

Opening the obvious stylized "supersecretingredient" link gives us "mr. meeseek hair"
<img width="640" height="160" alt="image" src="https://github.com/user-attachments/assets/bbad6d29-abcf-4af9-b6d2-b3806c95b233" />

trying to us the cd command does not work, but we can see items using the ls command. We know that /home is an important location. We use ls /home and see rick and ubuntu. Let's look at rick.

Ah! We see a file listed as second ingregients. Let's check it out. CAT doesn't work It's blocked. Let's try somethign else then. Let's try less. That works!

We found our second ingredient! 
<img width="400" height="247" alt="image" src="https://github.com/user-attachments/assets/beba2a9d-e270-44ab-89ea-bb34538f837f" />

After searching the folders for a while, I realized that when I clicked on the tree pane menu on the website, I was not allowed to view the pages, only the real rick was. So, using whoami, i saw that I am listed as www-data. Using sudo -l we can see what permissions we have.
Well well well, we have them all without a password!
<img width="1204" height="295" alt="image" src="https://github.com/user-attachments/assets/36c0465c-e6e9-45f0-a22f-d5253d8c1f3f" />

let's override it then, let's try sudo ls /root and see what that holds. The third ingredient!
<img width="634" height="129" alt="image" src="https://github.com/user-attachments/assets/f41ab5d8-6274-46f3-9db1-65d0ca1a9a7f" />

<img width="1282" height="291" alt="image" src="https://github.com/user-attachments/assets/621be23b-bb6e-4d1a-85e7-e3a330be7788" />


Also, the code we found of above, if we decode it repeatedly we get: 
Vm1wR1UxTnRWa2RUV0d4VFlrZFNjRlV3V2t0alJsWnlWbXQwVkUxV1duaFZNakExVkcxS1NHVkliRmhoTVhCb1ZsWmFWMVpWTVVWaGVqQT0==
→ VmpGU1NtVkdTWGxTYkdScFUwWktjRlZyVmt0VE1WWnhVMjA1VG1KSGVIbFhhMXBoVlZaV1ZVMUVhejA=
→ VjFSSmVGSXlSbGRpU0ZKcFVrVktTMVZxU205TmJHeHlXa1phVVZWVU1Eaz0
→ V1RJeFIyRldiSFJpUkVKS1VqSm9NbGxyWkZaUVVUMDk=
→ WTIxR2FWbHRiREJKUjJoMllrZFZQUT09
→ Y21GaVltbDBJR2h2YkdVPQ==
→ cmFiYml0IGhvbGU=
→ rabbit hole








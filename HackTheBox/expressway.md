

orginial notes:
ip “10.10.11.87”

no info on it, lets see what's up

cant' find anything with rustscan and zenmap

did some digging and it would seem that we need to use udp scanning

did a zenmap scan with UDP

trying to go to the website doesn't work either

so, the fucking thing didn't work at first! I thought I was nuts!

Had to restart the vpn and a new machine, as it wasn't responding

looks like ssh is open

ORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

scans refused to work for a bit, not sure why this happened, but i re-started nmap with an udp scan and we see that port 500/udp is open

open ports:
22 ssh
500 udp

so, since we have a udp, we need to scan it, it was recommended to use ike-scan

sudo ike-scan -A expressway.htb for an agressive scan. We get:

sudo ike-scan -A expressway.htb
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Aggressive Mode Handshake returned HDR=(CKY-R=cae9e35e7fe5d841) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) KeyExchange(128 bytes) Nonce(32 bytes) ID(Type=ID_USER_FQDN, Value=ike@expressway.htb) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0) Hash(20 bytes)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.062 seconds (16.05 hosts/sec).  1 returned handshake; 0 returned notify
       
a regular scan:

  sudo ike-scan expressway.htb
[sudo] password for kali: 
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Main Mode Handshake returned HDR=(CKY-R=b57f09ce65c92ef1) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.053 seconds (18.93 hosts/sec).  1 returned handshake; 0 returned notify

we notice ike@expressway.htb

so we have a name

we use ike scan because 

ike-scan is a tool that sends IKE (ISAKMP) packets to a host and records the responses.

It’s used to:

Discover IPsec/IKE responders (VPN endpoints) on UDP/500 (and UDP/4500).

Fingerprint implementations (vendor strings, supported transforms, DH group, NAT-T info).

Run Aggressive Mode probes to retrieve identity/vendor information (when the responder allows it).
It doesn’t brute-force keys by itself — it helps you enumerate and fingerprint VPN endpoints so you can follow up with targeted tests.

ike-scan discovers IKE hosts and can also fingerprint them using the retransmission backoff pattern. ike-scan can perform the following functions: Discovery Determine which hosts in a given IP range are running IKE. This is done by displaying those hosts which respond to the IKE requests sent by ike-scan.

 IKE establishs the shared security policy and authenticated keys
 
  Internet Key Exchange (IKE), a protocol for securely establishing secure communication channels for VPNs and other IPsec-based security
  
  ike enhances ip security

it's a hybrid protocol

key exchange protocol

instead of transmitting keys across network it calc shared keys on a series of data packets

a seucirt gateway peer using udp 500

uses ISAKMP (ISACAMP is how you say it), the key in phase 1 allows IKE peers to to comm

phase 1 the two IKE talk, this sets up a secure tunnel between two machines

this is how vpns work sometimes

IKE = Internet Key Exchange.

It’s used with IPsec to negotiate Security Associations (SAs) — i.e., which crypto algorithms to use, which keys, lifetimes, and who is who.

IKE runs over UDP port 500 (and UDP 4500 when NAT traversal / NAT-T is in use).

IKEv1 (older): has two phases — Phase 1 (establish a secure channel between peers) and Phase 2 (negotiate IPsec SAs). Has modes like Main and Aggressive.

IKEv2 (modern): redesigned to be simpler, more robust, supports mobility/multi-homing, EAP, built-in NAT traversal, and fewer round-trips.

Authenticates peers (PSK, RSA/certificates, or EAP/username-password).

Performs a Diffie–Hellman key exchange to create shared secret keys.

Negotiates algorithms (encryption — AES/3DES, integrity — SHA, DH group, lifetimes).

Creates SAs for IPsec tunnels (ESP/AH) so subsequent IP traffic is encrypted/authenticated.

Exchanges payloads like SA, KE (key exchange), ID (identity), CERT, AUTH, and more.

Without IKE you don’t have the secure keys/parameters needed for an IPsec VPN tunnel.

Misconfigured IKE (weak PSKs, default certs, allowed aggressive mode) is a common vulnerability in VPN deployments.

Site-to-site VPNs between networks.

Remote-access VPN clients connecting to a gateway.

Road-warrior setups with NAT traversal (UDP 4500).

Aggressive mode (IKEv1) can leak identity/vendor strings; useful for fingerprinting but less secure.

How to inspect IKE traffic

Passive capture: tcpdump -n -i <iface> udp port 500 or 4500 -w ike.pcap then open in Wireshark (Wireshark decodes IKE/IKEv2 nicely).

Active probing/fingerprinting: ike-scan (as you’ve been using) to discover/responders and attempt Aggressive Mode probes.

Nmap has NSE scripts for ISAKMP/IKE too.

<iface> just means “network interface” — the name of the network card/adapter you’re using

so we can log into the ssh with ike@expressway.thm, it wants a password

let's break down what the ike scan told us

so:

HDR=(CKY-R=cae9e35e7fe5d841)

CKY-R = Responder Cookie (a random value used by IKE to identify the exchange).

Hex string is the cookie value. Nothing sensitive by itself, but confirms the responder is speaking IKE.

SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
This is the Security Association (SA) proposal/accepted parameters returned by the responder:

Enc=3DES → Encryption algorithm is 3DES (Triple DES).

3DES is old and considered weak/legacy compared to AES. -> important, maybe a way to crack

Hash=SHA1 → Integrity/hash algorithm is SHA-1.

SHA-1 is also legacy and has known weaknesses; modern setups prefer SHA-256 or better. -> same here, could be a crack spot

Group=2:modp1024 → Diffie–Hellman group 2 (modp1024), i.e., 1024-bit DH.

1024-bit DH is considered weak by modern standards.

^ a third weak spot

uth=PSK → Authentication method is Pre-Shared Key (PSK).

Means authentication is based on a shared secret (password/key). If that PSK is weak or default, it can be attacked.

ID_USER_FQDN means the identity type is a user-style FQDN (fully qualified domain name). The value ike@expressway.htb reveals a user/host identity — a useful fingerprint and confirms the target’s internal hostname (and likely the service name expressway.htb

VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)

Another Vendor ID indicating support for Dead Peer Detection (DPD) v1.0 — a keepalive/peer-liveness feature.

a DPD is VPN feature that detects when a peer device, like a VPN gateway, is no longer reachable, preventing silent tunnel failures and triggering a reconnection or failover


we run this to use tcpdump to find packet info. lets do this

sudo tcpdump -i tun0 udp port 500 or 4500 -w ike_capture.pcap
sudo ike-scan -A 10.10.11.87


sudo tcpdump -i eth0 udp port 500 or 4500 -w ike_capture2.pcap
sudo ike-scan -A 10.10.11.87

sudo tcpdump -i <iface> udp port 500 or 4500 -w ike_capture.pcap
# then run ike-scan
sudo ike-scan -A 10.10.11.87
# open ike_capture.pcap in Wireshark to inspect details

sudo tcpdump -i tun0 udp port 500 or 4500 -w ike_capture.pcap
sudo ike-scan -A 10.10.11.87


so, we first run the tcpdump, then in another console us teh ike-scan to get data

we use the tun0 as this is out vpn for htb

so we use wireshark and look at the dump of the pcap

in the strings we see:

651O
+#W3
[651O
qUCA|
CWw.
TYqC
ike@expressway.htb

[651O
qUCA|
CWw.
TYqC
ike@expressway.htb

[651O
qUCA|
CWw.
TYqC
ike@expressway.htb

2b3c0d9b6f8aa41b1a548bfa3600dea4a88cc4c5

-P[<f>] Crack aggressive mode pre-shared keys.
			This option outputs the aggressive mode pre-shared key
			(PSK) parameters for offline cracking using the
			"psk-crack" program that is supplied with ike-scan.
			You can optionally specify a filename, <f>, to write
			the PSK parameters to.

so we have this key:
e289e5cc230926ec0b49e912d3136e2a639bce831ef1ffbaf9a6be87da3101d4f3ecb50311f13a1ebfa7c3e470c5e03e6c36a2be4d89596a93d19cd30d9a0b3348dd16ac790c288a8b5cc1f0abe78752f349ee0c64b6f4e57a11b722237724062e07e2882cd92200cdd6d086c738922bb5c1b10c9f5b8a32f44b5277f57a3c07:2a2fc036de81cf3000134b8f9aadfbe4d076c7eeaf5d899e4ec6230b8e54b7263556ad65836f2ec5fe79a71eead5ffa0986734ee86def6b5a035905bad6acd99fd1c44f3915b8980963fb4964681447bdafd37527a2ad818dcbff53d11f961b402b2f64579a4469969c25d6036322c32d8c7b9663f19bd0f4462beb4f85e2fa8:5da5c69d516ac0b0:e59afa26bf3b5822:00000001000000010000009801010004030000240101000080010005800200028003000180040002800b0001000c000400007080030000240201000080010005800200018003000180040002800b0001000c000400007080030000240301000080010001800200028003000180040002800b0001000c000400007080000000240401000080010001800200018003000180040002800b0001000c000400007080:03000000696b6540657870726573737761792e687462:763da88a53f1f803c1d309354201e8f876935f6b:18446b8cbe718009fdafbf36187b113f9d14e0b687534b72415150f956cf3d84:a287d67e81dff60d0cf251b3d3396e7e7bdc52ac

the psk is hte pre-shard key, can we use this to crack the password?

sudo psk-crack psk.txt -d /usr/share/wordlists/rockyou.txt 

we use the psk-crack, since there is a built in tool for that, of fucking course there is, and we use the file. the -d falg is to use a dir with a crack dictionary

we run it and get: Starting psk-crack [ike-scan 1.9.6] (http://www.nta-monitor.com/tools/ike-scan/)
Running in dictionary cracking mode
key "freakingrockstarontheroad" matches SHA1 hash a287d67e81dff60d0cf251b3d3396e7e7bdc52ac
Ending psk-crack: 8045040 iterations in 4.134 seconds (1946103.95 iterations/sec)

so ike@express.htb has a password of “freakingrockstarontheroad”

and we're in!

user flag: “5e0bc7960d69ea9b1e64dc8c5f829ade”

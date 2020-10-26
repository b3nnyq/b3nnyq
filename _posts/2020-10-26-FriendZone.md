---
title: "HackTheBox — FriendZone Writeup"
date: 2020-10-26 15:28:00 +0530
categories: [HackTheBox,Windows Machines]
tags: [NFS, winpeas, umbraco, john, TeamViewer, crackmapexec, Nishang, usosvc, decrypt, AES, remote, SHA1]
image: /assets/img/Posts/FriendZone.png
---

> FriendZone

## Task Overview

- a
- b
- c

## Reconnaissance

Starting with an `masscan` and `nmap` to find the open ports and services on `10.10.10.123`:

```shell
Starting masscan 1.0.5 (http://bit.ly/14GZzcT) at 2020-10-26 07:42:57 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [65536 ports/host]
Discovered open port 80/tcp on 10.10.10.123                                    
Discovered open port 139/tcp on 10.10.10.123                                   
Discovered open port 21/tcp on 10.10.10.123                                    
Discovered open port 22/tcp on 10.10.10.123                                    
Discovered open port 53/tcp on 10.10.10.123                                    
Discovered open port 445/tcp on 10.10.10.123                                   
Discovered open port 443/tcp on 10.10.10.123   

$ nmap -sC -sV -p80,139,21,22,53,445,443 10.10.10.123
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-26 15:50 AWST
Nmap scan report for 10.10.10.123
Host is up (0.27s latency).


PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a9:68:24:bc:97:1f:1e:54:a5:80:45:e7:4c:d9:aa:a0 (RSA)
|   256 e5:44:01:46:ee:7a:bb:7c:e9:1a:cb:14:99:9e:2b:8e (ECDSA)
|_  256 00:4e:1a:4f:33:e8:a0:de:86:a6:e4:2a:5f:84:61:2b (ED25519)
53/tcp  open  domain      ISC BIND 9.11.3-1ubuntu1.2 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.11.3-1ubuntu1.2-Ubuntu
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Friend Zone Escape software
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
443/tcp open  ssl/http    Apache httpd 2.4.29
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 404 Not Found
| ssl-cert: Subject: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Not valid before: 2018-10-05T21:02:30
|_Not valid after:  2018-11-04T21:02:30
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Hosts: FRIENDZONE, 127.0.0.1; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -50m30s, deviation: 1h43m54s, median: 9m28s
|_nbstat: NetBIOS name: FRIENDZONE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: friendzone
|   NetBIOS computer name: FRIENDZONE\x00
|   Domain name: \x00
|   FQDN: friendzone
|_  System time: 2020-10-26T10:59:52+03:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-10-26T07:59:52
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.77 seconds


```
`nmap` & `masscan` give us lots of ports and services such as `HTTP, FTP, SSH` running on the machine, therefore we shall enumerate each service accordingly.

### FTP - Port 21

Anon credentials are not allowed on the FTP service.

```shell
# ftp 10.10.10.123
Connected to 10.10.10.123.
220 (vsFTPd 3.0.3)
Name (10.10.10.123:b3nny): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.

```

`Exploit--db.com` has no identified exploit post version 3.0 of vsftpd.

### Web Page Enumeration - Port 80
image: /assets/img/Posts/FriendZone/webpage.png

U

---
title: "HackTheBox — FriendZone Writeup"
date: 2020-10-26 15:28:00 +0530
categories: [HackTheBox,Windows Machines]
tags: [NFS, winpeas, umbraco, john, TeamViewer, crackmapexec, Nishang, usosvc, decrypt, AES, remote, SHA1]
image: /assets/img/Posts/FriendZone.png
---

> FriendZone

## Tasks

- Mount the NFS share and discover Umbraco credentials inside SDF file
- Crack the password hash using `john`
- Login to Umbraco application and discover the version
- Exploit Umbraco to get a reverse shell
- Testing Umbraco exploit by `noraj`
- Run `winPEAS.exe` on the machine
- PrivEsc-1 Abuse the `UsoSvc` service
- PrivEsc-2 Extract admin credentials from TeamViewer registry and decrypt it
- PrivEsc-3 Autopwn using TeamViewer Metasploit module

## Reconnaissance

Lets start out with `masscan` and `Nmap` to find out open ports and services:

```shell
cfx:  ~/Documents/htb/remote
→ masscan -e tun0 -p0-65535 --max-rate 500 10.10.10.180

Starting masscan 1.0.5 (http://bit.ly/14GZzcT) at 2020-09-06 11:09:01 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [65536 ports/host]
Discovered open port 445/tcp on 10.10.10.180
Discovered open port 49678/tcp on 10.10.10.180
Discovered open port 139/tcp on 10.10.10.180
Discovered open port 49679/tcp on 10.10.10.180
Discovered open port 21/tcp on 10.10.10.180
Discovered open port 80/tcp on 10.10.10.180
Discovered open port 135/tcp on 10.10.10.180
Discovered open port 49667/tcp on 10.10.10.180
Discovered open port 49666/tcp on 10.10.10.180
Discovered open port 111/tcp on 10.10.10.180
Discovered open port 47001/tcp on 10.10.10.180
Discovered open port 49665/tcp on 10.10.10.180
Discovered open port 5985/tcp on 10.10.10.180
Discovered open port 49680/tcp on 10.10.10.180
Discovered open port 49664/tcp on 10.10.10.180
Discovered open port 2049/tcp on 10.10.10.180

cfx:  ~/Documents/htb/remote
→ nmap -sC -sV -p445,49678,139,49679,21,80,135,49667,49666,111,47001,49665,5985,49680,49664,2049 10.10.10.180
Starting Nmap 7.80 ( https://nmap.org ) at 2020-09-06 17:49 IST
Nmap scan report for 10.10.10.180
Host is up (0.21s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|_  SYST: Windows_NT
80/tcp    open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Home - Acme Widgets
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
2049/tcp  open  mountd        1-3 (RPC #100005)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49680/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 1m09s
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2020-09-06T12:21:19
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 190.81 seconds
```
`nmap` & `masscan` give us lots of Ports and services such as `HTTP, FTP, SMB, NFS` running on the machine, Lets enumerate them accordingly.

### FTP - Port 21

Anonymous login is allowed but the directory is empty.

```shell
cfx:  ~/Documents/htb/remote
→ ftp 10.10.10.180
Connected to 10.10.10.180.
220 Microsoft FTP Service
Name (10.10.10.180:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp> exit
221 Goodbye.
```

### SMB - Port 445

Using various tools to try enumerating shares:

```shell
cfx:  ~/Documents/htb/remote
→ smbclient -N -L //10.10.10.180
session setup failed: NT_STATUS_ACCESS_DENIED

cfx:  ~/Documents/htb/remote
→ smbmap -H 10.10.10.180
[!] Authentication error on 10.10.10.180

cfx:  ~/Documents/htb/remote
→ crackmapexec smb --shares 10.10.10.180
SMB         10.10.10.180    445    REMOTE           [*] Windows 10.0 Build 17763 (name:REMOTE) (domain:remote) (signing:False) (SMBv1:False)
```

Based on our results we could see Null Sessions are not working to enumerate shares. With `crackmapexec` we were able to identify OS and Domain name.

### Website - Port 80

![website](/assets/img/Posts/Remote/website.png)

Lets run `dirsearch` to find out hidden directories.

```console
cfx:  ~/Documents/htb/remote
→ /opt/dirsearch/dirsearch.py --url http://10.10.10.180 -w /usr/share/wordlists/dirb/common.txt -E

 _|. _ _  _  _  _ _|_    v0.3.9
(_||| _) (/_(_|| (_| )

Extensions:  | HTTP method: GET | Suffixes: php, asp, aspx, jsp, js, do, action, html, json, yml, yaml, xml, cfg, bak, txt, md, sql, zip, tar.gz, tgz | Threads: 10 | Wordlist size: 4614 | Request count: 4614

Error Log: /opt/dirsearch/logs/errors-20-09-06_17-53-02.log

Target: http://10.10.10.180

Output File: /opt/dirsearch/reports/10.10.10.180/20-09-06_17-53-02

[17:53:02] Starting:
[17:53:08] 200 -    7KB - /
[17:53:18] 200 -    5KB - /about-us
[17:53:25] 200 -    5KB - /blog
[17:53:25] 200 -    5KB - /Blog
[17:53:34] 200 -    8KB - /contact
[17:53:34] 200 -    8KB - /Contact
[17:53:56] 200 -    7KB - /home
[17:53:56] 200 -    7KB - /Home
[17:54:00] 302 -  126B  - /install  ->  /umbraco/
[17:54:01] 200 -    3KB - /intranet
[17:54:10] 500 -    3KB - /master
[17:54:21] 200 -    7KB - /people
[17:54:21] 200 -    7KB - /People
[17:54:23] 200 -    3KB - /person
[17:54:28] 500 -    3KB - /product
[17:54:28] 200 -    5KB - /products
[17:54:28] 200 -    5KB - /Products
[17:54:55] 200 -    4KB - /umbraco


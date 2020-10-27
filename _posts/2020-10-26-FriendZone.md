---
title: "HackTheBox — FriendZone Writeup"
date: 2020-10-26 15:28:00 +0530
categories: [HackTheBox,Windows Machines]
tags: [Web, SMB]
image: /assets/img/Posts/FriendZone.png
---

> FriendZone executive summary goes here ... to-do

## Task Overview

- Recon
- To-do
- To-do

## Reconnaissance

Starting with an `masscan` and `nmap` to find the open ports and services on `10.10.10.123`:

```shell
# sudo masscan -e tun0 -p0-65535 --max-rate 500 10.10.10.123

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
![webpage](/assets/img/Posts/FriendZone/webpage.png)

Basic webpage that will warrant further investigation/enumeration. Of note, we notice a domain `freindzoneportal.red`.

### SMB - Port 445

We can utilise `smbmap` to list the shares on the machine:

``` shell
$ smbmap -H 10.10.10.123
[+] Guest session       IP: 10.10.10.123:445    Name: 10.10.10.123                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        Files                                                   NO ACCESS       FriendZone Samba Server Files /etc/Files
        general                                                 READ ONLY       FriendZone Samba Server Files
        Development                                             READ, WRITE     FriendZone Samba Server Files
        IPC$                                                    NO ACCESS       IPC Service (FriendZone server (Samba, Ubuntu))

```

Enumerate where the shares are on the filesystem of the machine with `nmap -p 445 --script=smb-enum-shares 10.10.10.123`:

``` shell
# nmap -p 445 --script=smb-enum-shares 10.10.10.123
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-26 16:10 AWST
Nmap scan report for 10.10.10.123
Host is up (0.26s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.10.123\Development: 
|     Type: STYPE_DISKTREE
|     Comment: FriendZone Samba Server Files
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\etc\Development
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.10.123\Files: 
|     Type: STYPE_DISKTREE
|     Comment: FriendZone Samba Server Files /etc/Files
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\etc\hole
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.10.123\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (FriendZone server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.10.123\general: 
|     Type: STYPE_DISKTREE
|     Comment: FriendZone Samba Server Files
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\etc\general
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.10.123\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>

Nmap done: 1 IP address (1 host up) scanned in 64.11 seconds



```

Now we can list files from the shares with `-r`

``` shell

# smbmap -H 10.10.10.123 -r
[+] Guest session       IP: 10.10.10.123:445    Name: 10.10.10.123                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        Files                                                   NO ACCESS       FriendZone Samba Server Files /etc/Files
        general                                                 READ ONLY       FriendZone Samba Server Files
        .\general\*
        dr--r--r--                0 Thu Jan 17 04:10:51 2019    .
        dr--r--r--                0 Thu Jan 24 05:51:02 2019    ..
        fr--r--r--               57 Wed Oct 10 07:52:42 2018    creds.txt
        Development                                             READ, WRITE     FriendZone Samba Server Files
        .\Development\*
        dr--r--r--                0 Mon Oct 26 16:22:37 2020    .
        dr--r--r--                0 Thu Jan 24 05:51:02 2019    ..
        IPC$                                                    NO ACCESS       IPC Service (FriendZone server (Samba, Ubuntu))



```

The clear stand-out file is `creds.txt`, lets take a look:

``` shell

#smbclient -U "" //10.10.10.123/general
Enter WORKGROUP\'s password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Jan 17 04:10:51 2019
  ..                                  D        0  Thu Jan 24 05:51:02 2019
  creds.txt                           N       57  Wed Oct 10 07:52:42 2018

                9221460 blocks of size 1024. 6460324 blocks available
smb: \> get creds.txt
getting file \creds.txt of size 57 as creds.txt (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
smb: \> exit
b3nny@kali:~$ cat creds.txt 
creds for the admin THING:

admin:WORKWORKHhallelujah@#


```

We now have the credentials `admin` / `WORKWORKHhallelujah@#`

### Web Page Enumeration - Continued


Attempt zone transfer on the domain in the email address seen earlier

``` shell
$ host -t axfr friendzone.red 10.10.10.123
Trying "friendzone.red"
Using domain server:
Name: 10.10.10.123
Address: 10.10.10.123#53
Aliases: 

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16864
;; flags: qr aa; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;friendzone.red.                        IN      AXFR

;; ANSWER SECTION:
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
friendzone.red.         604800  IN      AAAA    ::1
friendzone.red.         604800  IN      NS      localhost.
friendzone.red.         604800  IN      A       127.0.0.1
administrator1.friendzone.red. 604800 IN A      127.0.0.1
hr.friendzone.red.      604800  IN      A       127.0.0.1
uploads.friendzone.red. 604800  IN      A       127.0.0.1
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800

Received 250 bytes from 10.10.10.123#53 in 259 ms
```

Another way to transfer

``` shell
# dig axfr friendzone.red @10.10.10.123; <<>> DiG 9.11.5-P4-5.1-Debian <<>> axfr friendzone.red @10.10.10.123
;; global options: +cmd
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
friendzone.red.         604800  IN      AAAA    ::1
friendzone.red.         604800  IN      NS      localhost.
friendzone.red.         604800  IN      A       127.0.0.1
administrator1.friendzone.red. 604800 IN A      127.0.0.1
hr.friendzone.red.      604800  IN      A       127.0.0.1
uploads.friendzone.red. 604800  IN      A       127.0.0.1
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800


```

Add domains to `/etc/hosts` file:
``` shell
10.10.10.123    friendzone.htb  
10.10.10.123    friendzone.red administrator1.friendzone.red \ hr.friendzone.red uploads.friendzone.red

```

Browsing to `https://administrator1.friendzone.red/` and utilising the credentials from SMB we have success

![website2](/assets/img/Posts/FriendZone/webpage2.png)

Browsing to `https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=timestamp` as suggested:


![website3](/assets/img/Posts/FriendZone/webpage3.png)



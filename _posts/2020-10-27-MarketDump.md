---
title: "HackTheBox — MarketDump Writeup"
date: 2020-10-27 15:28:00 +0530
categories: [HackTheBox, Challenges]
tags: [pcap, wireshark, forensics]
image: /assets/img/Posts/MarketDump.png
---

> We have got informed that a hacker managed to get into our internal network after pivoting through the web platform that runs in public internet. He managed to bypass our small product stocks logging platform and then he got our costumer database file. We believe that only one of our costumers was targeted. Can you find out who the customer was? 


# Task Overview

- 1
- 2
- 3
## Initial Analysis

Looking over the protocol hierarchy we see that a majority of the network traffic occurs over the TCP, in particular HTTP and an unreckognised Data. 

Therefore, lets take a look at what WireShark did not categorise.

![protocol](/assets/img/Posts/MarketDump/protocol.png)

Applying the filter to the identified protocol, we look at the conversations taking place:

![conversations](/assets/img/Posts/MarketDump/convo.png)

We see that there was some traffic between the host `10.0.2.3` and `10.0.2.15`. Lets take a look a further look.

## Host Compromise (TCP)

The first bit of traffic shows some ping requests between the aforementioned hosts. Proceeding this the unidentified host `10.0.2.15:34976` establishes a connection to `10.0.2.3:9999`.

Following the TCP stream we can see that we have located the C2, with the adversary attempting to locate the customer database file.
![tcpstream1](/assets/img/Posts/MarketDump/tcpstream1.png)

After quickly scanning through it was immediately obvious which customer was targetted and I looked for further C2 between the machines. With no luck I took another look at the output and noticed:

![tcpstream2](/assets/img/Posts/MarketDump/tcpstream2.png)

Here we can see on the customer numbers has some encoding or encryption. I wasn't familiar immediatly  so I utilise `CyberChef.com` and the magic wand predicts Base58:

![flag](/assets/img/Posts/MarketDump/flag.png)

And we have the flag! BQ




---
title: "CTF — Down The Rabbit Hole"
date: 2020-11-19 00:34:00 +0530
categories: [CTF, Steg, Encoding]
tags: [CyberChef, binwalk]
image: /assets/img/Posts/Rabbit/RabbitHole.png
---

> CTF Guide on how to find the flag for Down The Rabbit Hole 2.

## Hole 1
Opening up the provided image doesn't provide anything usefull. Other than the fact from the challenge description that there will be lots of files within files.. backed up with the theme of down the rabbit hole meme from Alice in Wonderland.

First I always like to check out a images metadata with `exiftool file` :

``` shell

# exiftool Down_the_rabbit_hole_2.jpg

ExifTool Version Number         : 12.08
File Name                       : Down_the_rabbit_hole_2.jpg
Directory                       : .
File Size                       : 2024 kB
File Modification Date/Time     : 2020:11:19 17:55:14+08:00
File Access Date/Time           : 2020:11:19 17:55:41+08:00
File Inode Change Date/Time     : 2020:11:19 17:55:16+08:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Exif Byte Order                 : Little-endian (Intel, II)
X Resolution                    : 300
Y Resolution                    : 300
Resolution Unit                 : inches
Software                        : GIMP 2.10.20


```

Nothing too exciting here, next I check out for hidden files due to the challenge hints with `binwalk filename`

``` shell

# binwalk Down_the_rabbit_hole_2.jpg 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
30            0x1E            TIFF image data, little-endian offset of first image directory: 8
292           0x124           JPEG image data, JFIF standard 1.01
1933578       0x1D810A        Zip archive data, encrypted at least v2.0 to extract, compressed size: 138532, uncompressed size: 570564, name: rabbit_hole.txt
2072290       0x1F9EE2        End of Zip archive, footer length: 22


```

So multiple files in there, lets drop them... `binwalk -e Down_the_rabbit_hole_2.jpg `

``` shell
# ls
1D810A.zip  rabbit_hole.txt

```
## Hole 2

Two files, the text file appears empty and the zip is password protected. Looking back at the `binwalk` extraction, it appears that a few files identified weren't extracted! I could try carve the image file out manually at `0x124`, but lazy is better so lets check `icyberchef.com`.


![rabbithole](/assets/img/Posts/Rabbit/RabbitHole1.png)

Bingo! Lets have a look...

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole2.png)

Password for the zip perhaps? Turns out it is, `backwardsisforwards` is the password to the zip file. It contains the text fille `rabbit_hole.txt`... having a look:

``` shell
# cat rabbit_hole.txt |less

=oAI6U0YyZFdgojR5Nlb2ByOuNncHBHIvR3bnB {truncated}

```

Shows a massive string that starts with a very interesting symbol. It appears that it is a base-64 string in reverse, and a follow on from the password hint. So reversing the string and decoding the base-64 string we get..

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole3.png)

## Hole 3

So a `.php` script now... having a closer look at the entire contents of the script shows some Alice in Wonderland dialogue and some very strange looking script. At first glance it looks obfuscated, therefore it is worth throwing it at an online `php` de-obfuscation tool.

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole4.png)

Inside we can see another monster string! This one is clearly hex.. lets see what we can do.


## Hole 4

Not going to lie, this one took longer than we would like to admit.. but converting the hex gives garbage at first, but scrolling down we can see some clear text!

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole5.png)

That is no accident, so we export the output and see if we can identify what it is exactly. Loading it into trusty cyber chef we can detect the file type is actually a `7z`!

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole6.png)

## Hole 5

Extracting the file we find that it contains `birtday_cake.db`.. running a quick check

``` shell

# file birthday_cake.db 

birthday_cake.db: SQLite 3.x database, last written using SQLite version 3032002

```

We see it is infact an SQLite database. Opening the file in `https://inloop.github.io/sqlite-viewer/` we can see what is inside:

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole7.png)

There are 5 strings, or layers of the cake.. so it makes sense to string them together in order. Clearly this is again a large string of hex.

## Hole 6

With all the layers of the cake put together, we can repeat the process from above and put it in cyber chef. Loading and converting from hex shows again garbage ASCII. This time there is no messing about and we download the file and identify it as an image! Infact looking at the HEX and the ASCII output it looked like some kind of bitmap anyway, a bit more obvious than previously.

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole8.png)

And bingo.. on to another rabbit hole (we getting close!)


## Hole 7

Ok so after we get our image file we find this

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole9.png)

Now, it looks like a chopped up QR code that can be reassmbled.. but we always try the lazy option first.. that is CyberChef. So I whack the image into cyberchef and parse the QR code...

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole10.png)

BINGO! I am not sure if this was intended, but we were happy with it.

## Hole 8

So it is a bit of rinse and repeat at this stage, again we parse the hex and detect the file type and this time we get a `.bz2` zip file.

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole11.png)

## Hole 9

Unzipping the file and displaying the contents shows us:

``` shell
ls
birthday_cake.db  Down_the_rabbit_hole_2.jpg  _Down_the_rabbit_hole_2.jpg.extracted  image  image.bz2  output  rabbit2.7z
b3nny@kali:~/Downloads/rabbit2$ cat image
FourKiloAlphaJulietYankeeVictorThreePapaOscarVictorWhiskeyGolfIndiaIndiaDeltaZuluNovemberFiveTwoSierraAlphaFiveDeltaFoxtrotNovember {truncated}


```
Nato phonetic.. so back to an online tool to do the translating for us. There are some dodgey online tools, `https://www.dcode.fr/nato-phonetic-alphabet` did the trick for me.

And we get the following string:

``` shell
4KAJYV3POVWGIIDZN52SA5DFNRWCA3LFFQQHA3DFMFZWKLBAO5UGSY3IEB3WC6JAJEQG65LHNB2CA5DPEBTW6IDGOJXW2IDIMVZGKP7CQCOQVYUATRKGQYLUEBSGK4D
FNZSHGIDBEBTW633EEBSGKYLMEBXW4IDXNBSXEZJAPFXXKIDXMFXHIIDUN4QGOZLUEB2G6LHCQCOSA43BNFSCA5DIMUQEGYLUFYFOFAE4JEQGI33O4KAJS5BANV2WG2
BAMNQXEZJAO5UGK4TF4KAJHYUATUQHGYLJMQQEC3DJMNSS4CXCQCOFI2DFNYQGS5BAMRXWK43O4KAJS5BANVQXI5DFOIQHO2DJMNUCA53BPEQHS33VEBTW6LHCQCOSA
43BNFSCA5DIMUQEGYLUFYFOFAE44KAJG43PEBWG63THEBQXGICJEBTWK5BAKNHU2RKXJBCVERJM4KAJ2ICBNRUWGZJAMFSGIZLEEBQXGIDBNYQGK6DQNRQW4YLUNFXW
4LQK4KAJYT3IFQQHS33V4KAJS4TFEBZXK4TFEB2G6IDEN4QHI2DBOQWOFAE5EBZWC2LEEB2GQZJAINQXILBA4KAJYZTMMFTXW2LGL54W65JNN5XGY6JOO5QWY227NQY
G4ZZNMVXG65LHNB66FAE5

```

## Hole 10 (Final)

Right so this string wasn't familiar to me, so one trick I do is use the magic wand on CyberChef.. cheating I know.. but lets see what we get:

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole12.png)

And the app has detected that it is most likely Base32 encoding.. okay lets try.. 



``` shell

“Would you tell me, please, which way I ought to go from here?”
“That depends a good deal on where you want to get to,” said the Cat.
“I don’t much care where–” said Alice.
“Then it doesn’t matter which way you go,” said the Cat.
“–so long as I get SOMEWHERE,” Alice added as an explanation.
“Oh, you’re sure to do that,” said the Cat, “flag{if_you-only.walk_l0ng-enough}”

```

Got him :D

Was a journey to get here!




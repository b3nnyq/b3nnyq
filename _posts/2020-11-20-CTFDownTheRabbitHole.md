---
title: "CTF — Down The Rabbit Hole"
date: 2020-11-20 00:34:00 +0530
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

Two files, the text file appears empty and the zip is password protected. Looking back at the `binwalk` extraction, it appears that a few files identified weren't extracted! I could try carve the image file out manually at `0x124`, but lazy is better so lets check `icyberchef.com`.


![rabbithole](/assets/img/Posts/Rabbit/RabbitHole1.png)

Bingo! Lets have a look...

![rabbithole](/assets/img/Posts/Rabbit/RabbitHole2.png)

Password for the zip perhaps?




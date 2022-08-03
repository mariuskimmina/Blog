---
title: Vulnhub Pinkys Palace
published: false
---

# 0x0 Introduction
![image](/assets/images/thumb.png "Setup")


# 0x1 Setup

I use KVM/Qemu with virt-manager as a GUI to setup my virtual maschines, for anyone that's using Linux as his Host System this is probably the best option.  
My simple Network configuration looks like this 

![image](/assets/images/pinkys-network.png "Setup")
![image](/assets/images/pop-network.png "Setup")

For the attacking System I'm using a copy of Pop-OS with various tools manually installed to tackle this challenge.

# 0x2 Getting started

As usual I start using nmap

```
$ nmap -sC -sV -oN recon/nmap 192.168.122.33
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-05 11:49 CEST
Nmap scan report for Pinkys-Palace (192.168.122.33)
Host is up (0.0010s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.10.3
|_http-server-header: nginx/1.10.3
|_http-title: Pinky's Dev Box

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.81 seconds
```

Visiting the Website we find a php-info page, which tells us that the server is running PHP version 7.0.30
![image](/assets/images/pinky-php.png "Setup")

I also ran gobuster, hoping to find more content on the Webserver, with only limited success tho

```
$ sudo gobuster dir -u 192.168.122.33 -w /opt/SecLists/Discovery/Web-Content/common.txt 
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://192.168.122.33
[+] Threads:        10
[+] Wordlist:       /opt/SecLists/Discovery/Web-Content/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/06/05 12:22:13 Starting gobuster
===============================================================
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/.hta (Status: 403)
/index.php (Status: 200)
===============================================================
2020/06/05 12:22:14 Finished
===============================================================
```

At this point I decided that further recon was required an ran a second nmap-scan, this time on all ports

```
$ nmap -p- -oN recon/all-ports 192.168.122.33
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-05 12:32 CEST
Stats: 0:00:52 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 40.94% done; ETC: 12:34 (0:01:15 remaining)
Nmap scan report for Pinkys-Palace (192.168.122.33)
Host is up (0.00083s latency).
Not shown: 65533 filtered ports
PORT      STATE SERVICE
80/tcp    open  http
65535/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 103.20 seconds
```

Yes indeed, pinky has another custom Webserver running on port 65535, going to that page we see:
![image](/assets/images/pinky-server.png "Setup")

Since I have done quite a few CTF before this one, my first guess from experience was that a custom webserver will probably be vulnerable to path-traversal, so I fired up burp and found out that I was right indeed.  
Which means we are now able to read files on the server
![image](/assets/images/pinkys-traversal.png "Setup")

Notice that the server answers has an Header *Server: pinkys-HTTP-server* which was a hint for us that this is also the name of the binary running the Server. We can Download this binary with curl

```
$ curl 192.168.122.33:65535/pinkys-HTTP-server --output ~/vulnhub/pinkys/server
```

And get further information about it using 'file'

```
$ file server 
server: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=74b82688f1a7fe82a9f460741572bd29b9d10eaa, stripped
```

I'm using Ghira (here link) to analyze the binary.  
The first thing I noticed was that the Server changes his uid and gid to 0x538 (which is 1338 in dezimal), which is the phs user that we have found before in the /etc/passwd file. Since it's setting his uid and gid to his user it must start as another one, probably root. Just something to keep in mind.  
![image](/assets/images/pinkys-setids.png "Setup")

We can also Identify how this server handles requests, It checks if we send a HEAD, or GET Request and if neither happens to be the case it is going to send us a '400 Bad Request'
![image](/assets/images/Only_Head_or_GET.png "Setup")

If it is a GET or HEAD Request it will be directly handed over to an open() call, which explains the path traversal.
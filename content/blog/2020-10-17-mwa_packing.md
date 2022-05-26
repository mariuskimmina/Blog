---
title: Malware Analysis - Packing
author: Marius Kimmina
date: 2020-10-17 14:10:00 +0800
categories: [Malware Analysis, Packing]
tags: [Malware, Packing]
published: true
---

## 0x0 Introduction
In this series of Blog Posts about Malware Analysis I will take a closer look at common techniques and tricks used by Malicious Software and analyse different Malware samples. 
This first post will focus on `packing` or `executeable compression`, a technique often used by malware to hide it's malicious code from security-software and researchers.


## 0x1 What is packing and how does it work?
When a Binary is packed you can think of it like a .zip archieve where you first have to extract the content out of the .zip file to view it. When a binary is packed it also first needs to be unpacked
before it can be executed. What can often be confusing about this part is, that the packed executeable, is still an executeable and not some kind of archieve format. so when you receive a packed executeable
you can execute it right away, and this execution will handle the unpacking for you. Some malware might even use nested packing, where the dropper first unpacks another packed 
executeable, which in turn might unpack another packed executeable, there is virtually no limit to this.  

In a nutshell the packing process looks something like this:
![image](/assets/images/malware/packing/packing-in-a-nutshell.png "packing")

and the unpacking in pseudo-code then may look like this:

```
E = [encoded Malware]
K = Key
M = decode_malware(M,K)
malware_address = load_in_memory(M)
jump_to(malware_address)
```

The malicious code is saved as a variable (possibly encrypted) and once the packed exe is being run, this code will be written into memory and execution gets transfered to it.  
There are a couple of different techniques that the malware can use to unpack itself once the dropper is on the system. For this first blog Post I want to give a very
highlevel overview of these different techniques, so please keep in mind that in pratice there are more details to each of these techniques then what I will be covering here.

## 0x2 DLL Injection
DLL Injection is the easiest method in this list, a (malicious) Program can drop a DLL file to disk and then use `CreateRemoteThread` to call `LoadLibrary` in the target process.
This technique is rarely used by more sophisticated malware as they want to avoid dropping any files to disk, because an AV engine might detect the DLL as malicious.
![image](/assets/images/malware/packing/DLL-injection.png "DLL injection")

## 0x3 Self-Injection
The single packed executeable that is dropped onto the system, thus often called dropper, contains 2 different code sections in it, the `stage code` and the `malicious code`.
The dropper is going to call VirtualAlloc to allocate memory and then write the `stage code` to this newly allocated memory section. It will then transfer execution to this new memory section.
the `stage code` is now being executed, this will again call VirtualAlloc and decrypt the malicious code and write it into this second block of allocated memory. Then the `stage code` is going to 
change the permissions on the PE section of the dropper, so that it can be written to. Now the malicious Code will be written to the dropper and execution is transfered back to the dropper, which now contains the malicious code.

## 0x4 Process-Injection
Remote process injection, also called 'Process Hollowing', describes a process that spawns a new process in a suspended state and replaces the benign code of this new process with the malicious code.
Once the malicious code is succesfully loaded into memory, the process is resumed and the malware starts to execute. I have a added a simple image to help clear things up.
![image](/assets/images/malware/packing/remote-injection.png "remote-injection")
Note that there are a lot more details that have to be fullfilled for this to work, but the goal of this post is to provide a high level overview of the exesting techniques.
I will dive into much more detail in the following posts



## 0x5 Thread Execution Hijacking
This technique is very similar to Process-Injection (or Process Hollowing) but instead of creating a new process in a suspended state, this technique trys to take over an existing process.
If succesfully, this can have 2 advantages over Process-Injection. The highjacked process could have more privileges thus enabling the malware to run with more privileges.
Besides that, the highjacked process might also be whitelisted by AV engines or other security solutions that may exist on the system.



## 0x6 Detecting packers
If you have read this far, one question might have come to your mind, why can't we just detect if an executeable is packed and if so, block it because it's gotta be malicious, right?
Well, there are still some legacy benign programs that are packed without any malicious intend, these packers were first intended to reduce the size of executeables to save space on disk.
Nowadays diskspace is not much a concern, if any but some older harmless programs might still be packed regardless. But that's not the only issue, if it were we could probably just create a whitelist for these programs.
The other, bigger problem is that it's hard to detect custom packers, most sophisticated malware will not use standard packers (I am going to talk about those in the next section). They will be packed by custom packers created only for this malware.
Thus a signature based approach of detecting packers hardly works. Some antivirus solutions do maintain a signature base of known malicious packers with the hope of catching some lazy malware authors that way, 
but for any sophisticated malware campaign this approach won't be effective.


## 0x7 Common examples
some common packers are:
* UPX
* ExeStealth
* VMProtect
* PESpin
* FSG
* MPRESS

With UPX being by far this most common of all of these.  
But as I said before, most sophisticated malware will bring it's own custom packer.

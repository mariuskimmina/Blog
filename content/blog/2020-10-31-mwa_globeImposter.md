---
title: Malware Analysis - unpacking GlobeImposter Ransomware 
author: Marius Kimmina
date: 2020-10-31 14:10:00 +0800
categories: [Malware Analysis, Packing]
tags: [Malware, Packing, Ransomware]
published: true
---

## 0x0 Introduction
In this post I'm going to show you how to extract the real executeable, that is the actual malware, from a packed sample of the GlobeImposter Ransomware.  
If you read the last post in my 'Malware analysis' series than you should already have a basic understanding of what a packed executeable is.
If not than please go and read that one first, other wise this might not make all that much sense to you.  

Hashes:  
* MD5: 612974dcb49adef982d9ad8d9cbdde36  
* SHA1: b817e361bd0cc1819d7f6a1189f0f5d56ed48721  
* SHA256: 13e164380585fe44ac56ed10bd1ed5e42873a85040aee8c40d7596fc05f28920  

## 0x1 Detecting that an executeable is packed
One method that has been identified as quite successful in identifying a packed executeable is to look at it's entropy, that is the level of randomness of the file.
In a packed executeable the malicious code is generally encrypted and only being decrypted at runtime, thus when we look at a packed executeable the entropy of a
packed executeable will often be significantly higher than a normal executeable due to the encrypted content.  
For our convenience there are already a few tools that do a good job at telling us the entropy of a file, one of them is "Detect it Easy (DiE)" which wil give us a  
nice graph view of the entropy for  the file.
![image](/assets/images/malware/globeImposter/entropy.png "entropy")

As you can see here, the sections .text and .rsrc have anusual high amounts of entropy, which leads to the conclusion that they are probably(!) packed.
The .text section contains the acutal code of the program and the .rsrc section contains all the resources that the program uses, a high entropy in these sections
hardens the suspision even further, because the encrypted malicious code is going to be placed into some variable which will of course be part of the .text section.


## 0x2 Dynamic unpacking
What I am going to show you now is a technique known as `dynamic unpacking` and its called dynamic because we are actually going to run the packed executeable.  
If you are going to do this yourself make sure you use a VM for this and that you created a snapshot of your VM before starting the execution. For extra safety
you might also want to disconnect your VM from the internet and isolate it completely. The goal here is not to actually let the malware run havoc but much rather
we are going to attach a debugger to it, find the point at which the actual executeable is being loaded into memory, set a breakpoint there and than dumb the 
memory section that contains the malware to disk. In an ideal case the malicious code is not being run and our VM is still fine afterwards but mistakes can 
easily happen during this procedure so it's definetly better to be careful.

## 0x3 Unpacking GlobeImposter
I will be using x32dbg to unpack this ransomware sample but any other debugger should also be capable of doing this. 
Now I'm going to go through this step by step, so there will be a lot of screenshots coming your way from here on, be prepared!  

So we start x32dbg and as nice as this program is, it has already set a breakpoint at the entrypoint for us, so this were we start
![image](/assets/images/malware/globeImposter/entrypoint.png "entrypoint")

We don't really care about the entrypoint to much tho, we want to find the main function or `WinMain`.  
An easy way to get there is to go into the graph view for this entrypoint function and look for a push of the base address followed by a function call.
![image](/assets/images/malware/globeImposter/find-winmain.png "winmain")

It's safe to assume that this function will be `WinMain`. If you look into the function definition of `WinMain` you will see that this argument, the base address is described as

```
hInstance is something called a "handle to an instance" or "handle to a module." 
The operating system uses this value to identify the executable (EXE) when it is loaded in memory. 
The instance handle is needed for certain Windows functions—for example, to load icons or bitmaps.
```

Now when we continue to look at winmain in the graph view, we immediately see something intreseting. A bunch of function calls are being made here with nothing but 0s as arguments.
This is a technique used by malware to look more like a normal program, because now when we look at the function the program uses it's not just these 'probably malware' functions but
also a lot of functions used all the time by normal benign programs. None of these functions is actually going to do anything tho, because the arguments are all 0s.
![image](/assets/images/malware/globeImposter/useless-functions.png "useless functions")

While that was some intreseting obfuscation technique, lets get back to the unpacking. With packed executeables you often want to look near the end of the function, because near the return
will actually be a jump to another code segment, and that's what we are looking for here.
You will find these 'weird' calls to some kind of memory segment and another call to ebp - 14.
![image](/assets/images/malware/globeImposter/weird-function-call.png "weird calls")

But there is more to be seen here, besides these weird calls they are also building an interesting string here, lets have a closer look
![image](/assets/images/malware/globeImposter/virtualprotect-stack-string.png "VirtualProtect Stack String")

They are building the string "VirtualProtect" which is a win32 API call to change the permissions of a memory region. Microsoft describes VirtualProtect as follows:
```
Changes the protection on a region of committed pages in the virtual address space of the calling process.
```

This is important to us because if the malicious code is written into memory, the memory region that it's located in has to be executable.
Knowing this, lets continue execution of the program till we had the breakpoints that I have set on the 2 'weird calls'. If you now go back into
the disassembler view, you will see that the first call is less weird now, x32dbg has evaluated for us that this is in fact a call to VirtualProtect.
![image](/assets/images/malware/globeImposter/virtualprotect-evaluated.png "VirtualProtect evaluated")

and if we now also take a look at the stack, to see which arguments are being parsed to `VirtualProtect` then we can see that it changes the permissions to Execute, Read and Write.
![image](/assets/images/malware/globeImposter/virtualprotect-arguments.png "VirtualProtect arguments")

Because if we look at the documentation for `VirtualProtect` then we can see that the third argument, 0x40, in our example is the `flNewProtect` and the constant 0x40 means:
![image](/assets/images/malware/globeImposter/virtualprotect-constants.png "VirtualProtect constants")

Next we are going to follow the jump to `ebp - 14` and this function is really to big to take some meaningfull screenshots but if you go through it, what you will see is that
it's building a lot of win32 API strings thus we can assume that this is still setup code and we have not yet reached the malicious code. So what we are going to do next is to 
look for another jump to a register or memory region and we find one right at the end of this function
![image](/assets/images/malware/globeImposter/jump-eax.png "jump eax")

If we follow this jump, you will find anohter immediate jump, follow that one as well and you find the Original Entrypoint (OEP) for the the malicious code
![image](/assets/images/malware/globeImposter/oep.png "OEP")

Now we want to extract the malware as a standalone PE file and x32dbg has super cool tool for that called scylla, once you open the plugin make sure that it has OEP set correctly and press "IAT Autosearch" and wait for it to finish.
When it's done you can press "Dump" and save the unpacked executeable. 
![image](/assets/images/malware/globeImposter/scylla.png "scylla")

We can load the new .exe into *Detect it Easy* again and confirm that it's not packed anymore.
![image](/assets/images/malware/globeImposter/unpacked.png "unpacked")

With that we are done here for this post, we have successfully unpacked the GlobeImposter Ransomware. 


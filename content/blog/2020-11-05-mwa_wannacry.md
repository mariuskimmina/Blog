---
title: Malware Analysis - WannaCry Part 1
author: Marius Kimmina
date: 2021-03-08 14:10:00 +0800
categories: [Malware Analysis]
tags: [Malware, WannaCry]
published: false
---


## 0x0 Introduction
First a little disclaimer, this is mostly based on the work of "stack smashing / ghidraninja", there is a link to his youtube channel (and all other ressources that I used) at the end of this post
It's also noteworthy that wannacry, even though it was one of the most devastating ransomware attacks that we have ever seen, hardly used any obfuscation or other anti-reverse-engineering techniques.
The authors probably just didn't care about it being analysed, that makes it a very good target for new Malware analysts to learn from.

## 0x1 WinMain
To start this off we are going to find the WinMain function, we can look at the entry function that ghidra has already identified for us and see these calls to GetStartupInfoA and GetModuleHandleA followed by another function call
right before the exit. We can thus be relatively certain that this function is in fact `WinMain`.
![image](/assets/images/malware/wannacry/find-winmain.png "find winmain")

Now that we have Identified `WinMain` we can help ghidra analysing by providing the function Signature.
You can simple look at the Microsoft Documentation to find that the signature for WinMain is: `int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PWSTR pCmdLine, int nCmdShow);`.
Just right click on the function call in ghidra and click "Edit Function Signature"
![image](/assets/images/malware/wannacry/winmain-signature.png "winmain signature")

Now that ghidra is aware of this function signature it has already further cleaned up the entry function.
![image](/assets/images/malware/wannacry/call-to-winmain.png "call to winmain")

Why was it so important to identify WinMain? Because that's genereally where we want to start analysing, you can of coures go line by line through all of the entry function setup code, but that's
gonna take forever. In reverse engineering one of the most important things is knowing what to look so that you don't get lost in boilerplate code or obfuscation. 

## 0x2 The Infamous Killswitch
You are reading this which means you probably have an overall interest in Malware and reverse engineering and if that's the case then you have probably already heard the story of the killswitch but if 
not, here it is. WannaCry had successfully spread through Europe and Asia like wildfire by abousing a Vulnerability in Windows Systems called EternalBlue. This Vulnerability allowed WanaCry to spread
without any human interaction, a patch for the Vulnerability already existed at the time but many systems were still unpatched and older systems, like Windows XP, didn't receive any patch in the first
place. When the danger of WannaCry became apparent, Microsoft even went out of there way of not supporting Windows XP any more and delivered an *Emergency Patch*. This emergency patch wasn't the only
thing that slowed the malware down, there was also MalwareTech's happy little accident. MalwareTech is an, at that time little kown, Malware analyst and reverse engineer who discovered that WannaCry
would check if some weirdly named URL is reachable, he registered that domain and as it turns out, that was enough to shut the whole thing down. That's $10.69 well invested.  
Enough of the story though, lets get technical again by taking a closer look at WinMain.
![image](/assets/images/malware/wannacry/WannaCry-Killswitch.png "call to winmain")

Here we can identify the call to that gibberish URL and if it failed to reach that URL then another function will be executed, otherwise the program is just going to exit.
I have cleaned up the decompilation output by providing ghidra with function signatures for InternetOpenA and InternetOpenUrlA also renamed some variables and added commands to make the output much
more readable
![image](/assets/images/malware/wannacry/WinMainClean.png "Cleaned up WinMain")





## 0xA Ressources

---
title: IPsec - How does it really work
author: Marius Kimmina
date: 2021-03-08 14:10:00 +0800
categories: [How does it really work]
tags: [docker, linux, tomcat]
published: false
---

## 0x00 Introduction to IPsec

The IP protocol was created without security in mind, it is a connectionless protocol that offers no protection against:

- spoofing of IP packets
- eavesdropping on IP packets
- manipulation of IP packets
- replay attacks

IPsec is an extension of the IP protocol with the goal of creating secure connections between two IP endpoints on OSI-layer three. This secure connection can then be used for all kinds of applications. The applications itself do not have to adjust at all.

## 0x01 Overview of IPsec

IPsec has been first introduced in 1995 by the IPsec Working Group.
The protocol is supposed to offer:

- Authentication
- Confidentiality
- Integrity
- Protection against Replay-attacks
- Key Management

IPsec has a quite complex architecture, consisting of 3 main parts:

- Encapsulation Security Payload (RFC 4303)
- Authentication Header (RFC 4302)
- Key Management
    - Internet Key Exchange (RFC 2409)
    - Internet Key Exchange v2 (RFC 7296)

On top of that it also support two modes of operation:

- Transport Mode
- Tunnel Mode

And it comes with 3 different Databases:

- Security Policy Database (SPD)
- Security Association Database (SAD)
- Peer Authorization Database (PAD)

## 0x02 IPsec Modes

### Transport Mode

IPSec in transport mode is meant to create a secure connection from one host to another.
If someone is eavesdropping on this connection, the attacker still knows exactly which two hosts are communication with each other.

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled.png)

### Tunnel Mode

IPSec in transport mode on the other side is meant to create a secure connection from one network to another. A prominent use case would be to connect two company sites to each other.
if someone is eavesdropping on this connection, the attacker only knows which network the traffic is coming from and which network it is going to, he has no idea which hosts are taking part in this communication.

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%201.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%201.png)

## 0x03 Authentication Header (AH)

The authentication Header offers:

**1. Integrity of header and data**

One part of the authentication Header is an "Integrity Check Value" (ICV) which is an HMAC of IP Header, Authentication Header and Data being sent. This value will tell the receiver if any part of the IPsec packet has been manipulated.
The receiver will also calculate the HMAC value of IP Header, AH and Data. If his HMAC matches the ICV then the packet has not been tampered with.

**2. Authentication of the sender**

Since the IP Header is also part of the Integrity Check Value, the authentication of the sender is verified as long as the ICV value is correct.

**3. Protection against replay-attacks**

Sequence numbers have long been established as an effective prevention of replay-attacks, they can be found in all kind of protocols and the authentication header is no exclusion. If the sender receives the same sequence number twice, he knows that something went wrong.

### AH Transport Mode

Green: Integrity Protection

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%202.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%202.png)

### AH Tunnel Mode

Green: Integrity Protection

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%203.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%203.png)

## 0x04 Encapsulating Security Payload

The Encapsulating Security Payload offers:

**1.** **Confidentiality of the Data**

This is the most important difference between ESP and AH, ESP actually encrypts the data that is being send

**2. Authenticity and Integrity of the Payload**

**3. Protection against Replay-Attacks**

### ESP Transport Mode

Red: Encrypted Data

Green: Integrity Protection

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%204.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%204.png)

### ESP Tunnel Mode

Red: Encrypted Data

Green: Integrity Protection

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%205.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%205.png)

## 0x05 Protection against Replay-Attacks

Sequence numbers are generated for each "Security Association" and they start from 0.
The Sender increments the sequence number by 1 for every packet send and writes the value into the field "Sequence Number".

The Receiver checks all incoming packets using the "sliding window" principle

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%206.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%206.png)

## 0x06 Combining AH and ESP

Blue: ESP Tunnel

Green: AH Transport

![IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%207.png](IPsec%20-%20How%20does%20it%20really%20work%2053ca64350cd343918b60466a24efb238/Untitled%207.png)

## 0x07 Security Policy Database (SPD)

This Database contains rules on how IPsec will treat IP packets. The rules in this database will be gone through by "match first" principle. For each IP packet IPsec will decide on one of these 3 actions:

- DISCARD
- BYPASS
- PROTECT

DISCARD means that the packet will be dropped immediately, BYPASS means that the packet will be forwarded immediately and PROTECT means that IPsec will be applied and the packet will be protected.

## 0x08 Security Association Database (SAD)
If the SPD decided that an IP packet is to be protected, then the SAD will determine how to protect the packet. 
for that, it uses Security Associatons, which are policies for the communication between peers.
Essentially, this database saves all information about what a connection between peers should looke like, such as which algorithms should be used, which keys have been exchanged, etc.

## 0x09 Peer Authorization Database
This final database contains all information regarding authentication and authorization of systems that want to communicate via IPsec

## 0x0A Internet Key Exchange


## 0x0B Internet Key Exchange v2

## 0x0C Summon it up
---
title: Adventours in selfhosting DNS
author: Marius Kimmina
date: 2022-06-03 14:10:00 +0800
tags: [DNS, Infra]
published: false
---


When you visited this domain, mariuskimmina.com, then you just used my own selfhosted DNS Server (assuming that you didn't have the IP in your local DNS cache).
For the most part, I did this to gain a deeper understanding of DNS. Knowing DNS well and having experimented with it before definetly comes in handy when you 
work in IT-Infrastructure.

### Self Hosted

Pros:

* You won't be affected by outages of your DNS Provider, like [the cloudflare outage in 2020][cloudflare] for example. Services like google or cloudflare's DNS are prime targets for DDOS attacks.
* Full control 
* Learning opportunity

Cons:
* You have to update/maintain the DNS server.
* If you run DNS and your Website on the same server, that's additional load you have to thing about.
* If you run your DNS and your Website on different servers, that's additional cost.
* While you are less likely to be targeted by a DDOS attack, you also have a much harder time protecting yourself form one.

---

Prerequisies:
* Owning a Domain name
* A linux server with a publicly accessbile IP
    * Cloud Providers like Digital Ocean or Linode work well

---

![image](/blog/Selfhosting-DNS/StackOverflow-GlueRecords.png "StackOverflow to the rescue")

---

```
$ORIGIN mariuskimmina.com.     ; designates the start of this zone file in the namespace
$TTL    604800
@   IN  SOA ns1.mariuskimmina.com. marius.mariuskimmina.com. (
                            2022072803      ; Serial
                                604800      ; Refresh
                                 86400      ; Retry
                               2419200      ; Expire
                                604800 )    ; Negative Cache TTL


                        IN  NS  ns1.mariuskimmina.com.
                        IN  NS  ns2.mariuskimmina.com.
                        IN  A   75.2.60.5

ns1.mariuskimmina.com.        IN  A   207.154.249.62
ns2.mariuskimmina.com.        IN  A   134.209.236.227
blog.mariuskimmina.com.       IN  A   75.2.60.5
www.mariuskimmina.com.        IN  CNAME mariuskimmina.com.
```

---

```
$ kdig -d @ns1.mariuskimmina.com mariuskimmina.com
;; DEBUG: Querying for owner(mariuskimmina.com.), class(1), type(1), server(ns1.mariuskimmina.com), port(53), protocol(UDP)
;; ->>HEADER<<- opcode: QUERY; status: NOERROR; id: 6412
;; Flags: qr aa rd; QUERY: 1; ANSWER: 1; AUTHORITY: 2; ADDITIONAL: 0

;; QUESTION SECTION:
;; mariuskimmina.com.           IN      A

;; ANSWER SECTION:
mariuskimmina.com.      604800  IN      A       75.2.60.5

;; AUTHORITY SECTION: mariuskimmina.com.      604800  IN      NS      ns1.mariuskimmina.com.
mariuskimmina.com.      604800  IN      NS      ns2.mariuskimmina.com.

;; Received 172 B
;; Time 2022-07-28 11:08:50 CEST
;; From 207.154.249.62@53(UDP) in 20.3 ms
```


### References
https://mattgadient.com/hosted-vs-self-hosted-dns-servers/


[script]: https://github.com/mariuskimmina/.dotfiles/blob/main/bin/.local/bin/pmux
[cloudflare]: https://blog.cloudflare.com/cloudflare-outage-on-july-17-2020/

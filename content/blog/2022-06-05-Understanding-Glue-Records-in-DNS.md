---
title: Adventours in selfhosting DNS
author: Marius Kimmina
date: 2022-06-03 14:10:00 +0800
tags: [DNS, Infra]
published: false
---


When you visited this domain, mariuskimmina.com, then you just used my own selfhosted DNS Server (assuming that you didn't have the IP cached).


## Why run your own DNS servers?
I am doing it to learn more about DNS, and also about running a highly available, globally distributed service. If you relly want to get into the technical
pros and cons about it I recommend you to read through the [References](#references) at the end of this post.

## Prerequisites

* Owning a Domain name
* A linux server with a publicly accessbile IP
    * Cloud Providers like Digital Ocean or Linode work well

---


## The Basics

I am using [CoreDNS][CoreDNS].

## Using the hosts plugin

```
mariuskimmina.com {
        log
        hosts {
                75.2.60.5 mariuskimmina.com
                207.154.249.62 ns1.mariuskimmina.com
        }
}
```



## Using a Zone file

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

## Setting Custom DNS Servers at my Domain registrar

![image](/blog/Selfhosting-DNS/StackOverflow-GlueRecords.png "StackOverflow to the rescue")

## Querying my DNS servers

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

## DNS over TLS and DNS over HTTPS
As you have seen, I am using CoreDNS as my DNS server. One of the reasons for that is that I like the plugin architecture of it and 
I have also once [written a plugin for CoreDNS myself][my-plugin]. The plugin that I made uses the [ACME][ACME] protocol to obtain and renew
tls certificates without any user interaction, as long as you own a domain and you have setup CoreDNS as the authoritative DNS server for it, 
which is exactly what we did in this blog, then you are able to provide DNS over TLS and DNS over HTTP pretty easily.


### References
https://mattgadient.com/hosted-vs-self-hosted-dns-servers/
https://www.reddit.com/r/selfhosted/comments/asd08x/do_you_selfhost_authoritative_name_server/
https://howdns.works/ep1/
https://hugopeixoto.net/articles/self-hosting-nameservers.html
https://www.cloudflare.com/learning/dns/dns-server-types/
https://www.cloudflare.com/learning/dns/what-is-anycast-dns/
https://cr.yp.to/djbdns/third-party.html
https://serverfault.com/questions/23744/should-we-host-our-own-nameservers
https://www.joshmcguigan.com/blog/run-your-own-dns-servers/
https://lantian.pub/en/article/modify-website/selfhost-dns-root-server.lantian/


[my-plugin]: https://github.com/mariuskimmina/coredns-tlsplus
[ACME]: https://www.rfc-editor.org/rfc/rfc8555
[script]: https://github.com/mariuskimmina/.dotfiles/blob/main/bin/.local/bin/pmux
[cloudflare]: https://blog.cloudflare.com/cloudflare-outage-on-july-17-2020/

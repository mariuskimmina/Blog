---
title: WIP Introducing Automated TLS Certificates in CoreDNS
author: Marius Kimmina
date: 2022-09-02 14:10:00 +0800
tags: [DNS, Infra]
published: true
---

Have you ever had to setup [DNS over TLS](https://www.cloudflare.com/learning/dns/dns-over-tls/) for your Domain Nameserver?

As a Developer who normally doesn't put too much thought into infrastructure related topics, that can be a daunting task.

This post introduces a new CoreDNS Plugin that allows for fully automatic TLS certificates in CoreDNS. No more worrying about
expiring certificates and no need to setup external programms such as certbot. CoreDNS can handle it all for you.

TLDR: https://github.com/mariuskimmina/coredns-tlsplus

## Table of Contents
1. [Managing Certificates before ACME](#managing-certificates-before-acme)
2. [Introduction of ACME](#introduction-of-acme)
3. [Integration of ACME into Caddy](#integration-of-acme-into-caddy)
4. [ACME for DNS Server](#acme-for-dns-server)
5. [Who should use this plugin and when?](#who-should-use-this-plugin-and-when)
6. [How it works](#how-it-works)
7. [Requirements](#requirements)
8. [Setup](#setup)
9. [Future Work](#future-work)
10. [Final Words](#final-words)
11. [References](#references)

## Managing Certificates before ACME
Back in the days, before let's encrypt was a thing, obtaining and renewing TLS certificates required a lot of work. 
To be more percise, one had to go to all of these steps to successfully manage TLS certificates 

* Generate Private Key
* Generate CSR
* Secure Key
* Order SSL certificate
* Paste CSR into online form
* Choose an email address 
* Wait for email
* Click link in email
* Wait for another email
* Download certificate
* Concat into bundle 
* Upload to server
* Configure server
* Reload configuration
* Don't forget to renew it

(Luckily, I never had to go through this myself, thus I have taken this list from [someone who did](https://www.youtube.com/watch?v=KdX51QJWQTA))

In those days, most CAs also charged money for signing certificates. As a result, many people didn't bother providing valid
certificates for their personal websites or blogs.

Charging money for this kind of service made sense since the CA had to put in some effort to actually verify that you own
a domain before you can get a certificate for it.

## Introduction of ACME
When [Let's Encrypt](https://letsencrypt.org/) came around, things changed drastically. 

![image](/blog/tlsplus/le-logo-small.png "Let's Encrypt")

They simplified obtaining a signed certificate for your domain into the following steps

* Install certbot
* Run a certbot command (`sudo certbot certonly --standalone`)
* Configure your application

To achieve this, they created a client server protocol called [ACME](https://letsencrypt.org/how-it-works/). This protocol allows them to verify the ownership of
a domain fully automatically. While old Certificate Authoritys had to manually ensure that the their client's actaully own a domain
before issueing a certificate, Let's Encrypt was able to do this any time of the day and almost instantly. So, not only could they offer
a service for free where others have been charging money for years, they were also much faster in issueing certificates than any traditional CA. 

According to [their own stats](https://letsencrypt.org/stats/) there are close 300M Domains that use certificates issued 
by Let's Encrypt.

The use of HTTPS has increased rappidly ever since their launch. They took the human out of the loop by building
a certificate authority that could validate your domain ownership fully automatically. They created a client-server
protocol called [ACME](https://letsencrypt.org/how-it-works/), which has become an open standard. 
Let's Encrypt (or other CAs) provide an ACME server with which any client (e.g. `certbot`) can use to obtain a certificate.  

## Integration of ACME into Caddy
In 2015 the [Webserver Caddy](https://caddyserver.com/) introduced [HTTPS by default](https://caddyserver.com/docs/automatic-https). 

To achieve this, caddy itself acts as an ACME client, removing the need of a programm like `certbot` to be installed. 
While `certbot` already made things easy, there was still some room for error. 

The steps for obtaining and using a TLS certificate with caddy are as follows:

* Configure your application

No need to install any other applications you just have to setup caddy to serve on the domain that you want a certificate for 
and it will obtain (and renew) it's certificate. This also made the entire process even more reliable, since it eliminiated 
the need for `certbot` and the server to work together.

![image](/blog/tlsplus/how-it-works-caddy.png "ACME in Caddy flow")

Caddy was the first application ever to manage it's own certificates and has further revolutionized the use of encryption
on the web. 



## ACME for DNS Server
Now, the idea was that this idea that originated in caddy could also be applied to DNS servers. 

Especially when the DNS server is actually itself the authoritative DNS server for a domain, then it should
be possible for this server to obtain a certificate for this domain. That's excatly what I did and the DNS server
I did it for is [CoreDNS](https://github.com/coredns/coredns). 

CoreDNS has a plugin architecture, which means that the server by itself provides only a bare minimum of functionaliy.
The user then adds plugins to make CoreDNS fit his use case. I have adjusted the `tls` plugin so that it can act as an 
ACME client and requests certificates from Let's Encrypt (or other CAs).
![image](/blog/tlsplus/how-it-works.png "CoreDNS Plugin flow")

Certificates that are obtained this way will also automatically be renewed once more than 70% of their validity period
have passed, just like certbot would do. There is a significan't advantage here tho, since CoreDNS has to be restarted to
work with new certificates, using a new certificate from certbot still required a manual restart of CoreDNS. This new 
tls plugin is performing this restart for you and allows administrators to completly forget about managing certificates for 
CoreDNS.

## Who should use this plugin and when?
First of all, using this plugin makes it really easy to setup DNS over TLS or DNS over HTTPS. If you are a Developer who wants to setup a DNS server with encryption, you can do this easily. You shouldn't need to be a SRE or any other kind of infrastructure expert to set this up. 

As to when you want to use this pluign, there are 2 cases in which this plugin might help you tremendously. 

* You want to setup an autoritative DNS server for your Domain and support DNS over TLS or DNS over HTTPS
* You work in a very restricted network and you need an encrypted DNS forwarder on a non-standard port

### authoritative DNS
Since CoreDNS has to be the authoritative DNS Server for a domain to make this plugin work, the most obvious use case is to serve DNS over TLS or DNS over HTTPS for this particular domain. 

If you are the owner of `example.com` and you want to setup your own nameservers at `ns1.example.com` and `ns2.example.com` for example (more about setting up multiple DNS Servers later) and you want to offer DNS over TLS or DNS over HTTPS then this plugin is for you!


### Forwarding
For this to work, the CoreDNS server still needs to be setup to be the authoriative DNS server for a domain.

(using a subdomain is totally fine as well)

Instead of only answering queries about this particular domain we instead forward all DoT querys to an upstream DNS Resolver 
such Google's 8.8.8.8 or Cloudflare's 1.1.1.1 Servers.

This can be extremly useful if you want to hide your DNS traffic in very restrictive Environments. You can setup up such 
a forwarder on a custom port, such as 8853 instead of the usual 853 because in a restrictive environments that port may be blocked.

Utilizing this plugin, you setup such a server once and then forget about it since the certificate will always be renewed automatically.



## How it works
This plugin utilizes the [ACME](https://letsencrypt.org/how-it-works/) protocol to obtain certificates from a CA such as Let's Encrypt  

On startup the plugin first checks if it already has a valid certificate, if it does there is nothing to do and the CoreDNS server will start. If it doesn't (or if the certificate will expire soon) then it will initialize the ACME DNS Challenge and ask Let's Encrypt (assuming you didn't configure another CA) for a Certificate for the domain you configured  (assume it's  `ns1.example.com`). 

The plugin will also start to serve DNS requests on port 53. Let's Encrypt receives our request and sends out DNS requests for `_acme-challenge.example.com`. Since CoreDNS is supposed to be setup as the autoritative DNS server for `example.com`, these requests will reach us. The Plugin can answer those requests and in return receiv a valid certificate for `ns1.example.com`. Notice that the usage of Port 53 is mandatory for this to work, Let's Encrypt won't use any other port for challenge.


Furthermore, the plugin then starts a loop that runs in the background and checks if the certificate is about to expire. If it is, CoreDNS will initialize a restart which in turn leads to the plugin setup being executed again, which leads to a new certificate being obtained.

## Requirements
In order for this plugin to work you need the following:
* Own a domain
* Setup CoreDNS on a publicly reachable IP
* Setup CoreDNS as the [authoritative DNS server](https://en.wikipedia.org/wiki/Name_server#Authoritative_name_server) for that domain
* Port 53 - While CoreDNS may serve DNS over TLS on any port, during startup the plugin will use port 53 to solve the ACME Challenge

To learn more about how to setup an authoritative DNS server, take a look at [this article](https://hugopeixoto.net/articles/self-hosting-nameservers.html) from [Hugo Peixoto](https://hugopeixoto.net/about.html) or [this article that also uses CoreDNS](https://www.gophp.io/run-your-own-nameservers-with-coredns/).
Also, if you need a general refresher on how DNS works, [here is my favourite ressource][comic]

## Setup
The goal is to have this plugin integrated into the main CoreDNS repository, once that happens there wont be any setup requirements.
Until then, the plugin exists as an external Plugin, which means you will have to go through some extra steps to compile 
CoreDNS with the Plugin integrated, which are described here:

```
# Clone CoreDNS
git clone https://github.com/coredns/coredns
cd coredns

# replace the original tls plugin with this tlsplus plugin
sed -i 's/tls:tls/tls:github.com\/mariuskimina\/coredns-tlsplus/g' plugin.cfg

# Get the module
go get github.com/mariuskimmina/coredns-tlsplus

# Tidy modules
go mod tidy

# Compile
go build
```

### authoritative Corefile

The following is a straightforward example configuration for a CoreDNS server that is setup as the authoritative DNS Server for `mydomain.com`.
this example assumes that there are two host under `mydomain.com` one is a website, reachable at `mydomain.com` directly. The other one is 
a CoreDNS server that's running at `ns1.mydomain.com`.

With this configuration, the DNS server answer queries over both UDP and DoT. On first start-up the server will obtain a certificate for `n1.mydomain.com`. This certificate will automatically be renewed once more than 70% of it's validity period have passed.

```
tls://example.com {
    tls acme {
        domain ns1.example.com
    }
    hosts {
        xxx.xxx.xxx.xxx example.com
        xxx.xxx.xxx.xxx ns1.example.com
    }
}

example.com {
    hosts {
        xxx.xxx.xxx.xxx example.com
        xxx.xxx.xxx.xxx ns1.example.com
    }
}
```

### forwarding Corefile
This Corefile forwards all DNS over TLS requests to 9.9.9.9, the DNS Server of [quad9](https://www.quad9.net/). 

Notice that this DNS Server listens on a custom port, 8853, while the standard port for DoT is 853. Being able to 
setup such a forwarding server can be useful in restrictive environments where the port 853 may be blocked to prevent
people from encrypting their DNS traffic.

With the use of this new plugin, you can setup such a server and completly forget about it since the plugin will handle all 
certificate renewals for you.

```
tls://.:8853 {
    tls acme {
        domain ns1.example.com
    }
    forward . tls://9.9.9.9 {
        tls_servername dns.quad9.net
    }
}

example.com {
    hosts {
        xxx.xxx.xxx.xxx example.com
        xxx.xxx.xxx.xxx ns1.example.com
    }
}
```


## Setting up multiple DNS Server
For the most part, dns server should be setup with some redundancy. If you want to use this plugin with multiple CoreDNS Server, they all need to be able to access
the same Certificate. This can be achivied using a shared filesystem, like NFS, and pointing the `certpath` of all your CoreDNS Server to a location on this shared
fileystem.

I have tested such a cluster setup by running two CoreDNS instances on different Servers from DigitalOcean, following [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-18-04) made
setting up NFS between them easy and worked right away.

```
tls acme {
    domain ns1.example.com
    certpath /some/path/on/nfs/certs/
}
```

## Futute work
There are more ways in which CoreDNS and the ACME protocol could be used for certificate management. 

* CoreDNS could provide an API to let solve the AMCE Challenge for other (web-)servers
* CoreDNS could use the API of another DNS provider to obtain a certificate for domain without having to be the autoritative DNS server itself

Caddy can also automatically perform local HTTPS by creating it's own trusted certificate chain. This feature could also be implemented in CoreDNS in the future.

## Final Words
This Plugin was created as part of the 2022 [Google Summer of Code](https://summerofcode.withgoogle.com/). As a student I had made some small contributions to open-source projects here and there 
but nothing that comes even close to the scale of this project. Participating in GSoC helped empower me to focus on a project all the way from start to finish,
which is something not just me but a lot of people struggle with. If you always wanted to have an impact on open-source software but could not never commit
yourself to a side project long enough, consider participating in next years Google Summer of Code

I want to thank my mentors [Yong Tang](https://www.linkedin.com/in/yong-tang/) and [Paul Greenberg](https://www.linkedin.com/in/greenpau/) who guided me on this journey. Due to them I was not only able to make a major contribution to CoreDNS but they also taught me a lot about the economics of open-source technologies. The lessons I learned during these
months will without a doubt be useful for many years to come.

## References

### Articles and Docs
https://zwischenzugs.com/2018/01/26/how-and-why-i-run-my-own-dns-servers/
https://www.cloudflare.com/learning/dns/dns-over-tls/
https://developers.google.com/speed/public-dns/docs/dns-over-tls
https://letsencrypt.org/docs/challenge-types/
https://www.joshmcguigan.com/blog/run-your-own-dns-servers/
https://opensource.com/article/17/4/build-your-own-name-server
https://www.gophp.io/run-your-own-nameservers-with-coredns/
https://hugopeixoto.net/articles/self-hosting-nameservers.html

### Open Source Software used
CoreDNS: https://github.com/coredns/coredns  
Certmagic: https://github.com/caddyserver/certmagic  
Pebble: https://github.com/letsencrypt/pebble  

[ACME]: https://www.rfc-editor.org/rfc/rfc8555
[comic]: https://howdns.works/ep1/
[adventours in selfhosting autoritative DNS servers]: https://mariuskimmina.com/blog/2022-06-05-adventours-in-selfhosting-dns/

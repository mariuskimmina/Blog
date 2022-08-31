---
title: Automated Certificate Management in CoreDNS
author: Marius Kimmina
date: 2022-06-03 14:10:00 +0800
tags: [DNS, Infra]
published: true
---

This post introduces a new CoreDNS Plugin that allows for fully automatic TLS certificates in CoreDNS. No more worrying about
expiring certificates and no need to setup external programms such as certbot. CoreDNS can handle it all for you. \
https://github.com/mariuskimmina/coredns-tlsplus

## Table of Contents
1. [Who should use this plugin and when?](#who-should-use-this-plugin-and-when)
2. [Requirements](#requirements)
3. [Setup](#setup)
4. [How it works](#how-it-works)
5. [Future Work](#future-work)
6. [References](#references)


## Who should use this plugin and when?
First of all, using this plugin makes it really easy to setup DNS over TLS or DNS over HTTPS. If you are a Developer who wants to setup a DNS server with encryption, you can do this easily. You shouldn't need to be SRE or any other kind of infrastructure expert to set this up. As to when you want to use this pluign, there are 2 cases in which this plugin might help you tremendously. 

* You want to setup an autoritative DNS server for your Domain and support DNS over TLS or DNS over HTTPS
* You work in a very restricted network and you need an encrypted DNS forwarder on a non-standard port


### authoritative DNS
Since CoreDNS has to be the authoritative DNS Server for a domain to make this plugin work, the most obvious use case is to serve DNS over TLS or DNS over HTTPS for this particular domain. If you are the owner of `example.com` and you want to setup your own nameservers at `ns1.example.com` and `ns2.example.com`  for example (should never rely on a single nameserver) and you want to offer DNS over TLS or DNS over HTTPS then this plugin is for you! 

### Forwarding
For this to work, the CoreNDS server still needs to be setup to be the authoriative DNS server for a domain. Instead of only
answering queries about this particular domain we instead forward all DoT querys to an upstream DNS Resolver such Google's 8.8.8.8 or 
Cloudflare's 1.1.1.1 Servers.
Once you have such a forwarder setup with this plugin you can just forget about it since the certificate renewal is going to happen automatically. 

## Requirements
This plugin uses [ACME][ACME] to obtain and renew certificates. In order for this to work you need to do the following:
* Own a domain
* Setup CoreDNS on a publicly reachable IP
* Setup CoreDNS as the authoritative DNS server for that domain
* Port 53 - While CoreDNS may serve DNS over TLS on any port, during startup the plugin will use port 53 to solve the ACME Challenge

To learn more about how to setup an authoritative DNS server, take a look at my other article about [adventours in selfhosting autoritative DNS servers].
Also, if you need a general refresher on how DNS works, take a look at [this comic][comic]


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

```
tls://mydomain.com {
    tls acme {
        domain *.mydomain.com
    }
    hosts {
        xxx.xxx.xxx.xxx mydomain.com
        xxx.xxx.xxx.xxx ns1.mydomain.com
    }
}

https://mydomain.com {
    tls acme {
        domain ns1.mydomain.com
    }
    hosts {
        xxx.xxx.xxx.xxx mydomain.com
        xxx.xxx.xxx.xxx ns1.mydomain.com
    }
}

mydomain.com {
    hosts {
        xxx.xxx.xxx.xxx mydomain.com
        xxx.xxx.xxx.xxx ns1.mydomain.com
    }
}
```

As you can see in this line `domain *.mydomain.com` we can obtain wildcard certificates with this method. 

### forwarding Corefile

```
tls://. {
    tls acme {
        domain ns1.mydomain.com
    }
    forward . tls://9.9.9.9 {
        tls_servername dns.quad9.net
    }
}

https://. {
    tls acme {
        domain ns1.mydomain.com
    }
    forward . https://9.9.9.9 {
        tls_servername dns.quad9.net
    }
}

mydomain.com {
    hosts {
        xxx.xxx.xxx.xxx mydomain.com
        xxx.xxx.xxx.xxx ns1.mydomain.com
    }
}
```


## Setting up multiple DNS Server
For the most part, dns server should be setup with some redundancy. If you want to use this plugin with multiple CoreDNS Server, they all need to be able to access
the same Certificate. This can be achivied using a shared filesystem, like NFS, and pointing the `certpath` of all your CoreDNS Server to a location on this shared
fileystem.

```
tls acme {
    domain ns1.mydomain.com
    certpath /some/path/on/nfs/certs/
}
```

## How it works
On startup the plugin first checks if it already has a valid certificate, if it does there is nothing to do and the CoreDNS server will start. If it doesn't (or if the certificate will expire soon) then it will initialize the ACME DNS Challenge and ask Let's Encrypt (assuming you didn't configure another CA) for a Certificate for the domain you configured  (assume it's  `ns1.example.com`). The plugin will also start to serve DNS requests on port 53. Let's Encrypt receives our request and sends out DNS requests for `_acme-challenge.example.com`. Since CoreDNS is supposed to be setup as the autoritative DNS server for `example.com`, these requests will reach us. The Plugin can answer those requests and in return receiv a valid certificate for `ns1.example.com`.

![image](/blog/tlsplus/how-it-works.png "Plugin flow")

Furthermore, the plugin then starts a loop that runs in the background and checks if the certificate is about to expire. If it is, CoreDNS will initialize a restart which in turn leads to the plugin setup being executed again, which leads to a new certificate being obtained.

## Futute work
There are more ways in which CoreDNS and the ACME protocol could be used for certificate management. 

* CoreDNS could provide an API to let solve the AMCE Challenge for other (web-)servers
* CoreDNS could use the API of another DNS provider to obtain a certificate for domain without having to be the autoritative DNS server itself


## References
CoreDNS: https://github.com/coredns/coredns
Certmagic: https://github.com/caddyserver/certmagic
Pebble: https://github.com/letsencrypt/pebble

[ACME]: https://www.rfc-editor.org/rfc/rfc8555
[comic]: https://howdns.works/ep1/
[adventours in selfhosting autoritative DNS servers]: https://mariuskimmina.com/blog/2022-06-05-adventours-in-selfhosting-dns/

---
title: Automated Certificate Management in CoreDNS
author: Marius Kimmina
date: 2022-06-03 14:10:00 +0800
tags: [DNS, Infra]
published: false
---

This post introduces a new CoreDNS Plugin that allows for fully automatic TLS certificates in CoreDNS. No more worrying about
expiring certificates and no need to setup external programms such as certbot. CoreDNS can handle it all for you. \
https://github.com/mariuskimmina/coredns-tlsplus

## Table of Contents
1. [Requirements](#requirements)
2. [Use Case](#use-case)
3. [Setup](#setup)
4. [Future Work](#future-work)
5. [References](#references)

## Requirements
This plugin uses [ACME][ACME] to obtain and renew certificates. In order for this to work you need to do the following:
* Own a domain
* Setup CoreDNS on a publicly reachable IP
* Setup CoreDNS as the authoritative DNS server for that domain
* Port 53 - While CoreDNS may serve DNS over TLS on any port, during startup the plugin will use port 53 to solve the ACME Challenge

To learn more about how to setup an authoritative DNS server click here(TODO: find a good link).\
Also, if you need a general refresher on how DNS works, take a look at [this comic][comic]

## Use Case

There are 2 cases in which this plugin can be useful

* authoritative DNS for a domain
* Forwarding

### authoritative DNS

Since CoreDNS has to be the authoritative DNS Server for a domain to make this plugin work, the most obvious use case is to serve
DNS over TLS or DNS over HTTPS for this particular domain.

### Forwarding

For this to work, the CoreNDS server still needs to be setup to be the authoriative DNS server for that domain. Instead of only
answering queries about this particular domain we instead forward all DoT querys to an upstream DNS Resolver such Google's 8.8.8.8 or 
Cloudflare's 1.1.1.1 Servers.


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
this example assumes that there are two host under `mydomain.com` one is a website, reachable at `mydomain.com` directly the other one is 
the CoreDNS server that's running at `ns1.mydomain.com`.

```
tls://mydomain.com {
    tls acme {
        domain ns1.mydomain.com
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

## Futute work

## References
Certmagic: https://github.com/caddyserver/certmagic

[ACME]: https://www.rfc-editor.org/rfc/rfc8555
[comic]: https://howdns.works/ep1/

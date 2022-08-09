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
* Setup CoreDNS as the authoritative DNS server for that domain
* Port 53 - While CoreDNS may serve DNS over TLS on any port, during startup the plugin will use port 53 to solve the ACME Challenge

To learn more about how to setup an authoritative DNS server click here(TODO: find a good link).\
Also, if you need a general refresher on how DNS works, take a look at [this comic][comic]

## Use Case

There are 2 cases in which this plugin can be useful

* authoritative DNS for a domain
* Forwarding

### authoritative DNS

### Forwarding

tesjtalkdfjlaskfdj

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

## Futute work

## References
Certmagic: https://github.com/caddyserver/certmagic

[ACME]: https://www.rfc-editor.org/rfc/rfc8555
[comic]: https://howdns.works/ep1/

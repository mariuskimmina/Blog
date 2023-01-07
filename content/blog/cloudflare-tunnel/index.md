---
title: Using Cloudflare tunnel to expose your application to the internet during development
author: Marius Kimmina
date: 2022-09-02 14:10:00 +0800
tags: [Infrastructure, Go, Cloudflare]
published: false
---


```
version: "3.3"
services:
  tunnel:
    container_name: cloudflared-tunnel
    image: cloudflare/cloudflared
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
      - TUNNEL_URL=http://backend:8080
    depends_on:
      - backend
    networks:
      - main_net
```

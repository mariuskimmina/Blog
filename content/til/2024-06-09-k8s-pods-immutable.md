---
title: Immutability of pods in Kubernetes
author: Marius Kimmina
date: 2024-06-08 14:10:00 +0800
categories: [TIL]
tags: [kubernetes]
published: true
---

First of, pods in Kubernetes aren't entirely immutable - just mostly.

> Most of the metadata about a Pod is immutable. For example, you cannot change the namespace, name, uid, or creationTimestamp fields;

See https://kubernetes.io/docs/concepts/workloads/pods/ for the above quote

If I want to change the name or the namespace of a pod, I can't. I have to instead make a new Pod and delelte the old one.

## Further reading

https://blog.dave.tf/post/new-kubernetes/#mutable-pods
https://kubernetes.io/docs/concepts/workloads/pods/

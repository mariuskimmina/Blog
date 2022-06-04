---
title: Google Summer of Code - Introduction
author: Marius Kimmina
date: 2022-06-05 14:10:00 +0800
categories: [GSoC]
tags: [GSoC]
published: true
---

![image](/images/gsoc/logo.png "gsoc-logo")

Welcome, in this series of blog posts I want to take you with me on my journey with Google Summer of Code, which I am participating in this year.
In case you have never heard about it, I will start with explaining what this event is all about and why I take part in it. If you have already know what it is, then feel free to [skip it].

### What is Google Summer of Code?
Let's start with the official definition by Google.
> Google Summer of Code is a global, online program focused on bringing new contributors into open source software development. GSoC Contributors work with an open source organization on a 12+ week programming project under the guidance of mentors.

Essentially, first open source projects apply to GSoC with projects that they would like students to work on. These projects will then be listed on the GSoC page and students can apply for them (but you can also propose your own project idea if you want to). Since Google is sponsoring these projects, only a limited amount of applications will be accepted. When you successfully applied for a project and have been accepted then you will spend around 12 weeks working on said project with guidance from your mentors, which will most likely be maintainers of the project.
If everything goes well and your mentors are happy with the results you deliver, then you get paid for your work by google with a stipend. That being said, being able to work together closely with experienced open-source developers and learning from them is worth far more than the payments you receiv.


### Which project(s) did I apply for?
I believe that students are allowed to apply for multiple projects but can only be accepted for one. However, I only applied for one project as writing a good project proposal, which is essential to being accepted, takes quite a bit of time. So I think it's better to carefully choose one project you are really interested in and only apply to one, with a really well written and thoughtout proposal.

There were two projects that seemed really interesting to me:

* [Kubebuilder][Kubebuilder]
* [CoreDNS][CoreDNS]

I thought about applying for the kubebuilder project first, but after joining one of their project meetings over zoom, it became aparent that there are many students which will apply for this project and the odds of being accepted are rather low.
The CoreDNS project on the other hand did not seem to gather as much attention and since both project seemed interesting and worthwhile, I applied for the CoreDNS project.
So, observing if other students show interests in the same project can also be useful and you might want to take that into consideration. There are far more students applying for GSoC every year than there are slots available.

### Project description
You can find my detailed project proposal: [here][proposal] \
TLDR: I am creating a new TLS plugin that can be used as a drop-in-replacement for the current one, so that CoreDNS operators can serve DNS over TLS without having to worry about certificates at all, they will be aquired, applied and renewed wihtout them having to do anything.

### Current State
I am currently developing the plugin as an [external plugin][explugins] first, the repository can be found: [here][tlsplus]
Keep in mind that this is a work in progress and there might still be a lot of messy code, debug information and missing functionality.
Once everything works I will also send a pull request to the CoreDNS repo and ask for it to replace the current tls internal plugin.

### Working with mentors
I am still very early in the project as of writing this, but so far I believe to have gotten lucky with 2 excellent mentors in [yong tang][yongtang] and [paul greenberg][greenpau].
We are doing weekly meeting to sync up on the state of the project and they are also always open to questions in the mean time.

### What's next?
In the next blog posts I will describe challenges that I faced and the solutions that I have come up with. I will provide updates on the state of the project and on my experiences as a GSoC Contributor.



[yongtang]: https://www.linkedin.com/in/yong-tang/
[greenpau]: https://www.linkedin.com/in/greenpau/
[Kubebuilder]: https://github.com/kubernetes-sigs/kubebuilder
[CoreDNS]: https://github.com/coredns/coredns
[explugins]: https://coredns.io/explugins/
[proposal]: https://github.com/coredns/rfc/pull/13
[tlsplus]: https://github.com/mariuskimmina/tlsplus


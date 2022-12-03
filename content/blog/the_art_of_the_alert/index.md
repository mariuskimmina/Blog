---
title: The Art of the Alert
author: Marius Kimmina
date: 2022-09-02 14:10:00 +0800
tags: [DNS, Infra]
published: false
---

## The need for alerts
## What to alert on
## Alerts for broken alert systems


Nowadays it has become more important than ever to ensure that your service or application is always up and running. Outages are a great way to get your customers to look for alternatives. 

There are many aspects of our service that we might be interested in, but having more data will increase your spending as well and it will make it harder to do effective troubleshooting. The gerneral advice is, that there are 4 signals that you want to monitor no matter what and these are

* Errors
* Latency
* Traffic
* Saturation

**Errors** are perhaps the most obvious. When the application is failing to handle user requests and responds with `500 Internal Server Error` to your customers, then you definitely want to know about it.

**Latency** tells us how well our application performs. A website with high latency will load slowly and leave a bad expression on the user.

**Traffic** indicates if our services is available for customers. sudden drops in traffic can be a sign that something is going wrong. Maybe the DNS reccord for you site is poiting to the wrong IP and customers can't reach your service any more.

**Saturation** tells us how much storage we still have available. Running out of storage can either lead to our services going down completely or in case of a more resilient application, some feature that require storing data becoming unavailable.

Measuring these things doesn't seem to hard when you think of the way we used to build online services. You would have one webserver and one database. Just add checks to make sure that you have enough space available on your database.


![image](/blog/art-of-the-alert/old-architecture.png "Old Architecture")

As we build applications for the scale of millions of customers tho, this approach couldn't work anymore. Applications like Google Search, Uber or Twitter aren't build like this anymore. They are using a [microservice architecture](). Which could look a little something like this, tho this image is still highly simplified compared to what big tech companies are using these days. It does not contain any monitoring system, message queues or other supporting infrastructure. 

![image](/blog/art-of-the-alert/new-architecture.png "New Architecture")

Microservices solve the problem of scalability but at the cost of rising complexity. Instead of having one application that is either up or down, there are now houndreds or potentially even thousands of individual services that we need to keep an eye on. Instead of making sure that our one big application has enough storage available, we now need to look at each of our hundreds of services and make sure that every single one has the ressources it needs to operate. 


Missing a critical alert can be devastating but alerting to much will also drag you down.

Not all hope is lost tho, there was also a lot of progress in the open-source world on building tools to deal with these challenges.



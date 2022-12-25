---
title: Monitoring Druid with Prometheus
author: Marius Kimmina
date: 2022-08-01 14:10:00 +0800
tags: [Infrastructure, Observability, Monitoring, Druid, Prometheus]
published: true
---
 
TLDR: Here is a table showing you which druid monitors to configure on wich component 

| Monitor                                                     | Historical | Broker | Router | Coordinator | MiddleManager |
| ----------------------------------------------------------- | ---------- | ------ | ------ | ----------- | ------------- |
| org.apache.druid.java.util.metrics.JvmMonitor               | yes        | yes    | yes    | yes         | yes           |
| org.apache.druid.java.util.metrics.JvmCpuMonitor            | no        | no    | no    | no         | no           |
| org.apache.druid.java.util.metrics.JvmThreadsMonitor        | no        | no    | no    | no         | no           |
| org.apache.druid.java.util.metrics.SysMonitor               | yes        | no     | no     | no          | yes           |
| org.apache.druid.client.cache.CacheMonitor                  | no     | no | no | no      | no        |
| org.apache.druid.server.metrics.SegmentStatsMonitor         | yes        | no     | no     | no          | no            | 
| org.apache.druid.server.metrics.TaskCountStatsMonitor       | no         | yes    | no     | yes         | no            |
| org.apache.druid.server.metrics.QueryCountStatsMonitor      | yes        | yes    | yes    | no          | no            |
| org.apache.druid.server.metrics.WorkerTaskCountStatsMonitor | no         | no     | no     | no          | yes           |
| org.apache.druid.server.metrics.HistoricalMetricsMonitor    | yes        | no    | no     | no          | no            |

If you want to know more about how I got there, continue reading.

---

Lately I was tasked to seutp monitoring for our [apache druid](https://druid.apache.org/) cluster. At first, this seemed like a straightforward task since there is already a [druid exporter](https://github.com/opstree/druid-exporter) but looking at the commit history and the general rather low amount of activity around this project, I was a little concerned about using it and started searching for an alternative. 

What I found was that there is a [prometheus druid extensions](https://druid.apache.org/docs/latest/development/extensions-contrib/prometheus.html) which lets all druid components emit metrics themselves.  Awesome I thought, this should go smoothly.  


### some heading

At first, everything seemed great, I would get rid of this exporter and replace it with a few extra lines in the druid configuraiton.

```
druid.emitter=prometheus
druid.emitter.prometheus.port=9141
```

But the result wasn't quite what I expected, there were very few metrics present that I actually cared about.
Looking at the documentation, it seems we can configure an array of `monitors` as they call it

The problem here is that the documentation does not clearly state which monitors can be used on which component of druid. 
Let's take a brief look at the architecture of druid - there are 5 components, which can be broadly categorized as follows:

* Data Servers
	* Historical
		 * Hot
		 * Cold
	* Middle Managers
* Query Servers
	* Routers (Optional)
	* Brokers
* Master Server
	* Coordinator
		* Overlord





| Monitor                                                     | Historical | Broker | Router | Coordinator | MiddleManager |
| ----------------------------------------------------------- | ---------- | ------ | ------ | ----------- | ------------- |
| org.apache.druid.java.util.metrics.JvmMonitor               | yes        | yes    | yes    | yes         | yes           |
| org.apache.druid.java.util.metrics.JvmCpuMonitor            | yes        | yes    | yes    | yes         | yes           |
| org.apache.druid.java.util.metrics.JvmThreadsMonitor        | yes        | yes    | yes    | yes         | yes           |
| org.apache.druid.java.util.metrics.SysMonitor               | yes        | no     | no     | no          | yes           |
| org.apache.druid.client.cache.CacheMonitor                  | yes/no     | yes/no | yes/no | yes/no      | yes/no        |
| org.apache.druid.server.metrics.SegmentStatsMonitor         | yes        | no     | no     | no          | no            | 
| org.apache.druid.server.metrics.TaskCountStatsMonitor       | no         | yes    | no     | yes         | no            |
| org.apache.druid.server.metrics.QueryCountStatsMonitor      | yes        | yes    | yes    | no          | no            |
| org.apache.druid.server.metrics.WorkerTaskCountStatsMonitor | no         | no     | no     | no          | yes           |
| org.apache.druid.server.metrics.HistoricalMetricsMonitor    | yes        | yes/no    | no     | no          | no            |

Druid version: `24.0.1`

Enabling JvmMonitor on all of them lead to no results, even tho there was no error and everything started normally, I got no data.
Enabling them only on historical and middlemanager nodes results in proper JVM metrics. 
Try disabling everything on common and add JVM to broker: JVM metrics on broker now work as well

Now try enabling it on router and coordinator individually as well: Now they are all there, wtf

Noticed that I still got cache metrics even with cacheMonitor only on broker and router - what if I don't put cache monitor anywhere? A: They are still there. I can't get them to show me anything tho, configured or not.

Oh aparently  `RealtimeMetricsMonitor` also exists but isn't listed on the configuration page at all. But this is also aparently deprecated.

cgroup metrics?


Adding Blackbox exporter to do health checks on all components of the cluster



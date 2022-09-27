---
layout: post
title: A Short Emergency Response Guide for Elasticsearch
excerpt: Help! Production is on fire! 
---

## Help! Production is on fire!

Our job is first to identify what exactly is on fire, and for that we need to get as much <mark>context</mark> as possible, as fast as possible.


#### <mark-soft>Step 1:</mark-soft> Check `_cat/nodes` for resource usage

Lets first check which nodes are actually in trouble.

Run:

```js
GET _cat/nodes?v&h=ip,name,cpu,ram.max,heap.max,heap.percent,node.role,diskAvail,master
```

The output looks like this.
  
  ```
  master ip           name                cpu ram.max heap.max heap.percent node.role diskAvail
  -      10.47.48.170 instance-0000000009  26     2gb    844mb           60 himrst       48.7gb
  -      10.47.48.127 instance-0000000008  16     2gb    844mb           64 himrst       52.7gb
  *      10.47.48.118 instance-0000000003   7     2gb    844mb           69 himrst         50gb
  -      10.47.48.61  instance-0000000005   0     1gb    268mb           41 lr            1.8gb
  ``` 

If we notice a high CPU usage on a group of nodes, verify the `node.role` of those roles, if you have data tiers (a hot-warm architecture) it might be that only `warm` nodes are high, and `hot` nodes are unaffected. 

There are more variables you can check like `search.query_current` and `search.fetch_current` which will show us the amount of time is being currently spent for search in query and fetch phases respectively. `GET _cat/nodes?help` is your friend


#### <mark>Step 2:</mark> Check the hot threads

This API yields a breakdown of the hot threads on each selected node in the cluster. The output is plain text with a breakdown of each node’s top hot threads.

Lets sample them by 1 second:

```js
GET _nodes/hot_threads?interval=1s&ignore_idle_threads
```

The output will be something like this, with only the important parts:

```
::: {warm-xxx}{XXXXXXX}{YYYYYYYY}{10.10.10.10}{10.10.10.10:9300}{aws_availability_zone=us-west-2b, data_type=warm, ml.machine_memory=64388997120, ml.max_open_jobs=20, xpack.installed=true, ml.enabled=true}
   Hot threads at 2019-12-30T23:22:24.304Z, interval=500ms, busiestThreads=3, ignoreIdleThreads=true:

   44.0% (440.2ms out of 1000ms) cpu usage by thread 'elasticsearch[warm-xxx][management][T#1]'
      ... omitted ...

   42.7% (413.4ms out of 1000ms) cpu usage by thread 'elasticsearch[warm-xxx][search][T#2]'                    
     ... omitted ...

   41.8% (408.9ms out of 1000ms) cpu usage by thread 'elasticsearch[warm-xxx][search][T#7]'
     ... omitted ...

```

#### <mark>Step 3:</mark> Check Tasks

The [Task Management API](https://www.elastic.co/guide/en/elasticsearch/reference/current/tasks.html) can give you a lot of information about the operations being executed at the cluster at any given time.

```js
GET _tasks?human&detailed
```

They can be filtered to include only read (search) operations

```js
GET _tasks?human&detailed&actions=indices:data/read/*
```

The output will show us the `running_time` and, in case of searches, the content of the specific search request being executed: 

```
      "0l67b6iLSzmp7v3TNtkjbQ:138659665" : {
          "node" : "0l67b6iLSzmp7v3TNtkjbQ",
          "id" : 138659665,
          "type" : "transport",
          "action" : "indices:data/read/search",
          "description" : "indices[tasks], types[], search_type[QUERY_THEN_FETCH], source[{\"size\":0,\"query\":{\"bool\":{\"must\":[{\"terms\":{\"User.Actions.ActionId\":[\"f6583dbd-4079-4efd-80c4-28e3f0606c1f\",\"2f80c480-18a4-4079-4efd-d3bdf9361164\",
          ...
          "start_time" : "2022-06-28T15:27:20.766Z",
          "start_time_in_millis" : 1656430040766,
          "running_time" : "36.6s",
          "running_time_in_nanos" : 36665783627,
```

The task `0l67b6iLSzmp7v3TNtkjbQ:138659665` is a **search** task that has been running for 36 seconds, its `source` is also there, which might help identify the culprit.

#### <mark>Step 4:</mark> Cancel long running tasks

If you see queries that are impacting performance for too long and you want to cancel them, you can with `PUT _tasks/<id>/_cancel`:


```
POST _tasks/0l67b6iLSzmp7v3TNtkjbQ:138659665/_cancel
```

However if you find yourself cancelling tasks every day, your team should really rethink their data structure, query design or cluster size. 


### Problem solved? Lets get a little bit more proactive by:

1) having a dedicated [Monitoring Cluster](https://www.elastic.co/guide/en/elasticsearch/reference/8.3/monitor-elasticsearch-cluster.html) 

  - In Elastic Cloud that is as easy as creating a secondary (small) cluster [and pointing Logs and Metrics over there](https://www.elastic.co/blog/monitoring-elastic-cloud-deployment-logs-and-metrics).
  - In on-premises clusters you need to start an instance of Metricbeat and [point it to your production cluster](https://www.elastic.co/guide/en/elasticsearch/reference/8.3/configuring-metricbeat.html):
     - The elasticsearch module will fetch monitoring info from your `host` with the following configuration (metricbeat.yml):
      
        ```yaml
        metricbeat.modules:
        - module: elasticsearch
          xpack.enabled: true
          period: 10s
          hosts: ["https://prod-cluster:9200"] 
          scope: cluster
          username: "user"
          password: "secret"
          ssl.enabled: true
          ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]
          ssl.verification_mode: "certificate"

        output.elasticsearch:
          hosts: ["https://my-monitoring-cluster:9200"]
          username: "metricbeat_writer"
          password: "secret"
        ```

2) Enabling [Slow Logs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-slowlog.html)

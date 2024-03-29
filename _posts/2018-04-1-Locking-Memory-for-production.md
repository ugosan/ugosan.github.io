---
layout: post
title: Elasticsearch - Locking Memory for Production 
tags: elasticsearch
---

A common error people face when putting an Elasticsearch cluster to production has to do with *memory locking*. Tipically users would see errors like "Unable to lock JVM memory (ENOMEM). This can result in part of the JVM being swapped out. Increase RLIMIT_MEMLOCK (ulimit)" or "memory locking requested for elasticsearch process but memory is not locked".

Here is what I have done to lock the memory on Elasticsearch nodes. This procedure will work in any distribution that use **systemd**.
 
<mark>1)</mark> Override the elasticsearch service 
 
```
systemctl edit elasticsearch
```

Then add the following and save:
```ruby
[Service]
LimitMEMLOCK=infinity
```


<mark>2)</mark> Check the elasticsearch.yml file

Make sure your `elasticsearch.yml` has the following config:
```
bootstrap.memory_lock: true
```


<mark>3)</mark> Reload the service and start Elasticsearch


```
systemctl daemon-reload
service elasticsearch restart
```

Thats it, the RAM should now be locked. Its also a good idea to disable swap




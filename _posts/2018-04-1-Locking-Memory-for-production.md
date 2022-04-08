---
layout: post
title: Elasticsearch - Locking Memory for Production 
---

A common error people face when putting an Elasticsearch cluster to production has to do with *memory locking*. Tipically users would see errors like "Unable to lock JVM memory (ENOMEM). This can result in part of the JVM being swapped out. Increase RLIMIT_MEMLOCK (ulimit)" or "memory locking requested for elasticsearch process but memory is not locked".

Here is what I have done to lock the memory on Elasticsearch nodes. This procedure will work in any distribution that use **systemd**.
 
1) Override the elasticsearch service 
 
```
systemctl edit elasticsearch
```

Then add the following and save:
```ruby
[Service]
LimitMEMLOCK=infinity
```

2) Reload the service and start Elasticsearch
```
systemctl daemon-reload
service elasticsearch restart
```


Thats it, restart your node and the RAM will be locked.


  [1]: https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html


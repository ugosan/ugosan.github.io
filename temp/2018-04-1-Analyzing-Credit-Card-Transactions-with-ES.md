---
layout: post
title: Elasticsearch - Locking Memory for Production 
---

Each row is a
An individual credit transaction made by a state employee on a specific date.

https://data.delaware.gov/Government-and-Finance/State-Employee-Credit-Card-Transactions/nurt-5rqw



Here is what I have done to lock the memory on my ES nodes, version 5.4.0 on RedHat/Centos 7 (it will work on other distributions if they use **systemd**).
 
You must make the change in 4 different places:

1) **/etc/sysconfig/elasticsearch**

On sysconfig: `/etc/sysconfig/elasticsearch` you should have:

```
ES_JAVA_OPTS="-Xms4g -Xmx4g" 
MAX_LOCKED_MEMORY=unlimited
```

`(replace 4g with HALF your available RAM as recommended [here][1])`

2) **/etc/security/limits.conf**

On security limits config: `/etc/security/limits.conf` you should have

    elasticsearch soft memlock unlimited
    elasticsearch hard memlock unlimited

3) **/usr/lib/systemd/system/elasticsearch.service**

On the service script: `/usr/lib/systemd/system/elasticsearch.service` you should uncomment:

    LimitMEMLOCK=infinity

you should do systemctl daemon-reload after changing the service script

4) **/etc/elasticsearch/elasticsearch.yml**

On elasticsearch config finally: `/etc/elasticsearch/elasticsearch.yml` you should add:

    bootstrap.memory_lock: true

Thats it, restart your node and the RAM will be locked, you should notice a major performance improvement.


  [1]: https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html


Next you can update your site name, avatar and other options using the _config.yml file in the root of your repository (shown below).

![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.
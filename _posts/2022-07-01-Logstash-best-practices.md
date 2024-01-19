---
layout: post
title: Logstash 8.x - Deploying, Ingesting and Testing the right way
excerpt: Some of the best practices for deploying Logstash in production in 8.x version, separated in  Deployment, Data Ingestion and Testing
---

<div class="header">

<img src="https://images.pexels.com/photos/2569842/pexels-photo-2569842.jpeg?auto=compress&cs=tinysrgb&h=450&dpr=2&fit=crop"/>
</div>

The **L** in ELK, Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite "stash", currently more than 50 outputs are supported, from Elasticsearch to Syslog.

Logstash is also currently the only option within the Elastic Stack if you want to fetch data that lives in a <mark>relational database</mark>, or if you want to write the same event to <mark>multiple outputs</mark>. But all that power also comes with a cost, a Logstash instance will require a lot more resources than an Elastic Agent with Filebeat, for instance.

Here are some of the best practices for deploying Logstash in production in 8.x version, separated in three areas: Deployment, Data Ingestion and Testing

## Deploying Logstash

We have several options for deploying Logstash

### As standalone

Deploy Logstash always as a <mark>system service</mark>, using the official packages for YUM or APT based distributions:

[Installing Logstash from Package Repositories](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html#package-repositories)



### In containers

The main advantage here is that you can run several logstash instances at the same time, they might even have different versions. You also have a better process isolation at the system-level.

    - Bind-mounted config files


### In Kubernetes 


## Data Ingestion

  - Use Datastreams instead of indices
    - Special type of index, data streams are well-suited for logs, events, metrics, and other continuously generated data that are rarely, if ever, updated.
    - standardized names
    - a common set of best practices for mappings
    ```ruby
      elasticsearch {
        hosts => "elasticsearch:9200"
        user => "elastic"
        password => "..."
        data_stream => true
        data_stream_type => "logs"
        data_stream_dataset => "hasura"
        data_stream_namespace => "%{log_type}"
      }
    ```
  - Pipelines
    - Maintain order? if order is not important, disable it
    - Dissect instead of Grok whenever possible



## Testing

- Testing
  - Use a Load Generator for testing
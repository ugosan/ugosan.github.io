---
layout: post
title: Debugging Logstash and Filebeat pipelines with Logshark
excerpt: Debugging Logstash and Filebeat pipelines with Logshark
tags: logstash filebeat logshark
---

I worked for many years as a [consultant in Elastic](https://www.elastic.co/consulting), and coding pipelines in Logstash and Filebeat for large Observability use cases that ingested terabytes worth of logs every day was our bread and butter.

Coding pipelines using those tools (and others) is a highly iterative process, specially when dealing with grok patterns to parse unstructured logs: you get some sample data, feed it to an input, and you will be repeating 1) coding the pipeline logic (*filters* in Logstash and *processors* in Filebeat) and 2) inspecting the output, until the logs are correctly parsed.

I always felt this iteration cycle of changing the pipeline and inspecting the output to be somewhat slow - sure you have the `console` output in both Logstash and Filebeat, but you end up with a mixed of those programs' outputs and *your* output, you will be surely scrolling a lot. And sure, you also have the `file` output on both tools but its easy to get lost when dealing with documents with hundreds of fields, since they are all written in a single line per document, no pretty-print.

I needed a way to tell right away if the output of my pipeline was correct or not, to have <mark>pretty printed</mark> and <mark>navigable</mark> output were my main requirements, if I had that my development iteration would be so much faster! Such tool didn't existed, so I've created it and called it [Logshark](https://github.com/ugosan/logshark/) (inspired by Wireshark, a popular network inspection tool)


<div class="header">
<img src="https://github.com/ugosan/logshark/raw/main/_doc/demo.gif" width="70%">
</div>

It's a CLI application with a terminal UI written in Go. It works by starting a small webserver that mimmicks Elasticsearch's behavior by accepting `_bulk` requests, so all you need to do is to redirect your Logstash/Filebeat `elasticsearch` output to the tool. 

This tool is particularly handy when changing *production* pipelines, as you can add a second elasticsearch output to your pipeline just to inspect the events, it will by default collect the first 100 events it sees and accept but discard the rest, you can inspect the next batch by hitting `r` to refresh it.

It will also tell you _events per second_ and the _average document size_, which are handy when you need to optimize your throughput by adjusting the bulk/batch size, something really important if, say, your are collecting logs from a machine in the southern hemisphere to send to an elasticsearch cluster in the northern.

You can run it using the [binary directly](https://github.com/ugosan/logshark/releases) (<5mb) or on [docker](https://github.com/ugosan/logshark/blob/main/docker-compose.yml). The UI can be used on anything that can emulate a terminal, like your regular Linux terminal, iTerm, tmux, PuTTY and even VSCode.


## Getting Started

## 1) Start the server

### binary

```perl
./logshark --host 0.0.0.0 --port 9200 --max 1000
```

### docker

```perl
docker run -p 9200:9200 -it ugosan/logshark -host 0.0.0.0 -port 9200
```


### docker-compose.yml

```yaml
version: "3.2"
services:

  logshark:
    image: ugosan/logshark
    tty: true
    stdin_open: true
```
<mark>note</mark> you should not use "docker-compose up" but instead "docker-compose run logshark sh" since docker-compose doesnt attach to containers with "up". e.g. docker-compose run -p 9200:9200 logshark -port 9200

```
docker-compose run -p 9200:9200 logshark -port 9200
```


## 2) Point your Logstash pipeline's output to it

Just like a normal `elasticsearch` output:

```ruby
input {}

filter {}

output {
  elasticsearch {
    hosts => ["http://host.docker.internal:9200"]
  }
  
}   
```

When using docker, you can reach the logshark container from another container using `host.docker.internal` like `docker run --rm byrnedo/alpine-curl -v -XPOST -d '{"hello":"test"}' http://host.docker.internal:9200`


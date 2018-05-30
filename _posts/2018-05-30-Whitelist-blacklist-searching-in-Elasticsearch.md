
---
layout: post
title: Whitelist / Blacklist searching in Elasticsearch
---

How do we match thousands of documents against a dynamic whitelist/blacklist in Elasticsearch?

You can use the [terms query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html)

Suppose we have an index of page access logs like so:

```json
PUT /mybeat-2018/_doc/1
{
    "host" : "elastic.co",
    "ttl" : 40
}

PUT /mybeat-2018/_doc/2
{
    "host" : "elastic.co",
    "ttl" : 666
}

PUT /mybeat-2018/_doc/3
{
    "host" : "google.com",
    "ttl" : 55
}
```

and an independent whitelist that can shrink or grow, with a bunch of hosts:

```json
PUT /whitelist/_doc/1
{
 "hosts" : [
   {
     "name" : "elastic.co"
   },
   {
     "name" : "twitter.com"
   }
 ]
}
```

Then a search on the `mybeat-*` for whatever is in the whitelist should reference the whitelist document (in our case the document with id: 1) like so:

```json
GET /mybeat-*/_search
{
    "query" : {
        "terms" : {
            "host" : {
                "index" : "whitelist",
                "type" : "_doc",
                "id" : "1",
                "path" : "hosts.name"
            }
        }
    }
}
```
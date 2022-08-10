---
layout: post
title: How to get and use the Root CA Certificate Fingerprint in the Elastic Stack
excerpt: More often than not, we have customers indexing fields "just in case" they need to be used for search.
---

Latest developments in Beats, Elastic Agent and Logstash now include a new parameter that makes easier to trust a self-signed certificate, we would just need <mark>A HEX encoded SHA-256 of a CA certificate</mark>. 

We can get this fingerprint by simply using `openssl` and connecting to the server you want to extract the fingerprint:

```
openssl s_client \
  -connect demos.es.us-east1.gcp.elastic-cloud.com:9243 \
  -servername demos.es.us-east1.gcp.elastic-cloud.com \
  -showcerts < /dev/null 2>/dev/null | \
  openssl x509 -in /dev/stdin -sha256 -noout -fingerprint | \
  sed 's/://g'  
```

Or, ff you have an actual certificate file for the CA you can just load it:

```
 openssl x509 -in ca_file.pem -sha256 -fingerprint | grep SHA256 | sed 's/://g'
```

The response will be something like

```
SHA256 Fingerprint=65C080BF18DBFA8F57606DBA0ED11D32DF42CF63B55CC07C7A764AA9597A9403
```

So you can use this fingerprint in `output.elasticsearch`:

```yaml
output.elasticsearch:
  hosts: ...
  api_key: ...
  index: ...
  ssl.ca_trusted_fingerprint: "65C080BF18DBFA8F57606DBA0ED11D32DF42CF63B55CC07C7A764AA9597A9403"
  ssl.verification_mode: full
```

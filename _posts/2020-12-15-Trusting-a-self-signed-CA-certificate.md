---
layout: post
title: Trusting a self-signed CA certificate
---

## Red Hat / CentOS / RockyOS

Copy the CA file to `/etc/pki/ca-trust/source/anchors/` and update with `sudo update-ca-trust`

## Ubuntu

Copy the CA file to `/usr/local/share/ca-certificates` and update with `sudo update-ca-certificates`


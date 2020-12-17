---
layout: post
title: Trusting a self-signed CA certificate
---

Follow this simple steps to make your OS to trust a self-signed Certificate Authority

## Red Hat / CentOS / RockyOS

Copy the CA file to `/etc/pki/ca-trust/source/anchors/` and update with `sudo update-ca-trust`

```bash
cp my-self-signed-ca.cer /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

## Ubuntu

Copy the CA file to `/usr/local/share/ca-certificates` and update with `sudo update-ca-certificates`

```bash
cp my-self-signed-ca.cer /usr/local/share/ca-certificates
sudo update-ca-certificates
```
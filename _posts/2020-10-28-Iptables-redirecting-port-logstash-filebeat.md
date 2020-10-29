---
layout: post
title: Need to bind Logstash/Filebeat to a port below 1000? Try iptables REDIRECT.
---

When using Logstash or Filebeat, sometimes we need to listen to a priviledged port (lower than 1000), for instance when listening to syslog in port 514 and we can't change the source port for whatever reason.

One solution is to run Logstash or Filebeat as `root`, but there is a much **safer** trick: redirect the priviledged port to a higher port.



```bash
sudo iptables -t nat -A PREROUTING -i ens192 -p udp --dport 514 -j REDIRECT --to-port 5514
sudo iptables -t nat -A PREROUTING -i ens192 -p tcp --dport 514 -j REDIRECT --to-port 5514
sudo iptables-save
```

The above `iptables` rules redirect the port `514` on the interface `ens192` to the port `5514`, for both UDP and TCP protocols. 

You can view the rules with: 

```
iptables -t nat -L -n -v
```
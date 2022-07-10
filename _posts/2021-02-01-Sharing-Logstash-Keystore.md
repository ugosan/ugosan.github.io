---
layout: post
title: Sharing Logstash keystore between multiple Logstash containers.
---

Create the keystore in one of the containers:

```
docker-compose exec logstash-example /usr/share/logstash/bin/logstash-keystore create
```

Copy the file from within the container to an outside folder, to be shared with the rest of the containers
  - To add a new key, just execute in any of the containers:
    - `docker-compose exec logstash-example /usr/share/logstash/bin/logstash-keystore add ES_PWD`
    - Confirm it was created with `docker-compose exec logstash-example /usr/share/logstash/bin/logstash-keystore list`

  - To use the keystore in a container, just pass the keystore file as a volume (`- ${PWD}/keystore/logstash.keystore:/usr/share/logstash/config/logstash.keystore`) and the keystore master password as an environment variable (`LOGSTASH_KEYSTORE_PASSWORD`):
  ```YML
  logstash-example:
    image: docker.elastic.co/logstash/logstash:7.8.1
    hostname: logstash-example
    user: root
    network_mode: host
    restart: always
    volumes:
      # - ...
      - ${PWD}/keystore/logstash.keystore:/usr/share/logstash/config/logstash.keystore
  environment:
    LS_JAVA_OPTS: "-Xms1024m -Xmx1024m"
    LOGSTASH_KEYSTORE_PASS: "${LOGSTASH_KEYSTORE_PASS}"
  ``
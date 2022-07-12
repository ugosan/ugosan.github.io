---
layout: post
title: Sharing Logstash keystore between multiple Logstash containers.
---

Create the keystore in one of the containers:

```
docker-compose exec logstash-example /usr/share/logstash/bin/logstash-keystore create
```

Copy the file from within the container to an outside folder, this is to be shared with the rest of the containers
  - To add a new key, just execute in any of the containers:
  ```
    docker-compose exec logstash-example /usr/share/logstash/bin/logstash-keystore add ES_PWD
  ```
  - Confirm the keystore was created with 
  ```
    docker-compose exec logstash-example /usr/share/logstash/bin/logstash-keystore list
  ```

  - To use the keystore in a container you should pass: 
    - the keystore file as a volume:
      ```
      ${PWD}/keystore/logstash.keystore:/usr/share/logstash/config/logstash.keystore
      ```
    - and the keystore master password as the environment variable ```LOGSTASH_KEYSTORE_PASSWORD``` :
 
<mark> Full docker compose example </mark>

  ```yaml
  logstash:
    image: docker.elastic.co/logstash/logstash:8.2.3
    volumes:
      # - ...
      - ${PWD}/keystore/logstash.keystore:/usr/share/logstash/config/logstash.keystore
  environment:
    LS_JAVA_OPTS: "-Xms1024m -Xmx1024m"
    LOGSTASH_KEYSTORE_PASS: "${LOGSTASH_KEYSTORE_PASS}"
  ```
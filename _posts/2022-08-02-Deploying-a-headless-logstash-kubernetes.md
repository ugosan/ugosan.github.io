---
layout: post
title: Deploying a headless Logstash in Kubernetes
excerpt: Deploying Logstash in Kubernetes with Centralized Pipeline Management 
---

<div class="header">

<img src="https://images.pexels.com/photos/2569842/pexels-photo-2569842.jpeg?auto=compress&cs=tinysrgb&h=450&dpr=2&fit=crop"/>
</div>

A "headless" Logstash is a Logstash that doesnt contain any pipeline logic in itself, instead it will fetch the pipelines definitions from a <mark> centralized location</mark> (Elasticsearch itself), so we can create our ETL pipelines through Kibana and deploy them automatically to several Logstash instances. This feature is called [Centralized Pipeline Management](https://www.elastic.co/guide/en/logstash/current/logstash-centralized-pipeline-management.html).

<div class="premonition info">
  <i class="premonition pn-info"></i>
  <div class="content">
    <p class="header">TLDR</p><p>Jump to the full Kubernetes <a href="#all-in-one-manifest">manifest file here</a></p>
  </div>
</div>


We are then going to create a Secret to store credentials, a ConfigMap to configure Logstash, a Deployment that will expose some container ports and finally a Service so Logstash is reachable from the outside. 


## <mark> Elasticsearch API Key </mark>
We first need to create an [API Key](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-create-api-key.html) so Logstash can communicate to Elasticsearch to fetch pipeline definitions. 
 
Let's define two roles: `my_role` which allows Logstash to fetch the pipeline definitions from Elasticsearch and `my_write_role` which will allow us to write to a datastream from our output. Those could be two different API Keys, but we are using just one to keep it simple.

 Open up Kibana and then run the following command:

```js
POST /_security/api_key
{
  "name": "logstash",   
  "role_descriptors": { 
    "my_role": {
      "cluster": ["monitor" ,"manage_logstash_pipelines"]
    },
    "my_write_role": {
      "index": [
        {
          "names": ["logs-*"],
          "privileges": ["all"]
        }
      ]
    }
  }
}
```

The response will have the `encoded` field, copy it:
```
{
  ...
  "encoded": "RlNXaEU0TUJXQWVtRnlHU3p6d0o6STkzVE9yX1RSTy03TGdiMHU5YWlZZw=="
}
```



## <mark> Secret </mark>

Next, we need to create a Kubernetes Secret with our API key, paste the value from the `encoded` field to a variable we are calling `ELASTICSEARCH_API_KEY`:
```yml
apiVersion: v1
kind: Secret
metadata:
  name: logstash-secrets
type: Opaque
data:
  ELASTICSEARCH_API_KEY: RlNXaEU0TUJXQWVtRnlHU3p6d0o6STkzVE9yX1RSTy03TGdiMHU5YWlZZw==
```

## <mark> ConfigMap </mark>


The `ELASTICSEARCH_API_KEY` variable will be used in a ConfigMap that will represent our `logstash.yml` with a very simple configuration that tells Logstash:

1) Where to fetch the pipeline configs (the `xpack.management.elasticsearch.hosts`)
   
2) What are the pipeline names to fetch (`xpack.management.pipeline.id`), in our case it will be anything starting with `my-pipeline-*`


```yml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-configmap
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    log.level: info
    xpack.management.enabled: true
    xpack.management.elasticsearch.hosts: ["https://my-cluster.es.us-east-1.aws.found.io:443"]  
    xpack.management.elasticsearch.api_key: "${ELASTICSEARCH_API_KEY}"
    xpack.management.logstash.poll_interval: 5s
    xpack.management.pipeline.id: ["my-pipeline-*"]
```

## <mark> Deployment </mark>


Next, we create a Deployment that will expose a couple `containerPort`, use the `ELASTICSEARCH_API_KEY` from the Secret `logstash-secrets` as an environment variable, and use our ConfigMap.
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-deployment
spec:
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.3.3
        ports:
        - containerPort: 5044
        - containerPort: 5045
        resources:
            limits:
              memory: "2Gi"
              cpu: "2500m"
            requests: 
              memory: "1Gi"
              cpu: "300m"
        env: 
          - name: ELASTICSEARCH_API_KEY
            valueFrom:
              secretKeyRef:
                name: logstash-secrets
                key: ELASTICSEARCH_API_KEY
        volumeMounts:
          - name: config-volume
            mountPath: /usr/share/logstash/config
      volumes:
      - name: config-volume
        configMap:
          name: logstash-configmap
          items:
            - key: logstash.yml
              path: logstash.yml
```

## <mark> Service </mark>

The service will expose the ports 5044 and 5045 to the outside through a LoadBalancer, depending on the provider that is enough - in GKE you will get  public ClusterIP automatically assigned.

```
kind: Service
apiVersion: v1
metadata:
  name: logstash-service
  labels: 
    app: logstash
spec:
  type: LoadBalancer
  selector:
    app: logstash
  ports:
  - protocol: TCP
    port: 5044
    targetPort: 5044
    name: my-pipeline-1
  - protocol: TCP
    port: 5045
    targetPort: 5045
    name: my-pipeline-2
```

## <mark> Logstash Pipeline </mark>

Our pipeline can open up a HTTP input, Beats input even Syslog on the available ports, for instance:

```ruby
input {
  http {
    port => 5045
    codec => json
    user => "webhook_admin"
    password => "verysecretpassword"
  }
}

filter { }

output {
    elasticsearch {
        hosts => "https://my-cluster.es.us-east-1.aws.found.io:443"
        ssl => true
        api_key => "${ELASTICSEARCH_API_KEY}"
        data_stream => true
        data_stream_type => "logs"
        data_stream_dataset => "my-datastream"
        data_stream_namespace => "dev"
    }
}

```

## All-in-one manifest

```yml
---
apiVersion: v1
kind: Secret
metadata:
  name: logstash-secrets
type: Opaque
data:
  ELASTICSEARCH_API_KEY: RlNXaEU0TUJXQWVtRnlHU3p6d0o6STkzVE9yX1RSTy03TGdiMHU5YWlZZw==

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-deployment
spec:
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.3.3
        ports:
        - containerPort: 5044
        - containerPort: 5045
        resources:
            limits:
              memory: "2Gi"
              cpu: "2500m"
            requests: 
              memory: "1Gi"
              cpu: "300m"
        env: 
          - name: ELASTICSEARCH_API_KEY
            valueFrom:
              secretKeyRef:
                name: logstash-secrets
                key: ELASTICSEARCH_API_KEY
        volumeMounts:
          - name: config-volume
            mountPath: /usr/share/logstash/config
      volumes:
      - name: config-volume
        configMap:
          name: logstash-configmap
          items:
            - key: logstash.yml
              path: logstash.yml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-configmap
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    log.level: info
    xpack.management.enabled: true
    xpack.management.elasticsearch.hosts: ["https://my-cluster.es.us-east-1.aws.found.io:443"]  
    xpack.management.elasticsearch.api_key: "${ELASTICSEARCH_API_KEY}"
    xpack.management.logstash.poll_interval: 5s
    xpack.management.pipeline.id: ["my-pipeline-*"]

---
kind: Service
apiVersion: v1
metadata:
  name: logstash-service
  labels: 
    app: logstash
spec:
  type: LoadBalancer
  selector:
    app: logstash
  ports:
  - protocol: TCP
    port: 5044
    targetPort: 5044
    name: my-pipeline-1
  - protocol: TCP
    port: 5045
    targetPort: 5045
    name: my-pipeline-2
```
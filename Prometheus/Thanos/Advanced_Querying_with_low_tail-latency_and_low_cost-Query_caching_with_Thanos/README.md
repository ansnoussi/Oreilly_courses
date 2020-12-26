## Initial Prometheus and Thanos Setup
#### Simple observability stack with Prometheus and Thanos
Let's imagine we have to deliver centralized metrics platform to multiple teams. For each team we will have a dedicated Prometheus. These could be in the same environment of in different environments (data centers, regions, clusters etc).

And then we will try to provide low cost, fast global overview. Let's see how we achieve that with Thanos

#### Let's lay the ground work
Let's quickly deploy Prometheuses with sidecars and Querier.

Execute following commands to setup Prometheus:
#### Prepare "persistent volumes" for Prometheuses
```
 mkdir -p prometheus_data
```
#### Deploy Prometheus
Let's deploy a couple of Prometheus instances and let them scrape themselves, so we can produce some metrics.
#### Prepare configuration
First, Prometheus server that scrapes itself:
```
global:
  scrape_interval: 5s
  external_labels:
    cluster: eu0
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
```

```
global:
  scrape_interval: 5s
  external_labels:
    cluster: eu1
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9091']
```

```
global:
  scrape_interval: 5s
  external_labels:
    cluster: eu2
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9092']
```
#### Deploy Prometheus
```
for i in $(seq 0 2); do
docker run -d --net=host --rm \
    -v $(pwd)/prometheus"${i}".yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus_data:/prometheus"${i}" \
    -u root \
    --name prometheus"${i}" \
    quay.io/prometheus/prometheus:v2.22.2 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:909"${i}" \
    --web.external-url=https://2886795303-909"${i}"-ollie08.environments.katacoda.com \
    --web.enable-lifecycle \
    --web.enable-admin-api && echo "Prometheus ${i} started!"
done
```

#### Inject Thanos Sidecars

```
for i in $(seq 0 2); do
docker run -d --net=host --rm \
    -v $(pwd)/prometheus"${i}".yml:/etc/prometheus/prometheus.yml \
    --name prometheus-sidecar"${i}" \
    -u root \
    quay.io/thanos/thanos:v0.17.0 \
    sidecar \
    --http-address=0.0.0.0:1909"${i}" \
    --grpc-address=0.0.0.0:1919"${i}" \
    --reloader.config-file=/etc/prometheus/prometheus.yml \
    --prometheus.url=http://127.0.0.1:909"${i}" && echo "Started Thanos Sidecar for Prometheus ${i}!"
done
```
#### Prepare Thanos Global View
And now, let's deploy Thanos Querier to have a global overview on our services.
```
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.17.0 \
    query \
    --http-address 0.0.0.0:10912 \
    --grpc-address 0.0.0.0:10901 \
    --query.replica-label replica \
    --store 127.0.0.1:19190 \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192 && echo "Started Thanos Querier!"
```


## Thanos Query Frontend

What if we can have one single point of entry in front of Queries instead of separate Queriers? And by doing so slice and dice our queries depend on the time and distribute them between queriers to balance the load? Moreover, why not cache these responses so that next time someone asks for the same time range we can just serve it from memory. Wouldn't it be faster?

Yes, we can do all these using Thanos Query Frontend. Let's see how we can do it

#### First let's deploy a nginx proxy to simulate latency
We are running this tutorial on a single machine in this setup, as a result it's really hard to observe latencies that you would normally experience in a real-life setups. In order to simulate a real-life latency we are going to put a proxy in front of our Thanos Querier.

For that let's setup a nginx instance:
```
server {
 listen 10902;
 server_name proxy;
 location / {
  echo_exec @default;
 }
 location ^~ /api/v1/query_range {
     echo_sleep 1;
     echo_exec @default;
 }
 location @default {
     proxy_pass http://127.0.0.1:10912;
 }
}
```

```
docker run -d --net=host --rm \
    -v $(pwd)/nginx.conf:/etc/nginx/conf.d/default.conf \
    --name nginx \
    yannrobert/docker-nginx && echo "Started Querier Proxy!"
```

#### Deploy Thanos Query Frontend
First, let's create necessary cache configuration for Frontend:
```
type: IN-MEMORY
config:
  max_size: "0"
  max_size_items: 2048
  validity: "6h"
```
And deploy Query Frontend:
```
docker run -d --net=host --rm \
    -v $(pwd)/frontend.yml:/etc/thanos/frontend.yml \
    --name query-frontend \
    quay.io/thanos/thanos:v0.17.0 \
    query-frontend \
    --http-address 0.0.0.0:20902 \
    --query-frontend.compress-responses \
    --query-frontend.downstream-url=http://127.0.0.1:10902 \
    --query-frontend.log-queries-longer-than=5s \
    --query-range.split-interval=1m \
    --query-range.response-cache-max-freshness=1m \
    --query-range.max-retries-per-request=5 \
    --query-range.response-cache-config-file=/etc/thanos/frontend.yml \
    --cache-compression-type="snappy" && echo "Started Thanos Query Frontend!"
```
#### Setup Verification
Once started you should be able to reach the Querier, Query Frontend and Prometheus.


Prometheus
Querier
Query Frontend

Now, go and execute a query on Querier and observe the latency. And then go and execute the same query on Query Frontend. For the fist execution you will observe that the query execution takes longer than the query on Querier. That's because we have an nginx proxy between Query Frontend and Querier.

Now if you execute the same query again on Query Frontend for the same time frame using time selector in graph section in the UI (time is always shifting). See that it's much faster? It's taking much less time because we are just serving the response from the cached results.
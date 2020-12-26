## Step 1 - Configure Prometheus
The Prometheus server requires a configuration file that defines the endpoints to scrape along with how frequently the metrics should be accessed.

The first half of the configuration defines the intervals.
```
global:
  scrape_interval:     15s
  evaluation_interval: 15s
```

The second half defines the servers and ports that Prometheus should scrape data from. In this example, we have defined two targets running on different ports.

```
scrape_configs:
  - job_name: 'prometheus'

    static_configs:
      - targets: ['127.0.0.1:9090', '127.0.0.1:9100']
        labels:
          group: 'prometheus'
```

9090 is Prometheus itself. Prometheus exposes information related to its internal metrics and performance and allows it to monitor itself.

9100 is the Node Exporter Prometheus process. This exposes information about the Node, such as disk space, memory and CPU usage.

## Step 2 - Start Prometheus

The following command launches the container with the prometheus configuration. Any data created by prometheus will be stored on the host, in the directory /prometheus/data. When we update the container, the data will be persisted.

```
docker run -d --net=host \
    -v /root/prometheus.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-server \
    prom/prometheus
```

Once started, the dashboard is viewable on port 9090. The next steps will explain the details and how to view the data.

## Step 3 - Start Node Exporter

To collect metrics related to a node it's required to run a Prometheus Node Exporter. Prometheus has many exporters that are designed to output metrics for a particular system, such as Postgres or MySQL.

Launch the Node Exporter container. By mounting the host /proc and /sys directory, the container has accessed to the necessary information to report on.

```
docker run -d \
  -v "/proc:/host/proc" \
  -v "/sys:/host/sys" \
  -v "/:/rootfs" \
  --net="host" \
  --name=prometheus \
  quay.io/prometheus/node-exporter:v0.13.0 \
    -collector.procfs /host/proc \
    -collector.sysfs /host/sys \
    -collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```

If you're interested in seeing the raw metrics, they can be viewed with `curl localhost:9100/metrics`

## Step 4 - View Metrics

With the containers launched, Prometheus will scrape and store the data based on the internals in the configuration.

Dashboards

The default Prometheus Dashboard reports internal information and provides a way to debug the metrics being collected.

The dashboard will report the status of the scraping and the different targets at via the /targets page.

When running in production, you may wish to build a dashboard using Grafana, or a hosted Prometheus instance such as Weave Cortex.

Query Prometheus

To query the underlying metrics and create graphs, visit the graph page on the Dashboard
From here different metrics are queryable based on their name.

For example, querying for `node_network_receive_bytes` will show how active the disk IO is. Querying using `node_cpu` will output the Docker Hosts CPU information.
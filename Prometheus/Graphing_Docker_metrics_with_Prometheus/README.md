## Step 1 - Enable Metrics

The first task is to enable the new experimental flag that will enable the metrics feature and define the metrics address to listen on localhost:9323.

The command below will update the systemd configuration used to start Docker to set the flags when the daemon starts and then restarts Docker.

```
echo '{ "metrics-addr" : "127.0.0.1:9323", "experimental" : true }' > /etc/docker/daemon.json
systemctl restart docker
```

The metrics endpoint will be accessible once Docker has begun. You can see the raw metrics using `curl localhost:9323/metrics`

These metrics are outputted in the Prometheus format and designed to be scraped by a Prometheus server which will launch in the next steps.

## Step 2 - Configure Prometheus

The Prometheus server requires a configuration file that defines the endpoints to scrape along with how frequently the metrics should be accessed.

The first half of the configuration defines the intervals.

```
global:
  scrape_interval:     15s
  evaluation_interval: 15s
```

The second half defines the servers and ports that Prometheus should scrape data from. In this example, we have defined three targets running on different ports.

```
scrape_configs:
  - job_name: 'prometheus'

    static_configs:
      - targets: ['127.0.0.1:9090', '127.0.0.1:9100', '127.0.0.1:9323']
        labels:
          group: 'prometheus'
```

9090 is Prometheus itself. Prometheus exposes information related to its internal metrics and performance.

9100 is the Node Exporter Prometheus process. This exposes information about the Node, such as disk space, memory and CPU usage.

9323 is the Docker Metrics port just exposed. This reports information from the Docker Engine.

## Step 3 - Start Prometheus

Prometheus runs as a Docker Container with a UI available on port 9090. Prometheus uses the configuration to scrape the targets, collect and store the metrics before making them available via API that allows dashboards, graphing and alerting.

The following command launches the container with the Prometheus configuration. Any data created by Prometheus will be stored on the host, in the directory /prometheus/data. When we update the container, the data will be persisted.

```
docker run -d --net=host \
    -v /root/prometheus.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-server \
    prom/prometheus
```

Once started, the dashboard. The next steps will explain the details and how to view the data.

## Step 4 - Start Node Exporter

The Docker Metrics endpoint will return information related to Docker. To collect metrics on the Docker Host it's required to run a Prometheus Node Exporter. Prometheus has many exporters that are designed to output metrics for a particular system, such as Postgres or MySQL.

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

## Step 5 - View Metrics

With the containers launched, Prometheus will scrape and store the data based on the internals in the configuration.

Dashboards

The default Prometheus Dashboard reports internal information and provides a way to debug the metrics being collected.

The dashboard will report the status of the scraping and the different targets at via the /targets page.

When running in production, you may wish to build a dashboard using Grafana, or a hosted Prometheus instance such as Weave Cortex.

To query the underlying metrics and create graphs, visit the graph page.

From here different metrics are queryable based on their name. This is done by adding a graph and entering a PromQL query or metrics label.

For example, querying for `engine_daemon_container_actions_seconds_sum` will show how many Docker actions are being performed. These actions are containers being started, deleted, created, committed, or changed.

The host metrics can be viewed too, for example, `node_cpu` will output the Docker Hosts CPU information.

Generate Metrics

Running additional containers will result in changes to the metrics produced, which are viewable via the graphs and queries.
```
docker run -d katacoda/docker-http-server:latest
```
# Platform Monitoring

## Description

With the release of Workflow Beta4 we now include a monitoring stack for introspection on a running Kubernetes cluster. The stack includes 4 components:

* [Telegraf](https://docs.influxdata.com/telegraf/v0.12/) - Metrics collection daemon written by team behind InfluxDB.
* [InfluxDB](https://docs.influxdata.com/influxdb/v0.12/) - Time series database
* [Grafana](http://grafana.org/) - Graphing tool for time series data
* [Stdout-Metrics](https://github.com/deis/stdout-metrics) - Tool for consuming metrics via standard out and forwards them to InfluxDB

## Architecture Diagram

```
                         ┌────────┐                  
                         │ Router │                  
                         └────────┘                  
                             │                      
                             │                      
                             ▼          ┌──────────┐
┌─────────────┐         ┌─────────┐     │ stdout   │
│ HOST        │         │ fluentd │────▶│  metrics │
│  Telegraf   │───┐     └─────────┘     └──────────┘
└─────────────┘   │                           │     
                  │                           │     
┌─────────────┐   │                           │     
│ HOST        │   │    ┌───────────┐          │     
│  Telegraf   │───┼───▶│ InfluxDB  │◀─────────┘     
└─────────────┘   │    └───────────┘                
                  │          │                      
┌─────────────┐   │          │                      
│ HOST        │   │          ▼                      
│  Telegraf   │───┘    ┌──────────┐                 
└─────────────┘        │ Grafana  │                 
                       └──────────┘                              
```

### Grafana

Deis Workflow exposes Grafana through the router using [service annotations](https://github.com/deis/router#how-it-works). This
allows users to access the Grafana UI at `http://grafana.mydomain.com`. The default username/password of
`admin/admin` can be overridden at any time by setting the following environment variables in
`$CHART_HOME/workspace/workflow-$WORKFLOW_RELEASE/manifests/deis-monitor-grafana-rc.yaml`: `GRAFANA_USER` and
`GRAFANA_PASSWD`.

Grafana will preload several dashboards to help operators get started with monitoring Kubernetes and Deis Workflow.
These dashboards are meant as starting points and don't include every item that might be desirable to monitor in a
production installation.

Deis Workflow monitoring does not currently write data to the host filesystem or to long-term storage. If the Grafana
instance fails, modified dashboards are lost. Until there is a solution to persist this, export dashboards and store
them separately in version control.

### InfluxDB

As of the Beta4 release InfluxDB is writing data to the host disk, however, if the InfluxDB pod dies and comes back on
another host the data will not be recovered. We intend to fix this in a future release. The InfluxDB Admin UI is also
exposed through the router allowing users to access the query engine by going to `influx.mydomain.com`. You will need to
configure where to find the `influx-api` endpoint by clicking the "gear" icon at the top right and changing the host to
`influxapi.mydomain.com` and port to `80`.

** Note: Each user accessing the Influx UI will need to make this change. **

You can choose to not expose the Influx UI and API to the world by updating
`$CHART_HOME/workspace/workflow-$WORKFLOW_RELEASE/manifests/deis-monitor-influxdb-api-svc.yaml` and
`$CHART_HOME/workspace/workflow-$WORKFLOW_RELEASE/manifests/deis-monitor-influxdb-ui-svc.yaml` and removing the
following line - `router.deis.io/routable: "true"`.

### Telegraf

Telegraf is the metrics collection daemon used within the monitoring stack. It will collect and send the following metrics to InfluxDB:

* System level metrics such as CPU, Load Average, Memory, Disk, and Network stats
* Container level memtrics such as CPU and Memory
* Kubernetes metrics such as API request latency, Pod Startup Latency, and number of running pods

It is possible to send these metrics to other endpoints besides InfluxDB. For more information please consult the following [file](https://github.com/deis/monitor/blob/master/telegraf/rootfs/config.toml.tpl)

### Stdout-Metrics

Stdout-Metrics is a custom tool built by the Deis team to provide metrics that are reported via standard out - like Nginx. It consumes the log stream from FluentD filtering out messages that are not from the [Deis Router](https://github.com/deis/router). Once it finds a message it can parse it will turn that into a metric and send it directly to InfluxDB.

### Customizing

Each of these components allows for customization via environment variables. If you would like to learn more please visit the following github repositories:

* [stdout-metrics](https://github.com/deis/stdout-metrics)
* [monitor](https://github.com/deis/monitor)

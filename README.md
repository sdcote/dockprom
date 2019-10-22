# dockprom - DevOps Monitoring wid Docker and Prometheus

A monitoring solution for Docker hosts and containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [cAdvisor](https://github.com/google/cadvisor),
[NodeExporter](https://github.com/prometheus/node_exporter) and alerting with [AlertManager](https://github.com/prometheus/alertmanager).

This project is designed to run on a standard Windows 10 workstation with Docker installed. This may be your personal laptop to experiment with the technologies, or you may have a "Wall PC" supporting your teams information radiator which is always on and displaying your teams status pages.

This is designed and intended to support DevOps practices, not the support of production applications. You may decide to use this project as the starting point, but your team should go through the normal development process to implement a production monitoring solution.

## Containers

* **Prometheus** (metrics database) `http://<host-ip>:9090`
* Prometheus-**Pushgateway** (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
* **AlertManager** (alerts management) `http://<host-ip>:9093`
* **Grafana** (visualize metrics) `http://<host-ip>:3000`
* **NodeExporter** (host metrics collector)
* **cAdvisor** (containers metrics collector)
* **Caddy** (reverse proxy and basic auth provider for prometheus and alertmanager)

## Install

Three simple commands from your host should be all that is needed to get started.

### Prerequisites:
* Git >= 2.1
* Docker Engine >= 1.13
* Docker Compose >= 1.11

### Quickstart
To get started quickly, follow the following three steps:

Clone this repository on your Docker host, cd into `dockprom` directory and run `compose up`:

```
git clone https://github.com/sdcote/dockprom
cd dockprom
docker-compose up -d
```
For new installs, Docker will start downloading the container images for the products listed above. These are currently being taken from DockerHub but in the future, these images will probably be official AEP images vetted by our security and operations teams.  

This will get everything installed using the default usernames and passwords. _**Only use this for testing and experimentation**_

The Grafana system has its own user management system is listening on port 3000. The default administration credentials are for Grafana are `admin:changeme`. To access the Prometheus time series database (port 9090) and the Prometheus AlertManager (port 9093) you will need to authenticate with a reverse proxy. The default credentials for these two components are `admin:changeme`. The Prometheus Push Gateway is also protected with a reverse proxy, but with a different set of credentials, the defaults are `metrics:monitor`. 

### Real Start

You don't want to accept the default usernames and passwords when you implement this for your team. You should edit the Grafana admin username and password in the `docker-compose.yml` (lines 79 & 80) and those of the reverse proxy (lines 129 - 132). The admin credentials are used for managing Grafana and the proxy credentials protect all access to the Prometheus components. There are two sets of proxy credentials, one set is those for the admin user to access Prometheus and the Alert Manager. Tho other is for the Push Gateway which allows authenticated access to manage metrics to be pushed to the Prometheus time series database.   

_**Security Notice**_ it is highly recommended that you choose appropriate passwords for the proxy user above. It will be the primary credentials used to access your time series database, push gateway and alert manager. If you want to push metrics to the `PushGateway`, you will have to provide a username and password to the reverse proxy using basic authentication. Do not use the defaults and strongly consider keeping the PUSH credentials different from any of the others. Remote monitors will be using these credentials to access your monitoring system and poor configuration on their part could expose the administration capabilities of your system.


## Setup Grafana

### Initial Login
Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You will be encouraged to change the admin password. You **should** do this.

When the system is restarted (`docker-compose up`), the credentials will be reset back to the defaults in the `docker-compose.yml` file. It is recommended to stop and start the containers as required and avoid completely resetting the system with `docker-compose down` as it will remove all resources but for the data volumes. All users will be lost.

### Add other users

For a DevOps team, you will probably only need a "viewer" account. This can be any account name you want to use. This is the account your information radiator will use to view your teams dashboard.

1. As the **admin** user, click on the "Invite your team" icon or click the gear icon on the left hand side of the screen and select "users".

1. Press the "Invite" button. Adding users is based on sending invites, but this project is not expected to be connected to the corporate email at this point so we will not be using emails, but just the links. Enter the following information for the account:
    * Username (do not use email until you configure Grafana to use the corporate email system)
    * Full name of thes
    * The role
    * Deselect the send invite email
1. Press the invite button.
1. Select the "Pending Invites (#)" tab on the users screen.
1. Select the "Copy Invite" link to copy the activation link for that user to the clipboard.
1. Open another tab in your browser and paste the link to your address panel. Make sure to replace `localhost` with the hostname of the docker host.
1. Complete the user registration process.

You can repeat this process for as many users as you need. Keep in mind this is just for your team and you probably only need two or three accounts in your DevOps monitor.

## Dashboards
Grafana is preconfigured with dashboards and Prometheus as the default data source:
* Name: Prometheus
* Type: Prometheus
* Url: http://prometheus:9090
* Access: proxy


***Docker Host Dashboard***

The Docker Host Dashboard shows key metrics for monitoring the resource usage of your server:

* Server uptime, CPU idle percent, number of CPU cores, available memory, swap and storage
* System load average graph, running and blocked by IO processes graph, interrupts graph
* CPU usage graph by mode (guest, idle, iowait, irq, nice, softirq, steal, system, user)
* Memory usage graph by distribution (used, free, buffers, cached)
* IO usage graph (read Bps, read Bps and IO time)
* Network usage graph by device (inbound Bps, Outbound Bps)
* Swap usage and activity graphs

For storage and particularly Free Storage graph, you have to specify the fstype in grafana graph request.
You can find it in `grafana/dashboards/docker_host.json`, at line 480 :

      "expr": "sum(node_filesystem_free_bytes{fstype=\"btrfs\"})",

If you work on BTRFS, you may want to change `aufs` to `btrfs`.

You can find right value for your system in Prometheus `http://<host-ip>:9090` launching this request :

      node_filesystem_free_bytes

***Docker Containers Dashboard***

The Docker Containers Dashboard shows key metrics for monitoring running containers:

* Total containers CPU load, memory and storage usage
* Running containers graph, system load graph, IO usage graph
* Container CPU usage graph
* Container memory usage graph
* Container cached memory usage graph
* Container network inbound usage graph
* Container network outbound usage graph

Note that this dashboard doesn't show the containers that are part of the monitoring stack.

***Monitor Services Dashboard***

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

* Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
* Container CPU usage graph
* Container memory usage graph
* Prometheus chunks to persist and persistence urgency graphs
* Prometheus chunks ops and checkpoint duration graphs
* Prometheus samples ingested rate, target scrapes and scrape duration graphs
* Prometheus HTTP requests graph
* Prometheus alerts graph

## Define alerts

Three alert groups have been setup within the [alert.rules](https://github.com/sdcote/dockprom/blob/master/prometheus/alert.rules) configuration file:

* Monitoring services alerts [targets](https://github.com/sdcote/dockprom/blob/master/prometheus/alert.rules#L2-L11)
* Docker Host alerts [host](https://github.com/sdcote/dockprom/blob/master/prometheus/alert.rules#L13-L40)
* Docker Containers alerts [containers](https://github.com/sdcote/dockprom/blob/master/prometheus/alert.rules#L42-L69)

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```
curl -X POST http://admin:changeme@<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
- alert: monitor_service_down
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Monitor service non-operational"
      description: "Service {{ $labels.instance }} is down."
```

***Docker Host alerts***

Trigger an alert if the Docker host CPU is under high load for more than 30 seconds:

```yaml
- alert: high_cpu_load
    expr: node_load1 > 1.5
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server under high load"
      description: "Docker host is under high load, the avg load 1m is at {{ $value}}. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

Modify the load threshold based on your CPU cores.

Trigger an alert if the Docker host memory is almost full:

```yaml
- alert: high_memory_load
    expr: (sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) ) / sum(node_memory_MemTotal_bytes) * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server memory is almost full"
      description: "Docker host memory usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

Trigger an alert if the Docker host storage is almost full:

```yaml
- alert: high_storage_load
    expr: (node_filesystem_size_bytes{fstype="aufs"} - node_filesystem_free_bytes{fstype="aufs"}) / node_filesystem_size_bytes{fstype="aufs"}  * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server storage is almost full"
      description: "Docker host storage usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

***Docker Containers alerts***

Trigger an alert if a container is down for more than 30 seconds:

```yaml
- alert: jenkins_down
    expr: absent(container_memory_usage_bytes{name="jenkins"})
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Jenkins down"
      description: "Jenkins container is down for more than 30 seconds."
```

Trigger an alert if a container is using more than 10% of total CPU cores for more than 30 seconds:

```yaml
- alert: jenkins_high_cpu
    expr: sum(rate(container_cpu_usage_seconds_total{name="jenkins"}[1m])) / count(node_cpu_seconds_total{mode="system"}) * 100 > 10
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Jenkins high CPU usage"
      description: "Jenkins CPU usage is {{ humanize $value}}%."
```

Trigger an alert if a container is using more than 1.2GB of RAM for more than 30 seconds:

```yaml
- alert: jenkins_high_memory
    expr: sum(container_memory_usage_bytes{name="jenkins"}) > 1200000000
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Jenkins high memory usage"
      description: "Jenkins memory consumption is at {{ humanize $value}}."
```

## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml](https://github.com/sdcote/dockprom/blob/master/alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

## Sending metrics to the Pushgateway

The [pushgateway](https://github.com/prometheus/pushgateway) is used to collect data from batch jobs or from services.

To push data, simply execute:

    echo "some_metric 3.14" | curl --data-binary @- http://user:password@localhost:9091/metrics/job/some_job

Please replace the `user:password` part with your user and password set in the initial configuration (default: `admin:changeme`).

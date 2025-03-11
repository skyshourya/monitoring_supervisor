# Supervisord Monitoring with Prometheus, Grafana, and Alertmanager

## Overview

This setup enables real-time monitoring of processes managed by Supervisord using the **Supervisor Exporter**, Prometheus, and Grafana. It also includes Alertmanager for sending notifications when a supervised process goes down.

## Table of Contents

- [Features](#features)
- [Getting Started](#getting-started)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Prometheus Metrics](#prometheus-metrics)
- [Alerting Setup](#alerting-setup)
- [License](#license)

## Features

- Collects process status information from Supervisord.
- Exposes process state, exit status, and more as Prometheus metrics.
- Configurable via command-line parameters.
- Provides a simple HTTP server for Prometheus to scrape metrics.
- Handles unreachable Supervisord XML-RPC endpoints gracefully.

## Getting Started

### Prerequisites

Before running the Supervisor Exporter, ensure the following:

- **Supervisord is installed and running**.
- **Supervisord XML-RPC interface is enabled**.

### Enabling the XML-RPC Endpoint

Edit the **Supervisord Configuration** file (typically `/etc/supervisor/supervisord.conf`):

```ini

[inet_http_server]
port = xx.xx.xx.xx:9002  # for all IPs use 0.0.0.0 
username = admin
password = admin 
```

Restart Supervisord to apply the changes:

```sh
supervisorctl reread
supervisorctl update
```

Verify the XML-RPC endpoint:

```sh
curl http://<HOST_IP>:9002/RPC2
```

## Installation

### First, check if Go is installed:
```sh
go version
```


### If it's not installed, install it using:
```sh
sudo apt update
sudo apt install -y golang
```

### Clone the Repository

```sh
git clone https://github.com/salimd/supervisord_exporter.git
cd supervisord_exporter
go build -o supervisord_exporter supervisord_exporter.go
ls -l supervisord_exporter
```

## Usage

Run the Supervisor Exporter with:

```sh
./supervisord_exporter -supervisord-url="http://admin:admin@xx.xx.xx.xx:9002/RPC2" -web.listen-address=":9876" -web.telemetry-path="/metrics"

```
                                                    OR                                                 
```sh
./supervisord_exporter --supervisord-url "http://admin:admin@127.0.0.1:9002/RPC2"
```

### By default, the exporter listens on **port 9876** and fetches data from [**http://localhost:9001/RPC2**](http://localhost:9001/RPC2).
## but here we have used port 9002.


## Docker Compose Configuration on the host where Prometheus, Grafana, and Alertmanager will be running.
Create a `docker-compose.yml` file with the following content:
```bash
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert.rules.yml:/etc/prometheus/alert.rules.yml
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    ports:
      - "9090:9090"
    depends_on:
      - alertmanager

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
    ports:
      - "9093:9093"

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
    grafana-data:

```

### Start the Monitoring Stack
```bash
docker-compose up -d
```

## Alerting Setup on the same host as prometheus

### Alert Rule (Prometheus `alert.rules.yml`)

```yaml
  - alert: SupervisordProcessDown
    expr: supervisor_process_up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Supervisord process is down"
      description: "Process {{ $labels.name }} in group {{ $labels.group }} is down."
```

### Alertmanager Configuration (`alertmanager.yml`) on the same host as prometheus

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 5s
  group_interval: 10s
  repeat_interval: 1m
  receiver: 'slack-notifications'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - send_resolved: true
        channel: '#sunny'
       api_url: 'https://hooks.slack.com/services/YOUR_SLACK_WEBHOOK_URL'

```

Check if an alert is triggered in Prometheus and sent via Alertmanager.

## Conclusion
This setup ensures **continuous monitoring** of Supervisord-managed processes. Any failures will be alerted via **Slack notifications**. You can further customize dashboards and alerts based on specific use cases.

---


Maintainer: **Shourya Yadav**



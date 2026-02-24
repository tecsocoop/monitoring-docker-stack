# monitoring-docker-stack

Simple monitoring stack based on Docker Compose.

## Server Installation

| Component    | Version |
|--------------|---------|
| Grafana      | 12.3.3  |
| Loki         | 3.6.5   |
| Prometheus   | v3.9.1  |
| Alertmanager | v0.31.1 |

### 1. Environment variables

Copy `.env.example` to `.env` and fill in the values:

```
cp .env.example .env
```

Edit `.env` with the Grafana credentials and, optionally, the domain if accessed
via a reverse proxy or ELB.

### 2. Alertmanager

Copy the configuration template and fill in the Slack webhook:

```
cp prometheus/alertmanager/alertmanager.yml.example prometheus/alertmanager/alertmanager.yml
```

Edit `prometheus/alertmanager/alertmanager.yml` and replace:
- `api_url`: Slack webhook URL (https://api.slack.com/messaging/webhooks)
- `channel`: Slack channel where alerts are sent

The `alertmanager.yml` file is listed in `.gitignore` and is not versioned.

### 3. Add nodes to monitor

```
cp prometheus/targets/nodes.json.example prometheus/targets/nodes.json
```

Edit `prometheus/targets/nodes.json` by adding one entry per node:

```json
[
  {
    "targets": ["NODE_IP:9100"],
    "labels": {
      "instance": "descriptive-name",
      "env": "production"
    }
  }
]
```

Prometheus reloads this file automatically every 1 minute without needing to
restart the container.

### 4. Start the stack

```
docker compose up -d
```

Grafana will be available at `http://HOST:3000`.

# Node Agent (nodes/monitoring-docker-node)

## Agent Installation on a Node

The `nodes/monitoring-docker-node` directory contains the compose file for the
agent that runs on each machine to be monitored.

| Component     | Version |
|---------------|---------|
| Fluent Bit    | 3.2.10  |
| Node Exporter | v1.9.1  |

### 1. Requirements

- The node must have network connectivity to the monitoring server.
- Required ports from the node to the server:
  - `3100` (Loki, for log shipping)

### 2. Configure the Loki endpoint

Edit `nodes/monitoring-docker-node/fluentbit/fluent-bit.conf` and update the
`Host` field in the `[OUTPUT]` section with the IP or hostname of the monitoring
server:

```
[OUTPUT]
    Host    SERVER_IP
```

### 3. Start the agent

```
docker compose up -d
```

### 4. Register the node in Prometheus

Add the node's IP to `prometheus/targets/nodes.json` on the server
(see step 3 of the server installation).


---

## Optional Features

### Loki

Install Docker plugin: 

```
docker plugin install grafana/loki-docker-driver:3.6.5-<amd64/arm64> --alias loki --grant-all-permissions
docker plugin enable loki
```

add to compose: 

```
services:
  app:
    image: postgres:latest
    restart: always
    ports:
      - 5432:5432
    logging:
      driver: loki
      options:
        loki-url: "http://localhost:3100/loki/api/v1/push
        loki-external-labels: container_name={{.Name}},env=$ENVIRONMENT # custom labels
```

### Grafana

#### Screenshots in Grafana alerts

Uncomment the `grafana-image-renderer` service in `docker-compose.yml` and the
`[unified_alerting.screenshots]` and `[rendering]` sections in
`grafana/grafana.ini`.

#### Google OAuth

Uncomment the `[auth.google]` section in `grafana/grafana.ini` and add the
`GF_AUTH_GOOGLE_CLIENT_ID` and `GF_AUTH_GOOGLE_CLIENT_SECRET` variables to `.env`.



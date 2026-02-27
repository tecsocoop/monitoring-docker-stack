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

## Docker logs to Loki

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

## Metrics 

### node-exporter

Based on our tests, we recommend installing Prometheus node-exporter directly on the monitored host.

example: 

```
sudo apt install prometheus-node-exporter
```

### cadvisor

Based on our tests, we recommend installing cadvisor with docker running: 

```
VERSION=0.55.1 # use the latest release version from https://github.com/google/cadvisor/releases
sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=18080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  ghcr.io/google/cadvisor:$VERSION # for versions prior to v0.53.0, use gcr.io/cadvisor/cadvisor instead
```

> port was changed for compatibility (18080)

## Grafana

### Screenshots in Grafana alerts

Uncomment the `grafana-image-renderer` service in `docker-compose.yml` and the
`[unified_alerting.screenshots]` and `[rendering]` sections in
`grafana/grafana.ini`.

### Google OAuth

Uncomment the `[auth.google]` section in `grafana/grafana.ini` and add the
`GF_AUTH_GOOGLE_CLIENT_ID` and `GF_AUTH_GOOGLE_CLIENT_SECRET` variables to `.env`.



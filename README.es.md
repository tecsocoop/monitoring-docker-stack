# monitoring-docker-stack

Stack de monitoreo simple sobre docker compose.

> [English version](README.md)

## Instalacion del servidor

| Componente   | Version |
|--------------|---------|
| Grafana      | 12.3.3  |
| Loki         | 3.6.5   |
| Prometheus   | v3.9.1  |
| Alertmanager | v0.31.1 |

### 1. Variables de entorno

Copiar `.env.example` a `.env` y completar los valores:

```
cp .env.example .env
```

Editar `.env` con las credenciales de Grafana y, opcionalmente, el dominio si se
accede via reverse proxy o ELB.

### 2. Alertmanager

Copiar el template de configuracion y completar el webhook de Slack:

```
cp prometheus/alertmanager/alertmanager.yml.example prometheus/alertmanager/alertmanager.yml
```

Editar `prometheus/alertmanager/alertmanager.yml` y reemplazar:
- `api_url`: webhook URL de Slack (https://api.slack.com/messaging/webhooks)
- `channel`: canal de Slack donde se envian las alertas

El archivo `alertmanager.yml` esta en `.gitignore` y no se versiona.

### 3. Agregar nodos a monitorear

```
cp prometheus/targets/nodes.json.example prometheus/targets/nodes.json
```

Editar `prometheus/targets/nodes.json` agregando una entrada por cada nodo:

```json
[
  {
    "targets": ["IP_DEL_NODO:9100"],
    "labels": {
      "instance": "nombre-descriptivo",
      "env": "production"
    }
  }
]
```

Prometheus recarga este archivo automaticamente cada 1 minuto sin necesidad de
reiniciar el contenedor.

### 4. Iniciar el stack

```
docker compose up -d
```

Grafana queda disponible en `http://HOST:3000`.

# Agente de nodo (nodes/monitoring-docker-node)

## Instalacion del agente en un nodo

El directorio `nodes/monitoring-docker-node` contiene el compose para el agente
que corre en cada maquina a monitorear.

| Componente    | Version |
|---------------|---------|
| Fluent Bit    | 3.2.10  |
| Node Exporter | v1.9.1  |

### 1. Requisitos

- El nodo debe tener conectividad de red al servidor de monitoreo.
- Los puertos requeridos desde el nodo hacia el servidor son:
  - `3100` (Loki, para envio de logs)

### 2. Configurar el endpoint de Loki

Editar `nodes/monitoring-docker-node/fluentbit/fluent-bit.conf` y actualizar el
campo `Host` en la seccion `[OUTPUT]` con la IP o hostname del servidor de
monitoreo:

```
[OUTPUT]
    Host    IP_DEL_SERVIDOR
```

### 3. Iniciar el agente

```
docker compose up -d
```

### 4. Registrar el nodo en Prometheus

Agregar la IP del nodo en `prometheus/targets/nodes.json` en el servidor
(ver paso 3 de la instalacion del servidor).


---

## Funcionalidades opcionales

### Screenshots en alertas de Grafana

Descomentar en `docker-compose.yml` el servicio `grafana-image-renderer` y en
`grafana/grafana.ini` las secciones `[unified_alerting.screenshots]` y `[rendering]`.

### Google OAuth

Descomentar la seccion `[auth.google]` en `grafana/grafana.ini` y agregar las
variables `GF_AUTH_GOOGLE_CLIENT_ID` y `GF_AUTH_GOOGLE_CLIENT_SECRET` en `.env`.

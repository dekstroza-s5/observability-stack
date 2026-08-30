# Observability Stack

Local observability platform based on Prometheus, Grafana, Alertmanager, Loki, Promtail and node_exporter. It provides metrics, alerts and centralized Linux logs in a single Docker Compose environment.

## Data flow

```text
node_exporter ----> Prometheus ----> alert rules ----> Alertmanager
                         |
                         +----------> Grafana
/var/log ----> Promtail ----> Loki ------^
```

Prometheus scrapes infrastructure metrics, evaluates alert rules and forwards firing alerts. Promtail tails host log files and sends them to Loki. Grafana is provisioned with both data sources.

## Components

| Component | Port | Purpose |
|---|---:|---|
| Grafana | 3000 | dashboards and log exploration |
| Prometheus | 9090 | metrics, queries and rules |
| Alertmanager | 9093 | alert grouping and routing |
| Loki | 3100 | log storage and queries |
| node_exporter | 9100 | host metrics |

## Prerequisites

- Docker Engine 24+
- Docker Compose v2
- Linux host for the node_exporter mount and system log example
- approximately 2 GB free memory

## Validate configuration

```bash
docker compose config
docker run --rm -v "$PWD/prometheus:/etc/prometheus:ro"   prom/prometheus:v3.2.1 promtool check config /etc/prometheus/prometheus.yml
docker run --rm -v "$PWD/prometheus:/etc/prometheus:ro"   prom/prometheus:v3.2.1 promtool check rules /etc/prometheus/rules.yml
```

## Start

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Expected state:

```text
prometheus      running
alertmanager    running
grafana         running
loki            running
promtail        running
node-exporter   running
```

Follow startup logs when a component is not ready:

```bash
docker compose logs --tail=100 prometheus
docker compose logs --tail=100 loki
docker compose logs --tail=100 grafana
```

## First checks

Prometheus targets:

```bash
curl --fail http://localhost:9090/-/ready
curl --silent http://localhost:9090/api/v1/targets
```

Metrics query:

```promql
100 - avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

Filesystem usage:

```promql
100 * (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes)
```

In Grafana, open **Explore**, choose Prometheus for metrics or Loki for logs. Example LogQL query:

```logql
{job="varlogs"} |= "error"
```

## Alert test

Temporarily stop node_exporter:

```bash
docker compose stop node-exporter
```

After the configured `for` interval, `TargetDown` should appear in Prometheus and Alertmanager. Restore it:

```bash
docker compose start node-exporter
```

## Persistence and retention

Prometheus, Grafana and Loki use named volumes. Prometheus keeps seven days of samples in this lab configuration. Back up Grafana dashboards and provisioning files rather than relying only on the container volume.

## Production adaptations

- place endpoints behind authentication and TLS;
- change the Grafana administrator password;
- send alerts to an actual receiver;
- restrict published ports;
- use object storage and appropriate retention for Loki;
- add application scrape targets and recording rules;
- apply resource limits and external backups.

## Troubleshooting

- Prometheus target down: check DNS inside the Compose network and the target's metrics endpoint.
- no host logs: verify the host path, file permissions and Promtail positions.
- Grafana data source error: test connectivity from the Grafana container.
- Loki rejects samples: check timestamps, schema and container clock.
- alert not firing: use the Prometheus expression browser and inspect rule evaluation.

## Stop and reset

```bash
docker compose down
# Remove stored lab data only when intentionally resetting:
docker compose down -v
```

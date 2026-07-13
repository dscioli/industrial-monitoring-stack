# Integrating with modbus-collector

The stack ships with a `modbus-collector` scrape job targeting
`collector:8080`. Three ways to make that name resolve:

## Option A — the combined demo (recommended)

The [`modbus-collector`](https://github.com/dscioli/modbus-collector) repo's
`docker-compose.demo.yml` runs everything (collector + 3 simulated stations +
this stack's Prometheus/Grafana + the operator dashboard) on one network:

```bash
cd ../modbus-collector
docker compose -f docker-compose.demo.yml up -d
```

It mounts this repo's `prometheus/` and `grafana/` directories directly, so
the dashboards and rules here are the ones that light up.

## Option B — join this stack to the collector's network

Run each compose separately, then attach Prometheus to the collector's
network so the `collector` service name resolves:

```bash
# in modbus-collector/
docker compose up -d
# in industrial-monitoring-stack/
docker compose up -d
docker network connect modbus-collector_default industrial-monitoring-stack-prometheus-1
```

(Compose network/container names vary with your project directory names;
`docker network ls` shows them.)

## Option C — collector running elsewhere

Edit the target in [`prometheus/prometheus.yml`](../prometheus/prometheus.yml):

```yaml
  - job_name: modbus-collector
    scrape_interval: 15s
    static_configs:
      - targets: ["192.0.2.10:8080"]   # your collector host:port
```

Also update the `blackbox-tcp` targets if you want reachability probes for
the Modbus listener (host port 502).

## What you should see

- Prometheus → Status → Targets: `modbus-collector` **UP**, scraped every 15 s.
- Grafana → Industrial → **Fleet status**: stations online, tank levels moving.
- Grafana → **Tank overview**, station 3: the raw-vs-filtered level pair.
- Silencing a station (stop a simulator) fires `StationOffline` after 5 m,
  visible in Prometheus → Alerts and delivered to your webhook by
  Alertmanager.

## Threshold coupling

`TankLevelLow`/`TankLevelHigh` in
[`prometheus/rules/telemetry.rules.yml`](../prometheus/rules/telemetry.rules.yml)
hardcode the demo fleet's control thresholds (180 / 420 cm from the
collector's `stations.yaml`). If you change one, change the other — the
collector's own alert engine and these Prometheus rules are deliberately
independent layers (application alerting vs. observability alerting).

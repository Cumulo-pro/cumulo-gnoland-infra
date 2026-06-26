# 📊 Gnoland Test13 — Monitoring & Metrics

This document describes Cumulo's monitoring stack for the Gnoland Test13 validator infrastructure, based on **OpenTelemetry Collector → Prometheus → Grafana**.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    VALIDATOR NODE                           │
│  cumveliamon-t                                             │
│                                                             │
│  gnoland-test13 ──OTLP──▶ OTEL Collector :4317            │
│                            Prometheus metrics :8889         │
└─────────────────────────────────┬───────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────┐
│                      RPC NODE                               │
│  cumvelia2                                                 │
│                                                             │
│  gnoland-test13 ──OTLP──▶ OTEL Collector :4317            │
│                            Prometheus metrics :8890         │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │   Central Prometheus    │
                    └────────────┬────────────┘
                                 │
                                 ▼
                         ┌──────────────┐
                         │    Grafana   │
                         └──────────────┘
```

---

## Components

### 1. Gnoland Telemetry (OTLP)

Gnoland natively exports metrics via **OTLP/gRPC**. The telemetry configuration is applied via CLI:

```bash
gnoland config set telemetry.metrics_enabled true   -config-path gnoland-data/config/config.toml
gnoland config set telemetry.exporter_endpoint "localhost:4317" -config-path gnoland-data/config/config.toml
gnoland config set telemetry.service_name "gnoland-test13" -config-path gnoland-data/config/config.toml
gnoland config set telemetry.service_instance_id "cumulo-validator-cumveliamon-t" -config-path gnoland-data/config/config.toml
```

> ⚠️ **Important:** The `exporter_endpoint` must **not** include `http://`. gRPC does not use the HTTP prefix — using `http://localhost:4317` will cause the OTEL Collector to receive no data.

### 2. OpenTelemetry Collector

**Version:** `otelcol-contrib v0.155.0`

The OTEL Collector acts as an intermediary: it receives OTLP metrics from Gnoland and re-exposes them in Prometheus format for scraping.

**Configuration file:** [`otel-collector-config.yaml`](otel-collector-config.yaml)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"   # validator node
    # endpoint: "0.0.0.0:8890" # RPC node

processors:
  batch:

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

| Node | OTLP receiver | Prometheus exporter |
|------|--------------|---------------------|
| Validator (`cumveliamon-t`) | `4317` | `8889` |
| RPC (`cumvelia2`) | `4317` | `8890` |

**Systemd service:** `otelcol-contrib`

```bash
sudo systemctl enable --now otelcol-contrib
sudo systemctl status otelcol-contrib
```

### 3. Central Prometheus

**Endpoint:** `http://<PROMETHEUS_IP>:9099`

Add the following scrape jobs to `prometheus.yml`:

```yaml
scrape_configs:

  - job_name: 'gnoland-test13-cumveliamon-t'
    static_configs:
      - targets: ['<VALIDATOR_NODE_IP>:8889']

  - job_name: 'gnoland-test13-cumvelia2'
    static_configs:
      - targets: ['<RPC_NODE_IP>:8890']
```

Reload after editing:

```bash
curl -X POST http://localhost:9099/-/reload
# or
sudo systemctl restart prometheus
```

### 4. Grafana Dashboard

**Dashboard file:** [`grafana-dashboard.json`](grafana-dashboard.json)

The dashboard covers:

- **Consensus** — block height, round, voting power
- **Mempool** — pending transactions, bytes
- **P2P** — connected peers, inbound/outbound traffic
- **Gas** — gas price histogram (`block_gas_price_hist`)
- **Node info** — service name, version, instance ID

---

## Public Endpoints

| Resource | URL |
|----------|-----|
| RPC (validator node) | `http://<VALIDATOR_NODE_IP>:26657/status` |
| RPC (public endpoint) | `https://gno.rpc.testnet.cumulo.me` |
| OTEL metrics (validator) | `http://<VALIDATOR_NODE_IP>:8889/metrics` |
| OTEL metrics (RPC node) | `http://<RPC_NODE_IP>:8890/metrics` |

---

## Key Metrics

Gnoland (TM2) exposes metrics under the `tm2` OpenTelemetry scope. Key metrics available:

| Metric | Description |
|--------|-------------|
| `consensus_height` | Current block height |
| `consensus_validators` | Number of validators in the active set |
| `consensus_validators_power` | Total voting power |
| `consensus_missing_validators` | Validators missing from the last commit |
| `consensus_rounds` | Number of rounds per block |
| `mempool_size` | Pending transactions in mempool |
| `mempool_size_bytes` | Mempool size in bytes |
| `p2p_peers` | Number of connected peers |
| `block_gas_price_hist` | Gas price histogram |

> All metrics carry labels: `job`, `instance`, `service_name`, `service_instance_id`, `otel_scope_name`.

Verify metrics are being received:

```bash
# Validator node
curl -s http://<VALIDATOR_NODE_IP>:8889/metrics | grep -v "^#" | head -20

# RPC node
curl -s http://<RPC_NODE_IP>:8890/metrics | grep -v "^#" | head -20
```

---

## Troubleshooting

**No metrics from OTEL Collector**
- Check that `telemetry.exporter_endpoint` does **not** include `http://`
- Verify OTEL Collector is running: `sudo systemctl status otelcol-contrib`
- Check OTEL Collector logs: `sudo journalctl -u otelcol-contrib -f`

**Port conflicts**
- Velia2 uses non-standard ports for Gnoland (`P2P: 26670`, `RPC: 26671`) and OTEL exporter (`8890`) due to other colocated services

**Target not appearing in Prometheus**
- Verify firewall allows the metrics port: `sudo ufw allow 8889/tcp`
- Test connectivity from Prometheus server: `curl -s http://<VALIDATOR_NODE_IP>:8889/metrics | head -5`

---

## Files in This Directory

| File | Description |
|------|-------------|
| `README.md` | This document |
| `otel-collector-config.yaml` | OTEL Collector config (validator node) |
| `otel-collector-config-rpc.yaml` | OTEL Collector config (RPC node) |
| `grafana-dashboard.json` | Grafana dashboard JSON for import |
| `prometheus-scrape-jobs.yaml` | Prometheus scrape job snippets |

---

*Maintained by [Cumulo](https://cumulo.pro) — validator infrastructure for Gnoland Test13*

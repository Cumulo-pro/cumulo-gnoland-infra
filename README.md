# cumulo-gnoland-infra

Infrastructure documentation for Cumulo's Gnoland Test13 validator and RPC nodes — installation guides, configuration references, monitoring stack, and operational runbooks.

---

## Overview

[Gno.land](https://gno.land) is a next-generation smart contract platform built on Tendermint2 (TM2), using an interpreted, deterministic variant of Go. Unlike Cosmos SDK chains, Gnoland exposes only an RPC endpoint (no REST, no gRPC) and uses OTLP for native telemetry.

This repository documents Cumulo's full infrastructure for **Test13**, the active Gnoland testnet, covering both the validator node and the public RPC node.

---

**Key identifiers**

| Parameter | Value |
|-----------|-------|
| Chain ID | `test13` |
| Operator address | `g1mndkvypn3339m6yvkyutal5hk6m2yeqp5d52ly` |
| Public RPC | `https://gno.rpc.testnet.cumulo.me` |

---

## Repository Structure

```
cumulo-gnoland-infra/
└── test-13/
    ├── install/
    │   └── INSTALL.md                         # Full node installation guide
    ├── configs/
    │   └── config.toml                        # Reference node configuration
    ├── scripts/
    │   └── (operational scripts)
    ├── metrics/
    │   ├── README.md                          # Monitoring stack documentation
    │   ├── otel-collector-config.yaml         # OTEL Collector (validator node)
    │   ├── otel-collector-config-rpc.yaml     # OTEL Collector (RPC node)
    │   ├── prometheus-scrape-jobs.yaml        # Prometheus scrape job snippets
    │   └── grafana-dashboard.json             # Grafana dashboard (import-ready)
    └── GNOLAND_CLI_COMMAND_REFERENCE.md       # Full CLI reference
```

---

## Documentation

### Installation

→ [`test-13/install/INSTALL.md`](test-13/install/INSTALL.md)

Complete step-by-step guide covering:

- Binary compilation from the `chain/test13` branch of `gnolang/gno`
- Genesis download and SHA256 verification
- Config and secrets initialization via CLI (`-config-path` flag)
- Systemd service setup with required `GNOROOT` environment variable
- Wallet creation with `gnokey` and on-chain validator registration via Valoper realm

> ⚠️ **Gnoland-specific gotchas documented:** `GNOROOT` must be set in the systemd `Environment=` directive; `secrets init` places files in a wrong path by default; `config init` uses `-config-path`, not `--data-dir`; gRPC telemetry endpoints must omit the `http://` prefix.

---

### CLI Command Reference

→ [`test-13/GNOLAND_CLI_COMMAND_REFERENCE.md`](test-13/GNOLAND_CLI_COMMAND_REFERENCE.md)

Complete reference covering:

- Systemd operations (start, stop, restart, status, logs)
- Node status and sync verification
- Peer management (Gnoland holds peers in memory — no `addrbook.json`)
- Key and wallet management with `gnokey`
- Balance and validator queries
- On-chain calls (Valoper realm registration and updates)
- Config and secrets management
- Destructive cleanup commands

---

### Monitoring

→ [`test-13/metrics/README.md`](test-13/metrics/README.md)

Full OTEL → Prometheus → Grafana stack documentation:

```
Gnoland (OTLP) → OpenTelemetry Collector → Central Prometheus → Grafana
```

- OTEL Collector v0.155.0 configuration for both nodes
- Prometheus scrape job configuration
- Grafana dashboard (JSON, ready to import)
- Key TM2 metrics reference (`consensus_height`, `mempool_size`, `p2p_peers`, `block_gas_price_hist`, …)

---

## Public Resources

| Resource | URL |
|----------|-----|
| Public RPC | [https://gno.rpc.testnet.cumulo.me](https://gno.rpc.testnet.cumulo.me) |
| OTEL Metrics (validator) | `http://<VALIDATOR_NODE_IP>:8889/metrics` |
| OTEL Metrics (RPC node) | `http://<RPC_NODE_IP>:8890/metrics` |

---

## About Cumulo

[Cumulo](https://cumulo.pro) is a professional validator operation providing infrastructure, community tooling, and educational content across multiple blockchain networks.

**Active networks:** Celestia · Espresso · Cosmos Hub · Dymension · Axone · Warden · Quicksilver · Seda · Avail · Starknet · Concordium · Fuel · XRP EVM

**Tooling:** Block Explorer · Live Peers · IBC Relayer (Hermes) · Activity Tracker · Validator Resources · OTL (Operational Transparency Layer)

**Community:** 500+ Spanish-speaking developers · [Medium](https://medium.com/cumulo-pro) · [X / Twitter](https://twitter.com/Cumulo_pro)

---

*Maintained by [Cumulo](https://cumulo.pro)*

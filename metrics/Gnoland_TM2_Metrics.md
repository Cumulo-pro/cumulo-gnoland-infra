# Gnoland TM2 Consensus Metrics Documentation

## Introduction

This document provides a comprehensive reference for the metrics used in **Gnoland**, a blockchain platform built on **TM2** (Tendermint 2), Gno.land's own consensus engine. Unlike Cosmos SDK chains that expose CometBFT metrics directly in Prometheus format, Gnoland uses **OpenTelemetry (OTEL)** to collect and export metrics, which are then scraped by Prometheus via an OTEL Collector.

The metrics are organized into categories covering block production, P2P network health, mempool activity, VM execution, gas usage, and validator set participation. Each metric is explained with its description, value interpretation, Prometheus query examples, and real values observed on the **Test13** testnet by **Cumulo**.

For ease of navigation, refer to the Table of Contents below.

> **Infrastructure:** Metrics collected from `cumulo-validator-cumveliamon-t` on Test13 (`test-13`), exported via OTEL Collector and scraped by Prometheus. Dashboard available at [Cumulo Grafana](https://cumulo.pro/services/).

---

## Table of Contents

- [BLOCK INTERVAL](#block-interval)
- [BLOCK SIZE](#block-size)
- [TRANSACTIONS PER BLOCK](#transactions-per-block)
- [GAS PRICE](#gas-price)
- [INBOUND PEERS](#inbound-peers)
- [OUTBOUND PEERS](#outbound-peers)
- [VALIDATOR COUNT](#validator-count)
- [VALIDATOR VOTING POWER](#validator-voting-power)
- [MEMPOOL SIZE](#mempool-size)
- [CACHED TRANSACTIONS](#cached-transactions)
- [VM EXECUTIONS](#vm-executions)
- [VM GAS USED](#vm-gas-used)
- [VM CPU CYCLES](#vm-cpu-cycles)
- [BLOCK PRODUCTION RATE](#block-production-rate)
- [HOW TO USE THESE METRICS](#how-to-use-these-metrics)
- [EXAMPLE PROMETHEUS OUTPUT](#example-prometheus-output)

---

## BLOCK INTERVAL

### Metric: `block_interval_hist_seconds`

**Description:**  
A histogram measuring the time elapsed between consecutive blocks. This is one of the most important metrics for assessing consensus health, it reflects how quickly the network reaches agreement and finalizes blocks.

**Value Interpretation:**

| Percentile | Description |
|---|---|
| `p50` | Median block time, typical interval under normal conditions |
| `p90` | 90th percentile, covers most blocks including slight delays |
| `p99` | 99th percentile, worst-case block times, useful for detecting stalls |

**Prometheus Query Examples:**
```promql
# Median block interval
histogram_quantile(0.5, rate(block_interval_hist_seconds_bucket{job="gnoland-test13-validator"}[10m]))

# p99 block interval
histogram_quantile(0.99, rate(block_interval_hist_seconds_bucket{job="gnoland-test13-validator"}[10m]))

# Block production rate (blocks per second)
rate(block_interval_hist_seconds_count{job="gnoland-test13-validator"}[10m])
```

**Example Values (Test13, Cumulo):**

| Percentile | Mean | Max | Min |
|---|---|---|---|
| p50 | 2.51 s | 2.54 s | 2.50 s |
| p90 | 4.51 s | 4.57 s | 4.50 s |
| p99 | 5.22 s | 6.70 s | 4.95 s |

**Interpretation:**
- **p50 ~2.5s** is expected on Test13 with `timeout_commit = 3s`
- **p99 > 6s** may indicate network latency or a round requiring more than one attempt
- **Sudden spikes in p99** warrant investigation of peer connectivity and consensus rounds

---

## BLOCK SIZE

### Metric: `block_size_hist_B`

**Description:**  
A histogram measuring the size of each block in bytes. Tracks how much data is being packed into each block over time.

**Value Interpretation:**

| Percentile | Description |
|---|---|
| `p50` | Median block size under normal transaction load |
| `p90` | Blocks with higher transaction density |
| `p99` | Largest blocks, useful for detecting storage growth trends |

**Prometheus Query Examples:**
```promql
# Median block size in bytes
histogram_quantile(0.5, rate(block_size_hist_B_bucket{job="gnoland-test13-validator"}[10m]))

# p99 block size
histogram_quantile(0.99, rate(block_size_hist_B_bucket{job="gnoland-test13-validator"}[10m]))
```

**Example Values (Test13, Cumulo):**

| Percentile | Mean | Max |
|---|---|---|
| p50 | 3.85 KiB | 3.91 KiB |
| p90 | 5.69 KiB | 6.16 KiB |
| p99 | 7.68 KiB | 8.68 KiB |

**Interpretation:**
- Blocks under 10 KiB indicate low-to-moderate transaction activity
- **Growing p99 values** may indicate increased on-chain activity or large contract deployments
- **Sudden size spikes** may correlate with governance proposals or batch transactions

---

## TRANSACTIONS PER BLOCK

### Metric: `block_txs_hist`

**Description:**  
A histogram measuring the number of transactions included in each block. Helps evaluate network utilization and transaction throughput.

**Prometheus Query Examples:**
```promql
# Median transactions per block
histogram_quantile(0.5, rate(block_txs_hist_bucket{job="gnoland-test13-validator"}[10m]))

# Block transaction rate
rate(block_txs_hist_sum{job="gnoland-test13-validator"}[10m]) / rate(block_txs_hist_count{job="gnoland-test13-validator"}[10m])
```

**Interpretation:**
- **Low values (< 5 txs/block)** indicate a lightly loaded network, expected on testnets
- **High values sustained over time** suggest increasing adoption or stress testing
- **Blocks with 0 transactions** are normal on testnets during low-activity periods

---

## GAS PRICE

### Metric: `block_gas_price_hist_token`

**Description:**  
A histogram tracking the gas price per block, labeled by function (`LastGasPrice`, `UpdateGasPrice`). Gas price in Gnoland is denominated in `ugnot`.

**Labels:**

| Label | Description |
|---|---|
| `func="LastGasPrice"` | The gas price of the most recently finalized block |
| `func="UpdateGasPrice"` | Gas price updates applied during block processing |

**Prometheus Query Examples:**
```promql
# Median gas price (last block)
histogram_quantile(0.5, rate(block_gas_price_hist_token_bucket{job="gnoland-test13-validator", func="LastGasPrice"}[10m]))

# p99 gas price
histogram_quantile(0.99, rate(block_gas_price_hist_token_bucket{job="gnoland-test13-validator", func="LastGasPrice"}[10m]))
```

**Example Values (Test13, Cumulo):**

| Percentile | Value |
|---|---|
| p50 | 875 ugnot |
| p99 | 998 ugnot |

**Interpretation:**
- A stable gas price around 875–1000 ugnot indicates predictable network costs
- **Gas price spikes** may indicate high transaction load or block space competition
- **Persistent low gas prices** are typical on testnets with low activity

---

## INBOUND PEERS

### Metric: `inbound_peers_gauge`

**Description:**  
A gauge representing the current number of inbound P2P connections, peers that have connected to this node from the network.

**Prometheus Query Examples:**
```promql
# Current inbound peers
inbound_peers_gauge{job="gnoland-test13-validator"}

# Inbound peers over time
inbound_peers_gauge{job="gnoland-test13-validator"}
```

**Example Values (Test13, Cumulo):**

| Metric | Value |
|---|---|
| Current | 40 |
| Mean (3h) | 40 |
| Min (3h) | 40 |
| Max (3h) | 40 |

**Interpretation:**
- **40 inbound peers** indicates this node is at the configured `max_num_inbound_peers` limit
- **Sudden drop to 0** would indicate a P2P layer failure or network isolation
- **Consistently at max** is healthy, the node is well-known in the network

---

## OUTBOUND PEERS

### Metric: `outbound_peers_gauge`

**Description:**  
A gauge representing the current number of outbound P2P connections, peers this node has actively dialed and connected to.

**Prometheus Query Examples:**
```promql
# Current outbound peers
outbound_peers_gauge{job="gnoland-test13-validator"}

# Total peers (inbound + outbound)
inbound_peers_gauge{job="gnoland-test13-validator"} + outbound_peers_gauge{job="gnoland-test13-validator"}
```

**Example Values (Test13, Cumulo):**

| Metric | Value |
|---|---|
| Current | 14 |
| Mean (3h) | 12.8 |
| Min (3h) | 9 |
| Max (3h) | 18 |

**Interpretation:**
- **Outbound fluctuation** is normal, peers come and go as the network evolves
- **Persistent outbound < 3** may indicate connectivity issues or misconfigured `persistent_peers`
- **Note:** TM2 does not persist the address book between restarts, so peer discovery restarts from scratch on each node reboot

---

## VALIDATOR COUNT

### Metric: `validator_count_hist`

**Description:**  
A histogram recording the number of validators observed per block. Captures how many validators are actively participating in consensus over time.

**Prometheus Query Examples:**
```promql
# Average validator count
validator_count_hist_sum{job="gnoland-test13-validator"} / validator_count_hist_count{job="gnoland-test13-validator"}

# Validator count over time (average per block)
rate(validator_count_hist_sum{job="gnoland-test13-validator"}[10m]) / rate(validator_count_hist_count{job="gnoland-test13-validator"}[10m])
```

**Example Values (Test13, Cumulo):**

| Metric | Value |
|---|---|
| Average validators per block | 10 |

**Interpretation:**
- **10 validators** is the current size of the Test13 active validator set
- In Gnoland, new validators start with **voting power 1** and can receive increases via GovDAO proposals
- **Decreasing count** may indicate validators going offline or being removed from the set

---

## VALIDATOR VOTING POWER

### Metric: `validator_vp_hist`

**Description:**  
A histogram recording the distribution of voting power across validators per block. Useful for assessing decentralization and stake concentration.

**Prometheus Query Examples:**
```promql
# Median voting power
histogram_quantile(0.5, rate(validator_vp_hist_bucket{job="gnoland-test13-validator"}[10m]))

# Maximum voting power (p99)
histogram_quantile(0.99, rate(validator_vp_hist_bucket{job="gnoland-test13-validator"}[10m]))
```

**Example Values (Test13, Cumulo):**

| Percentile | Value |
|---|---|
| Median VP | 62.5 |
| Max VP (p99) | 74.8 |

**Interpretation:**
- A **median VP of 62.5** with max ~75 suggests moderate concentration among top validators
- In Test13, initial voting power is **1 per validator**, higher values indicate validators that have received GovDAO power increases
- **High p99/median ratio** may indicate centralization of stake; monitoring this over time supports decentralization analysis

---

## MEMPOOL SIZE

### Metric: `num_mempool_txs_hist`

**Description:**  
A histogram measuring the number of transactions currently in the mempool, transactions that have been broadcast but not yet included in a block.

**Prometheus Query Examples:**
```promql
# Median mempool size
histogram_quantile(0.5, rate(num_mempool_txs_hist_bucket{job="gnoland-test13-validator"}[10m]))
```

**Example Values (Test13, Cumulo):**

| Metric | Value |
|---|---|
| Current (p50) | ~1.84 txs |
| Max (3h) | 2.13 txs |

**Interpretation:**
- **Near-zero mempool** is normal on a lightly loaded testnet
- **Growing mempool** indicates transactions are arriving faster than blocks can include them
- **Persistent large mempool** may signal block size limits being reached or consensus slowdowns

---

## CACHED TRANSACTIONS

### Metric: `num_cached_txs_hist`

**Description:**  
A histogram measuring the number of transactions cached by the node, transactions that have been seen and stored but may not yet be in the active mempool.

**Prometheus Query Examples:**
```promql
# Median cached txs
histogram_quantile(0.5, rate(num_cached_txs_hist_bucket{job="gnoland-test13-validator"}[10m]))
```

**Example Values (Test13, Cumulo):**

| Metric | Value |
|---|---|
| Current (p50) | ~375 txs |
| Mean (3h) | 254 txs |

**Interpretation:**
- **Cached txs > mempool txs** is normal, the cache retains recently seen transactions to avoid reprocessing
- **Very high cache values** could indicate memory pressure on long-running nodes
- Cache grows organically with network activity and is bounded by the mempool configuration

---

## VM EXECUTIONS

### Metric: `vm_exec_msg_counter_total`

**Description:**  
A counter tracking the total number of messages (transactions/calls) executed by the Gnoland virtual machine since process start. This is the primary indicator of on-chain computation activity.

**Prometheus Query Examples:**
```promql
# VM execution rate (messages per second)
rate(vm_exec_msg_counter_total{job="gnoland-test13-validator"}[10m])

# VM messages per hour
increase(vm_exec_msg_counter_total{job="gnoland-test13-validator"}[1h])
```

**Example Values (Test13, Cumulo):**

| Metric | Value |
|---|---|
| Execution rate | ~0.854 msg/s |
| Messages per hour | ~1.24K |

**Interpretation:**
- **~0.85 msg/s** reflects moderate testnet activity
- **Rate spikes** indicate bursts of contract calls or governance activity
- This metric is a proxy for overall network utilization and developer activity

---

## VM GAS USED

### Metric: `vm_gas_used_hist`

**Description:**  
A histogram measuring the amount of gas consumed per VM execution. Reflects the computational cost of on-chain operations.

**Prometheus Query Examples:**
```promql
# Median gas per execution
histogram_quantile(0.5, rate(vm_gas_used_hist_bucket{job="gnoland-test13-validator"}[10m]))

# p99 gas per execution
histogram_quantile(0.99, rate(vm_gas_used_hist_bucket{job="gnoland-test13-validator"}[10m]))
```

**Interpretation:**
- **High gas usage per execution** indicates complex contract interactions or large state changes
- **Consistently maxed-out buckets** may indicate the histogram range needs adjustment for the specific workload
- Useful for detecting expensive operations that could impact block processing time

---

## VM CPU CYCLES

### Metric: `vm_cpu_cycles_hist`

**Description:**  
A histogram measuring the number of CPU cycles consumed per VM execution. Provides a low-level view of computational intensity for each on-chain operation.

**Prometheus Query Examples:**
```promql
# Median CPU cycles per execution
histogram_quantile(0.5, rate(vm_cpu_cycles_hist_bucket{job="gnoland-test13-validator"}[10m]))

# p99 CPU cycles
histogram_quantile(0.99, rate(vm_cpu_cycles_hist_bucket{job="gnoland-test13-validator"}[10m]))
```

**Interpretation:**
- **High CPU cycles** correlate with complex Gno programs (loops, large data structures, recursive calls)
- **Monitoring p99 CPU cycles** helps identify pathological contract executions that could degrade node performance
- Complements `vm_gas_used_hist`, a high-cycle low-gas execution may indicate gas pricing calibration issues

---

## BLOCK PRODUCTION RATE

### Metric: derived from `block_interval_hist_seconds_count`

**Description:**  
The rate at which the node is observing new blocks, derived from the count of block interval observations. Expressed in blocks per second.

**Prometheus Query Examples:**
```promql
# Block production rate (blocks/s)
rate(block_interval_hist_seconds_count{job="gnoland-test13-validator"}[10m])
```

**Example Values (Test13, Cumulo):**

| Metric | Value |
|---|---|
| Mean rate | ~0.199 blocks/s |
| Max rate | ~0.250 blocks/s |

**Interpretation:**
- **~0.2 blocks/s** corresponds to ~1 block every 5 seconds, consistent with p50 block interval of 2.5s
- **Rate dropping to 0** indicates a chain halt or node disconnection
- **Rate exceeding expected** may indicate the node has fallen behind and is catching up

---

## How to Use These Metrics

- **Monitor node sync status:** Track `block_interval_hist_seconds_count` rate, a drop to 0 means the node has stopped seeing blocks.
- **Detect consensus delays:** Use `block_interval_hist_seconds` p90 and p99 to identify rounds requiring multiple attempts.
- **Track network health:** Monitor `inbound_peers_gauge` + `outbound_peers_gauge`, a sudden drop indicates P2P issues.
- **Evaluate validator participation:** Use `validator_count_hist` to detect validators going offline.
- **Assess decentralization:** Track `validator_vp_hist` p99 vs median to monitor voting power concentration.
- **Identify transaction bursts:** Correlate `num_mempool_txs_hist` spikes with `vm_exec_msg_counter_total` rate.
- **Monitor computational cost:** Use `vm_cpu_cycles_hist` p99 to detect expensive contract executions.
- **Estimate gas costs:** Track `block_gas_price_hist_token` to understand fee trends on the network.

---

## Example Prometheus Output

```
# HELP block_interval_hist_seconds_bucket Histogram of block intervals in seconds
# TYPE block_interval_hist_seconds_bucket histogram
block_interval_hist_seconds_bucket{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator",le="1.0"} 0
block_interval_hist_seconds_bucket{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator",le="3.0"} 450
block_interval_hist_seconds_bucket{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator",le="5.0"} 498
block_interval_hist_seconds_bucket{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator",le="+Inf"} 500
block_interval_hist_seconds_count{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator"} 500
block_interval_hist_seconds_sum{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator"} 1257.3

# HELP inbound_peers_gauge Current number of inbound P2P connections
# TYPE inbound_peers_gauge gauge
inbound_peers_gauge{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator"} 40

# HELP outbound_peers_gauge Current number of outbound P2P connections
# TYPE outbound_peers_gauge gauge
outbound_peers_gauge{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator"} 14

# HELP vm_exec_msg_counter_total Total VM message executions since process start
# TYPE vm_exec_msg_counter_total counter
vm_exec_msg_counter_total{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator"} 44821

# HELP num_mempool_txs_hist_bucket Histogram of mempool transaction counts
# TYPE num_mempool_txs_hist_bucket histogram
num_mempool_txs_hist_bucket{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator",le="5.0"} 280
num_mempool_txs_hist_bucket{instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator",le="+Inf"} 312

# HELP block_gas_price_hist_token_bucket Histogram of gas prices per block
# TYPE block_gas_price_hist_token_bucket histogram
block_gas_price_hist_token_bucket{func="LastGasPrice",instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator",le="1000.0"} 59
block_gas_price_hist_token_sum{func="LastGasPrice",instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator"} 59000
block_gas_price_hist_token_count{func="LastGasPrice",instance="cumulo-validator-cumveliamon-t",job="gnoland-test13-validator"} 59
```

---

*Document maintained by [Cumulo](https://cumulo.pro), Gnoland Test13 Validator*  
*Chain: `test-13` | Node: `cumulo-validator-cumveliamon-t` | Updated: June 2026*

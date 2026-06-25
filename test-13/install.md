# Gno.land Test13 — Node & Validator Installation Guide

> **Cumulo Infrastructure Runbook**
> Network: `test-13` | Binary: `gnoland` | Chain: Gno.land Testnet

---

## Overview

Gno.land differs from standard Cosmos SDK chains in several important ways:

- **Custom binary** — `gnoland` is compiled from the monorepo; no prebuilt releases.
- **Config via CLI** — configuration is set with `gnoland config set`, not by editing TOML directly.
- **Two separate identities** — the operator address (`g1...`) is your on-chain wallet; the consensus pubkey (`gpub1...`) belongs to the node validator key. These are completely independent.
- **No `addrbook.json`** — peer discovery is handled differently from Cosmos SDK; peers are obtained via `net_info` RPC endpoint.

---

## Prerequisites

| Requirement | Version |
|---|---|
| OS | Ubuntu 22.04 / 24.04 |
| Go | 1.22+ |
| Git | Any recent version |
| Open ports | `26656/tcp` (P2P), `26657/tcp` (RPC) |

### Install Go (if needed)

```bash
go version  # must be 1.22+
```

### Set GNOROOT permanently

```bash
echo 'export GNOROOT=$HOME/gno' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 1 — Compile gnoland from source

```bash
cd $HOME
git clone https://github.com/gnolang/gno.git
cd gno
git checkout chain/test13
make -C gno.land install.gnoland install.gnokey

# Verify
gnoland version   # must show: chain/test13
```

The binary is installed at `$GOPATH/bin/gnoland` (typically `$HOME/go/bin/gnoland`).

---

## Step 2 — Create node directory

```bash
mkdir -p $HOME/gnoland-test13/gnoland-data/config
mkdir -p $HOME/gnoland-test13/gnoland-data/secrets
cd $HOME/gnoland-test13
```

---

## Step 3 — Download and verify genesis

```bash
wget -O gnoland-data/genesis.json \
  https://github.com/gnolang/gno/releases/download/chain/test13/genesis.json

# Verify SHA256
shasum -a 256 gnoland-data/genesis.json
# Expected: 56f56e135174feff9f93283d5ec7e4ec955cd5155108aff5009d4fd51c5adaf2
```

---

## Step 4 — Initialize config and secrets

```bash
# Initialize config
gnoland config init -config-path gnoland-data/config/config.toml

# Initialize secrets (generates priv_validator_key, node_key, priv_validator_state)
gnoland secrets init -data-dir gnoland-data/secrets
```

> ⚠️ The secrets directory contains your validator signing key. Back it up securely.

---

## Step 5 — Configure network parameters

```bash
cd $HOME/gnoland-test13

# Set chain ID
gnoland config set chainid test-13 \
  -config-path gnoland-data/config/config.toml

# Set official Test13 persistent peers
gnoland config set p2p.persistent_peers \
  "g142k7zc2qym3c0u6jmkf6rv26llgr2f4nakmlmt@54.145.44.95:26656,g1lxkf9gn7kddrr26c640ww5wg3ezsm22we8cjpc@99.81.240.125:26656,g17su28ydtj8jsdqt2c7m3jn3mysqlz6n57vxd5t@62.210.125.225:26656,g1uycj5lkvu97jddywjttd8xq53u3p6eyhh2js25@62.210.124.8:26656,g1zz2yvc23ts7uk05gemxmj9dlrhdt7w8pxtcdy5@62.210.207.58:26656,g12gxe0qpq90vhhpp5gtavafgr2nl9cntntvrjkj@186.233.184.95:37656" \
  -config-path gnoland-data/config/config.toml
```

---

## Step 6 — Create systemd service

```bash
sudo tee /etc/systemd/system/gnoland-test13.service << EOF
[Unit]
Description=Gno.land Test13 Node
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/gnoland-test13
ExecStart=$HOME/go/bin/gnoland start \
  --chainid test-13 \
  --genesis $HOME/gnoland-test13/gnoland-data/genesis.json \
  --data-dir $HOME/gnoland-test13/gnoland-data \
  --skip-genesis-sig-verification
Environment=GNOROOT=$HOME/gno
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now gnoland-test13
```

---

## Step 7 — Verify sync

```bash
# Check service status
sudo systemctl status gnoland-test13

# Follow logs
journalctl -u gnoland-test13 -f

# Check sync status via RPC
curl -s http://localhost:26657/status | python3 -m json.tool | grep -E '"catching_up"|"latest_block_height"'
```

The node is fully synced when `catching_up` is `false`.

---

## Step 8 — Create operator wallet

```bash
gnokey add cumulo --home $HOME/gnoland-test13/gnoland-data/secrets
```

> Save the mnemonic in a secure location. This is your operator address (`g1...`).

To recover an existing wallet:
```bash
gnokey add cumulo --recover --home $HOME/gnoland-test13/gnoland-data/secrets
```

---

## Step 9 — Get consensus pubkey

```bash
gnoland secrets get ValidatorPrivateKey \
  -data-dir $HOME/gnoland-test13/gnoland-data/secrets
```

Note the `pub_key` value (`gpub1...`) — you will need it for validator registration.

---

## Step 10 — Register validator on-chain

```bash
gnokey maketx call \
  -pkgpath "gno.land/r/gnops/valopers" \
  -func "Register" \
  -args "<CONSENSUS_PUBKEY>" \
  -args "Cumulo" \
  -args "https://cumulo.pro" \
  -gas-fee 1000000ugnot \
  -gas-wanted 10000000 \
  -broadcast \
  -chainid test-13 \
  -remote https://rpc.test13.testnets.gno.land:443 \
  cumulo \
  --home $HOME/gnoland-test13/gnoland-data/secrets
```

Replace `<CONSENSUS_PUBKEY>` with the `gpub1...` value from Step 9.

---

## Cumulo Node Reference

| Parameter | Value |
|---|---|
| Moniker | `Cumulo` |
| Operator Address | `g1mndkvypn3339m6yvkyutal5hk6m2yeqp5d52ly` |
| Consensus Address | `g1kxrqt5yrhtccs5zpghpc85t6fyxvpu3mlnk76e` |
| Consensus PubKey | `gpub1pggj7ard9eg82cjtv4u52epjx56nzwgjyg9zqrvt4nx6rwyz08w9l09er9tgf8xsjqvs5radd89s7atequsxkq4563pc0l` |
| Server | `cumveliamon-t` |
| Public IP | `192.155.100.132` |
| Node directory | `$HOME/gnoland-test13/` |
| Binary | `$HOME/go/bin/gnoland` |
| P2P Port | `26656` |
| RPC Port | `26657` |
| Validator profile | [on-chain](https://test13.testnets.gno.land/r/gnops/valopers:g1mndkvypn3339m6yvkyutal5hk6m2yeqp5d52ly) |

---

## Useful Commands

```bash
# Node status
curl -s http://localhost:26657/status | python3 -m json.tool

# Connected peers
curl -s http://localhost:26657/net_info | python3 -m json.tool | grep remote_ip

# Service logs
journalctl -u gnoland-test13 -f --no-hostname

# Restart node
sudo systemctl restart gnoland-test13

# Check wallet balance
gnokey query bank/balances/g1mndkvypn3339m6yvkyutal5hk6m2yeqp5d52ly \
  -remote https://rpc.test13.testnets.gno.land:443
```

---

## Network Reference

| Parameter | Value |
|---|---|
| Chain ID | `test-13` |
| RPC (official) | `https://rpc.test13.testnets.gno.land:443` |
| RPC (Onbloc) | `https://test13.rpc.onbloc.xyz:443` |
| Explorer | `https://test13.testnets.gno.land` |
| Genesis SHA256 | `56f56e135174feff9f93283d5ec7e4ec955cd5155108aff5009d4fd51c5adaf2` |
| Cumulo Live Peers | `https://cumulo.pro` |

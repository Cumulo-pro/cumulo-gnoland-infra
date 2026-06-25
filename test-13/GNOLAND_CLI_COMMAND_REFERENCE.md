# Gno.land Node CLI Command Reference

*(gnoland / gnokey - test-13)*

This document is a **practical operator command reference** for running a **Gno.land** node and validator.
It is intentionally concise, structured, and focused on **day-to-day operations**.

All commands assume:

- Chain ID: `test-13`
- Node binary: `gnoland`
- Key binary: `gnokey`
- Data dir: `$HOME/gnoland-test13/gnoland-data`
- Secrets dir: `$HOME/gnoland-test13/gnoland-data/secrets`
- Remote RPC: `https://rpc.test13.testnets.gno.land:443`

---

## ⚙️ systemd — Service Operations

### Reload systemd units

```bash
sudo systemctl daemon-reload
```

### Start service

```bash
sudo systemctl start gnoland-test13
```

### Stop service

```bash
sudo systemctl stop gnoland-test13
```

### Restart service

```bash
sudo systemctl restart gnoland-test13
```

### Enable service at boot

```bash
sudo systemctl enable gnoland-test13
```

### Disable service

```bash
sudo systemctl disable gnoland-test13
```

### Check service status

```bash
sudo systemctl status gnoland-test13
```

---

## 📜 Logs

### Follow logs (live)

```bash
journalctl -u gnoland-test13 -f
```

### Show last 200 lines

```bash
journalctl -u gnoland-test13 -n 200 --no-hostname -o cat
```

### Filter errors only

```bash
journalctl -u gnoland-test13 -f | grep -i error
```

---

## 📡 Node Status & Sync

### Full node status (JSON)

```bash
curl -s http://localhost:26657/status | python3 -m json.tool
```

### Latest block height

```bash
curl -s http://localhost:26657/status | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['result']['sync_info']['latest_block_height'])"
```

### Syncing status

```bash
curl -s http://localhost:26657/status | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['result']['sync_info']['catching_up'])"
```

### Node ID

```bash
curl -s http://localhost:26657/status | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['result']['node_info']['id'])"
```

---

## 🌐 Peers

### Number of connected peers

```bash
curl -s http://localhost:26657/net_info | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['result']['n_peers'])"
```

### List connected peers (IP only)

```bash
curl -s http://localhost:26657/net_info | python3 -m json.tool | grep remote_ip
```

### List peers in persistent_peers format

```bash
curl -s http://localhost:26657/net_info | python3 -c "
import json, sys
data = json.load(sys.stdin)
for p in data['result']['peers']:
    node_id = p['node_info']['id']
    remote_ip = p['remote_ip']
    port = p['node_info']['listen_addr'].split(':')[-1]
    print(f'{node_id}@{remote_ip}:{port}')
"
```

### Your node peer string

```bash
NODE_ID=$(curl -s http://localhost:26657/status | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['result']['node_info']['id'])")
PUBLIC_IP=$(curl -s ifconfig.me)
echo "${NODE_ID}@${PUBLIC_IP}:26656"
```

---

## 🔑 Keys & Wallet Management

### Add new wallet

```bash
gnokey add $WALLET --home $HOME/gnoland-test13/gnoland-data/secrets
```

### Restore wallet from mnemonic

```bash
gnokey add $WALLET --recover --home $HOME/gnoland-test13/gnoland-data/secrets
```

### List wallets

```bash
gnokey list --home $HOME/gnoland-test13/gnoland-data/secrets
```

### Delete wallet

```bash
gnokey delete $WALLET --home $HOME/gnoland-test13/gnoland-data/secrets
```

### Export wallet key

```bash
gnokey export $WALLET --home $HOME/gnoland-test13/gnoland-data/secrets
```

### Import wallet key

```bash
gnokey import $WALLET --home $HOME/gnoland-test13/gnoland-data/secrets
```

---

## 💰 Balances & Transfers

### Check wallet balance

```bash
gnokey query bank/balances/$WALLET_ADDRESS \
  -remote https://rpc.test13.testnets.gno.land:443
```

### Transfer funds

```bash
gnokey maketx send \
  -to $TO_ADDRESS \
  -send "1000000ugnot" \
  -gas-fee 1000000ugnot \
  -gas-wanted 10000000 \
  -broadcast \
  -chainid test-13 \
  -remote https://rpc.test13.testnets.gno.land:443 \
  $WALLET \
  --home $HOME/gnoland-test13/gnoland-data/secrets
```

---

## 🧑‍⚖️ Validator Operations

### Get consensus pubkey

```bash
gnoland secrets get ValidatorPrivateKey \
  -data-dir $HOME/gnoland-test13/gnoland-data/secrets
```

### Register validator on-chain

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
  $WALLET \
  --home $HOME/gnoland-test13/gnoland-data/secrets
```

### Update validator profile

```bash
gnokey maketx call \
  -pkgpath "gno.land/r/gnops/valopers" \
  -func "UpdateProfile" \
  -args "<FIELD>" \
  -args "<VALUE>" \
  -gas-fee 1000000ugnot \
  -gas-wanted 10000000 \
  -broadcast \
  -chainid test-13 \
  -remote https://rpc.test13.testnets.gno.land:443 \
  $WALLET \
  --home $HOME/gnoland-test13/gnoland-data/secrets
```

### View validator profile on-chain

```bash
# Via browser
https://test13.testnets.gno.land/r/gnops/valopers:$WALLET_ADDRESS
```

---

## ⚙️ Node Configuration

### View current config

```bash
cat $HOME/gnoland-test13/gnoland-data/config/config.toml
```

### Set a config value

```bash
gnoland config set <key> <value> \
  -config-path $HOME/gnoland-test13/gnoland-data/config/config.toml
```

### Set persistent peers

```bash
gnoland config set p2p.persistent_peers \
  "nodeid@ip:26656,nodeid@ip:26656" \
  -config-path $HOME/gnoland-test13/gnoland-data/config/config.toml
```

### Re-initialize config (⚠️ overwrites)

```bash
gnoland config init \
  -config-path $HOME/gnoland-test13/gnoland-data/config/config.toml
```

---

## 🔐 Secrets Management

### List secrets

```bash
gnoland secrets get \
  -data-dir $HOME/gnoland-test13/gnoland-data/secrets
```

### Get validator private key

```bash
gnoland secrets get ValidatorPrivateKey \
  -data-dir $HOME/gnoland-test13/gnoland-data/secrets
```

### Get node key

```bash
gnoland secrets get NodeKey \
  -data-dir $HOME/gnoland-test13/gnoland-data/secrets
```

### Get validator state

```bash
gnoland secrets get ValidatorState \
  -data-dir $HOME/gnoland-test13/gnoland-data/secrets
```

### Re-initialize secrets (⚠️ generates new keys)

```bash
gnoland secrets init \
  -data-dir $HOME/gnoland-test13/gnoland-data/secrets
```

---

## 🏛️ On-chain Calls (gnokey maketx call)

### Generic contract call

```bash
gnokey maketx call \
  -pkgpath "gno.land/r/<realm>" \
  -func "<FunctionName>" \
  -args "<arg1>" \
  -gas-fee 1000000ugnot \
  -gas-wanted 10000000 \
  -broadcast \
  -chainid test-13 \
  -remote https://rpc.test13.testnets.gno.land:443 \
  $WALLET \
  --home $HOME/gnoland-test13/gnoland-data/secrets
```

### Query a realm (read-only)

```bash
gnokey query vm/qrender \
  -data "gno.land/r/<realm>" \
  -remote https://rpc.test13.testnets.gno.land:443
```

---

## 🧹 Cleanup (Destructive)

### Stop service

```bash
sudo systemctl stop gnoland-test13
```

### Remove systemd service

```bash
sudo rm /etc/systemd/system/gnoland-test13.service
sudo systemctl daemon-reload
```

### Remove node data (⚠️ irreversible)

```bash
rm -rf $HOME/gnoland-test13/gnoland-data/db
```

### Remove everything (⚠️ irreversible — includes keys and secrets)

```bash
rm -rf $HOME/gnoland-test13
rm -rf $HOME/gno
```

---

**End of Gno.land CLI Command Reference**

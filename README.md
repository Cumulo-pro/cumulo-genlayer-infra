# GenLayer Validator Node — FAQ

This FAQ compiles the most common doubts when running a **GenLayer Asimov Testnet validator**.

## 📚 Documentation Index

- **Staking Guide**  
  How to stake, deposit, prime epochs, and manage validator status.  
  👉 https://github.com/Cumulo-pro/cumulo-genlayer-infra/blob/main/genlayer_staking_guide.md

- **Key Management**  
  Backup, export, restore, and rotate Owner/Operator keys safely.  
  👉 https://github.com/Cumulo-pro/cumulo-genlayer-infra/blob/main/genlayer-key-management.md

- **Validator FAQ**  
  Quick answers on config, errors, systemd, monitoring, and troubleshooting.  
  👉 https://github.com/Cumulo-pro/cumulo-genlayer-infra/blob/main/faq


---

## 1. Wallets & Roles

### Q1: What is the difference between Owner, Operator and Validator Wallet?

- **Owner**
  - Controls the stake.
  - Signs transactions to create/join/update the validator.
  - Should be stored in a safer environment (cold storage if possible).

- **Operator**
  - Hot wallet on the validator server.
  - Used by the node to sign blocks and participate in consensus.
  - Imported via `genlayernode account import ... --setup`.

- **Validator Wallet (contract)**
  - Smart contract address that represents your validator.
  - Returned by `genlayer staking wizard`.
  - Must be set in `node.validatorWalletAddress` in `config.yaml`.

### Q2: Where are the keystores stored?

- Owner keystore (CLI machine):
  - `~/.genlayer/keystores/owner.json` (path may vary).
- Operator keystore (node):
  - `./data/node/keystore/UTC--...` inside the node directory.
- Recommended backup:
  - Use `genlayernode account export` to produce `backup.key` files and store them securely.

---

## 2. Configuration & `config.yaml`

### Q3: Which fields must be correct for validator mode to work?

At minimum:

```yaml
node:
  mode: "validator"
  validatorWalletAddress: "0x..."
  operatorAddress: "0x..."
```

Plus, correct rollup / consensus config:

```yaml
rollup:
  genlayerchainrpcurl: "https://genlayer-testnet.rpc.caldera.xyz/http"
  genlayerchainwebsocketurl: "wss://genlayer-testnet.rpc.caldera.xyz/ws"

consensus:
  contractmainaddress: "0x..."
  contractdataaddress: "0x..."
  genesis: <block_number>
```

### Q4: Can I override config.yaml with environment variables?

Yes. Any key can be overridden via `GENLAYERNODE_`-prefixed environment variables:

- `rollup.genlayerchainrpcurl` → `GENLAYERNODE_ROLLUP_GENLAYERCHAINRPCURL`
- `node.validatorWalletAddress` → `GENLAYERNODE_NODE_VALIDATORWALLETADDRESS`
- `node.operatorAddress` → `GENLAYERNODE_NODE_OPERATORADDRESS`

Example:

```bash
export GENLAYERNODE_NODE_VALIDATORWALLETADDRESS="0x..."
export GENLAYERNODE_NODE_OPERATORADDRESS="0x..."
```

---

## 3. Node Fails to Start

### Q5: Error: `build dependencies: validator address is not set`

This usually means one of:

1. `node.mode` is `validator` but:
   - `validatorWalletAddress` is empty or invalid, or
   - `operatorAddress` is empty or invalid.
2. No operator account imported with `--setup`.

**Fix:**

1. Check `configs/node/config.yaml`:

```bash
grep -n "validatorWalletAddress\|operatorAddress" configs/node/config.yaml
```

2. Re-import the operator using:

```bash
./bin/genlayernode account import   --password "<NODE_KEYSTORE_PASSWORD>"   --passphrase "<BACKUP_FILE_PASSPHRASE>"   --path "$(pwd)/old-keystores/backup.key"   -c "$(pwd)/configs/node/config.yaml"   --setup
```

3. Re-run:

```bash
./bin/genlayernode doctor -c configs/node/config.yaml
```

### Q6: Error: `genvm: executable file not found in $PATH`

GenVM was not set up yet.

**Fix:**

```bash
python3 ./third_party/genvm/bin/setup.py
```

Then re-run:

```bash
./bin/genlayernode doctor -c configs/node/config.yaml
```

---

## 4. Doctor Command

### Q7: What does `doctor` check?

`./bin/genlayernode doctor -c configs/node/config.yaml` validates:

- zkSync RPC & WebSocket connectivity.
- Consensus contract addresses and genesis.
- Validator wallet & operator configuration.
- GenVM binaries and configs.
- LLM provider environment variables.
- WebDriver availability and response.

### Q8: Doctor fails with WebDriver 404 on `/render`, but node works. Is it a problem?

Usually not critical.  
The `doctor` tool may use an endpoint (`/render`) that older WebDriver images don’t expose.

- If your node logs show GenVM working.
- And WebDriver container is healthy (`docker ps` OK).

…you can keep running while planning a later update to the recommended WebDriver image.

---

## 5. GenVM & LLM

### Q9: How do I enable a single LLM provider (Heurist, example)?

1. In `third_party/genvm/config/genvm-module-llm.yaml`, set:

```yaml
providers:
  heurist:
    enabled: true
    # ...
  comput3:
    enabled: false
  ionet:
    enabled: false
  libertai:
    enabled: false
```

2. Export the API key in the environment (systemd or shell):

```bash
export HEURISTKEY="your_heurist_api_key"
```

3. Restart the node.

---

## 6. Systemd & Operations

### Q10: How do I manage the node with Systemd?

Assuming the unit is named `genlayernode.service`:

```bash
# Start / stop / restart
sudo systemctl start genlayernode.service
sudo systemctl stop genlayernode.service
sudo systemctl restart genlayernode.service

# Status
sudo systemctl status genlayernode.service

# Logs (live)
sudo journalctl -u genlayernode.service -f --no-hostname -o cat
```

### Q11: What are the main ports?

From `config.yaml`:

- RPC: `9151`
- Ops (metrics/health): `9153`
- Admin: `9155` (if used)

---

## 7. Monitoring & Metrics

### Q12: Where can I see metrics?

Once running:

```bash
curl http://127.0.0.1:9153/metrics
```

You can scrape this with Prometheus and build dashboards in Grafana.

### Q13: How can I monitor node health?

- HTTP health endpoint:

```bash
curl http://127.0.0.1:9153/health
```

- Systemd status & journald logs.
- Alerts based on:
  - Node sync status.
  - Consensus participation.
  - GenVM / LLM errors.
  - RPC/ops ports accessibility.

---

## 8. Migration & Recovery

### Q14: How do I move the node to a new server?

1. Install:
   - Docker (for WebDriver).
   - Python3 + `python3-venv`.
2. Copy or re-download the node tarball.
3. Copy `configs/node/config.yaml`.
4. Import the operator key with `--setup` from a secure `backup.key`.
5. Run `python3 ./third_party/genvm/bin/setup.py`.
6. Configure and start WebDriver.
7. Install and start the Systemd service.

### Q15: What MUST be preserved to avoid losing the validator?

- OWNER keystore (controls stake).
- OPERATOR keystore or backup (`backup.key`).
- Validator wallet address (contract).
- Enough GEN balance on operator for gas costs.

Losing the operator key is recoverable (you can change operator), but losing the owner key can mean loss of control over staked funds.

---

## 9. Troubleshooting Snapshot

### Q16: How do I quickly see if the node is running as validator?

Check logs:

```bash
journalctl -u genlayernode.service -f --no-hostname -o cat
```

You should see lines similar to:

```text
Node is synced!!! mode=validator node=0x... operator=0x...
Successfully started mode=validator node=0x... operator=0x...
```

If you see `mode=full` instead, validator-specific config is probably missing or incorrect.

---

## 10. Safety Notes

- Never expose your operator or owner private keys in logs, scripts, or screenshots.
- Use strong passphrases when exporting keystores to `backup.key`.
- Always verify on-chain addresses (contracts, wallets) with official GenLayer docs or trusted community references.

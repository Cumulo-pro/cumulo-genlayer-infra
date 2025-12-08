# GenLayer Validator Node — Cumulo Setup

This repository documents the configuration and operational procedures for running a **GenLayer Asimov Testnet validator node**.

> **Chain:** GenLayer Asimov Testnet  
> **Mode:** `validator`  
> **Node Binary:** `genlayernode` (v0.4.0)  
> **Environment:** Bare-metal / VM with Docker (for WebDriver)

---

## 1. Overview

This node participates as a **validator** in the GenLayer network.  
Responsibilities include:

- Connecting to the **GenLayer zkSync chain** (Caldera endpoint).
- Running **GenVM** with a configured LLM provider.
- Signing and validating blocks using the **operator key**.
- Exposing metrics and health endpoints for monitoring.

This README focuses on:

- File layout and key configuration files.
- Wallet and role design (Owner / Operator / Validator Wallet).
- Systemd service management.
- Operational and maintenance commands.
- Backup and recovery considerations.

---

## 2. Roles & Wallet Design

GenLayer splits responsibilities into three logical roles:

### 2.1 Roles

- **OWNER**
  - Long-term custody of stake.
  - Used to create/join the validator and manage delegations.
  - Should be stored safely (cold wallet wherever possible).
  - Used with the `genlayer` CLI (outside the node).

- **OPERATOR**
  - Hot wallet on the validator server.
  - Signs blocks and participates in consensus.
  - Imported into the node via `genlayernode account import`.
  - Address must match `node.operatorAddress` in `config.yaml`.

- **VALIDATOR WALLET**
  - Smart contract (ValidatorWallet) on-chain.
  - Represents your validator’s identity and stake.
  - Returned by `genlayer staking wizard`.
  - Must be set in `node.validatorWalletAddress` in `config.yaml`.

### 2.2 Typical Mapping (example)

```text
OWNER
  Address: 0xea75b2f766eb89c40d636f11663143f9a618623f
  Keystore: /home/<user>/.genlayer/keystores/owner.json

OPERATOR
  Address: 0x07db557f71ff00817755E13F87165E167C3B95A6
  Keystore (node): ./data/node/keystore/UTC--...

VALIDATOR WALLET
  Address: 0x0F67c2fa8bb42969CE34517d911efFC025bedF07
  Config: configs/node/config.yaml → node.validatorWalletAddress
```

> **Important:** The node **must** know both `validatorWalletAddress` and `operatorAddress` to run in `validator` mode.

---

## 3. Directory Layout

Assuming the extracted tarball directory:

```bash
~/genlayer-node-linux-amd64-v0.4.0/genlayer-node-linux-amd64
```

Key paths:

```text
bin/genlayernode                   # Node binary
configs/node/config.yaml           # Main node configuration
third_party/genvm/                 # GenVM binaries & config
third_party/genvm/bin/setup.py     # GenVM setup/precompile script
third_party/genvm/config/          # genvm-module-*.yaml, etc.
data/                              
  node/
    keystore/                      # Operator keystore (hot wallet)
    logs/                          # Node logs (via lumberjack config)
systemd/genlayernode.service       # (recommended location: /etc/systemd/system/)
```

---

## 4. Key Files to Track & Back Up

You should **version-control or back up** at least:

1. **Node config**
   - `configs/node/config.yaml`  
   Contains:
   - Rollup RPC / WebSocket URLs.
   - Consensus contract addresses.
   - `node.mode`, `node.validatorWalletAddress`, `node.operatorAddress`.
   - Metrics & logging settings.

2. **Operator keystore (hot)**
   - `data/node/keystore/UTC--...`  
   Back this up safely:
   - Use `genlayernode account export` with a strong passphrase.
   - Store the resulting `backup.key` offline.

3. **Owner keystore (cold)**
   - Typically under `~/.genlayer/keystores/owner.json` on the machine where you run the `genlayer` CLI.
   - **Do not** store this on production validator hosts if avoidable.

4. **GenVM configuration**
   - `third_party/genvm/config/genvm-module-llm.yaml`
   - `third_party/genvm/config/genvm-module-web.yaml`
   - `third_party/genvm/config/genvm-manager.yaml`
   - Optional: your `genvm-greyboxing.lua` scripts if you customize prompts.

5. **Systemd unit**
   - `/etc/systemd/system/genlayernode.service`  
   Keep a copy in git or infra repo for reproducibility.

6. **Logs (for debugging)**
   - `data/node/logs/` (depending on `config.yaml` lumberjack settings)
   - Systemd journal via `journalctl -u genlayernode.service`.

---

## 5. Example config.yaml (Validator Mode)

Simplified, focusing on key fields:

```yaml
# rollup configuration
rollup:
  genlayerchainrpcurl: "https://genlayer-testnet.rpc.caldera.xyz/http"
  genlayerchainwebsocketurl: "wss://genlayer-testnet.rpc.caldera.xyz/ws"

# consensus contracts configuration
consensus:
  contractmainaddress: "0x67fd4aC71530FB220E0B7F90668BAF977B88fF07"
  contractdataaddress: "0xB6E1316E57d47d82FDcEa5002028a554754EF243"
  genesis: 4632386   # or value provided by GenLayer docs

# data directory
datadir: "./data/node"

# logging configuration
logging:
  level: "INFO"
  json: false
  file:
    enabled: true
    level: "DEBUG"
    folder: logs
    maxsize: 10
    maxage: 7
    maxbackups: 100
    localtime: false
    compress: true

# node configuration
node:
  mode: "validator"
  validatorWalletAddress: "0x0F67c2fa8bb42969CE34517d911efFC025bedF07"
  operatorAddress: "0x07db557f71ff00817755E13F87165E167C3B95A6"

  admin:
    port: 9155

  rpc:
    port: 9151
    endpoints:
      groups:
        genlayer: true
        genlayer_debug: true
        ethereum: true
        zksync: true

  ops:
    port: 9153
    endpoints:
      metrics: true
      health: true
      balance: true

# genvm configuration
genvm:
  root_dir: ./third_party/genvm
  start_manager: true
  manager_url: http://127.0.0.1:3999
  permits:
    # optional, usually left empty
```

---

## 6. Wallet Operations

### 6.1 Import operator key (validator setup)

```bash
cd ~/genlayer-node-linux-amd64-v0.4.0/genlayer-node-linux-amd64

./bin/genlayernode account import   --password "<NODE_KEYSTORE_PASSWORD>"   --passphrase "<BACKUP_FILE_PASSPHRASE>"   --path "$(pwd)/old-keystores/backup.key"   -c "$(pwd)/configs/node/config.yaml"   --setup
```

Expected output:

```text
Account imported:
        Address: 0x07db557f71ff00817755E13F87165E167C3B95A6
        Account setup as a validator
```

### 6.2 Export operator key (backup)

```bash
./bin/genlayernode account export   --password "<NODE_KEYSTORE_PASSWORD>"   --address "0x07db557f71ff00817755E13F87165E167C3B95A6"   --passphrase "<NEW_BACKUP_PASSPHRASE>"   --path "/secure/location/genlayer-operator-backup.key"   -c configs/node/config.yaml
```

---

## 7. GenVM Setup & LLM Provider

### 7.1 One-time GenVM setup

From the node directory:

```bash
python3 ./third_party/genvm/bin/setup.py
```

This will:

- Create a Python virtual environment under `third_party/genvm/data/venvs/`.
- Precompile WASM runners for GenVM.
- Patch executors as needed (e.g. v0.2.7).

Make sure `python3-venv` is installed on the host.

### 7.2 LLM provider (Heurist example)

Set an environment variable for one provider, e.g.:

```bash
export HEURISTKEY="your_heurist_api_key"
```

Ensure `genvm-module-llm.yaml` has `heurist` enabled and others disabled, or adjust according to your preference.

---

## 8. WebDriver (Selenium) Container

GenVM’s web module needs a WebDriver instance. In the official tarball:

- A reference docker-compose file is included: `docker-compose.yaml`.
- The WebDriver service is usually named `genlayer-node-webdriver` or `webdriver-container`.

Example container check:

```bash
docker ps | grep webdriver
```

Expected (example):

```text
8130b5f685b0  yeagerai/genlayer-genvm-webdriver:0.0.3  "bash /src/start.sh"  ...
```

If you use `docker-compose`, you can manage it as:

```bash
docker compose up -d webdriver-container
docker compose logs -f webdriver-container
```

> Note: In some environments `docker compose` plugin is not available, and the container may have been created manually. Adapt commands to your setup.

---

## 9. Systemd Service

### 9.1 Unit file

Example `/etc/systemd/system/genlayernode.service`:

```ini
[Unit]
Description=GenLayer Validator Node
After=network-online.target docker.service
Wants=network-online.target

[Service]
User=genlayer
Group=genlayer
WorkingDirectory=/home/genlayer/genlayer-node-linux-amd64-v0.4.0/genlayer-node-linux-amd64

ExecStart=/home/genlayer/genlayer-node-linux-amd64-v0.4.0/genlayer-node-linux-amd64/bin/genlayernode   run -c configs/node/config.yaml   --password "<NODE_KEYSTORE_PASSWORD>"

Restart=on-failure
RestartSec=5
LimitNOFILE=65535

# LLM / GenVM related env vars:
Environment="HEURISTKEY=your_heurist_api_key"
# Add other LLM keys if needed:
# Environment="COMPUT3KEY=..."
# Environment="IOINTELLIGENCEKEY=..."
# Environment="LIBERTAI_API_KEY=..."

[Install]
WantedBy=multi-user.target
```

Remember to:

```bash
sudo systemctl daemon-reload
sudo systemctl enable genlayernode.service
sudo systemctl start genlayernode.service
```

---

## 10. Useful Commands

### 10.1 Node health & configuration

```bash
# Doctor: validates config, contracts, zkSync, GenVM & WebDriver
./bin/genlayernode doctor -c configs/node/config.yaml
```

### 10.2 Systemd management

```bash
# Start / stop / restart
sudo systemctl start genlayernode.service
sudo systemctl stop genlayernode.service
sudo systemctl restart genlayernode.service

# Status
sudo systemctl status genlayernode.service

# Live logs (follow)
sudo journalctl -u genlayernode.service -f --no-hostname -o cat
```

### 10.3 RPC & metrics

Once running:

```bash
# Basic RPC info
curl http://127.0.0.1:9151

# Health & metrics
curl http://127.0.0.1:9153/health
curl http://127.0.0.1:9153/metrics
```

---

## 11. Backup & Recovery Checklist

- [ ] Backup **owner keystore** (`~/.genlayer/keystores/owner.json`) in secure cold storage.
- [ ] Backup **operator keystore** using `genlayernode account export` to a `.key` file with strong passphrase.
- [ ] Version-control or archive:
  - `configs/node/config.yaml`
  - `third_party/genvm/config/*`
  - Systemd unit file (`genlayernode.service`).
- [ ] Document:
  - `validatorWalletAddress` (ValidatorWallet contract).
  - `operatorAddress` (hot wallet).
  - `owner` address.
- [ ] Periodically copy node logs for incident analysis.

---

## 12. FAQ (Quick Reference)

**Q: Doctor says `Validator Wallet Configuration` is missing or invalid.**  
A: Check `node.validatorWalletAddress` and `node.operatorAddress` in `config.yaml`. Ensure:
- You have completed `genlayer staking wizard`.
- The validator wallet address is the contract address returned by the wizard.
- The operator key imported into the node matches `operatorAddress`.

**Q: Node fails with `validator address is not set`.**  
A: This usually happens when:
- `node.mode` is `validator` but `validatorWalletAddress` or `operatorAddress` is empty.
- Or the account wasn’t imported with `--setup`.  
Fix:
- Verify `config.yaml`.
- Re-run `genlayernode account import ... --setup` for the operator key.

**Q: `genvm: executable file not found in $PATH`.**  
A: Run:

```bash
python3 ./third_party/genvm/bin/setup.py
```

Ensure that `third_party/genvm/bin/genvm` exists afterwards.

**Q: Doctor shows WebDriver 404 (`/render`).**  
A: The WebDriver version might not support the `/render` endpoint used by `doctor`.  
However, as long as:
- Container is healthy, and
- GenVM can access it at `http://127.0.0.1:4444`  
you can usually ignore the 404 in doctor, or update to the recommended WebDriver image from GenLayer docs.

**Q: How do I safely migrate this node to another server?**  

1. Install dependencies (Docker, Python, etc.) on the new host.
2. Copy:
   - `genlayer-node-linux-amd64-v0.4.0/genlayer-node-linux-amd64` directory (or re-download).
   - `configs/node/config.yaml`.
   - Operator backup key (`backup.key`) and re-import with `--setup`.
3. Re-run GenVM setup with `python3 ./third_party/genvm/bin/setup.py`.
4. Configure and start the WebDriver container.
5. Install and enable `genlayernode.service`.

---

## 13. Disclaimer

This repository is intended for **operational documentation** only.  
Always cross-check contract addresses, RPC URLs, and operational details with the **official GenLayer documentation and announcements**, as parameters may change between testnet phases and releases.

# GenLayer Validator — Initial Installation Guide

> **Target version:** v0.4.1  
> **Network:** GenLayer Testnet (Asimov)  


---

## 1. System Requirements

**Minimum recommended hardware**
- **CPU:** 8 cores / 16 threads
- **RAM:** 16 GB
- **Disk:** SSD / NVMe — **≥128 GB**
- **Network:** 100 Mbps (1 Gbps recommended)
- **OS:** Linux x86_64 (Ubuntu / Debian recommended)

> **Note:** No GPU is required. GenLayer Testnet uses **Heurist‑hosted LLMs** by default.

---

## 2. Dependencies

Install the following packages:

```bash
sudo apt update && sudo apt install -y   curl wget tar git jq docker.io docker-compose
```

(Optional but recommended)
- `screen` or `tmux` for persistent SSH sessions

Enable Docker:
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

## 3. Download GenLayer Node

### 3.1 Discover latest available version

```bash
curl -s "https://storage.googleapis.com/storage/v1/b/gh-af/o?prefix=genlayer-node/bin/amd64" | grep -o '"name": *"[^"]*"' | sed -n 's/.*\/\(v[^/]*\)\/.*/\1/p' | sort -ru
```

### 3.2 Download and extract (example: v0.4.1)

```bash
export VERSION=v0.4.1
cd ~

wget https://storage.googleapis.com/gh-af/genlayer-node/bin/amd64/${VERSION}/genlayer-node-linux-amd64-${VERSION}.tar.gz
tar -xzf genlayer-node-linux-amd64-${VERSION}.tar.gz
cd genlayer-node-linux-amd64
```

Verify version:
```bash
./bin/genlayernode version
```

---

## 4. Node Configuration

File:
```
configs/node/config.yaml
```

### 4.1 GenLayer chain endpoints (Testnet)

```yaml
genlayerchainrpcurl: "https://genlayer-testnet.rpc.caldera.xyz/http"
genlayerchainwebsocketurl: "wss://genlayer-testnet.rpc.caldera.xyz/ws"
```

---

### 4.2 Consensus contracts (Asimov)

> **Important:** Contract addresses change between phases. Always verify with official docs.

```yaml
consensus:
  contractmanageraddress: "0x0761ff3847294eb3234f37Bf63fd7F1CA1E840bB"
  contractmainaddress: "0x67fd4aC71530FB220E0B7F90668BAF977B88fF07"
  contractdataaddress: "0xB6E1316E57d47d82FDcEa5002028a554754EF243"
  genesis: 817855
```

---

### 4.3 Logging (recommended)

```yaml
log:
  maxsize: 500
  maxbackups: 10
  compress: true
```

Logs path:
```
data/node/logs/
```

Archive example:
```bash
tar -czvf genlayer-logs-$(date +%F).tar.gz data/node/logs
```

---

## 5. GenVM & LLM Configuration

GenVM enables **Intelligent Contracts** powered by LLMs.

### 5.1 Files

```
third_party/genvm/config/
├── genvm-module-llm.yaml
├── genvm-module-web.yaml
├── genvm.yaml
```

Only **genvm-module-llm.yaml** must be edited.

---

### 5.2 Heurist (required for Testnet)

- Enable **only** Heurist
- Disable all other providers
- Export `HEURISTKEY`

```bash
export HEURISTKEY="YOUR_HEURIST_API_KEY"
echo 'export HEURISTKEY="YOUR_HEURIST_API_KEY"' >> ~/.bashrc
source ~/.bashrc
```

Heurist panel:
https://www.heurist.ai/credits

---

## 6. Precompile GenVM (recommended)

```bash
./third_party/genvm/bin/genvm precompile
```

Repeat after every node upgrade.

---

## 7. Create Validator Account

```bash
./bin/genlayernode account new   -c $(pwd)/configs/node/config.yaml   --setup   --password "STRONG_PASSWORD"
```

Save securely:
- Validator address (`0x...`)
- Password

---

## 8. Backup Validator Key

```bash
./bin/genlayernode account export   --password "STRONG_PASSWORD"   --address "0xYOUR_VALIDATOR"   --passphrase "BACKUP_PASSPHRASE"   --path "./backup.key"   -c $(pwd)/configs/node/config.yaml
```

⚠️ **Never share your private key**.

---

## 9. Restore Validator Key (if needed)

```bash
./bin/genlayernode account import   --password "STRONG_PASSWORD"   --passphrase "BACKUP_PASSPHRASE"   --path "./backup.key"   --setup   -c $(pwd)/configs/node/config.yaml
```

---

## 10. Validate Configuration

```bash
./bin/genlayernode doctor -c $(pwd)/configs/node/config.yaml
```

All checks should pass.

---

## 11. Start WebDriver (required by GenVM)

```bash
docker compose up -d
docker ps
```

---

## 12. Run Node (manual)

```bash
./bin/genlayernode run   -c $(pwd)/configs/node/config.yaml   --password "STRONG_PASSWORD"
```

---

## 13. Systemd Service (recommended)

File:
```
/etc/systemd/system/genlayernode.service
```

```ini
[Unit]
Description=GenLayer Validator Node (Cumulo)
After=network-online.target docker.service
Wants=network-online.target docker.service

[Service]
User=dedicaovh
WorkingDirectory=/home/dedicaovh/genlayer-node-linux-amd64

Environment="GENLAYERNODE_PASSWORD=STRONG_PASSWORD"
Environment="HEURISTKEY=YOUR_HEURIST_API_KEY"

ExecStart=/home/dedicaovh/genlayer-node-linux-amd64/bin/genlayernode run   -c /home/dedicaovh/genlayer-node-linux-amd64/configs/node/config.yaml   --password ${GENLAYERNODE_PASSWORD}

Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Enable & start:
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable genlayernode
sudo systemctl start genlayernode
```

Logs:
```bash
journalctl -u genlayernode -f
```

---

## 14. Sync Status

```bash
journalctl -u genlayernode | grep "Node is synced"
```

Expected:
```
Node is synced!!! blockNumber=XXXXX
```

---

## 15. Validator Status

```bash
genlayer staking validator-info   --validator 0xYOUR_VALIDATOR
```

---

## 16. Notes on Quarantine

- Temporary **Quarantined** state is normal after upgrades or restarts
- Clears automatically once epochs finalize correctly
- Ensure:
  - Node running continuously
  - LLM module active
  - WebDriver running

---

## Appendix — What is GenVM?

GenVM is GenLayer’s execution engine for **Intelligent Contracts**.  
It integrates LLMs to enable:

- Natural‑language logic
- Non‑deterministic computation with consensus reconciliation
- Advanced off‑chain reasoning

Uniform LLM configuration across validators is **critical** to prevent forks.

---

**Maintained by Cumulo**  
Infrastructure, monitoring, and validator tooling for GenLayer and beyond.

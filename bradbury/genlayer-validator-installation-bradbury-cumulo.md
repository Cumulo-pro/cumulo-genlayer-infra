# GenLayer Validator — Bradbury Installation Guide

![Version](https://img.shields.io/badge/version-v0.5.5-blue)
![Network](https://img.shields.io/badge/network-Testnet%20Bradbury%20Phase%201-orange)
![OS](https://img.shields.io/badge/OS-Ubuntu%2020.04%20LTS-green)
![Maintained by Cumulo](https://img.shields.io/badge/maintained%20by-Cumulo-purple)

> Installation guide for running a GenLayer validator node on the **Bradbury Phase 1** testnet (v0.5.5).

---

## Table of Contents

- [1. System Requirements](#1-system-requirements)
- [2. Dependencies](#2-dependencies)
- [3. Node.js and GenLayer CLI](#3-nodejs-and-genlayer-cli)
- [4. Python 3.10 for GenVM Setup](#4-python-310-for-genvm-setup)
- [5. Download GenLayer Node](#5-download-genlayer-node)
- [6. GenVM Setup](#6-genvm-setup)
- [7. Node Configuration](#7-node-configuration)
- [8. Owner, Operator and Validator](#8-owner-operator-and-validator)
- [9. Import Operator Key into the Node](#9-import-operator-key-into-the-node)
- [10. LLM Provider Configuration](#10-llm-provider-configuration)
- [11. Start WebDriver](#11-start-webdriver)
- [12. Validate Configuration](#12-validate-configuration)
- [13. Run Node Manually](#13-run-node-manually)
- [14. Health Check](#14-health-check)
- [15. Systemd Service](#15-systemd-service)
- [16. Validator Status](#16-validator-status)
- [17. Important Paths](#17-important-paths)
- [18. Operational Commands](#18-operational-commands)
- [19. Notes](#19-notes)

---

## 1. System Requirements

### Minimum recommended hardware

| Component | Requirement |
|-----------|-------------|
| CPU | 8 cores / 16 threads |
| RAM | 16 GB |
| Disk | SSD / NVMe — **≥ 128 GB** |
| Network | 100 Mbps |
| OS | Linux x86_64 (Ubuntu / Debian recommended) |

> **Note:** No GPU is required unless you plan to run LLMs locally. This guide uses a hosted LLM provider.

---

## 2. Dependencies

Install the required packages:

```bash
sudo apt update && sudo apt install -y \
  curl wget tar git jq docker.io docker-compose \
  python3 python3.8-venv software-properties-common
```

Enable Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

## 3. Node.js and GenLayer CLI

GenLayer CLI requires **Node.js v18+**.

### Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify versions:

```bash
node -v
npm -v
```

### Install GenLayer CLI

```bash
sudo npm install -g genlayer
```

Verify:

```bash
genlayer --version
```

---

## 4. Python 3.10 for GenVM Setup

On Ubuntu 20.04, Python 3.8 is not sufficient for the current GenVM post-install scripts. Python 3.10 must be compiled manually.

### Build Python 3.10.16

```bash
sudo apt update
sudo apt install -y build-essential curl wget git \
  libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev \
  libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev \
  libffi-dev liblzma-dev uuid-dev

cd /tmp
wget https://www.python.org/ftp/python/3.10.16/Python-3.10.16.tgz
tar -xzf Python-3.10.16.tgz
cd Python-3.10.16
./configure --prefix=/opt/python3.10 --enable-optimizations
make -j$(nproc)
sudo make altinstall
```

Verify:

```bash
/opt/python3.10/bin/python3.10 --version
```

---

## 5. Download GenLayer Node

### Download v0.5.5

```bash
export VERSION=v0.5.5
cd ~
mkdir -p genlayer-bradbury
cd genlayer-bradbury

wget https://storage.googleapis.com/gh-af/genlayer-node/bin/amd64/${VERSION}/genlayer-node-linux-amd64-${VERSION}.tar.gz
mkdir -p ${VERSION}
tar -xzvf genlayer-node-linux-amd64-${VERSION}.tar.gz -C ./${VERSION}
cd ./${VERSION}/genlayer-node-linux-amd64
```

Verify version:

```bash
./bin/genlayernode version
```

---

## 6. GenVM Setup

Run setup using **Python 3.10**:

```bash
cd ~/genlayer-bradbury/v0.5.5/genlayer-node-linux-amd64
/opt/python3.10/bin/python3.10 ./third_party/genvm/bin/setup.py
```

### Known fixes required on Ubuntu 20.04

During this installation, three manual fixes were needed:

1. **Install `python3.8-venv`** if not already present.

2. **Replace `lief` version** in `third_party/genvm/lib/python/post-install/requirements.txt`:

   ```diff
   - lief==0.17.0
   + lief==0.17.5
   ```

3. **Update `create_venv.py`** at `third_party/genvm/lib/python/post-install/create_venv.py` to upgrade `pip`, `setuptools`, and `wheel` before installing dependencies.

> These fixes apply to **Ubuntu 20.04** specifically and may not be needed on newer OS versions.

---

## 7. Node Configuration

Configuration file location:

```
configs/node/config.yaml
```

### 7.1 Rollup Endpoints

```yaml
rollup:
  genlayerchainrpcurl: "https://zksync-os-testnet-genlayer.zksync.dev"
  genlayerchainwebsocketurl: "wss://zksync-os-testnet-alpha.zksync.dev/ws"
```

### 7.2 Consensus Configuration (Bradbury)

```yaml
consensus:
  consensusaddress: "0x8aCE036C8C3C5D603dB546b031302FCf149648E8"
  genesis: 501711
```

### 7.3 Validator Configuration

```yaml
node:
  mode: "validator"
  validatorWalletAddress: "0xf5E017C93f37E3f03595C03D5A6A98396769C175"
  operatorAddress: "0x9f1d7bf03c5bc11644bf4d5cce92cb9a6352f4ef"
```

### 7.4 Ports

```yaml
node:
  admin:
    port: 9155
  rpc:
    port: 9151
  ops:
    port: 9153
```

---

## 8. Owner, Operator and Validator

Bradbury uses a three-role model:

| Role | Description |
|------|-------------|
| **Owner** | Same owner wallet used previously in Asimov |
| **Operator** | **New** operator created specifically for Bradbury |
| **Validator Wallet** | **New** validator wallet created specifically for Bradbury |

### Owner address

```
0xea75b2f766eb89c40d636f11663143f9a618623f
```

### Create validator in Bradbury

```bash
genlayer staking wizard --network testnet-bradbury
```

### Result of this installation

| Field | Value |
|-------|-------|
| Validator Wallet | `0xf5E017C93f37E3f03595C03D5A6A98396769C175` |
| Operator | `0x9f1d7bf03c5bc11644bf4d5cce92cb9a6352f4ef` |
| Stake | `101000gen` |

---

## 9. Import Operator Key into the Node

The wizard exports the operator keystore to:

```
/home/noditoovh/operator-keystore.json
```

Import it into the node:

```bash
./bin/genlayernode account import \
  --password "YOUR_NODE_PASSWORD" \
  --passphrase "YOUR_KEYSTORE_PASSPHRASE" \
  --path "/home/noditoovh/operator-keystore.json" \
  -c "$(pwd)/configs/node/config.yaml" \
  --setup
```

---

## 10. LLM Provider Configuration

This installation uses **Heurist** as the LLM provider.

Export the API key:

```bash
export HEURISTKEY="YOUR_HEURIST_API_KEY"
```

To persist it across reboots, create an environment file:

```bash
mkdir -p ~/genlayer-bradbury/env
nano ~/genlayer-bradbury/env/bradbury.env
```

File contents:

```bash
HEURISTKEY=YOUR_HEURIST_API_KEY
```

Restrict permissions:

```bash
chmod 600 ~/genlayer-bradbury/env/bradbury.env
```

---

## 11. Start WebDriver

The WebDriver container is required by GenVM:

```bash
docker compose up -d
docker ps
```

Expected container:

```
genlayer-node-webdriver
```

---

## 12. Validate Configuration

Run the full diagnostics check:

```bash
./bin/genlayernode doctor -c "$(pwd)/configs/node/config.yaml" --network bradbury
```

Expected output:

```
All configuration checks passed!
```

---

## 13. Run Node Manually

```bash
export HEURISTKEY="YOUR_HEURIST_API_KEY"
./bin/genlayernode run -c "$(pwd)/configs/node/config.yaml" --password "YOUR_NODE_PASSWORD"
```

---

## 14. Health Check

```bash
curl -s http://127.0.0.1:9153/health
```

When fully synced and validating, the expected response is:

```json
{
  "checks": {
    "validating": {
      "status": "up"
    },
    "zksync-connectivity": {
      "status": "up"
    }
  },
  "genvm_version": "v0.2.11",
  "network": "bradbury-phase1",
  "node_version": "v0.5.5",
  "protocol_version": "v0.5:5a1e04df",
  "status": "up"
}
```

Pretty-printed output:

```bash
curl -s http://127.0.0.1:9153/health | jq
```

> **Note:** The node reports `status: down` while syncing — this is expected. Once synced, `validating.status` becomes `up`.

---

## 15. Systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/genlayer-bradbury.service
```

Service definition:

```ini
[Unit]
Description=GenLayer Bradbury Validator
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
User=noditoovh
Group=noditoovh
WorkingDirectory=/home/noditoovh/genlayer-bradbury/v0.5.5/genlayer-node-linux-amd64
EnvironmentFile=/home/noditoovh/genlayer-bradbury/env/bradbury.env
ExecStart=/home/noditoovh/genlayer-bradbury/v0.5.5/genlayer-node-linux-amd64/bin/genlayernode run \
  -c /home/noditoovh/genlayer-bradbury/v0.5.5/genlayer-node-linux-amd64/configs/node/config.yaml \
  --password YOUR_NODE_PASSWORD
Restart=always
RestartSec=5
LimitNOFILE=65535
KillSignal=SIGINT
TimeoutStopSec=60

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable genlayer-bradbury
sudo systemctl start genlayer-bradbury
```

View logs:

```bash
journalctl -u genlayer-bradbury -f
```

Check status:

```bash
sudo systemctl status genlayer-bradbury --no-pager -l
```

---

## 16. Validator Status

```bash
genlayer staking validator-info \
  --validator 0xf5E017C93f37E3f03595C03D5A6A98396769C175 \
  --network testnet-bradbury
```

---

## 17. Important Paths

| Resource | Path |
|----------|------|
| Node root | `/home/noditoovh/genlayer-bradbury/v0.5.5/genlayer-node-linux-amd64` |
| Main binary | `.../bin/genlayernode` |
| Main config | `.../configs/node/config.yaml` |
| Data directory | `.../data/node` |
| Logs | `.../data/node/logs` |
| Owner keystore | `/home/noditoovh/.genlayer/keystores/owner.json` |
| Operator keystore | `/home/noditoovh/operator-keystore.json` |
| Env file | `/home/noditoovh/genlayer-bradbury/env/bradbury.env` |
| Systemd service | `/etc/systemd/system/genlayer-bradbury.service` |

---

## 18. Operational Commands

### Health

```bash
curl -s http://127.0.0.1:9153/health
```

### Metrics

```bash
curl -s http://127.0.0.1:9153/metrics | head
```

### Check ports

```bash
ss -lntp | grep -E '9151|9153|9155|4444'
```

### Docker

```bash
docker ps
docker logs -f genlayer-node-webdriver
```

### Systemd

```bash
sudo systemctl status genlayer-bradbury --no-pager -l
journalctl -u genlayer-bradbury -f
journalctl -u genlayer-bradbury -n 100 --no-pager
```

---

## 19. Notes

- Bradbury and Asimov share the **same owner wallet** in this setup.
- Bradbury requires a **new operator** and a **new validator wallet**.
- The node reports `status: down` while syncing — this is normal behaviour.
- Once synced, `validating.status` transitions to `up`.
- This installation was completed successfully on **Ubuntu 20.04.6 LTS**.

---

*Maintained by **[Cumulo](https://cumulo.pro)** — Infrastructure, monitoring, and validator tooling for GenLayer and beyond.*

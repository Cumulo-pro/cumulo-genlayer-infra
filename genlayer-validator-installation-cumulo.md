# GenLayer Validator Node: Installation with Docker Compose (Bradbury Phase 2)

How Cumulo runs its GenLayer validator on **Bradbury Phase 2** (testnet, chain id `4221`) with **Docker Compose**, node **v0.6.0-rc.5**.

_Last updated: 2026-11-21. Node `v0.6.0-rc.5`, GenVM `v0.6.0-rc10`, protocol `v0.6-rc-1`._

> This guide follows the [official setup guide](https://docs.genlayer.com/validators/setup-guide) and shows the values of our own deployment. Always confirm the exact version approved for Bradbury Phase 2 in the [changelog](https://docs.genlayer.com/validators/changelog) before installing.

## 1. What runs on the server

Everything runs in containers, managed by one `docker-compose.yaml` that ships inside the release package. The project name is fixed (`genlayer-node`), so a new version started from a new directory replaces the containers of the old one.

| Service | Container | Image | Profile | Purpose |
|---|---|---|---|---|
| `genlayer-node` | `genlayer-node` | `yeagerai/genlayer-node:v0.6.0-rc.5` | `node` | The validator. Starts GenVM itself |
| `webdriver-container` | `genlayer-node-webdriver` | `yeagerai/genlayer-genvm-webdriver:0.0.12` | always | Browser used by the GenVM web module (port 4444) |
| `docker-socket-proxy` | `genlayer-node-docker-proxy` | `tecnativa/docker-socket-proxy:v0.5.0` | `node`, `monitoring` | Read-only Docker API, loopback only (`127.0.0.1:2375`), for WebDriver resource metrics |
| `alloy` | `genlayer-node-alloy` | `grafana/alloy:v1.20.1` | `monitoring` | Pushes metrics and logs to Grafana Cloud |

The node container runs as uid `10001`, drops all capabilities and uses `no-new-privileges`. It does not mount the host Docker socket.

Ports used by the node:

| Port | Use | Exposure |
|---|---|---|
| `9151` | JSON-RPC of the node | Optional, publish only if you serve RPC |
| `9153` | Ops server: `/health`, `/metrics`, `/balance`, `/snapshot` | Restrict by firewall to your monitoring hosts |
| `9155` | Admin | Not published by default |
| `4444` | WebDriver | Local use by the node |

## 2. Requirements

| Resource | Requirement |
|---|---|
| RAM | 16 GB |
| CPU | 8 cores, AMD64 only (ARM64 is not supported) |
| Disk | 128 GB SSD or NVMe |
| Network | 100 Mbps (1 Gbps recommended) |
| OS | 64-bit Linux |
| Software | Docker with the Compose plugin |

Release candidate images (`v0.6.0-rc.N`) are published for `linux/amd64` only.

## 3. Before the server: wallet and operator key

A validator uses three addresses. The staking wizard prints all three at the end:

| Address | Role | Where it is used |
|---|---|---|
| Owner | Holds the stake and the only key that can withdraw it. Keep it off the server | CLI, signs staking transactions |
| Operator | Hot key on the server, signs consensus duties | `node.operatorAddress` and the keystore |
| Validator Wallet | Contract created when you join, the on-chain identity | `node.validatorWalletAddress` |

Create them with the CLI (see the [Staking guide](https://cumulo.pro/services/genlayer/staking) for details):

```bash
npm install -g genlayer@rc
genlayer staking wizard --network testnet-bradbury
```

Keep the **operator keystore export** and its passphrase: you install it on the server in section 7.

## 4. Prepare the directory

Create a directory for this version and extract the release package into it, for example `~/genlayer-node/v0.6.0-rc.5`.

```bash
export version=v0.6.0-rc.5   # confirm it is the version approved for Bradbury Phase 2
base=https://storage.googleapis.com/gh-af/genlayer-node/bin/amd64/${version}

# check the three files exist before downloading
curl -fI "${base}/genlayer-node-linux-amd64-${version}.tar.gz"
curl -fI "${base}/genlayer-node_${version}_SHA256SUMS"
curl -fI "${base}/genlayer-node_${version}_SHA256SUMS.sigstore.json"

wget "${base}/genlayer-node-linux-amd64-${version}.tar.gz" \
     "${base}/genlayer-node_${version}_SHA256SUMS" \
     "${base}/genlayer-node_${version}_SHA256SUMS.sigstore.json"
```

Verify the signed checksum before extracting (needs [cosign](https://docs.sigstore.dev/cosign/system_config/installation/)):

```bash
cosign verify-blob \
  --bundle "genlayer-node_${version}_SHA256SUMS.sigstore.json" \
  --certificate-identity-regexp '^https://github\.com/genlayerlabs/genlayer-node/\.github/workflows/release-package\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  "genlayer-node_${version}_SHA256SUMS"

sha256sum -c "genlayer-node_${version}_SHA256SUMS" --ignore-missing
```

Extract. The package contains `docker-compose.yaml`, `.env.example`, `genvm-module-web-docker.yaml` and `alloy-healthcheck.sh`:

```bash
mkdir -p ${version}
tar -xzf genlayer-node-linux-amd64-${version}.tar.gz -C ./${version} --strip-components=1
cd ./${version}
```

With Docker you do not run the GenVM setup script: the image's entrypoint prepares GenVM for the host CPU on the first start.

## 5. Configuration

### 5.1 `configs/node/config.yaml`

Take the `config.yaml` from the package and set these sections. Values for Bradbury Phase 2:

```yaml
rollup:
  genlayerchainrpcurl: "https://rpc.testnet-chain.genlayer.com/"
  genlayerchainwebsocketurl: "wss://rpc.testnet-chain.genlayer.com/"
  provider: "conduit"

consensus:
  consensusaddress: "0x3F71AE080BD6270A91dBcc7960d88797828538E0"   # AddressManager
  chainid: "4221"
  manifestpath: "configs/node/protocol-manifest.json"
  genesis: 47930

datadir: "./data/node"

node:
  mode: "validator"
  validatorWalletAddress: "0xYourValidatorWallet"
  operatorAddress: "0xYourOperatorAddress"
  admin:
    port: 9155
  rpc:
    port: 9151
    endpoints:
      genlayer:       { enabled: true }
      genlayer_debug: { enabled: false }
      ethereum:       { enabled: true }
      zksync:         { enabled: false }
  ops:
    port: 9153
```

Keep the rest of the file as shipped. Important points:

- `genvm.root_dir` must stay `./opt/genvm/current`.
- `logging.file` keeps `level: DEBUG`, `maxsize: 10`, `maxage: 7`, `maxbackups: 100`, `compress: true`.
- `genvm.permits: 32` and `genvm.sync_permits: 256` are the shipped values we run with.
- `metrics.interval: 15s`, with the `node`, `genvm` and `webdriver` collectors enabled.
- The config is mounted **read-only** into the container, so edit it on the host and recreate the container.

Our validator for reference (public on-chain data): wallet `0xe2c63756722362D78eA9AFd8232A89cb7d85fbbF`, operator `0xccfce5bbb87e9a71f903a7e86acfafb6677646d6`.

### 5.2 Protocol manifest

Since v0.6 the node refuses to start without the trusted manifest of the network, and it checks it against `consensusaddress` and `chainid`:

```bash
mkdir -p configs/node
curl -fsSL https://docs.genlayer.com/validators/bradbury.protocol-manifest.json -o configs/node/protocol-manifest.json
```

### 5.3 `.env`

Copy `.env.example` to `.env` (mode `600`) and set at least the variables below. **Never commit this file** and never put keys in `config.yaml` or in URLs.

| Variable | What it is |
|---|---|
| `NODE_VERSION` | Pin the exact version, `v0.6.0-rc.5`. Do not use `latest` |
| `NODE_PASSWORD` | Password of the operator keystore. Reaches the node as `GENLAYERNODE_PASSWORD`. No default |
| `GENLAYERNODE_ROLLUP_GENLAYERCHAINRPCKEY` | Conduit API key. The Bradbury WebSocket refuses connections without it. Create it at [app.conduit.xyz/rpc-keys](https://app.conduit.xyz/rpc-keys) |
| `GENLAYERNODE_NODE_OPERATOR_TOKEN` | Bearer token for `POST /snapshot` (`openssl rand -hex 32`). Without it the route answers 401 |
| LLM keys | `HEURISTKEY`, `OPENAIKEY`, `ANTHROPICKEY`, `GEMINIKEY`, `XAIKEY`, `OPENROUTERKEY`, `COMPUT3KEY`, `IOINTELLIGENCEKEY`, `LIBERTAI_API_KEY`, `MORPHEUS_API_KEY`, `ATOMAKEY`: set the providers you use |
| `NETWORK_NAME` | `bradbury-phase2` (label for monitoring) |

Monitoring variables (`CENTRAL_*`, `NODE_ID`, `VALIDATOR_NAME`) are explained in the [Metrics guide](https://cumulo.pro/services/genlayer/metrics).

Use **more than one LLM provider**. With a single provider, an outage makes the node vote Timeout on every transaction that needs an LLM, and Timeout votes are penalized. See [GenVM configuration](https://docs.genlayer.com/validators/genvm-configuration) for partner credits.

## 6. Permissions on the data directory

The container runs as uid `10001` and `./data` keeps the host owner, so give it access once:

```bash
mkdir -p ./data && sudo chown -R 10001:10001 ./data
```

## 7. Install the operator key

Copy the encrypted operator keystore exported by the CLI wizard and install it with the right owner. `NODE_PASSWORD` must be the **export passphrase**:

```bash
sudo install -d -m 700 -o 10001 -g 10001 ./data/node/keystore
test ! -e ./data/node/keystore/operator.json || exit 1
sudo install -m 600 -o 10001 -g 10001 /path/to/operator-keystore.json ./data/node/keystore/operator.json
```

Never run two nodes with the same operator key at the same time. More in [Key Management](https://cumulo.pro/services/genlayer/key).

## 8. Check, start and verify

Run `doctor` with the same config, manifest and environment as the node. It checks chain connectivity, the consensus deployment against the manifest, GenVM, every enabled LLM provider (live call) and the WebDriver:

```bash
docker compose --profile node run --rm --entrypoint /app/bin/genlayernode genlayer-node doctor
```

Start the node (add `--profile monitoring` to also start Alloy):

```bash
docker compose --profile node up -d
docker compose --profile node ps
docker compose logs -f genlayer-node
```

In the logs you should see the node sync and then run in validator mode:

```text
Node is synced!!! mode=validator node=0x... operator=0x...
```

If you see `mode=full`, the validator configuration is incomplete.

Then query the ops server:

```bash
curl -s http://127.0.0.1:9153/health | jq '{status, node_version, genvm_version, network}'
curl -s http://127.0.0.1:9153/balance
```

`status` is `healthy` when all is well. `warning` can be legitimate, for example while the validator is banned or waiting to prime. See the [FAQ](https://cumulo.pro/services/genlayer/faq) and [Metrics](https://cumulo.pro/services/genlayer/metrics) pages.

The operator must hold GEN to pay gas. Watch `/balance` and the `consensus.operator_funds` condition.

## 9. Upgrades

Read the [changelog](https://docs.genlayer.com/validators/changelog) of the target version first.

- **Non-breaking** (patch or minor, no consensus contract change): download and extract into a **new directory**, set `config.yaml` and `.env` from the new examples (do not blindly copy the old ones), copy `data/`, run `doctor`, then bring the new stack up from the new directory. The fixed project name makes Compose replace the old containers.
- **Breaking** (new consensus deployment, as v0.5 to v0.6): back up `data/node/keystore/`, configure from scratch, keep only `keystore/` inside `data/node/`, stake again with the wizard, and let the node sync from genesis.

Do not use `genlayernode run --upgrade` in v0.6: the call that resumes the node is not available yet, so the node waits forever.

## 10. Useful commands

```bash
docker compose --profile node ps                    # status
docker compose logs --since 30m genlayer-node       # recent logs
docker restart genlayer-node                        # restart the node only
docker compose --profile node down                  # stop everything (data stays in ./data)
docker inspect --format '{{.State.Health.Status}}' genlayer-node-webdriver
```

## References

- [Setup guide](https://docs.genlayer.com/validators/setup-guide)
- [System requirements](https://docs.genlayer.com/validators/system-requirements)
- [Upgrade guide](https://docs.genlayer.com/validators/upgrade)
- [Changelog](https://docs.genlayer.com/validators/changelog)

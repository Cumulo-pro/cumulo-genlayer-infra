# GenLayer Validator Node FAQ (Bradbury Phase 2, Docker Compose)

Common questions when running a GenLayer validator on **Bradbury Phase 2** with Docker Compose, node **v0.6.0-rc.5**.

_Last updated: 2026-11-21._

---

## 1. Wallets and roles

### What is the difference between Owner, Operator and Validator Wallet?

- **Owner**: holds the stake, signs deposits, exits, claims and identity changes. Cold key, never on the server.
- **Operator**: hot key on the server that signs consensus duties and pays gas.
- **Validator Wallet**: the contract created when you join. Goes in `node.validatorWalletAddress`.

See [Key Management](key).

### Where are the keystores?

- Owner: in the CLI keystore of the machine where you ran the wizard (`genlayer account list`).
- Operator on the server: `./data/node/keystore/operator.json` (container path `/app/data/node/keystore/`), owned by uid `10001`.

---

## 2. Configuration

### Which fields must be right for validator mode?

```yaml
node:
  mode: "validator"
  validatorWalletAddress: "0x..."
  operatorAddress: "0x..."
```

plus the Bradbury Phase 2 `rollup` and `consensus` sections and the protocol manifest. See [Install](install). Also `NODE_PASSWORD` in `.env`.

### Can I override `config.yaml` with environment variables?

Yes. Prefix `GENLAYERNODE_`, dots become underscores, uppercase. For example `node.operatorAddress` is `GENLAYERNODE_NODE_OPERATORADDRESS`. Only keys that exist (uncommented) in the file are picked up. Put them in the `.env` next to `docker-compose.yaml`.

### I edited `config.yaml` and nothing changed.

The file is mounted read-only into the container and read at start. Recreate the container:

```bash
docker compose --profile node up -d --force-recreate genlayer-node
```

### Can I put the RPC API key in the URL?

No. Set `GENLAYERNODE_ROLLUP_GENLAYERCHAINRPCKEY` in `.env`. The node sends it as the `X-AccessKey` header. A key in a URL ends up in logs and error messages.

---

## 3. The node does not start

### `password is required for validator mode`

`NODE_PASSWORD` is empty in `.env`. It has no default on purpose.

### Permission denied writing to `data`

The container runs as uid `10001`:

```bash
sudo chown -R 10001:10001 ./data
```

### The manifest is refused at startup

Since v0.6 the node checks `configs/node/protocol-manifest.json` against `consensus.consensusaddress` and `consensus.chainid`. Download the Bradbury manifest again and check that `chainid` is `"4221"` and `consensusaddress` is `0x3F71AE080BD6270A91dBcc7960d88797828538E0`:

```bash
curl -fsSL https://docs.genlayer.com/validators/bradbury.protocol-manifest.json -o configs/node/protocol-manifest.json
```

### `chain.websocket` is `degraded` or the node never connects

The Bradbury WebSocket refuses connections without an API key. Create one at [app.conduit.xyz/rpc-keys](https://app.conduit.xyz/rpc-keys) and set `GENLAYERNODE_ROLLUP_GENLAYERCHAINRPCKEY`.

### The node starts as `mode=full`

`validatorWalletAddress` or `operatorAddress` is missing in `config.yaml`. Run `doctor`.

### Container name already in use after an upgrade

Always start the new version with the same Compose project. The release file fixes `name: genlayer-node` for this. If you removed that line, restore it.

### My custom GenVM config is ignored after v0.6

GenVM moved from `/app/third_party/genvm/config/` to `/app/opt/genvm/current/config/`. Update the mount path in your `docker-compose.override.yml`. A mount left at the old path is ignored without an error.

---

## 4. Doctor

### What does `doctor` check?

Chain connectivity, the consensus deployment against the manifest, the GenVM configuration (binaries, module config, LLM keys with a live call to each enabled provider) and the WebDriver.

```bash
docker compose --profile node run --rm --entrypoint /app/bin/genlayernode genlayer-node doctor
```

---

## 5. GenVM and LLM

### How many LLM providers do I need?

Configure **more than one**. With a single provider, an outage or an empty balance makes the node vote Timeout on every transaction that needs an LLM, and Timeout votes are penalized. `doctor` warns when only one backend is usable.

### I added or changed an API key and nothing happened.

Keys are read from the environment at start. Recreate the container:

```bash
docker compose --profile node up -d --force-recreate genlayer-node
```

### Where do I find free LLM credits?

See [GenVM Configuration](https://docs.genlayer.com/validators/genvm-configuration) (Heurist, Comput3, io.net, Chutes, Morpheus).

---

## 6. Operations

### How do I start, stop and see logs?

```bash
docker compose --profile node up -d          # start
docker compose --profile node down           # stop (data stays in ./data)
docker restart genlayer-node                 # restart the node only
docker compose logs -f genlayer-node         # live logs
docker compose --profile node ps             # status
```

The container's own log rotates at 100 MB with 3 files. The node also writes its own log files to `./data/node/logs` (DEBUG level, rotated).

### Which ports does the node use?

| Port | Use |
|---|---|
| `9151` | Node JSON-RPC |
| `9153` | Ops: `/health`, `/metrics`, `/balance` |
| `9155` | Admin |
| `4444` | WebDriver |

### How do I know it is running as a validator?

The logs should show:

```text
Node is synced!!! mode=validator node=0x... operator=0x...
```

and `curl -s http://127.0.0.1:9153/health` returns your `validator_address` and `validator_moniker`.

---

## 7. My validator shows `warning` in /health

`warning` is not always an incident. Read the `reason`, `impact` and `action` of the condition that is not healthy:

| Condition and reason | Meaning | What to do |
|---|---|---|
| `consensus.ban` / `validator_banned` | The validator is banned until `until_epoch` (or permanently). The node syncs but earns nothing | Wait for that epoch. If permanent, join again with a new validator |
| `consensus.prime` / `prime_not_landing` | The prime window is open and the prime did not land. The validator forfeits that epoch's inflation | Check the operator balance and the logs. Anyone can prime it: `genlayer staking validator-prime 0xWallet` |
| `consensus.operator_funds` / `operator_balance_critical` | Operator is low on gas | Send GEN to the operator address |
| `consensus.stake` / `self_stake_below_minimum` | Self-stake is under the minimum, validator is not selected | Add stake with `validator-deposit` (owner) |
| `genvm.llm` | A provider fails | Check API keys and credits, configure more than one provider |
| `webdriver.render` / `webdriver_check_failed` | WebDriver cannot render the probe page | `docker logs genlayer-node-webdriver`, check its health status |
| `node.snapshot_age` / `unknown` | No snapshots are taken | Normal, ignore |

Only `unavailable` returns HTTP 503. See [Metrics](metrics) for the full catalog.

### Why does the explorer show my validator as quarantined or banned while the node is healthy?

Quarantine and ban are decisions of the consensus contracts, not of the node. The node keeps syncing and serving. Check `genlayer staking validator-info 0xWallet` and `validator-history`.

---

## 8. Staking

### How much do I need to stake?

Read the deployed minimum with `genlayer staking epoch-info --network testnet-bradbury`. 42,000 GEN is a protocol default, not a guaranteed current value. Delegation does not replace the required self-stake.

### How long does unbonding take?

7 epochs after `validator-exit`, then `validator-claim`.

### `OwnableUnauthorizedAccount`

You signed an owner-only action with another key, or the wallet is owned by a vesting contract. In the second case use `genlayer vesting validator ...`.

See the [Staking guide](staking).

---

## 9. Migration and recovery

### How do I move the node to a new server?

1. Stop the old node.
2. Install Docker and extract the release as in [Install](install).
3. Copy `config.yaml`, the manifest and `.env`.
4. Install the operator keystore (uid `10001`) and set `NODE_PASSWORD`.
5. Run `doctor` and start.

Never run two nodes with the same operator.

### What must I keep safe?

The **owner key**, the **operator keystore** and its passphrase, and the validator wallet address. Losing the operator is recoverable, losing the owner is not.

---

## 10. Safety

- Never share private keys, passphrases or your `.env` in logs, screenshots or chats.
- Restrict port `9153` with the firewall: `/metrics` and `/health` have no authentication.
- Check contract and wallet addresses against the official GenLayer documentation.

# GenLayer Validator Key Management (Bradbury Phase 2)

Owner and operator keys: where they live, how to back them up, install them on a Docker Compose node, and rotate the operator.

_Last updated: 2026-11-21. Node `v0.6.0-rc.5`, CLI `genlayer@rc`._

## 1. Key model

Each validator has two keys with very different risk profiles.

| | Owner key (cold) | Operator key (hot) |
|---|---|---|
| Purpose | Owns the validator wallet and the stake. Only key that can deposit, exit, claim, set identity and change the operator | Signs consensus duties and pays gas from the node |
| Lives on | Your CLI machine or hardware wallet, **never the server** | The validator server, as an encrypted keystore |
| If it leaks | Stake can be withdrawn by the attacker | Attacker can sign duties and spend the operator's gas balance, but cannot touch the stake |
| If it is lost | **Stake can no longer be exited or claimed**. No recovery | Recoverable: the owner assigns a new operator |

The third address, the **Validator Wallet**, is a contract. It is public and goes in `node.validatorWalletAddress`.

## 2. Where the keys are

- **Owner:** in the CLI keystore of the machine where you ran the wizard (`genlayer account list`).
- **Operator on the server (Docker Compose):** `./data/node/keystore/operator.json` in the directory of the deployment, mounted into the container at `/app/data/node/keystore/`. It is an encrypted Web3 (geth compatible) keystore.
- **Password:** `NODE_PASSWORD` in the private `.env`, delivered to the node as `GENLAYERNODE_PASSWORD`. It must be the passphrase the keystore was exported with. There is no default.

## 3. Install the operator keystore on the server

The CLI wizard exports an encrypted operator keystore. Transfer **only that file** to the server and install it with the container user (uid `10001`):

```bash
sudo install -d -m 700 -o 10001 -g 10001 ./data/node/keystore
test ! -e ./data/node/keystore/operator.json || exit 1
sudo install -m 600 -o 10001 -g 10001 /path/to/operator-keystore.json ./data/node/keystore/operator.json
```

Then:

1. Set `NODE_PASSWORD` in `.env` to the export passphrase.
2. Set the public `node.operatorAddress` and `node.validatorWalletAddress` in `configs/node/config.yaml` (it is mounted read-only).
3. Run `doctor` to verify the key unlocks and the registration matches:

   ```bash
   docker compose --profile node run --rm --entrypoint /app/bin/genlayernode genlayer-node doctor
   ```

Alternatively, create the operator on the server (`genlayernode account new ... --setup`). For Phase 2 the CLI also needs that operator account to sign its proof of possession when you join, so export and import it into the owner-side CLI securely. Never transfer the owner key to the server.

## 4. Export and import with the CLI

```bash
genlayer account list
genlayer account show --account operator

# export an encrypted keystore (you will be prompted for passwords)
genlayer account export --account operator --output operator-backup.json

# import a keystore into the CLI
genlayer account import --path operator-backup.json
```

Prefer the interactive prompts. Passing `--password` or `--source-password` on the command line leaves the secret in your shell history and in the process list.

Other account commands: `create`, `use`, `remove`, `send`, `unlock`, `lock`. See the [account reference](https://docs.genlayer.com/api-references/genlayer-cli/accounts/account).

## 5. Back up

Back up, offline and encrypted:

- The **owner** private key or seed phrase (most important).
- The **operator keystore** `operator.json` and its passphrase (store the passphrase separately).
- `configs/node/config.yaml`, the protocol manifest and your `.env` (the `.env` holds secrets: treat it like a key).

Keep owner-only permissions on every copy, test a restore before discarding the source, and never print or export a raw private key as part of routine work. Never store the owner key on the node server.

Before a **breaking upgrade**, copy `data/node/keystore/` first: it is the only part of `data/node/` you keep.

## 6. Restore or move the node to another server

1. **Stop the old node first.** Never run two nodes with the same operator key at once.
2. Prepare the new server as in the [Install](https://cumulo.pro/services/genlayer/install) page.
3. Install the keystore as in section 3 and set `NODE_PASSWORD`.
4. Verify that the operator address derived from the keystore matches the registered operator (`genlayer staking validator-info <wallet>`).
5. Run `doctor`, then start the node.

## 7. Rotate the operator key

Rotate if the server was exposed, or as routine hygiene. The owner signs the change, and the stake is not touched.

```bash
# 1. create a new operator key on the CLI machine
genlayer account create --name new-operator

# 2. point the validator wallet to it (signed by the owner)
genlayer staking set-operator 0xYourValidatorWallet 0xNewOperatorAddress --account owner --network testnet-bradbury

# 3. check
genlayer staking validator-info 0xYourValidatorWallet --network testnet-bradbury
```

Then on the server: install the new keystore, update `NODE_PASSWORD` and `node.operatorAddress`, fund the new operator address with GEN for gas, run `doctor` and restart the node with `docker restart genlayer-node`.

> **Check before relying on it.** Phase 2 registration binds the operator with a proof of possession. Confirm on a test the behaviour of `set-operator` with the new operator, and keep the old operator running until the new one is confirmed in `validator-info`.

## 8. Operator balance

The operator pays gas for every duty. If it runs out, the node cannot submit commits, reveals or priming. Watch it on the ops server:

```bash
curl -s http://127.0.0.1:9153/balance
```

and in `GET /health` the condition `consensus.operator_funds`. Keep a comfortable margin: our node tracks it on the [Dashboard](https://cumulo.pro/services/genlayer/index).

## 9. Common errors

| Message | Cause and fix |
|---|---|
| `password is required for validator mode` | `NODE_PASSWORD` is empty in `.env` |
| `account not found` or `no key for given address or file` | Keystore not installed at `data/node/keystore/`, wrong owner (must be uid 10001) or wrong operator address in `config.yaml` |
| `could not decrypt key with given passphrase` | `NODE_PASSWORD` differs from the passphrase used at export |
| `OwnableUnauthorizedAccount` | You signed an owner-only action with the operator key, or the wallet is owned by a vesting contract (use `genlayer vesting validator ...`) |
| Node starts as `mode=full` | `validatorWalletAddress` or `operatorAddress` missing in `config.yaml` |

## Warning

If you lose the **owner** key you permanently lose the ability to withdraw, to rotate the operator and to control the validator. If you lose only the **operator** key, the validator can be recovered with a new operator signed by the owner.

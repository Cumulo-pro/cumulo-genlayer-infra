# GenLayer Validator Staking Guide (Bradbury Phase 2)

Create, fund and manage a validator with the GenLayer CLI on **Bradbury Phase 2** (testnet, chain id `4221`, network name `testnet-bradbury` in the CLI).

_Last updated: 2026-11-21. CLI: `genlayer@rc` (v0.40 line)._

> Official reference: [Setup guide](https://docs.genlayer.com/validators/setup-guide) and the [staking CLI reference](https://docs.genlayer.com/api-references/genlayer-cli/staking/staking).

---

## 1. Install the CLI

Phase 2 needs the release candidate line. The default npm tag still installs an older release.

```bash
# Node.js 20 or higher (22 for a fresh install)
npm install -g genlayer@rc
genlayer --version
genlayer network set testnet-bradbury
```

Run the CLI on a machine you trust, **not on the validator server**. It holds your owner key.

---

## 2. The three addresses

| Address | Role |
|---|---|
| **Owner** | Only address that can deposit, exit, claim and change identity. Cold key, never on the node server |
| **Operator** | Hot key on the server that signs consensus duties. Can be rotated by the owner |
| **Validator Wallet** | Smart contract created when you join. Its address goes in `node.validatorWalletAddress` |

Back up the **owner** private key or seed phrase. The wizard's keystore export covers only the operator key, and a lost owner key means the stake can no longer be exited or claimed.

---

## 3. Requirements before joining

- **GEN for self-stake.** Read the deployed minimum instead of trusting a documented number. 42,000 GEN is a protocol default, not a guaranteed current value:

  ```bash
  genlayer staking epoch-info --network testnet-bradbury
  ```

- **GEN for gas** in the owner account and in the operator account.
- Delegation does not replace the required self-stake.
- Phase 2 registration needs the **operator public key and a proof of possession** signed by the operator. The operator account must therefore be available to the CLI when you join. An external address alone is not enough, and the old `validatorJoin(address)` flow is gone.

---

## 4. Join with the wizard

```bash
genlayer staking wizard --network testnet-bradbury
```

The wizard walks through: owner account, network, funding source (wallet or vesting contract), balance check, operator keystore export, stake amount, validator creation, identity (skip with `--skip-identity`) and a final summary with the three addresses.

Save the **Validator Wallet** address and the **operator keystore** file. Install the keystore on the server as explained in the [Key Management](key) page and the [Install](install) page.

Non-interactive alternative:

```bash
genlayer staking validator-join --amount 42000gen --operator 0xOperator --account owner --network testnet-bradbury
```

---

## 5. Verify the validator

```bash
# one validator
genlayer staking validator-info 0xYourValidatorWallet --network testnet-bradbury

# all registered validators, including those not active yet
genlayer staking validators --network testnet-bradbury

# current epoch and staking parameters
genlayer staking epoch-info --network testnet-bradbury
```

The same data, with history, is on the [Validators](validators) page of this explorer.

---

## 6. Validator states

| State | Meaning |
|---|---|
| **Active** | In the validator set for the current epoch, selected for duties |
| **Inactive** | Registered but not in the set (not primed, or stake below the threshold) |
| **Quarantined** | Removed from selection for strikes (idle, wrong votes). Returns after the quarantine ends |
| **Banned** | Cannot be selected until the stated epoch, or permanently if the ban is permanent |

To see where yours stands:

```bash
genlayer staking active-validators --network testnet-bradbury
genlayer staking quarantined-validators --network testnet-bradbury
genlayer staking banned-validators --network testnet-bradbury
genlayer staking validator-history 0xYourValidatorWallet --network testnet-bradbury
```

The node reports the same on its ops server. In `GET /health`, `consensus.ban` shows `until_epoch` and `permanent`, and `consensus.prime` shows whether priming is landing. A banned validator still syncs and serves, it only earns nothing until the ban ends.

### Priming

A validator has to be **primed** to prepare its stake record for the next epoch. The node does it automatically and needs operator funds for the gas. If it fails, the health condition `consensus.prime` goes to `warning` with reason `prime_not_landing`, and the validator forfeits that epoch's inflation until the next prime lands. Any address can prime a validator:

```bash
genlayer staking validator-prime 0xYourValidatorWallet --network testnet-bradbury
genlayer staking prime-all --network testnet-bradbury   # every validator that needs it
```

---

## 7. Manage the stake

Deposits, exits, claims and identity changes are accepted only from the **owner**. The operator key cannot sign them (the call reverts with `OwnableUnauthorizedAccount`).

```bash
# add stake
genlayer staking validator-deposit 0xYourValidatorWallet --amount 1000gen --account owner --network testnet-bradbury

# exit: starts the unbonding period (7 epochs)
genlayer staking validator-exit 0xYourValidatorWallet --shares 100 --account owner --network testnet-bradbury

# claim after the unbonding period
genlayer staking validator-claim 0xYourValidatorWallet --account owner --network testnet-bradbury
```

### Identity

```bash
genlayer staking set-identity 0xYourValidatorWallet \
  --moniker "My Validator" --website "https://example.com" \
  --description "What my validator does" --account owner --network testnet-bradbury
```

> **Each `set-identity` call replaces the whole identity.** All nine fields (moniker, logo URI, website, description, email, Twitter, Telegram, GitHub, extra CID) are written every time, and any field you leave out is cleared. To change one field, pass the others again with their current values. `validator-info` shows the current values.

### Change the operator

```bash
genlayer staking set-operator 0xYourValidatorWallet 0xNewOperator --account owner --network testnet-bradbury
```

See [Key Management](key) for the full rotation procedure.

### If you staked from a vesting contract

The vesting contract owns the validator wallet, so the `genlayer staking` commands above revert. Use the `genlayer vesting validator` commands signed with the **beneficiary** account:

```bash
genlayer vesting validator deposit 0xYourValidatorWallet --amount 1000gen --account beneficiary --network testnet-bradbury
genlayer vesting validator exit    0xYourValidatorWallet --shares 100     --account beneficiary --network testnet-bradbury
genlayer vesting validator claim   0xYourValidatorWallet                  --account beneficiary --network testnet-bradbury
genlayer vesting validator list --account beneficiary --network testnet-bradbury
```

`validator-info` shows `Source: vesting (owner is a contract)` for these wallets.

---

## 8. Command reference

| Command | What it does |
|---|---|
| `staking wizard` | Interactive validator creation |
| `staking validator-join` | Join as validator by staking |
| `staking validator-deposit` | Add stake to a validator wallet |
| `staking validator-exit` | Withdraw shares (starts unbonding) |
| `staking validator-claim` | Claim after unbonding |
| `staking validator-prime`, `staking prime-all` | Prepare the stake record for the next epoch |
| `staking set-operator` | Change the operator address |
| `staking set-identity` | Set moniker, website, socials |
| `staking validator-info` | Details of one validator |
| `staking validators` | Validator set with stake, status and voting power |
| `staking active-validators`, `quarantined-validators`, `banned-validators` | Lists by status |
| `staking validator-history` | Slash and reward history (default last 10 epochs) |
| `staking epoch-info` | Current epoch and parameters |
| `staking delegator-join`, `delegator-exit`, `delegator-claim`, `delegation-info` | Delegator side |

Common options: `--network testnet-bradbury`, `--account <name>`, `--rpc <url>`, `--staking-address <addr>`. Avoid `--password`: it is visible in the process list and shell history, use the interactive prompt.

---

## 9. Cumulo validator

| Item | Value |
|---|---|
| Moniker | Cumulo |
| Validator Wallet | `0xe2c63756722362D78eA9AFd8232A89cb7d85fbbF` |
| Operator | `0xccfce5bbb87e9a71f903a7e86acfafb6677646d6` |
| Network | `bradbury-phase2` |

Live stake, status and ranking are on the [Dashboard](index) and in the [official explorer](https://scan.bradbury.genlayer.com/address/0xe2c63756722362D78eA9AFd8232A89cb7d85fbbF).

# GenLayer Validator Metrics and Health (Bradbury Phase 2)

What a GenLayer node exposes on its ops port, how to read it, and how Cumulo monitors it.

_Last updated: 2026-11-21. Based on our production node: `v0.6.0-rc.5`, GenVM `v0.6.0-rc10`, protocol `v0.6-rc-1`._

---

## 1. Ops server

The node serves four routes on the **ops port** (`node.ops.port`, default `9153`). They are always on:

| Route | Description |
|---|---|
| `GET /health` | Overall status plus a catalog of typed conditions |
| `GET /metrics` | Prometheus metrics, about 130 `genlayer_*` series plus Go and process metrics |
| `GET /balance` | Operator balance in hex, wei and GEN (validator mode only) |
| `POST /snapshot` | Operator snapshots. Denied (401) unless `GENLAYERNODE_NODE_OPERATOR_TOKEN` is set |

```bash
curl -s http://127.0.0.1:9153/health
curl -s http://127.0.0.1:9153/metrics | grep '^genlayer_node_'
curl -s http://127.0.0.1:9153/balance
```

With Docker Compose the port is published by `NODE_OPS_PORT` (default `9153`). `/metrics` and `/health` have no authentication: restrict the port with the firewall to your monitoring hosts.

---

## 2. `/health` (v0.6)

The format changed in v0.6: there is no `checks` object any more, the node returns a **catalog of conditions** and every condition is always present.

```json
{
  "status": "warning",
  "network": "bradbury-phase2",
  "node_version": "v0.6.0-rc.5",
  "genvm_version": "v0.6.0-rc10",
  "protocol_version": "v0.6-rc-1",
  "validator_address": "0xe2c6...fbbF",
  "validator_moniker": "Cumulo",
  "conditions": {
    "chain.rpc": { "state": "healthy", "...": "..." },
    "consensus.ban": {
      "state": "warning",
      "reason": "validator_banned",
      "details": { "permanent": "false", "until_epoch": "27" },
      "impact": "The consensus contracts do not select this validator for a round. The node earns nothing.",
      "action": "Wait for the stated epoch. If the ban is permanent, restake with a new validator."
    }
  }
}
```

### States

| State | Meaning | Alerted |
|---|---|---|
| `unknown` | Never observed (for example `node.snapshot_age` on a node that takes no snapshots) | no |
| `healthy` | Last observation succeeded | no |
| `suspect` | One failure, not confirmed yet | no |
| `warning` | Works, but confirmed sub-optimal | yes |
| `degraded` | Failure confirmed, the capability does not work | yes |
| `unavailable` | The node cannot do its duty | yes |

The overall `status` is the most urgent state, in the order `unavailable`, `degraded`, `warning`, `suspect`, `unknown`, `healthy`. Only `unavailable` returns **HTTP 503**, so a banned or unfunded node still answers 200.

### Conditions

| Condition | What it tells you | Typical reasons |
|---|---|---|
| `chain.rpc` | HTTP RPC to the GenLayer chain works | `chain_rpc_unreachable` (503 after 30 s) |
| `chain.websocket` | WebSocket works (needs the Conduit key) | `websocket_disconnected` (503 after 30 s) |
| `chain.sync` | Node follows the chain head | `sync_blocked` (503 after 5 min without progress) |
| `consensus.ban` | Validator is banned | `validator_banned`, details `until_epoch`, `permanent` |
| `consensus.stake` | Self-stake meets the minimum | `self_stake_pending`, `self_stake_below_minimum` |
| `consensus.operator_funds` | Operator has gas | `operator_balance_critical` |
| `consensus.prime` | Priming for the next epoch lands | `prime_not_landing` |
| `genvm.llm` | LLM providers answer | Provider down or without credit |
| `genvm.web` | GenVM web module works | |
| `webdriver.render` | Fresh render probe of the WebDriver (10 s limit) | `webdriver_check_failed` |
| `node.disk` | Free disk | `disk_space_critical` (503 at 2% free) |
| `node.snapshot_age` | Age of the newest snapshot | Stays `unknown` if no snapshots are taken |

`consensus.*` conditions are capped at `warning`, and the `genvm.*` conditions at `degraded`: a dead LLM provider keeps the node booted, synced and voting, it votes Timeout on what it cannot execute.

---

## 3. `/metrics`

Series carry the labels `network` and `node`. Main families (all prefixed `genlayer_`):

### Sync and chain

| Metric | Description |
|---|---|
| `genlayer_node_synced` | 1 when the node is synced |
| `genlayer_node_blocks_behind` | Blocks behind the chain head |
| `genlayer_node_latest_block`, `genlayer_node_synced_block`, `genlayer_node_processing_block` | Block heights |
| `genlayer_node_sync_halted`, `genlayer_node_sync_stuck_block`, `genlayer_node_sync_consecutive_failures` | Sync problems |
| `genlayer_node_uptime_seconds` | Seconds since start, detects restarts |
| `genlayer_node_info` | Build information |
| `genlayer_node_health_state`, `genlayer_node_operator_condition_state` | Health roll-up and each condition as a one-hot gauge |

### Consensus duties

| Metric | Description |
|---|---|
| `genlayer_node_consensus_submit_total{method,outcome}` | Outcome of every duty submission: `success`, `reverted` (mined, gas billed), `failed` (never landed) |
| `genlayer_node_consensus_revert_total` | Decoded revert reasons for duty transactions |
| `genlayer_node_transactions_validator_commit_total`, `..._validator_reveal_total` | Validator commits and reveals |
| `genlayer_node_transactions_leader_proposed_total`, `..._leader_revealed_total` | Leader duties |
| `genlayer_node_transactions_activated_total`, `..._accepted_synced_total` | Transactions seen |
| `genlayer_node_prime_blocked_insufficient_funds` | Priming blocked by lack of funds |

### Gas, fees and operator wallet

| Metric | Description |
|---|---|
| `genlayer_node_gas_spent_wei_total` | Cumulative gas billed to the operator |
| `genlayer_node_fees_operator_insufficient_funds_total` | Duties rejected for lack of funds. A sustained increase means the operator needs GEN |
| `genlayer_node_gas_estimation_total`, `..._gas_shortfall_total`, `..._gas_block_limit_exhausted_total` | Gas estimation quality |
| `genlayer_node_fees_distributed_wei_total`, `..._fees_settlement_events_total` | Fee settlement (`severity=alarm` needs attention) |
| `genlayer_node_transact_*` | Nonce state: stuck, dropped, gap, desync, pending |

### GenVM

| Metric | Description |
|---|---|
| `genlayer_node_genvm_executions`, `..._executions_total` | Executions running and total |
| `genlayer_node_genvm_execution_time_seconds_total` | Execution time |
| `genlayer_node_genvm_permits_current`, `..._concurrent_executions_max` | Concurrency |

### Resources

| Metric | Description |
|---|---|
| `genlayer_node_cpu_usage_percent{component}` | CPU for `node`, `genvm`, `webdriver` |
| `genlayer_node_memory_rss_bytes`, `..._memory_usage_bytes`, `..._memory_percent`, `..._memory_limit_bytes` | Memory |
| `genlayer_node_disk_usage_percent`, `..._disk_free_bytes`, `..._disk_total_bytes{directory}` | Disk per data directory |
| `genlayer_node_state_disk_bytes`, `genlayer_node_state_pruning_*` | Contract state size and pruning |
| `genlayer_node_network_rx_bytes_total`, `..._tx_bytes_total` | Traffic |

### Chain RPC client

`genlayer_rollup_rpc_requests_total`, `..._errors_total`, `..._retries_total`, `..._request_duration_seconds` (histogram), `..._circuit_state`, and the WebSocket series `genlayer_rollup_ws_*` (connections, reconnections, `ws_rpc_healthy`).

The full list with HELP texts is in the [official reference](https://docs.genlayer.com/api-references/genlayer-node/ops/metrics). Go runtime (`go_*`) and process (`process_*`) metrics are also exposed.

---

## 4. What the node does not expose

Stake, epoch, active/quarantined/banned status and rewards are **chain data**, not node metrics. They come from the CLI (`genlayer staking ...`) or from the [official explorer API](https://scan.bradbury.genlayer.com). This explorer reads them from there and shows them on the [Dashboard](index) next to the node's own health.

---

## 5. Prometheus scrape

```yaml
scrape_configs:
  - job_name: genlayer-node
    scrape_interval: 15s
    metrics_path: /metrics
    static_configs:
      - targets: ["NODE_IP:9153"]
```

Suggested alerts:

| Alert | Expression |
|---|---|
| Node down | `up{job="genlayer-node"} == 0` for 2m |
| Not synced | `genlayer_node_synced == 0` for 5m |
| Falling behind | `genlayer_node_blocks_behind > 50` for 10m |
| Restart | `changes(genlayer_node_uptime_seconds[10m]) > 0` or `resets(genlayer_node_uptime_seconds[10m]) > 0` |
| Operator low on funds | `increase(genlayer_node_fees_operator_insufficient_funds_total[15m]) > 0` |
| Failed submissions | `increase(genlayer_node_consensus_submit_total{outcome="failed"}[15m]) > 0` |
| Disk | `genlayer_node_disk_usage_percent > 85` |

A health-based alert is also useful, for example a small exporter or a blackbox probe on `/health` that fires on `status` other than `healthy`. Remember that `warning` can be legitimate (for example during a ban).

---

## 6. Centralized push to the Foundation Grafana Cloud (Alloy)

The release `docker-compose.yaml` defines an `alloy` service under the `monitoring` profile that pushes metrics and logs to a Grafana Cloud tenant. All six `CENTRAL_*` values are required (Alloy refuses to start with any missing), get them from the Foundation team in the `#bradbury` channel:

```env
CENTRAL_MONITORING_URL=
CENTRAL_LOKI_URL=
CENTRAL_MONITORING_USERNAME=
CENTRAL_MONITORING_PASSWORD=
CENTRAL_LOKI_USERNAME=
CENTRAL_LOKI_PASSWORD=

NODE_ID=<validator wallet address>
VALIDATOR_NAME=<moniker>
NETWORK_NAME=bradbury-phase2
NODE_METRICS_ENDPOINT=host.docker.internal:9153
LOG_FILE_PATTERN=/var/log/genlayer/node*.log
NODE_LOGS_PATH=./data/node/logs
```

Start it together with the node:

```bash
docker compose --profile node --profile monitoring up -d
```

Verify:

- Alloy UI: `http://localhost:12345/targets`, the node target must be **UP**.
- `docker logs genlayer-node-alloy` has no 401, 403 or 500.
- In Grafana Cloud, query `genlayer_node_uptime_seconds{instance="<NODE_ID>"}`.
- Public dashboard: [GenLayer Foundation](https://genlayerfoundation.grafana.net/public-dashboards/66a372d856ea44e78cf9ac21a344f792).

Common problems: URLs with a trailing slash, wrong credentials (401/403), ops port not reachable from the Alloy container.

---

## 7. Container health

The WebDriver and Alloy containers have health checks and restart themselves when a check keeps failing:

```bash
docker inspect --format '{{.State.Health.Status}}' genlayer-node-webdriver
docker inspect --format '{{.State.Health.Status}}' genlayer-node-alloy
docker compose logs --since 30m webdriver-container alloy
```

Set `HEALTHCHECK_AUTO_RESTART=false` in `.env` to keep the checks but turn off the automatic restarts.

---

## 8. How Cumulo uses it

The [Dashboard](index) of this explorer reads `/health`, `/metrics` and `/balance` of our node and shows: sync and block lag, uptime, CPU, memory, disk, the health conditions with their reason and action text, operator balance, and consensus submission outcomes. Stake and ranking come from the official explorer API.

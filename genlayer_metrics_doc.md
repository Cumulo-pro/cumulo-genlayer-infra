# GenLayer Validator Metrics Documentation

This document describes all **currently exposed metrics** by a GenLayer validator node, based on a **real production node running v0.4.2**, and provides guidance for professional monitoring setups.

---

## 1. Overview

A GenLayer node exposes two diagnostic endpoints on the **Ops port (default `9153`)**:

| Endpoint | Description |
|--------|-------------|
| `/health` | High-level node health summary in JSON format |
| `/metrics` | Prometheus-formatted metrics (Go runtime + process) |

Example:

- `http://<node-ip>:9153/health`
- `http://<node-ip>:9153/metrics`

---

## 2. `/health` Endpoint

### Example Output (Real Node)

```json
{
  "checks": {
    "validating": {
      "status": "up",
      "timestamp": "2026-01-02T18:32:49.291636613Z"
    },
    "zksync": {
      "status": "up",
      "timestamp": "2026-01-02T18:32:49.386973419Z"
    },
    "zksync-websocket": {
      "status": "up",
      "timestamp": "2026-01-02T18:32:49.416261997Z"
    }
  },
  "genvm_version": "v0.2.7",
  "node_version": "v0.4.2",
  "protocol_version": "v0.4:1dd08066",
  "status": "up"
}
```

### 2.1 Meaning of Each Field

| Field | Description |
|------|-------------|
| `status` | Global node status (`up` means operational). |
| `checks.validating.status` | Validator mode is active and the validation loop is running. |
| `checks.zksync.status` | RPC connectivity to zkSync L1. |
| `checks.zksync-websocket.status` | WebSocket connectivity to zkSync L1. |
| `genvm_version` | Installed GenVM module version. |
| `node_version` | Node binary version running. |
| `protocol_version` | Consensus protocol version/hash. |

### 2.2 Suggested Prometheus Metrics (via small exporter)

`/health` is JSON, so you typically transform it into Prometheus metrics:

```text
genlayer_health_up 1
genlayer_validating_up 1
genlayer_zksync_rpc_up 1
genlayer_zksync_ws_up 1

genlayer_node_version_info{version="v0.4.2"} 1
genlayer_genvm_version_info{version="v0.2.7"} 1
genlayer_protocol_version_info{version="v0.4:1dd08066"} 1
```

---

## 3. `/metrics` Endpoint (Prometheus Format)

The `/metrics` output currently exposes **Go runtime**, **memory**, **CPU**, **network IO**, and **process-level** information.

> Important: these are mostly infrastructure metrics. Validator-state and staking metrics are not included (see section 4).

Below is a categorized breakdown of what your node currently emits.

---

## 3.1 Go Runtime Metrics

### Garbage Collection

- `go_gc_duration_seconds`: GC pause times (summary).
- `go_gc_gogc_percent`: GC target heap percentage (GOGC).
- `go_gc_gomemlimit_bytes`: Configured Go runtime memory limit (GOMEMLIMIT).
- `go_memstats_last_gc_time_seconds`: Timestamp of the last garbage collection.

### Goroutines & Threads

- `go_goroutines`: Total goroutines running.
- `go_threads`: OS threads created.
- `go_sched_gomaxprocs_threads`: Current GOMAXPROCS value.

### Memory Statistics

- `go_memstats_alloc_bytes`: Live heap allocations (in use).
- `go_memstats_alloc_bytes_total`: Total heap allocated since start.
- `go_memstats_buck_hash_sys_bytes`: Bytes used by profiling bucket hash table.
- `go_memstats_frees_total`: Total number of heap frees.
- `go_memstats_gc_sys_bytes`: Bytes used for GC metadata.
- `go_memstats_heap_alloc_bytes`
- `go_memstats_heap_idle_bytes`
- `go_memstats_heap_inuse_bytes`
- `go_memstats_heap_objects`
- `go_memstats_heap_released_bytes`
- `go_memstats_heap_sys_bytes`
- `go_memstats_mallocs_total`: Total number of heap mallocs.
- `go_memstats_mcache_inuse_bytes`
- `go_memstats_mcache_sys_bytes`
- `go_memstats_mspan_inuse_bytes`
- `go_memstats_mspan_sys_bytes`
- `go_memstats_next_gc_bytes`
- `go_memstats_other_sys_bytes`
- `go_memstats_stack_inuse_bytes`
- `go_memstats_stack_sys_bytes`
- `go_memstats_sys_bytes`

These are standard runtime metrics useful for detecting memory leaks, GC pressure, and capacity limits.

### Go Info

- `go_info{version="go1.23.12"}`: Go runtime version info (labels may vary).

---

## 3.2 Process Metrics

### CPU & Memory

- `process_cpu_seconds_total`: Total CPU time consumed.
- `process_resident_memory_bytes`: RAM usage (RSS).
- `process_virtual_memory_bytes`: Virtual memory usage.
- `process_virtual_memory_max_bytes`: Maximum addressable virtual memory.

### File Descriptors

- `process_open_fds`: Open file descriptors.
- `process_max_fds`: Maximum allowed descriptors.

### Network IO

- `process_network_receive_bytes_total`: Bytes received by the process over the network.
- `process_network_transmit_bytes_total`: Bytes sent by the process over the network.

These metrics help track load patterns, resource usage, and network behavior of the validator process.

### Process Start Time

- `process_start_time_seconds`: Start time since Unix epoch — useful to detect restarts.

---

## 3.3 Prometheus Handler Metrics

Used internally by the Prometheus HTTP handler:

- `promhttp_metric_handler_requests_in_flight`
- `promhttp_metric_handler_requests_total{code="200"}`
- `promhttp_metric_handler_requests_total{code="500"}`
- `promhttp_metric_handler_requests_total{code="503"}`

---

## 4. What Is Missing (Validator-Specific Metrics)

Although useful, the built-in metrics do **NOT** cover validator-critical information such as:

| Needed Metric | Available? | Notes |
|---------------|------------|-------|
| Are we signing consensus events? | ❌ | Not exposed. |
| Self stake / total stake | ❌ | Only available via CLI (`validator-info`). |
| Epoch number | ❌ | Available via CLI (`epoch-info`). |
| Live / banned / quarantined status | ❌ | CLI only. |
| Pending deposits or withdrawals | ❌ | CLI only. |
| Activation epoch countdown | ❌ | CLI only. |
| Needs priming | ❌ | CLI only (when present). |

This means a professional monitoring setup requires an **external exporter**, which polls the CLI and exposes Prometheus metrics.

---

## 5. Recommendations for Monitoring

### 5.1 Essential Infra Metrics (Already Available)

| Purpose | Metric |
|---------|--------|
| Detect node crash / restart | `process_start_time_seconds` |
| Memory leak / heap growth | `go_memstats_alloc_bytes`, `go_memstats_heap_alloc_bytes` |
| CPU overload | `process_cpu_seconds_total` |
| Network stall | `process_network_receive_bytes_total`, `process_network_transmit_bytes_total` |
| FD exhaustion | `process_open_fds / process_max_fds` |
| Prometheus scrape issues | `promhttp_metric_handler_requests_total{code!="200"}` |

### 5.2 Essential Health Signals (From `/health`, via exporter)

| Purpose | Suggested Metric |
|---------|------------------|
| Global health | `genlayer_health_up` |
| Validator loop health | `genlayer_validating_up` |
| zkSync RPC connectivity | `genlayer_zksync_rpc_up` |
| zkSync WS connectivity | `genlayer_zksync_ws_up` |
| Version tracking | `genlayer_node_version_info`, `genlayer_genvm_version_info`, `genlayer_protocol_version_info` |

---

## 6. Essential Validator Logic (Must Be Added Externally)

Suggested custom metrics your exporter should provide:

```text
genlayer_epoch_current
genlayer_epoch_time_until_next_seconds

genlayer_validator_stake_current
genlayer_validator_in_active_set
genlayer_validator_live
genlayer_validator_banned
genlayer_validator_needs_priming

genlayer_validator_epochs_remaining_for_activation
genlayer_validator_pending_deposits
genlayer_validator_pending_withdrawals
```

---

## 7. Summary

GenLayer exposes **excellent infra metrics**, but **no validator-state metrics**.

To monitor the node professionally:

1. **Scrape `/metrics`** for infra.
2. **Scrape `/health`** for high-level node status (convert JSON → Prometheus).
3. **Build a Staking Exporter** for:
   - Validator stake
   - Epoch info
   - Activation countdown
   - Quarantine/banning/live status
   - Pending deposits/withdrawals

---

_Last updated: 2026-01-02 (Node v0.4.2, GenVM v0.2.7, Protocol v0.4:1dd08066)_

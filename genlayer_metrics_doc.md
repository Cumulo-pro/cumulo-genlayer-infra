# GenLayer Validator Metrics Documentation

This document describes all currently available **built‑in metrics** exposed by a GenLayer validator node, along with recommendations for monitoring and alerting.

## 1. Overview

A GenLayer node exposes two diagnostic endpoints:

- **/health** — High‑level node health summary in JSON format.
- **/metrics** — Prometheus‑formatted metrics, mostly Go runtime + process metrics.

These endpoints are served on the **Ops port (default 9153)**.

---

## 2. `/health` Endpoint

Example:

```json
{
  "checks": {
    "validating": {"status": "up", "timestamp": "..."},
    "zksync": {"status": "up", "timestamp": "..."},
    "zksync-websocket": {"status": "up", "timestamp": "..."}
  },
  "genvm_version": "v0.2.7",
  "node_version": "v0.4.0",
  "protocol_version": "v0.4:d9974d16",
  "status": "up"
}
```

### Meaning of Each Field

| Field | Description |
|-------|-------------|
| `status` | Global node status (`up` means operational). |
| `checks.validating.status` | Indicates if validator mode is active and validation loop is running. |
| `checks.zksync.status` | RPC connectivity to zkSync L1. |
| `checks.zksync-websocket.status` | WebSocket connectivity to zkSync L1. |
| `genvm_version` | Installed GenVM module version. |
| `node_version` | Node binary version running. |
| `protocol_version` | Consensus protocol version. |

### Suggested Prometheus Metrics

You can transform these fields into metrics such as:

```
genlayer_health_up 1
genlayer_validating_up 1
genlayer_zksync_up 1
genlayer_zksync_ws_up 1
genlayer_node_version_info{version="v0.4.0"} 1
genlayer_genvm_version_info{version="v0.2.7"} 1
```

---

## 3. `/metrics` Endpoint (Prometheus Format)

The `/metrics` output currently exposes **Go runtime**, **memory**, **CPU**, and **process-level** information.

Below is a categorized breakdown of everything the node emits.

---

## 3.1 Go Runtime Metrics

### Garbage Collection

- `go_gc_duration_seconds`: GC pause times.
- `go_gc_gogc_percent`: GC target heap percentage.
- `go_gc_gomemlimit_bytes`: Configured memory limit.
- `go_memstats_last_gc_time_seconds`: Timestamp of the last garbage collection.

### Goroutines & Threads

- `go_goroutines`: Total goroutines running.
- `go_threads`: OS threads used.

### Memory Statistics

- `go_memstats_alloc_bytes`: Live heap allocations.
- `go_memstats_alloc_bytes_total`: Total heap allocated since start.
- `go_memstats_heap_alloc_bytes`
- `go_memstats_heap_idle_bytes`
- `go_memstats_heap_inuse_bytes`
- `go_memstats_heap_objects`
- `go_memstats_heap_released_bytes`
- `go_memstats_heap_sys_bytes`
- `go_memstats_next_gc_bytes`

These are standard runtime metrics useful for detecting memory leaks or excessive load.

---

## 3.2 Process Metrics

### CPU & Memory

- `process_cpu_seconds_total`: Total CPU consumed by the node.
- `process_resident_memory_bytes`: RAM usage (RSS).
- `process_virtual_memory_bytes`: Virtual memory usage.

### File Descriptors

- `process_open_fds`: Open file descriptors.
- `process_max_fds`: Maximum allowed descriptors.

### Network IO

- `process_network_receive_bytes_total`: Bytes received by the process.
- `process_network_transmit_bytes_total`: Bytes sent by the process.

These metrics help track load patterns and network behavior of the validator.

---

## 3.3 Prometheus Handler Metrics

Used internally for Prometheus:

- `promhttp_metric_handler_requests_in_flight`
- `promhttp_metric_handler_requests_total{code="200"}`

---

## 4. What Is Missing (Validator-Specific Metrics)

Although useful, the built‑in metrics do **NOT** cover validator‑critical information such as:

| Needed Metric | Available? | Notes |
|---------------|------------|-------|
| Are we signing consensus events? | ❌ | Not exposed. |
| Self stake / total stake | ❌ | Only available via CLI (`validator-info`). |
| Epoch number | ❌ | Available via CLI (`epoch-info`). |
| Live / banned / quarantined status | ❌ | CLI only. |
| Pending deposits or withdrawals | ❌ | CLI only. |
| Activation epoch countdown | ❌ | CLI only. |

This means a professional monitoring setup requires an **external exporter**, which polls the CLI and exposes Prometheus metrics.

---

## 5. Recommendations for Monitoring

### Essential Infra Metrics (Already Available)

| Purpose | Metric |
|---------|--------|
| Detect node crash | `process_start_time_seconds`, `genlayer_health_up` |
| Detect memory leak | `go_memstats_alloc_bytes` |
| Detect CPU overload | `process_cpu_seconds_total` |
| Detect network stall | `process_network_receive_bytes_total` |
| Detect zkSync RPC outage | `genlayer_zksync_up` |

### Essential Validator Logic (Must Be Added Externally)

Suggested custom metrics:

```
genlayer_epoch_current
genlayer_epoch_time_until_next_seconds
genlayer_validator_stake_current
genlayer_validator_in_active_set
genlayer_validator_live
genlayer_validator_banned
genlayer_validator_epochs_remaining_for_activation
genlayer_validator_needs_priming
```

---

## 6. Summary

GenLayer exposes *excellent infra metrics*, but **no validator-state metrics**.

To monitor the node professionally:

1. **Scrape `/metrics`** for infra.
2. **Scrape `/health`** for high‑level node status.
3. **Build a Staking Exporter** for:
   - Validator stake
   - Epoch info
   - Activation countdown
   - Quarantine/banning
   - Pending deposits



# GenLayer Node Upgrade — v0.4.2

**Network:** Asimov Testnet  
**Node Type:** Validator  
**Upgrade Date:** 2025-12-29  
**Operator:** Cumulo  

This document describes the **in-place upgrade** procedure used to migrate an
existing GenLayer validator node to version **v0.4.2**, without moving or
recreating the node directory.

---

## 🎯 Upgrade Goals

- Upgrade GenLayer node binary to `v0.4.2`
- Keep existing data directory, keystore and configuration
- Continue running the node via `systemd`
- Add required telemetry environment variables
- Minimize downtime during the upgrade

---

## 🧠 Important Notes

- The official GenLayer documentation recommends replacing the full node folder.
- Cumulo follows an **in-place upgrade strategy**, proven reliable in previous
  versions (`v0.3.x` and `v0.4.x`).
- This approach avoids copying large state directories and reduces downtime to
  only a few seconds.

---

## 📍 Current Node Layout

```
/home/dedicaovh/genlayer-node-linux-amd64/
├── bin/genlayernode
├── configs/
│   └── node/config.yaml
├── data/
│   └── node/
│       ├── genlayer.db/
│       └── logs/node.log
└── third_party/
    └── genvm/
```

---

## ⬇️ Download v0.4.2 Release

```
VER=v0.4.2
cd /tmp
rm -rf genlayer-$VER
mkdir -p genlayer-$VER
cd genlayer-$VER

wget https://storage.googleapis.com/gh-af/genlayer-node/bin/amd64/${VER}/genlayer-node-linux-amd64-${VER}.tar.gz
tar -xzf genlayer-node-linux-amd64-${VER}.tar.gz
```

---

## 🔁 In-place Upgrade Procedure

```
LIVE=/home/dedicaovh/genlayer-node-linux-amd64
NEW=/tmp/genlayer-v0.4.2/genlayer-node-linux-amd64

sudo systemctl stop genlayernode

cp -a "$NEW/bin/genlayernode" "$LIVE/bin/genlayernode"

rm -rf "$LIVE/third_party"
cp -a "$NEW/third_party" "$LIVE/third_party"

sudo systemctl start genlayernode
```

---

## ✅ Version Verification

```
EXE=$(readlink -f /proc/$(systemctl show -p MainPID --value genlayernode)/exe)
$EXE version
```

Expected:

```
GenLayer Node
Version:  v0.4.2
Protocol: v0.4:1dd08066
Commit:   9780557
Date:     2025-12-23
```

---

## 📡 Telemetry Configuration (Required)

Environment variables added to systemd service:

```
CENTRAL_MONITORING_USERNAME
CENTRAL_MONITORING_PASSWORD
CENTRAL_LOKI_USERNAME
CENTRAL_LOKI_PASSWORD
```

Reload and restart:

```
sudo systemctl daemon-reload
sudo systemctl restart genlayernode
```

---

## 🩺 Health Check

```
curl http://127.0.0.1:9153/health
```

Expected status while priming:

```
node is not primed
```

---

## ⚠️ Known Issue — Validator Priming Loop

After upgrading to `v0.4.2`, validators may repeatedly send `ValidatorPrime`
transactions while remaining unprimed.

This is a **known testnet issue** acknowledged by the GenLayer team.

---

## 📄 Logs (systemd)

```
/home/dedicaovh/genlayer-node-linux-amd64/data/node/logs/node.log
```

---

## 📌 Conclusion

- Upgrade completed successfully
- Node running v0.4.2
- Telemetry configured
- Validator priming pending due to testnet state

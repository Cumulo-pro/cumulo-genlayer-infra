# GenLayer Validator Staking & Operations Guide

## Overview
This guide documents the staking operations, validator status queries, and validator-set interactions for the Cumulo GenLayer validator on **Testnet Asimov**.

## Validator Details
- **Validator Address:** `0x0F67c2fa8bb42969CE34517d911efFC025bedF07`
- **Network:** `testnet-asimov`
- **RPC:** `https://genlayer-testnet.rpc.caldera.xyz/http`

## 1. Fetch Epoch Information
```bash
genlayer staking epoch-info   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

## 2. Fetch Validator Information
```bash
genlayer staking validator-info   --validator 0x0F67c2fa8bb42969CE34517d911efFC025bedF07   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

## 3. Make Additional Self-Stake Deposit
```bash
genlayer staking validator-deposit   --validator 0x0F67c2fa8bb42969CE34517d911efFC025bedF07   --amount 10000gen   --account owner   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

## 4. Verify Pending Deposits
```bash
genlayer staking validator-info   --validator 0x0F67c2fa8bb42969CE34517d911efFC025bedF07   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

## 5. List Active Validators
```bash
genlayer staking active-validators   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

## 6. Check Quarantined Validators
```bash
genlayer staking quarantined-validators   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

## 7. Check Banned Validators
```bash
genlayer staking banned-validators   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

## Quick Commands Summary

### Epoch Info
```bash
genlayer staking epoch-info --network testnet-asimov --rpc <RPC>
```

### Validator Info
```bash
genlayer staking validator-info --validator <ADDR> --network testnet-asimov --rpc <RPC>
```

### Stake Deposit
```bash
genlayer staking validator-deposit --validator <ADDR> --amount 1000gen --account owner --network testnet-asimov --rpc <RPC>
```

### Active Validators
```bash
genlayer staking active-validators --network testnet-asimov --rpc <RPC>
```

### Quarantined Validators
```bash
genlayer staking quarantined-validators --network testnet-asimov --rpc <RPC>
```

### Banned Validators
```bash
genlayer staking banned-validators --network testnet-asimov --rpc <RPC>
```

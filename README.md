# Cumulo: GenLayer Validator Infrastructure

**Bradbury Phase 2 (testnet): validator, staking, key management and monitoring with Docker Compose**

This repository documents how Cumulo runs a **GenLayer validator on Bradbury Phase 2** (chain id `4221`): installation with Docker Compose, staking operations, key management, monitoring endpoints and the operational practices we follow.

Current setup: node `v0.6.0-rc.5`, GenVM `v0.6.0-rc10`, protocol `v0.6-rc-1`, deployed with the release `docker-compose.yaml`.

Live status of our validator, node health and the validator set: https://cumulo.pro/services/genlayer/

## Official GenLayer resources

- Docs: https://docs.genlayer.com/
- Validator setup guide: https://docs.genlayer.com/validators/setup-guide
- Changelog: https://docs.genlayer.com/validators/changelog
- Explorer (Bradbury): https://scan.bradbury.genlayer.com/
- GitHub: https://github.com/genlayerlabs

## Documentation index

| Guide | What it covers |
|---|---|
| [Validator installation](genlayer-validator-installation-cumulo.md) | Docker Compose deployment: services and ports, release download and signature check, `config.yaml`, protocol manifest, `.env`, operator keystore, `doctor`, start, upgrades |
| [Staking guide](genlayer_staking_guide.md) | CLI install, wizard, validator states, priming, deposit, exit, claim, identity, operator change, vesting |
| [Key management](genlayer-key-management.md) | Owner and operator keys, keystore installation, backup, restore, operator rotation |
| [Metrics and monitoring](genlayer_metrics_doc.md) | `/health` conditions, `/metrics` families, Prometheus scrape and alerts, Alloy push to Grafana Cloud |
| [Validator FAQ](faq.md) | Configuration, startup errors, health warnings, GenVM and LLM providers, migration and recovery |

## Historical documents

These describe earlier networks and versions and are kept for reference only.

- [Upgrade to v0.4.2 (Asimov testnet)](upgrade-v0.4.2.md)
- [Telemetry (Alloy) update fix for v0.5.0](genlayer-v0_5_0-telemetry-update-fix_md)
- [Bradbury Phase 1 installation (v0.5.5, binary install)](bradbury/genlayer-validator-installation-bradbury-cumulo.md)

## About this repository

- Documentation written from our own production deployment.
- Values specific to a deployment (secrets, keys, hosts) are never published here.
- Commands are meant to be adapted: confirm the approved node version for the network in the official changelog before installing.

## Upgrade history

| Version | Date | Method | Notes |
|---|---|---|---|
| v0.4.0 | 2025-12-05 | Fresh | Initial setup (Asimov) |
| v0.4.1 | 2025-12-19 | In-place | Binary upgrade |
| v0.4.2 | 2025-12-29 | In-place | Telemetry env vars |

## About Cumulo

Cumulo is a professional blockchain infrastructure operator running validators, RPC nodes and DA infrastructure across modular ecosystems such as Celestia, Starknet, Dymension, Avail and GenLayer. Our priority is reliability, transparency and high-quality tooling for decentralized ecosystems.

Website: https://cumulo.pro/

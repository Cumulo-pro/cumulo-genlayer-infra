# 🚀 Cumulo — GenLayer Validator Infrastructure  
**Asimov Testnet — Validator, Staking, Key Management & Monitoring**

This repository documents Cumulo’s complete infrastructure, tooling, and operational workflows for running a **GenLayer validator on Asimov Testnet**.  
It includes staking operations, key management procedures, monitoring endpoints, validator lifecycle steps, and production-grade operational practices.

GenLayer introduces a modular validation layer built on zkSync technology, enabling decentralized validation markets, operator separation, and flexible staking economics.  
Cumulo operates a fully configured validator node and maintains this repository to help other operators understand and replicate a professional-grade setup.

---

## 🔗 Official GenLayer Resources

- 📘 Docs → https://docs.genlayer.com/  
- 📊 Points Program → https://points.genlayer.foundation/  
- 🔍 Explorer → https://genlayer-explorer.vercel.app/  
- 📰 Blog → https://www.genlayer.com/blog  
- 💻 GitHub → https://github.com/genlayerlabs  

---

## 📚 Documentation Index (Cumulo)

### **1️⃣ Staking Guide**  
How to stake, deposit more GEN, prime epochs, activate stake, exit, and claim withdrawals.  
👉 https://github.com/Cumulo-pro/cumulo-genlayer-infra/blob/main/genlayer_staking_guide.md

---

### **2️⃣ Key Management**  
Backup, export, restore, rotate Owner & Operator keys safely.  
👉 https://github.com/Cumulo-pro/cumulo-genlayer-infra/blob/main/genlayer-key-management.md

---

### **3️⃣ Validator FAQ**  
Quick answers about configuration, systemd, troubleshooting, WebDriver, operator import, migration & recovery.  
👉 https://github.com/Cumulo-pro/cumulo-genlayer-infra/blob/main/faq

---

### **4️⃣ Metrics & Monitoring Guide**  
Full breakdown of `/health` and `/metrics` endpoints, Prometheus metrics, monitoring strategy, and limitations.  
👉 https://github.com/Cumulo-pro/cumulo-genlayer-infra/blob/main/genlayer_metrics_doc.md  

---

## 🏗️ About This Repository

This repository includes:

- Complete validator setup carried out by Cumulo.  
- Step-by-step documentation for:
  - Running the GenLayer node
  - Staking & validator deposit flows
  - Key backups & operator setup
  - Monitoring with Prometheus
  - Systemd service management  
- Operational notes and incident logs relevant to GenLayer testnet participation.

The goal is to provide a **transparent, reproducible, and well-documented validator stack** while contributing back to the GenLayer validator community.

---

## 🛠️ What This Repo Does *Not* Contain

- No private keys  
- No confidential operator credentials  
- No system-specific secrets  

Only documentation, guides, and safe-to-share configuration examples.

---

## 🏢 About Cumulo

Cumulo is a professional blockchain infrastructure operator running validators, RPC nodes, and DA infrastructure across modular ecosystems such as Celestia, Story, Starknet, Dymension, Avail, and now **GenLayer**.  
Our priority is reliability, transparency, and high-quality tooling for decentralized ecosystems.

Visit us: https://cumulo.pro/

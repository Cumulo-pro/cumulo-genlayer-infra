
# GenLayer Validator Key Management  
**Backup • Export • Import • Rotation (Owner & Operator Keys)**  
_Last updated: December 2025_

## 📌 Key Architecture Summary
GenLayer uses a dual‑key model for each validator:

### **1. Owner Key (Cold Key)**
- Controls withdrawals and validator ownership.
- Should be stored offline.
- Rarely used.
- Highly sensitive.

### **2. Operator Key (Hot Key)**
- Used by the validator node to sign consensus messages.
- Safe to keep online.
- Can be rotated.
- Must be different for each validator.

---

## 🔐 Exporting a Key (Owner or Operator)

### **List accounts**
```bash
genlayer account list
```

### **Export a key**
```bash
genlayer account export   --account <name>   --source-password "<current_password>"   --password "<new_export_password>"   --output <filename>.json
```

### ✅ Example (Owner key)
```bash
genlayer account export   --account owner   --source-password "LV@1tRf5EQ^"   --password "LV@1tRf5EQ^"   --output owner-backup.json
```

Output:
```
✔ Account 'owner' exported to owner-backup.json
ℹ Address: 0xea75b2f766eb89c40D636f11663143F9A618623F
```

---

## 🔑 Importing a Key Into GenLayer

### **Import from keystore**
```bash
genlayer account import   --path <keystore.json>   --password "<password>"
```

---

## ♻️ Rotating the Operator Key

You can rotate your operator key at any time.

### **1. Create new key**
```bash
genlayer account create --name new-operator
```

### **2. Set new operator**
```bash
genlayer staking set-operator   --validator <validator_address>   --operator <new_operator_address>   --account owner   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

---

## 🔍 Validating Key Status

### Check validator info
```bash
genlayer staking validator-info   --validator <validator_address>   --network testnet-asimov   --rpc https://genlayer-testnet.rpc.caldera.xyz/http
```

You will see:
```
owner: <address>
operator: <address>
```

---

## 🧪 Test the Operator Key is Recognized

If operator key is correctly imported:
```bash
genlayer account list
```

If the node complains:
```
account not found
no key for given address or file
```
➡️ You must import the operator account using `genlayer account import`.

---

## 🔥 Critical Backup Recommendations

### **Backup these files OFFLINE:**
- Owner key keystore JSON  
- Operator key keystore JSON  
- Passwords (NEVER store on server)

### **Also backup:**
- `configs/node/config.yaml`  
- `data/node/keystore/*` (only if you keep operator online)

Store them in:
- Encrypted USB  
- Password manager  
- Offline machine  

Never store owner key on the node server.

---

## 🛑 Warning  
If you lose your **owner key**, you permanently lose:  
✔ ability to withdraw  
✔ ability to rotate operator  
✔ validator ownership  

If you lose only the **operator key**:  
✔ validator keeps working until rotated  
✔ you can assign a new operator using owner key  

---

## 🎯 Summary Cheat Sheet

### Export owner key
```bash
genlayer account export --account owner --source-password "<pw>" --password "<pw>" --output owner.json
```

### Export operator key
```bash
genlayer account export --account operator --source-password "<pw>" --password "<pw>" --output operator.json
```

### Import
```bash
genlayer account import --path operator.json --password "<pw>"
```

### Rotate operator
```bash
genlayer staking set-operator --validator <val> --operator <new_op> --account owner
```

### Check status
```bash
genlayer staking validator-info --validator <val>
```

---

## ✔️ File Ready for Download

This file contains the complete backup + export + import + rotation guide.


# 🧭 SYSVOL Replication Failure – Step-by-Step Fix Guide

A **comprehensive PowerShell guide** to diagnose and fix **SYSVOL replication failures** between multiple **Windows Server 2019 Domain Controllers**.

---

## 🧩 Problem Statement

**Issue:** SYSVOL and NETLOGON shares are missing on some domain controllers.  

**Environment:**
- **Domain:** `VNKN01.local`
- **Domain Controllers:**
  - 🖥️ `VNKN01-ADS-01` — Authoritative DC (PDC Emulator) – `192.168.62.11`
  - 💻 `VNKN01-ADS-02` — Non-Authoritative DC – `192.168.62.12`
  - 💻 `VNKN01-ADS-03` — Non-Authoritative DC – `192.168.62.15`

**Symptom:**  
SYSVOL and NETLOGON shares exist on `VNKN01-ADS-01` but are missing on the other two DCs.  

**Observation:**  
Active Directory object replication (users, groups, OUs) works fine — only SYSVOL replication is broken.  

**Event:**  
🪶 *Event ID 4012* — DFS Replication stopped or is inconsistent.

---

## ⚙️ Root Cause

A failure in **DFS Replication (DFSR)** is preventing SYSVOL from syncing between domain controllers.  
The **Authoritative DC (VNKN01-ADS-01)** holds the correct and updated policy data.

---

## 🚀 Recovery Overview

| Phase | Action | Purpose |
|-------|---------|----------|
| **Phase 0** | Non-Authoritative Sync | Fix one broken DC using data from another healthy DC. |
| **Phase 1** | Authoritative Sync | Rebuild full SYSVOL replication across all DCs. |
| **Phase 2** | Verify & Rebuild Shares | Ensure SYSVOL and NETLOGON are recreated and working. |

---

## 🧰 Step-by-Step PowerShell Guide

### ✅ Step 1: Verify DFSR and Netlogon Services

```powershell
Get-Service -Name DFSR, Netlogon | Select-Object Name, Status, StartType
Start-Service -Name DFSR, Netlogon
```
### **✅ Step 2: Check SYSVOL and NETLOGON Shares**

Could you confirm the presence of the required shares and the DFSR staging folder?

```powershell
# Verify SYSVOL and NETLOGON are shared
Get-SmbShare -Name SYSVOL, NETLOGON

# If shares are missing, confirm the DFSR staging folder exists
Get-ChildItem "C:\System Volume Information\DFSR"
```


### **🔄 Phase 0 – Non-Authoritative Synchronization**

Use this procedure when one Domain Controller's SYSVOL is outdated or empty and needs to pull the correct data from a healthy peer. **Run this on the broken (non-authoritative) DC.**

```powershell
# Stop the DFS Replication service
Stop-Service DFSR -Force

# Disable DFSR replication for this DC by setting msDFSR-Enabled=$false
# NOTE: Replace 'VNKN01-ADS-03' with the actual non-authoritative DC's name
$TargetDC = "VNKN01-ADS-03" 
$DN = "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=$TargetDC,OU=Domain Controllers,DC=VNKN01,DC=local"

Set-ADObject -Identity $DN -Replace @{msDFSR-Enabled=$false}


# Wait for Event ID 4114 (Replication disabled) to confirm the change has replicated.
Read-Host "Press Enter after confirming Event ID 4114 in the DFS Replication logs."

# Re-enable replication by setting msDFSR-Enabled=$true
Set-ADObject -Identity $DN -Replace @{msDFSR-Enabled=$true}

# Restart DFSR to initiate synchronization
Start-Service DFSR
```
**Expected Events (in DFS Replication Log):**

* **4614** $\to$ SYSVOL initialized (non-authoritative data pulled).
* **4602** $\to$ Replication started successfully.

### **🔑 Phase 1 – Authoritative Synchronization**

Use this when replication is broken across all domain controllers.

```powershell
# Stop DFSR on all DCs
Stop-Service -Name DFSR -Force

# Disable DFSR replication
Set-ADObject -Identity "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,
CN=VNKN01-ADS-01,OU=Domain Controllers,DC=VNKN01,DC=local" `
-Replace @{msDFSR-Enabled=$false}

# Mark Authoritative DC (VNKN01-ADS-01)
Set-ADObject -Identity "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,
CN=VNKN01-ADS-01,OU=Domain Controllers,DC=VNKN01,DC=local" `
-Replace @{msDFSR-Options=1}

# Re-enable replication on all DCs
Set-ADObject -Identity "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,
CN=<DCName>,OU=Domain Controllers,DC=VNKN01,DC=local" `
-Replace @{msDFSR-Enabled=$true}

# Start DFSR service
Start-Service -Name DFSR

```
### **🧮 Phase 2 – Validation and Share Rebuild**
  #### Check Replication State
  ```powershell
  dfsrdiag pollad
  dfsrdiag ReplicationState
  ```
  #### Rebuild Missing SYSVOL and NETLOGON Shares
  ```powershell
    Stop-Service DFSR
    Rename-Item "C:\Windows\SYSVOL\sysvol" "sysvol.old"
    New-Item -ItemType Directory -Path "C:\Windows\SYSVOL\sysvol\VNKN01.local"
    Start-Service DFSR
    Restart-Service Netlogon
    Get-SmbShare -Name SYSVOL, NETLOGON

  ```
---

## **📊 Event Log Validation**

After performing the synchronization steps, verify success by checking the Event Viewer ($\to$ Applications and Services Logs $\to$ DFS Replication) for the following events:

| Event ID | Description |
|:---------|:------------|
| **4114** | Replication disabled (Occurs after setting `msDFSR-Enabled=$false`) |
| **4602** | Authoritative DC initialized (Occurs after setting `msDFSR-Options=1` and restarting DFSR on the Authoritative DC) |
| **4614** | SYSVOL synced successfully (Occurs when a non-authoritative DC completes its initial synchronization) |

---

## **🧠 Summary**

This summary highlights the key components and concepts used in fixing SYSVOL replication:

* **AD Replication** handles user and group data via the **Remote Procedure Call (RPC)** mechanism.
* **SYSVOL Replication** handles policies and scripts via **DFS Replication (DFSR)**.
* Use **Authoritative Synchronization** (setting `msDFSR-Options=1`) when you need to **push** the correct, validated data from one healthy DC (e.g., your PDC Emulator) out to all others.
* Use **Non-Authoritative Synchronization** (disabling/re-enabling DFSR) when a broken DC needs to **pull** the current data from its replication partners.
* **Always** validate replication health (using `repadmin` and `dfsrdiag`) both before and after synchronization to confirm the fix.

https://youtube.com/playlist?list=PLKKb8uIoiVlNNynV0DzMMZZ14sC7sWaUe&si=EYkEvR9CQaRwECVx

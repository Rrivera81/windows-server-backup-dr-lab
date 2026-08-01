# Disaster Recovery Plan: FS01 Client File Server

**System:** FS01, Windows Server 2022, client matter file server  
**Scope:** E:\Clients and the operating system volume  
**Author:** Raymond Rivera  
**Status:** Lab environment. Targets below are drawn from measured lab results and would be revalidated against production hardware and data volume before being committed to.

---

## 1. What this plan protects

`E:\Clients` holds client matter files: case notes, financial disclosures, retainer agreements, correspondence, and sealed settlements. Loss of this data is a client service failure and a confidentiality problem at the same time. Unauthorized disclosure is as damaging as deletion, so access control is treated here as part of the recovery posture, not a separate topic.

---

## 2. Definitions

**RTO, recovery time objective.** How long the business can tolerate the data being unavailable.

**RPO, recovery point objective.** How much recent work the business can tolerate losing, measured backward from the moment of failure. RPO is set by backup frequency, not by restore speed.

---

## 3. Recovery tiers

Recovery is scoped to the smallest action that fixes the problem. Restoring an entire server to recover one file wastes time and introduces risk.

### Tier 1: Single file loss

**Trigger:** A user deletes or overwrites an individual document.  
**Method:** Windows Server Backup, Recover, files and folders, restore to original location.  
**Measured lab time:** Under 1 minute.  
**Target RTO:** 15 minutes, allowing for the user to report it and for the file to be located in the catalog.

### Tier 2: Folder or matter loss

**Trigger:** An entire matter folder is deleted or corrupted.  
**Method:** Same recovery wizard, folder level selection, restore to original location.  
**Measured lab result:** Full folder and contents recovered intact.  
**Target RTO:** 30 minutes.

### Tier 3: Volume or server loss

**Trigger:** Disk failure, OS corruption, failed update, or ransomware encryption across the server.  
**Method:** Boot to Windows Recovery Environment from installation media, Repair your computer, Troubleshoot, System Image Recovery, select the most recent full server backup.  
**Measured lab time:** 5 minutes 22 seconds, timed end to end from starting the restore to reaching the login screen.  
**Target RTO:** 4 hours in production, allowing for hardware replacement or provisioning, media retrieval, and post-restore verification. The restore itself is the short part. Everything around it is what consumes the window.

---

## 4. Recovery point objective

**Current lab state:** Backups were run manually, so RPO is undefined.

**Production target:** Nightly full server backup with more frequent data-only backups during business hours, giving a worst-case RPO of one business day for

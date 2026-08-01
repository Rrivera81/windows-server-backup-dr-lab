# Restore Test Log: FS01

Each test below began with real destruction of data, not a simulation. Files were permanently deleted with Shift+Delete, bypassing the Recycle Bin, so that recovery depended entirely on the backup.

**System:** FS01, Windows Server 2022, VirtualBox  
**Tool:** Windows Server Backup  
**Tester:** Raymond Rivera

---

## Backups taken before testing

### Backup A: data only

| Field | Value |
|---|---|
| Source | E: (client data) |
| Destination | F: (separate physical disk) |
| Type | Backup Once, Custom |
| Data transferred | 86.3 MB |
| Result | Completed successfully |

### Backup B: full server

| Field | Value |
|---|---|
| Source | C: (system) and E: (client data) |
| Destination | F: |
| Type | Backup Once, Full server |
| Result | Completed successfully |

**Note on source selection:** F: was excluded from the source items. A volume cannot be both the source and the destination of the same job. This is the kind of detail that produces a failed job at 2 a.m. if it is not caught at build time.

**Why both backups:** The data-only job is faster and supports day-to-day file recovery. The full server job is the only one that supports bare-metal recovery, because a file-level backup restores documents but does not produce a bootable system.

---

## Test 1: Single file recovery

**Scenario:** A user permanently deletes an active case document.

**Setup:** `E:\Clients\Smith_v_Smith\case_notes` deleted with Shift+Delete. Confirmed gone from the folder and absent from the Recycle Bin.

**Procedure:**
1. Windows Server Backup, Local Backup, Recover
2. Source: this server
3. Selected the most recent backup date
4. Recovery type: Files and folders
5. Navigated the backup tree to the Smith matter folder and selected the file
6. Destination: original location
7. Ran the recovery

**Result:** Completed successfully. File restored to its original path with contents intact.  
**Elapsed time:** Under 1 minute.

**Takeaway:** This is the most common real-world recovery request by a wide margin, and it does not require touching server state at all. Scoping the recovery correctly is what keeps a routine request from turning into an outage.

---

## Test 2: Folder level recovery

**Scenario:** An entire client matter folder is deleted.

**Setup:** `E:\Clients\Johnson_Divorce` deleted with Shift+Delete, including all four documents inside it.

**Procedure:** Same recovery wizard as Test 1, selecting the folder rather than an individual file, restored to original location.

**Result:** Completed successfully. Folder recreated with all contents present.

**Takeaway:** Folder level recovery uses the same path as single file recovery. The escalation from Tier 1 to Tier 2 is a change in selection scope, not a change in tooling or risk.

---

## Test 3: Bare-metal recovery

**Scenario:** Total server loss. Operating system unbootable or destroyed.

**Setup:** The full server backup from Backup B was the only recovery source.

**Procedure:**
1. Rebooted and booted from Windows Server 2022 installation media
2. Repair your computer
3. Troubleshoot, System Image Recovery
4. Selected the backup image, verified it by matching date, time, and size
5. Confirmed the restore scope would write only to Disk 0 (C:), leaving the data and backup volumes untouched
6. Executed the restore and timed it with a stopwatch

**Result:** Completed successfully.  
**Elapsed time: 5 minutes 22 seconds**, measured from starting the restore to reaching the login screen.

**Post-restore verification:**
- Logged in as Administrator
- C:, E:, and F: all present
- `E:\Clients` intact with all matter folders and documents
- NTFS permissions survived the restore, confirmed in Test 4 below

**Note on drive letters in recovery environment:** The backup volume appeared under a different drive letter in Windows RE than it uses in the running OS. Drive letter assignment in the recovery environment is independent of the installed OS, so this is expected behavior, not an error. The correct image was confirmed by matching the backup date, time, and size rather than trusting the letter.

---

## Test 4: Access control verification after restore

**Scenario:** Confirm that least-privilege permissions survived the bare-metal restore, and that the Deny rule actually blocks access rather than merely appearing in the ACL.

**Setup:** Local user `testparalegal` had previously been given an explicit **Deny Read** on `E:\Clients\Corporate_Confidential`.

**Procedure:**
1. Signed out of Administrator
2. Signed in as `testparalegal`
3. Opened File Explorer and navigated to `E:\Clients`
4. Attempted to open `Corporate_Confidential`

**Result:** Access denied. Windows returned a permission error stating the account does not currently have permission to access the folder.

The elevation prompt offering to grant access permanently was **cancelled, not accepted**. Clicking through it would have written a new ACE and defeated the control being tested.

**Takeaway:** Configuring a permission and verifying a permission are two different pieces of work. The ACL entry only proves intent. Logging in as the restricted user and hitting the denial is what proves the control functions, and it also confirms the rule survived a full server rebuild.

---

## Summary

| Test | Tier | Result | Time |
|---|---|---|---|
| 1 | Single file | Recovered | Under 1 minute |
| 2 | Folder | Recovered | Intact, all contents present |
| 3 | Bare-metal | Recovered | **5 minutes 22 seconds** |
| 4 | Access control | Denial confirmed after restore | n/a |

All measured in a lab on local virtual disks. Production times would be longer and would need to be measured against real data volume and hardware rather than assumed from these figures.

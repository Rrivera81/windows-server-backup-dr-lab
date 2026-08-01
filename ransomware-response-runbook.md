# Ransomware Response Runbook: FS01 File Server

**Purpose:** A decision sequence for a suspected ransomware or mass encryption event affecting the client file server.  
**Author:** Raymond Rivera  
**Status:** Lab document, written against the FS01 build. In production this would require sign-off from firm leadership and legal counsel, since client notification obligations are a legal question and not an IT decision.

---

## Governing principle

The instinct in an encryption event is to start restoring immediately. That instinct destroys evidence, and it frequently restores straight back into an environment where the attacker still has access, which means encrypting the restored data as well.

**Order of operations: contain, then assess, then recover.** Recovery is the last step, not the first.

---

## Phase 1: Contain

Speed matters most here, because encryption spreads over network shares.

1. **Disconnect the server from the network.** Pull the network connection or disable the adapter. Do not power the server off. Shutting down destroys volatile evidence and can leave files in a partially encrypted state.
2. **Disconnect or take offline the backup target.** If the backup volume is still reachable from the compromised host, it is a target. This is the single most important step for preserving the ability to recover at all.
3. **Isolate other affected endpoints.** The server is often not the origin. Identify which workstation the encrypting account was logged in from.
4. **Disable the compromised account.** If the encrypting process ran under a known user account, disable it before it can be reused.
5. **Notify.** Firm leadership and whoever owns the incident response decision. In a law firm, potential exposure of client matter files is a privilege issue, so counsel is notified early rather than after the technical work is finished.

> **Do not pay, negotiate, or contact the attacker without an explicit decision from firm leadership and counsel.** That is not an administrator's call to make.

---

## Phase 2: Assess

1. **Determine scope.** Which volumes and which folders are encrypted. Check whether `E:\Clients` is affected in whole or in part, and whether the system volume is affected.
2. **Determine time of onset.** File modification timestamps on the earliest encrypted files establish roughly when it started. This is the number that determines which backup is clean.
3. **Verify backup integrity before trusting it.** Check backup completion history and confirm a restore point exists that predates the earliest encrypted file. A backup taken after onset may contain encrypted data.
4. **Identify the entry point if possible.** Restoring without answering this means restoring into the same vulnerability.
5. **Preserve evidence.** Note ransom note filenames, file extensions used, and affected paths before any changes are made.

---

## Phase 3: Recover

Recovery tier is chosen by scope, using the [disaster recovery plan](disaster-recovery-plan.md).

### If only client data is encrypted and the OS is clean

1. Confirm the server is still isolated from the network.
2. Restore `E:\Clients` from the most recent backup that predates onset, using Windows Server Backup, Recover, files and folders.
3. Verify restored file contents open normally and are not encrypted.
4. Verify NTFS permissions on restored folders, particularly any Deny rules protecting sealed matters.

### If the operating system is compromised or the scope is unclear

Do not attempt to clean the existing OS. Assume persistence.

1. Perform a bare-metal restore from the most recent clean full server backup, per Tier 3 of the disaster recovery plan. Boot from installation media, Repair your computer, Troubleshoot, System Image Recovery.
2. **Measured restore time in this lab: 5 minutes 22 seconds.** The restore itself is short. The assessment work in Phase 2 is what determines the real outage length.
3. Keep the server off the network after the restore completes.

---

## Phase 4: Before reconnecting

Reconnecting too early is how a second encryption event happens.

1. **Change credentials.** Administrator and any service or user account that may have been exposed.
2. **Patch.** Apply outstanding operating system and application updates before the server is reachable again.
3. **Verify the entry point is closed.** If the origin was a workstation, that workstation is rebuilt, not cleaned.
4. **Confirm endpoint protection is running and current on the restored server.**
5. **Spot check data integrity** across multiple matter folders, not just one.
6. **Verify access controls** by testing with a restricted account rather than assuming the ACLs came back correctly.
7. **Reconnect and monitor** for unusual file modification activity.

---

## Phase 5: After

1. Record the timeline: onset, detection, containment, recovery complete, service restored.
2. Compare actual recovery time against the RTO targets in the disaster recovery plan and correct the targets if they were wrong.
3. Identify the control that would have prevented it, and the control that would have detected it sooner. These are usually two different controls.
4. Confirm client notification decisions with counsel.

---

## The gap this lab exposes

The current FS01 build has **no immutable off-site copy.** Backups sit on a second disk in the same host, reachable from the same operating system. An attacker with administrator credentials on FS01 can delete them, which turns a recoverable incident into an unrecoverable one.

The planned fix is an off-site copy in Amazon S3 with versioning and Object Lock enabled, written by a least-privilege IAM user holding put permissions and no delete permission. Object Lock means that even valid credentials cannot remove an object before its retention period expires, so the off-site copy survives full compromise of the server. This is the single highest-value improvement available to this build.

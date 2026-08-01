# Windows Server 2022 File Server: Backup, Access Control, and Disaster Recovery Lab

A self-built lab simulating a small law firm file server, covering disk provisioning, NTFS least-privilege access control, scheduled backup, and three tiers of restore, including a full bare-metal recovery timed at **5 minutes 22 seconds**.


---

## Why I built this

Client matter files are the most sensitive data a law firm holds. Two questions matter more than anything else:

1. If a matter folder is deleted, encrypted, or corrupted, how fast can it come back?
2. Can someone who should not see a file actually see it?

This lab answers both with measured results rather than assumptions.

---

## Environment

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Guest OS | Windows Server 2022 Standard Evaluation (Desktop Experience) |
| Hostname | FS01 |
| Disk 0 | 50 GB, C: (operating system) |
| Disk 1 | 40 GB, E: (client data) |
| Disk 2 | 80 GB, F: (backup target) |
| Backup tool | Windows Server Backup |

Data and backup live on separate physical disks so that a single disk failure does not take out both the source and the copy.

---

## What was built

**Disk provisioning.** Two additional virtual disks were attached, initialized, partitioned, formatted NTFS, and assigned drive letters. Provisioned with DiskPart after the PowerShell storage cmdlets failed to load in a fresh install session. See [troubleshooting-log.md](troubleshooting-log.md).

**Client data structure.** `E:\Clients\` containing three matter folders with representative documents:

```
E:\Clients\
├── Smith_v_Smith\          case notes, financial disclosure, invoice, retainer agreement
├── Johnson_Divorce\        billing, case notes, correspondence, parenting plan
└── Corporate_Confidential\ sealed settlement
```

**Least-privilege access control.** A local user account, `testparalegal`, was created to represent a staff member with no business need to see sealed matters. An explicit **Deny Read** ACE was applied to `Corporate_Confidential` for that account. The rule was then verified by logging in as that user and attempting to open the folder, which returned an access denied error.

**Backup.** Two backup jobs were run with Windows Server Backup:

- Data-only backup of E: to F: (86.3 MB transferred)
- Full server backup of C: and E: to F:, which is what makes bare-metal recovery possible

F: was deliberately excluded from the source selection, since a volume cannot serve as both source and destination.

---

## Restore results

Three restore scenarios were tested, each starting from a real deletion or wipe, not a simulation.

| Test | Scenario | Method | Result |
|---|---|---|---|
| 1 | Single file permanently deleted | Windows Server Backup, Recover, files and folders | Recovered to original location in under 1 minute |
| 2 | Entire matter folder permanently deleted | Windows Server Backup, Recover, files and folders | Folder and all contents recovered intact |
| 3 | Full server loss | Windows RE, System Image Recovery | **5 minutes 22 seconds**, timed |

After the bare-metal restore, C:, E:, and F: were all present and intact, and the folder structure and permissions survived the rebuild.

Full detail in [restore-test-log.md](restore-test-log.md).

---

## Documents in this repo

| File | Contents |
|---|---|
| [disaster-recovery-plan.md](disaster-recovery-plan.md) | Recovery tiers, RTO and RPO targets backed by measured times, and the recovery decision path |
| [restore-test-log.md](restore-test-log.md) | Step by step record of each restore test with results |
| [ransomware-response-runbook.md](ransomware-response-runbook.md) | Containment through recovery procedure for an encryption event |
| [troubleshooting-log.md](troubleshooting-log.md) | Problems hit during the build and how each was resolved |

---

## Screenshots

See [screenshots/](screenshots/) for evidence of each stage, including the disk layout, the folder structure, the Deny permission entry, both backup completion screens, all three restore completions, and the access denied error produced when the restricted user attempted to open the sealed matter folder.

---

## Known gaps and next steps

Being direct about what this lab does not yet cover:

- **No off-site copy yet.** Both the data and the backup live on the same host. The next build step is replicating the backup to Amazon S3 with versioning and Object Lock enabled, written by a least-privilege IAM user with no delete permission, so that an attacker holding server credentials still cannot destroy the off-site copy.
- **Backups were run manually.** A production build would use a scheduled job with alerting on failure, since an unmonitored backup is an assumption rather than a control.
- **Single server, no domain.** Permissions were applied to a local account. In a domain or Entra ID environment, the same rule would be applied to a security group rather than an individual user, so that access follows role rather than person.
- **Restore times are lab times.** A 40 GB volume on a local virtual disk restores faster than a production file server over a network. The number that transfers is the method and the discipline of testing, not the raw figure.

---

## What I would carry into a production environment

Backups that have never been restored are not backups. Every recovery path in this lab was tested by actually destroying the data first and timing the recovery, and every result above is a measured outcome rather than an expectation.

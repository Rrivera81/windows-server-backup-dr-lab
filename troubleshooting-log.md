# Troubleshooting Log: FS01 Build

Problems encountered during the build and how each was resolved. Included because the resolution path is usually more useful than a clean walkthrough that pretends nothing went wrong.

---

## Issue 1: Unattended installation failed with a license terms error

**Symptom:** Creating the VM with VirtualBox's unattended installation enabled produced an error during setup stating that the Microsoft software license terms could not be found. Setup could not continue.

**Cause:** VirtualBox's unattended install automation injects an answer file and does not reliably match the media layout of the Windows Server 2022 evaluation ISO. The installer looked for license content where the automation expected it to be and did not find it.

**Resolution:** Deleted the VM and rebuilt it with **unattended installation unchecked**, then performed a manual installation, selecting Standard Evaluation with Desktop Experience and Custom install.

**Takeaway:** Automation that saves ten minutes is not worth an hour of diagnosing a failure inside a black box. When the automated path fails on a one-off build, the manual path is usually faster than debugging the automation.

---

## Issue 2: PowerShell storage cmdlets unavailable in a fresh install

**Symptom:** After attaching the two additional virtual disks, `Get-Disk` returned the expected inventory, confirming all three disks were visible to the OS. However `Initialize-Disk` and the related partition and format cmdlets failed to run, reporting that the commands were not recognized.

**Diagnosis:** `Get-Disk` succeeding while `Initialize-Disk` failed indicated the issue was module-scoped rather than a hardware or attachment problem. The disks were present and readable. The Storage module was not fully loaded into the session. An explicit `Import-Module Storage` did not resolve it cleanly.

**Resolution:** Switched to **DiskPart** from Command Prompt, which is present in every Windows Server installation and does not depend on module loading. Per disk:

```
diskpart
select disk 1
clean
create partition primary
select partition 1
format fs=ntfs quick label=Data
assign letter=E
```

Repeated for Disk 2 with label Backup and letter F.

**Result:** Both disks provisioned successfully. Verified in File Explorer and Disk Management.

**Takeaway:** Confirming that the underlying object exists before troubleshooting the tool saved time here. `Get-Disk` returning correct output ruled out the disks themselves in one command, which pointed straight at the tooling. Knowing a second path to the same outcome is what kept a blocked task moving.

---

## Issue 3: Drive letter D unavailable

**Symptom:** Attempting to assign letter D to the first data disk failed.

**Cause:** D was already assigned to the virtual optical drive holding the Windows Server installation ISO.

**Resolution:** Assigned E to the data volume and F to the backup volume, and kept that mapping consistent through the rest of the build and all documentation.

**Takeaway:** Minor, but worth recording. Inconsistent drive letters between documentation and reality is exactly the kind of small discrepancy that causes a restore to be attempted against the wrong volume under pressure.

---

## Issue 4: Password rejected when creating the test user

**Symptom:** Creating the local user `testparalegal` failed on the first attempt when the password contained the word "paralegal."

**Cause:** Windows password complexity policy rejects passwords containing the account name or significant portions of it.

**Resolution:** Chose a compliant password that did not derive from the username.

---

## Issue 5: Backup job would have included the destination volume

**Symptom:** On the full server backup, the default source selection included F:, the backup destination.

**Cause:** Full server backup selects all volumes by default.

**Resolution:** Manually deselected F: from the source items, keeping C: and E: as sources and F: as the destination only.

**Takeaway:** A volume cannot be both the source and the target of the same job. Caught at build time, this is a checkbox. Missed, it becomes a failed or bloated backup discovered at the worst possible moment.

---

## Issue 6: Backup volume appeared under a different drive letter in the recovery environment

**Symptom:** During the bare-metal restore, the Windows Recovery Environment displayed the backup volume under a different drive letter than F:, which is what it uses in the running OS.

**Cause:** Windows RE assigns drive letters independently of the installed operating system. This is expected behavior, not a fault.

**Resolution:** Verified the correct image by matching the backup **date, time, and size** against the known backup job rather than relying on the drive letter. Proceeded once confirmed.

**Takeaway:** Under recovery conditions, the identifiers you trust in normal operation may not be present. Verifying against an attribute that does not change, in this case the backup timestamp, is what prevented restoring the wrong image.

---

## Issue 7: Ctrl+Alt+Del not reaching the guest OS

**Symptom:** After the bare-metal restore, the guest was sitting at a lock screen requiring Ctrl+Alt+Del, but the key combination was being intercepted by the host operating system.

**Resolution:** Used the VirtualBox menu path: Input, Keyboard, Insert Ctrl-Alt-Del, which sends the sequence to the guest rather than the host. Right Ctrl plus Delete also works as a shortcut using the default VirtualBox host key.

---

## Summary

Seven issues across one build, none of which were in the lab instructions. The two worth carrying forward are the DiskPart fallback and the drive letter verification during recovery, because both are situations where the expected tool or the expected identifier was unavailable and the task still had to complete.

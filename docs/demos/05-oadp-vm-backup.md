---
title: "Demo 5 (bonus) — Back up and restore your VM with OADP"
subtitle: "OpenShift API for Data Protection: back up your namespace (VM, disks, Service), lose it, restore it, prove the data"
version: 0.3.0
status: framework
date: 2026-10-17
---

# Demo 5 (bonus): Back up and restore your VM with OADP

> **Version 0.3.0 · framework · 2026-10-17** · class/demos/05-oadp-vm-backup.md

> 🚧 **Bonus, framework.** OADP is now built by the lab when a class books it (option `backup: oadp`, the default for this
> class). The commands below are unproven **[verify]** until a booking has run them. Ask your instructor whether your
> lab has OADP before you start.

## What you will learn

Disaster recovery (Demo 4) keeps a running copy on another site. A **backup** is different: a point-in-time copy you
can restore after a mistake, a bad change or a ransomware event, even when every replica has the damage too. You will
back up your whole namespace with **OADP** (the OpenShift API for Data Protection, Red Hat's Velero-based operator): the
`VirtualMachine`, its disks and its Service. Then you delete it all, restore it, and prove with your own rows that the
restored VM holds exactly the data it had at backup time.

## How the lab is set up for this demo

| Where | What is pre-built (planned) | How it is built |
|---|---|---|
| Both sites (demo-openshift, `backup: oadp`) | the **OADP operator** (1.5) in `openshift-adp`, a `DataProtectionApplication` with the `openshift`, `aws`, `kubevirt` and `csi` plugins, the node agent and the **Data Mover** (disk snapshots copied off the cluster), and **non-admin backups**: you back up and restore your **own namespace** yourself | [OADP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/index) [verify link] |
| Both sites | a **backup location** on the lab's object store (MinIO, outside the clusters), one bucket per cluster and a folder per booking: backups outlive the cluster | [Backup locations](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/index) [verify link] |
| You | a VM running on site A in your namespace: first your **cloud-native** server (after [Demo 1a](01a-snapshot-restore.md) Part 1, while Demo 1's copy runs), then, optionally, your **migrated Windows** VM (after [Demo 1](01-vmware-to-ove-cold.md) §1.5) | — |

**You do yourself:** inject data, request a backup of your namespace, delete your VM, restore it, and verify.

> 🖥️ **Where you type the commands: your browser terminal** ([Toolkit T.1](toolkit.md)), signed in on site A, with
> `dblogin` done. `powershell` blocks run inside your Windows VM, from its console.

**Which VM.** The terminal's commands (`alive`, `mark`, `verify`) follow `$VM` and `$PGHOST`. **On your cloud-native
server** (the first time through), point them at it, with your own number:

```bash
export VM=pg-s01-centos10 PGHOST=10.101.23.151
```

**On your Windows VM** (after Demo 1), open a new terminal tab, or `unset PGHOST; export VM=win-pg-s01`.

<!-- 🎬 video slot 05-oadp-vm-backup: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/05-oadp-vm-backup.jpg">
  <source src="../.images/05-oadp-vm-backup.mp4" type="video/mp4">
  <a href="../.images/05-oadp-vm-backup.mp4">🎬 Video: Demo 5: OADP backup and restore</a>
</video>
<sub>🎬 *Demo 5: OADP backup and restore: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 5.1 What a VM backup contains

A backup of your namespace captures its **objects** (the `VirtualMachine`, its `DataVolume`s and PVCs, the Service) and
its **data**: each disk is snapshotted, and the Data Mover copies the snapshot to object storage, so the backup
survives even if the cluster's storage is lost. A running Windows VM is backed up **crash-consistent** unless the guest
agent freezes it first: PostgreSQL recovers from that like from a power cut, and your hash chain proves it.

```bash
oc --context site-a -n "$VMNS" get vm,dv,pvc,svc               # what your backup will contain
oc --context site-a -n "$VMNS" get nonadminbackupstoragelocations.oadp.openshift.io 2>/dev/null || echo "(the class's location is the default one)"   # [verify]
```

---

> 💡 **Use full resource names here.** On these clusters `oc get backup` finds ODF's database-backup type and
> `oc get restore` finds ACM's, not OADP's. Several operators define a "backup" or "restore". Always write the full name,
> for example `nonadminbackups.oadp.openshift.io` or `backups.velero.io`.

## 5.2 Inject before the backup

```bash
alive                              # your VM is writing
mark d5-in-backup                  # ✍️ note the id: this row MUST come back
verify                             # chain_ok = t
```

On Windows, also write a line into a file in the VM, from its console (PowerShell):

```powershell
Add-Content "C:\Users\Public\ovedr\$env:USERNAME.txt" "$(Get-Date -Format o)  d5-in-backup"
```

---

## 5.3 Back up your namespace

You create a **`NonAdminBackup`** in your own namespace; OADP turns it into a backup of that namespace only.
**[verify: the NonAdminBackup fields and the student rights on OADP 1.5 (demo-openshift reports what the operator
grants)]**

```bash
oc --context site-a -n "$VMNS" apply -f - <<EOF
apiVersion: oadp.openshift.io/v1alpha1
kind: NonAdminBackup
metadata: { name: d5 }
spec:
  backupSpec:
    snapshotMoveData: true        # the Data Mover: your disks' data leaves the cluster's storage
    ttl: 24h0m0s
EOF
oc --context site-a -n "$VMNS" get nonadminbackups.oadp.openshift.io d5 -w            # Ctrl-C when its phase is Created and the backup Completed
mark d5-after-backup               # this row is NOT in the backup: it must be gone after the restore
```

**Console:** your project → **Operators → Installed Operators → OADP → NonAdminBackup → Create**.

✍️ **Row 5.3:** the backup's duration and size (`oc -n "$VMNS" get nonadminbackups.oadp.openshift.io d5 -o yaml`, under `status`).

---

## 5.4 Lose it: delete your VM

```bash
oc --context site-a -n "$VMNS" delete vm "$VM"                     # the VM and its disks are gone
oc --context site-a -n "$VMNS" get vm,pvc                          # nothing left of your VM
alive                                                              # fails: that is the disaster
```

---

## 5.5 Restore, and prove the data

```bash
oc --context site-a -n "$VMNS" apply -f - <<EOF
apiVersion: oadp.openshift.io/v1alpha1
kind: NonAdminRestore
metadata: { name: d5 }
spec:
  restoreSpec:
    backupName: d5                # your NonAdminBackup   [verify]
EOF
oc --context site-a -n "$VMNS" get nonadminrestores.oadp.openshift.io d5 -w           # Ctrl-C when the restore Completed
oc --context site-a -n "$VMNS" get vm "$VM" -w                      # Running again
until alive >/dev/null 2>&1; do sleep 10; done; alive              # the VM booted, PostgreSQL recovered
verify                                                             # chain_ok = t
sql -c "SELECT stage FROM marks WHERE student = '$ME' AND stage LIKE 'd5-%' ORDER BY id;"
```

✅ `d5-in-backup` is back and `chain_ok = t`. `d5-after-backup` is **gone**: it was written after the backup, which is
exactly what a restore means. On Windows, your file in the VM ends at `d5-in-backup` too.

**Done on the cloud-native server?** Delete it, then go back to Demo 1 and start your migrated Windows VM:

```bash
oc --context site-a -n "$VMNS" delete vm pg-s01-centos10          # the CentOS server and its disks
```

**Next:** [Demo 1 §1.4](01-vmware-to-ove-cold.md#14-review-your-vm-on-openshift-then-power-it-on).

✍️ **Row 5.5:** the restore's duration, `chain_ok`, and which of your rows came back.

---

## Thank you

You protected a VM the way a platform team protects any application on OpenShift: one backup of the namespace, its
objects and its disks, restored in place. To go deeper:

- [OADP: backup and restore on OpenShift (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/index) [verify link]
- [Backing up and restoring virtual machines (OpenShift Virtualization 4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index) [verify link]
- [Velero, the upstream project](https://velero.io/docs/)

## Proof table: Demo 5

| Row | Stage | Duration | Size | Rows back | `chain_ok` |
|---|---|---|---|---|---|
| 5.3 | backup | | | — | |
| 5.5 | restore | | — | | |

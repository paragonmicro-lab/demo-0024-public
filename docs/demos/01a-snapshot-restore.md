---
title: "Demo 1a — Snapshot, destroy the database, restore from the snapshot (bonus)"
subtitle: "Snapshot your VM · write a row after it · drop the database · bring D: and L: back from the snapshot · read back"
version: 1.1.0
date: 2026-10-15
---

# Demo 1a: snapshot and restore (bonus)

> **Version 1.1.0 · draft · 2026-10-15** · class/demos/01a-snapshot-restore.md

## What you will learn

You take a snapshot of your running VM, destroy its database for real, then bring the data back by attaching the
snapshot's disks. The rows you read back show the **recovery point**: everything written before the snapshot
returns, and what you wrote after it does not.

**Before you start:** finish [Demo 1](01-vmware-to-ove-cold.md). Your VM runs on **site A**, and every command runs
in the site A console's **Web Terminal** (`>_`). `powershell` blocks run inside your Windows VM: **Virtualization →
VirtualMachines → your VM → Console**, then log in as `.\student-01` (your Windows login).

> ⚠ **The commands are an example** for `student-01-0024-012`, VM `win-pg-s01` at `10.101.23.101`. Change them to
> your class ID, your VM and its address before you run them.

> ⚠️ **Data and WAL go together.** PostgreSQL keeps its data on `D:` and its write-ahead log on `L:`. If you restore
> only `D:` and keep today's `L:`, PostgreSQL replays the newer WAL on start, including the `DROP DATABASE`, and you
> lose the database again. Restore **both** disks from the **same** snapshot. A VM snapshot with the guest agent
> freezes all disks at the same instant, so its `D:` and `L:` match.

<!-- 🎬 video slot 01a-snapshot-restore: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01a-snapshot-restore.jpg">
  <source src="../.images/01a-snapshot-restore.mp4" type="video/mp4">
  <a href="../.images/01a-snapshot-restore.mp4">🎬 Video: Demo 1a: snapshot and restore</a>
</video>
<sub>🎬 *Demo 1a: snapshot and restore: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1a.1 Inject, then snapshot

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'before-snapshot', 'this row must come back');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
```

**Console:** **Virtualization → VirtualMachines → `win-pg-s01` → Snapshots → Take snapshot**, name it
`win-pg-s01-snap`. Or from the Web Terminal:

```bash
oc apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata: { name: win-pg-s01-snap, namespace: student-01-0024-012 }
spec:
  source: { apiGroup: kubevirt.io, kind: VirtualMachine, name: win-pg-s01 }
EOF
oc -n student-01-0024-012 get vmsnapshot win-pg-s01-snap -w            # Ctrl-C when READYTOUSE is true
```

✅ The snapshot is **Ready**, with indications *Online* and *GuestAgent*: taken while running, with the disks frozen
together.

Now write a row **after** the snapshot. It exists only in the live database, so the restore must lose it:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'after-snapshot', 'this row must NOT come back');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
```

---

## 1a.2 Destroy the database

In your VM's console, in an **elevated** PowerShell (*Run as administrator*), stop the heartbeat writer so that it
doesn't recreate anything:

```powershell
Disable-ScheduledTask -TaskName bcdr-heartbeat | Out-Null
Stop-ScheduledTask    -TaskName bcdr-heartbeat
```

Then drop the database. The `bcdr` role owns it, and its password is the instructor's, so **🧑‍🏫 ask your
instructor** to run this with **your** VM's address:

```bash
# 🧑‍🏫 instructor, in their Web Terminal: the class pod in ovedr-app holds the bcdr owner's password
oc -n ovedr-app exec -i deploy/sql-client -- psql -h 10.101.23.101 -U bcdr -d postgres -c 'DROP DATABASE bcdr WITH (FORCE);'
```

Confirm the damage:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
```

✅ `FATAL: database "bcdr" does not exist`. That is the disaster.

---

## 1a.3 Take the damaged disks offline

In the VM's elevated PowerShell:

```powershell
Stop-Service postgresql-16
$bad = foreach ($l in 'D','L') { Get-Partition -DriveLetter $l | Get-Disk }
$bad | Format-Table Number, FriendlyName, SerialNumber, @{n='GB';e={[math]::Round($_.Size/1GB)}}
foreach ($l in 'D','L') { Get-Partition -DriveLetter $l | Remove-PartitionAccessPath -AccessPath "$($l):\" }
$bad | ForEach-Object { Set-Disk -Number $_.Number -IsOffline $true }
```

---

## 1a.4 Create volumes from the snapshot's `D:` and `L:`, and hot-plug them

Set the values once. Every command in this step reads them:

```bash
# your class ID: the project your VM, its disks and the snapshot live in
CLASS_ID=student-01-0024-012
# your VM, and the snapshot you took in 1a.1
VM=win-pg-s01
SNAP=$VM-snap
```

The snapshot holds one **VolumeSnapshot** per disk. List them, and note the **10Gi** (`D:`, data) and **5Gi** (`L:`,
WAL) rows:

```bash
CONTENT=$(oc -n $CLASS_ID get vmsnapshot $SNAP -o jsonpath='{.status.virtualMachineSnapshotContentName}')
oc -n $CLASS_ID get vmsnapshotcontent "$CONTENT" \
  -o jsonpath='{range .spec.volumeBackups[*]}{.persistentVolumeClaim.metadata.name}{"  "}{.persistentVolumeClaim.spec.resources.requests.storage}{"  "}{.volumeSnapshotName}{"\n"}{end}'
```

Copy four names from that list:

```bash
# the 10Gi row: your data disk (D:) and its VolumeSnapshot
DATA_PVC='<10Gi pvc>'
DATA_SNAP='<its volumesnapshot>'
# the 5Gi row: your WAL disk (L:) and its VolumeSnapshot, from the SAME snapshot so data and WAL match
WAL_PVC='<5Gi pvc>'
WAL_SNAP='<its volumesnapshot>'
```

Create a volume **from** each VolumeSnapshot, with the same class, mode and size as the original:

```bash
restore_pvc() {   # restore_pvc <original pvc> <volumesnapshot> <new name>
  oc -n $CLASS_ID get pvc "$1" -o json | jq --arg s "$2" --arg n "$3" '{
    apiVersion: "v1", kind: "PersistentVolumeClaim",
    metadata: {name: $n, namespace: .metadata.namespace},
    spec: {storageClassName: .spec.storageClassName, accessModes: .spec.accessModes,
           volumeMode: .spec.volumeMode, resources: .spec.resources,
           dataSource: {apiGroup: "snapshot.storage.k8s.io", kind: "VolumeSnapshot", name: $s}}}' | oc apply -f -
}
restore_pvc "$DATA_PVC" "$DATA_SNAP" $VM-restored-data
restore_pvc "$WAL_PVC"  "$WAL_SNAP"  $VM-restored-wal
oc -n $CLASS_ID get pvc $VM-restored-data $VM-restored-wal -w     # Bound (Ctrl-C)
```

Hot-plug both into the running VM. The serial (`data`, `wal`) is how Windows will tell them apart:

```bash
virtctl -n $CLASS_ID addvolume $VM --volume-name=$VM-restored-data --serial=data --persist
virtctl -n $CLASS_ID addvolume $VM --volume-name=$VM-restored-wal  --serial=wal  --persist
oc -n $CLASS_ID get vmi $VM -o jsonpath='{range .status.volumeStatus[*]}{.name}{"  "}{.phase}{"\n"}{end}'
```

✅ Both restored volumes show `Ready` **[verify: hot-plug on this cluster, and the `vioscsi` driver in the guest]**.

---

## 1a.5 Bring the restored disks online as `D:` and `L:`, start PostgreSQL

```powershell
Update-HostStorageCache
foreach ($m in @(@{Serial='data'; Letter='D'}, @{Serial='wal'; Letter='L'})) {
  $d = Get-Disk | Where-Object { $_.SerialNumber -eq $m.Serial }
  Set-Disk -Number $d.Number -IsOffline   $false
  Set-Disk -Number $d.Number -IsReadOnly  $false
  Get-Partition -DiskNumber $d.Number | Where-Object Type -eq 'Basic' | Set-Partition -NewDriveLetter $m.Letter
}
Get-Volume -DriveLetter D, L | Format-Table DriveLetter, FileSystemLabel, Size, HealthStatus
Start-Service postgresql-16
Enable-ScheduledTask -TaskName bcdr-heartbeat | Out-Null
Start-ScheduledTask  -TaskName bcdr-heartbeat
```

(The restored disks are copies of the originals, so Windows may first report a *signature collision*. Bringing the
copy online while the originals are offline gives it a new signature. That is expected.)

---

## 1a.6 Inject, and read back what came back

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'snapshot-restored', 'database back from the snapshot');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ The database exists again and `chain_ok = t`. The read-back runs `start-class` … `cold-migration-complete`,
`before-snapshot`, then `snapshot-restored`.
✅ **`after-snapshot` is gone.** It was written after the snapshot, so it was never in it. That is the **recovery
point**: a restore returns you to the moment of the snapshot, not to the moment of the disaster.

---

## The other way: restore the whole VM

Swapping disks by hand shows what a restore really is. In production you would restore the whole VM in one step,
at the cost of a restart: **VM → Snapshots → `win-pg-s01-snap` → Restore**. Or:

```bash
virtctl -n student-01-0024-012 stop win-pg-s01
oc apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata: { name: win-pg-s01-restore, namespace: student-01-0024-012 }
spec:
  target: { apiGroup: kubevirt.io, kind: VirtualMachine, name: win-pg-s01 }
  virtualMachineSnapshotName: win-pg-s01-snap
EOF
oc -n student-01-0024-012 get vmrestore win-pg-s01-restore -w          # COMPLETE true (Ctrl-C)
virtctl -n student-01-0024-012 start win-pg-s01
```

**Tidy up:** after the disk swap, your VM carries two extra disks and its original `D:` and `L:` are offline. Before
the next demo, either restore the whole VM (above), or stop the VM and remove the old volumes in **Configuration →
Storage** **[verify: BL-D11, a clean "post-bonus" state for demos 2-4]**.

**Back to:** [Demo 1](01-vmware-to-ove-cold.md) · **Next:** [Demo 2: warm migration with storage copy offload](02-vmware-to-ove-warm.md) (🚧 coming soon: read only).

## Thank you

- [OpenShift Virtualization: backup and restore (VM snapshots)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/backup-and-restore) [verify link]
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)

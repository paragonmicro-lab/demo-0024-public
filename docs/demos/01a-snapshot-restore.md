---
title: "Demo 1a — Snapshot, destroy the database, restore from the snapshot (bonus)"
subtitle: "Snapshot your VM · write a row after it · drop the database · bring D: and L: back from the snapshot · read back"
version: 1.2.0
date: 2026-10-17
---

# Demo 1a: snapshot and restore (bonus)

> **Version 1.2.0 · draft · 2026-10-17** · class/demos/01a-snapshot-restore.md

## What you will learn

You take a snapshot of your running VM, destroy its database for real, then bring the data back from the snapshot.
The rows you read back show the **recovery point**: everything written before the snapshot returns, and what you
wrote after it does not.

You do it twice, on two servers:

| Part | Server | When | Restore |
|---|---|---|---|
| **1** | your **cloud-native** CentOS server `pg-s01-centos10` ([Demo 1d](01d-vm-cloudnative-golden-image.md)) | while Demo 1's copy runs | the **whole VM** from the snapshot, in one step |
| **2** (optional) | your **migrated Windows** VM `win-pg-s01` | after [Demo 1](01-vmware-to-ove-cold.md) §1.5 | only the database's two disks, swapped in while it runs |

Every command runs in the site A console's **Web Terminal** (`>_`).

> ⚠ **The commands are an example** for `student-01-0024-012`: the cloud-native server `pg-s01-centos10` at
> `10.101.23.151`, the Windows VM `win-pg-s01` at `10.101.23.101`. Change them to your class ID, your VMs and their
> addresses before you run them.

## Part 1: your cloud-native server (while Demo 1's copy runs)

### 1a.1 Inject, then snapshot

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'before-snapshot', 'this row must come back');"
```

**Console:** **Virtualization → VirtualMachines → `pg-s01-centos10` → Snapshots → Take snapshot**, name it
`pg-s01-centos10-snap`. Or from the Web Terminal:

```bash
oc apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata: { name: pg-s01-centos10-snap, namespace: student-01-0024-012 }
spec:
  source: { apiGroup: kubevirt.io, kind: VirtualMachine, name: pg-s01-centos10 }
EOF
oc -n student-01-0024-012 get vmsnapshot pg-s01-centos10-snap -w       # Ctrl-C when READYTOUSE is true
```

✅ The snapshot is **Ready**, with indications *Online* and *GuestAgent*: all three disks frozen at the same instant.

Write a row **after** the snapshot. It exists only in the live database, so the restore must lose it:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'after-snapshot', 'this row must NOT come back');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
```

### 1a.2 Destroy the database

You own this server, so you do the damage yourself. In the VM's **Console**, logged in as `student-01`:

```bash
sudo -u postgres psql -c 'DROP DATABASE bcdr WITH (FORCE);'
```

Confirm it from the Web Terminal:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
```

✅ `FATAL: database "bcdr" does not exist`. That is the disaster.

### 1a.3 Restore the whole VM from the snapshot

A restore needs the VM stopped. **Console:** **Actions → Stop**, then **Snapshots → `pg-s01-centos10-snap` → ⋮ →
Restore**, then **Actions → Start**. Or:

```bash
virtctl -n student-01-0024-012 stop pg-s01-centos10
oc -n student-01-0024-012 get vm pg-s01-centos10 -w                    # Ctrl-C at Stopped
oc apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata: { name: pg-s01-centos10-restore, namespace: student-01-0024-012 }
spec:
  target: { apiGroup: kubevirt.io, kind: VirtualMachine, name: pg-s01-centos10 }
  virtualMachineSnapshotName: pg-s01-centos10-snap
EOF
oc -n student-01-0024-012 get vmrestore pg-s01-centos10-restore -w     # COMPLETE true (Ctrl-C)
virtctl -n student-01-0024-012 start pg-s01-centos10
```

All three disks come back as they were at the snapshot: the system, the data and the WAL together, so PostgreSQL
starts clean. **[verify: first run on a booking]**

### 1a.4 Inject, and read back what came back

Give it a minute to boot, then:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'snapshot-restored', 'whole VM back from the snapshot');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ `chain_ok = t`, and `before-snapshot` is back. ✅ **`after-snapshot` is gone**: it was written after the
snapshot. That is the **recovery point**: a restore takes you back to the snapshot, not to the moment of the disaster.

**Next:** [Demo 5: OADP backup and restore](05-oadp-vm-backup.md) on the same server, if your lab has OADP. Otherwise
delete the server and go back to [Demo 1 §1.4](01-vmware-to-ove-cold.md#14-review-your-vm-on-openshift-then-power-it-on).

---

## Part 2 (optional): your migrated Windows VM, restore only the database disks

After [Demo 1](01-vmware-to-ove-cold.md) §1.5, do the same on the server you moved, the precise way: swap in only the
snapshot's data and WAL disks while the VM keeps running. `powershell` blocks run inside your Windows VM:
**Virtualization → VirtualMachines → `win-pg-s01` → Console**, then log in as `.\student-01` (your Windows login).

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

### 1a.W1 Inject, then snapshot

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

### 1a.W2 Destroy the database

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

### 1a.W3 Take the damaged disks offline

In the VM's elevated PowerShell:

```powershell
Stop-Service postgresql-16
$bad = foreach ($l in 'D','L') { Get-Partition -DriveLetter $l | Get-Disk }
$bad | Format-Table Number, FriendlyName, SerialNumber, @{n='GB';e={[math]::Round($_.Size/1GB)}}
foreach ($l in 'D','L') { Get-Partition -DriveLetter $l | Remove-PartitionAccessPath -AccessPath "$($l):\" }
$bad | ForEach-Object { Set-Disk -Number $_.Number -IsOffline $true }
```

---

### 1a.W4 Create volumes from the snapshot's `D:` and `L:`, and hot-plug them

Set the values once. Every command in this step reads them:

```bash
# your class ID: the project your VM, its disks and the snapshot live in
CLASS_ID=student-01-0024-012
# your VM, and the snapshot you took in 1a.W1
VM=win-pg-s01
SNAP=$VM-snap
```

The snapshot holds one **VolumeSnapshot** per disk. List them. Sizes print in **bytes**: note the **1 GiB** row
(`1073741824`, `D:`, data) and the **256 MiB** row (`268435456`, `L:`, WAL). The 32 GiB row is `C:`. A disk can show a
little more than its round size; go by the order, largest to smallest: `C:`, `D:`, `L:`.

```bash
CONTENT=$(oc -n $CLASS_ID get vmsnapshot $SNAP -o jsonpath='{.status.virtualMachineSnapshotContentName}')
oc -n $CLASS_ID get vmsnapshotcontent "$CONTENT" \
  -o jsonpath='{range .spec.volumeBackups[*]}{.persistentVolumeClaim.metadata.name}{"  "}{.persistentVolumeClaim.spec.resources.requests.storage}{"  "}{.volumeSnapshotName}{"\n"}{end}'
```

Copy four names from that list:

```bash
# the 1 GiB row (1073741824): your data disk (D:) and its VolumeSnapshot
DATA_PVC='<1 GiB pvc>'
DATA_SNAP='<its volumesnapshot>'
# the 256 MiB row (268435456): your WAL disk (L:) and its VolumeSnapshot, from the SAME snapshot so data and WAL match
WAL_PVC='<256 MiB pvc>'
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

✅ Both restored volumes show `Ready`. (Proven on this cluster on 0024-015: snapshot, a volume from it, hot-plug into the
running VM, and the restored disk held the snapshot-time data.) **[verify: the `vioscsi` driver in the Windows guest]**

---

### 1a.W5 Bring the restored disks online as `D:` and `L:`, start PostgreSQL

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

### 1a.W6 Inject, and read back what came back

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

### The other way: restore the whole VM

Swapping disks by hand shows what a restore really is. In production you would restore the whole VM in one step,
at the cost of a restart (as you did in Part 1): **VM → Snapshots → `win-pg-s01-snap` → Restore**. Or:

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

**Back to:** [Demo 1](01-vmware-to-ove-cold.md) · **Next:** [Demo 5: OADP backup and restore](05-oadp-vm-backup.md), on your Windows VM this time.

## Thank you

- [OpenShift Virtualization: backup and restore (VM snapshots)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/backup-and-restore) [verify link]
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)

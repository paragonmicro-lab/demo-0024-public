---
title: "Demo 1b — Snapshot, destroy the database, restore from the snapshot"
subtitle: "Snapshot your cloud-native server · destroy its database · restore only the database's disks while it runs · then the whole VM · read back the recovery point"
version: 3.0.0
date: 2026-10-20
---

# Demo 1b: snapshot and restore

> **Version 3.0.0 · draft · 2026-10-20** · class/demos/01b-snapshot-restore.md

## What you will learn

You take **one** snapshot of your running server, destroy its database for real, and bring the data back from that
snapshot **twice**:

| Part | What you restore | The server | Time |
|---|---|---|---|
| **1** | only PostgreSQL's **data** and **WAL** disks | keeps running: you stop the database, not the OS | about 10 min |
| **2** | the **whole VM** | stopped, restored, started | about 5 min |

Each time, the rows you read back show the **recovery point**: everything written before the snapshot returns, and
what you wrote after it does not. Part 2 also puts your server back in its standard shape for Demo 1c.

You do it on your cloud-native CentOS server `pg-sNN-centos10` ([Demo 1a](01a-vm-cloudnative-golden-image.md)),
while Demo 1's copy runs. Your migrated Windows VM is the optional second walk-through, at the end.

<!-- 🎬 video slot 01b-snapshot-restore: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01b-snapshot-restore.jpg">
  <source src="../.images/01b-snapshot-restore.mp4" type="video/mp4">
  <a href="../.images/01b-snapshot-restore.mp4">🎬 Video: Demo 1b: snapshot and restore</a>
</video>
<sub>🎬 *Demo 1b: snapshot and restore: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

Every command runs in the site A console's **Web Terminal** (`>_`). 1b.0 sets your values once, so every command
after it is yours as it stands: no names or addresses to change.

---

## 1b.0 Get ready: your values, your server, your sign-in

**1. Your values.** Change the **one** line marked ✎ to your class ID, then run the block. It reads the rest from your
class terminal (your VLAN) and your class ID (your student number):

```bash
CLASS_ID=student-01-0024-012                                       # ✎ your class ID
NN=${CLASS_ID#student-}; NN=${NN%%-*}                              # your student number: 01
VLAN=$(oc -n $CLASS_ID exec deploy/sql-client -- printenv VM_IP | cut -d. -f3)
VM=pg-s$NN-centos10; IP=10.101.$VLAN.$((150 + 10#$NN))
echo "server $VM · address $IP · Linux login student-$NN · database login student_$NN"
```

✅ One line, e.g. `server pg-s01-centos10 · address 10.101.23.151 · Linux login student-01 · database login
student_01`. The Web Terminal forgets these when it closes: if a command says `error: resource name may not be
empty`, run this block again.

**2. Is your server ready?**

```bash
oc -n $CLASS_ID get vm $VM                                                         # STATUS Running, READY True
oc -n $CLASS_ID exec deploy/sql-client -- pg_isready -h $IP                        # … accepting connections
```

✅ Both, and you built it from the class Template in Demo 1a: go on to step 3. Anything else (no such VM, not
**Running**, `no response`, or you built it by hand in 1a's console twisty): reset it first.

<details markdown="1">
<summary>Reset: your server, rebuilt from the class Template (about 3 minutes)</summary>

This gives every student the same starting point: the class's server from Demo 1a, at **your** address, with
**your** Linux login, database login, password and SSH key. It **deletes** your server, its disks, and its 1b
snapshot and restored disks first, if there are any.

```bash
oc -n $CLASS_ID delete vm,secret -l app=$VM --wait=true                                  # your old server, if any
oc -n $CLASS_ID wait --for=delete vmi/$VM --timeout=5m 2>/dev/null; ssh-keygen -R $IP >/dev/null 2>&1   # gone, and forgotten
oc -n $CLASS_ID delete vmrestore,vmsnapshot $VM-restore $VM-snap --ignore-not-found      # its 1b snapshot, if any
oc -n $CLASS_ID delete pvc $VM-restored-pgdata $VM-restored-pgwal --ignore-not-found     # 1b's restored disks, if any
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -q -t ed25519 -N '' -C student-$NN -f ~/.ssh/id_ed25519
oc -n $CLASS_ID create secret generic student-$NN-ssh --from-file=key=$HOME/.ssh/id_ed25519.pub \
  --dry-run=client -o yaml | oc -n $CLASS_ID apply -f -                                   # your public key, for the new server
read -rsp 'Your class password: ' PW; echo
oc process -n openshift ovedr-centos10-postgres -p NAME=$VM -p VLAN=$VLAN -p IP=$IP -p OS_USER=student-$NN \
  -p DB_USER=student_$NN -p DB_PASSWORD="$PW" -p SSH_KEY_SECRET=student-$NN-ssh | oc -n $CLASS_ID apply -f -; unset PW
until ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new student-$NN@$IP \
  test -f /var/lib/cloud/instance/boot-finished 2>/dev/null; do sleep 10; done; echo "✅ $VM is ready"   # cloud-init finished
```

Paste it as it is: `'Your class password: '` is the prompt, and you type your password after it (nothing shows).
✅ `secret/pg-sNN-centos10-cloudinit created` and `virtualmachine.kubevirt.io/pg-sNN-centos10 created`, then, after
about two to three minutes, `✅ pg-sNN-centos10 is ready`: cloud-init has finished, so PostgreSQL, the class database
and your database login all exist. (PostgreSQL answers a minute before that, while your login is still being made:
wait for the ✅.)

The SSH key is this Web Terminal's (the block makes one if it has none), so `ssh` from here works by key; your
password works too. Your other work is untouched: your Windows VM and the Demo 1 copy carry on.

</details>

**3. Sign in to your server's database** (once per class-terminal pod):

```bash
oc exec -it -n $CLASS_ID deploy/sql-client -- env PGHOST=$IP dblogin
```

**4. Mark the start**, and read it back:

```bash
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT * FROM ovedr_mark('$CLASS_ID', '1b-started', 'snapshot task starting');"
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
```

✅ Your newest row is `1b-started` · `snapshot task starting`: the row the snapshot must keep.

---

## Part 1: restore only the database's disks, while the server runs

### 1b.1 Snapshot your server

**Console:** **Virtualization → VirtualMachines → `pg-sNN-centos10` → Snapshots → Take snapshot**, name it
`pg-sNN-centos10-snap`, **Save**. Or:

```bash
oc apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata: { name: $VM-snap, namespace: $CLASS_ID }
spec:
  source: { apiGroup: kubevirt.io, kind: VirtualMachine, name: $VM }
EOF
oc -n $CLASS_ID wait vmsnapshot $VM-snap --for=jsonpath='{.status.phase}'=Succeeded --timeout=5m
oc -n $CLASS_ID get vmsnapshot $VM-snap -o jsonpath='{.status.indications}{"\n"}'     # ["GuestAgent","Online"]
```

✅ **Succeeded**, with indications **Online** and **GuestAgent**: taken while the server ran, with the QEMU guest
agent freezing every filesystem at the same instant. That matters here: you will restore only two of its three
disks, and PostgreSQL's data and its WAL (the write-ahead log) must come from the same moment.

> ⚠️ **Data and WAL go together.** Restore only the data disk and keep today's WAL, and PostgreSQL replays the newer
> log on start, `DROP DATABASE` included: you lose the database again. Restore **both**, from the **same** snapshot.

> 🛠 **Failed, with *"guest-fsfreeze-freeze': failed to open /var/lib/pgsql/wal: Permission denied"*?** Your server
> was built before the Template let the guest agent reach PostgreSQL's disks. Reset it (1b.0, step 2), or allow it
> and take the snapshot again:
> `ssh student-$NN@$IP sudo setsebool -P virt_qemu_ga_read_nonsecurity_files on; oc -n $CLASS_ID delete vmsnapshot $VM-snap`

### 1b.2 The disaster

Write a row **after** the snapshot, and read it back. It exists only in the live database, so no restore can bring
it back:

```bash
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT * FROM ovedr_mark('$CLASS_ID', 'after-snapshot', 'all your data is destroyed 😈 mwahaha');"
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
```

Then make it true. You own this server, so you do the damage yourself, over SSH from the Web Terminal (or in the
VM's **Console** tab, logged in as `student-NN`, with only the part in quotes):

```bash
ssh -o StrictHostKeyChecking=accept-new student-$NN@$IP 'sudo -u postgres psql -X -c "DROP DATABASE bcdr WITH (FORCE);"'
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
```

✅ `DROP DATABASE`, then `FATAL: database "bcdr" does not exist`. That is the disaster.

### 1b.3 Stop PostgreSQL, and take the damaged disks out of use

**a) Stop PostgreSQL** (and the heartbeat that writes to it). **b) Unmount** its two disks, and relabel them so
nothing mounts them by mistake: the disks you restore carry the same labels (`pgdata`, `pgwal`), which is how
`/etc/fstab` finds them.

```bash
ssh student-$NN@$IP 'sudo systemctl stop ovedr-heartbeat postgresql; systemctl is-active postgresql
  sudo umount /var/lib/pgsql/wal /var/lib/pgsql/data
  sudo xfs_admin -L bad-pgdata /dev/disk/by-id/virtio-pgdata; sudo xfs_admin -L bad-pgwal /dev/disk/by-id/virtio-pgwal'
```

✅ `inactive`, then `new label = "bad-pgdata"` and `new label = "bad-pgwal"`. The OS is still up: only the
database stopped.

### 1b.4 Restore the two disks from the snapshot, into the running server

**c) Restore.** Your snapshot holds one **VolumeSnapshot** per disk. Make a new volume from each of the two you
need, then hot-plug both into the running VM:

```bash
CONTENT=$(oc -n $CLASS_ID get vmsnapshot $VM-snap -o jsonpath='{.status.virtualMachineSnapshotContentName}')
for d in pgdata pgwal; do        # a volume from the snapshot of each disk, the same size and class as the original
  VS=$(oc -n $CLASS_ID get vmsnapshotcontent $CONTENT -o jsonpath="{.spec.volumeBackups[?(@.volumeName==\"$d\")].volumeSnapshotName}")
  oc -n $CLASS_ID get pvc $VM-$d -o json | jq --arg s "$VS" --arg n "$VM-restored-$d" '{apiVersion: "v1",
    kind: "PersistentVolumeClaim", metadata: {name: $n, namespace: .metadata.namespace},
    spec: {storageClassName: .spec.storageClassName, accessModes: .spec.accessModes, volumeMode: .spec.volumeMode,
           resources: .spec.resources, dataSource: {apiGroup: "snapshot.storage.k8s.io", kind: "VolumeSnapshot", name: $s}}}' \
    | oc apply -f -
done
oc -n $CLASS_ID wait pvc $VM-restored-pgdata $VM-restored-pgwal --for=jsonpath='{.status.phase}'=Bound --timeout=2m
virtctl -n $CLASS_ID addvolume $VM --volume-name=$VM-restored-pgdata --serial=pgdata-r --persist
virtctl -n $CLASS_ID addvolume $VM --volume-name=$VM-restored-pgwal --serial=pgwal-r --persist
```

**Console**, the same by hand: **Storage → VolumeSnapshots** in your project, the two named
`vmsnapshot-…-volume-pgdata` and `vmsnapshot-…-volume-pgwal` → **⋮ → Restore as new PVC**, named
`pg-sNN-centos10-restored-pgdata` and `pg-sNN-centos10-restored-pgwal`. Then, for each, **VM → Configuration →
Storage → Add disk → Use an existing PVC**, with **hot plug** on. **[verify: the console's field names]**

Wait for both to be **in** the VM (they go *Pending*, *AttachedToNode*, then *Ready*, in about half a minute):

```bash
until [ "$(oc -n $CLASS_ID get vmi $VM -o jsonpath="{.status.volumeStatus[?(@.name==\"$VM-restored-pgdata\")].phase} {.status.volumeStatus[?(@.name==\"$VM-restored-pgwal\")].phase}")" = "Ready Ready" ]
do sleep 5; done; echo "✅ both restored disks are in $VM"
```

✅ `✅ both restored disks are in pg-sNN-centos10`. Don't mount before it: the disks aren't there yet.

### 1b.5 Mount the restored disks, start PostgreSQL

**d) Mount** them: `/etc/fstab` mounts by label, and the only `pgdata` and `pgwal` now are the restored disks.
**e) Start** PostgreSQL and the heartbeat:

```bash
ssh student-$NN@$IP 'sudo udevadm settle; lsblk -o NAME,SIZE,SERIAL,LABEL,MOUNTPOINT
  sudo mount /var/lib/pgsql/data && sudo mount /var/lib/pgsql/wal && findmnt -o SOURCE,TARGET /var/lib/pgsql/data /var/lib/pgsql/wal
  sudo systemctl start postgresql ovedr-heartbeat; systemctl is-active postgresql ovedr-heartbeat'
```

✅ `lsblk` shows the restored disks as `sda` (serial `pgdata-r`, label `pgdata`) and `sdb` (`pgwal-r`, `pgwal`),
beside the damaged `vdc` (`bad-pgdata`) and `vdd` (`bad-pgwal`). Then `/dev/sda /var/lib/pgsql/data`,
`/dev/sdb /var/lib/pgsql/wal`, and `active`, `active`.

### 1b.6 Read back what came back

**f) Read back:**

```bash
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT * FROM ovedr_mark('$CLASS_ID', 'snapshot-restored', 'data and WAL disks back from the snapshot, the OS never stopped');"
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT * FROM ovedr_verify('$CLASS_ID');"
```

✅ The database is back, and the read-back ends at **`1b-started` · `snapshot task starting`**.
✅ **`after-snapshot` · `all your data is destroyed 😈 mwahaha` is gone**: it was written after the snapshot. That
is the **recovery point**: a restore takes you back to the moment of the snapshot, not to the moment of the disaster.
✅ `chain_ok = t`, after your `snapshot-restored` row.

---

## Part 2: restore the whole VM, from the same snapshot

The one-step way, at the cost of a restart: every disk, the system disk too, goes back to the snapshot. It also
removes the two disks you hot-plugged, so your server ends in its standard shape for Demo 1c.

### 1b.7 Stop, restore, start

**Console:** **Actions → Stop**; when it is **Stopped**, **Snapshots → `pg-sNN-centos10-snap` → ⋮ → Restore** →
**Restore**; then **Actions → Start**. Or:

```bash
virtctl -n $CLASS_ID stop $VM
oc -n $CLASS_ID wait vm $VM --for=jsonpath='{.status.printableStatus}'=Stopped --timeout=3m
oc apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata: { name: $VM-restore, namespace: $CLASS_ID }
spec:
  target: { apiGroup: kubevirt.io, kind: VirtualMachine, name: $VM }
  virtualMachineSnapshotName: $VM-snap
EOF
oc -n $CLASS_ID wait vmrestore $VM-restore --for=jsonpath='{.status.complete}'=true --timeout=5m
virtctl -n $CLASS_ID start $VM
until oc -n $CLASS_ID exec deploy/sql-client -- pg_isready -q -h $IP 2>/dev/null; do sleep 5; done; echo "✅ $VM is back"
```

✅ About a minute and a half from stop to `✅ pg-sNN-centos10 is back`.

### 1b.8 Read back, and tidy up

```bash
oc exec -n $CLASS_ID deploy/sql-client -- sql -h $IP -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
ssh student-$NN@$IP 'lsblk -o NAME,SIZE,SERIAL,LABEL,MOUNTPOINT'
```

✅ The read-back ends at **`1b-started`** again: Part 1's `snapshot-restored` row is gone too, because it was
written after the snapshot. ✅ `lsblk` shows your server's own disks again, `vdc` (`pgdata`) and `vdd` (`pgwal`),
mounted; the hot-plugged ones are gone.

The two volumes you restored in Part 1 are no longer attached. Delete them (the snapshot stays):

```bash
oc -n $CLASS_ID delete pvc $VM-restored-pgdata $VM-restored-pgwal
```

**That is the end of Demo 1b.** Keep your server running. **Next:** [Demo 1c: live migration and storage
migration](01c-live-and-storage-migration.md), on the same server.

---

## Second walk-through: your Windows VM (optional)

<details markdown="1">
<summary>The same on your migrated Windows VM, after Demo 1 §1.5: restore only the database disks</summary>

After [Demo 1](01-vmware-to-ove-cold.md) §1.5, do the same on the server you moved, the precise way: swap in only the
snapshot's data and WAL disks while the VM keeps running. `powershell` blocks run inside your Windows VM:
**Virtualization → VirtualMachines → `win-pg-s01` → Console**, then log in as `.\student-01` (your Windows login).

> ⚠️ **Data and WAL go together.** PostgreSQL keeps its data on `D:` and its write-ahead log on `L:`. If you restore
> only `D:` and keep today's `L:`, PostgreSQL replays the newer WAL on start, including the `DROP DATABASE`, and you
> lose the database again. Restore **both** disks from the **same** snapshot. A VM snapshot with the guest agent
> freezes all disks at the same instant, so its `D:` and `L:` match.

### 1b.W1 Inject, then snapshot

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

### 1b.W2 Destroy the database

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

### 1b.W3 Take the damaged disks offline

In the VM's elevated PowerShell:

```powershell
Stop-Service postgresql-16
$bad = foreach ($l in 'D','L') { Get-Partition -DriveLetter $l | Get-Disk }
$bad | Format-Table Number, FriendlyName, SerialNumber, @{n='GB';e={[math]::Round($_.Size/1GB)}}
foreach ($l in 'D','L') { Get-Partition -DriveLetter $l | Remove-PartitionAccessPath -AccessPath "$($l):\" }
$bad | ForEach-Object { Set-Disk -Number $_.Number -IsOffline $true }
```

---

### 1b.W4 Create volumes from the snapshot's `D:` and `L:`, and hot-plug them

Set the values once. Every command in this step reads them:

```bash
# your class ID: the project your VM, its disks and the snapshot live in
CLASS_ID=student-01-0024-012
# your VM, and the snapshot you took in 1b.W1
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

### 1b.W5 Bring the restored disks online as `D:` and `L:`, start PostgreSQL

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

### 1b.W6 Inject, and read back what came back

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

**Back to:** [Demo 1](01-vmware-to-ove-cold.md) · **Next on Windows:** [Demo 1c: live migration and storage migration](01c-live-and-storage-migration.md).

</details>

## Thank you

- [OpenShift Virtualization: backup and restore (VM snapshots)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/backup-and-restore) [verify link]
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)

---
title: "Demo 1c — Live migration and storage migration"
subtitle: "Move the running VM to another node · move a running disk to another storage class · no reboot · read back"
version: 2.0.0
date: 2026-10-15
---

# Demo 1c: live migration and storage migration

> **Version 2.0.0 · draft · 2026-10-19** · class/demos/01c-live-and-storage-migration.md

## What you will learn

Two moves on a **running** VM, with no reboot: a **live migration** moves its memory and CPU to another node, and
a **storage migration** moves one of its disks to another storage class (ODF → NFS) while Linux and PostgreSQL keep
running. You write a row before and after each move and read them back.

**Before you start:** your cloud-native server `pg-s01-centos10` runs on **site A** ([Demo 1a](01a-vm-cloudnative-golden-image.md),
then [Demo 1b](01b-snapshot-restore.md)), while Demo 1's copy runs. Every command runs in the site A console's
**Web Terminal** (`>_`).

> ⚠ **The commands are an example** for `student-01-0024-012`, the cloud-native server `pg-s01-centos10` at
> `10.101.23.151`. Change them to your class ID, your server and its address before you run them.
>
> **Second walk-through on Windows:** use `win-pg-s01` and `10.101.23.101`, and move its **1 GiB** data disk (`D:`)
> instead of the 10 GiB one.

<!-- 🎬 video slot 01c-live-and-storage-migration: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01c-live-and-storage-migration.jpg">
  <source src="../.images/01c-live-and-storage-migration.mp4" type="video/mp4">
  <a href="../.images/01c-live-and-storage-migration.mp4">🎬 Video: Demo 1c: live migration and storage migration</a>
</video>
<sub>🎬 *Demo 1c: live migration and storage migration: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1c.1 Inject, and check the VM can live-migrate

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'before-live-migration', 'running on one node');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
oc -n student-01-0024-012 get vmi pg-s01-centos10 -o jsonpath='{.status.nodeName}  LiveMigratable={.status.conditions[?(@.type=="LiveMigratable")].status}{"\n"}'
```

✅ `LiveMigratable=True`. ✍️ Note the node it runs on now. A live migration needs the disks reachable from **both**
nodes at once (ReadWriteMany): ODF's `ocs-storagecluster-ceph-rbd-virtualization` is. If it says `False`, the
condition's message names the reason, usually a mounted CD.

---

## 1c.2 Live-migrate to another node

**Console:** **Virtualization → VirtualMachines → `pg-s01-centos10` → Actions → Migrate → Compute**. Or:

```bash
virtctl -n student-01-0024-012 migrate pg-s01-centos10
oc -n student-01-0024-012 get vmim -w                                 # … Succeeded (Ctrl-C)
oc -n student-01-0024-012 get vmi pg-s01-centos10 -o jsonpath='{.status.nodeName}{"\n"}'
```

✅ The VM runs on a **different** node, and it never stopped: the console's VM **Overview** shows the new node and no
restart.

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'after-live-migration', 'running on another node');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
```

---

## 1c.3 Storage migration: move the data disk from ODF to NFS, while it runs

Your server's disks are on ODF (`ocs-storagecluster-ceph-rbd-virtualization`). Move its 10 GiB **data** disk
(PostgreSQL's data, `/var/lib/pgsql`) to `nfs-csi`, with the server running. OpenShift Virtualization copies the disk in the background, then switches the VM to
the new copy with a live migration.

Proven on this cluster (0024-015, 2026-10-16) with a Windows VM's 1 GiB disk: ODF (block) to `nfs-csi`
(filesystem) in 46 seconds, the VM never restarted, and its data was intact. Your 10 GiB disk takes a few minutes
**[verify: the 10 GiB CentOS data disk's time]**. **[verify: whether the console's **Actions → Migrate →
Storage** offers the same]**

First check the VM allows it:

```bash
oc -n student-01-0024-012 get vm pg-s01-centos10 -o jsonpath='StorageLiveMigratable={.status.conditions[?(@.type=="StorageLiveMigratable")].status}{"\n"}'
```

✅ `StorageLiveMigratable=True`.

Find the data disk's volume (the 10 GiB one; after a Demo 1b restore its name starts `restore-`):

```bash
oc -n student-01-0024-012 get vm pg-s01-centos10 -o json | jq -r '.spec.template.spec.volumes[] | "\(.name)  \(.persistentVolumeClaim.claimName // .dataVolume.name)"'
oc -n student-01-0024-012 get pvc -o custom-columns=NAME:.metadata.name,SIZE:.spec.resources.requests.storage,CLASS:.spec.storageClassName
```

Set the one value this step needs, from the list above:

```bash
# the data disk's current volume (the 10 GiB one on ODF): the disk you are moving
OLD='<the 10 GiB pvc from the list above>'
```

Create an empty 10Gi target volume on `nfs-csi`, the same size as the disk you move:

```bash
oc apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata: { name: pg-s01-centos10-pgdata-nfs, namespace: student-01-0024-012 }
spec:
  source: { blank: {} }
  storage:
    storageClassName: nfs-csi
    accessModes: [ ReadWriteMany ]
    resources: { requests: { storage: 10Gi } }
EOF
oc -n student-01-0024-012 get dv pg-s01-centos10-pgdata-nfs -w                # Succeeded (Ctrl-C)
```

Point the VM's data disk at the new volume, and tell it to **migrate** the data (not just swap the disk):

```bash
oc -n student-01-0024-012 get vm pg-s01-centos10 -o json | jq --arg old "$OLD" '
  .spec.updateVolumesStrategy = "Migration"
  | .spec.template.spec.volumes |= map(
      if ((.persistentVolumeClaim.claimName // .dataVolume.name) == $old)
      then {name: .name, dataVolume: {name: "pg-s01-centos10-pgdata-nfs"}} else . end)
  | if .spec.dataVolumeTemplates then .spec.dataVolumeTemplates |= map(select(.metadata.name != $old)) else . end' \
  | oc replace -f -
oc -n student-01-0024-012 get vmim -w                                 # a migration runs and Succeeds (Ctrl-C)
oc -n student-01-0024-012 get vmi pg-s01-centos10 -o jsonpath='{range .status.volumeStatus[*]}{.name}  {.persistentVolumeClaimInfo.claimName}{"\n"}{end}'
```

✅ The data disk now reads `pg-s01-centos10-pgdata-nfs` (`nfs-csi`), and the VM never restarted. In the console: **VM →
Configuration → Storage** shows the data disk on `nfs-csi`.

---

## 1c.4 Inject, and read back

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'after-storage-migration', 'data disk on NFS');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ After Demo 1's rows: `before-live-migration`, `after-live-migration` and `after-storage-migration`, in order, with
`chain_ok = t`. PostgreSQL kept running through both moves, so nothing between them is missing.

**Next:** [Demo 1d: cross-cluster migration](01d-ocp-vm-cross-cluster-migration.md) (**BCDR class only**: it needs a
second OpenShift cluster). In the *VM Hosting Alternative* class, go straight to
[Demo 1e: OADP backup and restore](01e-oadp-vm-backup.md). · **Back to:** [Demo 1](01-vmware-to-ove-cold.md)

## Thank you

- [OpenShift Virtualization: live migration](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/live-migration) [verify link]
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)

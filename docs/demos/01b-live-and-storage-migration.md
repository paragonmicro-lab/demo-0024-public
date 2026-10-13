---
title: "Demo 1b — Live migration and storage migration (bonus)"
subtitle: "Move the running VM to another node · move a running disk to another storage class · no reboot · read back"
version: 1.0.0
date: 2026-10-13
---

# Demo 1b: live migration and storage migration (bonus)

> **Version 1.0.0 · draft · 2026-10-13** · class/demos/01b-live-and-storage-migration.md

## What you will learn

Two moves on a **running** VM, with no reboot: a **live migration** moves its memory and CPU to another node, and
a **storage migration** moves one of its disks to another storage class (ODF → NFS) while Windows and PostgreSQL keep
running. You write a row before and after each move and read them back.

**Before you start:** finish [Demo 1](01-vmware-to-ove-cold.md). Your VM runs on **site A**, and every command runs
in the site A console's **Web Terminal** (`>_`).

> ⚠ **The commands are an example** for `student-01-0024-012`, VM `win-pg-s01` at `10.101.23.101`. Change them to
> your class ID, your VM and its address before you run them.

<!-- 🎬 video slot 01b-live-and-storage-migration: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01b-live-and-storage-migration.jpg">
  <source src="../.images/01b-live-and-storage-migration.mp4" type="video/mp4">
  <a href="../.images/01b-live-and-storage-migration.mp4">🎬 Video: Demo 1b: live migration and storage migration</a>
</video>
<sub>🎬 *Demo 1b: live migration and storage migration: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1b.1 Inject, and check the VM can live-migrate

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'before-live-migration', 'running on one node');"
oc -n student-01-0024-012 get vmi win-pg-s01 -o jsonpath='{.status.nodeName}  LiveMigratable={.status.conditions[?(@.type=="LiveMigratable")].status}{"\n"}'
```

✅ `LiveMigratable=True`. ✍️ Note the node it runs on now. A live migration needs the disks reachable from **both**
nodes at once (ReadWriteMany): ODF's `ocs-storagecluster-ceph-rbd-virtualization` is. If it says `False`, the
condition's message names the reason, usually a mounted CD.

---

## 1b.2 Live-migrate to another node

**Console:** **Virtualization → VirtualMachines → `win-pg-s01` → Actions → Migrate → Compute**. Or:

```bash
virtctl -n student-01-0024-012 migrate win-pg-s01
oc -n student-01-0024-012 get vmim -w                                 # … Succeeded (Ctrl-C)
oc -n student-01-0024-012 get vmi win-pg-s01 -o jsonpath='{.status.nodeName}{"\n"}'
```

✅ The VM runs on a **different** node, and it never stopped: the console's VM **Overview** shows the new node and no
restart.

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'after-live-migration', 'running on another node');"
```

---

## 1b.3 Storage migration: move the data disk from ODF to NFS, while it runs [verify]

Your VM's disks are on ODF (`ocs-storagecluster-ceph-rbd-virtualization`). Move the 10 GB **data** disk (`D:`) to
`nfs-csi`, with the VM running. OpenShift Virtualization copies the disk in the background, then switches the VM to
the new copy with a live migration.

**[verify: storage live migration on OpenShift Virtualization 4.19 (`spec.updateVolumesStrategy: Migration`,
`vmRolloutStrategy: LiveUpdate` is set on this cluster); ODF block → NFS filesystem; whether the console's
**Actions → Migrate → Storage** offers the same]**

Find the data disk's volume (the 10Gi one):

```bash
oc -n student-01-0024-012 get vm win-pg-s01 -o json | jq -r '.spec.template.spec.volumes[] | "\(.name)  \(.persistentVolumeClaim.claimName // .dataVolume.name)"'
oc -n student-01-0024-012 get pvc -o custom-columns=NAME:.metadata.name,SIZE:.spec.resources.requests.storage,CLASS:.spec.storageClassName
```

Set the one value this step needs, from the list above:

```bash
# the data disk's current volume (the 10Gi one on ODF): the disk you are moving
OLD='<the 10Gi pvc from the list above>'
```

Create an empty 10Gi target volume on `nfs-csi`:

```bash
oc apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata: { name: win-pg-s01-data-nfs, namespace: student-01-0024-012 }
spec:
  source: { blank: {} }
  storage:
    storageClassName: nfs-csi
    accessModes: [ ReadWriteMany ]
    resources: { requests: { storage: 10Gi } }
EOF
oc -n student-01-0024-012 get dv win-pg-s01-data-nfs -w                # Succeeded (Ctrl-C)
```

Point the VM's data disk at the new volume, and tell it to **migrate** the data (not just swap the disk):

```bash
oc -n student-01-0024-012 get vm win-pg-s01 -o json | jq --arg old "$OLD" '
  .spec.updateVolumesStrategy = "Migration"
  | .spec.template.spec.volumes |= map(
      if ((.persistentVolumeClaim.claimName // .dataVolume.name) == $old)
      then {name: .name, dataVolume: {name: "win-pg-s01-data-nfs"}} else . end)' | oc replace -f -
oc -n student-01-0024-012 get vmim -w                                 # a migration runs and Succeeds (Ctrl-C)
oc -n student-01-0024-012 get vmi win-pg-s01 -o jsonpath='{range .status.volumeStatus[*]}{.name}  {.persistentVolumeClaimInfo.claimName}{"\n"}{end}'
```

✅ The data disk now reads `win-pg-s01-data-nfs` (`nfs-csi`), and the VM never restarted. In the console: **VM →
Configuration → Storage** shows the data disk on `nfs-csi`.

---

## 1b.4 Inject, and read back

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'after-storage-migration', 'data disk on NFS');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ After Demo 1's rows: `before-live-migration`, `after-live-migration` and `after-storage-migration`, in order, with
`chain_ok = t`. PostgreSQL kept running through both moves, so nothing between them is missing.

**Back to:** [Demo 1](01-vmware-to-ove-cold.md) · **Next:** [Demo 2: warm migration](02-vmware-to-ove-warm.md).

## Thank you

- [OpenShift Virtualization: live migration](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/live-migration) [verify link]
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)

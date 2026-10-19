---
title: "Demo 1d — Cross-cluster migration: OpenShift site A to site B, with its storage (BCDR class)"
subtitle: "ACM links the clusters · MTV moves your cloud-native server from site A to site B, with its disks · inject and read back"
version: 2.0.0
date: 2026-10-15
---

# Demo 1d: cross-cluster migration, site A to site B (BCDR class)

> **Version 2.0.0 · draft · 2026-10-19** · class/demos/01d-ocp-vm-cross-cluster-migration.md

## What you will learn

Your cloud-native server `pg-s01-centos10` ([Demo 1a](01a-vm-cloudnative-golden-image.md)) runs on **site A**. Now you
move it to a **second OpenShift cluster**, site B. Advanced
Cluster Management (ACM) links the two clusters into one fleet, and MTV moves the VM between them. The two sites use
their own storage, so the move is also a **storage migration**: every disk is copied into site B's storage class.
As in every demo, you write a row before and after the move and read them all back.

> 🏢 **BCDR class only** (*VMware Migration & BCDR*, demo-0024): this demo needs two OpenShift clusters. The *VM
> Hosting Alternative* class (demo-0002) has one, so skip to [Demo 1e](01e-oadp-vm-backup.md).

**Before you start:** your cloud-native server runs on **site A** (after Demos 1a – 1c), while Demo 1's copy runs.
You are signed in to **both** consoles. Commands
run in each console's **Web Terminal** (`>_`); each step says which site.

> ⚠ **The commands are an example. Change these values to yours before you run them:**
>
> | In the example | Is the example's | Replace with yours |
> |---|---|---|
> | `student-01-0024-012` | class ID and project, the same on both sites | your class ID |
> | `pg-s01-centos10` | your cloud-native server (Windows second walk-through: `win-pg-s01`) | your server |
> | `10.101.23.151` | its address, the SQL server: it keeps it on site B too (Windows: `10.101.23.101`) | its address |
> | `ocp-lz1` → `ocp-lz2` | site A → site B (provider `ovedr-ocp-lz1`, on site B) | your booking's `A` and `B` |

<!-- 🎬 video slot 01d-ocp-vm-cross-cluster-migration: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01d-ocp-vm-cross-cluster-migration.jpg">
  <source src="../.images/01d-ocp-vm-cross-cluster-migration.mp4" type="video/mp4">
  <a href="../.images/01d-ocp-vm-cross-cluster-migration.mp4">🎬 Video: Demo 1d: cross-cluster migration</a>
</video>
<sub>🎬 *Demo 1d: cross-cluster migration: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1d.0 Inject: the demo has started (site A)

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'cross-cluster-started', 'Demo 1d: running on site A');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
```

---

## 1d.1 ACM: the two clusters, linked (site B)

Site B runs the ACM **hub**, and site A is imported into it as a **managed cluster**.

**Console (site B):** switch the perspective at the top left to **All Clusters** → **Infrastructure → Clusters**.

✅ `ocp-lz1` (site A) shows **Ready**, next to `local-cluster` (site B itself).

Site A's own console shows **no clusters** in this view, and that's by design: a cluster belongs to one hub only,
and site A's belongs to site B's. The hub supervises; the sites aren't peers ([Demo 4](04-odf-regional-dr.md#who-is-in-charge-the-hub-supervises-the-sites-are-not-peers)).

```bash
oc get managedclusters                              # ocp-lz1  JOINED True  AVAILABLE True
```

**Your VM, seen from the hub:** **All Clusters → Infrastructure → Virtual machines** **[verify: ACM 2.13 fleet
virtualization view on the hub]**. `pg-s01-centos10` shows on cluster `ocp-lz1`, in your project, **Running**.

ACM shows you where the VM is; **MTV does the move.**

---

## 1d.2 MTV: the link to site A, and the maps (site B)

**Console (site B):** **Migration for Virtualization → Providers → `ovedr-ocp-lz1`**.

✅ **Ready** and **Connected**: site B's MTV reads site A's VMs.

| Map (on site B) | What it does for your VM |
|---|---|
| Network map `ovedr-ocp-lz1-net` | `default/vmnet23` on site A → `default/vmnet23` on site B: same VLAN, so your VM **keeps its address** |
| Storage map `ovedr-ocp-lz1-storage` | site A's `ocs-storagecluster-ceph-rbd-virtualization` (ODF) **and** its `nfs-csi` (where Demo 1c moved your data disk) → site B's storage class: the **storage migration** |

```bash
oc -n openshift-mtv get provider ovedr-ocp-lz1                                     # READY True · CONNECTED True
oc -n openshift-mtv get storagemap ovedr-ocp-lz1-storage -o jsonpath='{.spec.map}{"\n"}'   # ODF → nfs-csi
```

MTV doesn't move the disks themselves: for each one it creates a new volume on site B in `nfs-csi` and fills it by
**reading the disk on site A over the network**. Site A's disks are left untouched.

---

## 1d.3 Inject, then stop your VM (site A)

Write the "start" row **before** the VM stops, while its database still answers:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'start-cross-cluster-migration', 'last row on site A');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
```

**Console (site A):** **Virtualization → VirtualMachines → `pg-s01-centos10` → Actions → Stop**.

✅ The VM shows **Stopped**.

**Unplug its cloud-init disk.** MTV copies the VM and its disks, not the cloud-init Secret it booted from (Demo 1a).
cloud-init only did its work on the first boot: the server's address, logins and PostgreSQL are now on its own disks.
So remove the disk while the VM is stopped, and it starts on site B without the Secret:

```bash
oc -n student-01-0024-012 get vm pg-s01-centos10 -o json \
  | jq 'del(.spec.template.spec.volumes[] | select(.cloudInitNoCloud)) | del(.spec.template.spec.domain.devices.disks[] | select(.name == "cloudinitdisk"))' \
  | oc replace -f -
```

✅ `virtualmachine.kubevirt.io/pg-s01-centos10 replaced`. (Windows second walk-through: skip this; a migrated
Windows VM has no cloud-init disk.) **[verify: MTV 2.11 OpenShift → OpenShift, a CentOS VM without its cloud-init
disk boots at its static address on site B]**

---

## 1d.4 Create your plan, and start it (site B)

This plan isn't staged: you create it, then start it.

**Console (site B):** **Migration for Virtualization → Migration plans → Create plan**:

| Field | Value |
|---|---|
| Plan name | `d1d-pg-s01-centos10` (`d1d-<your VM>`) |
| Plan project | `openshift-mtv` |
| Source provider | `ovedr-ocp-lz1` |
| Target provider | `host` (site B) |
| Target project | `student-01-0024-012` |
| Virtual machines | **only** your VM, `pg-s01-centos10` (in project `student-01-0024-012`) |
| Network map | `ovedr-ocp-lz1-net` (use the existing map) |
| Storage map | `ovedr-ocp-lz1-storage` (use the existing map) |
| Migration type | **Cold** |

**Create plan**. When it shows **Ready**, click **Start** → confirm. Expand your VM on the plan's page to watch
**DiskTransfer**: three disks copied from site A into site B's storage class.

```bash
oc -n openshift-mtv get plan d1d-pg-s01-centos10 -w      # Ready → Executing → Succeeded (Ctrl-C)
```

<details markdown="1">
<summary>Or: create and start the plan from the CLI (site B Web Terminal)</summary>

This builds the **same** plan under its own name, `d1d-cli-pg-s01-centos10`, so it never touches one you made in the
console. Use **one** of them: a VM can be in only one running plan.

Set the values once. Every command below reads them:

```bash
# your class ID: your project on both sites, and where MTV creates the VM on site B
CLASS_ID=student-01-0024-012
# your VM's name on site A (MTV keeps the name on site B)
VM=pg-s01-centos10
# the provider on site B that reads site A ("ovedr-ocp-" + site A's slot), and its two maps
SRC=ovedr-ocp-lz1
# a NEW plan name, so this never overwrites a plan you created in the console
PLAN=d1d-cli-$VM
```

Create the plan:

```bash
oc apply -f - <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata: { name: $PLAN, namespace: openshift-mtv }
spec:
  warm: false
  targetNamespace: $CLASS_ID
  provider:
    source:      { name: $SRC, namespace: openshift-mtv }
    destination: { name: host, namespace: openshift-mtv }
  map:
    network: { name: $SRC-net,     namespace: openshift-mtv }
    storage: { name: $SRC-storage, namespace: openshift-mtv }
  vms:
    - { name: $VM, namespace: $CLASS_ID }
EOF
oc -n openshift-mtv get plan $PLAN -w                # READY True (Ctrl-C)
```

Start it: a **Migration** is what the console's Start button creates.

```bash
oc apply -f - <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata: { name: $PLAN-run1, namespace: openshift-mtv }
spec: { plan: { name: $PLAN, namespace: openshift-mtv } }
EOF
oc -n openshift-mtv get plan $PLAN -w                # Executing → Succeeded (Ctrl-C)
```

If the plan reports a **critical** condition, read it before you start: usually a network or storage class on site
A that the maps don't cover.

</details>

✅ The plan shows **Succeeded**.

---

## 1d.5 Review your VM on site B, then power it on (site B)

**Console (site B):** **Virtualization → VirtualMachines**, project **`student-01-0024-012`** → **`pg-s01-centos10`**.

| Tab | What you see |
|---|---|
| **Configuration → Network** | the NIC on **`default/vmnet23`**: the same VLAN as on site A |
| **Configuration → Storage** | three disks, now on **site B's storage class**: the storage migration |

```bash
oc -n student-01-0024-012 get pvc -o custom-columns=NAME:.metadata.name,SIZE:.spec.resources.requests.storage,CLASS:.spec.storageClassName
```

If it isn't running yet: **Actions → Start**, and wait for **Running**. On its **Overview**, **IP addresses** shows
`10.101.23.151`: the same address on a different cluster.

✅ Running on site B, disks on `nfs-csi`, address `10.101.23.151`.

> 💡 **The source is stopped, not deleted.** Site A still holds your VM, stopped, as of `start-cross-cluster-migration`.
> **Never run both copies at once:** they share an address and a database identity. To roll back, stop it on site B
> and start it on site A (rows written on B are not on A): that is what you do in 1d.7, so **keep** the site A copy.

---

## 1d.6 Inject: the move is complete, and read your rows back (site B)

Site B has its own class terminal (`sql-client`, in your project), which needs its own database sign-in once:

```bash
oc exec -it -n student-01-0024-012 deploy/sql-client -- dblogin
```

Give the server a minute to start PostgreSQL (Windows: a few minutes), then:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'cross-cluster-migration-complete', 'running on site B, disks on NFS');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ Every row your server held is there (Demos 1a – 1c), then `cross-cluster-started`, `start-cross-cluster-migration`
(both written on site A) and `cross-cluster-migration-complete` (written on site B), in order, with `chain_ok = t`.

**That is the end of Demo 1d.** Your server crossed from one OpenShift cluster to another, with its storage and every
row it held.

---

## 1d.7 Back to site A, before Demo 1e

Demo 1e backs up your project on **site A**. Run your server there again, **one copy at a time**:

1. **Console (site B):** **Virtualization → VirtualMachines → `pg-s01-centos10` → Actions → Stop**. Wait for
   **Stopped**.
2. **Console (site A):** the same VM → **Actions → Start**. It comes back as of `start-cross-cluster-migration`:
   the row written on site B stays on site B.

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"   # site A: chain_ok = t
```

**Next:** [Demo 1e: OADP backup and restore](01e-oadp-vm-backup.md), on the same server. · **Back to:**
[Demo 1](01-vmware-to-ove-cold.md)

## Thank you

- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11): OpenShift-to-OpenShift migration, network and storage maps
- [Red Hat Advanced Cluster Management for Kubernetes 2.13 documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13): importing clusters, fleet virtualization
- [Lab S — how this lab was built](setup-how-the-lab-is-built.md): the OpenShift-to-OpenShift provider, maps and the ACM import
- [OpenShift Data Foundation 4.19](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19)
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)

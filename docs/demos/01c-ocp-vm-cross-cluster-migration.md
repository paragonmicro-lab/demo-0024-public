---
title: "Demo 1c — Cross-cluster migration: OpenShift site A to site B, with its storage (bonus)"
subtitle: "ACM links the clusters · MTV moves your VM from site A (ODF) to site B (NFS) · inject and read back"
version: 1.1.0
date: 2026-10-15
---

# Demo 1c: cross-cluster migration, site A to site B (bonus)

> **Version 1.1.0 · draft · 2026-10-15** · class/demos/01c-ocp-vm-cross-cluster-migration.md

## What you will learn

Your VM already runs on OpenShift: [Demo 1](01-vmware-to-ove-cold.md) (cold) or, once it ships, [Demo 2](02-vmware-to-ove-warm.md)
(warm) moved it off VMware onto **site A**. Now you move it to a **second OpenShift cluster**, site B. Advanced
Cluster Management (ACM) links the two clusters into one fleet, and MTV moves the VM between them. The two sites use
different storage, so the move is also a **storage migration**: site A's disks on ODF are copied onto site B's NFS.
As in every demo, you write a row before and after the move and read them all back.

**Before you start:** your VM runs on **site A** (after Demo 1 or 2). You are signed in to **both** consoles. Commands
run in each console's **Web Terminal** (`>_`); each step says which site.

> ⚠ **The commands are an example. Change these values to yours before you run them:**
>
> | In the example | Is the example's | Replace with yours |
> |---|---|---|
> | `student-01-0024-012` | class ID and project, the same on both sites | your class ID |
> | `win-pg-s01` | your VM | your VM |
> | `10.101.23.101` | your VM's address, the SQL server: it keeps it on site B too | your VM's address |
> | `ocp-lz1` → `ocp-lz2` | site A → site B (provider `ovedr-ocp-lz1`, on site B) | your booking's `A` and `B` |

<!-- 🎬 video slot 01c-ocp-vm-cross-cluster-migration: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01c-ocp-vm-cross-cluster-migration.jpg">
  <source src="../.images/01c-ocp-vm-cross-cluster-migration.mp4" type="video/mp4">
  <a href="../.images/01c-ocp-vm-cross-cluster-migration.mp4">🎬 Video: Demo 1c: cross-cluster migration</a>
</video>
<sub>🎬 *Demo 1c: cross-cluster migration: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1c.0 Inject: the demo has started (site A)

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'cross-cluster-started', 'Demo 1c: running on site A');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
```

---

## 1c.1 ACM: the two clusters, linked (site B)

Site B runs the ACM **hub**, and site A is imported into it as a **managed cluster**.

**Console (site B):** switch the perspective at the top left to **All Clusters** → **Infrastructure → Clusters**.

✅ `ocp-lz1` (site A) shows **Ready**, next to `local-cluster` (site B itself).

```bash
oc get managedclusters                              # ocp-lz1  JOINED True  AVAILABLE True
```

**Your VM, seen from the hub:** **All Clusters → Infrastructure → Virtual machines** **[verify: ACM 2.13 fleet
virtualization view on the hub]**. `win-pg-s01` shows on cluster `ocp-lz1`, in your project, **Running**.

ACM shows you where the VM is; **MTV does the move.**

---

## 1c.2 MTV: the link to site A, and the maps (site B)

**Console (site B):** **Migration for Virtualization → Providers → `ovedr-ocp-lz1`**.

✅ **Ready** and **Connected**: site B's MTV reads site A's VMs.

| Map (on site B) | What it does for your VM |
|---|---|
| Network map `ovedr-ocp-lz1-net` | `default/vmnet23` on site A → `default/vmnet23` on site B: same VLAN, so your VM **keeps its address** |
| Storage map `ovedr-ocp-lz1-storage` | `ocs-storagecluster-ceph-rbd-virtualization` (ODF, site A) → `nfs-csi` (NFS, site B): the **storage migration** |

```bash
oc -n openshift-mtv get provider ovedr-ocp-lz1                                     # READY True · CONNECTED True
oc -n openshift-mtv get storagemap ovedr-ocp-lz1-storage -o jsonpath='{.spec.map}{"\n"}'   # ODF → nfs-csi
```

MTV doesn't move the disks themselves: for each one it creates a new volume on site B in `nfs-csi` and fills it by
**reading the disk on site A over the network**. Site A's disks are left untouched.

---

## 1c.3 Inject, then stop your VM (site A)

Write the "start" row **before** the VM stops, while its database still answers:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'start-cross-cluster-migration', 'last row on site A');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"   # read back: your rows so far, oldest first
```

**Console (site A):** **Virtualization → VirtualMachines → `win-pg-s01` → Actions → Stop**.

✅ The VM shows **Stopped**.

---

## 1c.4 Create your plan, and start it (site B)

This plan isn't staged: you create it, then start it.

**Console (site B):** **Migration for Virtualization → Migration plans → Create plan**:

| Field | Value |
|---|---|
| Plan name | `d1c-win-pg-s01` (`d1c-<your VM>`) |
| Plan project | `openshift-mtv` |
| Source provider | `ovedr-ocp-lz1` |
| Target provider | `host` (site B) |
| Target project | `student-01-0024-012` |
| Virtual machines | **only** your VM, `win-pg-s01` (in project `student-01-0024-012`) |
| Network map | `ovedr-ocp-lz1-net` (use the existing map) |
| Storage map | `ovedr-ocp-lz1-storage` (use the existing map) |
| Migration type | **Cold** |

**Create plan**. When it shows **Ready**, click **Start** → confirm. Expand your VM on the plan's page to watch
**DiskTransfer**: three disks copied from ODF on site A into `nfs-csi` on site B.

```bash
oc -n openshift-mtv get plan d1c-win-pg-s01 -w      # Ready → Executing → Succeeded (Ctrl-C)
```

<details markdown="1">
<summary>Or: create and start the plan from the CLI (site B Web Terminal)</summary>

This builds the **same** plan under its own name, `d1c-cli-win-pg-s01`, so it never touches one you made in the
console. Use **one** of them: a VM can be in only one running plan.

Set the values once. Every command below reads them:

```bash
# your class ID: your project on both sites, and where MTV creates the VM on site B
CLASS_ID=student-01-0024-012
# your VM's name on site A (MTV keeps the name on site B)
VM=win-pg-s01
# the provider on site B that reads site A ("ovedr-ocp-" + site A's slot), and its two maps
SRC=ovedr-ocp-lz1
# a NEW plan name, so this never overwrites a plan you created in the console
PLAN=d1c-cli-$VM
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

## 1c.5 Review your VM on site B, then power it on (site B)

**Console (site B):** **Virtualization → VirtualMachines**, project **`student-01-0024-012`** → **`win-pg-s01`**.

| Tab | What you see |
|---|---|
| **Configuration → Network** | the NIC on **`default/vmnet23`**: the same VLAN as on site A |
| **Configuration → Storage** | three disks, now on **`nfs-csi`** (they were on ODF on site A): the storage migration |

```bash
oc -n student-01-0024-012 get pvc -o custom-columns=NAME:.metadata.name,SIZE:.spec.resources.requests.storage,CLASS:.spec.storageClassName
```

If it isn't running yet: **Actions → Start**, and wait for **Running**. On its **Overview**, **IP addresses** shows
`10.101.23.101`: the same address on a different cluster.

✅ Running on site B, disks on `nfs-csi`, address `10.101.23.101`.

> 💡 **The source is stopped, not deleted.** Site A still holds your VM, stopped, as of `start-cross-cluster-migration`.
> **Never run both copies at once:** they share an address and a database identity. To roll back, stop it on site B
> and start it on site A (rows written on B are not on A). Once you've checked site B, your instructor can delete the
> site A copy.

---

## 1c.6 Inject: the move is complete, and read your rows back (site B)

Site B has its own class terminal (`sql-client`, in your project), which needs its own database sign-in once:

```bash
oc exec -it -n student-01-0024-012 deploy/sql-client -- dblogin
```

Give Windows a minute to start PostgreSQL, then:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'cross-cluster-migration-complete', 'running on site B, disks on NFS');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ Every row from Lab 0 and Demo 1 (or 2) is there, then `cross-cluster-started`, `start-cross-cluster-migration`
(both written on site A) and `cross-cluster-migration-complete` (written on site B), in order, with `chain_ok = t`.

**That is the end of Demo 1c.** Your VM crossed from one OpenShift cluster to another, and from ODF to NFS, with every
row it held.

**Back to:** [Demo 1](01-vmware-to-ove-cold.md) · **Next:** [Demo 2: warm migration with storage copy offload](02-vmware-to-ove-warm.md) (🚧 coming soon: read only).

## Thank you

- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11): OpenShift-to-OpenShift migration, network and storage maps
- [Red Hat Advanced Cluster Management for Kubernetes 2.13 documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13): importing clusters, fleet virtualization
- [Lab S — how this lab was built](setup-how-the-lab-is-built.md): the OpenShift-to-OpenShift provider, maps and the ACM import
- [OpenShift Data Foundation 4.19](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19)
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)

---
title: "Demo 1 — VMware to OpenShift Virtualization: cold migration"
subtitle: "Inject · check your plan and the vCenter link · shut down · migrate · review on OpenShift · inject and read back"
version: 1.1.2
date: 2026-10-14
---

# Demo 1: VMware to OpenShift Virtualization (cold migration)

> **Version 1.1.2 · draft · 2026-10-14** · class/demos/01-vmware-to-ove-cold.md

## What you will learn

You move your own Windows Server + PostgreSQL VM off VMware vSphere and onto OpenShift Virtualization with the
Migration Toolkit for Virtualization (MTV). A **cold** migration copies the disks while the VM is **off**: simple and
consistent, at the cost of an outage for the whole copy. Before and after the move you write a row into your VM's
database and read every row back, so you can see your data survive the move.

**Before you start:** do [Lab 0](00-access-and-tools.md). You are signed in to the **site A** console, and your
database rows from Lab 0 are there. Every command below runs in the console's **Web Terminal** (`>_`).

> ⚠ **The commands are an example. Change these values to yours before you run them:**
>
> | In the example | Is the example's | Replace with yours |
> |---|---|---|
> | `student-01-0024-012` | class ID and project (`student-01` on booking `0024-012`) | your class ID |
> | `win-pg-s01` | your VM (`s` + your student number) | your VM |
> | `10.101.23.101` | your VM's address, the SQL server (it keeps it on OpenShift) | your VM's address, from Lab 0.1 |
> | `vcenter01` / VLAN `23` | your booking's vCenter (`N` = 1) and VLAN (`22 + N`) | from your instructor's `N` |

<!-- 🎬 video slot 01-vmware-to-ove-cold: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01-vmware-to-ove-cold.jpg">
  <source src="../.images/01-vmware-to-ove-cold.mp4" type="video/mp4">
  <a href="../.images/01-vmware-to-ove-cold.mp4">🎬 Video: Demo 1: cold migration</a>
</video>
<sub>🎬 *Demo 1: cold migration: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1.0 Inject: the demo has started

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'demo-started', 'Demo 1: cold migration');"
```

✅ It prints the row it wrote, with stage `demo-started`. (If it says *First sign in … dblogin*, run Lab 0's step 1 again.)

---

## 1.1 Check your migration plan, and the link to vCenter

The lab staged everything for you: a **provider** that links OpenShift to your vCenter, and **your own migration
plan**, ready but not started.

**Console (site A):** **Migration for Virtualization → Providers → `ovedr-vcenter01`**.

✅ Status **Ready**, and **Connected**: OpenShift can log in to vCenter and read its inventory.

**Console:** **Migration for Virtualization → Migration plans → `d1-win-pg-s01`** (your plan is `d1-<your VM>`).

✅ Status **Ready**. Source provider `ovedr-vcenter01`, your VM, target project **your class ID**, type **Cold**.
The network map puts your VM on your VLAN's bridge, so it keeps its address; the storage map puts its disks on ODF.

The same from the Web Terminal:

```bash
oc -n openshift-mtv get provider ovedr-vcenter01                     # READY True · CONNECTED True
oc -n openshift-mtv get plan d1-win-pg-s01                           # READY True
```

<details markdown="1">
<summary>My plan says "Cannot start" (critical concerns)</summary>

![A plan that cannot start: VMNetworksNotMapped and VMStorageNotMapped](../.images/01-plan-cannot-start.png)
*Two critical concerns block the start: the plan's maps don't cover your VM's network or its datastore. The two
information concerns are fine for a cold migration.*

| Concern | What it means | Fix |
|---|---|---|
| **VMNetworksNotMapped** (critical) | Your VM's NIC is on a vSphere network (`vm-network`) that the plan's **network map** has no row for | Map `vm-network` |
| **VMStorageNotMapped** (critical) | Your VM's disks are on a datastore (`coresan01_data00`) that the plan's **storage map** has no row for | Map `coresan01_data00` |
| VM snapshot detected (info) | Your VM has a snapshot. MTV migrates the VM's **current** state | Nothing to do |
| Disk serial numbers may be truncated (info) | SCSI serials are shortened on OpenShift | Nothing to do |

**Fix in the console:** open the plan → **Mappings** tab → **Edit mappings**:

1. **Network:** source `vm-network` → target **your VLAN's bridge** `default/vmnet23` (keeps your address).
2. **Storage:** source `coresan01_data00` → target `ocs-storagecluster-ceph-rbd-virtualization`.
3. **Save.** The concerns clear within a minute, and the status changes from *Cannot start* to *Ready*.

</details>

---

## 1.2 Inject, then shut your VM down

Your database lives **inside** the VM, so write the "start" row **before** you shut it down:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'start-cold-migration', 'on vSphere');"
```

**vCenter:** **VMs and Templates → datacenter01 → vms → students → `student-01-0024-012` → `win-pg-s01`** →
**Actions → Power → Shut Down Guest OS**. A clean guest shutdown flushes PostgreSQL to disk first.

✅ The VM shows **Powered Off** in vCenter.

---

## 1.3 Start the migration

**Console (site A):** **Migration for Virtualization → Migration plans → `d1-win-pg-s01`** → **Start** → confirm.

Watch it on the plan's page: expand your VM to see each step with a progress bar.

1. **Initialize** and **PreflightInspection**
2. **DiskTransfer**: three disks (`C:`, `D:` data, `L:` WAL), copied from vSphere
3. **ImageConversion**: `virt-v2v` installs the virtio drivers and the QEMU guest agent, and removes VMware Tools
4. **VirtualMachineCreation**

```bash
oc -n openshift-mtv get plan d1-win-pg-s01 -w                        # Executing → Succeeded (Ctrl-C to stop watching)
```

✅ The plan shows **Succeeded**.

<details markdown="1">
<summary>Or: create and start your own cold plan from the CLI (instead of the staged one)</summary>

This builds a cold plan like your staged `d1-win-pg-s01`, under a **new name**, so the staged plan stays as the lab
built it. Use **one** of them: a VM can be in only one running plan. If you start this one, skip the console's
**Start** above.

Set the values once. Every command below reads them:

```bash
# your class ID: MTV creates your VM in this project on site A
CLASS_ID=student-01-0024-012
# your VM's name in vCenter (MTV keeps the name on OpenShift)
VM=win-pg-s01
# the provider that links site A to your booking's vCenter ("ovedr-vcenter0" + N), and its maps
SRC=ovedr-vcenter01
# a NEW plan name, so this never touches the staged d1-win-pg-s01
PLAN=$CLASS_ID-cold-cli
```

Create the plan. The maps are the class's: `-net-bridge` keeps your VM on its VLAN (and its address), `-storage`
puts its disks on ODF:

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
    network: { name: $SRC-net-bridge, namespace: openshift-mtv }
    storage: { name: $SRC-storage,    namespace: openshift-mtv }
  vms:
    - name: $VM
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

</details>

<details markdown="1">
<summary>A step failed: read the logs before you delete anything</summary>

![A failed plan: ImageConversion, Guest conversion failed](../.images/01-plan-failed-imageconversion.png)
*A failed cold migration: ImageConversion (virt-v2v) failed, and the error says "See pod logs for details". There is no
Cancel on a failed plan, only Delete, and **deleting the plan deletes its pods and their logs**, so read them first.*

The conversion runs in a **pod in your project**, named after your plan. After a failure it stays there in status
*Error*, with its full log.

**Console:** **Workloads → Pods**, project **your class ID** → the pod named after your plan → **Logs**.

**Web Terminal:**

```bash
oc -n student-01-0024-012 get pods | grep d1-win-pg-s01                                     # the conversion pod (Error)
POD=$(oc -n student-01-0024-012 get pods -o name | grep d1-win-pg-s01 | head -1)
oc -n student-01-0024-012 logs "$POD" | grep -E 'virt-v2v: error|error:|HTTP' | sort | uniq -c | tail
```

Read the log **bottom up**: the last `virt-v2v: error:` line is the verdict, and the lines above it are the cause.
For example, `HTTP 404` on a `…-flat.vmdk` URL means MTV could not read the VM's disk from vCenter. Show your
instructor before you delete or archive the plan.

</details>

---

## 1.4 Review your VM on OpenShift, then power it on

**Console (site A):** **Virtualization → VirtualMachines**, project **`student-01-0024-012`** → **`win-pg-s01`**.

| Tab | What you see |
|---|---|
| **Configuration → Network** | one NIC on **`default/vmnet23`**: your VLAN's bridge, the same VLAN the VM had on vSphere (port group `vm-network`, VLAN 23) |
| **Configuration → Storage** | **three** disks, now persistent volume claims on **`ocs-storagecluster-ceph-rbd-virtualization`** (ODF): 60 GB `C:`, 10 GB data, 5 GB WAL |

```bash
oc -n student-01-0024-012 get vm win-pg-s01 -o jsonpath='{range .spec.template.spec.networks[*]}{.multus.networkName}{"\n"}{end}'   # default/vmnet23
oc -n student-01-0024-012 get pvc -o custom-columns=NAME:.metadata.name,SIZE:.spec.resources.requests.storage,CLASS:.spec.storageClassName
```

**Power it on:** **Actions → Start**. Wait for **Running**.

```bash
oc -n student-01-0024-012 get vmi win-pg-s01 -w                      # Scheduling → Running (Ctrl-C to stop watching)
```

On the VM's **Overview**, the *Guest agent* shows as connected, and **IP addresses** shows `10.101.23.101`: your VM
kept its address, so your SQL server is still at the same `-h` as on vSphere.

✅ Running, guest agent connected, address `10.101.23.101`.

---

## 1.5 Inject: the migration is complete, and read your rows back

Give Windows a minute to start PostgreSQL, then:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'cold-migration-complete', 'on OpenShift site A');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ The read-back lists, in order: `start-class` (Lab 0), `demo-started`, `start-cold-migration` (written on vSphere)
and `cold-migration-complete` (written on OpenShift). `ovedr_verify` shows `chain_ok = t`: no row lost, changed or
reordered by the move.

**That is the end of Demo 1.** Your VM now runs on OpenShift Virtualization, with every row it held on VMware.

---

## Bonus labs (optional)

1a to 1c start from your VM running on site A, where Demo 1 left it. 1d builds a new server next to it.

| Bonus | What you do |
|---|---|
| [Demo 1a: snapshot and restore](01a-snapshot-restore.md) | snapshot your VM, destroy the database, bring it back from the snapshot's disks |
| [Demo 1b: live migration and storage migration](01b-live-and-storage-migration.md) | move the **running** VM to another node, then move its disks to another storage class, with no reboot |
| [Demo 1c: cross-cluster migration](01c-ocp-vm-cross-cluster-migration.md) | link the two clusters in ACM, then move your VM from site A (ODF) to site B (NFS) |
| [Demo 1d: a cloud-native golden image](01d-vm-cloudnative-golden-image.md) | build a new CentOS Stream 10 + PostgreSQL server purely in code, from Red Hat's boot source and a Template, and inject into it |

**Next:** [Demo 2: warm migration with storage copy offload](02-vmware-to-ove-warm.md) (🚧 coming soon: read only).

## Addendum: how the lab staged this demo, as code

Nothing you used in this demo was clicked together. The class's CI (the GitHub Actions workflow `deploy.yml`, action
`customize`) declared every object, and it runs again on every booking with the same result: run it twice and the
second run changes nothing. In order:

| CI job | Code | What it declares |
|---|---|---|
| `mtv-install` | Ansible role `mtv_operator` | checks MTV (installed by the OpenShift build, option `ove: cnv-mtv`), sets the pre-copy interval, grants students the right to create plans and migrations |
| `mtv-fabric` | `automation/customize/mtv_fabric.sh` | the link to vCenter (a Secret + a **Provider**), and the **NetworkMap**s and **StorageMap** |
| `stage` | `automation/customize/stage.sh` | one **Plan** per class VM, `d1-<vm>` (cold) and `d2-<vm>` (warm), **not started** |
| `migration-test` | `automation/test/migrate_both.sh` | the proof: a test clone migrates from vSphere to each site with the same Plan + Migration you used, and its data is checked (action `test-migration`) |

The chain MTV follows, and each object as the lab declares it (the Secret's password never appears in the code: CI
reads it from the vault at run time):

```yaml
# 1 · Provider: the link to vCenter. MTV reads its inventory (VMs, networks, datastores) through it.
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata: { name: ovedr-vcenter01, namespace: openshift-mtv }
spec:
  type: vsphere
  url: https://vcenter01.demo.lab-paragonmicro.com/sdk
  secret: { name: ovedr-vcenter01, namespace: openshift-mtv }   # user, password, the vCenter CA
---
# 2 · NetworkMap: vSphere port group → your VLAN's bridge on OpenShift, so the VM keeps its address
apiVersion: forklift.konveyor.io/v1beta1
kind: NetworkMap
metadata: { name: ovedr-vcenter01-net-bridge, namespace: openshift-mtv }
spec:
  provider: { source: { name: ovedr-vcenter01, namespace: openshift-mtv }, destination: { name: host, namespace: openshift-mtv } }
  map:
    - { source: { name: vm-network }, destination: { type: multus, name: vmnet23, namespace: default } }
---
# 3 · StorageMap: vSphere datastore → OpenShift storage class (ODF on site A)
apiVersion: forklift.konveyor.io/v1beta1
kind: StorageMap
metadata: { name: ovedr-vcenter01-storage, namespace: openshift-mtv }
spec:
  provider: { source: { name: ovedr-vcenter01, namespace: openshift-mtv }, destination: { name: host, namespace: openshift-mtv } }
  map:
    - { source: { name: coresan01_data00 }, destination: { storageClass: ocs-storagecluster-ceph-rbd-virtualization } }
---
# 4 · Plan: which VM, to which project, through which maps. Staged, not started.
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata: { name: d1-win-pg-s01, namespace: openshift-mtv, labels: { ovedr/stage: plan, ovedr/type: cold } }
spec:
  warm: false
  targetNamespace: student-01-0024-012
  provider: { source: { name: ovedr-vcenter01, namespace: openshift-mtv }, destination: { name: host, namespace: openshift-mtv } }
  map:
    network: { name: ovedr-vcenter01-net-bridge, namespace: openshift-mtv }
    storage: { name: ovedr-vcenter01-storage,    namespace: openshift-mtv }
  vms: [ { name: win-pg-s01 } ]
```

5 · **Migration**: the only object left to you. The console's **Start** creates it, as does the CLI box in 1.3.

Read the live ones from your Web Terminal:

```bash
oc -n openshift-mtv get provider ovedr-vcenter01 -o yaml
oc -n openshift-mtv get networkmap ovedr-vcenter01-net-bridge -o yaml
oc -n openshift-mtv get storagemap ovedr-vcenter01-storage -o yaml
oc -n openshift-mtv get plan d1-win-pg-s01 -o yaml
```

Field by field, with the console screens for each: [Lab S §S.5–S.8 and §S.12](setup-how-the-lab-is-built.md).

## Thank you

- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11)
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)
- [Lab S — how this lab was built](setup-how-the-lab-is-built.md): the provider, maps and plan, built by hand
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization (course)](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)
- [Red Hat interactive labs (free, in the browser)](https://www.redhat.com/en/interactive-labs)

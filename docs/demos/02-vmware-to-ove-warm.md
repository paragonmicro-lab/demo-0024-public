---
title: "Demo 2 — VMware to OpenShift Virtualization: warm migration"
subtitle: "Inject · turn on Changed Block Tracking · start the warm plan while the VM runs · cut over · review on OpenShift · inject and read back"
version: 2.1.0
date: 2026-10-13
---

# Demo 2: VMware to OpenShift Virtualization (warm migration)

> **Version 2.1.0 · draft · 2026-10-13** · class/demos/02-vmware-to-ove-warm.md

## What you will learn

You move your Windows Server + PostgreSQL VM off VMware vSphere and onto OpenShift Virtualization again, this time
**warm**. A warm migration copies the disks while the VM keeps **running**: a first full copy, then every 5 minutes only
the blocks that changed. The VM stops only for the **cutover**, a last small copy, so the outage is minutes instead of
the whole copy. vSphere's **Changed Block Tracking** (CBT) is what tells MTV which blocks changed, so you turn it on
first. You write rows before, during and after the move, read them all back, and measure the real outage from your
VM's own heartbeat.

| Phase | Your VM | What MTV does |
|---|---|---|
| **Pre-copy 1** | running | snapshots each disk on vSphere and copies the **whole** disk |
| **Pre-copy 2, 3, …** | running | every 5 minutes, copies **only the blocks changed** since the last round (CBT) |
| **Cutover** | **stopped** | one last snapshot, copies the last small change, converts the drivers, starts the VM on OpenShift |

**Before you start:** do [Lab 0](00-access-and-tools.md). You are signed in to the **site A** console. Every command
below runs in the console's **Web Terminal** (`>_`). Your VM must be **running on vSphere**:

- **Straight after Lab 0:** it is. Go to 2.0.
- **After Demo 1:** your VM runs on OpenShift and the vSphere original is powered off. Both use the same address, so
  only one may run at a time. Stop the OpenShift copy first (**Virtualization → VirtualMachines → `win-pg-s01` →
  Actions → Stop**), then power the original on in vCenter (**`win-pg-s01` → Actions → Power → Power On**). The
  original holds your rows up to `start-cold-migration`. `cold-migration-complete` was written on the OpenShift copy,
  so the original doesn't have it, and that's expected.

> ⚠ **The commands are an example. Change these values to yours before you run them:**
>
> | In the example | Is the example's | Replace with yours |
> |---|---|---|
> | `student-01-0024-012` | class ID and project (`student-01` on booking `0024-012`) | your class ID |
> | `win-pg-s01` | your VM (`s` + your student number); on OpenShift it lands as `win-pg-s01-warm` | your VM |
> | `10.101.23.101` | your VM's address, the SQL server (it keeps it on OpenShift) | your VM's address, from Lab 0.1 |
> | `vcenter01` / VLAN `23` | your booking's vCenter (`N` = 1) and VLAN (`22 + N`) | from your instructor's `N` |

<!-- 🎬 video slot 02-vmware-to-ove-warm: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/02-vmware-to-ove-warm.jpg">
  <source src="../.images/02-vmware-to-ove-warm.mp4" type="video/mp4">
  <a href="../.images/02-vmware-to-ove-warm.mp4">🎬 Video: Demo 2: warm migration</a>
</video>
<sub>🎬 *Demo 2: warm migration: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 2.0 Inject: the demo has started

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'demo-started', 'Demo 2: warm migration');"
```

✅ It prints the row it wrote, with stage `demo-started`. (If it says *First sign in … dblogin*, run Lab 0's step 1
again. If it can't connect, your VM isn't running on vSphere yet: see **Before you start**.)

---

## 2.1 Check the link to vCenter, and your warm plan

The lab staged a **warm** plan for you, ready but not started: **`d2-win-pg-s01`** (`d2-<your VM>`). It lands your
VM on site A as **`win-pg-s01-warm`**, so it never collides with a Demo 1 copy.

**Console (site A):** **Migration for Virtualization → Providers → `ovedr-vcenter01`**.

✅ Status **Ready**, and **Connected**.

**Console:** **Migration for Virtualization → Migration plans → `d2-win-pg-s01`**.

✅ Type **Warm**, source provider `ovedr-vcenter01`, your VM, target project **your class ID**. The status is **not**
Ready yet: expand your VM to see why.

| Concern | What it means | Who fixes it |
|---|---|---|
| **VMMissingChangedBlockTracking** | your VM's disks don't have CBT on, so MTV can't copy only the changes | **you**, in 2.2 |
| **VDDKInitImageUnavailable** | the provider has no VDDK image, the VMware library MTV needs to read changed blocks | the **instructor**: tell them and stop here (Lab S §S.15) |

The same from the Web Terminal:

```bash
oc -n openshift-mtv get provider ovedr-vcenter01 -o jsonpath='{.status.phase}  vddk={.spec.settings.vddkInitImage}{"\n"}'   # Ready  vddk=<an image>
oc -n openshift-mtv get plan d2-win-pg-s01 -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
```

✅ The provider shows a `vddk=` image, and the plan's only concern is `VMMissingChangedBlockTracking`.

---

## 2.2 Turn on Changed Block Tracking

CBT is one switch for the VM (`ctkEnabled`) **plus** one for each disk (`scsi0:N.ctkEnabled`). vCenter accepts them
only while the VM is **off**, and they take effect at the next power-on.

**Shut it down.** **vCenter:** **VMs and Templates → datacenter01 → vms → students → `student-01-0024-012` →
`win-pg-s01`** → **Actions → Power → Shut Down Guest OS**. Wait for **Powered Off**.

**Add the four parameters.** **`win-pg-s01` → Edit Settings → Advanced Parameters**. For each row, type the
**Attribute** and **Value**, then click **ADD**:

| Attribute | Value |
|---|---|
| `ctkEnabled` | `true` |
| `scsi0:0.ctkEnabled` | `true` |
| `scsi0:1.ctkEnabled` | `true` |
| `scsi0:2.ctkEnabled` | `true` |

![Adding ctkEnabled = true on the Advanced Parameters tab](../.images/02-cbt-advanced-params-add.png)
*Adding a row: Attribute `ctkEnabled`, Value `true`, then **ADD**. The filter (funnel icon) set to `ctk` shows only the
CBT rows.*

![The Advanced Parameters list with all four ctkEnabled rows set to true](../.images/02-cbt-advanced-params.png)
*All four rows set: the VM, plus `scsi0:0`, `scsi0:1` and `scsi0:2` for its three disks. Click **OK**.*

<details markdown="1">
<summary>Check your disks really are scsi0:0, 0:1 and 0:2</summary>

**Edit Settings → Virtual Hardware**: expand each **Hard disk** and read its **Virtual Device Node**, for example
`SCSI controller 0 : SCSI(0:1)`. The class VMs have three disks on one controller: 60 GB `C:`, 5 GB `L:` (WAL) and
10 GB `D:` (data). If a node differs, use its number in the parameter name.

</details>

**Power it back on:** **Actions → Power → Power On**.

✅ Within a minute or two the plan `d2-win-pg-s01` shows **Ready**, and the CBT concern is gone:

```bash
oc -n openshift-mtv get plan d2-win-pg-s01 -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'   # Ready=True
```

If the concern stays, a disk is missing its `scsi0:N.ctkEnabled`, or the VM hasn't been powered on since you set it.

Give Windows a minute to start PostgreSQL, then check that the shutdown lost nothing:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ `chain_ok = t`.

---

## 2.3 Start the warm migration, and keep writing

**Console (site A):** **Migration for Virtualization → Migration plans → `d2-win-pg-s01`** → **Start** → confirm.

Your VM **keeps running on vSphere**. Expand it on the plan's page: MTV takes a snapshot on vSphere and copies the
whole of each disk (the first pre-copy). After that, every 5 minutes it copies only what changed. In vCenter, your
VM's **Snapshots** show MTV's snapshot for each round.

```bash
oc -n openshift-mtv get plan d2-win-pg-s01 -w                        # EXECUTING True (Ctrl-C to stop watching)
```

**Give it something to copy.** The database is busy while it copies: write a burst of 500 rows now, and again
after each pre-copy round if you like:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_load('student-01-0024-012', 'warm-precopy-load', 500);"
```

✅ It prints `rows_written = 500`. Your VM is still serving its database in the middle of the migration.

**Count the pre-copies:**

```bash
oc -n openshift-mtv get plan d2-win-pg-s01 -o jsonpath='pre-copies done: {.status.migration.vms[0].warm.successes}  next: {.status.migration.vms[0].warm.nextPrecopyAt}{"\n"}'
```

✅ Wait for **at least two** pre-copies. The first carries the whole disks; later ones carry only the changed
blocks, so they're much quicker. That difference is CBT at work.

<details markdown="1">
<summary>Or: create and start your own warm plan from the CLI (instead of the staged one)</summary>

This builds a warm plan like your staged `d2-win-pg-s01`, under a **new name**, so the staged plan stays as the lab
built it. Use **one** of them: a VM can be in only one running plan. If you start this one, skip the console's
**Start** above, and cut over with the last command below instead of the console.

Set the values once. Every command below reads them:

```bash
# your class ID: MTV creates your VM in this project on site A
CLASS_ID=student-01-0024-012
# your VM's name in vCenter
VM=win-pg-s01
# the provider that links site A to your booking's vCenter ("ovedr-vcenter0" + N), and its maps
SRC=ovedr-vcenter01
# a NEW plan name, so this never touches the staged d2-win-pg-s01
PLAN=$CLASS_ID-warm-cli
```

Create the plan. `warm: true` is the only difference from a cold plan; `targetName` gives the copy on OpenShift its
own name:

```bash
oc apply -f - <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata: { name: $PLAN, namespace: openshift-mtv }
spec:
  warm: true
  targetNamespace: $CLASS_ID
  provider:
    source:      { name: $SRC, namespace: openshift-mtv }
    destination: { name: host, namespace: openshift-mtv }
  map:
    network: { name: $SRC-net-bridge, namespace: openshift-mtv }
    storage: { name: $SRC-storage,    namespace: openshift-mtv }
  vms:
    - { name: $VM, targetName: $VM-warm-cli }
EOF
oc -n openshift-mtv get plan $PLAN -w                # READY True (Ctrl-C)
```

Start it: a **Migration** is what the console's Start button creates. A warm migration pre-copies until it has a
**cutover** time:

```bash
oc apply -f - <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata: { name: $PLAN-run1, namespace: openshift-mtv }
spec: { plan: { name: $PLAN, namespace: openshift-mtv } }
EOF
oc -n openshift-mtv get plan $PLAN -w                # EXECUTING True (Ctrl-C)
```

In 2.4, after your `start-warm-migration` row, cut over by giving the Migration a cutover time of **now**:

```bash
oc -n openshift-mtv patch migration $PLAN-run1 --type merge -p "{\"spec\":{\"cutover\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}}"
oc -n openshift-mtv get plan $PLAN -w                # SUCCEEDED True (Ctrl-C)
```

</details>

---

## 2.4 Inject, then cut over

Write the "start" row while your VM still runs on vSphere. It must survive the cutover:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'start-warm-migration', 'on vSphere, before cutover');"
```

**Console (site A):** **Migration plans → `d2-win-pg-s01`** → **Cutover** → keep the date and time it offers
(**now**) → **Set cutover**.

MTV now shuts your VM down on vSphere, takes one last snapshot, copies the small change since the last round,
converts the drivers (`virt-v2v` installs virtio and the QEMU guest agent, and removes VMware Tools) and creates your VM
on OpenShift.

```bash
oc -n openshift-mtv get plan d2-win-pg-s01 -w                        # SUCCEEDED True (Ctrl-C to stop watching)
```

✅ The plan shows **Succeeded**.

---

## 2.5 Review your VM on OpenShift

**Console (site A):** **Virtualization → VirtualMachines**, project **`student-01-0024-012`** → **`win-pg-s01-warm`**.

| Tab | What you see |
|---|---|
| **Configuration → Network** | one NIC on **`default/vmnet23`**: your VLAN's bridge, so it keeps its address |
| **Configuration → Storage** | **three** disks on **`ocs-storagecluster-ceph-rbd-virtualization`** (ODF) |

```bash
oc -n student-01-0024-012 get vm win-pg-s01-warm -o jsonpath='{range .spec.template.spec.networks[*]}{.multus.networkName}{"\n"}{end}'   # default/vmnet23
oc -n student-01-0024-012 get vmi win-pg-s01-warm -w                 # Running (Ctrl-C to stop watching)
```

Your VM was running on vSphere when it was cut over, so MTV starts it on OpenShift. If it shows **Stopped**, start
it: **Actions → Start** **[verify]**.

On the VM's **Overview**, the *Guest agent* shows as connected, and **IP addresses** shows `10.101.23.101`.

✅ Running, guest agent connected, address `10.101.23.101`.

---

## 2.6 Inject: the migration is complete, read back, and measure the outage

Give Windows a minute to start PostgreSQL, then:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'warm-migration-complete', 'on OpenShift site A');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ The read-back ends with `demo-started`, the 500 `warm-precopy-load` rows (written **while** the disks copied),
`start-warm-migration` (the last row on vSphere) and `warm-migration-complete` (written on OpenShift).
`ovedr_verify` shows `chain_ok = t`: nothing written during the copy was lost, changed or reordered.

**The real outage.** Your VM's heartbeat writer adds a row every 5 seconds, so the outage is the gap in it after
your `start-warm-migration` row:

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT ts AS back_at, date_trunc('second', gap) AS outage FROM (SELECT ts, ts - lag(ts) OVER (ORDER BY id) AS gap FROM heartbeat) g WHERE gap > interval '10 seconds' AND ts > (SELECT max(ts) FROM marks WHERE student = 'student-01-0024-012' AND stage = 'start-warm-migration') ORDER BY ts;"
```

✅ One row: the cutover outage. Compare it with how long the first pre-copy took. A cold migration would have
been down for all of that copy.

**The guest tools were swapped:**

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -x -c "SELECT ts, platform, vmtools, qemu_ga, virtio FROM guest_tools ORDER BY id DESC LIMIT 1;"
```

✅ `qemu_ga` is **Running**, `virtio` lists the network and disk drivers, and `vmtools` is not running.

**That is the end of Demo 2.** Your VM moved to OpenShift Virtualization while it kept writing, and stopped only for
the cutover.

> 💡 **One copy on your VLAN at a time.** If you did Demo 1, your project now holds two VMs, `win-pg-s01` (stopped)
> and `win-pg-s01-warm` (running), with the same address. Never run both at once.

---

## Bonus labs (optional)

The Demo 1 bonus labs work on this VM too: use **`win-pg-s01-warm`** wherever they say `win-pg-s01`.

| Bonus | What you do |
|---|---|
| [Demo 1a: snapshot and restore](01a-snapshot-restore.md) | snapshot your VM, destroy the database, bring it back from the snapshot's disks |
| [Demo 1b: live migration and storage migration](01b-live-and-storage-migration.md) | move the **running** VM to another node, then move its disks to another storage class, with no reboot |
| [Demo 1c: cross-cluster migration](01c-ocp-vm-cross-cluster-migration.md) | link the two clusters in ACM, then move your VM from site A (ODF) to site B (NFS) |

**Back to:** [Demo 1](01-vmware-to-ove-cold.md) · [Student guide](../student-guide.md).

## Addendum: how the lab staged this demo, as code

The provider, the maps and the CI pipeline are the same as Demo 1's ([Demo 1 addendum](01-vmware-to-ove-cold.md#addendum-how-the-lab-staged-this-demo-as-code)).
A warm migration needs three more things, and CI declares each one:

| CI job | Code | What it declares for warm |
|---|---|---|
| `mtv-install` | Ansible role `mtv_operator` | the **pre-copy interval**: 5 minutes instead of MTV's default 60, so a class sees several rounds |
| `vddk` | `automation/customize/vddk.sh` | the **VDDK image**: Broadcom's library, which can't be redistributed, built from the lab's private copy into the cluster's own registry, then set on the provider ([Lab S §S.15](setup-how-the-lab-is-built.md)) |
| `stage` | `automation/customize/stage.sh` | the **warm plan** `d2-<vm>`, landing as `<vm>-warm`. Staged and reported, never fatal: it waits for CBT, which you turn on in 2.2 |

```yaml
# the pre-copy interval, on MTV's own controller
apiVersion: forklift.konveyor.io/v1beta1
kind: ForkliftController
metadata: { name: forklift-controller, namespace: openshift-mtv }
spec:
  controller_precopy_interval: "5"        # minutes between pre-copy rounds
---
# the provider, plus the VDDK image (the only difference from Demo 1's)
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata: { name: ovedr-vcenter01, namespace: openshift-mtv }
spec:
  type: vsphere
  url: https://vcenter01.demo.lab-paragonmicro.com/sdk
  secret: { name: ovedr-vcenter01, namespace: openshift-mtv }
  settings:
    vddkInitImage: image-registry.openshift-image-registry.svc:5000/openshift-mtv/vddk:8.0.3
---
# the staged warm plan: warm: true, and a target name so it never collides with Demo 1's copy
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata: { name: d2-win-pg-s01, namespace: openshift-mtv, labels: { ovedr/stage: plan, ovedr/type: warm } }
spec:
  warm: true
  targetNamespace: student-01-0024-012
  provider: { source: { name: ovedr-vcenter01, namespace: openshift-mtv }, destination: { name: host, namespace: openshift-mtv } }
  map:
    network: { name: ovedr-vcenter01-net-bridge, namespace: openshift-mtv }
    storage: { name: ovedr-vcenter01-storage,    namespace: openshift-mtv }
  vms: [ { name: win-pg-s01, targetName: win-pg-s01-warm } ]
```

What stays a hands-on step, on purpose: **CBT** (2.2). It's a setting on the VMware VM, which the lab could set too
(`govc vm.change -e ctkEnabled=TRUE …`), but turning it on yourself is the lesson. Then the **Migration** and its
**cutover** (2.3, 2.4).

```bash
oc -n openshift-mtv get plan d2-win-pg-s01 -o yaml
oc -n openshift-mtv get provider ovedr-vcenter01 -o jsonpath='{.spec.settings}{"\n"}'
```

## Thank you

- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11): warm migration, VDDK and the pre-copy settings
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)
- [Lab S — how this lab was built](setup-how-the-lab-is-built.md): the provider, VDDK (§S.15), maps and plans, built by hand
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization (course)](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)
- [Red Hat interactive labs (free, in the browser)](https://www.redhat.com/en/interactive-labs)

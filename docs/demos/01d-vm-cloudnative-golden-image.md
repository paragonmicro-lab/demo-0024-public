---
title: "Demo 1d — A cloud-native golden image: a new Linux database server, built from code (bonus)"
subtitle: "Red Hat's CentOS Stream 10 boot source · the customer's Template and cloud-init · deploy in the console or with one command · inject into the new PostgreSQL"
version: 1.0.1
date: 2026-10-16
---

# Demo 1d: a cloud-native golden image (bonus)

> **Version 1.0.1 · draft · 2026-10-16** · class/demos/01d-vm-cloudnative-golden-image.md

## What you will learn

In Demo 1 you moved an existing Windows VM: a **pet**, built once by hand in vCenter, patched in place, and copied
whenever someone needs another. Here you build its replacement **from scratch, in code**: a **CentOS Stream 10**
server running the same PostgreSQL database, with its own logins, its data and its WAL (the log) on separate disks,
and its services running. Nothing is installed by hand and nothing is baked into an image:

| | The pet golden image (vSphere) | The cloud-native golden image (OpenShift) |
|---|---|---|
| **Base OS** | an ISO you install, then keep patching yourself | Red Hat's **boot source**, a cloud image that OpenShift Virtualization keeps up to date |
| **Customization** | clicks and scripts inside the image, then *sysprep* and *Convert to Template* | a **Template** with **cloud-init**: text in git, reviewed like code |
| **A new server** | clone the template, then fix what drifted | the same code, every time: **identical** servers in minutes |
| **Proof it works** | someone checks by hand | **CI** deploys it on every lab build and tests it with SQL |

You deploy it twice over: in the console (as in the video), and with one command. Then you inject rows into the new
Linux PostgreSQL and read them back, exactly as you did on Windows.

**Before you start:** do [Lab 0](00-access-and-tools.md). You are signed in to the **site A** console, and your
class terminal is signed in to the database (`dblogin`, once). Your Windows VM can be anywhere: this demo builds a
**new** VM next to it. Commands run in the console's **Web Terminal** (`>_`).

> ⚠ **The commands are an example. Change these values to yours before you run them:**
>
> | In the example | Is the example's | Replace with yours |
> |---|---|---|
> | `student-01-0024-012` | class ID and project | your class ID |
> | `pg-s01-centos10` | the new VM's name | `pg-s<NN>-centos10` |
> | `10.101.23.151` | the new VM's address: VLAN 23, **150 + your student number** | `10.101.<22 + N>.<150 + NN>` |
> | `23` | your booking's VLAN (`22 + N`, `vcenter01` → 23) | from your instructor's `N` |
> | `student-01` / `student_01` | your Linux login / your database login (underscore) | yours |

<!-- 🎬 video slot 01d-vm-cloudnative-golden-image: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01d-vm-cloudnative-golden-image.jpg">
  <source src="../.images/01d-vm-cloudnative-golden-image.mp4" type="video/mp4">
  <a href="../.images/01d-vm-cloudnative-golden-image.mp4">🎬 Video: Demo 1d: a cloud-native golden image</a>
</video>
<sub>🎬 *Demo 1d: a cloud-native golden image: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1d.1 Look at the golden image: it's code

The lab published the customer's Template, **`ovedr-centos10-postgres`**, on both clusters. It lives in the
**`openshift`** project, the shared one, so every project's catalog offers it.

**Console (site A):** first set the **Project** dropdown at the top to **your** project (`student-01-0024-012`),
then open **Virtualization → Catalog → Template catalog**. Find **CentOS Stream 10 + PostgreSQL 16 (class)**
(filter on *PostgreSQL*; it is a user template, not a Red Hat one). Open it: the boot source is Red Hat's
**CentOS Stream 10**, with three disks and your VLAN's network.

> ⚠ **"No Templates found"?** Check the **Project** dropdown. On **Virtualization → Templates** the list shows
> only the project you have selected, and the Template is not in `openshift-mtv` or your own project: switch
> it to **`openshift`** (or **All projects**). The Template catalog does not have this problem, because it
> always adds the `openshift` project's Templates to yours.

The same from the Web Terminal: what it asks for, and the code itself. **`oc process`** is how a Template is
used from the command line. A Template is YAML with blanks in it (`${NAME}`, `${IP}`, …):
`oc process --parameters` lists the blanks, which are the same fields the console's form asks for.
`oc process -p NAME=…` fills them in and prints the finished VM; it creates nothing until you pipe it to
`oc apply` (1d.2).

```bash
oc process --parameters -n openshift ovedr-centos10-postgres
oc get template -n openshift ovedr-centos10-postgres -o yaml | less          # q to quit
```

| In the Template | What it does |
|---|---|
| `dataVolumeTemplates` | the system disk, cloned from Red Hat's `centos-stream10` boot source, plus two **empty** disks: 10 GB for the data, 5 GB for the WAL |
| `networks` | one NIC on your VLAN's bridge (`default/vmnet23`), so the server has its own address on the lab network |
| `cloudInitNoCloud.networkData` | that address, its gateway and the lab's DNS |
| `cloudInitNoCloud.userData` | **cloud-init**, run on first boot: formats and mounts the two disks, installs PostgreSQL 16 with its WAL on its own disk, loads the class's `marks` toolkit, creates your Linux and database logins, starts the database and a heartbeat |

The whole file is also on the class site: [`ovedr-centos10-postgres.yaml`](../downloads/ovedr-centos10-postgres.yaml).

---

## 1d.2 Deploy it

Do **one** of the two. Both create the same VM.

**Console (site A), as in the video:** in the Template catalog, open **CentOS Stream 10 + PostgreSQL 16 (class)** →
**Customize VirtualMachine** → project **`student-01-0024-012`**, and fill in the parameters:

| Parameter | Example |
|---|---|
| VM name | `pg-s01-centos10` |
| VLAN | `23` |
| IP address | `10.101.23.151` |
| Linux login | `student-01` |
| Database login | `student_01` |
| Password (Linux and database) | your class password |

Then **Create VirtualMachine**.

**Or, the Web Terminal: the whole server is one command.** `oc process` fills in the Template's blanks, exactly
as the form did, and `oc apply` creates the result in your project. It asks for your password, so it never
appears on screen:

```bash
read -rsp 'Your class password: ' PW; echo; oc process -n openshift ovedr-centos10-postgres -p NAME=pg-s01-centos10 -p VLAN=23 -p IP=10.101.23.151 -p OS_USER=student-01 -p DB_USER=student_01 -p DB_PASSWORD="$PW" | oc apply -n student-01-0024-012 -f -; unset PW
```

✅ `virtualmachine.kubevirt.io/pg-s01-centos10 created`.

> 💡 **Your password is in the VM's cloud-init**, readable by you and the instructors in your project. That is fine
> for a lab account that dies with the booking. In production, cloud-init comes from a **Secret**
> (`cloudInitNoCloud.secretRef`), so the VM's definition holds no password.

---

## 1d.3 Watch it build itself

**Console:** **Virtualization → VirtualMachines → `pg-s01-centos10`**. It goes **Provisioning** (the disks are
created: the system disk is cloned from the boot source), **Starting**, then **Running**.

```bash
oc -n student-01-0024-012 get dv                                     # pg-s01-centos10, -pgdata, -pgwal: Succeeded
oc -n student-01-0024-012 get vmi pg-s01-centos10 -w                 # Running (Ctrl-C to stop watching)
```

Once it is **Running**, cloud-init installs and configures PostgreSQL: allow a few minutes. Open the VM's
**Console** tab and log in as **`student-01`** with your class password, then:

```bash
cloud-init status --wait                                 # status: done
lsblk -o NAME,SIZE,LABEL,MOUNTPOINT                      # vdc pgdata /var/lib/pgsql/data · vdd pgwal /var/lib/pgsql/wal
systemctl is-active postgresql ovedr-heartbeat           # active, active
sudo ls -l /var/lib/pgsql/data/pg_wal                    # → /var/lib/pgsql/wal/pg_wal: the WAL on its own disk
```

✅ cloud-init **done**, both database disks mounted, both services **active**. Every one of these was declared in
the Template, and none was typed by hand.

---

## 1d.4 Inject into your new Linux PostgreSQL, and read it back

Same commands as on Windows, at the new address. Your class terminal's `dblogin` works here too: same login, same
password, a different server.

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'cloudnative-started', 'Demo 1d: CentOS Stream 10 from the Template');"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_load('student-01-0024-012', 'cloudnative-load', 100);"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
```

✅ The read-back starts **fresh** at `cloudnative-started`, then your 100 `cloudnative-load` rows, and
`ovedr_verify` shows `chain_ok = t`. A new server has a new database: your Windows VM's rows are still on your
Windows VM.

**Is it alive, and what is it?**

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT count(*) AS beats, date_trunc('second', clock_timestamp() - max(ts)) AS staleness FROM heartbeat;"
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -x -c "SELECT version(); SELECT platform, qemu_ga, virtio FROM guest_tools ORDER BY id DESC LIMIT 1;"
```

✅ `staleness` under 10 seconds, PostgreSQL **16** on Linux, platform **kvm**, QEMU guest agent **active**, virtio
drivers: the same checks as your migrated Windows VM, on a server that was never migrated.

---

## 1d.5 Rebuild it: the point of a golden image as code

A pet is repaired. Cattle are replaced. Delete the server and deploy it again from the same Template:

```bash
oc -n student-01-0024-012 delete vm pg-s01-centos10                  # the VM and its three disks
```

Then run **1d.2** again. In a few minutes you have the **same** server at the same address, with an empty `marks`
table: every setting came from code, so nothing drifted and nothing was forgotten.

**That is the end of Demo 1d.** You built a database server purely in code, from Red Hat's maintained image, and
proved it with your own rows.

> 💡 **One copy at a time on an address.** Delete or stop your `pg-s01-centos10` when you're done: its address
> is yours alone.

---

## Addendum: how the lab staged this demo, as code

| CI job | Code | What it does |
|---|---|---|
| `golden-linux` | `automation/customize/golden_linux.sh` | builds the Template and applies it to namespace `openshift` on every site that has the CentOS Stream 10 boot source |
| (same job) | the proof | deploys a test VM from the Template (`ovedr-ci-centos10`, `10.101.<VLAN>.98`), waits for PostgreSQL, writes a mark, checks `chain_ok`, the heartbeat and the guest tools, then deletes the VM. The run's summary shows how long the server took to answer |
| (build) | `automation/tools/centos10_template.sh` | assembles the Template from **one** cloud-init file (`workload/centos10/user-data.yaml`) and the class's SQL (`workload/sql/`), the same schema and toolkit the Windows VMs run. A test fails if the published copy differs from what it builds |

The customer's version of this is a short pipeline: change the cloud-init in git → CI builds the Template, deploys a
test VM, runs its checks → the Template is published for everyone. The base OS comes from Red Hat's boot source,
which OpenShift Virtualization updates on its own; your code stays the same.

**Back to:** [Demo 1](01-vmware-to-ove-cold.md) · [Student guide](../student-guide.md).

## Thank you

- [OpenShift Virtualization: creating VMs from templates and boot sources (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)
- [OpenShift templates](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/images/using-templates) [verify link]
- [cloud-init documentation](https://cloudinit.readthedocs.io/)
- [PostgreSQL 16 documentation](https://www.postgresql.org/docs/16/)
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization (course)](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)

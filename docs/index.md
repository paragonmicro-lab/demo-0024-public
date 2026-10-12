---
title: "Start here — VMware to OpenShift Virtualization"
subtitle: "Thank you for joining · start here · how the class works · the day · Kubernetes in a few ideas"
version: 1.1.0
date: 2026-10-12
---

<div class="pm-hero" markdown>
<p class="pm-kicker">Student guide</p>

# VMware to OpenShift Virtualization

<p>Welcome, and thank you for joining us. Today you move a real Windows + PostgreSQL VM from VMware onto OpenShift
Virtualization, between two clusters and through a disaster-recovery failover, and prove with your own data that nothing
was lost. Everything happens in your web browser: this site is all you need.</p>

[Begin: Lab 0](demos/00-access-and-tools.md){ .md-button .md-button--primary }
[Read the student guide](student-guide.md){ .md-button }
</div>

## Start here

<div class="pm-steps" markdown>
<div markdown>
**1 · Get your login**

Your instructor gives you one username and password. It opens everything: vCenter, both OpenShift clusters, your
Windows VM and its database.
</div>
<div markdown>
**2 · Read the student guide**

The [student guide](student-guide.md) explains how the class works and lists the demos in order, with their times.
</div>
<div markdown>
**3 · Do Lab 0**

[Lab 0](demos/00-access-and-tools.md) takes about 5 minutes: sign in, open a command line, reach your VM's database.
</div>
</div>

!!! tip "Stuck, or something not working? Ask your instructor."
    They can reset your login, catch you up, or show any step on the big screen. Stuck for five minutes means ask. See
    [Get help](help.md) for everything else.

## What you will do today

- Sign in once and reach vCenter and two OpenShift clusters with the same account
- Migrate **your own** Windows + PostgreSQL VM from vSphere to OpenShift Virtualization, cold and warm
- Move it again: to the second cluster, and between clusters
- Prove with your own data that nothing was lost, changed or reordered on any move
- Bonus rounds for early finishers: live migration, snapshot restore, a hostile disaster-recovery failover

## How the class works

- **One VM each:** `win-pg-s<NN>`, a full copy of the instructor's, in your own vCenter folder, at its own address.
- **One namespace each** on both clusters, named your class ID. You are admin there.
- **One password** everywhere.
- **Only a web browser:** nothing to install. Commands run in a terminal inside the OpenShift console.
- **Every step:** **inject** your data → **change** something → **verify** it all arrived. Your **proof table** grows
  through every demo: it is your evidence at the end.
- 🧑‍🏫 marks a step only the instructor runs. **[verify]** marks a step still being proven: if it fails, ask.
- 🎬 every section has a recording of the same steps in the console: watch it, then do it.

## The day

| Block | Time |
|---|---|
| Welcome, and Kubernetes in a few ideas (this page) | 20 min |
| Lab 0 · Demo 1 | 1 h 25 |
| *Break* | 15 min |
| Demo 1b · Demo 3 | 1 h 25 |
| *Lunch* | 45 min |
| Demo 2 (warm) | 1 h |
| Demo 4 (disaster recovery) and bonus rounds | 1 h 30 |
| Wrap-up: your proof table, questions | 15 min |

Times are estimates until the class has been timed. The demos and their bonus rounds, each with its time, are in the
[Student Guide](student-guide.md#the-demos-in-order).

## Kubernetes in a few ideas

**A cluster.** A set of machines (**nodes**) run as one computer. The **control plane** keeps the desired state; the
**worker nodes** run the workloads, your VMs included. One **API** serves everything: the web console, `oc` and
automation. **OpenShift** is Kubernetes plus an installer, a console, security defaults and operators, supported by
Red Hat. Each of our two clusters has 4 nodes.

**You declare it; controllers make it so.** Everything is an **object** in YAML: what you want (`spec`) and what is
true now (`status`). **Controllers** keep comparing the two and act until they match. An **operator** is a controller
with its own object types. Today's: **OpenShift Virtualization** (VMs), **MTV** (migrations), **ODF** (storage),
**NMState** (host networking), **ACM** (many clusters). You don't run "migrate" as a command: you create a migration
`Plan`, and MTV does the work and reports progress in its `status`.

**A VM is a pod.** A **pod** is the smallest thing Kubernetes schedules. OpenShift Virtualization runs each VM's
QEMU/KVM inside a pod (`virt-launcher-<vm>`): `VirtualMachine` is the definition, `VirtualMachineInstance` the running
VM. So VMs and containers share one scheduler, network, storage and set of permissions. Live migration is a new pod on
another node, with the memory copied across.

**Namespaces and RBAC.** A **namespace** groups objects and is where permissions apply, like a vCenter folder with its
own permissions. **RBAC** binds roles (what you may do) to users and groups (who).

**Storage.** A VM disk is a **PersistentVolumeClaim** ("I need 10 GiB like this"). A **StorageClass** says how claims
are met, through a **CSI** driver. Site A runs **ODF** (Ceph, on the lab's SSDs: snapshots, clones and replication
for disaster recovery); site B runs **NFS** for now.

**Networking.** Every pod gets an address on the **pod network**; a **Service** gives a stable name. **NMState**
builds a bridge for a lab VLAN on each node, and a **NetworkAttachmentDefinition** (NAD) puts a VM's NIC on it. So your
migrated VM keeps its address.

## From vSphere words to OpenShift words

| vSphere | OpenShift Virtualization |
|---|---|
| vCenter | the cluster's API and web console |
| ESXi host | node |
| VM / `.vmx` | `VirtualMachine` (running: `VirtualMachineInstance`) |
| folder + permissions | namespace + RBAC |
| datastore | StorageClass |
| VMDK | PersistentVolumeClaim (a DataVolume while importing) |
| port group | NetworkAttachmentDefinition (NAD) |
| vMotion | live migration |
| VM snapshot | `VirtualMachineSnapshot` |
| VMware Tools | QEMU guest agent + virtio drivers |
| *(no equivalent)* | MTV: plans that move VMs into the cluster |

**Ready? Open the [Student Guide](student-guide.md), then [Lab 0](demos/00-access-and-tools.md).**

---

> **Version 1.1.0 · draft · 2026-10-12** · class/welcome.md

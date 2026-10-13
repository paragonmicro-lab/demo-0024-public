---
title: "Student Guide — VMware to OpenShift Virtualization: migrate, move and protect your VM"
subtitle: "Start here · how the class works · the demos in order"
version: 0.8.0
date: 2026-10-12
---

# Student Guide

> **Version 0.8.0 · draft · 2026-10-12** · class/student-guide.md

## What you will learn

You take a real Windows Server + PostgreSQL VM from VMware vSphere onto Red Hat OpenShift Virtualization, cold and then
warm. You move it between two OpenShift clusters and protect it with OpenShift Data Foundation disaster recovery. At
every step you write your own data into the VM's database and prove that none of it was lost, changed or reordered.

This guide is the **map**. The steps are in the demo guides below.

## Your login, and what you need

**The instructor gives you your login.** One student username and one password open everything in this class:

| Resource | Sign in as |
|---|---|
| vSphere Client (vCenter) and both OpenShift consoles | your class ID, with **SSO (Entra)** |
| Your Windows VM | `.\student-NN` |
| Your VM's PostgreSQL database | `student_NN` |

**All you need is a web browser.** Nothing is installed on your computer. You work in the vSphere Client, the two
OpenShift consoles, and **your browser terminal**: a shell in your own pod in the OpenShift console, where every
command in these guides runs. Copy a command here with its copy button; paste it there with **right-click → Paste**.

**[Lab 0](demos/00-access-and-tools.md) walks you through it in about 5 minutes:** open the tabs, sign in to vCenter,
both OpenShift clusters and PostgreSQL, and see your VM and your access.

> 💡 A page stuck after a timeout? Sign in again in a **private window**. Nothing is lost: your VM, your data and your
> terminal live in the lab, not in the browser.

## How the lab is put together

Each class gets its **own** three environments ([lab diagram](demos/lab-diagram.md)):

| Environment | What it is | You |
|---|---|---|
| **vSphere** | vCenter + ESXi, where your VM starts | inspect your VM, enable CBT, shut it down |
| **Site A** | OpenShift + OpenShift Virtualization + MTV, on **ODF** storage | migrate your VM in, run it, snapshot it |
| **Site B** | the same on **NFS** storage, plus the ACM / DR hub | move your VM in, fail over to it |

The zone names (`lz1`, `ocp-lz1`, …) change per booking; the guides use them as examples. Your browser terminal already
knows your own (type `ovedr`).

- **Your own VM**, `win-pg-s<NN>`: a full copy of the instructor's, with its own address, in your own vCenter folder.
  You power it, snapshot it, migrate it and fail it over yourself.
- **Your own namespace** on both clusters, named your class ID, where you are admin. Your VM lands there, and your
  browser terminal lives there.
- **Everything is staged; you do the last step.** Each demo's pieces (operators, maps, a migration plan for your VM)
  are pre-built, and each guide shows how to build them yourself ([Lab S](demos/setup-how-the-lab-is-built.md)). Your
  task is to review and start.

## The demos, in order

Every stage follows one rhythm: **inject** your own data → **change** something → **verify** it all arrived intact.

**Core: the migration class.** Everyone does these, in order.

| # | Guide | You will | Time | Bonus (optional) |
|---|---|---|---|---|
| 0 | [Access and tools](demos/00-access-and-tools.md) | sign in to vCenter, both clusters and PostgreSQL; open a command line | 5 min | — |
| 1 | [Cold migration](demos/01-vmware-to-ove-cold.md) | check your plan and the vCenter link; shut down, migrate, review and power on in OpenShift; inject and read back your rows | 45 min | [1a: snapshot and restore](demos/01a-snapshot-restore.md) (25 min) · [1b: live and storage migration](demos/01b-live-and-storage-migration.md) (20 min) |
| 2 | [Warm migration](demos/02-vmware-to-ove-warm.md) | enable CBT, pre-copy while the VM runs, cut over in moments | 60 min | §2.8 live migration (15 min) |

**Go deeper**, when your instructor offers it:

| Group | Guide | You will | Time | Bonus (optional) |
|---|---|---|---|---|
| Storage | [Demo 4: ODF Regional-DR](demos/04-odf-regional-dr.md) | replicate every 15 minutes, fail over, reverse, come home clean | 90 min | §4.8 hostile failover (30 min) |
| Backup | [Demo 5: OADP backup and restore](demos/05-oadp-vm-backup.md) | back up your namespace, delete your VM, restore it, prove the data | 30 min | *framework: when OADP is staged* |
| More | [Demo 3: OpenShift → OpenShift](demos/03-ove-to-ove-cold.md) | link the clusters in ACM; move your VM to the other cluster, storage and all | 45 min | — |

Reference: [Toolkit](demos/toolkit.md) (your terminal, the data commands, the proof table) · [Lab S](demos/setup-how-the-lab-is-built.md)
(how every staged piece was built, with the commands and the Red Hat docs) · [Lab diagram](demos/lab-diagram.md).

Times are estimates until the class has been timed. Demo 1 starts your **proof table** ([Toolkit](demos/toolkit.md)); each demo adds rows (your last
mark, `chain_ok`, the fingerprint, and each move's outage, bytes and time), and by the end it is your evidence.

**Getting help:** a step marked **[verify]** is still being proven in the lab. If one fails, tell the instructor rather
than improvising. Each section has a 🎬 recording of the same steps in the console.

**Red Hat documentation:**
[OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index) ·
[MTV 2.11](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11) ·
[ODF](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19) ·
[ODF disaster recovery](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19/html/configuring_openshift_data_foundation_disaster_recovery_for_openshift_workloads/index) ·
[ACM](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13) ·
[DO316 training](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization) ·
[Red Hat interactive labs](https://www.redhat.com/en/interactive-labs)

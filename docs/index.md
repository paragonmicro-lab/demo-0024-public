---
title: "Student Guide — VMware to OpenShift Virtualization: migrate, move and protect your VM"
subtitle: "Start here · how the class works · your credentials · the demos in order"
version: 0.6.0
date: 2026-10-12
---

# Student Guide

> **Version 0.6.0 · draft · 2026-10-12** · class/student-guide.md

## What you will learn

In this class you take a real Windows Server + PostgreSQL VM from VMware vSphere onto Red Hat OpenShift
Virtualization: first cold, then warm with minimal downtime. You move it between two OpenShift clusters, and you
protect it with OpenShift Data Foundation disaster recovery: replication, failover, and a clean failback. At every
step you write your own data into the VM's database and then prove that none of it was lost, changed or reordered.
You leave knowing what each tool does, and with the evidence that it worked.

This guide is the **map**. The commands are in the demo guides it points to.

> 🌐 **All you need is a web browser.** You install nothing and need no access to the lab's network from your computer.
> You work in three browser pages: the **vSphere Client**, the **site A** and **site B** OpenShift consoles, plus this
> class site. Every command runs in **your browser terminal**, a shell in your own pod in the OpenShift console (Lab 0
> §0.3), with `psql`, `oc`, `virtctl` and the class commands ready. Copy a command here with its copy button, and
> paste it there with **right-click → Paste**.

---

## 1. How the lab is put together

Each class gets its **own** three environments, built together when the booking starts and removed together when it
ends ([lab diagram](demos/lab-diagram.md)):

| Environment | What it is | Your role there |
|---|---|---|
| **vSphere** `lzN` | VMware vCenter + ESXi, where your VM starts | inspect your VM, enable CBT, shut it down |
| **Site A** `ocp-lzX` | OpenShift + OpenShift Virtualization + MTV, on **ODF** storage | migrate your VM in, run it, snapshot it |
| **Site B** `ocp-lzY` | the same: OpenShift + OpenShift Virtualization + MTV, on **ODF** storage, plus the ACM / DR hub | move your VM in, fail over to it |

Both clusters are built the same way and carry the same operators (OpenShift Virtualization, MTV with its console menu,
ODF, ACM, cert-manager).

> ⚠️ **The zone numbers change from booking to booking.** The scheduler gives each class whichever zones are free,
> so your vSphere zone can be `lz1`, `lz2` or `lz3`, and your two clusters any two of `ocp-lz1`-`ocp-lz3`, in any
> order. The guides show `lz1` / `ocp-lz1` / `ocp-lz2` as **examples only**. **Use the values on your booking card**
> (Lab 0 §0.1) every time.

**You get your own VM.** `win-pg-s<NN>` (for example `student-03` → `win-pg-s03`) is a full copy of the instructor's
`win-pg-i01`: Windows Server 2022, PostgreSQL 16 with its log on `L:` and data on `D:`, a heartbeat writer, and the
data-integrity toolkit. It lives in **your own vCenter folder** (`vms/students/<your class ID>`) at **its own
address** (`.100+NN`; the instructor's is `.62`), and you can power it on and off, open its console, change its
settings (Changed Block Tracking in Demo 2) and snapshot it **at your own pace**. You migrate it, snapshot it and
fail it over yourself.

**You get your own OpenShift namespace.** On **both** clusters you have a namespace (project) named exactly your
class ID in lower case, for example `student-03-0024-005`, the same name as your vCenter folder. You are **admin**
there: full rights, but only in that namespace. Your migration plans move your VM into it, and your VM's database
Service (`<your VM>-sql`) lives there, and so does **your browser terminal** (pod `sql-client`), where you run every
command. Lab 0 sets it up as `$VMNS`.

**Everything is staged; you do the last step.** The lab pre-builds every customization a demo needs: the migration
operator and its connection to vCenter, the network and storage maps, the class namespace, and **a migration plan for
your VM on each cluster, ready but not started** (`d1-<your VM>` on site A, `d1b-<your VM>-to-<B>` on site B). Each
demo shows you what was staged and **how you would build it yourself**, from the command line and in the console,
with links to the Red Hat documentation for every step (that is what **Lab S** is for). Your task is the last step:
review it, then start it.

---

## 2. Signing in: SSO, and keeping your credentials safe

**Sign in with SSO (Entra), everywhere.** On vCenter and on both OpenShift consoles choose the **SSO / Entra** sign-in
and use your class account. Sign in once in your browser and the other two consoles reuse it. For the command line use
the OpenShift console's **Copy login command**. The Windows VM is the one exception: it uses a local login (see
[Your VM: Windows and database logins](#your-vm-windows-and-database-logins)).

**Your credentials:**

1. **Save them** in a password manager or your company's secure notes **before class**. The email you received
   holds your class ID, your password and your booking card.
2. **Never share them**: not with a classmate, not in chat, not in a screenshot, not on a shared screen. Everything
   you do is recorded against your ID.
3. **Treat tokens as passwords.** An `oc login --token=…` line logs in as you.
4. **They expire with the booking.** Accounts and environments are removed when the class ends.
5. **Lost or exposed?** Tell the instructor straight away. They can reset it.

> 💡 **Page stuck after a timeout? Use a private window.** Sometimes a console page or vCenter stops responding after
> your session times out. Typical signs: a refresh brings back the same stale page, you loop back to the login page,
> or you see "unauthorized" even after signing in. The browser has cached your old, expired sign-in.
> Open a **private / incognito window** (Chrome/Edge: `Ctrl+Shift+N`, Firefox: `Ctrl+Shift+P`, Safari:
> `Cmd+Shift+N`) and sign in there with SSO again. If that works, carry on in the private window, or clear the
> site's cookies in your normal window. Your work is not lost: the VM, your data and your browser terminal's sign-ins
> live on the clusters, not in the browser.

Full steps: [Lab 0 §0.0-0.4](demos/00-access-and-tools.md).

### Your VM: Windows and database logins

Your VM has **one password for both**: your class password.

| Where | Login | Password |
|---|---|---|
| Windows (console or RDP) | `.\student-NN` (your number, e.g. `.\student-03`) | your class password |
| PostgreSQL, database `bcdr` | `student_NN` (underscore, e.g. `student_03`) | the same class password |

1. **Open the VM.** vCenter: your VM → **Launch Web Console**. After a move to OpenShift: your namespace →
   **VirtualMachines → your VM → Console**. Or RDP to your VM's address (`.100+NN` on your booking's VLAN).
2. **Sign in to Windows** as `.\student-NN` with your class password. You are an administrator on your own VM.
3. **Open the database.** In PowerShell:
   ```powershell
   & 'C:\Program Files\PostgreSQL\16\bin\psql.exe' -h 127.0.0.1 -U student_NN -d bcdr
   ```
   Enter your class password when asked.
4. **Write and read a row** to prove it works:
   ```sql
   SELECT * FROM ovedr_mark('<your class ID>', 'login-check', 'first row');
   SELECT * FROM ovedr_verify('<your class ID>');      -- chain_ok = t
   ```

From your browser terminal, the class commands (`mark`, `verify`, `alive`) reach the same database as your own
PostgreSQL login: run `dblogin` once (Lab 0 §0.5).

---

## 3. The demos, in order

Each demo guide opens with what you will learn and **how the lab was prepared for it** (what is installed, and the
state your VM starts in), with a pointer into Lab S for every pre-built piece. Every stage follows the same rhythm:
**inject** (a hash-chained row of your own, plus a line in a file in your VM) → **change** something → **verify**
that it all arrived intact.

| # | Guide | You will | Time | Bonus (optional, for early finishers) |
|---|---|---|---|---|
| 0 | [Access and tools](demos/00-access-and-tools.md) | sign in everywhere, open your browser terminal, write and verify your first row | 20 min | §0.6 a pod-network address for your VM (15 min) |
| S | [Lab S — how this lab was built (CLI + console), read alongside each demo](demos/setup-how-the-lab-is-built.md) | every staged piece (operators, bridges, storage, MTV providers and maps, the class namespace, your VM, the staged plans, the healthcheck), each with the commands, the console path and the Red Hat docs | read alongside | — |
| 1 | [VMware → OpenShift, cold](demos/01-vmware-to-ove-cold.md) | inspect your VM, review MTV, networking (VLAN → bridge) and templates; link OpenShift to vCenter; review your staged plan and start it; swap VMware Tools for virtio and the guest agent | 80 min | A: live migration between nodes (15 min) · B: snapshot, drop the database, restore the disks (25 min) |
| 1b | [VMware → both clusters](demos/01b-vmware-to-both-sites.md) | build a plan to site A from the command line; review and start your staged plan to site B in the console | 40 min | — |
| 2 | [VMware → OpenShift, warm](demos/02-vmware-to-ove-warm.md) | enable Changed Block Tracking, pre-copy while the VM runs, cut over in moments | 60 min | §2.8 live migration between nodes (15 min) |
| 3 | [OpenShift → OpenShift](demos/03-ove-to-ove-cold.md) | link the clusters in ACM, move your stopped VM to the other cluster, storage and all | 45 min | — |
| 4 | [ODF Regional-DR](demos/04-odf-regional-dr.md) | replicate every 15 minutes, prove a sync completed, snapshot, fail over, reverse, come home clean | 90 min | §4.8 a hostile failover: break the link, force a failover, resync (30 min) |
| 4b | [Portworx DR](demos/04b-portworx-dr.md) | *framework, to be completed when Portworx is offered* | — | — |

Times are estimates until the class has been run with a stopwatch; they include reading and waiting for copies.

**Your VM keeps its own address after every move.** The staged plans map its network card onto the same VLAN on
OpenShift (a bridge network map), so it answers at the same address on vSphere, site A and site B. Only if you choose
the pod network in a plan you build yourself do you need [Lab 0 §0.6](demos/00-access-and-tools.md), the optional
NIC swap. *(That swap is a lab-only workaround; a real migration keeps the VM's own NIC and address.)*

**Red Hat documentation used across the class:**
[OpenShift Virtualization 4.19](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index) ·
[Migration Toolkit for Virtualization 2.11](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11) ·
[OpenShift Data Foundation 4.19](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19) ·
[ODF disaster recovery](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19/html/configuring_openshift_data_foundation_disaster_recovery_for_openshift_workloads/index) ·
[ACM 2.13](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13) ·
[Kubernetes NMState](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/kubernetes_nmstate/index) [verify link] ·
[DO316 training](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization) ·
[Red Hat interactive labs](https://www.redhat.com/en/interactive-labs)

---

## 4. Your proof table

Lab 0 starts a proof table. Each demo adds rows to it: your last mark, `chain_ok`, the fingerprint, and the outage,
bytes and time of every move. By the end of the class it is your evidence that every migration and every failover
kept your data intact.

## 5. Getting help

- A command marked **[verify]** is still being proven in the lab. If it fails, tell the instructor rather than
  improvising.
- If something breaks mid-demo, the instructor can show the step from the screenshot deck and help you catch up.
- Earlier versions of this guide held every command for one combined flow (M0-M10). That flow now lives in the demo
  guides above, chiefly [Demo 4](demos/04-odf-regional-dr.md) for disaster recovery. Its full text is in this file's
  git history.

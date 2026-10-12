---
title: "Lab diagram — demo-0024 (VMware → OpenShift Virtualization, OVE ⇄ OVE, ODF DR)"
version: 0.3.2
date: 2026-10-12
---

# Lab diagram

> **Version 0.3.2 · draft · 2026-10-12** · class/demos/lab-diagram.md

One booking reserves **three environments** together: a vSphere landing zone (`lzN`) and two OpenShift clusters
(`ocp-lzX` = **site A**, `ocp-lzY` = **site B**). They all run as VMs on the same four physical hosts
(`demosvr01`-`04`), each zone on its own VLAN. The numbers come from your booking (the instructor gives them out) and change from
booking to booking. How each piece was built, from the CLI and in the console: [Lab S](setup-how-the-lab-is-built.md).

![Lab diagram](img/lab-diagram.svg)

<details><summary>Diagram source (Mermaid). The SVG above is rendered from this; edit here, then re-render.</summary>

```mermaid
flowchart TB
  STU(["👩‍🎓 Students: a web browser only<br/>(vSphere Client, both consoles, their browser terminal)<br/>one identity (SSO), one password everywhere"])

  subgraph VS["① vSphere · lzN · VLAN 22+N"]
    direction LR
    VC["vCenter vcenter0N<br/>4 × nested ESXi"]
    VMI["vms/ovedr<br/>win-pg-i01 (instructor) · .62"]
    VMS["vms/students/&lt;class ID&gt;<br/>win-pg-sNN (one per student) · .100+NN<br/>Windows 2022 + PostgreSQL 16<br/>C: OS · L: WAL · D: data"]
    DS[("NFS datastore coresan01_data00<br/>folders ovedr-lzN-*")]
    VC --- VMI --- VMS --- DS
  end

  subgraph A["② Site A · OpenShift ocp-lzX"]
    direction LR
    A1["OpenShift Virtualization · ACM · cert-manager<br/>bridge → NAD vmnet(22+N)"]
    A2["MTV + console menu<br/>providers ovedr-vcenter0N · ovedr-ocp-lzY<br/>staged plans d1-&lt;vm&gt; (not started)"]
    A3[("ODF (4 scratch SSDs)<br/>ceph-rbd-virtualization")]
    A4["one namespace per student (admin)<br/>with their browser terminal sql-client<br/>+ ovedr-app"]
    A1 --- A2 --- A3 --- A4
  end

  subgraph B["③ Site B · OpenShift ocp-lzY"]
    direction LR
    B1["OpenShift Virtualization · ACM · cert-manager<br/>the same bridges / NADs"]
    B2["MTV + console menu<br/>providers ovedr-vcenter0N · ovedr-ocp-lzX<br/>staged plans d1b-&lt;vm&gt;-to-lzY (not started)"]
    B3[("NFS · nfs-csi<br/>(ODF when more SSDs land)")]
    B4["one namespace per student (admin)<br/>with their browser terminal sql-client<br/>+ ovedr-app"]
    B1 --- B2 --- B3 --- B4
  end

  STU --> VS
  STU --> A
  STU --> B
  VS == "Demo 1 cold · Demo 2 warm (CBT)<br/>each VM keeps its own address (bridge map)" ==> A
  VS == "Demo 1b cold" ==> B
  A == "Demo 3 cold OVE → OVE<br/>ODF → NFS storage migration" ==> B
  A -. "Demo 4 ODF Regional-DR · RBD mirror every 15 min<br/>(needs ODF on both sites)" .-> B
  B -. "ACM hub manages site A (BL-D11)" .-> A
```

</details>

**Shared underneath all three:** the four physical hosts `demosvr01`-`04` (one scratch SSD each, which is why only one
site can run ODF today), lab DNS (`dns01`/`dns02`), the NFS export `coresan01_data00` (shared by every vSphere zone,
so every class VM's folder carries its zone: `ovedr-lzN-<vm>`), and core-sso, which provisions the same roster and
password to vCenter, both clusters and the Windows VMs.

## What runs where

| | vSphere `lzN` | Site A `ocp-lzX` | Site B `ocp-lzY` |
|---|---|---|---|
| Built by | demo-vmware (core-web dispatches) | demo-openshift (`ove: cnv`, `storage: odf`, ACM, cert-manager) | demo-openshift (`ove: cnv`, `storage: na`, ACM, cert-manager) |
| Customized by this demo | the instructor's `win-pg-i01` (`vms/ovedr`, `.62`) and **one full clone per student** `win-pg-sNN` in **that student's own folder** `vms/students/<class ID>` (`.100+NN`), each with PostgreSQL (WAL on `L:`, data on `D:`), the toolkit, Windows logins and WinRM; each student may power, open the console, Edit Settings (CBT) and snapshot **their own** VM | MTV with its console menu, providers + maps, **a staged plan `d1-<vm>` per class VM (not started)** landing each student's VM in **that student's own namespace** (their class ID, where they are admin), `ovedr-app` with `sql-client` and the instructor's VM, a Service per VM, the demo prep (pre-copy interval, snapshot class) | the same as A, with **staged plans `d1b-<vm>-to-<lzY>`**, plus the ACM hub role and DR (Demo 4) |
| Storage | NFS datastore `coresan01_data00` | **ODF** on the 4 scratch SSDs (`ocs-storagecluster-ceph-rbd-virtualization`) | **NFS** (`nfs-csi`); ODF once more SSDs are installed |
| Network | port group `vm-network`, VLAN 22+N; every class VM its own static address | NMState bridge onto VLAN 22+N (`vmnet<VLAN>`), plus the pod network | the same VLAN bridges, plus the pod network |
| Demos | 1, 1b, 2 (source) | 1, 2 (target) · 3 (source) · 4 (primary) | 1b (target) · 3 (target) · 4 (DR site, hub) |

The operators are the **same on both clusters**; the one deliberate difference today is storage (one ODF site,
because the lab's four scratch SSDs fit one ODF cluster). The customize healthcheck reports any other difference.

---
title: "Lab 0 — Access and tools (do this first, every session)"
subtitle: "One identity on vCenter and both OpenShift clusters · your own VM · the data-integrity toolkit every demo uses"
version: 0.4.0
date: 2026-10-06
---

# Lab 0: Access and tools

> **Version 0.4.0 · draft · 2026-10-06** · class/demos/00-access-and-tools.md

## What you will learn

You will sign in to all three platforms of the lab (vSphere and two OpenShift clusters) with one class identity
and one password, and set up a terminal that always says which cluster it is talking to. You will find **your own**
Windows Server + PostgreSQL VM and leave your first mark in it: a line in a file on its disk, and a hash-chained row
in its database. Every later demo repeats that pattern (inject, change, verify), so by the end of this lab you can
prove, with one command, that none of your data was lost, changed or reordered.

## How the lab is set up for this demo

The CI customization built all of this before you arrived (see the [lab diagram](lab-diagram.md)). Each row points
at the section of **[Lab S — how this lab was built](setup-how-the-lab-is-built.md)** that shows how to build it yourself, from the CLI and in
the console, with links to the Red Hat documentation.

| Where | What is pre-built and installed | How it was built |
|---|---|---|
| **vSphere** `vcenter0<N>` (demo-vmware) | Nested ESXi cluster with the shared NFS datastore `coresan01_data00`, and port group `vm-network` on VLAN `22+N`. The instructor's VM **`win-pg-i01`** (folder `vms/ovedr`, address `.62`), plus **your own full clone `win-pg-s<NN>`** in **your own vCenter folder `vms/students/<your class ID>`** (address `.100+NN`). Each runs Windows Server 2022 with PostgreSQL 16: WAL on **`L:` (5 GB)**, data on **`D:` (10 GB)**, the 5-second heartbeat writer, and the data-integrity toolkit. Each has a Windows login for every class account. You can power **your** VM on and off, open its console, Edit Settings (CBT in Demo 2) and snapshot it, at your own pace. | [S.11](setup-how-the-lab-is-built.md) · [vSphere VM administration](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-vm-administration/GUID-55238059-912E-411F-A0E9-A7A536972A91.html) [verify link] |
| **Site A** and **site B** (demo-openshift, `ove: cnv`) | **The same operators on both clusters:** OpenShift Virtualization, NMState bridges `br<VLAN>` with NADs `vmnet<VLAN>` (one per lab VLAN), **ACM** and **cert-manager**. **Storage differs, for now:** site A runs **ODF** (`ocs-storagecluster-ceph-rbd-virtualization`, on the lab's 4 scratch SSDs) and site B runs **NFS** (`nfs-csi`): the SSDs fit one ODF cluster today. | [S.2](setup-how-the-lab-is-built.md) · [S.3](setup-how-the-lab-is-built.md) · [S.4](setup-how-the-lab-is-built.md) · [OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index) · [NMState](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/kubernetes_nmstate/index) [verify link] · [ODF](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19) |
| **Site A** and **site B** (this demo's customization) | **MTV** on both tooled sites, with its **Migration for Virtualization** console menu, the vSphere and peer-cluster providers and the network and storage maps. **A migration plan for every class VM, staged but not started**: `d1-<vm>` on site A and `d1b-<vm>-to-<B>` on site B. **Your own namespace on both sites** (your class ID, you are admin there) where your VM lands, with your `Service/<VM>-sql`; the class namespace **`ovedr-app`** with the shared `sql-client` pod. The instructor has their own namespace (the instructor VM lands there) and is cluster-admin. Read access for students to the migration objects, bridges, storage classes and ACM clusters, and the right to run their own plans. | [S.5](setup-how-the-lab-is-built.md)-[S.10](setup-how-the-lab-is-built.md) · [S.12](setup-how-the-lab-is-built.md) · [MTV 2.11](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11) |
| **Accounts** (core-web + core-sso) | One roster. `$ME` signs in with **SSO (Entra)** on vCenter and on both OpenShift consoles (one browser sign-in carries across all three), and the CLIs use a console token, and in Windows (`.\student-NN`). | [Authentication and authorization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/index) |

**You do yourself:** sign in everywhere, set up the CLI contexts and helpers, open your VM's console, and write your
first file line and first database mark.

> **How to read these guides**
> - `$ME` is your class ID, for example `student-03-0024-005`. **One password** works everywhere: vCenter, both
>   OpenShift clusters, and the Windows VM.
> - **Each student has their own VM** (`win-pg-s<NN>`, in your own vCenter folder). You run every step yourself. 🧑‍🏫 marks the few steps only
>   the **instructor** runs (shared infrastructure, or the instructor's demo VM `win-pg-i01`).
> - ✍️ marks a value to write in your **proof table** (end of each guide).
> - 📸 marks a screenshot slot. The instructor fills these in for the handout.
> - **[verify]** marks a command still being proven in lab testing. If one fails, tell the instructor rather than
>   improvising.

---

## 0.0 Your credentials: save them, never share them

<!-- 🎬 video slot 00-0-0-credentials-save-them-never-share-them -->
[![🎬 Video: 0.0 Your credentials: save them, never share them. Watch it, then do the same in the console](../.images/00-0-0-credentials-save-them-never-share-them.jpg)](../.images/00-0-0-credentials-save-them-never-share-them.mp4)
<sub>🎬 *0.0 Your credentials: save them, never share them: a recording of this whole section, done in the console (placeholder until recorded: `class/.images/00-0-0-credentials-save-them-never-share-them.mp4`).*</sub>

You receive your class account by email when the booking is approved. It holds your **class ID** (`$ME`), your
**password**, and the **booking card** values (§0.1).

- **Save them** in a password manager, or the secure note your company uses, **before class starts**. You will
  need the password for SSO and for your Windows VM and its database (one password for both).
- **Never share them**, not with a classmate, in chat, in a screenshot, or on a shared screen. Every action in the
  lab is recorded against your ID. If someone else uses it, their mistakes become yours.
- **Tokens are credentials too.** The `oc login --token=…` line from "Copy login command" logs in as you. Don't
  paste it anywhere but your own terminal.
- **They expire with the booking.** When the class ends, the account and every environment are removed. Nothing
  from this lab is reusable afterwards.
- **Lost or exposed?** Tell the instructor straight away. They can reset it for this booking.

## 0.1 Your booking card

<!-- 🎬 video slot 00-0-1-booking-card -->
[![🎬 Video: 0.1 Your booking card. Watch it, then do the same in the console](../.images/00-0-1-booking-card.jpg)](../.images/00-0-1-booking-card.mp4)
<sub>🎬 *0.1 Your booking card: a recording of this whole section, done in the console (placeholder until recorded: `class/.images/00-0-1-booking-card.mp4`).*</sub>

The instructor gives you a card with four values. Fill them in here. Every command below uses them.

> ⚠️ **The landing zones change from booking to booking.** The scheduler picks whichever zones are free, so your
> vSphere zone can be `lz1`, `lz2` or `lz3`, and your two clusters any two of `ocp-lz1`, `ocp-lz2`, `ocp-lz3`, in any
> order (for example vSphere `lz2` with site A `ocp-lz3` and site B `ocp-lz1`). The examples in these guides use
> `lz1` / `ocp-lz1` / `ocp-lz2`. **Always use the values on your own card**, never the examples.

| Value | Example | Yours |
|---|---|---|
| `ME`: your class ID | `student-03-0024-005` | |
| `N`: vSphere landing zone number | `1` (→ `vcenter01`) | |
| `A`: site A cluster | `ocp-lz1` | |
| `B`: site B cluster | `ocp-lz2` | |

Your VM name and its vSphere address follow from your ID:

| You are | VM (`$VM`) | Address on vSphere (`$VM_IP`) |
|---|---|---|
| `student-NN-…` | `win-pg-sNN` (e.g. `student-03` → `win-pg-s03`) | `10.101.<22+N>.<100+NN>` (e.g. `10.101.23.103`) |
| the instructor | `win-pg-i01` (the instructor's VM, in `vms/ovedr`) | `10.101.<22+N>.62` |

**[verify: BL-D10, per-student clones and their address plan]**

```bash
# ── paste once per terminal, with YOUR values ────────────────────────────────────────────────
export ME=student-03-0024-005
export N=1 A=ocp-lz1 B=ocp-lz2
export DOMAIN=demo.lab-paragonmicro.com
export VC=vcenter0$N.$DOMAIN
export WINUSER=${ME%-*-*}                    # your Windows login: student-03 (Windows caps names at 20 characters)
case "$WINUSER" in
  student-*) NN=${WINUSER#student-}                          # 03
             export VM=win-pg-s$NN                           # your own VM
             export VM_IP=10.101.$((22 + N)).$((100 + 10#$NN))    # its address while it still runs on vSphere
             export VMNS=$(printf '%s' "$ME" | tr '[:upper:]' '[:lower:]') ;;   # YOUR namespace on both clusters
  *)         export VM=win-pg-i01 VM_IP=10.101.$((22 + N)).62 VMNS=$(printf '%s' "$ME" | tr '[:upper:]' '[:lower:]') ;;   # the instructor's VM and namespace
esac
echo "me=$ME  vcenter=$VC  site-a=$A  site-b=$B  vm=$VM ($VM_IP)  namespace=$VMNS  windows=.\\$WINUSER"
```

**Your namespace, `$VMNS`.** On **both** OpenShift clusters you have your own namespace (project), named exactly your
class ID in lower case, for example `student-03-0024-005`. It matches your vCenter folder. You are **admin** there:
full rights, scoped to that namespace only. Your migration plans land your VM there, and your VM's SQL Service
(`$VM-sql`) lives there. The shared `sql-client` pod that the helpers below use stays in the class namespace
`ovedr-app`. Instructors get their own namespace too (their class ID, where `win-pg-i01` lands), are admin in every student
namespace, and are **cluster-admin** on both clusters, so they can help with any student, storage or operator issue.

```bash
oc --context site-a get project "$VMNS"                    # yours, on site A …
oc --context site-b get project "$VMNS"                    # … and on site B
oc --context site-a auth can-i '*' '*' -n "$VMNS"          # yes: admin in your own namespace
```

---

## 0.2 Log in to vCenter

<!-- 🎬 video slot 00-0-2-login-vcenter -->
[![🎬 Video: 0.2 Log in to vCenter. Watch it, then do the same in the console](../.images/00-0-2-login-vcenter.jpg)](../.images/00-0-2-login-vcenter.mp4)
<sub>🎬 *0.2 Log in to vCenter: a recording of this whole section, done in the console (placeholder until recorded: `class/.images/00-0-2-login-vcenter.mp4`).*</sub>

1. Open `https://vcenter0<N>.demo.lab-paragonmicro.com/ui` and choose **Launch vSphere Client**.
2. Choose the **SSO / Entra** sign-in and use your class account. If you already signed in to an OpenShift console in
   this browser, it carries over.
3. Go to **VMs and Templates → datacenter01 → vms → students → `<your class ID>`** and find **your VM**
   (`win-pg-s<NN>`). The instructor's `win-pg-i01` is in **vms → ovedr**.

📸 `00-vcenter-login`: the vSphere Client, with your `students/<class ID>` folder expanded and your VM selected.

**Check:** ✅ you can see your VM, powered on, with three disks (60 GB, 5 GB and 10 GB).

---

## 0.3 Log in to both OpenShift clusters

<!-- 🎬 video slot 00-0-3-login-both-openshift-clusters -->
[![🎬 Video: 0.3 Log in to both OpenShift clusters. Watch it, then do the same in the console](../.images/00-0-3-login-both-openshift-clusters.jpg)](../.images/00-0-3-login-both-openshift-clusters.mp4)
<sub>🎬 *0.3 Log in to both OpenShift clusters: a recording of this whole section, done in the console (placeholder until recorded: `class/.images/00-0-3-login-both-openshift-clusters.mp4`).*</sub>

### Web console (both sites)

| Site | Console |
|---|---|
| Site A | `https://console-openshift-console.apps.<A>.demo.lab-paragonmicro.com` |
| Site B | `https://console-openshift-console.apps.<B>.demo.lab-paragonmicro.com` |

On the login page choose **Entra (SSO)** and sign in with your class account. Use SSO every time: it is how the
class signs in, and it puts the same identity and group (`demo-students`) on both clusters. If SSO is down, tell the
instructor.

📸 `00-ocp-login`: the login page's identity-provider choice, with **Entra (SSO)** highlighted.

### CLI: two named contexts

With SSO, the CLI logs in with a **token** from the console, not a password:

1. In the site A console, open your name (top right) → **Copy login command** → **Display Token**.
2. Copy the `oc login --token=… --server=…` line and run it. Then do the same on site B.

Every later command names the site it talks to (`--context site-a` / `--context site-b`), so you never run a
step against the wrong cluster.

```bash
oc login --token='<site A token>' --server="https://api.$A.$DOMAIN:6443"     # from "Copy login command" on site A
oc config rename-context "$(oc config current-context)" site-a
oc login --token='<site B token>' --server="https://api.$B.$DOMAIN:6443"     # from "Copy login command" on site B
oc config rename-context "$(oc config current-context)" site-b
oc config get-contexts
oc --context site-a whoami && oc --context site-b whoami      # both print $ME
```

**Check:** ✅ you are signed in on vCenter, site A and site B as yourself (`oc whoami` prints your class ID).

---

## 0.4 Log in to your Windows VM

<!-- 🎬 video slot 00-0-4-login-windows-vm -->
[![🎬 Video: 0.4 Log in to your Windows VM. Watch it, then do the same in the console](../.images/00-0-4-login-windows-vm.jpg)](../.images/00-0-4-login-windows-vm.mp4)
<sub>🎬 *0.4 Log in to your Windows VM: a recording of this whole section, done in the console (placeholder until recorded: `class/.images/00-0-4-login-windows-vm.mp4`).*</sub>

Your VM runs Windows Server 2022 with PostgreSQL 16. The WAL (write-ahead log) is on **`L:`** (5 GB) and the data is
on **`D:`** (10 GB).

- **While it is on vSphere:** in vCenter, select your VM and choose **Launch Web Console**.
- **Once it is on OpenShift:** in the console, go to **Virtualization → VirtualMachines → your VM → Console**.

Send **Ctrl-Alt-Del** from the console toolbar, then log in as **`.\<WINUSER>`** (for example `.\student-03`)
with your class password **[verify: BL-D08, student Windows accounts]**.

### Your file in the VM

Open **PowerShell** in the VM and leave your mark on disk, not just in the database:

```powershell
$f = "C:\Users\Public\ovedr\$env:USERNAME.txt"
New-Item -ItemType Directory -Force (Split-Path $f) | Out-Null
Add-Content $f "$(Get-Date -Format o)  lab0-hello"
Get-Content $f
```

Every demo adds a line to this file at each stage. Its last step reads the file back on the target platform.

📸 `00-windows-touch`: PowerShell showing `Get-Content` of your file.

---

## 0.5 The data-integrity toolkit

<!-- 🎬 video slot 00-0-5-data-integrity-toolkit -->
[![🎬 Video: 0.5 The data-integrity toolkit. Watch it, then do the same in the console](../.images/00-0-5-data-integrity-toolkit.jpg)](../.images/00-0-5-data-integrity-toolkit.mp4)
<sub>🎬 *0.5 The data-integrity toolkit: a recording of this whole section, done in the console (placeholder until recorded: `class/.images/00-0-5-data-integrity-toolkit.mp4`).*</sub>

The database `bcdr` has two tables you will use:

| Table | Written by | What it proves |
|---|---|---|
| `heartbeat` | a writer in the VM, every 5 s | the VM is alive. Gaps in it are outages. |
| `marks` | **you**, at every stage | your data survived every move, unchanged and in order |

Each of your `marks` rows carries a SHA-256 hash of the previous row. If a row is lost, changed or reordered,
`ovedr_verify` reports `chain_ok = f` and names the first broken row.

### Helpers (paste once per terminal)

```bash
# where is the database? Your VM keeps its OWN address ($VM_IP) on vSphere AND after a move: the staged plans map
# its NIC onto the same VLAN (bridge map). Only a plan that uses the POD network needs §0.6 and a new PGHOST.
export PGHOST_SITE_A=$VM_IP
sql()    { local ctx=$1; shift
           oc --context "$ctx" -n ovedr-app exec -i deploy/sql-client -- \
             psql -h "${PGHOST:-$PGHOST_SITE_A}" -U bcdr -d bcdr -v ON_ERROR_STOP=1 "$@"; }
mark()   { sql "$1" -c "SELECT * FROM ovedr_mark('$ME', '$2', '${3:-}');"; }            # inject one row
load()   { sql "$1" -c "SELECT * FROM ovedr_load('$ME', '$2', ${3:-100});"; }           # inject a burst
verify() { sql "$1" -c "SELECT * FROM ovedr_verify('$ME');" \
                   -c "SELECT * FROM ovedr_checksum();"; }                             # read + prove
alive()  { sql "$1" -c "SELECT count(*) AS beats, max(ts) AS last_beat,
                        date_trunc('second', clock_timestamp() - max(ts)) AS staleness,
                        pg_postmaster_start_time() AS postgres_started FROM heartbeat;"; }
where()  { for s in site-a site-b; do printf '%-7s ' "$s"
             oc --context $s -n "$VMNS" get vmi "$VM" \
               -o jsonpath='{.status.phase}{"  node="}{.status.nodeName}{"\n"}' 2>/dev/null || echo "-"; done; }
```

> `ovedr_checksum` fingerprints **every** mark in your VM's database. On your own VM that is your rows only. On the
> instructor's `win-pg-i01` it covers everyone who marked it.

### Try it now

```bash
alive  site-a                    # staleness under 10 s: your VM is writing
mark   site-a lab0-hello         # your first row: note its id and sha
verify site-a                    # chain_ok = t, marks = 1
```

📸 `00-verify`: the `verify` output with `chain_ok = t`.

✍️ **Proof table, row 0:** your first mark's `id`, and the `fingerprint` from `ovedr_checksum`.

> 💡 **Why a hash chain?** A count can't tell you that row 7 was replaced, or that rows 7 and 8 swapped places.
> A chain can. A migration or a failover that loses, rewrites or reorders even one of your rows shows up here
> as `chain_ok = f`.

---

## 0.6 (Optional) Give your VM a pod-network address on OpenShift

<!-- 🎬 video slot 00-0-6-optional-give-vm-pod-network-address -->
[![🎬 Video: 0.6 (Optional) Give your VM a pod-network address on OpenShift. Watch it, then do the same in the console](../.images/00-0-6-optional-give-vm-pod-network-address.jpg)](../.images/00-0-6-optional-give-vm-pod-network-address.mp4)
<sub>🎬 *0.6 (Optional) Give your VM a pod-network address on OpenShift: a recording of this whole section, done in the console (placeholder until recorded: `class/.images/00-0-6-optional-give-vm-pod-network-address.mp4`).*</sub>

**You usually skip this.** Every class VM has its **own** static address (instructor `.62`, students `.100+NN`), and
the staged migration plans use the **bridge** network map (`ovedr-vcenter0<N>-net-bridge`), which puts the migrated
NIC on the **same VLAN**. So after a move your VM answers at the same `$VM_IP` as before, and every helper keeps
working.

**Do this step only when your VM landed on the pod network:** a plan you built yourself with the pod map
(`ovedr-vcenter0<N>-net`), or a site without the bridge. A Windows static address on a pod-network NIC answers nothing,
so you swap it for a pod NIC that gets its address by DHCP, and then test over that address.

> ℹ️ **Demo-only, not a production pattern.** Swapping the NIC is a lab workaround for a VM that landed on the pod
> network. In a real migration the VM keeps its own NIC, MAC and address (a bridge map onto the same VLAN, as the staged
> plans do, or a planned re-IP), and nobody swaps NICs. Proper per-VM addressing (DHCP on the lab VLANs) is on the
> backlog (BL-D12). Background: [Virtual machine networking](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/networking) [verify link].

`CTX` is the site your VM is now on (`site-a` after demos 1-2, `site-b` after demo 3).

### 1. Stop the VM

```bash
CTX=site-a
virtctl --context "$CTX" -n "$VMNS" stop "$VM" 2>/dev/null \
  || oc --context "$CTX" -n "$VMNS" patch vm "$VM" --type merge -p '{"spec":{"runStrategy":"Halted"}}'
oc --context "$CTX" -n "$VMNS" get vm "$VM" -w            # Ctrl-C at Stopped
```

### 2. See the migrated interface(s)

```bash
oc --context "$CTX" -n "$VMNS" get vm "$VM" -o json \
  | jq -r '.spec.template.spec.domain.devices.interfaces as $i | .spec.template.spec.networks as $n
           | range(0; $i|length) as $k | "\($k)  \($i[$k].name)  mac=\($i[$k].macAddress // "-")  net=\($n | map(select(.name == $i[$k].name))[0] | (.pod // .multus) | tostring)"'
```

✍️ The index (`0`, `1`, …) of the migrated NIC: ____

### 3. Set the migrated NIC's link down, and add a pod-network NIC

**Console:** **VM → Configuration → Network → ⋮ on the migrated NIC → Set link down** (or *Delete*), then
**Add network interface**: network **Pod networking**, type **Masquerade**.

**CLI:**

```bash
IDX=0                                    # ← the index you wrote down
oc --context "$CTX" -n "$VMNS" patch vm "$VM" --type json -p "[
 {\"op\":\"add\",\"path\":\"/spec/template/spec/domain/devices/interfaces/$IDX/state\",\"value\":\"down\"},
 {\"op\":\"add\",\"path\":\"/spec/template/spec/domain/devices/interfaces/-\",
  \"value\":{\"name\":\"default\",\"masquerade\":{},\"model\":\"virtio\"}},
 {\"op\":\"add\",\"path\":\"/spec/template/spec/networks/-\",
  \"value\":{\"name\":\"default\",\"pod\":{}}}]"
```

**[verify: interface `state: down` (link state) on OpenShift Virtualization 4.19]**. If your cluster rejects
`state`, remove the migrated NIC instead:
`oc --context "$CTX" -n "$VMNS" patch vm "$VM" --type json -p "[{\"op\":\"remove\",\"path\":\"/spec/template/spec/domain/devices/interfaces/$IDX\"}]"`,
then remove the matching entry under `/spec/template/spec/networks/` the same way.

> If the migrated NIC is already named `default` on the pod network (the `…-net` map), it is the same kind of NIC
> and is already unique per VM. Skip step 3 and go to step 4.

### 4. Start the VM and confirm DHCP inside Windows

```bash
virtctl --context "$CTX" -n "$VMNS" start "$VM" 2>/dev/null \
  || oc --context "$CTX" -n "$VMNS" patch vm "$VM" --type merge -p '{"spec":{"runStrategy":"Always"}}'
oc --context "$CTX" -n "$VMNS" get vmi "$VM" -w           # Ctrl-C at Running
```

In the VM console (`.\<WINUSER>`):

```powershell
Get-NetAdapter | Format-Table Name, Status, MacAddress, LinkSpeed
ipconfig /all | Select-String 'Description|Physical|DHCP Enabled|IPv4|Default Gateway'
```

✅ The new adapter shows **DHCP Enabled: Yes**, and an address from OpenShift's masquerade network (the guest
side is `10.0.2.2` **[verify]**).

> The old static IP was configured on the *old* adapter, so the new adapter starts clean on DHCP. If Windows has
> pinned the static IP to the new adapter, reset it:
> `Set-NetIPInterface -InterfaceAlias '<new adapter>' -Dhcp Enabled; ipconfig /renew`.

### 5. Read your VM's address, and use it for every test

The guest's address is private to the VM. From outside, your VM is reached at its **pod IP**:

```bash
export PGHOST=$(oc --context "$CTX" -n "$VMNS" get vmi "$VM" \
  -o jsonpath='{.status.interfaces[?(@.name=="default")].ipAddress}')
echo "PGHOST=$PGHOST"                    # your VM's own address on $CTX
```

The alternative is your VM's **Service**, `Service/$VM-sql`. It selects `vm.kubevirt.io/name=$VM` on port 5432, so
it follows your VM whatever its IP: `export PGHOST=$VM-sql`.

### 6. Connection test

```bash
oc --context "$CTX" -n ovedr-app exec deploy/sql-client -- pg_isready -h "$PGHOST" -p 5432   # accepting connections
alive  "$CTX"
mark   "$CTX" own-address "ip=$PGHOST"
verify "$CTX"
```

✅ `pg_isready` answers *accepting connections*, the heartbeat is fresh, and `chain_ok = t`.

> Windows has no SSH server by default, so the PostgreSQL connection above **is** the connection test. If your
> instructor enables RDP, the same address works: `virtctl --context "$CTX" -n "$VMNS" port-forward "vm/$VM" 3389`,
> then RDP to `localhost:3389` as `.\<WINUSER>` **[verify]**.

📸 `00-own-address`: `ipconfig` on the new adapter beside the `PGHOST` echo.

---

## Proof table (carry it through every demo)

| Row | Stage | Your last mark id | `chain_ok` | Fingerprint | Notes (outage, bytes, time) |
|---|---|---|---|---|---|
| 0 | lab0-hello | | | | |

---

## Thank you

Thank you for setting up your lab. The hardest part of every later demo, knowing for certain that your data
survived, is now one command. To go deeper on the platform you'll use today:

- [Lab S — how this lab was built](setup-how-the-lab-is-built.md): every pre-built piece, from the CLI and in the console
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)
- [OpenShift authentication and authorization (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/index)
- [OpenShift CLI (`oc`)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cli_tools/openshift-cli-oc) [verify link]
- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11)
- [Red Hat OpenShift Virtualization (product overview)](https://www.redhat.com/en/technologies/cloud-computing/openshift/virtualization)
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization (course)](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)
- [Red Hat interactive labs (free, in the browser)](https://www.redhat.com/en/interactive-labs)
- [Red Hat Developer: OpenShift](https://developers.redhat.com/products/openshift/overview)

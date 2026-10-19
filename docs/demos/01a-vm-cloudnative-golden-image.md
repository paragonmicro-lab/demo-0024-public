---
title: "Demo 1a — A cloud-native golden image: a new Linux database server, built from code"
subtitle: "Red Hat's CentOS Stream 10 boot source · the customer's Template and cloud-init · deploy in the console or with one command · inject into the new PostgreSQL"
version: 2.2.0
date: 2026-10-17
---

# Demo 1a: a cloud-native golden image

> **Version 2.2.0 · draft · 2026-10-19** · class/demos/01a-vm-cloudnative-golden-image.md

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
class terminal is signed in to the database (`dblogin`, once). **Do it while Demo 1's copy runs** (started in
[Demo 1 §1.3](01-vmware-to-ove-cold.md#13-start-the-migration)): this demo builds a **new** VM, at its own address,
and **Demos 1b to 1e all use it** while your Windows VM is still copying: snapshot, live and storage migration,
cross-cluster migration (BCDR class) and OADP backup. Commands run in the console's **Web Terminal** (`>_`).

> ⚠ **The commands are an example. Change these values to yours before you run them:**
>
> | In the example | Is the example's | Replace with yours |
> |---|---|---|
> | `student-01-0024-012` | class ID and project | your class ID |
> | `pg-s01-centos10` | the new VM's name | `pg-s<NN>-centos10` |
> | `10.101.23.151` | the new VM's address: VLAN 23, **150 + your student number** | `10.101.<22 + N>.<150 + NN>` |
> | `23` | your booking's VLAN (`22 + N`, `vcenter01` → 23) | from your instructor's `N` |
> | `student-01` / `student_01` | your Linux login / your database login (underscore) | yours |

<!-- 🎬 video slot 01a-vm-cloudnative-golden-image: one walk-through of the whole demo in the console -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/01a-vm-cloudnative-golden-image.jpg">
  <source src="../.images/01a-vm-cloudnative-golden-image.mp4" type="video/mp4">
  <a href="../.images/01a-vm-cloudnative-golden-image.mp4">🎬 Video: Demo 1a: a cloud-native golden image</a>
</video>
<sub>🎬 *Demo 1a: a cloud-native golden image: a walk-through of the whole demo in the console. Watch it, then do the same with the steps below.*</sub>

---

## 1a.1 Look at the golden image: it's code

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
`oc apply` (1a.2).

```bash
oc process --parameters -n openshift ovedr-centos10-postgres
oc get template -n openshift ovedr-centos10-postgres -o yaml | less          # q to quit
```

| In the Template | What it does |
|---|---|
| `dataVolumeTemplates` | the system disk, cloned from Red Hat's `centos-stream10` boot source, plus two **empty** disks: 10 GB for the data, 5 GB for the WAL |
| `networks` | one NIC on your VLAN's bridge (`default/vmnet23`), so the server has its own address on the lab network |
| `cloudInitNoCloud.networkData` | that address, its gateway and the lab's DNS |
| `accessCredentials` | **your public SSH key**, read from a Secret you create (parameter *SSH key Secret*), put on your Linux login at first boot |
| a **Secret** `<VM name>-cloudinit`, read by `cloudInitNoCloud.secretRef` | **cloud-init**, run on first boot: formats and mounts the two disks, installs PostgreSQL 16 with its WAL on its own disk, loads the class's `marks` toolkit, creates your Linux and database logins (SSH takes your key **or** your password), starts the database and a heartbeat |

The whole file is also on the class site: [`ovedr-centos10-postgres.yaml`](../downloads/ovedr-centos10-postgres.yaml).

---

## 1a.2 Deploy it

Two steps: save your public SSH key in OpenShift, then deploy the Template. Each has a console way and a Web
Terminal way, and both ways give the same result.

### Step 1: your SSH key, saved in OpenShift as a Secret

A server built from code gets its SSH key the way a cloud server does: OpenShift holds your **public** key in a
**Secret** in your project, and the Template hands it to the VM on its first boot. Your **private** key never leaves
your terminal.

**Make a key pair in the Web Terminal** (both ways start here, since the class installs nothing on your laptop):

```bash
ssh-keygen -t ed25519 -N '' -C student-01 -f ~/.ssh/id_ed25519      # your key pair: no passphrase, for the lab
cat ~/.ssh/id_ed25519.pub                                            # the PUBLIC key: one line, ssh-ed25519 AAAA… student-01
```

**Console (site A), as in the video:** **Workloads → Secrets → Create → Key/value secret**, in project
**`student-01-0024-012`**:

| Field | Value |
|---|---|
| Secret name | `student-01-ssh` |
| Key | `key` |
| Value | paste the **public** key line from `cat ~/.ssh/id_ed25519.pub` (or **Browse** to upload a `.pub` file) |

Then **Create**. This is the same Secret the VM wizards' **Public SSH key → Add new** dialog saves ("SSH key is saved
in the project as a secret").

**Or, the Web Terminal:**

```bash
oc -n student-01-0024-012 create secret generic student-01-ssh --from-file=key=$HOME/.ssh/id_ed25519.pub
```

✅ `secret/student-01-ssh created`. Check it holds your public key, and only that:

```bash
oc -n student-01-0024-012 get secret student-01-ssh -o jsonpath='{.data.key}' | base64 -d     # ssh-ed25519 AAAA… student-01
```

> 💡 **The Web Terminal's home directory lasts as long as the terminal does.** If it restarts, your private key is
> gone: make a new pair and replace the Secret's value (**Secret → Actions → Edit Secret**). Your password still
> signs you in meanwhile.

### Step 2: deploy the Template

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
| SSH key Secret | `student-01-ssh` (the Secret from step 1) |

> ⚠️ **Check the project before you create.** The VM is created in the project the wizard shows, and a VM can
> **never** change project afterwards: Kubernetes refuses to edit `metadata.namespace` ("Cannot change resource
> namespace"). It must be **your** project, `student-01-0024-012`, next to your Windows VM and your class terminal.
> Built it in the wrong one (`default`, say)? Delete it there and deploy again here: it's code, so that takes
> minutes and loses nothing.

Then **Create VirtualMachine**. Leave the wizard's own **Public SSH key** setting at **None**: the Template already
takes your key from the Secret you named. **[verify: the console's Template form shows the *SSH key Secret*
parameter]**

> ⚠️ **Not the InstanceTypes tab.** **Catalog → InstanceTypes** builds a plain CentOS VM from Red Hat's boot source,
> with none of this Template's code: no PostgreSQL, no data disks, no class logins. Use the **Template catalog**.
> (Its **N series** (`n1.*`) is for network appliances: dedicated CPUs and 1 GiB hugepages that this lab's nodes
> don't offer, so such a VM stays **ErrorUnschedulable**.)

**Or, the Web Terminal: the whole server is one command.** `oc process` fills in the Template's blanks, exactly
as the form did, and `oc apply` creates the result in your project. It asks for your password, so it never
appears on screen:

```bash
read -rsp 'Your class password: ' PW; echo; oc process -n openshift ovedr-centos10-postgres -p NAME=pg-s01-centos10 -p VLAN=23 -p IP=10.101.23.151 -p OS_USER=student-01 -p DB_USER=student_01 -p DB_PASSWORD="$PW" -p SSH_KEY_SECRET=student-01-ssh | oc apply -n student-01-0024-012 -f -; unset PW
```

✅ `secret/pg-s01-centos10-cloudinit created` and `virtualmachine.kubevirt.io/pg-s01-centos10 created`.

> 💡 **Your cloud-init, password included, is in a Secret**, `pg-s01-centos10-cloudinit`, in your project. The
> VM points at it (`cloudInitNoCloud.secretRef`), so the VM's definition holds no password. A Secret is also the
> only way for a real server: KubeVirt refuses cloud-init written inline in the VM above 2 KB, and this one is
> about 11 KB. The Template creates both, so both are yours to delete (1a.5).

<details markdown="1">
<summary>Build it yourself in the console: your own VM, your own cloud-init (optional)</summary>

The Template did the work above. Here you build the same database server **by hand in the console**, from a plain
CentOS Stream 10 VM plus a short **cloud-init**: the standard way a cloud server is customized on its first boot. The
PostgreSQL build itself (install, the class database and toolkit, your database login, the heartbeat) is the lab's
published script, [`ovedr-pg-setup.sh`](../downloads/ovedr-pg-setup.sh): your cloud-init downloads and runs it, so you
write 30 lines, not 300. Read the script first: it is what will run as root on your server.

One address per student: build this **instead of** the Template's server, or delete that one first (1a.5), since
both would use `10.101.23.151`. This server keeps its data on its system disk (the Template's has separate data and
WAL disks); Demos 1b – 1e work on either.

**Console (site A):** **[verify: the field names on OpenShift 4.19's VM wizard]**

1. Set the **Project** dropdown to **your** project, `student-01-0024-012`.
2. **Virtualization → Catalog → InstanceTypes**: boot volume **CentOS Stream 10**; series **U series** (general
   purpose), size **u1.medium**. Not the **N series**: it needs dedicated CPUs and hugepages this lab doesn't have.
3. **VirtualMachine name** `pg-s01-centos10` → **Customize VirtualMachine**.
4. **Network**: edit the NIC → network **`default/vmnet23`** (your VLAN's bridge), type **Bridge**.
5. **Scripts → Cloud-init → Edit**: switch to the **script** (YAML) view and paste the block below, with your values
   in the lines marked ✎. Tick **Add network data** and set it as in the second block (or its fields: Ethernet name
   `enp1s0`, IP address `10.101.23.151/24`, gateway `10.101.23.1`).
6. **Scripts → Public SSH key → Use existing** → `student-01-ssh` (the Secret from step 1 above).
7. **Create VirtualMachine**, then carry on with **1a.3** as for the Template's server.

The cloud-init (user data), about 1.2 KB: the console keeps it inside the VM, where KubeVirt allows at most 2 KB.

```yaml
#cloud-config
# Demo 1a, built yourself: the class's PostgreSQL server from a plain CentOS Stream 10 VM.
# Change the values marked ✎ to yours.
hostname: pg-s01-centos10                     # ✎ your VM's name
system_info:
  default_user:
    name: student-01                          # ✎ your Linux login (your SSH key lands here too)
    groups: [ wheel ]
    sudo: [ "ALL=(ALL) NOPASSWD:ALL" ]
    lock_passwd: false
users: [ default ]
ssh_pwauth: true                              # SSH with your password as well as your key
chpasswd:
  expire: false
  users:
    - { name: student-01, password: "<your class password>", type: text }   # ✎ your login and password
packages: [ postgresql-server, qemu-guest-agent ]
write_files:
  - path: /root/.ovedr-db                     # read once by the setup script, then shredded
    permissions: "0600"
    content: |
      DB_USER=student_01
      DB_PASSWORD=<your class password>
runcmd:
  # the class's PostgreSQL build, published on the class site: read it before you run it
  - curl -fsSL -o /usr/local/sbin/ovedr-pg-setup https://paragonmicro-lab.github.io/demo-0024-public/downloads/ovedr-pg-setup.sh
  - bash /usr/local/sbin/ovedr-pg-setup
```

The network data: your address, its gateway and the lab's DNS.

```yaml
version: 2
ethernets:
  enp1s0:
    addresses: [ 10.101.23.151/24 ]                # ✎ 10.101.<VLAN>.<150 + your number>
    routes: [ { to: 0.0.0.0/0, via: 10.101.23.1 } ]  # ✎ your VLAN's gateway, 10.101.<VLAN>.1
    nameservers: { addresses: [ 10.101.6.201, 10.101.6.202 ] }
```

What each part does, the same jobs the Template's cloud-init does:

| cloud-init | What it does |
|---|---|
| `system_info.default_user`, `users` | makes **your** Linux login the VM's default user, in `wheel` (sudo); your SSH key from the Secret lands on it |
| `ssh_pwauth`, `chpasswd` | SSH and the console accept your password too |
| `packages` | installs PostgreSQL and the QEMU guest agent from the CentOS mirrors |
| `write_files` | your database login and password, in a root-only file the script reads once and then shreds |
| `runcmd` | downloads the class's setup script and runs it; its last line in `/var/log/cloud-init-output.log` is `OVEDR-READY` |

Your password is in the VM's definition this time (cloud-init written in the console stays inline). The Template
kept it in a Secret; for a lab login that dies with the booking, either is fine.

</details>

---

## 1a.3 Watch it build itself

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

**Sign in over SSH, from the Web Terminal:** first with your key, then with your password. Both work, because the
Template turned on both.

```bash
ssh -o StrictHostKeyChecking=accept-new student-01@10.101.23.151 'hostname; whoami'                          # key: no password asked
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password student-01@10.101.23.151 'whoami'       # password: asks for it
```

✅ The first prints `pg-s01-centos10` and `student-01` without asking anything: your key from the Secret is in
`~/.ssh/authorized_keys`. The second asks for your class password, then prints `student-01`. (Also works through
the API server, without reaching the VLAN: `virtctl ssh -n student-01-0024-012 -i ~/.ssh/id_ed25519 student-01@vm/pg-s01-centos10`.)
**[verify: `ssh` and `virtctl` in the Web Terminal tooling]**

---

## 1a.4 Inject into your new Linux PostgreSQL, and read it back

Same commands as on Windows, at the new address. Your class terminal's `dblogin` works here too: same login, same
password, a different server.

```bash
oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.151 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'cloudnative-started', 'Demo 1a: CentOS Stream 10 from the Template');"
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

## 1a.5 Rebuild it: the point of a golden image as code

A pet is repaired. Cattle are replaced. Delete the server and deploy it again from the same Template:

```bash
oc -n student-01-0024-012 delete vm,secret -l app=pg-s01-centos10    # the VM, its three disks and its cloud-init Secret
```

Your SSH key Secret `student-01-ssh` stays (it isn't labelled with the VM), so the rebuild reuses it.

Then run **1a.2** again. In a few minutes you have the **same** server at the same address, with an empty `marks`
table: every setting came from code, so nothing drifted and nothing was forgotten.

**That is the end of Demo 1a.** You built a database server purely in code, from Red Hat's maintained image, and
proved it with your own rows. **Keep it running** (redeploy it if you deleted it above): Demos 1b to 1e use it.

> 💡 **One copy at a time on an address.** `10.101.23.151` is yours alone. When you've finished Demo 1e, go back to
> [Demo 1 §1.4](01-vmware-to-ove-cold.md#14-review-your-vm-on-openshift-then-power-it-on) and **stop**
> `pg-s01-centos10` there (no need to delete it).

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

**Next:** [Demo 1b: snapshot and restore](01b-snapshot-restore.md), on this server · **Back to:** [Demo 1](01-vmware-to-ove-cold.md) · [Student guide](../student-guide.md).

## Thank you

- [OpenShift Virtualization: creating VMs from templates and boot sources (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)
- [OpenShift templates](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/images/using-templates) [verify link]
- [cloud-init documentation](https://cloudinit.readthedocs.io/)
- [PostgreSQL 16 documentation](https://www.postgresql.org/docs/16/)
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization (course)](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)

---
title: "Lab 0 — Access and tools (5 minutes)"
subtitle: "Sign in to vCenter and both OpenShift clusters · open a command line · reach the cluster and your VM's database"
version: 1.1.0
date: 2026-10-13
---

# Lab 0: Access and tools

> **Version 1.1.0 · draft · 2026-10-13** · class/demos/00-access-and-tools.md

**About 5 minutes.** You sign in to the lab in your browser, open a command line, and prove you can reach the cluster
and your own VM's database. One recording covers the whole lab: watch it, then do the same with the steps below.

<!-- 🎬 video slot 00-access-and-tools -->
<video controls preload="metadata" playsinline width="100%" poster="../.images/00-access-and-tools.jpg">
  <source src="../.images/00-access-and-tools.mp4" type="video/mp4">
  <a href="../.images/00-access-and-tools.mp4">🎬 Video: Lab 0: Access and tools</a>
</video>
<sub>🎬 *Lab 0: Access and tools: a recording of the whole lab, done in the browser (3 min 47 s). Watch it, then do the same with the steps below.*</sub>

**You need** your class ID and password from the instructor (one login for everything), and your booking's three
addresses. The instructor gives you `N`, `A` and `B`; the examples below use `1`, `ocp-lz1` and `ocp-lz2`.

**Your names, by example.** The examples in this lab are for **`student-01`** on booking **`0024-012`**. Swap in your own:

| What | Example | Where it comes from |
|---|---|---|
| Class ID (also your project) | `student-01-0024-012` | your sign-in name, the part before `@` |
| Your VM | `win-pg-s01` | `s` + your student number |
| Your VM's address | `10.101.23.101` | vCenter, step 0.1 |
| Database login | `student_01` | your student number with an **underscore**, not a hyphen, and not your email address |

| Open in a browser tab | Address |
|---|---|
| vCenter | `https://vcenter0<N>.demo.lab-paragonmicro.com/ui` |
| Site A console | `https://console-openshift-console.apps.<A>.demo.lab-paragonmicro.com` |
| Site B console | `https://console-openshift-console.apps.<B>.demo.lab-paragonmicro.com` |

---

## 0.1 Sign in to vCenter, and note your VM's address

1. Open the vCenter tab, choose **Launch vSphere Client**, then the **SSO / Entra** sign-in with your class account.
2. Go to **VMs and Templates → datacenter01 → vms → students → `<your class ID>`** and select your VM,
   **`win-pg-s<NN>`** (for example `student-03` → `win-pg-s03`).
3. On its **Summary** tab, find **IP Addresses**. ✍️ **Write it down:** this is your VM's address (for example
   `10.101.23.103`). You use it in step 0.4, and your VM keeps it after every move.

✅ Your VM is powered on, with three disks (60 GB, 5 GB and 10 GB).

---

## 0.2 Sign in to both OpenShift consoles

1. Open the **site A** console tab and choose **Entra (SSO)**. Your vCenter sign-in carries over.
2. Do the same in the **site B** console tab.
3. In each, **Home → Projects** shows your own project, named your class ID (for example `student-01-0024-012`). You
   are admin there.

---

## 0.3 Open a command line

The command line is `oc`, signed in **as you** with a token you copy from the console:

1. In the **site A** console, open your name (top right) → **Copy login command** → **Display Token**.
2. Copy the line that starts with `oc login --token=… --server=…`.

You can run it in either of two places.

### Option 1: `oc` on your own computer (outside this class)

If your company allows it, install the `oc` client yourself: [Getting started with the OpenShift
CLI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cli_tools/openshift-cli-oc#cli-getting-started)
(the console's **? → Command Line Tools** has the download). **Installing it is outside this class's scope:** the
class needs only your browser. Paste the copied line into your terminal and press Enter.

### Option 2: a command line inside OpenShift (the class way)

Nothing to install. Use one of:

- **The console's Web Terminal:** the **`>_`** icon at the top right of the console opens a terminal already signed
  in as you. Click **Start** (leave the project as it is). It has `oc`; for your VM's database you use the one-line `oc exec` commands in 0.4.
- **Your class terminal:** in your project, **Workloads → Pods → `sql-client-…` → Terminal**. It has `oc` and the
  class commands (`dblogin`, `mark`, `sql`, `verify`) ready, and already knows your VM's address and your database
  login. Sign its `oc` in: type `oclogin` and a space, paste the copied line, and press Enter.

> ⚠ **Not the Windows VM's command prompt.** Everything in this lab runs in one of the two terminals above.

> 📋 **Copy and paste:** paste with **right-click → Paste** (or Ctrl+Shift+V, ⌘V on a Mac); copy by selecting the text,
> then **right-click → Copy**. **[verify: per browser]**

---

## 0.4 Test: the cluster, then your VM's database

**The cluster:**

```bash
oc whoami            # your class ID, e.g. student-01-0024-012@lab-paragonmicro.com
oc get nodes         # the cluster's nodes, all Ready
```

**Your VM's database**, PostgreSQL inside your Windows VM, at **your VM's address from 0.1** (the `-h` below). From `oc`
anywhere (the Web Terminal, or your own computer), each step is **one line**: `oc exec` runs it inside your class
terminal (`sql-client`).

> ⚠ **Steps 1–4 are an example. Change two values to yours before you run them:**
>
> | In the example | Is the example's | Replace with yours |
> |---|---|---|
> | `student-01-0024-012` | class ID (`student-01` on booking `0024-012`) | your class ID, e.g. `student-03-0024-012` (appears in every line, and twice in steps 2 and 4) |
> | `10.101.23.101` | address of `win-pg-s01`, the Windows VM running the SQL server | your VM's address, from vCenter in 0.1 |

1. **Sign in to the database**, once (enter your class password):

   ```bash
   oc exec -it -n student-01-0024-012 deploy/sql-client -- dblogin
   ```

2. **Inject** a row: the class has started:

   ```bash
   oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_mark('student-01-0024-012', 'start-class', 'Lab 0: signed in');"
   ```

3. **Read back** every row you have posted, oldest first:

   ```bash
   oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT id, ts, stage, payload FROM marks ORDER BY id;"
   ```

4. **Prove** them: no row lost, changed or reordered (one summary line, `chain_ok = t`):

   ```bash
   oc exec -n student-01-0024-012 deploy/sql-client -- sql -h 10.101.23.101 -c "SELECT * FROM ovedr_verify('student-01-0024-012');"
   ```

> 💡 If a step says *First sign in to your VM's database … dblogin*, your class terminal restarted: do step 1 again.

✅ `oc get nodes` lists the nodes Ready, the read-back lists your `start-class` row, and `verify` shows `chain_ok = t`.

---

## Done

You are signed in to vCenter and both OpenShift clusters, you have a command line, and you reached your own VM's
database. **Go back to the [Student Guide](../student-guide.md)** and continue with the next module, Demo 1.

Every demo uses the class commands for the same inject, read back and prove (`mark`, `verify`), plus `load` and
`alive`: they are in the [Toolkit](toolkit.md), which each demo points to when you need it.

## Thank you

- [OpenShift CLI (`oc`)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cli_tools/openshift-cli-oc)
- [OpenShift authentication and authorization (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/index)
- [OpenShift Virtualization documentation (4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)
- [Red Hat interactive labs (free, in the browser)](https://www.redhat.com/en/interactive-labs)

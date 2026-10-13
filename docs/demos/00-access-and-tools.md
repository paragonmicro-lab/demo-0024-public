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
[![🎬 Video: Lab 0: Access and tools. Watch it, then do the same in the console](../.images/00-access-and-tools.jpg)](../.images/00-access-and-tools.mp4)
<sub>🎬 *Lab 0: Access and tools: a recording of the whole lab, done in the browser (placeholder until recorded: `class/.images/00-access-and-tools.mp4`).*</sub>

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
  in as you. Click **Start** (leave the project as it is). It has `oc`, but **not** `psql` or the class commands.
- **Your class terminal:** in your project, **Workloads → Pods → `sql-client-…` → Terminal**. It has `oc`, `psql`
  and the class commands (`dblogin`, `mark`, `sql`, `verify`) ready, and already knows your VM's address and your
  database login. Sign its `oc` in: type `oclogin` and a space, paste the copied line, and press Enter.

> ⚠ **Not the Windows VM's command prompt.** `psql` isn't on its path; everything in this lab runs in one of the two
> terminals above.

> 📋 **Copy and paste:** paste with **right-click → Paste** (or Ctrl+Shift+V, ⌘V on a Mac); copy by selecting the text,
> then **right-click → Copy**. **[verify: per browser]**

---

## 0.4 Test: the cluster, then your VM's database

**The cluster:**

```bash
oc whoami            # your class ID, e.g. student-01-0024-012@lab-paragonmicro.com
oc get nodes         # the cluster's nodes, all Ready
```

**Your VM's database**, PostgreSQL inside your Windows VM. From `oc` anywhere (the Web Terminal, or your own computer),
run `psql` inside your class terminal, with your class ID, your VM's address (0.1) and your database login. Example:

```bash
oc exec -it -n student-01-0024-012 deploy/sql-client -- psql -h 10.101.23.101 -U student_01 -d bcdr
```

Enter your class password. At the `bcdr=>` prompt, **inject** your first row (the class has started), **read it back**,
then **prove** your rows. Use your class ID where the example has `student-01-0024-012`:

```sql
SELECT * FROM ovedr_mark('student-01-0024-012', 'start-class', 'Lab 0: signed in');   -- inject
SELECT id, ts, stage, payload FROM marks ORDER BY id;                                  -- read back
SELECT * FROM ovedr_verify('student-01-0024-012');                                     -- chain_ok = t
\q
```

Every demo does the same at each stage (`before-…`, `after-…`), so you watch your rows grow as your VM moves.

> 💡 `no pg_hba.conf entry … user "student-01"` means you typed a hyphen: the database login is `student_01`. In your
> class terminal (0.3) the same three steps are short commands: `dblogin` once, then `mark start-class`, then `verify`.

✅ `oc get nodes` lists the nodes Ready, and reading back your rows shows `start-class`, with `chain_ok = t`.

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

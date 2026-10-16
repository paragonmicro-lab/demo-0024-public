---
title: "Lab S — How this lab was built: every operator and customization, by command line and by console"
subtitle: "What CI staged for you · how to check it · how you would build it yourself (CLI and console) · where Red Hat documents it"
version: 0.7.4
date: 2026-10-15
---

# Lab S: How this lab was built

> **Version 0.7.4 · draft · 2026-10-16** · class/demos/setup-how-the-lab-is-built.md

## What you will learn

Before your class started, a CI pipeline built three environments and then customized them: operators, networks,
storage, migration providers and maps, a namespace with a database client, one Windows + PostgreSQL VM per student,
and a staged migration plan for each VM. This guide walks through every one of those pieces in the order CI builds
them. For each piece you see what was built, how to check it, how to build it yourself from the **command line**, and
how to do the same in the **console**, with links to the Red Hat documentation that describes it. The demo guides only
ask you to do the **last step** yourself (start your migration). This guide is how you would do everything before it.

## How the lab is set up

| Built by | What | Where |
|---|---|---|
| **demo-vmware** (core-web dispatches it) | vCenter `vcenter0<N>`, 4 nested ESXi hosts, the shared NFS datastore `coresan01_data00` (the class VMs live there, or on **vSAN** when the booking sets `vmware_storage: vsan`), port group `vm-network` on VLAN 22+N, your vCenter folder `vms/students/<your ID>` | vSphere `lzN` |
| **demo-openshift** (core-web dispatches it, twice) | OpenShift 4.19, OpenShift Virtualization (`ove: cnv`), ACM and cert-manager, NMState bridges + NADs per VLAN, `nfs-csi` storage; **ODF on site A only** | site A `<A>` and site B `<B>` |
| **this demo's CI** (`customize`, sections S.5–S.12) | MTV, providers, maps, the class namespace, the class VMs, demo prep, the staged plans | all three |
| **you** | start your own migration plan (Demo 1 / 1b), then everything in the demo guides | |

Picture: [lab diagram](lab-diagram.md). Every command below uses your class terminal's variables (`$A`, `$B`, `$N`, `$VM`, `$DOMAIN`; [Toolkit T.1](toolkit.md))
and the contexts `site-a` / `site-b`. Most "do it yourself" steps need **cluster-admin**. In class you only **read**
these objects; the instructor (or a cluster you own) can run the build steps.

```bash
export VLAN=$((22 + N))                 # the vSphere zone's VLAN: also the bridge your migrated VM can join
```

---

## S.1 The three environments, and what both clusters should look like

**What CI built:** core-web booked one vSphere zone and two OpenShift clusters together. Both clusters are built from the
**same** member selections (`ove: cnv`, ACM, cert-manager), so they carry the same operators. The one difference is
storage: the lab has four scratch SSDs, enough for **one** ODF cluster, so site A runs ODF and site B uses NFS (`nfs-csi`).
The healthcheck compares the two sites' operators and reports any difference (ODF excluded).

**Check:**

```bash
for s in site-a site-b; do echo "== $s"; oc --context $s get csv -A --no-headers \
  | awk '{print $2}' | sed 's/\.v\{0,1\}[0-9].*$//' | sort -u; done
oc --context site-a get storageclass; oc --context site-b get storageclass
```

**Console:** **Operators → Installed Operators**, project *All Projects*, on each site.
📸 `S-operators-site-a` · 📸 `S-operators-site-b`: the two installed-operator lists side by side.

**Red Hat docs:**
- [OpenShift Container Platform 4.19 documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19)
- [Operators: understanding OLM and installed operators](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/operators/index)
- [About OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/about)

---

## S.2 OpenShift Virtualization (built by demo-openshift)

**What was built:** the **OpenShift Virtualization** operator (`kubevirt-hyperconverged`) in `openshift-cnv`, and its
`HyperConverged` resource `kubevirt-hyperconverged`. That resource deploys KubeVirt, CDI (the disk importer), the
templates and the boot sources.

**Check:**

```bash
oc --context site-a -n openshift-cnv get hyperconverged kubevirt-hyperconverged \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}{"\n"}'      # True
oc --context site-a -n openshift-cnv get csv | grep -i kubevirt
```

**Do it yourself (CLI):**

```bash
oc apply -f - <<'YAML'
apiVersion: v1
kind: Namespace
metadata: { name: openshift-cnv }
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata: { name: kubevirt-hyperconverged-group, namespace: openshift-cnv }
spec: { targetNamespaces: [ openshift-cnv ] }
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: { name: hco-operatorhub, namespace: openshift-cnv }
spec: { source: redhat-operators, sourceNamespace: openshift-marketplace, name: kubevirt-hyperconverged, channel: stable }
YAML
# once the CSV says Succeeded:
oc apply -f - <<'YAML'
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata: { name: kubevirt-hyperconverged, namespace: openshift-cnv }
spec: {}
YAML
```

**Console:**
1. **Operators → OperatorHub** → search *OpenShift Virtualization* → **Install** (namespace `openshift-cnv`).
   📸 `S-cnv-operatorhub`
2. **Installed Operators → OpenShift Virtualization → Create HyperConverged** → **Create**.
   📸 `S-cnv-hyperconverged`
3. A **Virtualization** menu appears in the left navigation. 📸 `S-cnv-menu`

**Red Hat docs:**
- [Installing OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/installing)
- [Preparing your cluster for OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/installing#preparing-cluster-for-virt) [verify link]
- [OpenShift Virtualization architecture](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/about#virt-architecture) [verify link]

---

## S.3 Networking: VLANs to bridges to network attachments

**What was built (demo-openshift):** the **Kubernetes NMState** operator, and on every node a Linux bridge
`br<VLAN>` on the trunk NIC for each lab VLAN. Each bridge is a `NodeNetworkConfigurationPolicy` (NNCP) named
`br<VLAN>-policy`. In `default` there is a `NetworkAttachmentDefinition` (NAD) `vmnet<VLAN>` that lets a VM plug into
that bridge. The VLAN is the same one the vSphere port group `vm-network` uses, so a migrated VM can **keep its address**.

| vSphere | OpenShift |
|---|---|
| port group `vm-network`, VLAN 22+N | NNCP `br<22+N>-policy` → bridge `br<22+N>` → NAD `default/vmnet<22+N>` |

**Check:**

```bash
oc --context site-a get nncp                                                   # br<VLAN>-policy … Available
oc --context site-a get nncp "br${VLAN}-policy" -o jsonpath='{.status.conditions[?(@.type=="Available")].status}{"\n"}'
oc --context site-a -n default get net-attach-def "vmnet${VLAN}" -o jsonpath='{.spec.config}{"\n"}' | jq .
```

**Do it yourself (CLI):** install *Kubernetes NMState Operator* from OperatorHub, create its `NMState` instance, then:

```bash
TRUNK='<trunk NIC on the nodes, e.g. ens224>'          # [verify] the trunk interface name on your nodes
oc apply -f - <<YAML
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata: { name: br${VLAN}-policy }
spec:
  desiredState:
    interfaces:
      - name: ${TRUNK}.${VLAN}
        type: vlan
        state: up
        vlan: { base-iface: ${TRUNK}, id: ${VLAN} }
      - name: br${VLAN}
        type: linux-bridge
        state: up
        ipv4: { enabled: false }
        bridge: { options: { stp: { enabled: false } }, port: [ { name: ${TRUNK}.${VLAN} } ] }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata: { name: vmnet${VLAN}, namespace: default }
spec:
  config: '{"cniVersion":"0.3.1","name":"vmnet${VLAN}","type":"bridge","bridge":"br${VLAN}","macspoofchk":true}'
YAML
```

**Console:**
1. **Networking → NodeNetworkConfigurationPolicy → Create** (form or YAML), as above. 📸 `S-nncp-create`
2. Wait for **Available** on every node. 📸 `S-nncp-available`
3. **Networking → NetworkAttachmentDefinitions → Create** in `default`: type *Linux bridge*, bridge name `br<VLAN>`.
   📸 `S-nad-create`

**Red Hat docs:**
- [Kubernetes NMState operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/kubernetes_nmstate/index) [verify link]
- [Connecting a VM to a Linux bridge network](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/networking#virt-connecting-vm-to-linux-bridge) [verify link]
- [Multiple networks: NetworkAttachmentDefinition](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/index) [verify link]

---

## S.4 Storage: whatever the booking chose, and snapshot classes

**Storage is not this demo's choice.** demo-openshift deploys each cluster's storage from its `storage` option (`na` = NFS,
`odf`; other storage providers such as Portworx are forks of this class, BL-D25) and marks the class VMs belong on. CI's customize only **reads** that class
(the `storageclass.kubevirt.io/is-default-virt-class` one, else the cluster default) and proves a test claim binds before
any VM lands there. The demos work the same on any of them: they are about migration and BCDR.

```bash
# the class your VMs land on, per site (what CI's sc_live reads)
for s in site-a site-b; do
  oc --context $s get sc -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubevirt\.io/is-default-virt-class=="true")]}{.metadata.name}{"\n"}{end}'
done
```

**What this booking's build has today:**
- **Site A**: OpenShift Data Foundation (ODF), Ceph on one scratch SSD per host. VMs use the storage class
  `ocs-storagecluster-ceph-rbd-virtualization` (block, RWX, so live migration works). Built by demo-openshift
  (`storage: odf`); core-web reserves the SSDs for the booking.
- **Site B**: **the same** on the lab standard (`storage: odf`, ODF on the second SSD per host). A booking with
  `odf-site-a` gives site B only `nfs-csi` on the lab NFS server, and has no Demo 4.
- **Both sites** also have `nfs-csi` (the cluster's default class; VMs default to the ODF virtualization class).
- **Snapshot classes** come with the storage, from demo-openshift: `nfs-csi-snapclass` (driver `nfs.csi.k8s.io`) and
  ODF's own. Demo 1's snapshot bonus needs one. CI's `demo-prep` only reports a VM storage driver without one.

**Check:**

```bash
oc --context site-a get storageclass ocs-storagecluster-ceph-rbd-virtualization
oc --context site-a -n openshift-storage get storagecluster      # Ready
oc --context site-b get storageclass nfs-csi
for s in site-a site-b; do oc --context $s get volumesnapshotclass; done
```

**Do it yourself (CLI):** a snapshot class for a driver that has none (what demo-openshift creates for NFS):

```bash
oc --context site-b apply -f - <<'YAML'
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata: { name: nfs-csi-snapclass }
driver: nfs.csi.k8s.io
deletionPolicy: Delete
YAML
```

ODF itself: install the **OpenShift Data Foundation** operator, label the storage nodes, and create a
`StorageSystem` / `StorageCluster` on the local SSDs (the Local Storage operator discovers them).

**Console:**
1. **Operators → OperatorHub → OpenShift Data Foundation → Install.** 📸 `S-odf-install`
2. **Storage → Data Foundation → Create StorageSystem** → *Create a new StorageClass using local storage devices*.
   📸 `S-odf-storagesystem`
3. **Storage → StorageClasses**: see `ocs-storagecluster-ceph-rbd-virtualization` (site A) and `nfs-csi` (site B).
   📸 `S-storageclasses`
4. **Storage → VolumeSnapshotClasses → Create.** 📸 `S-snapclass`

**Red Hat docs:**
- [Deploying ODF using local storage devices](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19/html/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index) [verify link]
- [Storage for virtual machines](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/storage) [verify link]
- [CSI volume snapshots](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/using-container-storage-interface-csi#persistent-storage-csi-snapshots) [verify link]

---

## S.5 The Migration Toolkit for Virtualization (MTV) operator

**Who built it:** the booking's `migration_tools` decides which sites get MTV (`both`: site A and site B; `site-a`: site B is
left bare so students install MTV there). On those sites **demo-openshift** installs it (`ove: cnv-mtv`), and CI
(`ansible/roles/mtv_operator`) checks it and customizes it for the class:
- *(demo-openshift)* Namespace **`openshift-mtv`**, OperatorGroup **`migration`** (targets `openshift-mtv`), and Subscription
  **`mtv-operator`**: channel **`release-v2.11`**, source `redhat-operators`. CI waits for the CSV to report **Succeeded**.
- *(demo-openshift)* **`ForkliftController/forklift-controller`** with `feature_ui_plugin`, `feature_validation`,
  `feature_volume_populator` and `feature_must_gather_api` all `"true"`. *(CI)* adds `controller_precopy_interval: "5"` for
  Demo 2 and waits for `deploy/forklift-controller` to be Available.
- *(demo-openshift; CI re-checks)* MTV's **console plugin** enabled in `consoles.operator.openshift.io/cluster` → `spec.plugins`.
  This is what adds the **Migration for Virtualization** menu.
- Class access: RoleBinding `ovedr-students-view` (ClusterRole `view`) and Role + RoleBinding
  `ovedr-students-migrate`. The second lets students create, start and delete **plans and migrations**, and only read
  providers, maps and hosts. It never grants secrets.

**Check:**

```bash
oc --context site-a -n openshift-mtv get subscription mtv-operator -o jsonpath='{.spec.channel}{"\n"}'   # release-v2.11
oc --context site-a -n openshift-mtv get csv | grep mtv
oc --context site-a -n openshift-mtv get forkliftcontroller,deploy/forklift-controller
oc --context site-a get console.operator cluster -o jsonpath='{.spec.plugins}{"\n"}'                    # includes forklift…
oc --context site-a auth can-i create plans.forklift.konveyor.io -n openshift-mtv                        # yes (students)
```

**Do it yourself (CLI):**

```bash
oc apply -f - <<'YAML'
apiVersion: v1
kind: Namespace
metadata: { name: openshift-mtv }
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata: { name: migration, namespace: openshift-mtv }
spec: { targetNamespaces: [ openshift-mtv ] }
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: { name: mtv-operator, namespace: openshift-mtv }
spec: { name: mtv-operator, channel: release-v2.11, source: redhat-operators, sourceNamespace: openshift-marketplace }
YAML
oc -n openshift-mtv get csv -w                                  # wait for Succeeded, then:
oc apply -f - <<'YAML'
apiVersion: forklift.konveyor.io/v1beta1
kind: ForkliftController
metadata: { name: forklift-controller, namespace: openshift-mtv }
spec:
  feature_ui_plugin: "true"
  feature_validation: "true"
  feature_volume_populator: "true"
  feature_must_gather_api: "true"
YAML
PLUGIN=$(oc get consoleplugin -o name | grep forklift | cut -d/ -f2); echo "$PLUGIN"
oc patch console.operator cluster --type json -p "[{\"op\":\"add\",\"path\":\"/spec/plugins/-\",\"value\":\"$PLUGIN\"}]"
```

**Console:**
1. **Operators → OperatorHub** → *Migration Toolkit for Virtualization Operator* → **Install**, channel `release-v2.11`,
   namespace `openshift-mtv`. 📸 `S-mtv-operatorhub`
2. **Installed Operators → MTV → ForkliftController → Create.** 📸 `S-mtv-forkliftcontroller`
3. Accept **"Web console update is available"** and refresh. **Migration for Virtualization** appears in the menu.
   📸 `S-mtv-menu`

**Red Hat docs:**
- [Installing and using the Migration Toolkit for Virtualization 2.11](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/index) [verify link]
- [MTV prerequisites](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites_mtv) [verify link]
- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11)

---

## S.6 The vSphere side: an MTV account, its role, and the vSphere provider

**What CI built** (`automation/customize/mtv_fabric.sh`):
- On vCenter, an SSO user **`ovedr-mtv@vsphere.local`**, with a minted password stored in the class vault and never
  shown, and a role **`ovedr-mtv`** with MTV's minimal privilege set:
  - power on/off and guest control;
  - all of *Provisioning*;
  - create/remove snapshot;
  - **ChangeTracking** (CBT, for warm migration);
  - datastore browse and file management;
  - session validation;
  - Cryptographer decrypt/access.

  It is granted on the vCenter root (`/`), propagating. CI checks the user can then list the inventory.
- A **network probe** run from inside each cluster before any provider: vCenter 443, every ESXi host on 443 and **902**
  (MTV's disk copy), and the peer cluster's `api-int` 6443 and ingress 443.
- On each tooled site, Secret + **`Provider/ovedr-vcenter0<N>`** (type `vsphere`, url `https://vcenter0<N>…/sdk`). It
  carries `user`, `password`, `url` and **`cacert`**: the certificate chain vCenter serves, so TLS is verified, never skipped.
- **VDDK: retired (2026-10-14).** The lab has no VDDK image, so its migrations are **cold** (Demo 1). Warm migration
  (Demo 2) is coming soon with **storage copy offload** on a SAN array instead; why the lab dropped VDDK:
  [Demo 2](02-vmware-to-ove-warm.md#why-this-lab-does-not-demo-vddk). [S.15](#s15-vddk-the-one-piece-the-customer-supplies-broadcom-licence) is kept for reference.

**Check:**

```bash
oc --context site-a -n openshift-mtv get provider "ovedr-vcenter0$N" \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}  {.spec.url}{"\n"}'
oc --context site-a -n openshift-mtv get provider "ovedr-vcenter0$N" -o jsonpath='{.spec.settings}{"\n"}'   # vddkInitImage?
```

**Do it yourself (CLI):** the vSphere role and user (govc, as a vCenter administrator), then the provider:

```bash
export GOVC_URL="https://vcenter0$N.$DOMAIN" GOVC_USERNAME='administrator@vsphere.local'   # GOVC_PASSWORD from your vault
govc role.create my-mtv VirtualMachine.Interact.PowerOff VirtualMachine.Interact.PowerOn \
  VirtualMachine.State.CreateSnapshot VirtualMachine.State.RemoveSnapshot VirtualMachine.Config.ChangeTracking \
  Datastore.Browse Datastore.FileManagement Sessions.ValidateSession   # … plus Provisioning.*, GuestControl, Cryptographer.*
govc sso.user.create -p '<password>' my-mtv        # the one place a password goes on argv: govc has no other way
govc permissions.set -principal my-mtv@vsphere.local -role my-mtv -propagate=true /

CA="$(openssl s_client -showcerts -servername "vcenter0$N.$DOMAIN" -connect "vcenter0$N.$DOMAIN:443" </dev/null 2>/dev/null \
      | sed -n '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/p')"
oc -n openshift-mtv create secret generic my-vcenter --from-literal=user=my-mtv@vsphere.local \
  --from-literal=password='<password>' --from-literal=url="https://vcenter0$N.$DOMAIN/sdk" --from-literal=cacert="$CA" \
  --from-literal=insecureSkipVerify=false
oc -n openshift-mtv label secret my-vcenter createdForProviderType=vsphere createdForResourceType=providers
oc apply -f - <<YAML
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata: { name: my-vcenter, namespace: openshift-mtv }
spec: { type: vsphere, url: "https://vcenter0$N.$DOMAIN/sdk", secret: { name: my-vcenter, namespace: openshift-mtv } }
YAML
```

**Console:**
1. vSphere Client: **Administration → Roles → New** (privileges as above); **Single Sign On → Users and Groups → Add**;
   **Hosts and Clusters → vCenter → Permissions → Add**. 📸 `S-vsphere-role` · 📸 `S-vsphere-permission`
2. OpenShift: **Migration for Virtualization → Providers → Create Provider → VMware**: URL, user, password, and either
   *fetch certificate* or paste the CA. 📸 `S-mtv-provider-vsphere`
3. The provider shows **Ready**, with VM and host counts. 📸 `S-mtv-provider-ready`

**Red Hat docs:**
- [VMware prerequisites and privileges for MTV](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites_mtv#vmware-prerequisites_mtv) [verify link]
- [Adding a VMware vSphere source provider](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/migrating-vmware_vmware) [verify link]
- [Creating a VDDK image](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites_mtv#creating-vddk-image_mtv) [verify link]

---

## S.7 OpenShift ⇄ OpenShift providers

**What CI built:** on **both** sites, ServiceAccount **`ovedr-mtv-peer`** in `openshift-mtv` (bound to `cluster-admin`,
MTV's documented simplest grant for an OpenShift source; the cluster exists only for the booking) and a long-lived token
Secret `ovedr-mtv-peer-token`. On each tooled site, a Provider **`ovedr-ocp-<peer>`** (type `openshift`) whose `url`
and `cacert` both come from the **peer's kubeconfig**: the URL is `https://api-int.<peer>…:6443` and the CA is the
cluster's internal CA.

> 💡 **Why `api-int` and not `api`?** After demo-openshift's customize, `api.<cluster>` serves a public Let's Encrypt
> certificate. `api-int` keeps the cluster's internal CA. URL and CA must come from the same place, or TLS verification
> fails (`x509: certificate signed by unknown authority`). That happened in this lab, and it is why CI reads both from one
> kubeconfig.

**Check:**

```bash
oc --context site-b -n openshift-mtv get provider "ovedr-ocp-${A#ocp-}" \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}  {.spec.url}{"\n"}'
```

**Do it yourself (CLI):** on the source cluster, create the service account and token. On the destination, create the provider.

```bash
oc --context site-a -n openshift-mtv create sa my-peer
oc --context site-a adm policy add-cluster-role-to-user cluster-admin -z my-peer -n openshift-mtv
oc --context site-a -n openshift-mtv apply -f - <<'YAML'
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata: { name: my-peer-token, namespace: openshift-mtv, annotations: { kubernetes.io/service-account.name: my-peer } }
YAML
TOKEN="$(oc --context site-a -n openshift-mtv get secret my-peer-token -o jsonpath='{.data.token}' | base64 -d)"
URL="https://api-int.$A.$DOMAIN:6443"                  # match the CA below
oc --context site-a config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > peer-ca.pem
oc --context site-b -n openshift-mtv create secret generic my-ocp-a --from-literal=token="$TOKEN" \
  --from-literal=url="$URL" --from-file=cacert=peer-ca.pem --from-literal=insecureSkipVerify=false
oc --context site-b -n openshift-mtv label secret my-ocp-a createdForProviderType=openshift createdForResourceType=providers
oc --context site-b apply -f - <<YAML
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata: { name: my-ocp-a, namespace: openshift-mtv }
spec: { type: openshift, url: "$URL", secret: { name: my-ocp-a, namespace: openshift-mtv } }
YAML
unset TOKEN; rm -f peer-ca.pem
```

**Console:** site B, **Migration for Virtualization → Providers → Create Provider → OpenShift Virtualization**: URL,
service-account token, CA certificate. 📸 `S-mtv-provider-ocp`

**Red Hat docs:**
- [Adding an OpenShift Virtualization source provider](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/migrating-virt_cnv) [verify link]
- [Using service accounts](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/understanding-and-creating-service-accounts) [verify link]

---

## S.8 Network maps and storage maps

**What CI built**, per tooled site, in `openshift-mtv`:

| Map | Source → destination | Why |
|---|---|---|
| `ovedr-vcenter0<N>-net` | vSphere `vm-network` → **pod** network | works everywhere; the VM gets a new, cluster-internal address |
| `ovedr-vcenter0<N>-net-bridge` | vSphere `vm-network` → **multus** `default/vmnet<22+N>` | the VM stays on its VLAN and **keeps its own address**. Created only when the NAD's bridge is **Available** |
| `ovedr-vcenter0<N>-storage` | datastore `coresan01_data00`, plus the **vSAN** datastore on a `vmware_storage: vsan` booking → **this site's** class | site A: `ocs-storagecluster-ceph-rbd-virtualization`; site B: `nfs-csi` |
| `ovedr-ocp-<peer>-net` | pod → pod, plus each VLAN NAD present on **both** sites → the same NAD | OVE ⇄ OVE |
| `ovedr-ocp-<peer>-storage` | **the peer's** class → **this** site's class | OVE ⇄ OVE converts ODF ⇄ NFS: the disks are copied into the destination's storage |

**Check:**

```bash
oc --context site-a -n openshift-mtv get networkmap,storagemap
oc --context site-a -n openshift-mtv get storagemap "ovedr-vcenter0$N-storage" -o jsonpath='{.spec.map}{"\n"}' | jq .
```

**Do it yourself (CLI):**

```bash
P="ovedr-vcenter0$N"
oc --context site-a apply -f - <<YAML
apiVersion: forklift.konveyor.io/v1beta1
kind: NetworkMap
metadata: { name: my-net-bridge, namespace: openshift-mtv }
spec:
  provider: { source: { name: $P, namespace: openshift-mtv }, destination: { name: host, namespace: openshift-mtv } }
  map: [ { source: { name: vm-network }, destination: { type: multus, name: vmnet${VLAN}, namespace: default } } ]
---
apiVersion: forklift.konveyor.io/v1beta1
kind: StorageMap
metadata: { name: my-storage, namespace: openshift-mtv }
spec:
  provider: { source: { name: $P, namespace: openshift-mtv }, destination: { name: host, namespace: openshift-mtv } }
  map: [ { source: { name: coresan01_data00 }, destination: { storageClass: ocs-storagecluster-ceph-rbd-virtualization } } ]
YAML
```

**Console:** **Migration for Virtualization → Network maps → Create** (source network → *Multus: default/vmnet<VLAN>*);
**Storage maps → Create** (datastore → storage class). 📸 `S-networkmap` · 📸 `S-storagemap`

**Red Hat docs:**
- [Creating network and storage mappings](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/index) [verify link: the mappings chapter]
- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11)

---

## S.9 Namespaces: one per student (admin), plus the class namespace `ovedr-app` (both sites)

**What CI built** (`ansible/roles/class_namespace`) on **both** sites:
- **One namespace per student**, named exactly the student's class ID in lower case (e.g. `student-01-0024-006`, the
  same name as their vCenter folder `vms/students/<class ID>`). Their VM migrates **there**.
  - The student is **`admin`** there: full rights, scoped to that namespace only. That is one RoleBinding
    (`ovedr-student-admin-<hash>`) per identity the student can log in as: the htpasswd user (their class ID) and, with
    SSO, their UPN.
  - The instructors' group `demo-instructors` is `admin` there too (`ovedr-instructors-admin`).
  - Their VM's Service **`<vm>-sql`** (selector `vm.kubevirt.io/name: <vm>`, port 5432).
  - **Their browser terminal**, Deployment **`sql-client`** + ConfigMap **`ovedr-tools`** (see *The browser terminal*
    below). Instructors get one in their own namespace too, pointed at `win-pg-i01`.
- **Instructors:** their own namespace (their class ID; the instructor VM `win-pg-i01` lands in the first instructor's), and
  **cluster-admin** on both clusters (`ClusterRoleBinding/ovedr-instructor-cluster-admin-*`, one per identity), so they can
  help with any student, storage or operator issue. CLI: `oc adm policy add-cluster-role-to-user cluster-admin <user>`; console:
  **User Management → RoleBindings → Create binding** (cluster-wide, role `cluster-admin`). [Using RBAC](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/using-rbac) [verify link]
- The class namespace **`ovedr-app`**: the shared `sql-client`. It holds RoleBinding
  `ovedr-students-edit` (ClusterRole `edit`, group `demo-students`), the instructor VM's Service `win-pg-i01-sql`, Secret
  **`bcdr-db`** (the `bcdr` database password, from the vault), and Deployment **`sql-client`**
  (`registry.redhat.io/rhel9/postgresql-16`), the instructor's tool as the database owner (`bcdr`): the healthcheck
  reads every VM's heartbeat through it, and the instructor uses it for owner-only steps (Demo 1a). Students
  never need it.
- ClusterRole + ClusterRoleBinding **`ovedr-class-read`**: students may read NNCPs and node network states, storage
  classes, snapshot classes, ACM managed clusters and cluster sets, DR policies and clusters, and nodes.

**Check:**

```bash
for s in site-a site-b; do
  oc --context $s get project "$VMNS"                                     # your namespace exists on both sites
  oc --context $s -n "$VMNS" get rolebinding,svc                          # your admin binding(s) and your VM's Service
  oc --context $s -n "$VMNS" get deploy/sql-client cm/ovedr-tools           # your browser terminal and its commands
done
oc --context site-a auth can-i '*' '*' -n "$VMNS"                         # yes: admin in your own namespace
oc --context site-a auth can-i create pods/exec -n "$VMNS"                  # yes: the console's Terminal tab needs it
oc --context site-a auth can-i list nodenetworkconfigurationpolicies      # yes (class read)
```

**Do it yourself (CLI):** a namespace for a student with admin rights, and the VM's Service:

```bash
NS=student-99-demo                         # a student's namespace = their class ID, lower case
U=student-99-demo                          # the user they log in as (add a second binding for their SSO UPN)
oc new-project "$NS" --display-name="$U: your VM migrates here"
oc adm policy add-role-to-user admin "$U" -n "$NS"                        # full rights, only in this namespace
oc adm policy add-role-to-group admin demo-instructors -n "$NS"           # instructors too
oc -n "$NS" create service clusterip "$VM-sql" --tcp=5432:5432 --dry-run=client -o yaml \
  | sed "s/app: $VM-sql/vm.kubevirt.io\/name: $VM/" | oc apply -f -
```

The class namespace, by hand:

```bash
oc new-project my-app
oc -n my-app create rolebinding students-edit --clusterrole=edit --group=demo-students
oc -n my-app create secret generic bcdr-db --from-literal=PGPASSWORD='<bcdr password>'
oc -n my-app create deployment sql-client --image=registry.redhat.io/rhel9/postgresql-16 -- sleep infinity
oc -n my-app set env deployment/sql-client --from=secret/bcdr-db
```

**Console:**
1. **Home → Projects → Create Project**: the name is the student's class ID. 📸 `S-namespace`
2. That project → **Project access** (or **User Management → RoleBindings → Create binding**): subject **User** = the
   student, role **admin**; add the group `demo-instructors` as **admin**. 📸 `S-project-access`
3. **Networking → Services → Create Service** in that project (selector `vm.kubevirt.io/name`). 📸 `S-service`
4. In `ovedr-app`: **Workloads → Secrets → Create** and **Workloads → Deployments → Create** for `sql-client`.

### The browser terminal (one per student and instructor, both sites)

The class runs **entirely in the browser** (operator, 2026-10-12): students are external, may not install anything on
their computers, and have no network path to the lab. So every command runs in a pod of their own, opened from the
console's **Terminal** tab (the console's `exec`, which `admin` in their namespace allows).

**What CI built**, in every student's and instructor's namespace (`ansible/roles/class_namespace`):
- ConfigMap **`ovedr-tools`**: the class commands (`dblogin`, `mark`, `load`, `verify`, `alive`, `sql`, `oclogin`,
  `where`, `ovedr`), mounted at `/opt/ovedr/bin` (mode 0755). Source: `ansible/roles/class_namespace/files/ovedr-tools/`.
- Deployment **`sql-client`** (`rhel9/postgresql-16`, `sleep infinity`), with:
  - init containers that copy **`oc`** from the cluster's own `openshift/cli` image, fetch **`virtctl`** from
    OpenShift Virtualization's in-cluster download service, and fetch **`jq`** (pinned, checksum-verified);
  - its environment preset per person: `ME`, `VM`, `VM_IP`, `VMNS`, `WINUSER`, `N`, `A`, `B`, `VC`, `SITE`, and
    `PGHOST` (the VM's own address), `PGUSER` (`student_NN`), `PGDATABASE=bcdr`;
  - **no password** anywhere: `dblogin` asks for the student's class password once and keeps it in the pod
    (`~/.pgpass`, 0600); `oclogin` takes the token from the console's **Copy login command**, so `oc` and `virtctl`
    act **as the student**. The pod has no service-account token (`automountServiceAccountToken: false`).
- A change to the commands rolls the pod (annotation `ovedr/tools-sha`). The healthcheck drives student-01's terminal
  on both sites, as the student: `dblogin`, `alive`, `verify`, and `oc`/`virtctl`/`jq` present.

**Check:**

```bash
oc --context site-a -n "$VMNS" get deploy/sql-client -o jsonpath='{.status.availableReplicas}{"\n"}'   # 1
ovedr                                                                    # in the Terminal: who, where, what is signed in
```

**Do it yourself (CLI):** the same idea for any namespace (here without the class commands):

```bash
oc -n "$NS" create deployment toolbox --image=registry.redhat.io/rhel9/postgresql-16 -- sleep infinity
oc -n "$NS" set env deployment/toolbox PGHOST=<VM address> PGUSER=<login> PGDATABASE=bcdr
```

**Console:** **Workloads → Pods → `sql-client-…` → Terminal**. 📸 `S-browser-terminal`

**Red Hat docs:**
- [Accessing a pod's terminal from the web console](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/building_applications/odc-viewing-application-composition-using-topology-view) [verify link]
- [Logging in to the OpenShift CLI with a web console token](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cli_tools/openshift-cli-oc#cli-logging-in-web_cli-developer-commands) [verify link]

**Red Hat docs:**
- [Working with projects](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/building_applications/projects) [verify link]
- [Using RBAC to define and apply permissions](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/using-rbac) [verify link]
- [Default cluster roles (admin, edit, view)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/using-rbac#default-roles_using-rbac) [verify link]
- [Exposing a VM by using a Service](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/networking#virt-exposing-vm-with-service) [verify link]

---

## S.10 Demo preparation

**What CI built** (`demo-prep` job, `ansible/roles/demo_prep` and the play after it):
- **MTV pre-copy interval**: `ForkliftController.spec.controller_precopy_interval: 5` (minutes), so a warm migration
  (Demo 2) shows several pre-copies instead of one per hour **[verify: field name on MTV 2.11]**.
- **Snapshot classes**: see S.4.
- **ACM**: if site B is an ACM hub and site A is **not**, CI creates a `ManagedCluster` for site A with an
  `auto-import-secret` (site A's kubeconfig; ACM deletes it after use) and waits for **Available**. ⚠ demo-openshift
  now installs ACM on **both** clusters. A cluster that is itself a hub cannot simply be imported into another hub, so CI
  **skips** the import and the readiness table says so (BL-D11, an open decision).
- A **per-demo readiness table** in the run summary: which demos this booking can run (for example, Demo 2 needs a SAN array with copy offload).

**Check:**

```bash
oc --context site-a -n openshift-mtv get forkliftcontroller forklift-controller -o jsonpath='{.spec}{"\n"}' | jq .
oc --context site-b get managedclusters                       # local-cluster, and site A if imported
```

**Do it yourself (CLI):**

```bash
oc -n openshift-mtv patch forkliftcontroller forklift-controller --type merge -p '{"spec":{"controller_precopy_interval":5}}'
oc --context site-b apply -f - <<YAML
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata: { name: $A, labels: { cloud: auto-detect, vendor: auto-detect } }
spec: { hubAcceptsClient: true }
YAML
# then, in namespace $A on the hub: Secret auto-import-secret with autoImportRetry and site A's kubeconfig (or token + server)
```

**Console:** ACM: **All Clusters → Infrastructure → Clusters → Import cluster** (name, *Kubeconfig* or *Server URL and API
token*). 📸 `S-acm-import` · 📸 `S-acm-clusters`

**Red Hat docs:**
- [Importing a managed cluster (ACM 2.13)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/clusters/cluster_mce_overview#import-intro) [verify link]
- [Red Hat Advanced Cluster Management 2.13 documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13)
- [Warm migration and pre-copy settings](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11) [verify link: the warm-migration section]

---

## S.11 The class VMs on vSphere

**What CI built** (`source_vm.sh`, `student_vms.sh`, `lib/vm.sh`, `windows/*.ps1`), entirely through vCenter's API (govc)
and VMware guest operations:
- **Instructor VM `win-pg-i01`** in `vms/ovedr`: Windows Server 2022 (BIOS, LSI Logic SAS, e1000e), installed unattended
  from the lab ISO plus an answer ISO (VMware Tools included). Three thin disks: **C:** 32 GB, **L:** 256 MB (PostgreSQL **WAL**,
  label `pg_wal`, `max_wal_size = 96MB`) and **D:** 1 GB (PostgreSQL **data**, label `pgdata`); small, so a copy moves little. Static address `10.101.<22+N>.62`.
  - PostgreSQL 16 runs as service `postgresql-16`, with `PGDATA=D:\pgdata` and WAL at `L:\pg_wal`.
  - The `bcdr` database holds the heartbeat writer (a startup task) and the **integrity toolkit**
    (`ovedr_mark/load/verify/checksum`).
  - Each roster account has a Windows login (`.\student-NN`).
  - WinRM is enabled over HTTPS (5986, lab networks only) so Ansible can manage the VM.
- **One VM per student, `win-pg-s<NN>`**: a full clone of the instructor VM's snapshot `ovedr-golden`, 2 vCPU / 4 GB.
  - Re-addressed to `10.101.<22+N>.<100+NN>` with its network card **disconnected**, so it never appears with the
    instructor's address.
  - Renamed, with its own WinRM certificate. Its student is a local administrator.
  - Moved into **`vms/students/<your ID>`**, where you hold vCenter rights on it (role `ovedr-student-vm`: power,
    console, Edit Settings, snapshots).
- Every VM is validated **in the guest** before you get it (`Test-SourceVm.ps1`: volumes, the PostgreSQL layout, the
  heartbeat, the toolkit, logins, the address, WinRM).
- On the shared datastore every VM's folder carries its zone (`ovedr-<zone>-<name>`), so classes on different zones never collide.

**Check:**

```bash
alive; verify                                                # toolkit commands: your VM answers and your chain is intact
```

In vCenter: **VMs and Templates → datacenter01 → vm → vms → students → <your ID> → `win-pg-s<NN>`** → Summary,
Configure → Hardware (three disks). 📸 `S-vm-in-student-folder` · 📸 `S-vm-disks`

**Do it yourself (CLI, condensed from the CI scripts):**

```bash
export GOVC_URL="https://vcenter0$N.$DOMAIN"                 # credentials from your vault
F=/datacenter01/vm/vms/ovedr; POOL="/datacenter01/host/cluster-lz$N/Resources"; ISO_DS=coresan01_data00
DS=coresan01_data00            # the VM's datastore: on a vmware_storage=vsan booking, the vSAN one (govc datastore.info)
govc vm.create -on=false -folder "$F" -pool "$POOL" -ds "$DS" -c 4 -m 8192 -g windows2019srvNext_64Guest \
  -firmware bios -net vm-network -net.adapter e1000e -disk.controller lsilogic-sas -disk 32GB \
  -iso '<path of the Windows ISO>' -iso-datastore "$ISO_DS" my-win-pg
govc vm.disk.create -vm my-win-pg -ds "$DS" -name my-win-pg/log  -size 256MB
govc vm.disk.create -vm my-win-pg -ds "$DS" -name my-win-pg/data -size 1GB
# … install Windows + PostgreSQL (initdb -D D:\pgdata -X L:\pg_wal), then clone one per student:
govc snapshot.create -vm my-win-pg -m=false golden
govc vm.clone -vm my-win-pg -snapshot golden -link=false -on=false -folder "$F" -pool "$POOL" -ds "$DS" -c 2 -m 4096 my-win-pg-s01
govc object.mv "$F/my-win-pg-s01" '/datacenter01/vm/vms/students/<student ID>'
```

**Console (vSphere Client):**
1. **New Virtual Machine** → create → guest *Windows Server 2022*; add two hard disks (256 MB, 1 GB). 📸 `S-vsphere-newvm`
2. Install Windows and PostgreSQL. In **Disk Management**, give `L:` (pg_wal) and `D:` (pgdata). 📸 `S-windows-disks`
3. **Snapshots → Take Snapshot**; **Clone → Clone to Virtual Machine** from it. 📸 `S-vsphere-clone`
4. Drag the clone into **vms → students → <ID>**; **Permissions → Add** for that student. 📸 `S-vsphere-move`

**Red Hat docs (what MTV needs from the VM):**
- [VMware prerequisites: the source VM](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites_mtv#vmware-prerequisites_mtv) [verify link]
- [Windows virtual machines on OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/creating-a-virtual-machine) [verify link]

---

## S.12 The staged migration plans: your task is to start yours

**What CI built** (`automation/customize/stage.sh`), for **every** class VM (`win-pg-i01` and every `win-pg-s<NN>`):

| Site | Plan | Moves |
|---|---|---|
| site A and site B | **`d1-<vm>`** | vSphere → that site, cold (Demo 1) |
| site A and site B | **`d2-<vm>`** | vSphere → that site, **warm** (Demo 2): lands as `<vm>-warm`. It waits on Changed Block Tracking, which the student enables in Demo 2 (staged and reported, never blocking) |

Each plan targets **its VM's owner's namespace**: a student's VM lands in that student's own namespace (their class ID,
`$VMNS`), the instructor's `win-pg-i01` in `ovedr-app`. It uses the **bridge** network map when it exists (your VM keeps its own address), plus the
site's storage map. CI waits until every plan is **Ready**: MTV found the VM and the maps validate. **No plan is started.**
That is the one step left to you (Demo 1, Demo 2). Both sites are staged the same (since 2026-10-16), but **the demos
start site A's plan only**: both sites share the VM's VLAN, so starting a plan on both would bring up two copies of
your VM at one address. Your VM reaches site B from site A in Demo 1c, with a plan you build. Each demo's own addendum shows its objects as code. A plan that has started is never touched again.

**Check:**

```bash
oc --context site-a -n openshift-mtv get plan "d1-$VM" \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}{"\n"}'     # True
oc --context site-a -n openshift-mtv get plan "d1-$VM" -o yaml | sed -n '/^spec:/,/^status:/p'
```

**Your task: start it (CLI):**

```bash
oc --context site-a apply -f - <<YAML
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata: { name: d1-$VM-run, namespace: openshift-mtv }
spec: { plan: { name: d1-$VM, namespace: openshift-mtv } }
YAML
```

**Your task: start it (console):** **Migration for Virtualization → Migration plans → `d1-<your VM>` → Start**, and
confirm. 📸 `S-plan-list` · 📸 `S-plan-start`

**How CI created it (so you can create one yourself):**

```bash
oc --context site-a apply -f - <<YAML
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata: { name: my-plan, namespace: openshift-mtv }
spec:
  warm: false
  targetNamespace: $VMNS                    # your own namespace
  provider:
    source:      { name: ovedr-vcenter0$N, namespace: openshift-mtv }
    destination: { name: host,             namespace: openshift-mtv }
  map:
    network: { name: ovedr-vcenter0$N-net-bridge, namespace: openshift-mtv }
    storage: { name: ovedr-vcenter0$N-storage,    namespace: openshift-mtv }
  vms: [ { name: $VM } ]
YAML
```

Console: **Migration plans → Create plan** → source provider → select your VM → target project **your own** (`<your class ID>`) → network and
storage maps → **Cold** → **Create**. 📸 `S-plan-create`

**Red Hat docs:**
- [Creating and running a migration plan](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/migrating-vmware_vmware) [verify link]
- [Cold and warm migration](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/about-mtv_mtv) [verify link]

---

## S.13 What "ready" means: the healthcheck

The booking turns **READY** only when CI's last job passes every check, against the live systems:

| Check | Proves |
|---|---|
| vCenter answers an admin API session; every ESXi host CONNECTED + POWERED_ON; the instructor VM is on | vSphere is up and responsive |
| each cluster: `/readyz` ok, every node Ready, no ClusterOperator degraded or unavailable, the console answers 200 | both OpenShift clusters are up and responsive |
| operator sets compared between the sites (ODF excluded) | the two sites are built the same (differences are reported) |
| **`win_ping`** over WinRM to every class VM | each VM answers as an Ansible-managed host |
| every VM's heartbeat is fresh, read over the network from `sql-client` | PostgreSQL is up and writing in every VM |
| the roster's instructor and student-01 log in on vCenter and both clusters with one password | one identity everywhere |
| every MTV provider Ready on the tooled sites (a bare site has no MTV, and does have OpenShift Virtualization + storage) | migration is ready |

**Red Hat docs:**
- [Verifying cluster health](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/support/troubleshooting) [verify link]
- [ansible.windows.win_ping](https://docs.ansible.com/ansible/latest/collections/ansible/windows/win_ping_module.html)

---

## S.14 How the two clusters are joined: ACM, Submariner, then ODF disaster recovery

Two clusters become one DR pair in **three layers**, and each layer needs the one before it. Demo 1c (moving a VM between
clusters) needs only MTV's providers (S.7). Demo 4 (ODF Regional-DR, async replication) needs all three.

```
  layer 1  ACM        one hub MANAGES both clusters        ManagedCluster, ManagedClusterSet
  layer 2  Submariner the clusters' pod/service networks   submariner add-on, Broker, Globalnet,
           can reach each other                            ServiceExport of ODF's provider server
  layer 3  ODF DR     Ceph RBD mirroring between the two   ODF Multicluster Orchestrator: MirrorPeer,
           ODF clusters, driven from the hub               DRCluster, DRPolicy; per app: DRPlacementControl
```

### What the lab builds (lab standard since 2026-10-15)

| | site A (`ocp-<A>`) | site B (`ocp-<B>`, the hub) |
|---|---|---|
| ACM | MultiClusterHub **2.16**, booked `acm: managed`: it does not manage itself, site B's hub does | MultiClusterHub **2.16**: **the hub**; manages itself as `local-cluster` |
| ODF | **ODF 4.19**, `ocs-storagecluster`, 1 scratch SSD per node | **the same**: ODF on both sites is the booking default (`storage: odf`) |
| Pod / service networks | dual-stack: `10.128.0.0/14`, `fd02::/48` / `172.30.0.0/16`, `fd03::/112` | ⛔ **the same today.** Globalnet can only translate the IPv4 half, so Submariner can't link the sites until each site has its own ranges (demo-openshift, 2026-10-16). The planned ranges don't overlap at all, so Submariner then runs **without Globalnet** |
| Ingress certificates | Let's Encrypt | Let's Encrypt: publicly trusted, so the DR guide's SSL (`user-ca-bundle`) step is **not needed** |

A booking with `storage: odf` turns DR on: CI's `dr-fabric` and `dr-policy` jobs build layers 2 and 3 **alongside** the class
staging. The healthcheck runs whatever they did and checks the DRPolicy only once `dr-policy` succeeded, so a DR
failure never hides the class's own health; `build-verified` (the booking's READY) needs both DR jobs green.
`odf-site-a` or `na` leaves DR off.

Set these once in your terminal; every command below reads them:

```bash
A=lz1; B=lz2                       # your booking's two OpenShift slots (site A, site B)
A_NAME="ocp-$A"                    # site A as the hub knows it
B_NAME=local-cluster               # site B is the hub, so it manages itself under this name
# contexts site-a / site-b: Lab S §S.1
```

### Layer 1: one hub manages both clusters (ACM)

**Automated on every deploy:** `acm_join.sh` (CI's `demo-prep` job). Site A is booked `acm: managed`, so its own hub
releases itself and site B's hub imports it. Both go into the cluster set `ovedr`, which DR selects clusters by.

**Console:** site B → **All Clusters** (top-left switcher) → **Infrastructure → Clusters → Import cluster**: name `ocp-<A>`,
cluster set `ovedr`, import mode **Kubeconfig** (paste site A's), **Import**. Then **Cluster sets → `ovedr`** holds both.
📸 `S-acm-import` · 📸 `S-acm-clusters`.

<details markdown="1">
<summary>Example code: import site A into site B's hub</summary>

```bash
# 1. site A's hub does not manage itself (demo-openshift's acm=managed does this at build time)
oc --context site-a get mch -A -o jsonpath='{.items[0].spec.disableHubSelfManagement}{"\n"}'      # true
#    by hand on a standalone cluster:
#    oc --context site-a patch mch multiclusterhub -n open-cluster-management --type merge -p '{"spec":{"disableHubSelfManagement":true}}'

# 2. on the hub: a namespace + ManagedCluster for site A, and a one-time auto-import secret with site A's kubeconfig
oc --context site-b create namespace "$A_NAME" --dry-run=client -o yaml | oc --context site-b apply -f -
oc --context site-b apply -f - <<YAML
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: $A_NAME
  labels: { cloud: auto-detect, vendor: auto-detect, cluster.open-cluster-management.io/clusterset: ovedr }
spec: { hubAcceptsClient: true }
YAML
oc --context site-b -n "$A_NAME" create secret generic auto-import-secret \
  --from-literal=autoImportRetry=5 --from-file=kubeconfig="$HOME/.kube/site-a.kubeconfig"   # ACM deletes it after import

# 3. the cluster set holding both
oc --context site-b apply -f - <<YAML
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata: { name: ovedr }
YAML
oc --context site-b label managedcluster "$B_NAME" cluster.open-cluster-management.io/clusterset=ovedr --overwrite
```

</details>

**Check:**

```bash
oc --context site-b get managedclusters          # local-cluster and ocp-<A>: JOINED True, AVAILABLE True
```

<details markdown="1">
<summary>Debug: site A never shows JOINED / AVAILABLE</summary>

| What you see | Why | Fix |
|---|---|---|
| `ocp-<A>` stays `JOINED Unknown` | the import agent never started on site A | `oc --context site-a -n open-cluster-management-agent get pods`: `klusterlet-*` must be Running. `oc --context site-b -n "$A_NAME" get secret auto-import-secret`: if it is still there, the import failed; read `oc --context site-b -n "$A_NAME" get managedcluster "$A_NAME" -o yaml` conditions |
| site A still lists `local-cluster` | its own hub still manages itself | the `disableHubSelfManagement` patch above, then wait for `oc --context site-a get managedcluster local-cluster` to say NotFound |
| `AVAILABLE False` after it joined | the agent lost the hub's API | `oc --context site-a -n open-cluster-management-agent logs deploy/klusterlet-registration-agent \| tail` |

</details>

Red Hat: [ACM 2.16 Clusters: importing a managed cluster, cluster sets](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html/clusters/index).

### Layer 2: the clusters' networks reach each other (Submariner)

ODF mirrors Ceph images over the clusters' **pod/service networks**, so Submariner joins them. Submariner refuses a peer
whose ranges overlap its own, **in either IP family**. Separate ranges are Red Hat's preferred layout and need nothing
extra. **Globalnet**, which gives each cluster a virtual IPv4 range, covers only an IPv4 overlap. CI reads both sites'
ranges first (`oc get network.config cluster`), turns Globalnet on only for an IPv4-only overlap, and stops straight
away if the IPv6 ranges overlap.

Each site needs one **gateway node** that carries the tunnel (IPsec, UDP 4500 between the two gateway nodes). On bare
metal nobody picks it for you, so label one node per site. Also pin its endpoint to its own node IPs. Left alone, it
asks internet resolvers for a public IP in each family. On these dual-stack nodes the IPv6 lookup finds nothing and the
gateway crash-loops.

**Console:** hub → **Infrastructure → Clusters → Cluster sets → `ovedr` → Submariner add-ons → Install Submariner add-ons**,
select both clusters, enable **Globalnet** only if the sites' IPv4 ranges overlap, keep the defaults, **Install**. Wait until both show *Healthy* (connection and agent).
📸 `S-submariner-install` · 📸 `S-submariner-healthy`.

<details markdown="1">
<summary>Example code: gateway nodes, the broker and the add-on (what <code>dr_fabric.sh</code> step 2 applies)</summary>

```bash
# a gateway node on each site (keeps an existing choice)
for s in site-a site-b; do
  oc --context $s get nodes -l submariner.io/gateway=true -o name | grep -q . || \
    oc --context $s label node "$(oc --context $s get nodes -o jsonpath='{.items[0].metadata.name}')" submariner.io/gateway=true
done

# pin each gateway's endpoint to its node IPs (no internet lookup; dual-stack needs the ipv6: entry)
for s in site-a site-b; do
  for gw in $(oc --context $s get nodes -l submariner.io/gateway=true -o name); do
    ips=$(oc --context $s get $gw -o json | jq -r '[.status.addresses[] | select(.type=="InternalIP") | .address
            | (if test(":") then "ipv6:" else "ipv4:" end) + .] | join(",")')
    oc --context $s annotate $gw --overwrite gateway.submariner.io/public-ip="$ips"
  done
done

# the broker lives in the cluster set's namespace on the hub; Globalnet only for an IPv4-only overlap
oc --context site-b apply -f - <<YAML
apiVersion: submariner.io/v1alpha1
kind: Broker
metadata: { name: submariner-broker, namespace: ovedr-broker }
spec: { globalnetEnabled: false }
YAML

# the add-on on each managed cluster
for c in "$A_NAME" "$B_NAME"; do
  oc --context site-b apply -f - <<YAML
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata: { name: submariner, namespace: $c }
spec: { installNamespace: submariner-operator }
YAML
done
```

</details>

**Check:**

```bash
oc --context site-b get managedclusteraddon -A | grep -E 'NAME|submariner'          # AVAILABLE True on both
for c in "$A_NAME" "$B_NAME"; do
  oc --context site-b -n $c get managedclusteraddon submariner \
    -o jsonpath='{.metadata.namespace}{"  connection-degraded="}{.status.conditions[?(@.type=="SubmarinerConnectionDegraded")].status}{"\n"}'
done                                                                                 # False on both
oc --context site-b -n ovedr-broker get broker submariner-broker -o jsonpath='{.spec.globalnetEnabled}{"\n"}'   # false (true only for an IPv4-only overlap)
```

<details markdown="1">
<summary>Debug: Submariner installed but not connected</summary>

| What you see | Why | Fix |
|---|---|---|
| `SubmarinerConnectionDegraded True` and the gateway pod logs `no gateway nodes` | no node carries `submariner.io/gateway=true` | label one node per site (example code above) |
| gateway pod in `CrashLoopBackOff`, log ends `could not determine public IPv6` | dual-stack node, no `public-ip` pin, so it asked internet resolvers for an IPv6 address | annotate the gateway node with its own IPs (example code above), then delete the gateway pod (0024-015) |
| gateways `Running`, no connection, log says `Skip processing the remote endpoint … as subnets are overlapping` | the two sites share a pod/service range. With Globalnet on, it's the IPv6 half (`fd02::/48`, `fd03::/112`), which Globalnet can't translate | not fixable on a built cluster: the member build gives each site its own ranges, then a fresh deploy (0024-015, BL-D18) |
| gateways up, connection stays `connecting` / `error` | the two gateway nodes can't reach each other on **UDP 4500** (site A's nodes and site B's nodes sit on different VLANs, so this crosses the lab router) | `oc --context site-a -n submariner-operator get gateways -o yaml` shows each side's endpoint IP and the last error. The router must allow UDP 4500 (and 4490) between the two gateway node IPs |
| connected, but `*.svc.clusterset.local` names don't resolve | Lighthouse DNS (the add-on's service discovery) isn't up | `oc --context site-a -n submariner-operator get pods \| grep lighthouse`: both agents Running |
| a pod on one site can't reach a service exported from the other | Globalnet hasn't given that service a global IP yet | `oc --context site-a get globalingressip -A`: the exported service has an allocated IP |

`subctl` (Submariner's CLI, from the Red Hat downloads) runs every one of these checks at once: `subctl show all` and
`subctl diagnose all` with each site's kubeconfig.

</details>

Red Hat: [ACM 2.16 Networking: Submariner multicluster networking and service discovery](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html/networking/index).

### Layer 3: linking ODF for asynchronous replication (Demo 4)

Each site runs its **own** Ceph cluster (ODF). Replication links the two, so every protected RBD image (a VM disk) on the
primary is **mirrored** to the other site as snapshots on a schedule (**async**: every 15 minutes in Demo 4). The order follows
Red Hat's ODF 4.19 Regional-DR workflow:

| Step | What it is | Built by |
|---|---|---|
| ODF on both sites | a Ready `ocs-storagecluster` on each | demo-openshift (`storage: odf`) |
| **MCG (NooBaa) on both sites** | object storage where Ramen keeps each protected app's DR metadata (its S3 store) | `dr_fabric.sh` step 1 (demo-openshift ships MCG off) |
| **Export ODF's provider server** (only when Globalnet is on) | a `ServiceExport` of `ocs-provider-server` on each site, and the StorageCluster annotated with its exported name, so the peers find each other through Submariner | `dr_fabric.sh` step 2b |
| **ODF Multicluster Orchestrator** on the hub | the operator that links ODF clusters; it brings the DR hub operator (Ramen) | `dr_fabric.sh` step 3 |
| **MirrorPeer** | pairs the two StorageClusters: exchanges Ceph keys and endpoints, turns on RBD mirroring (`rbd-mirror`) and creates the S3 profiles | `dr_fabric.sh` step 3 (or the console's DRPolicy wizard) |
| **DRCluster** × 2 | the hub's view of each site as a DR peer (its S3 profile, fencing state) | the Orchestrator, from the MirrorPeer |
| **DRPolicy** | "these two clusters, async, every **15m**" | `dr_policy.sh` (`ocp-<A>-local-cluster-15m`) |

Then each protected application (Demo 4: your namespace) gets a **DRPlacementControl** on the hub, which tells Ramen which
cluster is primary and drives Failover and Relocate (Demo 4 §4.3).

**Console** (after layers 1 and 2): hub → **All Clusters → Data Services → Disaster recovery → Create DRPolicy**: select
both clusters; replication is set to **Asynchronous**; sync schedule **15m**; **Create**. The wizard creates the MirrorPeer
and both DRClusters. One DRPolicy per interval: a `5m` policy is a second DRPolicy with its own name.
📸 `S-drpolicy-create` · 📸 `S-drpolicy-validated`.

<details markdown="1">
<summary>Example code: MCG, provider-server export, Orchestrator, MirrorPeer and DRPolicy (what <code>dr_fabric.sh</code> and <code>dr_policy.sh</code> apply)</summary>

```bash
# 1. MCG on both sites (Ramen keeps DR metadata in S3 buckets there)
for s in site-a site-b; do
  oc --context $s -n openshift-storage patch storagecluster ocs-storagecluster --type merge \
    -p '{"spec":{"multiCloudGateway":{"reconcileStrategy":"manage"}}}'
done
oc --context site-a -n openshift-storage get noobaa noobaa -o jsonpath='{.status.phase}{"\n"}'   # Ready (both sites)

# 2. Globalnet: export each site's ODF provider server, and tell its StorageCluster the exported name
for pair in "site-a:$A_NAME" "site-b:$B_NAME"; do
  s=${pair%%:*}; c=${pair#*:}
  oc --context $s apply -f - <<'YAML'
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata: { name: ocs-provider-server, namespace: openshift-storage }
YAML
  oc --context $s -n openshift-storage annotate storagecluster ocs-storagecluster --overwrite \
    "ocs.openshift.io/api-server-exported-address=$c.ocs-provider-server.openshift-storage.svc.clusterset.local:50051"
done

# 3. the Orchestrator on the hub (brings Ramen, the DR hub operator)
oc --context site-b apply -f - <<'YAML'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: { name: odf-multicluster-orchestrator, namespace: openshift-operators }
spec: { name: odf-multicluster-orchestrator, channel: stable-4.19, source: redhat-operators,
        sourceNamespace: openshift-marketplace, installPlanApproval: Automatic }
YAML
oc --context site-b -n openshift-operators get pods | grep -E 'odfmo-controller|ramen-hub|odf-multicluster-console'

# 4. the MirrorPeer: async, offering every interval the class can book
oc --context site-b apply -f - <<YAML
apiVersion: multicluster.odf.openshift.io/v1alpha1
kind: MirrorPeer
metadata: { name: ovedr-$A-$B }
spec:
  type: async
  manageS3: true
  schedulingIntervals: [ 2m, 5m, 15m ]
  items:
    - { clusterName: $A_NAME, storageClusterRef: { name: ocs-storagecluster, namespace: openshift-storage } }
    - { clusterName: $B_NAME, storageClusterRef: { name: ocs-storagecluster, namespace: openshift-storage } }
YAML

# 5. the DRPolicy: these two clusters, every 15 minutes
oc --context site-b apply -f - <<YAML
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata: { name: $A_NAME-$B_NAME-15m }
spec:
  drClusters: [ $A_NAME, $B_NAME ]
  schedulingInterval: 15m
  replicationClassSelector: {}
  volumeSnapshotClassSelector: {}
YAML
```

</details>

**Check, from the top down:**

```bash
oc --context site-b get mirrorpeer -o custom-columns=NAME:.metadata.name,TYPE:.spec.type,PHASE:.status.phase
oc --context site-b get drcluster \
  -o custom-columns=NAME:.metadata.name,S3:.spec.s3ProfileName,VALIDATED:'.status.conditions[?(@.type=="Validated")].status'
oc --context site-b get drpolicy -o custom-columns=NAME:.metadata.name,INTERVAL:.spec.schedulingInterval,VALIDATED:'.status.conditions[?(@.type=="Validated")].status'
for s in site-a site-b; do
  echo "== $s"
  oc --context $s -n openshift-storage get storageclusterpeer -o custom-columns=NAME:.metadata.name,STATE:.status.state   # Peered
  oc --context $s -n openshift-storage get cephblockpoolradosnamespaces ocs-storagecluster-cephblockpool-builtin-implicit \
    -o jsonpath='{.status.mirroringStatus.summary}{"\n"}'                     # "daemon_health":"OK", "health":"OK"
  oc --context $s get volumereplicationclass                                 # one per interval (2m, 5m, 15m)
done
```

✅ DRClusters and the DRPolicy `Validated True`, both peers `Peered`, mirroring health `OK` on both sites. A fresh link
can show `WARNING` for up to 10 minutes before it settles at `OK`.

<details markdown="1">
<summary>Debug: the MirrorPeer, DRClusters or DRPolicy never validate</summary>

| What you see | Why | Fix |
|---|---|---|
| MirrorPeer `PHASE` empty or `ExchangingSecret` for minutes | the Orchestrator can't reach a site's StorageCluster, or MCG isn't Ready | `oc --context site-b -n openshift-operators logs deploy/odfmo-controller-manager \| tail -30`; `oc --context site-X -n openshift-storage get noobaa` must be `Ready` |
| DRCluster `Validated False`, message about S3 | Ramen can't reach a site's S3 store (MCG route) | `oc --context site-b -n openshift-operators get cm ramen-hub-operator-config -o yaml`: an `s3StoreProfiles` entry per site; `curl -sI` its `s3CompatibleEndpoint` from your terminal must answer |
| StorageClusterPeer not `Peered` | the peers can't reach each other's provider server across Submariner | the step 2 annotation is on both StorageClusters; `oc --context site-a -n openshift-storage get serviceexport ocs-provider-server -o yaml` shows `Valid True`; Layer 2's debug table |
| mirroring `health` stays `WARNING` / `ERROR` | `rbd-mirror` can't reach the peer's Ceph monitors/OSDs | `oc --context site-X -n openshift-storage get pods \| grep rbd-mirror` Running; instructor: `oc --context site-X -n openshift-storage rsh deploy/rook-ceph-tools rbd mirror pool status ocs-storagecluster-cephblockpool --verbose` names the failing peer |
| DRPolicy `Validated False`: interval not supported | the MirrorPeer doesn't offer that interval | add it to the MirrorPeer's `schedulingIntervals`, wait for the new `VolumeReplicationClass` on both sites |
| DRPolicy `Validated False`: clusters not found | a `drClusters` name isn't a DRCluster | `oc --context site-b get drcluster`: the names are the managed-cluster names (`ocp-<A>`, `local-cluster`) |

Gather it all for a support case: `oc adm must-gather --image=registry.redhat.io/odf4/odf-must-gather-rhel9:v4.19` on each
cluster (the hub included).

</details>

Red Hat: [ODF 4.19: Configuring disaster recovery for OpenShift workloads, Regional-DR](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19/html/configuring_openshift_data_foundation_disaster_recovery_for_openshift_workloads/index) ·
[Protecting OpenShift Virtualization VMs with Regional-DR](https://access.redhat.com/articles/7053115).

### What CI automates

| Layer | Automated | Proven |
|---|---|---|
| MTV between the clusters (Demo 1c) | ✅ every deploy: providers `ovedr-ocp-<peer>` + maps, both directions | ✅ |
| 1 · ACM join | ✅ every deploy: site A booked `acm: managed`, then `acm_join.sh` (`demo-prep`) imports it into site B's hub; both in set `ovedr` | ⏳ checked again on the first ODF-on-both booking |
| 2 · Submariner (Globalnet only for an IPv4 overlap) | ✅ when DR is on (`storage: odf`): `dr_fabric.sh` | ⛔ 0024-015: gateways up after the public-IP pin; sites share IPv6 ranges, waiting on demo-openshift (BL-D18) |
| 3 · ODF replication link | ✅ when DR is on: `dr_fabric.sh` (MCG, provider-server export, MirrorPeer, DRClusters, peering + mirror health) + `dr_policy.sh` (DRPolicy 15m) | ⏳ first ODF-on-both booking |


## S.15 VDDK: the one piece the customer supplies (Broadcom licence)

> **Retired, 2026-10-14.** This lab no longer uses VDDK: it can't obtain the kit, and Demo 2 moves to storage copy
> offload ([why](02-vmware-to-ove-warm.md#why-this-lab-does-not-demo-vddk)). This section stays as reference for a
> customer who already holds a VDDK copy and the right to use it.

**What it is.** The VMware **Virtual Disk Development Kit (VDDK)** is VMware's library for reading a VM's disks
straight from ESXi. MTV uses it to copy disks, and for a **warm** migration to copy only the blocks that changed since
the last pass (VMware's Changed Block Tracking, Demo 2). **Warm migration needs it**; a cold migration works without it,
only slower **[verify: MTV 2.11 cold path without VDDK]**.

**Why you supply it.** VDDK is Broadcom's software, and its licence does **not** allow Red Hat, Paragon Micro or anyone
else to redistribute it. So OpenShift doesn't ship it, and neither does this lab's code: **each customer downloads VDDK
themselves** from Broadcom's developer portal with their own account, accepts the licence, builds a small container
image from it, and keeps that image in **their own private registry**. Never put the tarball or the image in git or a
public registry. In a real migration project this is one of the first prerequisites to arrange.

**How to build it** (Red Hat's procedure):

1. Download **VDDK 8.0.x for Linux** (`VMware-vix-disklib-<version>.x86_64.tar.gz`) from Broadcom's developer portal.
2. Unpack it next to this `Containerfile`:

   ```dockerfile
   FROM registry.access.redhat.com/ubi8/ubi-minimal
   USER 1001
   COPY vmware-vix-disklib-distrib /vmware-vix-disklib-distrib
   RUN mkdir -p /opt
   ENTRYPOINT ["cp", "-r", "/vmware-vix-disklib-distrib", "/opt"]
   ```

3. Build and push it to a **private** registry. On OpenShift, the cluster's own registry works, with no tool on your
   laptop:

   ```bash
   tar -xzf VMware-vix-disklib-*.x86_64.tar.gz                  # → vmware-vix-disklib-distrib/
   oc -n openshift-mtv new-build --name vddk --binary --strategy docker
   oc -n openshift-mtv start-build vddk --from-dir=. --follow   # the directory with the Containerfile and the unpacked kit
   oc -n openshift-mtv get istag vddk:latest                    # image-registry.openshift-image-registry.svc:5000/openshift-mtv/vddk:latest
   ```

   MTV's migration pods run in the **target** namespace, so let every service account pull from `openshift-mtv`:

   ```bash
   oc -n openshift-mtv policy add-role-to-group system:image-puller system:serviceaccounts
   ```

4. Point the vSphere provider at it. **Console:** Migration for Virtualization → Providers → your vSphere provider →
   **VDDK init image** → edit. **CLI:**

   ```bash
   oc -n openshift-mtv patch provider "ovedr-vcenter0$N" --type merge \
     -p '{"spec":{"settings":{"vddkInitImage":"image-registry.openshift-image-registry.svc:5000/openshift-mtv/vddk:latest"}}}'
   ```

**How this lab does it** (`automation/customize/vddk.sh`, job `vddk`, after MTV is installed):

1. **Once per VDDK version, the operator** downloads the kit with the lab's Broadcom account and runs
   `automation/tools/vddk_image.sh <kit>`. It builds the image above and saves it in the lab's **private** registry
   (never git, never public: the script refuses to finish if the image can be read without credentials). The image
   holds the unpacked kit, so it is also the lab's only stored copy of the download.
2. **Every booking, on each cluster that runs MTV:** a Docker `BuildConfig` whose `Containerfile` is just
   `FROM <the saved image>` copies it into the cluster's own registry as `openshift-mtv/vddk:<version>`, with a
   short-lived read-only pull credential that is deleted when the build ends. Then the `image-puller` grant above.
3. **The vSphere provider** gets `vddkInitImage: image-registry.openshift-image-registry.svc:5000/openshift-mtv/vddk:<version>`,
   and the run's readiness table shows Demo 2 ready.

So nothing is redistributed, the licensed file leaves the lab's private storage only into the lab's own clusters, and a
booking never waits on a download. If no saved image exists yet, the `vddk` job warns, Demo 2 shows **blocked** and
the warm plans show `VDDKInitImageUnavailable` (Demo 2 §2.5); the cold class runs as normal.

**Where Broadcom keeps it:** VDDK moved to Broadcom's developer portal
([developer.broadcom.com → SDKs → VMware Virtual Disk Development Kit](https://developer.broadcom.com/sdks/vmware-virtual-disk-development-kit-vddk/latest)),
behind a Broadcom sign-in. Older VMware download links no longer work. Use a VDDK version your vSphere release and your
MTV release both support (Red Hat's MTV prerequisites list them).

**Check:**

```bash
oc --context site-a -n openshift-mtv get provider "ovedr-vcenter0$N" -o jsonpath='{.spec.settings.vddkInitImage}{"\n"}'   # the image
oc --context site-a -n openshift-mtv get plan -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.status.conditions[?(@.type=="VDDKInitImageUnavailable")].status}{"\n"}{end}'   # empty: no plan lacks it
```

**Red Hat docs:**
- [Creating a VDDK image](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites_mtv#creating-vddk-image_mtv) [verify link]
- [Warm migration](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11) [verify link]

---

## Thank you

Thank you for looking under the hood. Everything your demos rely on is listed here, and you could rebuild each piece
yourself. To go further:

- [OpenShift Virtualization 4.19 documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/index)
- [Migration Toolkit for Virtualization 2.11 documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.11)
- [OpenShift Data Foundation 4.19 documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.19)
- [Red Hat Advanced Cluster Management 2.13 documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13)
- [Red Hat OpenShift Virtualization](https://www.redhat.com/en/technologies/cloud-computing/openshift/virtualization)
- [DO316: Managing Virtual Machines with Red Hat OpenShift Virtualization](https://www.redhat.com/en/services/training/do316-managing-virtual-machines-red-hat-openshift-virtualization)
- [Red Hat interactive labs](https://www.redhat.com/en/interactive-labs)
- [Red Hat Developer: OpenShift](https://developers.redhat.com/products/openshift/overview)

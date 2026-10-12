# Ceph in a Gardener Shoot, shared with other Shoots

This repository deploys a Ceph cluster with [Rook](https://rook.io) into one Gardener Shoot, the
**provider**, and makes its CephFS filesystem available as a `ReadWriteMany` storage class in other
Shoots, the **consumers**.

> **Disclaimer.** This is a starting point that shows what is possible on Open Sovereign Cloud (OSC).
> OSC gives no support or warranty for this way of running Ceph. Use it at your own risk.

| Component     | Version                                                         |
|---------------|-----------------------------------------------------------------|
| Rook          | v1.21.0, on the provider with [one patch](#why-a-patched-rook-operator) |
| Ceph          | v20.2.4 (Tentacle)                                              |
| Kubernetes    | v1.32 or newer (required by Rook v1.21)                         |
| Node kernel   | 5.11 or newer (msgr2); see [key types](#cephx-key-types-and-the-node-kernel) |

Everything below was run end to end on the OSC dev landscape on 2026-10-12: provider on Kubernetes
1.34.8, consumer on 1.32.9, all nodes on Garden Linux 2150.4.0 with kernel 6.18.

The earlier version of this guide, for Rook v1.16 and Ceph v19, is on the `main` branch.
[What changed](#what-changed-since-the-rook-v116-version) is listed at the end.

## How it works

```
        provider shoot                                    consumer shoot
 ┌───────────────────────────────┐                 ┌───────────────────────────┐
 │ dedicated nodes, one per zone │                 │ stock Rook v1.21          │
 │  mon  mgr  osd  mds           │   peered node   │  CephCluster (external)   │
 │  (host network)               │◄───networks────►│  CephFS CSI driver        │
 │                               │                 │  StorageClass "cephfs"    │
 │ one internal load balancer    │                 │                           │
 │ per mon = stable mon address  │                 │ pods mount the filesystem │
 └───────────────────────────────┘                 └───────────────────────────┘
```

* **Only the node networks are connected.** Shoots are peered through their node networks. Pod and
  service ranges are the same in every Shoot and are not routed between them. The Ceph daemons
  therefore run on the host network, so that consumers can reach them on node addresses.
* **Mons need addresses that do not change.** A Ceph client finds the cluster through its mons, and
  a mon is identified by its address. On the host network that address is the node address, and
  Gardener replaces nodes on every operating system or Kubernetes update. Each mon therefore gets a
  load balancer with an internal address and advertises that address instead.
* **Consumers run unmodified Rook.** They connect in Rook's
  [external cluster](https://rook.io/docs/rook/v1.21/CRDs/Cluster/external-cluster/external-cluster/)
  mode. Each consumer gets its own Ceph users, restricted to the filesystem.

### Why a patched Rook operator

Upstream Rook cannot put a load balancer in front of a mon: a mon must advertise the address that
clients connect to, and Rook only knows the node address or the cluster IP. With plain host
networking Rook also pins each mon to its node. When that node is replaced, Rook fails over to a new
mon with a new name and a new address. Consumers follow such changes only while they stay connected.

The provider therefore runs a Rook operator with one additional setting on the `CephCluster`:

```yaml
spec:
  network:
    provider: host
    internalLoadBalancerService:
      enabled: true
      labels: # applied as annotations to the mon services
        service.beta.kubernetes.io/onmetal-load-balancer-internal: "true"
```

With it, each mon service is of type `LoadBalancer`, the mon advertises the load balancer address,
and the mon is free to move to another node together with its volume. For that, the mons have to
store their data on volumes (`mon.volumeClaimTemplate`), as they do in `cluster/cluster.yaml`. Set
it when the cluster is created; changing it on a running cluster is not supported.

* Source: tag [`v1.21.0-osc.2`](https://github.com/opensovereigncloud/rook/tree/v1.21.0-osc.2) on
  the branch `release-1.21-osc` of the OSC fork, three commits on top of Rook v1.21.0.
* Image: `registry.osc.live/osc/iaas/rook:v1.21.0-osc.2`. It can be pulled without credentials.
  [Building the operator image](#building-the-operator-image) shows how to build your own.

## Repository layout

```
deployment/
  shoot/                   Shoot manifest of the provider
  rook-operator/           provider: Rook operator (OSC image) and CSI settings
  cluster/                 provider: CephCluster, CephFS, toolbox, mgr load balancer
  rook-operator-external/  consumer: Rook operator (upstream image)
  cluster-external/        consumer: CephCluster in external mode
  test/                    consumer: three web servers sharing one volume
```

The kustomizations take the Rook manifests straight from GitHub at the pinned version, so there is
nothing vendored to keep in sync.

## Prerequisites

* A Gardener project in which you can create the provider Shoot, and admin access to each consumer
  Shoot.
* `kubectl`. It contains `kustomize` as `kubectl kustomize`.
* Kubernetes v1.32 or newer in all Shoots.
* Node networks that do not overlap between the provider and any consumer. Many Shoots use the
  landscape default `10.250.0.0/27`. The provider below uses `10.251.0.0/26`.

All commands run from the `deployment` directory. `KUBECONFIG` must point at the cluster named in
each step: the **garden** cluster, the **provider** Shoot or the **consumer** Shoot.

## Provider

### 1. Create the Shoot

[`shoot/shoot.yaml`](deployment/shoot/shoot.yaml) holds a complete example. The parts that matter for
Ceph:

* **One dedicated worker pool per zone** (`rook-az1` to `rook-az3`), each with one node of type
  `vm-dedicated-4-16`. The nodes carry the labels `dedicated: rook` and `role: osd-node` and the taint
  `dedicated=rook:NoSchedule`, so only Ceph runs there. `systemComponents.allow: false` keeps the
  Gardener system components off them.
* **`machineHealthTimeout: 168h`** on these pools. A node that is unhealthy for a short while is not
  replaced right away, which would move Ceph daemons for no good reason.
* **A general pool** (`pool1`) for system components and everything else.
* **`networking.nodes`** that does not overlap with any consumer.
* **The `osc-peering-service` extension**, which lists every consumer Shoot. See
  [Peer the Shoots](#1-peer-the-shoots).

Create it on the **garden** cluster and wait until it is reconciled:

```bash
kubectl apply -f shoot/shoot.yaml
kubectl -n garden-prod get shoot ceph --watch
```

### 2. Deploy the Rook operator

On the **provider**:

```bash
kubectl kustomize rook-operator | kubectl apply --server-side -f -
```

The first run ends with two errors, `no matches for kind "OperatorConfig"` and `"Driver"`, because
the CRDs it has just created are not registered yet. Run the same command again; the second run
applies everything.

```console
$ kubectl -n rook-ceph get pods
NAME                                                        READY   STATUS
ceph-csi-controller-manager-64fd56fff4-hjl8j                1/1     Running
rook-ceph-operator-7c6fb4db8d-bnjm7                         1/1     Running
rook-ceph.cephfs.csi.ceph.com-ctrlplugin-7487bff5c7-mbg9t   5/5     Running
rook-ceph.cephfs.csi.ceph.com-ctrlplugin-7487bff5c7-ts9vh   5/5     Running
rook-ceph.cephfs.csi.ceph.com-nodeplugin-69jjz              2/2     Running
...
```

What the kustomization changes compared to upstream Rook:

* the CRDs and the operator image come from the OSC fork,
* the operator and the CSI provisioner run on the dedicated nodes, and the CSI node plugin
  tolerates their taint,
* the RBD driver is removed. Only CephFS is offered, because block storage already comes with the
  default storage class of every Shoot.

Since Rook v1.21 the CSI drivers are run by the
[ceph-csi-operator](https://github.com/ceph/ceph-csi-operator). CSI settings are no longer keys in
the `rook-ceph-operator-config` ConfigMap but fields of the `OperatorConfig` and `Driver` resources;
see [`rook-operator/operator-patch.yaml`](deployment/rook-operator/operator-patch.yaml).

### 3. Deploy the Ceph cluster

```bash
kubectl kustomize cluster | kubectl apply --server-side -f -
```

This creates

* the `CephCluster` with three mons and one OSD per zone, both on volumes of the default storage
  class, and two mgrs,
* the filesystem `cephfs` with a replicated and an erasure coded data pool, the subvolume group
  `csi`, and the storage class `cephfs` for pods on the provider itself,
* the load balancer `external-mgr`, which gives the dashboard and the metrics endpoint of the active
  mgr a stable address,
* a toolbox that contains the script for [exporting the connection data](#2-export-the-connection-data).

After about four minutes:

```console
$ kubectl -n rook-ceph get cephcluster,cephfilesystem
NAME                                 MONCOUNT   PHASE   MESSAGE                        HEALTH
cephcluster.ceph.rook.io/rook-ceph   3          Ready   Cluster created successfully   HEALTH_OK

NAME                                 ACTIVEMDS   PHASE
cephfilesystem.ceph.rook.io/cephfs   1           Ready

$ kubectl -n rook-ceph get svc
NAME              TYPE           EXTERNAL-IP   PORT(S)
external-mgr      LoadBalancer   10.251.0.6    8443/TCP,9283/TCP
rook-ceph-mon-a   LoadBalancer   10.251.0.7    3300:30238/TCP
rook-ceph-mon-b   LoadBalancer   10.251.0.8    3300:31364/TCP
rook-ceph-mon-c   LoadBalancer   10.251.0.9    3300:30150/TCP
```

The mons are registered in Ceph with the load balancer addresses, not with the node addresses
(`10.251.0.1`, `.2` and `.4` in this run):

```console
$ kubectl -n rook-ceph exec deploy/rook-ceph-tools-operator-image -- ceph mon dump
0: v2:10.251.0.7:3300/0 mon.a
1: v2:10.251.0.8:3300/0 mon.b
2: v2:10.251.0.9:3300/0 mon.c
```

## Consumer

Repeat this section for every Shoot that should use the filesystem.

### 1. Peer the Shoots

The node networks of the provider and the consumer are connected with the `osc-peering-service`
extension. It is self-service: each of the two Shoots names the other one in its own spec.

In the provider Shoot (already part of `shoot/shoot.yaml`):

```yaml
spec:
  extensions:
    - type: osc-peering-service
      providerConfig:
        apiVersion: service.peering.extensions.osc.t-systems.com/v1
        kind: PeeringConfig
        shootPeerings:
          - name: test            # the consumer Shoot
            namespace: garden-dev # its project namespace
            seed: ds0-0           # the seed it runs on
```

In the consumer Shoot, the same block with the provider's name, namespace and seed.

Check from a pod on the **consumer** that a mon answers. Use one of the load balancer addresses
from the previous step:

```bash
kubectl run peer-probe --rm -it --restart=Never \
  --image=registry.k8s.io/e2e-test-images/busybox:1.36.1-1 -- nc -z -w 3 10.251.0.7 3300 && echo open
```

### 2. Export the connection data

On the **provider**, run Rook's export script in the toolbox. It creates Ceph users for this
consumer and prints their keys together with the cluster id and a mon address.

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools-operator-image -- \
  python3 /etc/rook-external/create-external-cluster-resources.py \
    --namespace rook-ceph \
    --format bash \
    --k8s-cluster-name test \
    --restricted-auth-permission true \
    --cephfs-filesystem-name cephfs \
    --cephfs-data-pool-name cephfs-erasurecoded \
    --rbd-data-pool-name .mgr \
    --alias-rbd-data-pool-name data \
    --cephx-key-type aes \
  | sed -n '/^export /,$p' > consumer.env

# address of the active mgr, for the dashboard link and the metrics endpoint
MGR_IP=$(kubectl -n rook-ceph get svc external-mgr -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

* `--k8s-cluster-name` is a name for this consumer. Together with `--restricted-auth-permission` it
  gives the consumer its own users, which may only use this filesystem.
* `--cephx-key-type aes` is required while nodes run a kernel older than 7.0. See
  [key types](#cephx-key-types-and-the-node-kernel).
* `--rbd-data-pool-name` is mandatory for restricted users even though no block storage is shared.
  `.mgr` stands in, and the RBD values are dropped again before the import.
* The `sed` cuts off two informational lines that the script prints before the variables.

`consumer.env` contains secret keys. Keep it out of version control and delete it after the import.

### 3. Deploy the Rook operator

On the **consumer**. This is upstream Rook without any change other than removing the RBD driver.
As on the provider, run the command twice.

```bash
kubectl kustomize rook-operator-external | kubectl apply --server-side -f -
```

### 4. Import the connection data

Still on the **consumer**, in the same shell that knows `MGR_IP`:

```bash
source consumer.env

# no block storage: drop the RBD values so the import script skips them
unset CSI_RBD_NODE_SECRET CSI_RBD_NODE_SECRET_NAME \
      CSI_RBD_PROVISIONER_SECRET CSI_RBD_PROVISIONER_SECRET_NAME RBD_POOL_NAME

# the script reports the node address of the active mgr; use the load balancer instead
export MONITORING_ENDPOINT=$MGR_IP
export ROOK_EXTERNAL_DASHBOARD_LINK=https://$MGR_IP:8443/

curl -fsSLO https://raw.githubusercontent.com/rook/rook/v1.21.0/deploy/examples/import-external-cluster.sh
bash import-external-cluster.sh
```

```console
secret/rook-csi-cephfs-node-test-cephfs created
secret/rook-csi-cephfs-provisioner-test-cephfs created
secret/rook-ceph-mon created
configmap/rook-ceph-mon-endpoints created
configmap/external-cluster-user-command created
storageclass.storage.k8s.io/cephfs created
```

### 5. Connect to the Ceph cluster

```bash
kubectl kustomize cluster-external | kubectl apply -f -
```

Within a minute the consumer is connected and has learned all three mons:

```console
$ kubectl -n rook-ceph get cephcluster
NAME                 PHASE       MESSAGE                          HEALTH      EXTERNAL
rook-ceph-external   Connected   Cluster connected successfully   HEALTH_OK   true

$ kubectl -n rook-ceph get cm rook-ceph-mon-endpoints -o jsonpath='{.data.data}'
b=10.251.0.8:3300,c=10.251.0.9:3300,a=10.251.0.7:3300
```

### 6. Test

```bash
kubectl kustomize test | kubectl apply -f -
```

This starts three web servers on three different nodes that share one `ReadWriteMany` volume of
the storage class `cephfs`, behind an internal load balancer. Write a file through one pod and read
it through the others:

```console
$ echo "hello" | kubectl -n cephfs-test exec -i deploy/web -- tee /usr/share/nginx/html/index.html
$ kubectl -n cephfs-test get pods -o name | xargs -I{} kubectl -n cephfs-test exec {} -- cat /usr/share/nginx/html/index.html
hello
hello
hello
```

The volume is mounted by the kernel client over msgr2:

```console
$ kubectl -n cephfs-test exec deploy/web -- grep nginx /proc/mounts
... ceph rw,relatime,name=csi-cephfs-node-test-cephfs,secret=<hidden>,ms_mode=prefer-crc,acl,mon_addr=10.251.0.7:3300/10.251.0.8:3300/10.251.0.9:3300
```

## Operations

### Node replacement

Gardener replaces worker nodes in every rolling update, for example for a new operating system
image. This is the situation the mon load balancers exist for. It was tested by changing the machine
image of the dedicated worker pools while a pod on the consumer wrote to the shared volume once per
second.

**One Ceph node.** The node of pool `rook-az1` carried mon `a`, the active mgr, one OSD and the Rook
operator. Gardener added the new node first and then drained the old one.

| Time after the drain started | What happened                                                    |
|------------------------------|------------------------------------------------------------------|
| 0:00                         | old node cordoned, mon `a` rescheduled to the new node with its volume |
| 0:43                         | mon `a` starting on the new node, quorum is `b`, `c`              |
| 1:02                         | mon `a` back in quorum, `HEALTH_OK`, all placement groups clean  |

* Mon `a` kept its name and its address `10.251.0.7`. Rook did not fail over to a new mon.
* The consumer stayed `Connected` and its list of mons did not change.
* The writer on the consumer paused once for 11 seconds, when the daemons on the drained node
  stopped. Every other write completed within 2 seconds.
* When the same pool was replaced again later, the mon was back just as fast, but the OSD waited six
  minutes for its volume: the volume stayed attached to the node that had just been deleted until
  Kubernetes detached it by force. Ceph was `HEALTH_OK` after seven minutes instead of one, and the
  writer paused once for 16 seconds.

**Two Ceph nodes at the same time.** Gardener rolls worker pools in parallel, so changing two
dedicated pools in one go drains two Ceph nodes at once. Rook's disruption budgets then decide the
order.

| Time after the drain started | What happened                                                    |
|------------------------------|------------------------------------------------------------------|
| 0:00                         | both old nodes cordoned; mon `b`, the first OSD and the active MDS evicted |
| 0:27                         | mon `b` back in quorum on a new node                             |
| 0:38                         | standby MDS active, filesystem no longer degraded                |
| 1:08 to 1:54                 | mon `c` moves, only after `b` is back                            |
| 6:12                         | first OSD up again, after waiting almost six minutes for its volume |
| 6:34                         | second OSD evicted; Rook lets it go once all placement groups are clean |
| 12:36                        | second OSD up again, after the same wait for its volume          |
| 13:03                        | `HEALTH_OK`, all placement groups clean                          |

* There were never fewer than two mons in quorum, and all three kept their addresses.
* The writer on the consumer paused once for 42 seconds, while the MDS failed over. Every other write
  completed within 3 seconds.
* Each moved OSD waited for its volume in the same way. For about twelve minutes in total the data
  was stored on two OSDs instead of three.

**All three Ceph nodes at the same time.** This is what one edit of the image version in the
example `shoot.yaml` does, because the three pools share their settings through a YAML anchor.

* The mons and the OSDs again moved one after the other, and two of the three OSDs waited six
  minutes for their volume. Ceph was `HEALTH_OK` 14 minutes after the drain started.
* The writer on the consumer paused twice for 12 seconds.
* Gardener reported the reconciliation of the Shoot as failed in between, because draining a node
  was held back by the disruption budget of the OSDs. It retried on its own and succeeded.

**Repeated replacement.** A mon keeps its name and address every time its node is replaced, not
only the first time. The first build of the operator, `v1.21.0-osc`, did not ensure that: once a mon
had moved, Rook replaced it by a new mon with a new address the next time a health check found it
out of quorum. This happened to one mon in the test with three nodes and is fixed since `v1.21.0-osc.1`. With the fix, a
mon that had already moved twice was kept out of quorum for two minutes, and then its node was
replaced two more times. It came back with the same name and address every time.

**Recommendation.** Roll the dedicated pools one after the other: change one pool, wait for
`HEALTH_OK`, then change the next. With one node at a time Ceph was healthy again after one to
seven minutes and the consumer paused for at most 16 seconds. With several nodes at once the
consumer paused for up to 42 seconds, the data was stored on two OSDs for more than ten minutes, and
Gardener reported a failed reconciliation in between.

### CephX key types and the node kernel

Ceph v20 introduces a stronger key type, `aes256k`, and creates new keys with it by default. The
Linux kernel can only use such keys from version 7.0 on. With an older kernel the mount fails:

```
adding ceph secret key to kernel failed: Invalid argument
couldn't append secret option: -22
```

Until every node that mounts a volume runs kernel 7.0 or newer, in the provider and in all
consumers, the keys used for mounting must be of type `aes`:

* for pods on the provider: `security.cephx.csi.keyType: aes` in
  [`cluster/cluster.yaml`](deployment/cluster/cluster.yaml),
* for consumers: `--cephx-key-type aes` when exporting.

Ceph reports `aes` keys with the health warnings `AUTH_INSECURE_CLIENT_KEY_TYPE`,
`AUTH_INSECURE_KEYS_ALLOWED` and `AUTH_INSECURE_KEYS_CREATABLE`. They cannot be resolved in this
situation, so `cluster.yaml` mutes them. The Rook documentation on
[CephX key rotation](https://rook.io/docs/rook/v1.21/Storage-Configuration/Advanced/cephx-key-rotation/)
describes the later migration to `aes256k`. Remove the mutes again at that point.

If a consumer was exported without `--cephx-key-type aes`, remove its users on the provider
(`ceph auth rm client.csi-cephfs-node-<name>-cephfs` and so on) and the imported secrets on the
consumer, then export and import again. The export script does not change keys that already exist.

### msgr2

The provider sets `network.connections.requireMsgr2: true`, as the Rook v1.21 examples do. The mons
then only listen on port 3300. A consumer must set the same field in its `CephCluster`
([`cluster-external/kustomization.yaml`](deployment/cluster-external/kustomization.yaml) does), so
that Rook mounts the volumes with `ms_mode=prefer-crc`. The kernel client needs that option to
speak msgr2; without it, it uses the old protocol, which port 3300 does not answer.

### Removing things

Order matters, because Rook protects what is still in use.

On a consumer:

```bash
kubectl kustomize test | kubectl delete -f -              # and every other volume of the storage class
kubectl kustomize cluster-external | kubectl delete -f -
kubectl kustomize rook-operator-external | kubectl delete -f -
```

The `CephCluster` is only removed once no volume of the storage class exists any more. Until then
the ConfigMap `rook-ceph-mon-endpoints` stays as well; it carries a finalizer of Rook. Do not delete
the imported secrets while volumes exist, because deleting a volume needs them.

On the provider, deleting the `CephCluster` destroys all data. Rook only wipes its state from the
nodes after an explicit confirmation:

```bash
kubectl -n rook-ceph patch cephcluster rook-ceph --type merge \
  -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'
kubectl kustomize cluster | kubectl delete -f -
```

## Building the operator image

The published image is the upstream image of Rook v1.21.0 with the operator binary replaced by a
build of the fork. Nothing else differs, so it can be rebuilt with `git`, `go` and
[`crane`](https://github.com/google/go-containerregistry/blob/main/cmd/crane/README.md), without a
container runtime:

```bash
IMAGE=registry.example.com/rook:v1.21.0-osc.2   # where to push

git clone --depth 1 --branch v1.21.0-osc.2 https://github.com/opensovereigncloud/rook.git
mkdir -p layer/usr/local/bin
(cd rook && CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath \
  -ldflags "-X github.com/rook/rook/pkg/version.Version=${IMAGE##*:}" \
  -o ../layer/usr/local/bin/rook ./cmd/rook)
tar --owner=0 --group=0 -C layer -cf layer.tar usr/local/bin/rook
crane append --platform linux/amd64 --base docker.io/rook/ceph:v1.21.0 \
  --new_layer layer.tar --new_tag "$IMAGE"
```

Then set the image in `rook-operator/kustomization.yaml` and `cluster/kustomization.yaml`.

## What changed since the Rook v1.16 version

* **Rook v1.16.3 to v1.21.0, Ceph v19.2.0 to v20.2.4.** The patch is ported to Rook v1.21. The
  operator now waits for the load balancer address instead of failing the first attempt, keeps a
  mon when its node is replaced a second time, ignores the setting without host networking, and
  the generated code and CRDs are complete.
* **Kubernetes v1.32 or newer** is required.
* **CSI settings moved** from the operator ConfigMap to the resources of the ceph-csi-operator.
* **Kernel client instead of ceph-fuse**, over msgr2.
* **CephX key type** has to be set to `aes`; see above.
* **Ceph upgrades check the cluster health again.** `skipUpgradeChecks` was `true` and is now
  `false`, the Rook default.
* **Peering is self-service** through the `osc-peering-service` extension. A change request is no
  longer needed.
* **Consumers run upstream Rook.** The fork is only used where the mons run.
* **Nothing is vendored.** The CRDs, the toolbox, the external cluster manifests and the import
  script are used unchanged from their source. The import script no longer needs a local edit,
  because it skips RBD when the RBD variables are unset.
* **The Rook and Ceph images of the provider come from `registry.osc.live`.** Consumers and the test
  use the upstream registries.
* **Fixes:** the mgr load balancer now selects the active mgr only; the test service matched no
  container port and its image could no longer be pulled; the guide referred to a `cluster-prepare`
  step that did not exist and contained keys of a test cluster.

# Lesson 41: Stateful Apps & Persistent Volumes

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-40 (cluster architecture, kubectl, Pods/Deployments/manifests, Services, Ingress, Gateway API, build-push-deploy practice); Lesson 32 (Docker volumes). Student can write Deployment and Service YAML and use `kubectl apply/get/describe/exec/logs`.

## Learning Objectives

By end of lesson student can:
- Explain why Pod storage is ephemeral and name the volume types that do and do not survive Pod restarts (`emptyDir`, `hostPath`, PVC)
- Create a static PersistentVolume and a PersistentVolumeClaim, and predict when and how they bind
- Choose correct `accessModes` (RWO, ROX, RWX, RWOP) and `persistentVolumeReclaimPolicy` (Retain, Delete) for a use case
- Explain what a StorageClass is, how dynamic provisioning replaces hand-made PVs, and what `volumeBindingMode` and `allowVolumeExpansion` do
- Mount a PVC into a Pod through `volumes` and `volumeMounts`
- Explain StatefulSet guarantees (stable name, stable DNS, ordered start/stop, sticky storage) and when to prefer it over a Deployment
- Write a headless Service plus StatefulSet with `volumeClaimTemplates`
- Deploy PostgreSQL as a StatefulSet and prove data survives Pod deletion
- Diagnose a Pending PVC and a stuck Terminating PV/PVC

## Topics

- PersistentVolume (PV): static provisioning, accessModes (RWO/ROX/RWX), reclaimPolicy
- PersistentVolumeClaim (PVC): requesting storage, binding, volumeMounts in pod spec
- StorageClass: dynamic provisioning, provisioner, reclaimPolicy, volumeBindingMode
- StatefulSet vs Deployment: stable network identity, ordered pod management, sticky storage
- Hands-on: deploy PostgreSQL as StatefulSet with PVC; verify data survives pod restart

## Concepts

### Why storage is a problem in Kubernetes

A container's filesystem is a thin writable layer on top of the image (lesson 30/32). When a container crashes and kubelet restarts it, the new container starts from a clean image: **everything written inside is gone**. When a Pod is deleted or rescheduled to another node, it is gone too, and the new Pod may land on a different machine.

Stateless apps (web frontends, APIs) do not care. Databases, queues, and anything that writes files do. Kubernetes solves this by separating three questions:

| Question | Kubernetes object |
|---|---|
| "How does the Pod ask for storage?" | **PersistentVolumeClaim (PVC)** |
| "What actual disk exists in the cluster?" | **PersistentVolume (PV)** |
| "How is a disk created on demand, and what kind?" | **StorageClass** |

Pod author asks for *"5Gi, read-write, fast"*. Cluster admin / cloud provider decides *how* that is satisfied. Same split as the dev/ops contract: developer does not care whether the disk is an AWS EBS volume, a Ceph RBD image, an NFS export, or a folder on the node.

### Volume types in a Pod spec (quick tour)

A Pod declares volumes under `spec.volumes` and mounts them in a container under `volumeMounts`. Many volume types exist; the ones you meet first:

| Type | Lifetime | Typical use |
|---|---|---|
| `emptyDir` | Lives as long as the **Pod** (survives container restart, dies with the Pod). Created empty on the node | Scratch space, cache, sharing files between containers in one Pod |
| `emptyDir: {medium: Memory}` | Same, but RAM-backed (tmpfs); counts against memory limit | Fast scratch, sensitive temp data |
| `hostPath` | Lives on the **node** disk, independent of Pod | Node agents (log collectors, node-exporter). **Not** for app data: Pod may move to another node |
| `persistentVolumeClaim` | Lives as long as the **PVC/PV** — independent of any Pod | Databases and any durable data |
| `nfs`, `csi`, `awsElasticBlockStore`, ... | Provider-specific, usually wrapped in a PV | Direct use is rare; use PV/PVC |

`emptyDir` example (two containers share a directory, data dies with the Pod):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-scratch
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
      volumeMounts:
        - name: scratch
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -F /data/log.txt"]
      volumeMounts:
        - name: scratch
          mountPath: /data
  volumes:
    - name: scratch
      emptyDir: {}
```

`hostPath` example (shown so students recognise it — and know why not to use it for databases):

```yaml
  volumes:
    - name: host-logs
      hostPath:
        path: /var/log
        type: Directory        # fail if path is missing; other values: DirectoryOrCreate, File, FileOrCreate, Socket
```

Security note: `hostPath` gives a Pod access to the node filesystem. Mounting `/` or `/var/run/docker.sock` is a classic container-escape vector.

### PersistentVolume (PV)

A **PersistentVolume** is a piece of storage in the cluster. It is a **cluster-scoped** object (no namespace). It represents a real disk: an NFS export, a cloud disk, a local path. Admin creates it by hand (**static provisioning**) or a provisioner creates it automatically (**dynamic provisioning**, see StorageClass).

Important fields:

| Field | Meaning |
|---|---|
| `spec.capacity.storage` | Size, e.g. `5Gi` |
| `spec.accessModes` | How the volume can be mounted (see below) |
| `spec.persistentVolumeReclaimPolicy` | What happens to the PV when its PVC is deleted: `Retain` or `Delete` |
| `spec.storageClassName` | Class this PV belongs to. Empty string `""` = no class (only static claims that also say `""` match) |
| `spec.volumeMode` | `Filesystem` (default) or `Block` (raw device) |
| `spec.mountOptions` | Extra mount flags, e.g. `nfsvers=4.1` |
| backend (`hostPath`, `nfs`, `csi`, ...) | Where the data physically is |

Static PV example (NFS):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-01
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-manual
  nfs:
    server: 10.0.0.50
    path: /exports/data01
```

Static PV example (hostPath — **lab only**, single-node clusters like minikube/kind/k3s):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-lab-01
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/pv-lab-01
    type: DirectoryOrCreate
```

#### PV lifecycle (phases)

```
Available  ->  Bound  ->  Released  ->  (Available again | deleted | Failed)
```

| Phase | Meaning |
|---|---|
| `Available` | Free, not bound to any claim |
| `Bound` | Bound to a PVC |
| `Released` | PVC was deleted, but PV still holds old data and `claimRef`; **not** automatically reusable |
| `Failed` | Automatic reclamation failed |

### Access modes

Access mode describes how many **nodes** (not Pods!) may mount the volume and with what permission.

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by a single **node**. Several Pods on the same node can share it |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read-write by many nodes (needs NFS, CephFS, EFS, Azure Files, ...) |
| `ReadWriteOncePod` | RWOP | Read-write by exactly **one Pod** (Kubernetes 1.29+ stable, CSI drivers only) |

Key facts:
- Block devices (AWS EBS, GCE PD, Azure Disk, Ceph RBD, local-path) are typically **RWO only**. Asking for RWX on such a class leaves the PVC Pending.
- The mode on a PVC is a **request/filter**, not a hard enforcement by Kubernetes; the backend decides what really works.
- A volume can be mounted with only **one** mode at a time even if it lists several.

### Reclaim policy

What happens to the PV and its data when the PVC is deleted:

| Policy | Behaviour | Use for |
|---|---|---|
| `Retain` | PV goes to `Released`, data kept. Admin must clean up and manually remove `claimRef` (or delete/recreate the PV) before reuse | Production data, anything you cannot afford to lose by accident |
| `Delete` | PV object **and underlying disk** are deleted automatically | Dynamic provisioning for dev/test/ephemeral data (default for most StorageClasses) |
| `Recycle` | Deprecated (`rm -rf` on volume). Do not use | — |

Change policy of an existing PV:

```bash
kubectl patch pv pv-lab-01 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

Good production habit: for a database, set `Retain` on the PV right after dynamic provisioning so that an accidental `kubectl delete pvc` does not destroy the disk.

### PersistentVolumeClaim (PVC)

A **PersistentVolumeClaim** is a *request for storage* by a user. It is **namespaced**. The Pod references the PVC by name; the PVC binds to a PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: manual        # omit => cluster default class; "" => no class (static only)
```

#### How binding works

Control-plane PV controller matches a PVC to a PV when **all** are true:
1. `storageClassName` is equal (the three cases: named class, omitted = default class, `""` = classless).
2. PV `accessModes` contain **all** modes requested by the PVC.
3. PV `capacity` >= PVC `requests.storage` (smallest fitting PV wins; a 10Gi PV can satisfy a 2Gi claim and the claim gets the whole 10Gi — wasteful).
4. PV is `Available` (or pre-bound via `claimRef`/`volumeName`).
5. Optional `selector` (label match) on the PVC matches PV labels.

Binding is **one-to-one and exclusive**: one PVC <-> one PV. Once bound, no other PVC can use that PV.

Pre-binding a PVC to a specific PV:

```yaml
spec:
  volumeName: pv-lab-01     # claim directly names the PV
```

#### Using a PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo hello >> /data/out.txt; sleep 3600"]
      volumeMounts:
        - name: storage            # must match volumes[].name
          mountPath: /data
          # subPath: app1          # optional: mount a sub-directory of the volume
          # readOnly: true
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: app-data        # PVC in the SAME namespace
```

A Pod referencing a PVC that does not exist (or is not bound) stays **Pending** with event `persistentvolumeclaim "app-data" not found` / `pod has unbound immediate PersistentVolumeClaims`.

#### Protection: `kubernetes.io/pvc-protection`

Deleting a PVC that is in use by a Pod does **not** remove it immediately: it stays `Terminating` (finalizer `kubernetes.io/pvc-protection`) until the last Pod using it is gone. Same for PVs bound to claims (`kubernetes.io/pv-protection`). This prevents yanking a disk out from under a running app.

#### Expanding a PVC

If the StorageClass has `allowVolumeExpansion: true`, edit the PVC request upward:

```bash
kubectl patch pvc app-data -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
```

Size can only **grow**, never shrink. Many drivers resize the filesystem online; some need Pod restart. Check `kubectl describe pvc` for condition `FileSystemResizePending`.

### StorageClass and dynamic provisioning

Creating PVs by hand does not scale. A **StorageClass** describes a *kind* of storage and names a **provisioner**; when a PVC asks for that class, the provisioner creates a disk **and** the PV automatically.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com          # CSI driver name; cluster-specific
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete                 # applied to PVs created from this class
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
mountOptions:
  - noatime
```

Fields:

| Field | Meaning |
|---|---|
| `provisioner` | Plugin that creates volumes. Examples: `ebs.csi.aws.com`, `pd.csi.storage.gke.io`, `disk.csi.azure.com`, `rancher.io/local-path` (k3s/rke2/kind), `k8s.io/minikube-hostpath` (minikube), `nfs.csi.k8s.io`, `rook-ceph.rbd.csi.ceph.com` |
| `parameters` | Provider-specific options (disk type, IOPS, encryption, ...) |
| `reclaimPolicy` | `Delete` (default) or `Retain` — copied into generated PVs |
| `volumeBindingMode` | `Immediate` (default) or `WaitForFirstConsumer` |
| `allowVolumeExpansion` | Whether PVCs may be grown |
| `mountOptions` | Flags passed to `mount` |

#### Default StorageClass

A class annotated `storageclass.kubernetes.io/is-default-class: "true"` is used by any PVC that **omits** `storageClassName`. See it with `kubectl get sc` — the default shows `(default)`.

```bash
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Only one default should exist. Two defaults make new PVCs fail/ambiguous.

#### `volumeBindingMode`

| Mode | Behaviour | Why |
|---|---|---|
| `Immediate` | PV is provisioned and bound as soon as PVC is created | Fine for network storage reachable from all nodes (NFS, Ceph) |
| `WaitForFirstConsumer` | PVC stays `Pending` until a Pod using it is **scheduled**; then the disk is created in the same zone/node as that Pod | Essential for zonal/local disks: avoids creating an EBS volume in zone A and then scheduling the Pod to zone B |

**A PVC in `Pending` with event `waiting for first consumer to be created before binding` is NORMAL**, not a bug. It binds when a Pod that mounts it appears.

#### What dynamic provisioning looks like

```
User creates PVC (class: fast-ssd, 5Gi, RWO)
        |
        v
PV controller sees no matching PV -> asks provisioner of class "fast-ssd"
        |
        v
Provisioner creates disk in cloud/backend + creates PV object
        |
        v
PV controller binds PVC <-> PV   (PVC: Bound)
        |
        v
Pod scheduled, kubelet attaches + mounts disk into the Pod
```

### Static vs dynamic provisioning

| | Static | Dynamic |
|---|---|---|
| Who creates PV | Admin by hand | Provisioner on demand |
| Needs StorageClass | No (can use a placeholder name or `""`) | Yes |
| Good for | Existing NFS shares, pre-existing disks, labs | Everything in cloud / modern clusters |
| Risk | Capacity mismatch, manual toil | Surprise cost, `Delete` policy data loss |

### Stateless vs stateful workloads

A **Deployment** treats Pods as interchangeable cattle: random names (`web-7d9f8c-x2kqp`), random order, shared (or no) storage, Service load-balances across all.

A database or clustered system (PostgreSQL primary/replica, Kafka, ZooKeeper, etcd, Cassandra, Elasticsearch, Redis cluster) needs:
1. **Stable identity** — node 0 is always node 0; peers find each other by predictable names.
2. **Stable storage** — when `db-0` is rescheduled, it must get **its own** disk back, not db-1's.
3. **Ordered operations** — start primary before replicas; shut down in reverse.

That is a **StatefulSet**.

### StatefulSet vs Deployment

| Aspect | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix | Ordinal: `name-0`, `name-1`, `name-2` |
| Pod identity after reschedule | New random name | **Same** name |
| Start order | All at once | Ordered `0 -> 1 -> 2` (default `OrderedReady`) |
| Stop/scale-down order | Any | Reverse `2 -> 1 -> 0` |
| Storage | Same PVC shared by all replicas (or none) | **Own PVC per replica** via `volumeClaimTemplates` |
| DNS | Only via Service VIP | Per-Pod DNS through **headless Service**: `db-0.db.ns.svc.cluster.local` |
| Rolling update | Surge + ready Pods | One Pod at a time, highest ordinal first |
| Typical workloads | Web, API, workers | Databases, brokers, consensus clusters |

**Rule of thumb:** if losing a Pod's local data matters, or peers address each other by name, use a StatefulSet. If you can kill any Pod any time with no consequences, use a Deployment.

Honest note for the course: running production databases on Kubernetes is possible but operationally hard (backups, failover, upgrades). In real life many teams use managed DBs (RDS, Cloud SQL) or an **Operator** (CloudNativePG, Percona, Zalando). StatefulSet is the underlying building block you must understand.

### Headless Service

StatefulSet requires a **headless Service** (`clusterIP: None`) referenced in `spec.serviceName`. Headless means: no virtual IP, no load balancing. DNS returns the **individual Pod IPs**, and each Pod gets its own A record.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: demo
spec:
  clusterIP: None            # headless
  selector:
    app: postgres
  ports:
    - name: pg
      port: 5432
```

DNS names created:

| Name | Resolves to |
|---|---|
| `postgres.demo.svc.cluster.local` | All Pod IPs (multiple A records) |
| `postgres-0.postgres.demo.svc.cluster.local` | IP of Pod `postgres-0` only |
| `postgres-1.postgres.demo.svc.cluster.local` | IP of Pod `postgres-1` only |

Pattern: `<pod-name>.<headless-svc>.<namespace>.svc.cluster.local`. Within the same namespace you can shorten to `postgres-0.postgres`. These names stay valid after the Pod is rescheduled (the IP changes, the name does not). That is how a replica finds "the primary at `postgres-0.postgres`".

### StatefulSet anatomy

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless          # REQUIRED: headless Service name
  replicas: 3
  podManagementPolicy: OrderedReady  # or Parallel
  updateStrategy:
    type: RollingUpdate              # or OnDelete
    rollingUpdate:
      partition: 0                   # only Pods with ordinal >= partition get updated
  persistentVolumeClaimRetentionPolicy:   # 1.27+ stable
    whenDeleted: Retain              # Retain | Delete  — PVCs when StatefulSet is deleted
    whenScaled: Retain               # Retain | Delete  — PVCs of removed replicas
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 1Gi
```

Notes:
- `volumeClaimTemplates` creates **one PVC per replica** named `<template-name>-<statefulset-name>-<ordinal>` → `www-web-0`, `www-web-1`, `www-web-2`.
- Pod `web-1` always remounts `www-web-1`, wherever it is scheduled.
- `podManagementPolicy: Parallel` launches/terminates all Pods at once (still stable names and storage) — use when ordering does not matter.
- `updateStrategy.partition` enables **canary** updates: set `partition: 2` and only `web-2` is updated; lower it step by step.
- **PVCs are not deleted on scale-down or StatefulSet delete by default** (safety). Scale 3 -> 1 and `www-web-1`, `www-web-2` remain; scale back up and the Pods reattach. Clean up manually, or set `persistentVolumeClaimRetentionPolicy`.

#### Scale and ordering behaviour

```
scale 0 -> 3 :  web-0 Ready -> web-1 created -> web-1 Ready -> web-2 created
scale 3 -> 1 :  web-2 terminated (waits fully) -> web-1 terminated
delete pod web-1 : recreated as web-1, same PVC, same DNS name
```

With `OrderedReady`, if `web-0` is stuck not-Ready, `web-1` is **never created** — a frequent source of "why is my StatefulSet stuck at 1/3".

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl get pv` | List PVs (cluster-wide) | `kubectl get pv` |
| `kubectl get pvc -n NS` | List claims in a namespace | `kubectl get pvc -n demo` |
| `kubectl get sc` | List StorageClasses (shows default) | `kubectl get storageclass` |
| `kubectl describe pvc NAME` | Binding state, events, why Pending | `kubectl describe pvc app-data -n demo` |
| `kubectl describe pv NAME` | Claim ref, reclaim policy, source | `kubectl describe pv pv-lab-01` |
| `kubectl get pv -o wide` | Show claim, class, reason | `kubectl get pv -o wide` |
| `kubectl patch pv NAME -p ...` | Change reclaim policy | `kubectl patch pv pv1 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'` |
| `kubectl patch pvc NAME -p ...` | Expand a PVC | `kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'` |
| `kubectl get sts` | List StatefulSets (`READY` column) | `kubectl get statefulset -n demo` |
| `kubectl scale sts NAME --replicas=N` | Scale a StatefulSet | `kubectl scale sts postgres --replicas=3` |
| `kubectl rollout status sts/NAME` | Watch a StatefulSet update | `kubectl rollout status sts/postgres` |
| `kubectl rollout undo sts/NAME` | Roll back StatefulSet template | `kubectl rollout undo sts/postgres` |
| `kubectl delete pod NAME` | Recreate Pod (data stays on PVC) | `kubectl delete pod postgres-0` |
| `kubectl delete sts NAME --cascade=orphan` | Delete StatefulSet but keep Pods running | `kubectl delete sts web --cascade=orphan` |
| `kubectl exec -it POD -- CMD` | Run command inside Pod | `kubectl exec -it postgres-0 -- psql -U postgres` |
| `kubectl get events --sort-by=.lastTimestamp` | Recent events | `kubectl get events -n demo --sort-by=.lastTimestamp` |
| `kubectl run tmp --rm -it --image=busybox -- nslookup X` | DNS check from a throwaway Pod | `kubectl run tmp --rm -it --image=busybox:1.36 -- nslookup postgres-0.postgres` |

### Reading `kubectl get pvc`

```
NAME        STATUS    VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
app-data    Bound     pv-lab-01   1Gi        RWO            manual         3m
pg-wait     Pending                                         local-path     10s
```

| STATUS | Meaning |
|---|---|
| `Pending` | Not bound yet (see `describe` events) |
| `Bound` | Bound to PV in `VOLUME` column |
| `Lost` | Bound PV disappeared |
| `Terminating` | Delete requested, still in use by Pod (finalizer) |

## Examples / Walkthrough

### Walkthrough 1 — static PV + PVC + Pod (hostPath, single-node lab)

Use minikube, kind, k3s, or any single-node cluster. On multi-node clusters `hostPath` data lives on whichever node the Pod lands on — fine for a demo, wrong for real use.

```bash
kubectl create namespace demo
kubectl config set-context --current --namespace=demo
```

```yaml
# pv-lab.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-lab-01
spec:
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/pv-lab-01
    type: DirectoryOrCreate
---
# pvc-lab.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-claim
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
---
# pod-lab.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "date >> /data/history.txt; cat /data/history.txt; sleep 3600"]
      volumeMounts:
        - name: store
          mountPath: /data
  volumes:
    - name: store
      persistentVolumeClaim:
        claimName: lab-claim
```

```bash
kubectl apply -f pv-lab.yaml
kubectl get pv                          # STATUS Available
kubectl apply -f pvc-lab.yaml
kubectl get pv,pvc                      # both Bound, PVC shows CAPACITY 1Gi (PV size, not 500Mi)
kubectl apply -f pod-lab.yaml
kubectl logs lab-pod                    # one date line

# Persistence proof
kubectl delete pod lab-pod
kubectl apply -f pod-lab.yaml
kubectl logs lab-pod                    # TWO date lines: old data survived

# Reclaim behaviour
kubectl delete pod lab-pod
kubectl delete pvc lab-claim
kubectl get pv                          # STATUS Released (Retain), CLAIM still shows demo/lab-claim
kubectl apply -f pvc-lab.yaml
kubectl get pvc                         # Pending! Released PV is not reusable automatically
# Make PV Available again
kubectl patch pv pv-lab-01 --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'
kubectl get pv,pvc                      # PVC binds again
```

Teaching points: PVC shows PV size (1Gi) even though it asked 500Mi; `Released` PV needs `claimRef` removed; data on disk is still there because policy is `Retain`.

### Walkthrough 2 — dynamic provisioning with the default StorageClass

```bash
kubectl get storageclass
# NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
# local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false
```

```yaml
# dyn-pvc.yaml  (no storageClassName => default class)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dyn-claim
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f dyn-pvc.yaml
kubectl get pvc dyn-claim               # Pending (WaitForFirstConsumer) — expected!
kubectl describe pvc dyn-claim | tail   # "waiting for first consumer to be created before binding"

kubectl run dyn-pod --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"dyn-pod","image":"busybox:1.36","command":["sleep","3600"],"volumeMounts":[{"name":"d","mountPath":"/data"}]}],"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"dyn-claim"}}]}}'

kubectl get pvc,pv                      # PVC Bound, a pvc-<uuid> PV appeared automatically
kubectl get pv -o custom-columns=NAME:.metadata.name,CLAIM:.spec.claimRef.name,POLICY:.spec.persistentVolumeReclaimPolicy
kubectl delete pod dyn-pod
kubectl delete pvc dyn-claim
kubectl get pv                          # PV gone too (reclaimPolicy Delete)
```

### Walkthrough 3 — StatefulSet with per-Pod storage (nginx)

```yaml
# sts-web.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
      name: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Mi
```

```bash
kubectl apply -f sts-web.yaml
kubectl get pods -w                     # web-0 first, then web-1, then web-2 (watch ordering)
kubectl get pvc                         # www-web-0, www-web-1, www-web-2

# each Pod writes its own identity to its own disk
for i in 0 1 2; do
  kubectl exec web-$i -- sh -c 'echo "I am $(hostname)" > /usr/share/nginx/html/index.html'
done

# Stable DNS
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'nslookup web-0.web-headless; wget -qO- web-1.web-headless'
# -> "I am web-1"

# Kill a Pod: same name, same data
kubectl delete pod web-1
kubectl get pods -w                     # web-1 recreated with the same name
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html    # "I am web-1" still there

# Scale down: PVCs stay
kubectl scale sts web --replicas=1
kubectl get pods                        # only web-0
kubectl get pvc                         # all three PVCs still exist
kubectl scale sts web --replicas=3
kubectl exec web-2 -- cat /usr/share/nginx/html/index.html    # old data reattached
```

### Walkthrough 4 — Hands-on: PostgreSQL as a StatefulSet with PVC

> **Note on the password.** This lesson has not yet introduced ConfigMaps or Secrets, so the password below is written as a plain `value:` in the manifest. **That is for the lab only** — anything in a manifest is visible to anyone who can read it (and in Git). **Lesson 42 replaces this with a Kubernetes Secret, and lessons 43-44 replace it again with External Secrets Operator pulling from Vault.**

```yaml
# postgres.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: db
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: db
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: pg
      port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: db
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
              name: pg
          env:
            - name: POSTGRES_PASSWORD
              value: "labpassword"          # LAB ONLY — lesson 42 moves this to a Secret
            - name: POSTGRES_DB
              value: shop
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata   # subdirectory, see pitfall below
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              memory: 512Mi
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        # storageClassName: omitted => default class (check `kubectl get sc`)
        resources:
          requests:
            storage: 2Gi
```

Deploy and verify:

```bash
kubectl apply -f postgres.yaml
kubectl -n db get sts,pods,pvc,pv
kubectl -n db rollout status sts/postgres
kubectl -n db logs postgres-0 | tail -5      # "database system is ready to accept connections"
```

Write data:

```bash
kubectl -n db exec -it postgres-0 -- psql -U postgres -d shop -c \
  "CREATE TABLE orders(id serial PRIMARY KEY, item text);
   INSERT INTO orders(item) VALUES ('keyboard'),('monitor');
   SELECT * FROM orders;"
```

Prove persistence — delete the Pod and check data returns:

```bash
kubectl -n db delete pod postgres-0
kubectl -n db get pods -w                    # StatefulSet recreates postgres-0 (Ctrl+C when Running)
kubectl -n db exec -it postgres-0 -- psql -U postgres -d shop -c "SELECT * FROM orders;"
# keyboard + monitor are still there
```

Connect from another Pod with the stable DNS name:

```bash
kubectl -n db run client --rm -it --image=postgres:16 --restart=Never --env PGPASSWORD=labpassword -- \
  psql -h postgres-0.postgres.db.svc.cluster.local -U postgres -d shop -c "SELECT count(*) FROM orders;"
```

Prove the data outlives even the StatefulSet:

```bash
kubectl -n db delete sts postgres            # Pods gone
kubectl -n db get pvc                        # data-postgres-0 STILL EXISTS
kubectl apply -f postgres.yaml               # new StatefulSet re-adopts the same PVC
kubectl -n db exec -it postgres-0 -- psql -U postgres -d shop -c "SELECT * FROM orders;"
```

Look inside the volume object:

```bash
kubectl -n db get pvc data-postgres-0 -o wide
kubectl get pv $(kubectl -n db get pvc data-postgres-0 -o jsonpath='{.spec.volumeName}') -o yaml | head -40
```

Cleanup (this one **does** destroy data when reclaimPolicy is `Delete`):

```bash
kubectl delete -f postgres.yaml
kubectl -n db delete pvc --all
kubectl delete ns db
```

### Walkthrough 5 — expanding a volume

```bash
kubectl get sc -o custom-columns=NAME:.metadata.name,EXPAND:.allowVolumeExpansion
kubectl -n db patch pvc data-postgres-0 -p '{"spec":{"resources":{"requests":{"storage":"4Gi"}}}}'
kubectl -n db describe pvc data-postgres-0 | grep -A3 Conditions
kubectl -n db get pvc data-postgres-0 -w
# StatefulSet note: volumeClaimTemplates are immutable — patching the template is rejected.
# Expand each PVC directly, then (optionally) recreate the StatefulSet with --cascade=orphan to update the template.
```

If `allowVolumeExpansion` is false: error `only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize`.

### Walkthrough 6 — troubleshooting a Pending PVC (instructor demo)

```bash
# 1. Wrong class name
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: broken1}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: does-not-exist
  resources: {requests: {storage: 1Gi}}
EOF
kubectl describe pvc broken1 | tail -3     # storageclass.storage.k8s.io "does-not-exist" not found

# 2. Static claim, no matching PV (asks 100Gi, only 1Gi PV exists)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: broken2}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: manual
  resources: {requests: {storage: 100Gi}}
EOF
kubectl describe pvc broken2 | tail -3     # no persistent volumes available for this claim and no storage class is set

# 3. Wrong access mode (RWX on RWO-only class)
# 4. Pod references claim in another namespace -> "persistentvolumeclaim not found"
kubectl delete pvc broken1 broken2
```

Diagnostic checklist:
1. `kubectl describe pvc X` — read **Events** first.
2. `kubectl get sc` — does the class exist? Is there a default? Is it `WaitForFirstConsumer` (Pending is normal until a Pod uses it)?
3. `kubectl get pv` — any `Available` PV with matching class, size >= request, and modes?
4. `kubectl -n kube-system get pods` — is the provisioner / CSI controller running?
5. `kubectl get events -A --sort-by=.lastTimestamp | tail -20`.

## Common Pitfalls

- **Using `hostPath` for database data** — Pod rescheduled to another node starts with an empty directory. `hostPath` is for node agents and single-node labs only.
- **PVC stuck `Pending` and student panics** — with `WaitForFirstConsumer` that is normal until a Pod mounts it. Check `describe` before "fixing" anything.
- **PVC and Pod in different namespaces** — PVCs are namespaced; a Pod can only mount a PVC from its **own** namespace. PVs are cluster-wide.
- **Misspelled `storageClassName`** — PVC Pending forever, event `storageclass ... not found`. Omit the field to get the default class, use `""` only when you really mean "no class".
- **Expecting a Released PV to be reused** — with `Retain`, the PV keeps its `claimRef` and stays `Released`. Remove `claimRef` (or delete and recreate the PV) after salvaging the data.
- **`reclaimPolicy: Delete` on production data** — `kubectl delete pvc` destroys the disk. Patch PV to `Retain`, or use a Retain StorageClass.
- **Requesting RWX on block storage** — EBS/PD/local-path support RWO only. Use NFS/CephFS/EFS, or redesign.
- **RWO is per node, not per Pod** — two Pods on the **same** node can share an RWO volume; two on different nodes cannot. Rolling a Deployment that holds an RWO PVC can deadlock: new Pod lands on another node while the old one still holds the disk (`Multi-Attach error`). Use `strategy: Recreate` or a StatefulSet.
- **PostgreSQL `initdb: directory exists but is not empty` / `lost+found`** — fresh ext4 volumes contain `lost+found` at the mount root. Set `PGDATA=/var/lib/postgresql/data/pgdata` (a subdirectory) or use `subPath`.
- **Deleting a StatefulSet and assuming data is gone (or safe)** — PVCs survive by default; a new StatefulSet with the same name re-adopts them. Delete PVCs explicitly when you want a clean slate.
- **Scaling StatefulSet down to save money, then forgetting PVCs** — you keep paying for the disks. Clean up leftover PVCs.
- **Stuck at `1/3 Ready`** — `OrderedReady` waits for `web-0` to be Ready before creating `web-1`. Fix pod 0 (or use `podManagementPolicy: Parallel` if order is irrelevant).
- **Editing `volumeClaimTemplates`** — immutable; the API rejects the change. Expand existing PVCs directly instead.
- **Forgetting the headless Service** — StatefulSet needs `serviceName` pointing at an existing `clusterIP: None` Service; without it, per-Pod DNS names do not resolve.
- **Shrinking a PVC** — not supported; only growth.
- **Two default StorageClasses** — new PVCs without a class become ambiguous. Keep exactly one default.
- **Plain-text password in the manifest** — acceptable only in this lab; replaced by Secrets (lesson 42) and ESO (lessons 43-44).
- **Treating a single-replica StatefulSet as "highly available"** — one Pod + one disk is still one failure domain; replication, backups and failover are the application's job (or an Operator's).
- **PV/PVC stuck `Terminating`** — protection finalizer: a Pod still uses the PVC. Delete the Pod first. Only as last resort patch out the finalizer (`kubectl patch pvc X -p '{"metadata":{"finalizers":null}}'`) and understand you bypass the safety.

## FAQ

**Q: What is the difference between PV and PVC?**
A: A PV is the actual storage resource (cluster-scoped, usually created by admin or provisioner). A PVC is a namespaced request for storage by a user. The PVC binds one-to-one to a PV that satisfies it, and Pods mount the PVC, never the PV directly.

**Q: Why is my PVC Pending?**
A: Run `kubectl describe pvc <name>`. Common causes: (1) StorageClass uses `WaitForFirstConsumer` and no Pod uses the claim yet (normal); (2) StorageClass name does not exist; (3) no static PV matches size/modes/class; (4) provisioner/CSI driver not running; (5) RWX asked from RWO-only storage; (6) quota exceeded.

**Q: Does data survive if the Pod is deleted?**
A: Data on the container filesystem — no. Data on `emptyDir` — only while the Pod exists (survives container restart, lost on Pod delete). Data on a PVC — yes, it survives Pod deletion, node failure (for network storage), and even StatefulSet deletion (unless reclaim/retention policy removes it).

**Q: Does `emptyDir` survive a container crash?**
A: Yes. The directory belongs to the Pod; kubelet restarting a crashed container remounts the same `emptyDir`. It is lost when the Pod is deleted or evicted.

**Q: What does RWO really mean?**
A: Mountable read-write by a single *node*. Multiple Pods on that node can use it. It does not mean "one Pod". For strictly one Pod use `ReadWriteOncePod`.

**Q: When do I need RWX?**
A: When Pods on different nodes must write to the same files — shared uploads directory, CMS media, shared build cache. It requires file storage like NFS, CephFS, EFS, Azure Files. Databases do **not** need RWX: each replica should have its own RWO disk.

**Q: What is the difference between Retain and Delete?**
A: `Delete`: removing the PVC also removes the PV and the underlying disk. `Retain`: PV and data stay, the PV becomes `Released`, and an admin must clean up. Dynamic classes default to `Delete`; use `Retain` for important data.

**Q: How do I reuse a Released PV?**
A: Copy/backup data if needed, then remove the stale reference: `kubectl patch pv NAME --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'`. PV returns to `Available`. Or delete the PV object and recreate it pointing at the same backend path.

**Q: What is a StorageClass and do I always need one?**
A: A StorageClass names a kind of storage and the provisioner that creates it, enabling automatic PV creation when a PVC is made. For static PVs you do not strictly need one (use a label string like `manual` or `""`), but in real clusters you use dynamic provisioning.

**Q: What happens if I omit `storageClassName` in a PVC?**
A: The cluster's *default* StorageClass is used. If no default exists, the PVC stays Pending (unless a classless PV matches). Setting `storageClassName: ""` explicitly means "no class — bind only to classless PVs".

**Q: What is `WaitForFirstConsumer`?**
A: A `volumeBindingMode` that delays creating/binding the volume until a Pod using the PVC is scheduled, so the disk is created where the Pod actually runs (right zone/node). It is the right choice for zonal and local storage. The opposite is `Immediate`.

**Q: Can I resize a PVC?**
A: Yes, if the StorageClass has `allowVolumeExpansion: true`. Increase `spec.resources.requests.storage` on the PVC. Only growth is supported. Some drivers need the Pod restarted to grow the filesystem.

**Q: Why use a StatefulSet instead of a Deployment with a PVC?**
A: A Deployment with one PVC shares that one disk among all replicas (impossible with RWO across nodes) and gives Pods random names. A StatefulSet gives each replica its own PVC, a fixed name, a fixed DNS entry, and ordered start/stop — what clustered databases need.

**Q: Why does StatefulSet need a headless Service?**
A: The headless Service (`clusterIP: None`) provides the DNS domain for per-Pod records (`pod-0.service.ns.svc.cluster.local`). Without it Pods have stable names but no resolvable DNS names. `spec.serviceName` must point to it.

**Q: Are StatefulSet PVCs deleted when I delete the StatefulSet or scale down?**
A: Not by default. They are kept to prevent data loss. Since Kubernetes 1.27 you can opt in with `persistentVolumeClaimRetentionPolicy` (`whenDeleted`, `whenScaled`: `Retain` or `Delete`). Otherwise delete PVCs manually.

**Q: How do I connect an app to a StatefulSet database?**
A: Use the Pod's stable DNS: `postgres-0.postgres.db.svc.cluster.local:5432`. A normal (non-headless) Service in front of the Pods is common for read/write routing; the headless one is for identity. Production setups often add a second ClusterIP Service for clients.

**Q: Is it a good idea to run PostgreSQL in Kubernetes?**
A: It works and is common, but you own backups, failover, upgrades, and storage performance. Use an Operator (CloudNativePG, Zalando, Percona) or a managed database for production. StatefulSet-from-scratch is how you learn the building blocks.

**Q: Why did my Pod get `Multi-Attach error for volume`?**
A: An RWO block volume is still attached to the old node while the new Pod is scheduled to another node. Wait for detach, use `strategy: Recreate`, or use a StatefulSet. Force-deleting a Pod on a dead node can cause this.

**Q: Where does `local-path` store data?**
A: On the node that ran the Pod, under `/opt/local-path-provisioner/` (k3s/rke2 default). It ties the volume to that node: if the node dies, data is unavailable. Fine for labs and edge, not for HA.

**Q: Can two PVCs bind to the same PV?**
A: No. Binding is exclusive and one-to-one. To share data between Pods use one PVC (RWX or same node), not two.

**Q: What are VolumeSnapshots?**
A: A CSI feature (`VolumeSnapshot`, `VolumeSnapshotClass`) to take point-in-time snapshots of PVCs and restore new PVCs from them. Useful for backups and cloning. Mentioned here only for awareness; requires a CSI driver with snapshot support.

**Q: How do I back up PVC data?**
A: Options: CSI snapshots, an application-level dump (`pg_dump` via CronJob to object storage), or tools like Velero. A PVC is not a backup — it is just persistence.

**Q: How do I see which Pod uses a PVC?**
A: `kubectl describe pvc NAME` shows `Used By:`. For all: `kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumes[*].persistentVolumeClaim.claimName}{"\n"}{end}'`.

## Practice / Exercise

### Core

1. **Ephemeral vs persistent.** Create a Pod with an `emptyDir` volume, write a file, delete the container process (`kubectl exec POD -- kill 1` style restart) and confirm the file remains; delete the Pod and confirm it is gone.
2. **Static PV/PVC.** Create a 1Gi hostPath PV (`Retain`, class `manual`), a 500Mi PVC, and a Pod that appends a timestamp to a file. Delete and recreate the Pod twice; show the file keeps growing. Explain why the PVC shows 1Gi.
3. **Release cycle.** Delete the PVC, observe PV `Released`, create the PVC again (Pending), then fix it by removing `claimRef`.
4. **Dynamic provisioning.** Run `kubectl get sc`; identify the default class, provisioner, reclaim policy, and binding mode. Create a PVC without `storageClassName`, observe Pending then Bound after a Pod mounts it. Find the generated PV.
5. **StatefulSet basics.** Deploy the 3-replica nginx StatefulSet with `volumeClaimTemplates`. List PVC names, write a unique `index.html` in each Pod, delete `web-1`, and show it returns with the same name and content.
6. **DNS identity.** From a temporary Pod, resolve `web-0.web-headless` and curl `web-2.web-headless`. Resolve `web-headless` alone and explain why you get several IPs.
7. **PostgreSQL hands-on.** Deploy PostgreSQL as a StatefulSet (manifest in Walkthrough 4), create a table with two rows, delete `postgres-0`, and confirm the rows survive. Then delete the StatefulSet (not the PVC), re-apply, and confirm again.
8. **Scale and PVC retention.** Scale `web` from 3 to 1 to 3. Which PVCs existed at each step? Which Pod names were created in which order?

### Stretch

1. **Reclaim policy experiment.** Create two PVCs from a class with `Delete` and `Retain` (create a second StorageClass copy with `reclaimPolicy: Retain` if the cluster allows it). Delete both PVCs and compare what happens to PV and data.
2. **Expansion.** On a class with `allowVolumeExpansion: true`, grow a PVC from 1Gi to 3Gi while a Pod is running; verify with `df -h` inside the Pod. Try shrinking and record the error.
3. **Canary StatefulSet update.** Change the image of the nginx StatefulSet, set `updateStrategy.rollingUpdate.partition: 2`, and confirm only `web-2` updates. Lower the partition to 0 to finish. Then `kubectl rollout undo sts/web`.
4. **`Parallel` vs `OrderedReady`.** Recreate the StatefulSet with `podManagementPolicy: Parallel`; compare startup order with `kubectl get pods -w`. Make `web-0` fail (bad image) with `OrderedReady` and explain why `web-1` never appears.
5. **PVC retention policy.** Add `persistentVolumeClaimRetentionPolicy: {whenDeleted: Delete, whenScaled: Delete}`; scale down and delete the StatefulSet and verify PVCs are removed.
6. **Break and fix.** Create three broken claims (wrong class name, size too large for static PV, RWX on RWO class). Diagnose each using only `describe` events and fix.
7. **Backup.** Write a `pg_dump` command run via `kubectl exec` that saves the `shop` database to your laptop; restore it into a fresh PostgreSQL StatefulSet in a second namespace.

## Further Reading

- Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Storage Classes: https://kubernetes.io/docs/concepts/storage/storage-classes/
- Volumes (emptyDir, hostPath, ...): https://kubernetes.io/docs/concepts/storage/volumes/
- StatefulSets: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Run a Replicated Stateful Application tutorial: https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/
- Dynamic volume provisioning: https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/
- Expanding PVCs: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims
- Volume Snapshots: https://kubernetes.io/docs/concepts/storage/volume-snapshots/
- Rancher local-path-provisioner: https://github.com/rancher/local-path-provisioner
- CloudNativePG (PostgreSQL operator): https://cloudnative-pg.io/

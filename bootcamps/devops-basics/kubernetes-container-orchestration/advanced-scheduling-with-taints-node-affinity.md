# Lesson 46: Advanced Scheduling with Taints & Node Affinity

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-45 (architecture, kubectl, Pods/Deployments, Services, Ingress, storage, ConfigMaps/Secrets, probes, requests/limits, QoS, LimitRange, ResourceQuota). Student can write Deployment YAML, label objects, and read `kubectl describe` events. Multi-node lab cluster (kind with 3 workers, minikube `--nodes 3`, or the rke2/kubeadm cluster) is strongly recommended.

## Learning Objectives

By end of lesson student can:
- Describe the scheduler's two phases (filter, score) and explain why a Pod ends up Pending
- Place a Pod with `nodeName` and `nodeSelector`, and explain why `nodeName` is rarely correct
- Add, list, and remove node labels and node taints with `kubectl label` and `kubectl taint`
- Explain taint effects `NoSchedule`, `PreferNoSchedule`, `NoExecute` and write tolerations (`Equal`, `Exists`, `tolerationSeconds`)
- Explain why a toleration does not attract a Pod to a node, only permits it
- Write `nodeAffinity` with `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`, using operators `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`
- Use `podAffinity` and `podAntiAffinity` with `topologyKey` to co-locate or spread replicas
- Use `topologySpreadConstraints` (`maxSkew`, `topologyKey`, `whenUnsatisfiable`, `labelSelector`) to spread evenly
- Combine taint + label + affinity to build dedicated node pools
- Diagnose a `Pending` Pod from `FailedScheduling` events
- Use `cordon`, `drain`, and `uncordon` for node maintenance

## Topics

- nodeSelector: simple label-based scheduling; kubectl label nodes
- Taints and tolerations: kubectl taint, effects (NoSchedule, PreferNoSchedule, NoExecute)
- nodeAffinity: requiredDuringScheduling, preferredDuringScheduling; operators (In, NotIn, Exists)
- podAffinity and podAntiAffinity: co-location and spread rules; topologyKey (concept + example)
- topologySpreadConstraints: maxSkew, whenUnsatisfiable (ScheduleAnyway/DoNotSchedule) — concept
- Recap of scheduler phases, `nodeName`, built-in node taints, dedicated-node pattern, cordon/drain, troubleshooting Pending Pods

## Concepts

### How the scheduler decides (recap from lesson 34)

`kube-scheduler` watches for Pods whose `spec.nodeName` is empty. For each one it runs two phases:

1. **Filtering** — remove nodes that cannot run the Pod: not enough allocatable CPU/memory for the **requests** (lesson 45), a taint the Pod does not tolerate, a `nodeSelector` or required affinity that does not match, a port conflict, a PVC that cannot attach on that node, node marked unschedulable.
2. **Scoring** — rank the surviving nodes (spread, image already present, preferred affinity weights, resource balance). Highest score wins; ties are broken randomly.

The winner is written into the Pod as `spec.nodeName` by a **Binding**. `kubelet` on that node then starts the containers. If filtering leaves zero nodes, the Pod stays `Pending` and the scheduler records a `FailedScheduling` event explaining each rejection ("0/4 nodes are available: 1 node(s) had untolerated taint ..., 3 node(s) didn't match Pod's node affinity/selector").

Everything in this lesson is a **constraint or hint fed into those two phases**:

| Tool | Direction | Strength | Who sets it |
|---|---|---|---|
| `nodeName` | Pod → one specific node | Bypasses scheduler | Pod author |
| `nodeSelector` | Pod → nodes with labels | Hard | Pod author |
| `nodeAffinity` | Pod → nodes with labels | Hard (required) or soft (preferred) | Pod author |
| Taints + tolerations | Node → repels Pods | Hard (`NoSchedule`/`NoExecute`) or soft (`PreferNoSchedule`) | Node admin (taint) + Pod author (toleration) |
| `podAffinity` | Pod → near other Pods | Hard or soft | Pod author |
| `podAntiAffinity` | Pod → away from other Pods | Hard or soft | Pod author |
| `topologySpreadConstraints` | Pod → even spread across domains | Hard or soft | Pod author |

Memory aid: **affinity/selector = "Pod picks node" (attract); taint = "node picks Pod" (repel)**.

### Node labels: the foundation

All node-targeting features match against **node labels**. Every node already carries some automatically:

```bash
kubectl get nodes --show-labels
kubectl get nodes -L kubernetes.io/hostname,topology.kubernetes.io/zone,kubernetes.io/arch
```

Well-known labels (set by kubelet / cloud provider):

| Label | Meaning |
|---|---|
| `kubernetes.io/hostname` | Node name — unique per node; the standard `topologyKey` for "per node" |
| `kubernetes.io/os`, `kubernetes.io/arch` | `linux`/`windows`, `amd64`/`arm64` |
| `topology.kubernetes.io/zone` | Availability zone (cloud) — standard `topologyKey` for "per zone" |
| `topology.kubernetes.io/region` | Region |
| `node.kubernetes.io/instance-type` | Cloud instance type, e.g. `m5.large` |
| `node-role.kubernetes.io/control-plane` | Empty-value label marking control-plane nodes |

Add your own for hardware or purpose:

```bash
kubectl label node worker-1 disktype=ssd
kubectl label node worker-2 disktype=hdd
kubectl label node worker-1 gpu=true team=ml
kubectl label node worker-1 disktype=nvme --overwrite   # change an existing value
kubectl label node worker-1 gpu-                         # trailing dash removes the label
kubectl get nodes -l disktype=ssd
```

Labels are cheap and reversible. Taints are not trivial to forget — they evict or block workloads. Keep them separate in your head.

### nodeName (and why not to use it)

`spec.nodeName` skips the scheduler entirely: the kubelet on that node is told to run the Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pinned
spec:
  nodeName: worker-1
  containers:
    - name: app
      image: nginx:1.27
```

Problems: if `worker-1` is down, the Pod just fails (no rescheduling to another node for a bare Pod); no resource-fit check, so the Pod can fail with `OutOfcpu`; taints are **not** checked by the scheduler (kubelet admission still rejects `NoExecute`). Useful only for debugging or custom schedulers. In real manifests use `nodeSelector` or `nodeAffinity`.

### nodeSelector: simplest placement

`nodeSelector` is a map of label key/values. A node must carry **all** of them (logical AND). Exact match only.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-app
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: app
      image: nginx:1.27
```

```bash
kubectl apply -f ssd-app.yaml
kubectl get pod ssd-app -o wide        # NODE column shows a node labelled disktype=ssd
```

No node with `disktype=ssd`? Pod stays Pending: `0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.` Add the label to a node and the Pod schedules on its own within seconds — the scheduler retries.

Limits of `nodeSelector`: only equality, only AND, no "prefer", no `NotIn`/`Exists`/numeric. Those gaps are why `nodeAffinity` exists. For new manifests nodeSelector remains fine for simple cases.

### Taints and tolerations

A **taint** is a property on a **node**: `key=value:effect` (value optional). It says "do not put Pods here unless they tolerate this." A **toleration** is a property on a **Pod** saying "I accept this taint."

```bash
kubectl taint nodes worker-2 dedicated=gpu:NoSchedule        # add
kubectl taint nodes worker-2 dedicated=gpu:NoSchedule-       # remove (trailing dash, same key+effect)
kubectl taint nodes worker-2 dedicated-                      # remove all taints with key "dedicated"
kubectl describe node worker-2 | grep -A3 Taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

#### Three effects

| Effect | New Pods without toleration | Pods already running without toleration |
|---|---|---|
| `NoSchedule` | Not scheduled | Stay (not touched) |
| `PreferNoSchedule` | Scheduler tries to avoid; may still place if nothing else fits | Stay |
| `NoExecute` | Not scheduled | **Evicted** (immediately, or after `tolerationSeconds`) |

`NoSchedule` is the everyday choice. `NoExecute` is for "get everything off this node" (node problems, strict isolation). `PreferNoSchedule` is a soft hint.

#### Writing a toleration

```yaml
spec:
  tolerations:
    - key: dedicated
      operator: Equal        # key AND value must match
      value: gpu
      effect: NoSchedule
```

Operators:

| Operator | Meaning | Example |
|---|---|---|
| `Equal` (default) | key, value, effect all match | `key: dedicated, operator: Equal, value: gpu, effect: NoSchedule` |
| `Exists` | key matches, **no value field** | `key: dedicated, operator: Exists, effect: NoSchedule` |

Special forms:

```yaml
tolerations:
  - operator: Exists                 # empty key + Exists = tolerate EVERY taint (used by node agents)
  - key: dedicated
    operator: Exists                 # any value, any effect (effect omitted = all effects)
```

Matching rule: a Pod is blocked by a node only if the node has at least one taint (with effect `NoSchedule`/`NoExecute`) that **no** toleration matches. Multiple taints on a node: Pod must tolerate **all** of them.

#### Critical point: toleration ≠ attraction

A toleration only **permits** the Pod to land on the tainted node. It does not **pull** it there. A Pod tolerating `dedicated=gpu` can still go to any untainted node. To force Pods onto the tainted nodes you also need `nodeSelector`/`nodeAffinity` (see "Dedicated nodes" below). Students mix this up constantly.

#### tolerationSeconds (NoExecute only)

```yaml
tolerations:
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 60
```

The Pod may stay on the node for 60 s after the taint appears, then it is evicted. Omit `tolerationSeconds` = tolerate forever.

#### Built-in taints (added by the node lifecycle controller)

| Taint | Added when | Effect |
|---|---|---|
| `node.kubernetes.io/not-ready` | Node `Ready` condition false | `NoExecute` |
| `node.kubernetes.io/unreachable` | Node unreachable from control plane | `NoExecute` |
| `node.kubernetes.io/memory-pressure` | Node low on memory | `NoSchedule` |
| `node.kubernetes.io/disk-pressure` | Node low on disk | `NoSchedule` |
| `node.kubernetes.io/pid-pressure` | Too many processes | `NoSchedule` |
| `node.kubernetes.io/network-unavailable` | Network not configured yet | `NoSchedule` |
| `node.kubernetes.io/unschedulable` | `kubectl cordon` | `NoSchedule` |

Kubernetes automatically gives every Pod (admission controller `DefaultTolerationSeconds`) tolerations for `not-ready` and `unreachable` with `tolerationSeconds: 300`. That is why a Pod on a dead node is evicted after about **5 minutes**, not instantly. Check any Pod:

```bash
kubectl get pod <name> -o jsonpath='{.spec.tolerations}' | jq
```

#### Control-plane taint

kubeadm/rke2 server nodes carry:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

so user workloads stay off them. That is why DaemonSets like `kube-proxy` and CNI agents carry tolerations. Single-node labs (minikube, kind with only a control-plane) remove it:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

### nodeAffinity: nodeSelector, but expressive

`nodeAffinity` lives under `spec.affinity.nodeAffinity` and has two flavors:

| Field | Meaning |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Hard** rule: node must match or the Pod stays Pending |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Soft** rule: scheduler adds `weight` to the score of matching nodes; Pod still schedules if none match |

**`IgnoredDuringExecution`** means: the rule is evaluated only **at scheduling time**. If node labels change later and the node no longer matches, the running Pod is **not** evicted. (A `requiredDuringExecution` variant was planned but does not exist.) The name is long but this is its whole point.

#### Required (hard) example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-required
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd", "nvme"]
              - key: kubernetes.io/arch
                operator: In
                values: ["amd64"]
  containers:
    - name: app
      image: nginx:1.27
```

Logic: inside one `matchExpressions` list, expressions are **ANDed**. Multiple items under `nodeSelectorTerms` are **ORed**. Example of OR:

```yaml
nodeSelectorTerms:
  - matchExpressions:           # term 1
      - {key: disktype, operator: In, values: [ssd]}
  - matchExpressions:           # OR term 2
      - {key: gpu, operator: Exists}
```

#### Preferred (soft) example

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
        - weight: 20
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["zone-a"]
```

`weight` is 1-100. For each node the scheduler sums the weights of the preferences it matches and adds that to the node's score. SSD nodes are strongly preferred, zone-a mildly. Nothing matches? Pod still lands somewhere.

#### Operators

| Operator | Matches when | `values` |
|---|---|---|
| `In` | label value is in the list | required, non-empty |
| `NotIn` | label value not in list (or label missing) | required |
| `Exists` | label key exists, any value | must be empty |
| `DoesNotExist` | label key absent | must be empty |
| `Gt` | label value (integer) greater than | one integer string, e.g. `["8"]` |
| `Lt` | label value (integer) less than | one integer string |

`NotIn` and `DoesNotExist` give you **anti-affinity for nodes** ("anything except spot nodes"). `Gt`/`Lt` need numeric labels, e.g. `cpu-cores=16`.

Both `nodeSelector` and `nodeAffinity` set on one Pod: **both must be satisfied**.

### Dedicated nodes: taint + label + affinity together

Goal: a node pool reserved for GPU/ML jobs; nothing else may use it, and ML jobs must use it.

| Piece | Why |
|---|---|
| Taint `dedicated=ml:NoSchedule` on the nodes | Keeps ordinary Pods **off** |
| Label `dedicated=ml` on the nodes | Gives something to select |
| Toleration for the taint in ML Pods | Lets ML Pods **onto** the nodes |
| `nodeSelector`/`nodeAffinity` for the label in ML Pods | Forces ML Pods **only** onto those nodes |

```bash
kubectl taint nodes worker-2 worker-3 dedicated=ml:NoSchedule
kubectl label nodes worker-2 worker-3 dedicated=ml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trainer
spec:
  replicas: 3
  selector:
    matchLabels: {app: trainer}
  template:
    metadata:
      labels: {app: trainer}
    spec:
      tolerations:
        - key: dedicated
          operator: Equal
          value: ml
          effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: dedicated
                    operator: In
                    values: ["ml"]
      containers:
        - name: trainer
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
          resources:
            requests: {cpu: 100m, memory: 64Mi}
```

Remove either half and you get a classic bug: only taint+toleration → trainers may land on normal nodes; only label+affinity → other teams' Pods may land on the ML nodes.

Cloud node pools usually apply both taint and label for you from one setting (e.g. EKS managed node group taints/labels).

### podAffinity and podAntiAffinity

Node affinity looks at **node labels**. Pod affinity looks at **labels of Pods already running** in a topology domain.

- **podAffinity** — "schedule me in the same domain as Pods matching selector X" (co-location: app + cache, low latency).
- **podAntiAffinity** — "schedule me in a *different* domain from Pods matching selector X" (spread replicas for high availability).

**`topologyKey`** defines what "same domain" means. It is a **node label key**; nodes sharing the same value for that key are one domain:

| `topologyKey` | Domain = |
|---|---|
| `kubernetes.io/hostname` | one node |
| `topology.kubernetes.io/zone` | one availability zone |
| `topology.kubernetes.io/region` | one region |
| any custom label (`rack`, `room`) | whatever the label groups |

#### Anti-affinity: one replica per node (hard)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      containers:
        - name: web
          image: nginx:1.27
```

Each replica refuses a node that already has an `app=web` Pod. With 3 replicas on a 2-node cluster the third stays **Pending**: `1 node(s) didn't match pod anti-affinity rules`. Hard anti-affinity can block rollouts: during a rolling update `maxSurge` creates a new Pod before the old one is gone, and with N replicas on N nodes there is no free node. Use soft anti-affinity or `topologySpreadConstraints` for that case.

#### Anti-affinity: prefer different nodes (soft)

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname
```

Note the nested shape: preferred pod-affinity entries wrap the term in `podAffinityTerm`; required ones are a plain list of terms.

#### Affinity: co-locate with a cache

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: ["redis"]
        topologyKey: kubernetes.io/hostname
```

The Pod lands only on a node already running an `app=redis` Pod. If no such Pod exists yet, it stays Pending — start the dependency first.

#### Namespaces

By default the `labelSelector` matches Pods **in the same namespace as the incoming Pod**. Use `namespaces: [a, b]` or `namespaceSelector: {}` (all namespaces) inside the term to widen.

#### Cost

Pod affinity/anti-affinity is computationally heavy; the Kubernetes docs advise against it on clusters with many hundreds of nodes. `topologySpreadConstraints` is the lighter modern alternative for plain spreading.

### topologySpreadConstraints

Spread matching Pods **evenly** across topology domains, measured by **skew**:

```
skew = (Pods in this domain) - (Pods in the domain with the fewest)
```

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
```

| Field | Meaning |
|---|---|
| `maxSkew` | Largest allowed difference between most- and least-populated domain. `1` = as even as possible |
| `topologyKey` | Node label defining the domains (zone, hostname, ...) |
| `whenUnsatisfiable` | `DoNotSchedule` = hard (Pod Pending if it would break maxSkew); `ScheduleAnyway` = soft (scheduler minimizes skew but still places) |
| `labelSelector` | Which Pods count when measuring (normally the Deployment's own labels) |
| `minDomains` (optional) | Minimum number of domains required before skew is computed against zero |

Worked example: 3 zones, `maxSkew: 1`, 6 replicas → 2/2/2. 7th replica → 3/2/2 (skew 1, fine). Zone-c fails → 3/3/0 isn't forced; the existing Pods stay, new ones follow the rule among the available zones.

Multiple constraints are **ANDed**: spread across zones (`maxSkew: 1`, hard) and across nodes (`maxSkew: 1`, soft) is a very common production pairing:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector: {matchLabels: {app: web}}
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector: {matchLabels: {app: web}}
```

Caveats (concept level): constraints apply only when Pods are **scheduled**; scale-down and node loss can leave uneven spread, and Kubernetes does not rebalance running Pods by itself (an add-on called the descheduler exists for that; not covered here). Nodes missing the `topologyKey` label are ignored for that constraint.

Rule of thumb: **spread replicas → topologySpreadConstraints; co-locate or hard-separate specific workloads → podAffinity/podAntiAffinity.**

### Cordon, drain, uncordon (node maintenance)

```bash
kubectl cordon worker-2          # mark unschedulable (adds node.kubernetes.io/unschedulable:NoSchedule); running Pods stay
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data   # cordon + evict all evictable Pods
kubectl uncordon worker-2        # schedulable again
```

`drain` uses the eviction API: Pods owned by a Deployment/ReplicaSet/StatefulSet are recreated on other nodes; bare Pods are lost (drain refuses unless `--force`). DaemonSet Pods cannot be moved so `--ignore-daemonsets` is required. `--delete-emptydir-data` acknowledges that `emptyDir` content is lost. `--timeout=120s` bounds the wait; `--grace-period` overrides termination time.

Cordon is "stop sending new Pods"; drain is "and move the existing ones". Neither changes labels.

### Reading Pending Pods

```bash
kubectl get pods --field-selector=status.phase=Pending
kubectl describe pod <name>          # Events section at the bottom
kubectl get events --field-selector reason=FailedScheduling --sort-by=.lastTimestamp
```

Decoder for common scheduler messages:

| Message fragment | Meaning | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Requests exceed free allocatable | Lower requests, add nodes, free capacity |
| `node(s) didn't match Pod's node affinity/selector` | nodeSelector or required nodeAffinity matches no node | Fix label/typo; label a node |
| `node(s) had untolerated taint {k: v}` | Taint blocks the Pod | Add toleration or remove taint |
| `node(s) didn't match pod anti-affinity rules` | Required anti-affinity leaves no free domain | Add nodes, reduce replicas, use soft rule/spread |
| `node(s) didn't match pod affinity rules` | Required co-location target not running | Start target first or fix selector |
| `node(s) didn't satisfy existing pods anti-affinity rules` | An *existing* Pod's anti-affinity blocks this one | Review other workloads |
| `node(s) didn't match pod topology spread constraints` | `DoNotSchedule` constraint cannot be met | Add domains, raise maxSkew, switch to `ScheduleAnyway` |
| `node(s) were unschedulable` | Nodes cordoned | `kubectl uncordon` |
| `pod has unbound immediate PersistentVolumeClaims` | Storage issue, not scheduling (lesson 41) | Fix PVC |

The message lists reasons **per node group**: "0/5 nodes are available: 2 untolerated taint, 3 Insufficient cpu" — read all parts.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl get nodes --show-labels` | Show all node labels | `kubectl get nodes --show-labels` |
| `kubectl get nodes -L key1,key2` | Add label values as columns | `kubectl get nodes -L disktype,topology.kubernetes.io/zone` |
| `kubectl get nodes -l key=val` | Filter nodes by label | `kubectl get nodes -l disktype=ssd` |
| `kubectl label node <n> k=v` | Add/change node label | `kubectl label node worker-1 disktype=ssd --overwrite` |
| `kubectl label node <n> k-` | Remove node label | `kubectl label node worker-1 disktype-` |
| `kubectl taint nodes <n> k=v:effect` | Add taint | `kubectl taint nodes worker-2 dedicated=gpu:NoSchedule` |
| `kubectl taint nodes <n> k=v:effect-` | Remove taint | `kubectl taint nodes worker-2 dedicated=gpu:NoSchedule-` |
| `kubectl describe node <n>` | Show taints, capacity, allocated resources, Pods | `kubectl describe node worker-2` |
| `kubectl get nodes -o custom-columns=...` | Tabulate taints | `kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints[*].key` |
| `kubectl get pod -o wide` | See which node each Pod landed on | `kubectl get pods -l app=web -o wide` |
| `kubectl cordon <n>` | Mark node unschedulable | `kubectl cordon worker-2` |
| `kubectl drain <n>` | Cordon + evict Pods | `kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data` |
| `kubectl uncordon <n>` | Mark schedulable again | `kubectl uncordon worker-2` |
| `kubectl get events --field-selector reason=FailedScheduling` | Show scheduling failures | `kubectl get events -A --field-selector reason=FailedScheduling` |
| `kubectl get pods -o custom-columns=POD:.metadata.name,NODE:.spec.nodeName` | Pod-to-node map | `kubectl get pods -A -o custom-columns=POD:.metadata.name,NODE:.spec.nodeName` |

Manifest quick reference:

| Field | Location | Notes |
|---|---|---|
| `nodeName` | `spec` | Bypasses scheduler |
| `nodeSelector` | `spec` | Map, AND, equality |
| `tolerations[]` | `spec` | `key`, `operator`, `value`, `effect`, `tolerationSeconds` |
| `affinity.nodeAffinity.required...` | `spec` | `nodeSelectorTerms[].matchExpressions[]` |
| `affinity.nodeAffinity.preferred...` | `spec` | `weight` + `preference.matchExpressions[]` |
| `affinity.podAffinity` / `podAntiAffinity` | `spec` | `labelSelector` + `topologyKey`; preferred wraps in `podAffinityTerm` |
| `topologySpreadConstraints[]` | `spec` | `maxSkew`, `topologyKey`, `whenUnsatisfiable`, `labelSelector` |

In a Deployment all of these go under `spec.template.spec`, not the top-level `spec`.

## Examples / Walkthrough

Lab assumes a cluster with at least 3 worker nodes named `worker-1..3` (kind: `kind create cluster --config` with 3 `worker` roles; names will be like `kind-worker`, `kind-worker2`, `kind-worker3` — substitute accordingly).

### Example 1: nodeSelector

```bash
kubectl get nodes
kubectl label node worker-1 disktype=ssd
kubectl label node worker-2 disktype=hdd

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: app
      image: nginx:1.27
EOF

kubectl get pod ssd-pod -o wide        # NODE = worker-1
```

Break it on purpose:

```bash
kubectl delete pod ssd-pod
kubectl label node worker-1 disktype-              # remove label
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: ssd-pod}
spec:
  nodeSelector: {disktype: ssd}
  containers: [{name: app, image: nginx:1.27}]
EOF
kubectl get pod ssd-pod                             # Pending
kubectl describe pod ssd-pod | tail -5              # FailedScheduling ... didn't match Pod's node affinity/selector
kubectl label node worker-1 disktype=ssd            # fix: Pod schedules by itself
kubectl get pod ssd-pod -w
```

### Example 2: taint blocks, toleration permits

```bash
kubectl taint nodes worker-2 dedicated=gpu:NoSchedule

# Plain Deployment: 6 replicas never use worker-2
kubectl create deployment plain --image=nginx:1.27 --replicas=6
kubectl get pods -l app=plain -o wide        # none on worker-2

# Toleration version
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: tolerant}
spec:
  replicas: 6
  selector: {matchLabels: {app: tolerant}}
  template:
    metadata: {labels: {app: tolerant}}
    spec:
      tolerations:
        - {key: dedicated, operator: Equal, value: gpu, effect: NoSchedule}
      containers:
        - {name: app, image: nginx:1.27}
EOF
kubectl get pods -l app=tolerant -o wide     # spread over all nodes INCLUDING worker-2, but not only worker-2
```

Point to make aloud: tolerant Pods are *allowed* on worker-2, not *sent* there.

### Example 3: NoExecute eviction

```bash
kubectl taint nodes worker-3 maintenance=true:NoExecute
kubectl get pods -o wide -w          # Pods on worker-3 without toleration are evicted and recreated elsewhere
kubectl taint nodes worker-3 maintenance=true:NoExecute-   # clean up
```

With a grace period:

```yaml
tolerations:
  - key: maintenance
    operator: Equal
    value: "true"
    effect: NoExecute
    tolerationSeconds: 30
```

Pod survives 30 s after the taint is applied, then is evicted.

### Example 4: dedicated node pool (taint + label + affinity)

```bash
kubectl taint nodes worker-2 dedicated=ml:NoSchedule
kubectl label nodes worker-2 dedicated=ml
kubectl apply -f trainer.yaml          # the Deployment from the Concepts section
kubectl get pods -l app=trainer -o wide   # all 3 on worker-2
kubectl get pods -A -o wide | grep worker-2   # only trainers + DaemonSet Pods
```

Why DaemonSet Pods are on worker-2 anyway: DaemonSet controller adds tolerations automatically for `not-ready`, `unreachable`, `unschedulable` etc.; system DaemonSets (CNI, kube-proxy) also carry broad tolerations.

### Example 5: nodeAffinity required vs preferred

```bash
kubectl label node worker-1 zone=a
kubectl label node worker-2 zone=b
kubectl label node worker-3 zone=c cpu-cores=16
kubectl label node worker-1 worker-2 cpu-cores=4 --overwrite

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: affinity-demo}
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - {key: cpu-cores, operator: Gt, values: ["8"]}
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          preference:
            matchExpressions:
              - {key: zone, operator: In, values: ["c"]}
  containers:
    - {name: app, image: nginx:1.27}
EOF
kubectl get pod affinity-demo -o wide      # worker-3 (only node with cpu-cores > 8)
```

Change `values: ["8"]` to `["32"]` and recreate: Pending (required fails). Change required to `Lt 8`: Pod lands on worker-1/2; preferred zone=c no longer matches any eligible node and just doesn't affect the score.

### Example 6: podAntiAffinity — one replica per node

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 4
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector: {matchLabels: {app: web}}
              topologyKey: kubernetes.io/hostname
      containers:
        - {name: web, image: nginx:1.27}
EOF
kubectl get pods -l app=web -o wide        # 3 Running (one per node), 1 Pending
kubectl describe pod -l app=web | grep -A2 "FailedScheduling"
```

Fix options: scale to 3, add a node, or change `required` to `preferred`.

### Example 7: podAffinity — cache next to app

```bash
kubectl run redis --image=redis:7 --labels=app=redis
kubectl get pod redis -o wide
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: app-with-cache}
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: {app: redis}
          topologyKey: kubernetes.io/hostname
  containers:
    - {name: app, image: nginx:1.27}
EOF
kubectl get pods -o wide     # app-with-cache on same node as redis
```

### Example 8: topologySpreadConstraints

```bash
kubectl label node worker-1 worker-2 topology.kubernetes.io/zone=zone-a --overwrite
kubectl label node worker-3 topology.kubernetes.io/zone=zone-b --overwrite

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: spread}
spec:
  replicas: 6
  selector: {matchLabels: {app: spread}}
  template:
    metadata: {labels: {app: spread}}
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector: {matchLabels: {app: spread}}
      containers:
        - {name: app, image: nginx:1.27}
EOF
kubectl get pods -l app=spread -o custom-columns=POD:.metadata.name,NODE:.spec.nodeName
```

Expected: 3 Pods on worker-3 (zone-b) and 3 across worker-1/2 (zone-a): zone skew 0. Scale to 7: one domain gets 4 (skew 1, allowed). Cordon worker-3 and scale to 12 with `DoNotSchedule`: Pods beyond skew 1 go Pending; switch to `ScheduleAnyway` and they schedule.

### Example 9: cordon and drain

```bash
kubectl cordon worker-1
kubectl get nodes                          # worker-1 Ready,SchedulingDisabled
kubectl create deployment after-cordon --image=nginx:1.27 --replicas=6
kubectl get pods -l app=after-cordon -o wide   # none on worker-1
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -A -o wide | grep worker-1    # only DaemonSet Pods remain
kubectl uncordon worker-1
```

Drain a node holding a bare Pod (not controller-owned): drain errors `cannot delete Pods declared without a controller` — add `--force` only if you accept losing it.

### Example 10: reading why a Pod is Pending

```bash
kubectl run huge --image=nginx:1.27 --requests='cpu=64'   # impossible request
kubectl describe pod huge | sed -n '/Events/,$p'
# Warning FailedScheduling  0/3 nodes are available: 3 Insufficient cpu.
kubectl delete pod huge
```

Cleanup for the whole lab:

```bash
kubectl delete deploy plain tolerant trainer web spread after-cordon --ignore-not-found
kubectl delete pod ssd-pod affinity-demo app-with-cache redis --ignore-not-found
kubectl taint nodes --all dedicated- maintenance- 2>/dev/null
kubectl label nodes --all disktype- zone- cpu-cores- dedicated- 2>/dev/null
kubectl uncordon worker-1 worker-2 worker-3
```

## Common Pitfalls

- **Toleration without selector/affinity** — Pod is allowed on the tainted node but is not forced there; it can still land on regular nodes. Add `nodeSelector`/`nodeAffinity` for dedicated pools.
- **Taint without toleration on important Pods** — tainting a node that already hosts DaemonSets (logging, monitoring, CNI) can break them if they lack tolerations. Check `kubectl get ds -A` first.
- **Typo in label key or value** — `disktype: sdd` silently matches nothing; Pod Pending. Compare with `kubectl get nodes --show-labels`.
- **`NoExecute` applied casually** — immediately evicts every Pod without a toleration, including important ones. Prefer `NoSchedule` unless you mean eviction.
- **Forgetting the trailing dash to remove** — `kubectl taint nodes n k=v:NoSchedule` adds again (error "already has taint"); removal needs `...NoSchedule-`. Same for labels (`k-`).
- **Expecting IgnoredDuringExecution to evict** — changing a node label later does not move running Pods. Delete/rollout the Pods to re-evaluate.
- **Hard anti-affinity with replicas > nodes** — extra replicas Pending forever, and rolling updates stall. Use `preferred`, or `topologySpreadConstraints`.
- **Hard podAffinity to a Pod that does not exist yet** — stays Pending. Start dependency first or make the rule soft.
- **Wrong nesting for preferred pod affinity** — preferred needs `weight` + `podAffinityTerm`; required is a plain list. Mixing them gives validation errors.
- **Placing scheduling fields at the wrong level in a Deployment** — they belong under `spec.template.spec`, not `spec`. API either rejects the field or ignores it.
- **`topologyKey` label missing on nodes** — nodes without the label are ignored by spread and by affinity domains; zone spread looks "wrong" when only some nodes have `topology.kubernetes.io/zone`.
- **`labelSelector` in spread constraint matching nothing** — skew computed over zero Pods; no effective spreading. Selector must match the Pods' own labels.
- **Using `nodeName` in production manifests** — bypasses the scheduler and resource checks; Pod is stuck if that node is gone.
- **Draining without `--ignore-daemonsets`** — drain refuses. Without `--delete-emptydir-data` it also refuses when Pods use `emptyDir`.
- **Drain with a PodDisruptionBudget-protected workload** — drain waits/blocks when eviction would violate the budget (PDBs are a concept to meet in production; for the lab check `kubectl get pdb -A` if drain hangs).
- **Control-plane taint left in single-node labs** — Pods Pending with `untolerated taint {node-role.kubernetes.io/control-plane}`; remove the taint on lab clusters only.
- **Assuming scheduling is rebalanced automatically** — Kubernetes only decides at creation time. Uneven clusters after node outages stay uneven until Pods are recreated.

## FAQ

**Q: What is the difference between nodeSelector and nodeAffinity?**
A: `nodeSelector` is exact-match AND on labels, always a hard rule. `nodeAffinity` supports operators (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`), OR between terms, and both hard (`required...`) and soft (`preferred...` with weights) rules. `nodeAffinity` is a superset; `nodeSelector` is simpler to read for simple cases.

**Q: What is the difference between taints and affinity?**
A: Affinity/selector is the **Pod** attracting itself to nodes. A taint is the **node** repelling Pods. They answer different questions and are combined for dedicated nodes: the taint keeps others out, the affinity pulls your Pods in.

**Q: Does a toleration make a Pod go to the tainted node?**
A: No. It only removes the block. The Pod can still be scheduled anywhere else it fits. To force it, add `nodeSelector` or required `nodeAffinity` for a label on those nodes.

**Q: What do the three taint effects do?**
A: `NoSchedule` — new Pods without toleration are not scheduled; running Pods stay. `PreferNoSchedule` — scheduler avoids it if possible, soft. `NoExecute` — no new Pods **and** running Pods without a toleration are evicted (after `tolerationSeconds` if set).

**Q: What does `IgnoredDuringExecution` mean?**
A: The affinity rule is checked only when the Pod is being scheduled. If node labels change afterwards, the running Pod stays where it is.

**Q: How do I make a toleration match any taint?**
A: `tolerations: [{operator: Exists}]` (empty key with `Exists`) tolerates every taint. Used by node-level agents; avoid for normal apps.

**Q: How do I check taints on a node?**
A: `kubectl describe node <n> | grep -A3 Taints` or `kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints[*].key`.

**Q: Why did my Pod on a dead node take 5 minutes to move?**
A: Every Pod automatically gets tolerations for `node.kubernetes.io/not-ready` and `unreachable` with `tolerationSeconds: 300`. Lower `tolerationSeconds` in the Pod spec for faster failover (with the risk of flapping on short network blips).

**Q: Why are control-plane nodes empty of my Pods?**
A: They carry the taint `node-role.kubernetes.io/control-plane:NoSchedule`. Add a toleration (not recommended) or remove the taint on small labs.

**Q: What is a topologyKey?**
A: A node label key that defines a "domain." Nodes with the same value of that label are in the same domain. `kubernetes.io/hostname` = one node per domain; `topology.kubernetes.io/zone` = one zone per domain.

**Q: When should I use podAntiAffinity vs topologySpreadConstraints?**
A: For "spread my replicas evenly" prefer `topologySpreadConstraints`: it controls evenness (`maxSkew`), works with replicas > domains, and is cheaper to compute. Use `podAntiAffinity` for strict "never two of these on one node" rules or "keep app A away from app B."

**Q: What are `maxSkew` and `whenUnsatisfiable`?**
A: `maxSkew` = max allowed difference in matching Pod count between the fullest and emptiest domain. `whenUnsatisfiable: DoNotSchedule` makes it a hard rule (Pod Pending if violated); `ScheduleAnyway` makes it a preference.

**Q: Hard anti-affinity left a replica Pending. What now?**
A: Either you have more replicas than domains. Add nodes, reduce replicas, or use `preferredDuringScheduling...` / a `ScheduleAnyway` spread constraint instead.

**Q: Where do these fields go in a Deployment?**
A: Under `spec.template.spec` (the Pod template), alongside `containers`.

**Q: How do I see why a Pod is Pending?**
A: `kubectl describe pod <name>` → Events → `FailedScheduling` message. It lists each reason and how many nodes hit it.

**Q: What is the difference between cordon and drain?**
A: `cordon` marks the node unschedulable (new Pods avoid it, running Pods stay). `drain` cordons and then evicts the Pods so controllers recreate them elsewhere. `uncordon` reverses cordon.

**Q: Why does `kubectl drain` fail?**
A: Usual reasons: DaemonSet Pods (add `--ignore-daemonsets`), Pods with `emptyDir` (add `--delete-emptydir-data`), bare Pods without controller (use `--force` and accept loss), or a disruption budget blocking eviction.

**Q: Can I combine nodeSelector and nodeAffinity?**
A: Yes. Both must be satisfied. Multiple `matchExpressions` inside one term are ANDed; multiple `nodeSelectorTerms` are ORed.

**Q: Does Kubernetes rebalance Pods when new nodes are added?**
A: No. The scheduler acts only on new/unscheduled Pods. To redistribute, restart the Deployment (`kubectl rollout restart`) or use external tools such as the descheduler.

**Q: How does the scheduler pick between two matching nodes?**
A: It scores them. Preferred affinity weights, spread, resource balance, image locality add to the score; highest wins, ties are random.

**Q: Are labels and taints on nodes preserved when a cloud node is replaced?**
A: Usually not unless set at the node pool / node template level. Define them in the node group configuration (Terraform, managed node group), not by hand with kubectl, for anything long-lived.

## Practice / Exercise

Prepare: a cluster with ≥ 3 workers (kind/minikube/rke2). Clean up between exercises.

### Core

1. **Labels.** List node labels. Add `disktype=ssd` to one node and `disktype=hdd` to another. Show nodes with `-L disktype`.
2. **nodeSelector.** Create a Pod with `nodeSelector: disktype=ssd`. Confirm placement with `-o wide`. Remove the label from the node, recreate the Pod, read the `FailedScheduling` message, restore the label and watch the Pod schedule.
3. **Taint and toleration.** Taint one worker `team=blue:NoSchedule`. Create a Deployment with 6 replicas **without** toleration; confirm none land on the tainted node. Add the toleration; observe that some (not all) land there. Explain why not all.
4. **NoExecute.** Apply `NoExecute` taint to a node hosting Pods of a Deployment; watch eviction and recreation elsewhere. Remove the taint.
5. **nodeAffinity.** Write a Pod requiring `disktype In [ssd, nvme]` and preferring `zone In [a]` with weight 60. Verify placement. Change the required values to a nonexistent one and read the event.
6. **podAntiAffinity.** Create a 3-replica Deployment with hard anti-affinity on `kubernetes.io/hostname`. Scale to 5 and explain the Pending Pods. Switch to the preferred form and re-scale.
7. **Spread constraint.** Label nodes into two zones. Deploy 6 replicas with `maxSkew: 1` over `topology.kubernetes.io/zone`. Print Pod → node and compute zone skew.
8. **Drain.** Cordon a node, deploy something new, confirm nothing lands there. Drain it, then uncordon it.

### Stretch

1. **Dedicated pool.** Build the full pattern (taint + label + toleration + required nodeAffinity) for a `batch` workload. Prove (a) a normal Deployment never lands on the pool and (b) the batch Deployment lands only on the pool. Then remove only the toleration from batch and show what happens (Pending).
2. **Eviction timing.** Create two Deployments, one with default tolerations and one with `tolerationSeconds: 10` for `node.kubernetes.io/unreachable`/`not-ready`. Discuss (or on a throwaway VM node, stop kubelet) how long each takes to be evicted.
3. **Two constraints.** Create a Deployment with a hard zone spread (`maxSkew: 1`) plus a soft hostname spread. Scale 2 → 3 → 9 and record Pod → node → zone each time.
4. **Rollout deadlock.** Reproduce a stuck rolling update: Deployment with replicas = number of nodes, hard hostname anti-affinity, `maxSurge: 1, maxUnavailable: 0`. Change the image and watch the new Pod Pending. Fix it two ways (`maxUnavailable: 1`, then soft anti-affinity/spread) and compare.
5. **Gt/Lt.** Label nodes with `cpu-cores` values and write required affinity using `Gt`. Add a second term (OR) using `DoesNotExist`. Predict placement before applying.
6. **Scheduler detective.** Create three broken Pods (typo label, untolerated taint, impossible CPU request). For each, state the root cause from `kubectl describe` only, then fix it.

## Further Reading

- Kubernetes docs — Assigning Pods to Nodes: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes docs — Taints and Tolerations: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
- Kubernetes docs — Pod Topology Spread Constraints: https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/
- Kubernetes docs — Node-pressure Eviction: https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/
- Kubernetes docs — Safely Drain a Node: https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/
- Kubernetes docs — Well-Known Labels, Annotations and Taints: https://kubernetes.io/docs/reference/labels-annotations-taints/
- Kubernetes docs — Kubernetes Scheduler: https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/

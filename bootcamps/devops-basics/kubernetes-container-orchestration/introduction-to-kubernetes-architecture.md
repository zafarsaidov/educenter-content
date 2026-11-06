# Lesson 34: Introduction to Kubernetes & Architecture

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 30-33 (containers, images, Dockerfile, volumes, networks, Compose, registries); Lesson 10 (TCP/IP, ports); Lesson 8 (processes); Lesson 7 (systemd services); Lesson 24 (YAML syntax)

## Learning Objectives

By end of lesson student can:
- Explain which problems appear when containers run on many hosts and why an orchestrator solves them
- Name Kubernetes core capabilities: scheduling, self-healing, scaling, service discovery, rollouts, config/secret handling
- Draw the cluster architecture: control plane (kube-apiserver, etcd, kube-scheduler, kube-controller-manager, optional cloud-controller-manager) and worker nodes (kubelet, kube-proxy, container runtime)
- Trace what happens, step by step, between "a user asks for something" and "a container is running"
- Describe the declarative model: desired state vs actual state, reconciliation loop, level-triggered controllers
- Describe the anatomy of any Kubernetes object: `apiVersion`, `kind`, `metadata`, `spec`, `status`
- Read a kubeconfig file (clusters, users, contexts) and switch contexts
- Create a local lab cluster (kind or minikube) and use `kubectl cluster-info`, `kubectl version`, `kubectl get nodes`, `kubectl get pods -n kube-system` to inspect it
- Explain the difference between Kubernetes, Docker, Docker Compose and managed Kubernetes offerings

## Topics

- Why Kubernetes: orchestration challenges, self-healing, autoscaling, service discovery
- Control plane: kube-apiserver, etcd, kube-scheduler, kube-controller-manager
- Worker node components: kubelet, kube-proxy, container runtime (containerd/CRI-O)
- Kubernetes object model: declarative API, desired vs actual state, reconciliation loop
- Cluster access: kubeconfig file, contexts, `kubectl cluster-info`, `kubectl version`

## Concepts

### From one host to many hosts: the orchestration problem

In Lessons 30-33 we ran containers on a single machine with `docker run` and Docker Compose. That works for a laptop or one small server. Production reality is different:

- One server is a **single point of failure**. If the host dies, every container dies with it.
- One server has a **capacity ceiling**. Traffic grows, you need ten copies of the web container across several machines.
- Somebody must decide **which machine** runs which container (placement), based on free CPU/RAM, labels, constraints.
- Containers crash. Somebody must **notice and restart** them — or recreate them on another machine if the machine is gone.
- Containers get new IP addresses every time they are recreated. Other containers need a **stable way to find them** (service discovery) and something to **spread traffic** across copies (load balancing).
- New versions must be **rolled out without downtime**, and rolled back if broken.
- Configuration and passwords must reach containers **without baking them into images**.
- Storage must follow the workload when it moves.

Doing all of this with shell scripts and `docker run` over SSH does not scale. A **container orchestrator** is software that takes a *description of what you want* ("run 5 copies of this image, reachable on this name") and continuously makes the cluster match it. Kubernetes (written **K8s**: K + 8 letters + s) is the dominant orchestrator.

### What Kubernetes is (and is not)

**Kubernetes** is an open-source platform for automating deployment, scaling and management of containerized applications across a cluster of machines. Originally designed at Google (descendant of their internal systems Borg and Omega), open-sourced in 2014, donated to the CNCF (Cloud Native Computing Foundation) in 2015. Written in Go. The name is Greek for "helmsman / pilot" — hence the ship's wheel logo.

What it **is**:
- A cluster manager and scheduler for containers
- A declarative API with a reconciliation engine behind it
- A platform to build platforms on (extensible via CRDs and operators — we meet those in later lessons)

What it is **not**:
- Not a container runtime — it *uses* one (containerd, CRI-O). Docker the tool is not needed on the nodes anymore.
- Not a PaaS like Heroku — no built-in CI/CD, no application-level logging/monitoring, no built-in database
- Not a build system — images are built elsewhere (Lesson 31) and pulled from a registry (Lesson 33)
- Not necessary for everything — one VM with Compose is perfectly fine for a small app

### Core capabilities

| Capability | What it means | Example |
|---|---|---|
| **Scheduling** | Picks a node for each container group based on resources and rules | "Run on a node with 500m CPU and 256Mi RAM free" |
| **Self-healing** | Restarts failed containers, replaces them, reschedules when a node dies, withholds traffic from unhealthy ones | Process crashes at 3 AM → restarted within seconds, nobody paged |
| **Horizontal scaling** | Change the replica count manually or automatically by CPU/custom metrics | 3 → 30 copies during a sale |
| **Service discovery & load balancing** | Stable name/IP in front of a changing set of containers | `http://backend` always works, whichever copies exist |
| **Automated rollouts & rollbacks** | Gradual version replacement; undo on failure | Rolling update with zero downtime |
| **Configuration & secret management** | Inject config/credentials at runtime without rebuilding images | Same image in dev and prod, different settings |
| **Storage orchestration** | Attach local, network or cloud volumes automatically | Database disk follows the pod |
| **Batch execution** | Run one-off and scheduled jobs to completion | Nightly backup |
| **Extensibility** | Add new resource types and controllers | Operators for databases, cert management |

Do not memorise manifests for these yet — each capability gets its own lesson (Deployments in lesson 36, Services in 37, Ingress in 38, storage in 41, config and secrets in 42, probes and autoscaling concepts in 45, and so on). Today is the map.

### Cluster architecture overview

A **cluster** = one **control plane** (the brain) + one or more **worker nodes** (the muscle).

```
                        ┌──────────────────────── CONTROL PLANE ───────────────────────┐
  kubectl / CI / UI ──► │  kube-apiserver  ◄──────►  etcd (cluster database)           │
  (HTTPS, port 6443)    │       ▲   ▲                                                  │
                        │       │   └────────────┐                                     │
                        │  kube-scheduler   kube-controller-manager   (cloud-ctrl-mgr) │
                        └───────┬──────────────────────────────────────────────────────┘
                                │  all components talk ONLY to the API server
          ┌─────────────────────┼────────────────────────┐
          ▼                     ▼                        ▼
   ┌─────────────┐      ┌─────────────┐          ┌─────────────┐
   │ WORKER NODE │      │ WORKER NODE │          │ WORKER NODE │
   │  kubelet    │      │  kubelet    │          │  kubelet    │
   │  kube-proxy │      │  kube-proxy │          │  kube-proxy │
   │  containerd │      │  containerd │          │  containerd │
   │  [pod][pod] │      │  [pod][pod] │          │  [pod]      │
   └─────────────┘      └─────────────┘          └─────────────┘
```

Key rule: **the API server is the only component that reads/writes etcd, and every other component talks to the API server.** Scheduler does not call kubelet directly; controllers do not call the scheduler. Everything is mediated by the API and stored objects. This is why the system is loosely coupled and resilient.

### Control plane components

The control plane makes global decisions (scheduling) and reacts to events (a node died, replicas are missing). It usually runs on dedicated machines ("control-plane nodes", formerly "master") but small clusters may run workloads there too.

#### kube-apiserver

The **front door** of the cluster. A stateless REST/HTTPS server (default secure port **6443**).

- Exposes the Kubernetes API: every `kubectl` command, every controller, every kubelet talks to it
- **Authenticates** (who are you? certificate, token, OIDC), **authorizes** (are you allowed? RBAC — lesson 47) and runs **admission control** (should the request be changed or rejected? — lesson 48) for every request
- **Validates** objects and persists them to etcd
- Provides **watch** streams: components subscribe to changes ("tell me when a pod appears") instead of polling
- Horizontally scalable: several API servers behind a load balancer for high availability

Request pipeline: `Client → Authentication → Authorization → Mutating admission → Schema validation → Validating admission → etcd write → response`.

#### etcd

A distributed, consistent **key-value store** (uses the Raft consensus algorithm). It is the **single source of truth**: every object in the cluster (all pods, nodes, services, secrets, configs) lives here.

- Only the API server talks to it
- Losing etcd without a backup = losing the cluster's state (workloads keep running on nodes for a while, but you can no longer manage them)
- Run 3 or 5 members for HA (odd number, **quorum** = majority; 3 members tolerate 1 failure, 5 tolerate 2)
- Needs fast disks (SSD) — latency of disk writes directly affects cluster responsiveness
- Contains secrets: Kubernetes Secrets are only base64-encoded by default, **not encrypted**, unless encryption at rest is configured (lessons 42-43 return to this)
- Backed up with `etcdctl snapshot save` (operations topic, not today)

#### kube-scheduler

Watches for newly created pods that have **no node assigned** and picks a node for each. Two phases:

1. **Filtering** — remove nodes that cannot run the pod (not enough CPU/RAM, taint not tolerated, port conflict, node selector mismatch)
2. **Scoring** — rank remaining nodes (spread pods, prefer nodes that already have the image, balance resource use) and pick the best

The scheduler only **writes the decision** (sets `spec.nodeName` on the pod through the API). It does not start anything. Scheduling rules (requests, taints, affinity) are lessons 45-46.

#### kube-controller-manager

One binary that runs many **controllers** — each a control loop watching one kind of object and driving reality toward the desired state:

| Controller | Job |
|---|---|
| Node controller | Notices nodes that stop reporting, marks them NotReady, evicts pods after a timeout |
| ReplicaSet / Deployment controller | Keeps the desired number of pod copies; manages rollouts |
| Job / CronJob controller | Runs pods to completion, on a schedule |
| EndpointSlice controller | Maintains the list of pod addresses behind a Service |
| ServiceAccount / token controllers | Creates default accounts and tokens for namespaces |
| Namespace, PV, StatefulSet, DaemonSet… controllers | Each handles its own resource type |

#### cloud-controller-manager (optional)

Present only on cloud-hosted or cloud-integrated clusters. Holds the cloud-provider-specific loops: create a cloud load balancer when asked, attach cloud disks, detect that a VM was deleted in the cloud console. Keeps cloud code out of the Kubernetes core. A kubeadm/bare-metal lab cluster does not have it.

### Worker node components

A **node** is a machine (VM or physical) that runs your workloads. Every node runs three things.

#### kubelet

The **node agent**. A process (a systemd service, not a container) on every node.

- Registers the node with the API server and sends heartbeats and status (capacity, conditions)
- Watches the API for pods assigned to *its* node
- Tells the container runtime to pull images and start/stop containers (via **CRI** — Container Runtime Interface)
- Runs the health checks (probes) and reports pod status back — lesson 45
- Mounts volumes, injects config and secrets into containers
- Can also run **static pods** from manifest files in a local directory (that is how control plane components run on kubeadm clusters — lesson 52)

If kubelet stops, the node goes **NotReady** after about 40 seconds (node-monitor-grace-period), and its pods are eventually evicted and recreated elsewhere.

#### kube-proxy

A network component on every node that implements the **Service** abstraction (lesson 37). It watches Services and EndpointSlices and programs the node's packet rules (iptables by default, or IPVS / nftables modes) so traffic to a Service virtual IP is spread across the pods behind it. Some modern CNIs (Cilium) replace kube-proxy entirely with eBPF.

#### Container runtime

The software that actually runs containers. Kubernetes talks to it over the **CRI** gRPC interface. Docker Engine is **not** a CRI runtime; since Kubernetes 1.24 the built-in "dockershim" bridge is gone.

| Runtime | Notes |
|---|---|
| **containerd** | Default in most distributions (kubeadm, kind, rke2, EKS, GKE, AKS). It is the same `containerd` that sits inside Docker (lesson 30). |
| **CRI-O** | Lightweight runtime built only for Kubernetes, default in OpenShift |

Both use a low-level OCI runtime (`runc` by default) to create the Linux namespaces/cgroups. Your Docker-built images still run unchanged — images are OCI-standard. Debug tool on nodes: `crictl` (like `docker` CLI for CRI runtimes) — mentioned for awareness.

#### Add-ons (not strictly components, but every real cluster has them)

- **CNI plugin** (Container Network Interface): gives pods IP addresses and cross-node connectivity. Examples: Flannel, Calico, Cilium. A node without a CNI stays `NotReady` (we install one in lesson 52).
- **CoreDNS**: in-cluster DNS for service names (lesson 38)
- **metrics-server**: resource usage data for `kubectl top` and autoscaling
- **Ingress/Gateway controller**, dashboards, monitoring agents

### Pod: the smallest unit (preview)

Kubernetes does not schedule bare containers; it schedules **Pods**. A Pod is one or more tightly coupled containers sharing a network namespace (one IP, localhost between them) and optionally volumes. Usually one container per pod. Pods are **ephemeral** — when one dies it is replaced by a *new* pod with a *new* IP, never "repaired". Writing pod and Deployment manifests is lesson 36; today we only need the word to read cluster output.

### Walkthrough of the architecture: from request to running container

Follow one request: "I want 3 copies of image X" (the mechanism of `Deployment`, lesson 36):

1. User runs a `kubectl` command → HTTPS request to **kube-apiserver**.
2. API server authenticates, authorizes, runs admission, validates, and **stores the object in etcd**. Returns "created". Nothing is running yet.
3. The **Deployment controller** (inside controller-manager) sees the new Deployment via a watch and creates a **ReplicaSet** object.
4. The **ReplicaSet controller** sees it and creates 3 **Pod** objects (still with no node).
5. The **scheduler** sees 3 pods with empty `nodeName`, filters/scores nodes and writes `nodeName` for each.
6. The **kubelet** on each chosen node sees a pod assigned to it, asks the **runtime** to pull the image and start containers, mounts volumes.
7. kubelet reports pod `status` (Running, IP, conditions) back to the API server → stored in etcd.
8. The **EndpointSlice controller** records the pod IPs for any matching Service; **kube-proxy** on every node updates rules.
9. A pod crashes later: kubelet restarts the container. A node dies: node controller marks it NotReady, pods are evicted, the ReplicaSet controller notices fewer than 3 and creates replacements; the scheduler places them elsewhere.

Teaching point: **no component gives orders to another.** Each watches the API for work in its own area and writes results back. That is the "choreography" model.

### The declarative model: desired state vs actual state

**Imperative**: tell the system *how* — "start container, then start another, then attach".
**Declarative**: tell the system *what* — "there should be 3 copies of this" — and let it work out the steps.

Kubernetes is declarative:

- **Desired state** — what you submitted, stored in `spec`.
- **Actual (observed) state** — what exists right now, reported in `status`.
- **Controllers** compare the two and act to close the gap. The endless loop is the **reconciliation loop**:

```
        ┌────────────────────────────────────────────┐
        ▼                                            │
   observe actual state ──► compare with desired ──► act (create / delete / update) ─┘
```

Properties worth stating:
- **Level-triggered, not edge-triggered.** A controller does not rely on having seen "event X happened"; it repeatedly looks at the *current level* of the world and fixes any difference. Missed events self-heal.
- **Idempotent.** Applying the same desired state twice changes nothing the second time. That is why you can re-run `kubectl apply` safely (lesson 35).
- **Self-healing falls out for free.** Delete a pod belonging to a 3-replica workload and the loop sees 2 ≠ 3 and creates one.
- **Git-friendly.** Desired state is text files — they can be reviewed, versioned and reapplied (GitOps; Kustomize/Helm lessons 53-58).

Compare with what students know: Ansible (lessons 24-29) is also declarative and idempotent, but runs *once when invoked*. A Kubernetes controller runs *forever*.

### Anatomy of a Kubernetes object

Everything in the cluster is an **object** (also "resource" — a persistent record of intent). Each kind of object is served by the API at a REST path and described in YAML or JSON. Every object has the same top-level shape:

| Field | Meaning |
|---|---|
| `apiVersion` | API group and version that defines this kind, e.g. `v1` (core group) or `apps/v1` |
| `kind` | The type: Pod, Deployment, Service, Node, Namespace… |
| `metadata` | Identity: `name`, `namespace`, `labels`, `annotations`, `uid`, `creationTimestamp`, `ownerReferences`, `resourceVersion` |
| `spec` | **Desired state** — written by you (or a controller on your behalf) |
| `status` | **Actual state** — written by the system; you never edit it |

Concepts to teach about metadata:
- **name** — unique per kind (and per namespace for namespaced kinds). Names are DNS-compatible: lowercase letters, digits, `-`, `.`.
- **namespace** — virtual partition of the cluster for isolation and quotas. Initial namespaces: `default`, `kube-system` (cluster components), `kube-public`, `kube-node-lease`. Details in lesson 35.
- **labels** — key/value tags used by selectors to find groups of objects (the glue of Kubernetes). Lesson 35.
- **uid** — unique per object lifetime. Delete and recreate → new uid.
- **ownerReferences** — "this pod belongs to that ReplicaSet"; deleting the owner deletes dependents (garbage collection).

Some objects are **cluster-scoped** (Node, Namespace, PersistentVolume, ClusterRole) and some are **namespaced** (Pod, Service, ConfigMap…).

Authors of manifests do not write `status`; try as you might, the API server overwrites it.

### API groups and versions

The API is organised into **groups**: the core group (`v1`, legacy path `/api/v1`: Pod, Service, Node, Namespace, ConfigMap, Secret) and named groups (`/apis/<group>/<version>`: `apps/v1` Deployment, `batch/v1` Job, `networking.k8s.io/v1` Ingress, `rbac.authorization.k8s.io/v1`…). Versions mature through **alpha** (`v1alpha1`, off by default, can vanish) → **beta** (`v1beta1`, may be on by default) → **stable** (`v1`, kept for a long time). Deprecated versions are removed after a deprecation window — a common reason old manifests fail after a cluster upgrade. Discovering them (`kubectl api-resources`, `explain`) is lesson 35.

### Kubernetes release cycle

New minor version (1.x) roughly **three per year**. Each minor is supported ~**14 months** with patch releases. Clusters need regular upgrades; skipping minors during upgrade is not supported (go 1.29 → 1.30 → 1.31). `kubectl` is supported within **±1 minor version** of the API server — mismatch beyond that causes odd errors. Check both with `kubectl version`.

### Cluster access: kubeconfig

`kubectl` is only a client. How does it know which cluster to talk to and who you are? Through a **kubeconfig** file.

- Default location: `~/.kube/config`
- Override: `KUBECONFIG` environment variable (can hold several files separated by `:`) or `--kubeconfig <file>` per command
- It is **a credential file** — contains client certs/keys or tokens. Protect it: `chmod 600`, never commit to Git, never paste in chat/tickets.

A kubeconfig has three lists and one pointer:

```yaml
apiVersion: v1
kind: Config
clusters:                      # WHERE: API server address + CA to trust
- name: kind-lab
  cluster:
    server: https://127.0.0.1:6443
    certificate-authority-data: LS0tLS1CRUdJTi...   # base64 CA cert
users:                         # WHO: credentials
- name: kind-lab-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTi...
    client-key-data: LS0tLS1CRUdJTi...
contexts:                      # BINDING: cluster + user (+ default namespace)
- name: kind-lab
  context:
    cluster: kind-lab
    user: kind-lab-admin
    namespace: default         # optional
current-context: kind-lab      # which context kubectl uses right now
```

- **cluster** = server URL + CA bundle (so kubectl trusts the API server's TLS certificate)
- **user** = how to authenticate: client certificate+key, bearer token, or an `exec` plugin (cloud CLIs like `aws eks get-token`, OIDC login helpers)
- **context** = named triple (cluster, user, namespace). One file can describe dozens of clusters (dev, staging, prod, several clouds).
- **current-context** = the active one. Every command goes there unless `--context` is passed.

Dangerous habit: working on prod while thinking you are on dev. **Always check the context before destructive commands**; many teams show it in the shell prompt (kube-ps1) or use tools such as `kubectx`/`kubens`.

Context commands (all safe, read-mostly):

```bash
kubectl config get-contexts            # list all; * marks current
kubectl config current-context         # print the active one
kubectl config use-context kind-lab    # switch
kubectl config view                    # show merged config (secrets redacted)
kubectl config view --minify           # only the active context's slice
kubectl config view --raw              # include real cert data (careful!)
```

How it connects: kubectl reads kubeconfig → opens HTTPS to `server` → verifies the server cert against the CA → presents client cert/token → API server authenticates and authorizes.

Authentication details (who you can *be*) and permissions (what you may *do*) are lesson 47. Here: just know that a kubeconfig is how identity reaches the API.

### Where do clusters come from? Options overview

| Option | What | Typical use |
|---|---|---|
| **kind** (Kubernetes IN Docker) | Nodes are Docker containers on your laptop; multi-node possible | Learning, CI tests; used in this lesson |
| **minikube** | Single-node cluster in a VM/container; many add-ons | Learning on laptop |
| **k3s / k3d** | Lightweight single-binary distribution | Edge, home labs |
| **kubeadm** | Official tool to bootstrap a cluster on your own machines | Self-managed clusters (lesson 52) |
| **rke2** | Hardened distribution from SUSE/Rancher | Self-managed, compliance (lesson 52) |
| **Managed services**: EKS (AWS), GKE (Google), AKS (Azure), DOKS… | Provider runs the control plane for you; you manage worker nodes (or not even those) | Most production workloads |

In managed services you **cannot SSH to or see the control plane**; you only get the API endpoint and a kubeconfig. That is a major operational difference.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl version` | Show client and server versions | `kubectl version` |
| `kubectl version --client` | Client version only (no cluster needed) | `kubectl version --client` |
| `kubectl cluster-info` | Show API server and CoreDNS addresses | `kubectl cluster-info` |
| `kubectl cluster-info dump` | Huge diagnostic dump of cluster state | `kubectl cluster-info dump --output-directory=./dump` |
| `kubectl get nodes` | List nodes and their status | `kubectl get nodes` |
| `kubectl get nodes -o wide` | Add internal IP, OS image, kernel, runtime | `kubectl get nodes -o wide` |
| `kubectl describe node <name>` | Capacity, conditions, taints, pods on the node | `kubectl describe node kind-lab-control-plane` |
| `kubectl get pods -n kube-system` | List system component pods | `kubectl get pods -n kube-system` |
| `kubectl get pods -A` | Pods in all namespaces | `kubectl get pods -A` |
| `kubectl get --raw='/readyz?verbose'` | Query API server readiness checks | `kubectl get --raw='/readyz?verbose'` |
| `kubectl config get-contexts` | List contexts | `kubectl config get-contexts` |
| `kubectl config current-context` | Show active context | `kubectl config current-context` |
| `kubectl config use-context <ctx>` | Switch active context | `kubectl config use-context kind-lab` |
| `kubectl config view --minify` | Show active context's config | `kubectl config view --minify` |
| `kubectl --context <ctx> ...` | One-off command against another context | `kubectl --context kind-lab get nodes` |
| `kubectl --kubeconfig <file> ...` | Use a different kubeconfig file | `kubectl --kubeconfig ./other.conf get nodes` |
| `export KUBECONFIG=a:b` | Merge several kubeconfig files | `export KUBECONFIG=~/.kube/config:~/dev.conf` |
| `kind create cluster --name lab` | Create local cluster | `kind create cluster --name lab` |
| `kind get clusters` | List kind clusters | `kind get clusters` |
| `kind delete cluster --name lab` | Remove cluster | `kind delete cluster --name lab` |
| `minikube start` | Start local cluster | `minikube start --driver=docker` |
| `minikube status` / `stop` / `delete` | Manage minikube | `minikube delete` |
| `crictl ps` | List containers via CRI on a node | `sudo crictl ps` |

## Examples / Walkthrough

### 1. Install the tools (Ubuntu / Linux amd64)

```bash
# Docker must already work (Lesson 30): docker version

# kubectl — download the stable binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# kind — a Kubernetes cluster whose "nodes" are Docker containers
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
sudo install -m 0755 kind /usr/local/bin/kind
kind version

# shell completion + short alias (saves lots of typing)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

macOS: `brew install kubectl kind`. Alternative lab cluster: `minikube start --driver=docker`. Everything in this lesson works the same on both.

### 2. Create a multi-node lab cluster with kind

Create `kind-config.yaml` (this is kind's own cluster-definition file, not a Kubernetes workload manifest):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

```bash
kind create cluster --name lab --config kind-config.yaml
# Output ends with: Set kubectl context to "kind-lab"

docker ps                       # 3 containers: lab-control-plane, lab-worker, lab-worker
```

Teaching moment: the "nodes" are Docker containers that each run systemd, kubelet and containerd inside. Containers inside a container — that is why `kind` is great for labs and CI but not for production.

### 3. Inspect cluster access

```bash
kubectl config get-contexts
# CURRENT   NAME      CLUSTER   AUTHINFO   NAMESPACE
# *         kind-lab  kind-lab  kind-lab

kubectl config current-context        # kind-lab
kubectl config view --minify          # see the server URL, cert data shown as DATA+OMITTED / REDACTED
cat ~/.kube/config | head -30         # same data, raw: note clusters / users / contexts / current-context
ls -l ~/.kube/config                  # should be -rw------- (600)
```

Ask students: which port is the API server on? (the `server:` URL). Which three things does the context bind?

### 4. Check versions and API server health

```bash
kubectl version
# Client Version: v1.31.x
# Kustomize Version: v5.x.x         (kustomize is built into kubectl — lessons 56-58)
# Server Version: v1.31.x

kubectl cluster-info
# Kubernetes control plane is running at https://127.0.0.1:XXXXX
# CoreDNS is running at https://127.0.0.1:XXXXX/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

kubectl get --raw='/readyz?verbose' | head -20     # many [+] ok lines = API server and etcd healthy
kubectl get --raw='/livez'                         # ok
```

Explain: client version vs server version — keep within ±1 minor. `[+]etcd ok` proves the API server can reach etcd.

### 5. Look at the nodes

```bash
kubectl get nodes
# NAME                STATUS   ROLES           AGE   VERSION
# lab-control-plane   Ready    control-plane   2m    v1.31.x
# lab-worker          Ready    <none>          2m    v1.31.x
# lab-worker2         Ready    <none>          2m    v1.31.x

kubectl get nodes -o wide
# adds INTERNAL-IP, OS-IMAGE, KERNEL-VERSION, CONTAINER-RUNTIME  -> containerd://1.7.x

kubectl describe node lab-worker | less
```

In `describe node` point to: **Conditions** (Ready, MemoryPressure, DiskPressure, PIDPressure), **Capacity / Allocatable** (CPU, memory, max pods — allocatable is capacity minus system reserved), **System Info** (kubelet version, container runtime, OS), **Taints** (note the control-plane node has one so ordinary workloads avoid it — lesson 46), **Non-terminated Pods** (what runs here), **Allocated resources** (requests vs limits — lesson 45), **Events**.

`ROLES <none>` for workers is only the absence of a `node-role.kubernetes.io/<role>` label — workers are not "broken".

### 6. Meet the control plane components as pods

```bash
kubectl get pods -n kube-system -o wide
# coredns-xxxx                       1/1  Running  (x2)
# etcd-lab-control-plane             1/1  Running
# kindnet-xxxx                       1/1  Running  (one per node)      <- CNI plugin used by kind
# kube-apiserver-lab-control-plane   1/1  Running
# kube-controller-manager-lab-...    1/1  Running
# kube-proxy-xxxx                    1/1  Running  (one per node)
# kube-scheduler-lab-...             1/1  Running
```

Map each to the architecture diagram. Observations:
- apiserver, etcd, scheduler, controller-manager exist **only on the control-plane node** — on kubeadm-style clusters they are **static pods** managed by kubelet from `/etc/kubernetes/manifests` (lesson 52).
- kube-proxy and the CNI (`kindnet`) have **one pod per node** (a pattern called DaemonSet — not covered until later; for now: "one per node").
- kubelet and containerd are **not** in the list: they are services on the node itself, not pods.
- CoreDNS has 2 copies for availability.

```bash
kubectl get pods -A          # same list plus any other namespaces
kubectl get namespaces       # default, kube-system, kube-public, kube-node-lease, local-path-storage
```

### 7. Look inside a node (kind: nodes are containers)

```bash
docker exec -it lab-control-plane bash

# inside the node:
ls /etc/kubernetes/manifests            # etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml
systemctl status kubelet --no-pager     # kubelet is a systemd service
crictl ps                               # containers as seen by containerd (not Docker!)
crictl images | head
ps aux | grep -E 'kube-apiserver|etcd' | head -3
exit
```

This makes the architecture concrete: control plane = ordinary processes; kubelet watches a directory and runs whatever manifests it finds. (Do not edit those files in the lab unless you want to break the cluster — that is the point of the stretch exercise in lesson 52, not today.)

### 8. Watch the reconciliation loop (observe only)

Today we only *observe* the self-healing of system pods; creating our own workloads comes in lesson 36.

```bash
# terminal 1: watch pods
kubectl get pods -n kube-system -w

# terminal 2: stop a CoreDNS container from the node's runtime view and see the loop restart it.
# CoreDNS pods are owned by a controller; deleting one makes the controller recreate it:
kubectl delete pod -n kube-system -l k8s-app=kube-dns --wait=false
```

In terminal 1: pods go `Terminating`, new ones appear `ContainerCreating` → `Running` with new names and new IPs. Desired = 2 replicas, actual dropped to 0, controller closed the gap. (This is the *only* delete we do on system pods — safe in a lab, never do it casually on production.)

### 9. Switch contexts (simulate two clusters)

```bash
kind create cluster --name second          # context kind-second is created and becomes current
kubectl config get-contexts                # two contexts now; * on kind-second
kubectl get nodes                          # shows the NEW cluster's single node
kubectl config use-context kind-lab
kubectl get nodes                          # back to 3 nodes
kubectl --context kind-second get nodes    # one-off without switching

kind delete cluster --name second
kubectl config get-contexts                # kind-second entry removed automatically
```

Show how easily one wrong `use-context` changes the target. Habit: `kubectl config current-context` before any destructive command.

### 10. Cleanup

```bash
kind delete cluster --name lab
docker ps -a                # kind node containers gone
```

## Common Pitfalls

- **"Kubernetes replaces Docker, so my Dockerfile is useless"** — wrong. K8s replaced Docker *as the node runtime* (containerd instead of dockershim); images built from your Dockerfile run unchanged because they are OCI images.
- **Running `kubectl` against the wrong cluster** — the active context decides. Check `kubectl config current-context` before `delete`/`apply`; use separate kubeconfig files or a prompt indicator for prod.
- **`The connection to the server localhost:8080 was refused`** — kubectl found no kubeconfig and fell back to the default. Fix: `export KUBECONFIG=...` or copy the file to `~/.kube/config`.
- **`Unable to connect to the server: x509: certificate signed by unknown authority`** — kubeconfig CA does not match the API server (wrong file, rebuilt cluster with old kubeconfig). Fetch a fresh kubeconfig.
- **`Unable to connect to the server: dial tcp ...:6443: i/o timeout`** — API server down, firewall blocking 6443, VPN not connected, or wrong server address.
- **Client/server version skew** — `kubectl` more than one minor version away from the API server may misbehave. Run `kubectl version` and install a matching client.
- **Committing kubeconfig to Git or posting it in chat** — it contains cluster-admin credentials in many setups. Treat like a private key. Rotate if leaked.
- **Editing `status` by hand / expecting `kubectl edit` on status to stick** — `status` is owned by controllers and kubelet; only `spec` is yours.
- **Expecting a node role of `worker`** — `ROLES <none>` is normal for workers; roles are just optional labels.
- **Thinking containers inside a pod can each have their own IP** — one pod = one IP, shared by its containers.
- **Using kind for production** — kind nodes are containers on one machine; there is no real fault isolation. Use it for learning and CI.
- **Treating the control plane as workload nodes** — a single-node cluster (minikube) runs everything together; production separates them and taints control-plane nodes so apps do not land there.
- **Deleting pods in `kube-system` "to see what happens" on shared clusters** — can take down DNS or networking cluster-wide.
- **Assuming a healthy `kubectl get nodes` means the apps are healthy** — nodes Ready only proves kubelet is reporting. Pod and Service health is checked separately.
- **Assuming etcd backups are automatic** — they are not (outside managed services). No backup, no recovery.

## FAQ

**Q: What is Kubernetes in one sentence?**
A: An open-source system that runs and manages containers across many machines: it decides where they run, restarts them when they fail, scales them, and connects them with stable names — all driven by a declared desired state.

**Q: Why is it called K8s?**
A: K, then 8 letters (ubernete), then s. Just a shorthand.

**Q: Do I need Kubernetes? Is Docker Compose not enough?**
A: Compose is great on one host. Kubernetes is for multiple hosts: automatic rescheduling when a machine dies, rolling updates, autoscaling, self-healing, a rich ecosystem. It adds real operational complexity; for a single small app, Compose or even plain systemd may be the better choice.

**Q: Is Kubernetes a replacement for Docker?**
A: Not exactly. Docker is a toolset (build images, run containers on one host). Kubernetes orchestrates containers on many hosts and uses a container runtime (containerd or CRI-O) underneath. You still build images (with Docker, BuildKit, Podman…) and push them to a registry.

**Q: Can Kubernetes run Docker images?**
A: Yes. Docker images follow the OCI image spec; containerd and CRI-O run them directly.

**Q: What happened to "dockershim"?**
A: It was the adapter that let kubelet talk to Docker Engine. It was removed in Kubernetes 1.24. Nodes now use CRI runtimes directly. Developer workflows (`docker build`) are unaffected.

**Q: What is the control plane?**
A: The set of components that make decisions and keep cluster state: kube-apiserver, etcd, kube-scheduler, kube-controller-manager (plus cloud-controller-manager in clouds).

**Q: What is a node?**
A: A machine (VM or physical) that runs workloads. It runs kubelet, kube-proxy and a container runtime. The control-plane machines are nodes too, usually tainted to keep ordinary workloads away.

**Q: What is the difference between a node and a pod?**
A: A node is the machine; a pod is a group of one or more containers scheduled onto a node. A node holds many pods.

**Q: Which component actually starts containers?**
A: kubelet, by asking the container runtime (containerd/CRI-O). The scheduler only chooses the node; the API server only stores objects.

**Q: What does the scheduler do exactly?**
A: Picks a node for each unscheduled pod: filter out unsuitable nodes, score the rest, choose the best, and write the node name into the pod object.

**Q: What is etcd and why does everyone say "back it up"?**
A: A consistent key-value database holding the entire cluster state. If it is lost with no backup, you lose the definitions of all your objects.

**Q: Does an application keep running if the control plane goes down?**
A: Pods already running on nodes keep running (kubelet and the runtime are local). But nothing can be created, scaled, rescheduled or healed until the control plane is back, and a node failure during that time is not repaired.

**Q: Why odd numbers of etcd/control-plane nodes?**
A: etcd needs a majority (quorum). 3 members survive 1 failure, 5 survive 2. An even count adds cost without more fault tolerance (4 still tolerates only 1).

**Q: What is a reconciliation loop?**
A: A controller repeatedly observes actual state, compares it to desired state, and acts to remove the difference. This is the mechanism behind self-healing and rollouts.

**Q: What does "declarative" mean here?**
A: You describe the end result you want in a file/object; Kubernetes figures out and continuously performs the steps. The opposite is imperative: you issue each step yourself.

**Q: What is the difference between `spec` and `status`?**
A: `spec` = desired state, written by you. `status` = observed state, written by Kubernetes components. Controllers work to make status match spec.

**Q: What are `apiVersion` and `kind`?**
A: `kind` is the object type (Pod, Deployment…). `apiVersion` is the API group/version that defines it (`v1`, `apps/v1`…). Both are mandatory in every manifest.

**Q: Which port does the API server use?**
A: 6443 on most distributions (kubeadm, rke2 uses 6443 too; some others use 443). It is HTTPS only.

**Q: What is kubeconfig?**
A: A YAML file (default `~/.kube/config`) telling kubectl which clusters exist, which credentials to use, and which cluster/user/namespace combination (context) is active.

**Q: What is a context?**
A: A named combination of cluster + user (+ default namespace) inside kubeconfig. Switching context switches the cluster you operate on.

**Q: How do I use two clusters at once?**
A: Merge their config into one kubeconfig (or set `KUBECONFIG=a:b`), then use `kubectl --context <name> ...` per command or `kubectl config use-context`.

**Q: Where is my kubeconfig on a kubeadm control-plane node?**
A: `/etc/kubernetes/admin.conf`. Copy it to `~/.kube/config` for your user (lesson 52).

**Q: What does `ROLES <none>` mean in `kubectl get nodes`?**
A: The node has no `node-role.kubernetes.io/...` label. Worker nodes normally have none; it is cosmetic.

**Q: What does a NotReady node mean?**
A: The control plane stopped receiving healthy heartbeats from that node's kubelet (kubelet down, node unreachable, runtime failing, CNI missing). Use `kubectl describe node` and check `systemctl status kubelet` and `journalctl -u kubelet` on the node.

**Q: What is a CNI plugin and why is it needed?**
A: Container Network Interface plugin: assigns pod IPs and connects pods across nodes. Kubernetes itself ships no pod network; without a CNI nodes stay NotReady and CoreDNS stays pending.

**Q: What is the difference between kind, minikube, kubeadm, k3s?**
A: kind and minikube are local learning/test clusters. k3s is a lightweight production-capable distribution. kubeadm is the official bootstrap tool for building clusters on machines you manage. rke2 is a hardened distribution.

**Q: What are managed Kubernetes services?**
A: EKS, GKE, AKS and similar: the provider operates the control plane (API server, etcd, scheduler, controllers) and upgrades it; you pay for and manage worker nodes (or use serverless nodes). You cannot SSH into the control plane.

**Q: Which Kubernetes version should I use?**
A: One of the three most recent minor versions (about 14 months of patch support each). Plan upgrades ahead and do them one minor at a time.

**Q: Is `kubectl` the only way to talk to the cluster?**
A: No. It is one client. Anything that speaks HTTPS to the API works: client libraries (Go, Python), `curl` with credentials, dashboards, Terraform's kubernetes provider, Helm, ArgoCD.

**Q: What is a CRD?**
A: CustomResourceDefinition — a way to add your own resource types to the API, usually paired with a controller (an operator). We meet real CRDs later (cert-manager, External Secrets, Gateway API, Kyverno).

**Q: Why are there so many system pods in `kube-system`?**
A: Each control-plane component and add-on (DNS, networking, kube-proxy) runs as pods there on most distributions. They are the cluster itself.

**Q: What is a namespace?**
A: A named partition of the cluster for grouping and isolating objects (teams, environments, apps). Default namespaces: default, kube-system, kube-public, kube-node-lease. More in lesson 35.

## Practice / Exercise

### Core

1. **Bring up the lab.** Install `kubectl` and `kind`, create the 1 control-plane + 2 worker cluster from `kind-config.yaml`. Confirm with `docker ps` that three node containers exist.
2. **Fill in the architecture table.** Run `kubectl get pods -n kube-system -o wide` and for every pod write: which architecture component it is, control plane or node-level, and on which node it runs.
3. **Versions.** Run `kubectl version` and write down client and server versions. Are they within one minor version?
4. **Node inspection.** From `kubectl describe node <worker>` extract: allocatable CPU and memory, container runtime and version, kubelet version, number of non-terminated pods, any taints. Do the same for the control-plane node and compare taints.
5. **Kubeconfig reading.** Open `~/.kube/config`. Identify cluster name, server URL, user name, context name, current-context. Run `kubectl config view --minify` and explain what is redacted and why.
6. **API health.** Run `kubectl get --raw='/readyz?verbose'` and find the line proving etcd is reachable.
7. **Two clusters.** Create a second kind cluster named `second`. Switch between contexts with `use-context`, then use `--context` for a single command. Delete the second cluster and confirm its context disappears.
8. **Reconciliation demo.** Run the `kubectl get pods -n kube-system -w` watch, delete the CoreDNS pods as in Example 8, describe in your own words what the controller did and which component of the architecture performed each step.
9. **Trace the request.** Without notes, draw (paper) the nine-step path from "kubectl command" to "container running", naming every component involved.

### Stretch

1. **Static pods.** Enter `lab-control-plane` with `docker exec -it`, list `/etc/kubernetes/manifests`, and find the `--advertise-address`/`--etcd-servers` flags of the API server in `kube-apiserver.yaml`. Explain why those pods are not recreated by a controller but by kubelet.
2. **Kubelet at work.** On a worker (`docker exec -it lab-worker bash`) run `crictl ps`, `crictl pods`, `journalctl -u kubelet --no-pager | tail`. Compare with `docker ps` on the host and explain why the two lists differ.
3. **Raw API.** Run `kubectl proxy --port=8001 &` then `curl localhost:8001/api/v1/nodes | head -50` and `curl localhost:8001/version`. Identify `metadata`, `spec`, `status` in the JSON of a node. Stop the proxy (`kill %1`).
4. **API discovery.** Run `kubectl get --raw /apis | head -40` and list five API groups. Which group would a Deployment belong to? (You will create one in lesson 36.)
5. **Multiple kubeconfigs.** Copy `~/.kube/config` to `~/lab.conf`, then run `kubectl --kubeconfig ~/lab.conf get nodes`. Set `KUBECONFIG=~/lab.conf:~/.kube/config`, run `kubectl config get-contexts` and describe how files are merged.
6. **Failure thinking.** Write down (no commands) what happens, in order, if (a) a worker node loses power, (b) the scheduler stops, (c) etcd becomes unreachable, (d) the API server restarts for 30 seconds. Which of these stops running containers? Which only stops *changes*?
7. **minikube comparison.** Install minikube, run `minikube start --driver=docker`, compare `kubectl get nodes -o wide` and `kubectl get pods -n kube-system` against kind. What differs (names, CNI, number of nodes)?
8. **Version skew research.** Read the official version-skew policy. State the allowed skew between kube-apiserver, kubelet and kubectl, and why kubelet may be older than the API server but never newer.

## Further Reading

- Kubernetes concepts overview: https://kubernetes.io/docs/concepts/overview/
- Cluster architecture: https://kubernetes.io/docs/concepts/architecture/
- Components: https://kubernetes.io/docs/concepts/overview/components/
- Objects in Kubernetes (spec and status): https://kubernetes.io/docs/concepts/overview/working-with-objects/
- Organizing cluster access with kubeconfig: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/
- Container runtimes and CRI: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- Version skew policy: https://kubernetes.io/releases/version-skew-policy/
- kind quick start: https://kind.sigs.k8s.io/docs/user/quick-start/
- Kubernetes the Hard Way (for the curious): https://github.com/kelseyhightower/kubernetes-the-hard-way

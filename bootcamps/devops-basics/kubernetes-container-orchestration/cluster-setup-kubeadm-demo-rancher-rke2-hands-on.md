# Lesson 52: Cluster Setup – kubeadm (demo) & Rancher rke2 (hands-on)

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-51 (cluster architecture, kubectl, workloads, Services, storage, security). Lesson 10 (networking, ports), lesson 11 (iptables/ufw), lesson 7-8 (systemd, journalctl), lesson 30 (containers, containerd). Student has 2-3 clean Ubuntu 22.04/24.04 VMs (2 vCPU, 2-4 GB RAM each, root or sudo, SSH between them) for the rke2 hands-on; the kubeadm part is an instructor demo.

## Learning Objectives

By end of lesson student can:
- List the node prerequisites for a Kubernetes cluster (swap off, kernel modules, sysctls, container runtime, open ports, unique node identity) and say why each exists
- Install containerd with the systemd cgroup driver and the kubeadm / kubelet / kubectl packages from `pkgs.k8s.io`, and pin their versions with `apt-mark hold`
- Run `kubeadm init` with the right flags and read its output step by step
- Copy the admin kubeconfig, install a CNI plugin (Flannel or Calico) and explain why the Pod CIDR must match
- Explain what kubeadm generates: certificates under `/etc/kubernetes/pki`, kubeconfigs, static Pod manifests, and which `kubeadm init phase` creates what
- Check and renew control-plane certificates, create a new join token, compute the CA cert hash, and join a worker
- Describe how an HA control plane and a kubeadm version upgrade work (overview)
- Install an rke2 server with a `config.yaml`, fetch its kubeconfig, join an rke2 agent and verify the cluster
- Compare kubeadm and rke2 (who installs what, defaults, hardening, upgrade path) and choose between them
- Troubleshoot the four classic failures: kubelet not starting, node `NotReady` (no CNI), cgroup driver mismatch, expired join token

## Topics

- kubeadm demo: prerequisites, kubeadm init (--pod-network-cidr, --apiserver-advertise-address)
- kubeadm internals: certificate generation, static pods (/etc/kubernetes/manifests), phases
- kubeadm join: worker node join with token; CNI plugin installation (Flannel/Calico)
- rke2 hands-on: install server, /etc/rancher/rke2/config.yaml, start service, get kubeconfig
- rke2 worker: install agent, join cluster; verify nodes; kubeadm vs rke2 comparison

## Concepts

### Why this lesson exists: so far the cluster was "given"

Lessons 34-51 used a cluster that already worked (kind, minikube, or a lab cluster prepared by the instructor). Real DevOps work starts earlier: somebody must **create** the cluster, keep its certificates alive, add nodes and upgrade it. This lesson shows the two most common self-managed ways on plain Linux machines:

| Tool | Philosophy | You do | Tool does |
|---|---|---|---|
| **kubeadm** | Official "building block". Bootstraps the Kubernetes control plane only. | Install runtime, kubelet, kubeadm; choose CNI; manage OS prerequisites; plan upgrades | Certificates, kubeconfigs, static Pods for control plane, bootstrap tokens, join |
| **rke2** (Rancher Kubernetes Engine 2, a.k.a. "RKE Government") | Opinionated distribution: one binary + one service, hardened by default | Write a small `config.yaml`, run installer | Embeds containerd, etcd, CNI (canal), CoreDNS, ingress-nginx, metrics-server; CIS profile available |

Managed services (EKS, GKE, AKS) hide all of this — but on-premises, in labs and in many companies you still build clusters yourself. Knowing what kubeadm does also explains what *every* distribution does underneath.

### What a node needs before it can join (prerequisites)

Every machine (control plane and worker) needs:

1. **A supported Linux** with systemd (Ubuntu 22.04/24.04, Debian 12, Rocky/Alma 9, etc.), 2 CPU / 2 GB RAM minimum (control plane realistically 2 CPU / 4 GB).
2. **Swap disabled.** The kubelet refuses to start with swap on by default (`failSwapOn: true`). Reason: the scheduler and the OOM/eviction logic assume memory limits are real; swap makes limits meaningless and latency unpredictable. Disable now and permanently (comment the swap line in `/etc/fstab`).
3. **Kernel modules:** `overlay` (overlayfs for container image layers, lesson 30) and `br_netfilter` (makes bridged traffic visible to iptables so Services and NetworkPolicy work).
4. **Sysctls:** `net.ipv4.ip_forward=1` (a node forwards packets between Pod network and outside), `net.bridge.bridge-nf-call-iptables=1` and `net.bridge.bridge-nf-call-ip6tables=1`.
5. **A container runtime with CRI:** containerd (default choice) or CRI-O. Docker Engine alone is *not* a CRI runtime since dockershim was removed in Kubernetes 1.24 (Docker uses containerd underneath, but kubelet talks CRI to containerd directly).
6. **Cgroup driver match.** On systemd distros both kubelet and containerd must use the **systemd** cgroup driver. Mismatch causes unstable Pods and restart loops (see Pitfalls).
7. **kubeadm, kubelet, kubectl** packages (same minor version; kubelet may be up to 3 minor versions older than the API server in current policy, never newer).
8. **Unique identity:** unique hostname, MAC address and `/sys/class/dmi/id/product_uuid` per node. Cloned VMs often share `product_uuid` or hostname and then nodes overwrite each other.
9. **Network reachability** between nodes and open ports (table below). Firewalls (ufw/firewalld/cloud security groups) are the most common reason `kubeadm join` hangs.
10. **Time sync** (chrony/systemd-timesyncd) — certificates and etcd are sensitive to clock drift.

**Ports that must be open**

| Node | Port | Protocol | Used by |
|---|---|---|---|
| Control plane | 6443 | TCP | kube-apiserver (everyone) |
| Control plane | 2379-2380 | TCP | etcd client/peer (control-plane nodes only) |
| Control plane | 10250 | TCP | kubelet API (API server → kubelet: logs, exec) |
| Control plane | 10257 | TCP | kube-controller-manager (local) |
| Control plane | 10259 | TCP | kube-scheduler (local) |
| Worker | 10250 | TCP | kubelet API |
| Worker | 10256 | TCP | kube-proxy health |
| Worker | 30000-32767 | TCP/UDP | NodePort Services (lesson 37) |
| All | CNI-specific | UDP 8472 (Flannel VXLAN, Canal), UDP 4789 / TCP 179 (Calico VXLAN/BGP) | Pod-to-Pod overlay between nodes |
| rke2 server | 9345 | TCP | rke2 supervisor (agents register here) |

### kubeadm: the flow in one picture

```
 control-plane node                                   worker node
 ──────────────────                                   ───────────
 1. prerequisites (swap, modules, sysctl, containerd, packages)   1. same prerequisites
 2. kubeadm init ──► certs, kubeconfigs, static Pods, etcd, CoreDNS, kube-proxy
 3. copy admin.conf → ~/.kube/config
 4. kubectl apply CNI (Flannel/Calico)  ← nodes stay NotReady until this
 5. kubeadm token create --print-join-command ────────────►  2. kubeadm join <ip>:6443 --token ... --discovery-token-ca-cert-hash sha256:...
                                                              3. kubelet TLS-bootstraps, node appears in `kubectl get nodes`
```

### kubeadm init: what it actually does (phases)

`kubeadm init` is a sequence of **phases**. You can run them individually (`kubeadm init phase <name>`), skip one (`--skip-phases`) or preview everything with `--dry-run`.

| Phase | What it does | Files / objects produced |
|---|---|---|
| `preflight` | Checks swap, ports, runtime socket, kernel modules, CPU/RAM | none (errors if a check fails) |
| `certs` | Creates the cluster CA and component certificates | `/etc/kubernetes/pki/*` |
| `kubeconfig` | Creates kubeconfig files for admin, kubelet, controller-manager, scheduler | `/etc/kubernetes/admin.conf`, `kubelet.conf`, `controller-manager.conf`, `scheduler.conf` (plus `super-admin.conf` in 1.29+) |
| `etcd` | Writes the local etcd static Pod manifest | `/etc/kubernetes/manifests/etcd.yaml`, data in `/var/lib/etcd` |
| `control-plane` | Writes static Pod manifests for apiserver, controller-manager, scheduler | `/etc/kubernetes/manifests/kube-*.yaml` |
| `kubelet-start` | Writes kubelet config and (re)starts kubelet | `/var/lib/kubelet/config.yaml`, `/var/lib/kubelet/kubeadm-flags.env` |
| `upload-config` | Stores ClusterConfiguration and kubelet config as ConfigMaps in `kube-system` | `kubeadm-config`, `kubelet-config` |
| `mark-control-plane` | Labels and taints the node | label `node-role.kubernetes.io/control-plane`, taint `node-role.kubernetes.io/control-plane:NoSchedule` |
| `bootstrap-token` | Creates the first join token and RBAC for TLS bootstrap | Secret `bootstrap-token-xxxxxx` in `kube-system` |
| `addon` | Installs CoreDNS (Deployment) and kube-proxy (DaemonSet) | in `kube-system` |

How the control plane runs without a control plane: the kubelet watches `staticPodPath` (`/etc/kubernetes/manifests`) and starts every manifest it finds as a **static Pod**, without the API server (lesson 34). kubeadm only has to drop YAML files there. Editing a file there makes the kubelet recreate that Pod — this is how API server flags are changed (lesson 49).

### kubeadm init: the important flags

| Flag | Meaning | Notes |
|---|---|---|
| `--pod-network-cidr=10.244.0.0/16` | CIDR from which Pod IPs are allocated (node-cidr allocation by controller-manager) | Must match the CNI: Flannel default `10.244.0.0/16`, Calico default `192.168.0.0/16`. Must not overlap node or Service networks |
| `--service-cidr=10.96.0.0/12` | ClusterIP range (default shown) | Change only if it overlaps your network |
| `--apiserver-advertise-address=<ip>` | IP the API server announces to other components | Use the node's private/LAN IP when the node has several NICs; default is the default-route interface |
| `--control-plane-endpoint=<dns-or-vip>:6443` | Stable address for the API (load balancer / VIP / DNS) | **Required for later HA** — cannot be added later without pain. Put it even on a single node if you may grow |
| `--kubernetes-version=v1.31.4` | Control-plane component version | Default is kubeadm's own version |
| `--cri-socket=unix:///run/containerd/containerd.sock` | Which runtime socket | Needed only when several runtimes are installed |
| `--node-name=<name>` | Node name in the cluster | Default is hostname |
| `--upload-certs` | Uploads control-plane certs (encrypted) so other control-plane nodes can join | HA only |
| `--config kubeadm-config.yaml` | Use a file instead of flags | Preferred for repeatability and Git storage |
| `--dry-run` | Print what would happen, write to a temp dir | Safe way to demo phases |
| `--skip-phases=addon/kube-proxy` | Skip a phase | Used when a CNI (e.g., Cilium) replaces kube-proxy |

Config-file equivalent (brief — YAML you already know):

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.56.10
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.31.4
controlPlaneEndpoint: "k8s-api.lab.local:6443"
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

Run: `sudo kubeadm init --config kubeadm-config.yaml`. (Use `kubeadm config print init-defaults` to see all fields. API version `v1beta3` is used by kubeadm before 1.31.)

### The CNI plugin and why the Pod CIDR must match

After `kubeadm init` the node is **NotReady** and CoreDNS Pods are **Pending**: no network plugin exists, so the kubelet reports `NetworkReady=false`. A CNI plugin (Container Network Interface) wires every Pod into the cluster network — assigns IPs, creates routes/overlay between nodes.

- **Flannel:** simple VXLAN overlay, easy for labs. Reads the Pod CIDR from the node (`spec.podCIDR`) and its own ConfigMap (`net-conf.json`, default `10.244.0.0/16`). **Does not implement NetworkPolicy** (lesson 48).
- **Calico:** routing (BGP/VXLAN/IP-in-IP) and full NetworkPolicy. Default IP pool `192.168.0.0/16` — if you init with another CIDR, the Calico `IPPool` (`CALICO_IPV4POOL_CIDR`) must be changed too.
- **Canal** (rke2 default): Flannel for networking + Calico for NetworkPolicy.
- **Cilium** (eBPF; can replace kube-proxy) — mention only.

If `--pod-network-cidr` and the CNI pool differ, Pods get IPs, but cross-node routing or IP allocation breaks (`podCIDR` missing on node, Pods stuck in `ContainerCreating` with `failed to allocate for range`).

### Certificates and kubeconfigs created by kubeadm

`/etc/kubernetes/pki/` holds:

| File | Purpose |
|---|---|
| `ca.crt` / `ca.key` | Cluster CA — signs all other certs. Valid **10 years** |
| `apiserver.crt/key` | API server serving cert (SANs: node IP, `kubernetes`, `kubernetes.default.svc...`, control-plane endpoint) |
| `apiserver-kubelet-client.crt/key` | API server → kubelet client cert |
| `front-proxy-ca.*`, `front-proxy-client.*` | Aggregated API layer |
| `sa.key` / `sa.pub` | Signing key pair for ServiceAccount tokens (lesson 47) |
| `etcd/ca.*`, `etcd/server.*`, `etcd/peer.*`, `etcd/healthcheck-client.*`, `apiserver-etcd-client.*` | etcd's own CA and certs |

Leaf certificates are valid **1 year** and are renewed automatically only on `kubeadm upgrade`. A cluster nobody upgrades for a year **stops working** (`x509: certificate has expired`). Check and renew:

```bash
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
# static Pods read certs only at start → restart them (move manifests out and back, or restart containers)
```

Kubeconfigs: `/etc/kubernetes/admin.conf` (cluster-admin, group `kubeadm:cluster-admins`, embedded client cert), `kubelet.conf`, `controller-manager.conf`, `scheduler.conf`. `admin.conf` is **the** most sensitive file on the node: anyone with it is cluster-admin. Since 1.29 there is also `super-admin.conf` (system:masters, bypasses RBAC) for break-glass use.

### Joining a worker

`kubeadm join` performs **TLS bootstrapping**:

1. The new node contacts `https://<endpoint>:6443` and downloads cluster info from the public `cluster-info` ConfigMap (`kube-public`) using the **bootstrap token** (format `abcdef.0123456789abcdef`, default TTL **24 h**).
2. It verifies the API server's CA against `--discovery-token-ca-cert-hash sha256:<hash>` (protects from man-in-the-middle).
3. Kubelet submits a CSR; it is auto-approved; the node gets its kubelet client certificate and registers.

```bash
# on control plane — prints a complete, ready join command with a fresh token
sudo kubeadm token create --print-join-command
# list/delete tokens
sudo kubeadm token list
sudo kubeadm token delete <token>
# recompute the CA hash by hand
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

### HA control plane (concept only)

Single control-plane node = single point of failure. HA topologies:

- **Stacked etcd:** each control-plane node runs apiserver, scheduler, controller-manager **and** a local etcd member. Simplest; 3 nodes tolerate 1 failure (etcd quorum = N/2+1).
- **External etcd:** etcd runs on separate machines. More machines, more isolation.
- A **load balancer / VIP** (HAProxy + keepalived, kube-vip, cloud LB — HAProxy from lesson 13) in front of port 6443; its address is `--control-plane-endpoint` given at `kubeadm init`.
- Additional control-plane nodes join with `kubeadm join <endpoint> --control-plane --certificate-key <key>` after `kubeadm init --upload-certs` (or `kubeadm init phase upload-certs --upload-certs`).

### kubeadm upgrade (overview)

Upgrade one minor version at a time (1.30 → 1.31 → 1.32, never skip), control plane first, then workers:

```bash
# first control-plane node
sudo apt-mark unhold kubeadm && sudo apt-get install -y kubeadm=1.31.x-* && sudo apt-mark hold kubeadm
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.x
# every node: drain, upgrade kubelet/kubectl, restart, uncordon
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
sudo apt-get install -y kubelet=1.31.x-* kubectl=1.31.x-*
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon <node>
# other control-plane nodes and workers use:  sudo kubeadm upgrade node
```

`kubeadm upgrade` also renews certificates — another reason to upgrade at least yearly. `apt-mark hold` stops an unattended `apt upgrade` from jumping the kubelet to a new minor version.

### rke2: an all-in-one distribution

rke2 ships as **one binary + one systemd unit** per node:

- `rke2-server` — runs control plane + etcd (embedded) + kubelet + containerd; also a **supervisor** on port **9345** that agents use to register and receive configuration.
- `rke2-agent` — worker: kubelet, containerd, kube-proxy, CNI.

Everything (kube-apiserver, scheduler, controller-manager, etcd, kube-proxy) runs as **static Pods** from `/var/lib/rancher/rke2/agent/pod-manifests/`, images are pulled from Rancher's registry (or loaded from air-gap tarballs). Packaged add-ons are deployed automatically: CNI (**canal** by default), CoreDNS, metrics-server, **ingress-nginx** (lesson 38 — already installed!), and a snapshot-controller.

Key paths:

| Path | Content |
|---|---|
| `/etc/rancher/rke2/config.yaml` | Your configuration (read at start) |
| `/etc/rancher/rke2/config.yaml.d/*.yaml` | Drop-in config fragments (merged alphabetically) |
| `/etc/rancher/rke2/rke2.yaml` | Admin kubeconfig (server only) |
| `/var/lib/rancher/rke2/server/node-token` | Join token for other nodes |
| `/var/lib/rancher/rke2/bin/` | `kubectl`, `crictl`, `ctr` (not in PATH by default) |
| `/var/lib/rancher/rke2/server/db/snapshots/` | etcd snapshots (taken automatically every 12 h, 5 kept) |
| `/var/lib/rancher/rke2/agent/pod-manifests/` | Static Pod manifests of control plane |
| `/var/lib/rancher/rke2/server/manifests/` | Add-on manifests applied automatically |
| `/usr/local/bin/rke2-uninstall.sh`, `rke2-killall.sh` | Cleanup scripts |

`config.yaml` keys (YAML form of CLI flags, drop the leading `--`):

| Key | Example | Meaning |
|---|---|---|
| `token` | `token: lab-shared-secret` | Shared cluster secret; set explicitly so you do not have to fetch `node-token` |
| `tls-san` | `tls-san: [rke2.lab.local, 192.168.56.10]` | Extra names/IPs in the API server cert (needed for LB / DNS / public IP) |
| `cni` | `cni: calico` | `canal` (default), `calico`, `cilium`, `none` |
| `node-label` | `node-label: ["role=ingress"]` | Labels the node (lesson 46) |
| `node-taint` | `node-taint: ["dedicated=db:NoSchedule"]` | Taints the node |
| `disable` | `disable: [rke2-ingress-nginx]` | Do not deploy a packaged component |
| `write-kubeconfig-mode` | `"0644"` | Permissions of `rke2.yaml` (default 0600, root only) |
| `server` | `server: https://192.168.56.10:9345` | **Agent** (or extra server): where to register |
| `cluster-cidr` / `service-cidr` | `10.42.0.0/16` / `10.43.0.0/16` | Pod / Service ranges (these are the defaults) |
| `node-name` | `node-name: worker-1` | Override the node name |
| `profile` | `profile: cis` | Apply CIS-hardened settings (requires prerequisites below) |
| `selinux` | `selinux: true` | Enable SELinux support on RHEL-family |

### kubeadm vs rke2

| Aspect | kubeadm | rke2 |
|---|---|---|
| Scope | Control-plane bootstrap only | Full distribution (runtime, CNI, DNS, ingress, metrics) |
| Container runtime | You install containerd | Embedded containerd |
| CNI | You choose and apply it manually (cluster is `NotReady` until you do) | canal pre-installed (switchable via `cni:`) |
| etcd | Stacked static Pod (or external) | Embedded etcd with automatic snapshots |
| Ingress / metrics-server | Not included | ingress-nginx, metrics-server included |
| Hardening | You apply CIS controls (lesson 49) | `profile: cis` and secure defaults (Pod Security, NetworkPolicies for system namespaces, restricted file permissions) |
| Config style | Flags / YAML config + OS steps | One `config.yaml` |
| Join | token + CA hash, explicit | `server:` + `token:` in config |
| Upgrade | Manual `kubeadm upgrade` per node, manage packages | Re-run installer with new version/channel + restart service (or system-upgrade-controller) |
| Best for | Learning, custom clusters, exam (CKA), when you want full control | Fast, reproducible, hardened on-prem / edge clusters |

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `swapoff -a` | Disable swap until reboot | `sudo swapoff -a` |
| `modprobe` | Load kernel module now | `sudo modprobe overlay && sudo modprobe br_netfilter` |
| `sysctl --system` | Apply all sysctl.d files | `sudo sysctl --system` |
| `containerd config default` | Print default containerd config | `containerd config default \| sudo tee /etc/containerd/config.toml` |
| `apt-mark hold` / `unhold` | Pin / unpin package versions | `sudo apt-mark hold kubelet kubeadm kubectl` |
| `kubeadm init` | Bootstrap control plane | `sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.10` |
| `kubeadm init --dry-run` | Preview without changes | `sudo kubeadm init --dry-run --pod-network-cidr=10.244.0.0/16` |
| `kubeadm init phase` | Run one phase | `sudo kubeadm init phase certs all` |
| `kubeadm config print init-defaults` | Show default config | `kubeadm config print init-defaults > kubeadm-config.yaml` |
| `kubeadm config images pull` | Pre-pull control-plane images | `sudo kubeadm config images pull` |
| `kubeadm token create --print-join-command` | New token + full join command | `sudo kubeadm token create --print-join-command` |
| `kubeadm token list` | Show tokens and TTL | `sudo kubeadm token list` |
| `kubeadm join` | Join a node | `sudo kubeadm join 192.168.56.10:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:<hash>` |
| `kubeadm certs check-expiration` | Show cert expiry dates | `sudo kubeadm certs check-expiration` |
| `kubeadm certs renew all` | Renew all leaf certs | `sudo kubeadm certs renew all` |
| `kubeadm upgrade plan` / `apply` / `node` | Upgrade cluster | `sudo kubeadm upgrade apply v1.31.4` |
| `kubeadm reset` | Undo init/join on this node | `sudo kubeadm reset -f` |
| `crictl ps` / `crictl logs` | Inspect containers via CRI (works when API is down) | `sudo crictl ps -a` |
| `journalctl -u kubelet` | Kubelet logs | `sudo journalctl -u kubelet -f --no-pager` |
| `curl -sfL https://get.rke2.io \| sh -` | Install rke2 | `curl -sfL https://get.rke2.io \| sudo INSTALL_RKE2_TYPE=agent sh -` |
| `systemctl enable --now rke2-server` | Start rke2 server | `sudo systemctl enable --now rke2-server.service` |
| `rke2 etcd-snapshot save` | Manual etcd snapshot | `sudo rke2 etcd-snapshot save --name pre-upgrade` |
| `rke2-uninstall.sh` | Remove rke2 | `sudo /usr/local/bin/rke2-uninstall.sh` |

## Examples / Walkthrough

### Part A — kubeadm demo (instructor shows on 1 control-plane + 1 worker)

#### A1. Prerequisites on every node

```bash
# 1. swap off now and after reboot
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab
free -h                                  # Swap line must be 0

# 2. kernel modules (load now + on boot)
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
lsmod | grep -E 'overlay|br_netfilter'

# 3. sysctls
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
sysctl net.ipv4.ip_forward               # = 1

# 4. unique identity checks
hostnamectl --static
ip link | grep ether
sudo cat /sys/class/dmi/id/product_uuid

# 5. hostnames resolvable (lab: /etc/hosts)
cat <<'EOF' | sudo tee -a /etc/hosts
192.168.56.10 cp1
192.168.56.11 worker1
EOF
```

#### A2. Install containerd with the systemd cgroup driver

```bash
sudo apt-get update
sudo apt-get install -y containerd          # Ubuntu package (or containerd.io from Docker's apt repo)

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
# flip the cgroup driver to systemd (the single most important setting)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep -n SystemdCgroup /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager | head -5
```

#### A3. Install kubeadm, kubelet, kubectl from pkgs.k8s.io

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings

# repo is per minor version — change v1.31 to the version you want
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # no accidental minor upgrades
sudo systemctl enable --now kubelet          # will crash-loop until init/join — this is NORMAL
kubeadm version -o short
```

#### A4. Initialize the control plane (cp1 only)

```bash
# optional: pull images ahead (shows what will be run)
sudo kubeadm config images pull

# safe preview first
sudo kubeadm init --dry-run \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.56.10 | head -40

# real init
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.56.10 \
  --kubernetes-version=v1.31.4 | tee ~/kubeadm-init.log
```

Walk through the output (read each line aloud):

```
[init] Using Kubernetes version: v1.31.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [cp1 kubernetes kubernetes.default ...] and IPs [10.96.0.1 192.168.56.10]
[kubeconfig] Writing "admin.conf" kubeconfig file
[control-plane] Creating static Pod manifest for "kube-apiserver"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[apiclient] All control plane components are healthy after 14.5s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node cp1 as control-plane by adding the labels/taints
[bootstrap-token] Using token: abcdef.0123456789abcdef
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
...
kubeadm join 192.168.56.10:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:1f9c...e3
```

Copy the kubeconfig and look at the state:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes                       # cp1   NotReady   control-plane
kubectl get pods -A                     # coredns Pending (no CNI yet), others Running
kubectl describe node cp1 | grep -A3 -i 'Ready\|NetworkReady'
# "container runtime network not ready: NetworkReady=false ... cni plugin not initialized"
```

#### A5. Install the CNI

```bash
# Flannel (default CIDR 10.244.0.0/16 matches --pod-network-cidr above)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

kubectl get pods -n kube-flannel -w
kubectl get nodes                       # cp1 Ready
kubectl get pods -n kube-system         # coredns now Running
```

Calico alternative (supports NetworkPolicy; download the manifest, edit the CIDR if you did not use `192.168.0.0/16`, apply):

```bash
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
# uncomment and set CALICO_IPV4POOL_CIDR to your --pod-network-cidr
kubectl apply -f calico.yaml
```

#### A6. Look inside what kubeadm built

```bash
ls /etc/kubernetes/                     # admin.conf controller-manager.conf kubelet.conf manifests pki scheduler.conf super-admin.conf
ls /etc/kubernetes/manifests/           # etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml
ls /etc/kubernetes/pki /etc/kubernetes/pki/etcd

# one certificate in detail
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject -issuer -dates -ext subjectAltName

# expiry overview
sudo kubeadm certs check-expiration

# the static Pod really is just a file
sudo grep -E 'image:|--advertise-address|--service-cluster-ip-range|--etcd-servers' /etc/kubernetes/manifests/kube-apiserver.yaml

# kubelet knows where to look
sudo grep staticPodPath /var/lib/kubelet/config.yaml

# mirror Pods visible through the API (suffix = node name)
kubectl get pods -n kube-system -o wide | grep cp1

# kubeadm's own state is in ConfigMaps
kubectl -n kube-system get cm kubeadm-config -o yaml | head -30
```

Static Pod demo (2 min): `sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/` → within seconds `kubectl get pods -n kube-system` loses `kube-scheduler-cp1` (new Pods stay `Pending`); move it back and it returns. Shows that the directory *is* the source of truth.

Running one phase by hand (explains the structure; use `--dry-run` or a scratch dir):

```bash
sudo kubeadm init phase certs --help
sudo kubeadm init phase kubeconfig admin --help
sudo kubeadm init --help | grep -A12 'skip-phases'
```

#### A7. Join the worker

On the control plane:

```bash
sudo kubeadm token create --print-join-command
# kubeadm join 192.168.56.10:6443 --token 5xk3...  --discovery-token-ca-cert-hash sha256:1f9c...
```

On worker1 (after steps A1-A3 — **not** A4):

```bash
sudo kubeadm join 192.168.56.10:6443 --token 5xk3p2.8h1q0d7v2c9a1x6z \
  --discovery-token-ca-cert-hash sha256:1f9c2b7e...
```

Verify from the control plane:

```bash
kubectl get nodes -o wide -w             # worker1 NotReady → Ready after CNI Pod starts on it
kubectl label node worker1 node-role.kubernetes.io/worker=
kubectl get pods -A -o wide              # a kube-proxy + CNI Pod on each node (DaemonSets)
kubectl create deployment web --image=nginx --replicas=3
kubectl get pods -o wide                 # Pods land on worker1 (control plane tainted)
```

Control-plane taint demo: `kubectl describe node cp1 | grep Taints` → `node-role.kubernetes.io/control-plane:NoSchedule` (lesson 46). On a 1-node lab remove it with `kubectl taint nodes cp1 node-role.kubernetes.io/control-plane:NoSchedule-`.

#### A8. Cleanup / reset

```bash
sudo kubeadm reset -f                    # on any node: stop kubelet, remove manifests, certs, etcd data
sudo rm -rf /etc/cni/net.d $HOME/.kube/config
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo ipvsadm --clear 2>/dev/null || true # only if ipvs was used
```

`kubeadm reset` does **not** clean CNI configuration, iptables rules or kubeconfig — do it manually (above), otherwise a re-init may behave oddly.

### Part B — rke2 hands-on (every student, 2-3 VMs)

Roles: `rke2-server` on VM1 (e.g. `192.168.56.20`), `rke2-agent` on VM2 (and VM3). Steps per node assume Ubuntu with sudo; swap should be disabled as in A1 (rke2 tolerates swap better but keep the habit) and port 9345/6443 reachable.

#### B1. Install and configure the server (VM1)

```bash
sudo -i
curl -sfL https://get.rke2.io | sh -         # type defaults to "server"
# pin a version / channel instead of "latest stable":
#   curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.31 sh -
#   curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.31.4+rke2r1 sh -

mkdir -p /etc/rancher/rke2
cat <<'EOF' > /etc/rancher/rke2/config.yaml
token: lab-shared-secret
tls-san:
  - 192.168.56.20
  - rke2.lab.local
write-kubeconfig-mode: "0644"
node-label:
  - "env=lab"
# cni: canal            # default; uncomment calico/cilium to switch
# disable:
#   - rke2-ingress-nginx
EOF

systemctl enable --now rke2-server.service   # first start takes 2-5 minutes (pulls images, creates etcd)
journalctl -u rke2-server -f                 # watch; Ctrl-C when "Running RKE2" and no errors
```

Get kubectl and the kubeconfig:

```bash
ls /var/lib/rancher/rke2/bin/                # kubectl crictl ctr
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc

kubectl get nodes -o wide                    # server node Ready with roles control-plane,etcd,master
kubectl get pods -A                          # canal, coredns, ingress-nginx, metrics-server, etcd, kube-apiserver...
kubectl get ingressclass                     # nginx — already installed
```

Use kubectl from your laptop: copy `/etc/rancher/rke2/rke2.yaml`, replace `server: https://127.0.0.1:6443` with `https://192.168.56.20:6443` (the IP/name must be in `tls-san`, otherwise `x509: certificate is valid for ..., not ...`).

Where the token lives (if you did not set `token:`):

```bash
sudo cat /var/lib/rancher/rke2/server/node-token    # K10<hash>::server:<secret>
```

#### B2. Install and join the agent (VM2, VM3)

```bash
sudo -i
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

mkdir -p /etc/rancher/rke2
cat <<'EOF' > /etc/rancher/rke2/config.yaml
server: https://192.168.56.20:9345
token: lab-shared-secret
node-label:
  - "env=lab"
EOF

systemctl enable --now rke2-agent.service
journalctl -u rke2-agent -f                  # look for "Successfully registered node"
```

Notice the **9345** port: agents talk to the rke2 supervisor first, not directly to 6443.

#### B3. Verify the whole cluster (from VM1)

```bash
kubectl get nodes -o wide                    # 2-3 nodes Ready
kubectl get nodes --show-labels | grep env=lab
kubectl label node <agent-name> node-role.kubernetes.io/worker=

kubectl create deployment hello --image=nginx --replicas=4
kubectl expose deployment hello --port 80
kubectl get pods -o wide                     # spread across the nodes
kubectl run -it --rm t --image=busybox --restart=Never -- wget -qO- hello

# host-level inspection of the runtime (no docker command exists here)
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
/var/lib/rancher/rke2/bin/crictl ps
ls /var/lib/rancher/rke2/agent/pod-manifests/    # etcd.yaml kube-apiserver.yaml ... static Pods
sudo rke2 etcd-snapshot save --name lab-first
ls /var/lib/rancher/rke2/server/db/snapshots/
```

Change a config value and restart (demo): add `node-taint` or `disable: [rke2-ingress-nginx]` to `config.yaml`, `systemctl restart rke2-server`, observe the change. The config file is read on every start — it is the single source of truth.

#### B4. CIS profile (preview)

```bash
# prerequisites on the server host (done once)
sudo cp -f /usr/local/share/rke2/rke2-cis-sysctl.conf /etc/sysctl.d/60-rke2-cis.conf
sudo systemctl restart systemd-sysctl
sudo useradd -r -c "etcd user" -s /sbin/nologin -M etcd -U
# config.yaml: add
#   profile: cis
```

Then run kube-bench from lesson 49 and compare the failures with a kubeadm cluster — the rke2 one fails far fewer checks out of the box. (Profile value is `cis` on current releases; older releases used versioned names such as `cis-1.23` — check the docs for your version.)

#### B5. Cleanup

```bash
sudo /usr/local/bin/rke2-killall.sh          # stop all processes and mounts
sudo /usr/local/bin/rke2-uninstall.sh        # remove binaries, data, configs (agent script on agents)
```

### Part C — troubleshooting cheat sheet (use live with intentionally broken steps)

| Symptom | Check | Cause → fix |
|---|---|---|
| `kubeadm init` fails preflight: `[ERROR Swap]: running with swap on is not supported` | `free -h` | `swapoff -a` + fstab |
| Preflight: `/proc/sys/net/bridge/bridge-nf-call-iptables does not exist` or `ip_forward ... 0` | `lsmod`, `sysctl` | Load `br_netfilter`, apply sysctl file |
| Preflight: `container runtime is not running` / CRI errors | `systemctl status containerd`, `crictl info` | Start containerd; if `config.toml` is missing/disabled_plugins contains `cri`, regenerate default config |
| `kubelet` crash-loops after init | `journalctl -u kubelet -n 50 --no-pager` | Read the *last* error: swap, cgroup driver, missing config, bad certificate |
| Control plane Pods restart repeatedly, `kubectl` flaps | `crictl ps -a`, `crictl logs <id>` | **cgroup driver mismatch**: containerd `SystemdCgroup=false` vs kubelet `systemd` → set true, restart containerd + kubelet |
| Node `NotReady`, `NetworkReady=false cni plugin not initialized` | `kubectl describe node`, `ls /etc/cni/net.d`, `kubectl get pods -n kube-flannel` | No CNI or CNI Pod failing; apply CNI, check CIDR match |
| CoreDNS `Pending` / `ContainerCreating` | `kubectl describe pod -n kube-system coredns-...` | Waiting for CNI; or CIDR conflict |
| `kubeadm join`: `couldn't validate the identity of the API Server` / timeout | `nc -vz cp 6443`, firewall | Port 6443 blocked, wrong IP, DNS name unresolved |
| `kubeadm join`: `invalid bootstrap token` / `token ... expired` | `kubeadm token list` on control plane | Token TTL 24 h → `kubeadm token create --print-join-command` |
| `kubeadm join`: `cluster CA found in cluster-info ConfigMap is invalid: none of the public keys ... match` | compare hash | Wrong `--discovery-token-ca-cert-hash` → recompute with the openssl one-liner |
| Two nodes with the same name / one replaces the other | `hostnamectl`, `product_uuid` | Cloned VMs → unique hostname (`hostnamectl set-hostname`), `kubeadm reset`, rejoin |
| `x509: certificate has expired or is not yet valid` | `kubeadm certs check-expiration`, `date` | Renew certs; fix clock (chrony) |
| `The connection to the server localhost:8080 was refused` | `echo $KUBECONFIG; ls ~/.kube` | kubeconfig not copied |
| rke2 server: service active but `kubectl` refused | `journalctl -u rke2-server -f`, `ss -ltnp \| grep -E '6443\|9345'` | Still starting; or port clash; wait for "Running RKE2" |
| rke2 agent stuck: `Failed to connect to proxy ... 9345` | `curl -k https://<server>:9345/ping` | Firewall/wrong `server:`; token mismatch → `token` must equal on both sides |
| rke2: `x509: certificate is valid for 127.0.0.1, ..., not <ip>` | kubeconfig server address | Add that name/IP to `tls-san` and restart server (certs regenerate) |
| rke2: `kubectl: command not found` | `ls /var/lib/rancher/rke2/bin` | Not in PATH — export it |

## Common Pitfalls

- **Forgetting to disable swap** — kubelet refuses to start; often re-appears after reboot because `/etc/fstab` was not edited.
- **cgroup driver mismatch (containerd `SystemdCgroup = false`)** — control plane Pods restart every few minutes for "no reason". Set `SystemdCgroup = true`; kubeadm 1.22+ defaults kubelet to `systemd`.
- **Skipping `br_netfilter` / `ip_forward`** — preflight fails, or init passes but Services do not work. Put settings in `/etc/modules-load.d` and `/etc/sysctl.d` so they survive reboots.
- **Running `kubeadm init` on a worker, or `kubeadm join` on the control plane** — init creates a *new* cluster; join is for nodes of an existing one. `kubeadm reset` to undo.
- **Pod CIDR different from the CNI default** — nodes Ready but cross-node Pod traffic broken. Match `--pod-network-cidr` with Flannel `net-conf.json` / Calico `CALICO_IPV4POOL_CIDR`.
- **Pod CIDR overlapping the node LAN or VPN network** — random unreachability. Choose non-overlapping ranges (e.g., `10.244.0.0/16` when LAN is `192.168.x.x`).
- **Using the cluster-info before CNI** — "nodes NotReady / CoreDNS Pending" right after init is expected, not a failure. Apply the CNI.
- **No `--control-plane-endpoint` on a cluster that may grow** — converting to HA later means regenerating API server certificates. Use a DNS name or VIP from day one.
- **Letting `apt upgrade` bump kubelet** — skewed versions after reboot. `apt-mark hold` and upgrade deliberately.
- **Ignoring certificate expiry** — a kubeadm cluster dies after 365 days without an upgrade. Add `kubeadm certs check-expiration` to monitoring/runbooks (upgrades renew certs).
- **Copying `admin.conf` around** — it is cluster-admin forever. Create per-user kubeconfigs with RBAC (lesson 47).
- **Forgetting that tokens expire** — the join command printed during `init` is valid only 24 h. Always generate a new one with `--print-join-command`.
- **rke2: wrong `tls-san`** — client from outside gets certificate errors. List every name/IP clients use (LB, DNS, public IP).
- **rke2: different `token` values** — agent loops with `401 Unauthorized`/`bad certificate`. Same token on all nodes.
- **rke2: editing `config.yaml` and not restarting the service** — config is read at start only.
- **rke2: expecting `docker`, `kubectl` in PATH** — binaries live in `/var/lib/rancher/rke2/bin`; use `crictl`, not `docker ps`.
- **Cloning VMs without changing hostname/machine-id** — duplicate node identity; use `hostnamectl set-hostname` and regenerate `/etc/machine-id`.
- **Reusing a node without full reset** — leftover `/etc/cni/net.d`, iptables rules, `/var/lib/etcd` break a new init. Reset and clean (A8).

## FAQ

**Q: What is the difference between kubeadm and kubectl?**
A: `kubectl` is the client that talks to a running cluster (lessons 35-36). `kubeadm` is an administrator tool that *creates* and maintains the cluster itself (init, join, certs, upgrade). kubeadm runs on nodes with root; kubectl runs anywhere that has a kubeconfig.

**Q: Why is my node `NotReady` right after `kubeadm init`?**
A: No CNI plugin is installed yet, so the kubelet reports that the network is not ready and CoreDNS stays `Pending`. Apply Flannel or Calico; within a minute the node becomes `Ready`.

**Q: Why must swap be disabled?**
A: Kubernetes resource limits and scheduling assume memory is real RAM. Swap makes memory behaviour unpredictable and defeats limits/eviction. The kubelet refuses to start with swap enabled (`failSwapOn: true`). Newer versions support swap on nodes as an explicit option, but the default for classic setups is "off".

**Q: Which `--pod-network-cidr` should I use?**
A: The one your CNI expects: `10.244.0.0/16` for Flannel's default manifest, `192.168.0.0/16` for Calico's default. It must not overlap your node network, Service CIDR (`10.96.0.0/12`) or VPN ranges.

**Q: What does `--apiserver-advertise-address` do?**
A: It is the IP the API server advertises to the rest of the cluster (it ends up in the API server certificate, in `kubernetes` Service endpoint and in etcd/peer URLs). If the node has several network cards, set it to the one the other nodes can reach. On cloud VMs use the private IP and add the public IP via `--apiserver-cert-extra-sans`.

**Q: Do I need `--control-plane-endpoint` for a single node?**
A: Not required, but recommended if the cluster may later become HA. It sets a stable DNS name / VIP that all kubeconfigs and kubelets use instead of one node's IP.

**Q: Where are the control plane components if there is no Deployment for them?**
A: They are **static Pods** — YAML files in `/etc/kubernetes/manifests` that the kubelet runs directly. The API shows read-only mirror Pods named `<component>-<nodename>`. To change API server flags edit the file; the kubelet restarts the Pod.

**Q: I deleted `kube-apiserver-cp1` Pod. Will it come back?**
A: Yes. You deleted only the mirror object; the kubelet keeps the real container running from the manifest. To remove a static Pod, delete/move its manifest file.

**Q: How do I add another worker later?**
A: Prepare it (prerequisites A1-A3), then run `sudo kubeadm token create --print-join-command` on the control plane and execute the printed command on the new node. For rke2 install the agent and write `server:` + `token:`.

**Q: My join token expired. What now?**
A: Tokens live 24 hours. Create a new one: `sudo kubeadm token create --print-join-command`. For a token that never expires use `--ttl 0` (not recommended for security).

**Q: How do I find `--discovery-token-ca-cert-hash` if I lost the output?**
A: `openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`, prefix result with `sha256:`. Or simply create a new token with `--print-join-command`.

**Q: How long are kubeadm certificates valid, and what happens when they expire?**
A: Leaf certificates 1 year, CA 10 years. After expiry the API server and kubelet cannot authenticate each other and kubectl fails with `x509: certificate has expired`. Renew with `kubeadm certs renew all` and restart the control-plane static Pods; upgrading with `kubeadm upgrade` also renews them.

**Q: How do I restart the control-plane static Pods after renewing certificates?**
A: Move the four YAML files out of `/etc/kubernetes/manifests` for ~20 seconds, then move them back. Or `crictl stop` their containers and let the kubelet recreate them.

**Q: Can I use Docker Engine instead of containerd?**
A: Kubernetes removed dockershim in 1.24. Docker Engine can be used only with the external `cri-dockerd` adapter. In practice, use containerd (Docker already runs on it) or CRI-O.

**Q: Why does `crictl` exist? Why not `docker ps`?**
A: On a Kubernetes node the runtime is containerd; `crictl` speaks the CRI protocol to it and shows Pod sandboxes and containers even when the API server is down. It is the main tool when `kubectl` does not work. Needs `/etc/crictl.yaml` (or `CRI_CONFIG_FILE` on rke2) pointing at the runtime socket.

**Q: What is a bootstrap token?**
A: A short-lived shared secret stored as a Secret `bootstrap-token-<id>` in `kube-system`. It authorizes a new node to download `cluster-info` and request a kubelet certificate. After the node joins it uses its own certificate; the token is no longer needed.

**Q: Why do I need the CA cert hash when I already have a token?**
A: The token proves the node is allowed to join; the hash proves the *API server* is the right one (not a man-in-the-middle). Without pinning, the first connection would trust whatever answers on that IP.

**Q: How do I make a highly available control plane with kubeadm?**
A: 3 (or 5) control-plane nodes, a load balancer/VIP on port 6443, `kubeadm init --control-plane-endpoint <vip>:6443 --upload-certs` on the first node, then `kubeadm join ... --control-plane --certificate-key <key>` on others. etcd uses quorum (N/2+1): 3 nodes tolerate 1 failure, 5 tolerate 2. Even numbers add no safety.

**Q: Can the control plane node also run my Pods?**
A: By default it is tainted `NoSchedule`. For a single-node lab remove the taint: `kubectl taint nodes <node> node-role.kubernetes.io/control-plane:NoSchedule-`. Not recommended for production.

**Q: Which Kubernetes version does kubeadm install?**
A: By default the same version as the kubeadm binary (so the apt repo minor version decides). Override with `--kubernetes-version`. The kubelet package version should match.

**Q: Can I skip minor versions when upgrading?**
A: No. 1.30 → 1.32 must go 1.30 → 1.31 → 1.32. Patch versions can be skipped. Always read the release notes, back up etcd first.

**Q: What does rke2 install on a node?**
A: The `rke2` binary, systemd units (`rke2-server`/`rke2-agent`), helper scripts (`rke2-killall.sh`, `rke2-uninstall.sh`), and on first start unpacks everything else (kubelet, containerd, etcd, kubectl, crictl) under `/var/lib/rancher/rke2`.

**Q: What is the difference between rke2 and k3s?**
A: Both are by Rancher/SUSE and look similar (single binary, `config.yaml`). **k3s** is lightweight (SQLite default, Traefik, fewer components) for edge and dev. **rke2** is closer to upstream Kubernetes, uses embedded etcd and containerd and focuses on security/compliance (CIS profile, FIPS builds).

**Q: Why does an rke2 agent connect to port 9345?**
A: 9345 is the rke2 *supervisor* port on servers: agents fetch their configuration, certificates and join info there. After registering, the kubelet talks to the API on 6443.

**Q: Where do I get `kubectl` on an rke2 node?**
A: It is installed as `/var/lib/rancher/rke2/bin/kubectl` and not in PATH. Add that directory to PATH and set `KUBECONFIG=/etc/rancher/rke2/rke2.yaml`, or copy the binary to `/usr/local/bin`.

**Q: How do I add a new server (HA) to rke2?**
A: Install as `server`, set the same `token` and `server: https://<first-server>:9345` in its `config.yaml`; embedded etcd joins automatically. Use an odd number (3). Put a DNS name/LB for the API in `tls-san`.

**Q: How do I upgrade rke2?**
A: Re-run the installer with `INSTALL_RKE2_VERSION` / `INSTALL_RKE2_CHANNEL` and restart the service (servers first, one by one; then agents), or automate it with Rancher's system-upgrade-controller. Take an etcd snapshot first (`rke2 etcd-snapshot save`).

**Q: How do I back up an rke2 / kubeadm cluster?**
A: Back up etcd. rke2 takes snapshots automatically (default every 12 h, 5 kept) in `/var/lib/rancher/rke2/server/db/snapshots`; take one manually before changes. kubeadm: `etcdctl snapshot save` using certs from `/etc/kubernetes/pki/etcd`. Keep a copy off the node, and back up `/etc/kubernetes/pki`.

**Q: Should I use kubeadm or rke2 in production?**
A: Both are used. kubeadm gives full control and matches CKA exam knowledge but you own every OS and upgrade detail. rke2 gives a reproducible hardened cluster fast and is easier to automate (Ansible/Terraform). Managed Kubernetes (EKS/GKE/AKS) removes control-plane operations altogether.

**Q: How can I automate these installs?**
A: Both are natural fits for Ansible (lessons 24-29) — package/sysctl/modprobe/template tasks for kubeadm, file template + service for rke2 — and Terraform for the VMs (lessons 20-23).

**Q: What is a `super-admin.conf`?**
A: Added in kubeadm 1.29: a kubeconfig with `system:masters` group (bypasses RBAC) for emergencies. `admin.conf` now uses group `kubeadm:cluster-admins` bound by a ClusterRoleBinding, so its access can be revoked.

**Q: Does `kubeadm reset` remove everything?**
A: It removes kubeadm-managed files and stops the kubelet's workload, but not CNI config (`/etc/cni/net.d`), iptables/IPVS rules, `$HOME/.kube`, or installed packages. Clean those manually.

## Practice / Exercise

### Core

1. **Prerequisite checklist.** On a clean VM, apply every prerequisite from A1-A3 and write a bash script `k8s-prereq.sh` that does it idempotently (running twice produces no errors). Verify with `lsmod`, `sysctl`, `free -h`, `containerd config dump | grep SystemdCgroup`, `kubeadm version`.
2. **Deliberate failures.** Break one prerequisite at a time (turn swap on; `sysctl net.ipv4.ip_forward=0`; set `SystemdCgroup=false`; block 6443 with ufw) and write down the preflight/join error message and the fix.
3. **rke2 cluster.** Build an rke2 cluster of 1 server + 1 agent using `config.yaml` files with `token`, `tls-san`, `node-label`. Show `kubectl get nodes -o wide` and `kubectl get pods -A`.
4. **Use the cluster.** Deploy `nginx` with 3 replicas and a Service, confirm that Pods run on both nodes, and reach the Service from a temporary Pod.
5. **Explore internals.** On the rke2 server list `/var/lib/rancher/rke2/agent/pod-manifests/`, run `crictl ps`, locate `rke2.yaml` and `node-token`. Explain in 3 sentences how each control-plane component is started.
6. **Snapshot.** Take a manual etcd snapshot, delete a Deployment, and explain what a restore would bring back (do not restore).

### Stretch

1. **kubeadm in config file form.** Write a `kubeadm-config.yaml` (InitConfiguration + ClusterConfiguration + KubeletConfiguration) with `controlPlaneEndpoint`, `podSubnet` and `cgroupDriver: systemd`; run `kubeadm init --config ... --dry-run` and list which phases would run.
2. **Certificate lifecycle.** On a kubeadm lab: run `kubeadm certs check-expiration`, change the system clock +366 days with `date -s` (VM only!), observe the failure, reset the clock, renew all certificates and restart the static Pods.
3. **Join automation.** Write a script that, on the control plane, prints a fresh join command and the CA hash computed manually; verify that both hashes match.
4. **CNI swap.** Re-init a kubeadm lab with Calico, apply a default-deny NetworkPolicy from lesson 48 and prove it works; then explain why the same policy has no effect on Flannel.
5. **rke2 variants.** Reinstall rke2 with `cni: calico`, `disable: [rke2-ingress-nginx]`, a `node-taint` on the agent, and apply `profile: cis`; run kube-bench (lesson 49) and compare the number of FAIL checks with a kubeadm cluster.
6. **HA design.** Draw (and justify in writing) a 3-control-plane + 3-worker topology with HAProxy/keepalived in front of 6443: list the ports on the firewall rules between every pair of node types and what `--control-plane-endpoint` would be.
7. **Ansible.** Convert part B (rke2 server + agent) into an Ansible playbook with a role for each node type, reading the token from a variable (do not commit real secrets).

## Further Reading

- Kubernetes docs — Installing kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Kubernetes docs — Creating a cluster with kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- Kubernetes docs — kubeadm init phases and reference: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/
- Kubernetes docs — Certificate management with kubeadm: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/
- Kubernetes docs — Upgrading kubeadm clusters: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
- Kubernetes docs — Ports and protocols: https://kubernetes.io/docs/reference/networking/ports-and-protocols/
- Kubernetes docs — Container runtimes (cgroup driver): https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- Kubernetes docs — Highly available topology options: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/
- RKE2 documentation — Quick start and configuration: https://docs.rke2.io/install/quickstart and https://docs.rke2.io/install/configuration
- RKE2 documentation — CIS hardening guide: https://docs.rke2.io/security/hardening_guide
- Flannel: https://github.com/flannel-io/flannel — Calico: https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises

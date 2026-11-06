# Lesson 37: Services & Networking – ClusterIP, NodePort

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-36 (Kubernetes architecture, kubectl & core resources, Pods/Deployments/manifests). Student can write a Deployment manifest, apply it, scale it and read `kubectl get/describe` output. Lessons 10 (networking essentials) and 32 (Docker networks) help.

## Learning Objectives

By end of lesson student can:
- Explain why Pod IPs are unreliable and what problem a Service solves
- Describe how a Service selects Pods (label selector), and how Endpoints / EndpointSlice objects track them
- Create ClusterIP, NodePort, LoadBalancer, ExternalName and headless Services from YAML and with `kubectl expose`
- Use in-cluster DNS names (`svc.namespace.svc.cluster.local`) and short forms to reach a Service from another Pod
- Explain `port`, `targetPort`, `nodePort` and named ports without mixing them up
- Describe the NodePort range (30000-32767), its limitations and when NodePort is acceptable
- Explain what LoadBalancer does on a cloud and what MetalLB does on bare metal
- Explain what a headless Service returns from DNS and why StatefulSets need it
- Debug a Service with `kubectl port-forward`, `kubectl get endpoints`, and a throw-away debug Pod
- Diagnose "Service has no endpoints", selector mismatch and wrong `targetPort`
- Explain at a high level how kube-proxy implements Services (iptables vs IPVS modes)

## Topics

- Service concepts: stable VIP, DNS name, selector-based targeting, Endpoints object
- ClusterIP: internal communication, kube-dns service discovery (`svc.namespace.svc.cluster.local`)
- NodePort: external access, port ranges (30000-32767), use cases and limitations
- LoadBalancer: cloud provider integration, external IP provisioning; ExternalName
- Headless service: `clusterIP: None`; `kubectl port-forward` for local debugging
- Extras (to over-fill): EndpointSlices, sessionAffinity, multi-port Services, named ports, kube-proxy modes, `externalTrafficPolicy`, troubleshooting playbook

## Concepts

### The problem: Pods are ephemeral

A Pod gets its own IP address from the cluster Pod network (Pod CIDR). That IP lives only as long as the Pod. When a Deployment replaces a Pod (crash, rolling update, node drain, scale down), the new Pod gets a **different IP**. If a frontend hard-coded `10.244.1.17` for the backend, it breaks the first time the backend restarts.

Also, a Deployment runs many replicas. The client needs **one address** that spreads traffic over all healthy replicas. Pod IPs give neither stability nor load balancing.

A **Service** is the Kubernetes answer: a stable virtual IP (VIP) plus a stable DNS name, in front of a changing set of Pods.

### What a Service is

A Service is an API object that defines:
1. A **selector** — a label query that picks the backend Pods
2. A **stable virtual IP** (`ClusterIP`) allocated from the Service CIDR (e.g. `10.96.0.0/12`)
3. A **DNS name** registered in CoreDNS
4. One or more **ports** (what port the Service listens on and where to forward)

Key facts to repeat to students:
- The Service IP is **virtual**. No network interface, no process owns it. `ping` to a ClusterIP usually gets no reply (ICMP is not handled; only the TCP/UDP ports in the Service are implemented).
- The Service IP only exists in kube-proxy rules (iptables/IPVS) on every node. A packet sent to the VIP is rewritten (DNAT) to a real Pod IP.
- A Service is **not** a proxy process and **not** a load balancer appliance by itself. It is a rule set programmed on every node.
- A Service is **namespaced**. Its DNS name includes the namespace.

```
 client Pod ──► my-svc (10.96.34.12:80) ──kube-proxy DNAT──► Pod A 10.244.1.5:8080
                                                       ├──► Pod B 10.244.2.9:8080
                                                       └──► Pod C 10.244.1.7:8080
```

### Selector-based targeting

```yaml
spec:
  selector:
    app: web
```

Every Pod in the **same namespace** with the label `app=web` that is **Ready** becomes a backend. Label the Pod template of the Deployment (`spec.template.metadata.labels`), not just the Deployment metadata. The Service selector must match the **Pod labels**, not the Deployment's own labels and not the Deployment's `spec.selector` by name — it matches whatever labels the Pods actually carry.

Selector rules:
- Equality only (`key: value` pairs, ANDed). Services do **not** support `matchExpressions` or set-based selectors.
- Selector is optional. A Service without a selector gets no automatic Endpoints; you create the Endpoints/EndpointSlice by hand (used to point a Service at an external IP or a database outside the cluster).
- Only **Ready** Pods are put in the endpoint list. A Pod that is Running but not Ready receives no traffic. (How readiness is computed — probes — is covered in lesson 45. Today: "Ready" is shown in the READY column of `kubectl get pods`.)

### Endpoints and EndpointSlices

The control plane runs an **endpoints controller** (part of kube-controller-manager). It watches Services and Pods and maintains:

- **Endpoints** object — same name as the Service, lists `IP:port` of Ready Pods. Legacy but still shown by `kubectl get endpoints`.
- **EndpointSlice** objects — the modern replacement (GA since 1.21). A big Service is split into slices of up to 100 endpoints each, so one Pod change does not rewrite one giant object. kube-proxy reads EndpointSlices.

```bash
kubectl get endpoints web
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl describe svc web       # shows Endpoints: line
```

`<none>` under ENDPOINTS is the single most common Service bug (see Pitfalls). Replicas change → endpoints controller updates the list in seconds → kube-proxy reprograms rules on every node.

### Service types — overview

| Type | Reachable from | Allocates | Typical use |
|---|---|---|---|
| `ClusterIP` (default) | inside cluster only | virtual IP | service-to-service traffic |
| `NodePort` | outside via `<NodeIP>:<nodePort>` | ClusterIP + port 30000-32767 on every node | quick external access, dev, lab, behind your own LB |
| `LoadBalancer` | outside via cloud LB IP | NodePort + ClusterIP + external LB | production exposure on cloud |
| `ExternalName` | inside cluster | no IP; DNS CNAME | alias to an outside hostname |
| Headless (`clusterIP: None`) | inside cluster | no VIP; DNS returns Pod IPs | StatefulSets, client-side balancing, discovery |

Types are **layered**: NodePort includes ClusterIP; LoadBalancer includes NodePort which includes ClusterIP. Delete the outer layer's exposure and the inner still works.

### ClusterIP

Default type. Reachable only from inside the cluster (Pods and nodes). Use for backends, databases, internal APIs.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: demo
spec:
  type: ClusterIP          # default, can be omitted
  selector:
    app: web
  ports:
    - name: http
      protocol: TCP        # TCP default; UDP, SCTP also allowed
      port: 80             # port the Service listens on (VIP:80)
      targetPort: 8080     # port on the Pod container
```

**Three port fields — memorize:**

| Field | Where | Meaning |
|---|---|---|
| `port` | Service | port clients use on the Service VIP / DNS name |
| `targetPort` | Pod | port the container actually listens on. Default = `port`. Can be a **name** or a number |
| `nodePort` | every Node (NodePort/LB only) | port opened on all node IPs. Range 30000-32767 |

Packet path: `client → ClusterIP:port → DNAT → PodIP:targetPort`.

You can set a static ClusterIP with `spec.clusterIP: 10.96.0.50` (must be inside the Service CIDR and unused). Rarely needed; avoid.

### Named ports

Pod declares a port name, Service refers to the name:

```yaml
# Pod template
containers:
  - name: app
    image: nginx:1.27
    ports:
      - name: web
        containerPort: 80
---
# Service
ports:
  - port: 80
    targetPort: web          # resolves per Pod
```

Benefit: Deployment can change the container port number without editing the Service; different Pod versions may even use different numbers.

### Multi-port Services

When a Service exposes several ports, **each port must have a `name`** (lowercase, DNS label style):

```yaml
ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
```

### Service discovery: environment variables and DNS

Two mechanisms exist for finding a Service from a Pod.

1. **Environment variables.** When a Pod starts, kubelet injects `WEB_SERVICE_HOST` and `WEB_SERVICE_PORT` for every Service that already exists in that namespace. Weakness: the Service must exist **before** the Pod. Mostly legacy.
2. **DNS** (preferred). CoreDNS (the cluster DNS, historically "kube-dns", the Service is still named `kube-dns`) resolves Service names to ClusterIPs. No ordering problems.

### DNS names

Full name (FQDN) of a Service:

```
<service>.<namespace>.svc.<cluster-domain>
web.demo.svc.cluster.local
```

`cluster.local` is the default cluster domain.

Pod `/etc/resolv.conf` is auto-generated by kubelet:

```
nameserver 10.96.0.10              # CoreDNS Service ClusterIP
search demo.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Because of the `search` list, short names work:

| From Pod in namespace | Name used | Resolves to |
|---|---|---|
| `demo` | `web` | `web.demo.svc.cluster.local` |
| `demo` | `web.demo` | same |
| `other` | `web.demo` | Service `web` in `demo` |
| any | `web.demo.svc.cluster.local` | FQDN, always works |

Rules:
- Same namespace → just `web`.
- Cross-namespace → at least `web.demo` (service.namespace).
- `ndots:5` means any name with fewer than 5 dots is tried with the search suffixes first. A lookup of `api.example.com` (2 dots) first tries `api.example.com.demo.svc.cluster.local`, etc. — extra DNS queries. Put a trailing dot (`api.example.com.`) to skip the search list. (More on CoreDNS in lesson 38.)
- Port numbers are not in DNS. For SRV lookups, named ports yield SRV records `_http._tcp.web.demo.svc.cluster.local`.

### NodePort

Opens the **same port on every node** in the cluster (default range **30000-32767**, set by kube-apiserver `--service-node-port-range`). Traffic hitting `<any-node-IP>:<nodePort>` is forwarded to the Service and then to a Pod — even if that Pod runs on a different node.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-np
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080        # optional; omit → random free port in range
```

Result: a ClusterIP is also allocated; `kubectl get svc` shows `80:30080/TCP`.

**Use cases:** labs, bare-metal dev, demos, a fixed backend port for an external load balancer you manage (e.g. HAProxy/nginx from lessons 13, pointing at all node IPs).

**Limitations:**
- Port range is high and awkward (no 80/443 without changing API server flags).
- One Service per nodePort; port collisions across Services are not allowed.
- Clients need a node IP; if that node dies the client must switch. No built-in HA address.
- Every node opens the port, increasing attack surface; firewall/security group must allow it.
- Source IP is lost (SNAT) by default — see `externalTrafficPolicy` below.
- Not meant for production exposure on cloud; use LoadBalancer or Ingress (lesson 38).

### externalTrafficPolicy

For NodePort and LoadBalancer Services:

| Value | Behaviour | Pros / cons |
|---|---|---|
| `Cluster` (default) | node forwards to any Pod in cluster, may hop to another node and SNAT | even spread; client IP lost; extra hop |
| `Local` | node forwards only to Pods on **that node**; drops if none | preserves client IP; no extra hop; uneven spread; nodes without Pods fail the LB health check |

```yaml
spec:
  type: NodePort
  externalTrafficPolicy: Local
```

`internalTrafficPolicy: Local` is the in-cluster equivalent (keep traffic on the same node).

### LoadBalancer

`type: LoadBalancer` asks the **cloud provider** (through the cloud-controller-manager) to create an external load balancer (AWS ELB/NLB, GCP LB, Azure LB, Hetzner LB, OpenStack Octavia...) and point it at the node ports. The cloud assigns an **external IP or hostname**, written into `status.loadBalancer.ingress`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  annotations:
    # provider-specific examples, differ per cloud:
    # service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

```
$ kubectl get svc web-lb
NAME     TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
web-lb   LoadBalancer   10.96.12.40    203.0.113.25    80:31544/TCP   2m
```

`EXTERNAL-IP <pending>` forever means **no controller is fulfilling the request** — typical on kubeadm, rke2, kind, minikube without extras.

Notes:
- One cloud LB (and one bill line) **per Service**. With many apps this is expensive → use one Ingress/Gateway (lessons 38-39) in front.
- `spec.loadBalancerIP` is deprecated; use provider annotations. `loadBalancerSourceRanges` restricts client CIDRs.
- `allocateLoadBalancerNodePorts: false` skips node ports for LBs that talk directly to Pods.

**Bare metal: MetalLB.** No cloud → no LB controller. **MetalLB** is an add-on that watches `LoadBalancer` Services and assigns IPs from a pool you configure; it announces them via **L2 mode** (ARP/NDP, one node answers) or **BGP mode** (peers with routers). After installing MetalLB and an `IPAddressPool`, `EXTERNAL-IP` stops being `<pending>`. Concept only today; installation is out of scope. minikube has `minikube tunnel` for the same effect locally.

### ExternalName

Maps a Service name to an **external DNS name** using a CNAME. No ClusterIP, no selector, no proxying.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: demo
spec:
  type: ExternalName
  externalName: mydb.rds.amazonaws.com
```

Pods look up `db.demo.svc.cluster.local` → CoreDNS answers CNAME `mydb.rds.amazonaws.com`. Use for: migrating an external dependency into the cluster later without changing app config; pointing different environments to different external hosts.

Caveats: HTTP `Host` header / TLS SNI will be the **original** Service name, not the external name → certificate mismatch is common. `externalName` must be a hostname, not an IP. No ports are defined or enforced.

### Services without selectors (manual Endpoints)

To front a thing outside the cluster by IP:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-db
spec:
  ports:
    - port: 5432
---
apiVersion: v1
kind: Endpoints
metadata:
  name: legacy-db          # must equal Service name
subsets:
  - addresses:
      - ip: 192.168.10.50
    ports:
      - port: 5432
```

Pods connect to `legacy-db:5432` and land on the external IP.

### Headless Service

Set `clusterIP: None`. Kubernetes allocates **no VIP** and kube-proxy does nothing for it. DNS returns the **A records of all ready Pods** directly instead of one virtual IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector:
    app: db
  ports:
    - port: 5432
```

```bash
# from a debug Pod
nslookup db-headless
# Name: db-headless.demo.svc.cluster.local  Address: 10.244.1.5
# Name: db-headless.demo.svc.cluster.local  Address: 10.244.2.9
```

Why use it:
- Client must know **each** Pod (database replicas, Kafka brokers, etcd, Elasticsearch nodes), e.g. to connect to a primary.
- **StatefulSets** use a headless Service to give each Pod a stable DNS name `pod-0.db-headless.demo.svc.cluster.local` (lesson 41).
- Client-side load balancing (gRPC clients) where the VIP's per-connection balancing is not wanted.

Headless with a selector → DNS A records from endpoints. Headless without selector → DNS from manually created Endpoints (or CNAME if ExternalName-like).

### Session affinity

By default each **new connection** is balanced independently (iptables: random; IPVS: round robin by default). To pin a client to the same Pod:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800     # default 3 hours
```

Only `None` (default) and `ClientIP` exist. It is by source IP, so clients behind a NAT or proxy look like one client. Cookie-based stickiness needs an Ingress/Gateway.

### kube-proxy: how Services are implemented

`kube-proxy` runs on every node (a DaemonSet in most distros). It watches Services and EndpointSlices through the API server and programs the node's packet-forwarding rules. It does **not** carry the traffic itself (in iptables/IPVS modes).

| Mode | How | Notes |
|---|---|---|
| `iptables` (default on Linux) | one chain per Service, one rule per endpoint, random probability selection | simple, robust; rule updates and lookup slow down with thousands of Services (O(n) chains) |
| `ipvs` | kernel IPVS virtual servers, hash table | scales to huge Service counts; real schedulers (rr, lc, sh, ...); needs `ip_vs` kernel modules |
| `nftables` (newer, GA in recent releases) | nftables rules | modern replacement for iptables mode |
| `userspace` | legacy | removed |

Inspect on a node:

```bash
sudo iptables -t nat -L KUBE-SERVICES -n | head
sudo iptables-save | grep web         # chains KUBE-SVC-XXXX / KUBE-SEP-XXXX
sudo ipvsadm -Ln                      # in ipvs mode
kubectl -n kube-system get cm kube-proxy -o yaml | grep -i mode
```

Some CNIs (Cilium) replace kube-proxy completely with eBPF.

Consequence for debugging: because the VIP exists only in NAT rules, you must test **from inside the cluster** (a Pod or a node). Hitting a ClusterIP from your laptop does not work.

### kubectl port-forward

Creates a tunnel from your local machine to a Pod/Service/Deployment through the API server. Works without any Service type, firewall rule or NodePort. Good for debugging; **not** for production exposure (one Pod, dies with your terminal).

```bash
kubectl port-forward pod/web-7d9c-abcde 8080:80           # local 8080 → pod port 80
kubectl port-forward svc/web 8080:80                      # picks ONE backing Pod
kubectl port-forward deploy/web 8080:80
kubectl port-forward --address 0.0.0.0 svc/web 8080:80    # listen on all local interfaces
kubectl port-forward -n demo svc/db 5432                  # local 5432 → 5432
```

Note: `port-forward svc/...` resolves one Pod at start and sticks to it; it does not load-balance and does not survive that Pod's death.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl expose` | Create a Service for an existing workload | `kubectl expose deploy web --port=80 --target-port=8080` |
| `kubectl expose --type` | Choose type | `kubectl expose deploy web --type=NodePort --port=80` |
| `kubectl create service clusterip` | Create Service without workload | `kubectl create service clusterip web --tcp=80:8080` |
| `kubectl create service nodeport` | NodePort imperatively | `kubectl create service nodeport web --tcp=80:8080 --node-port=30080` |
| `kubectl create service externalname` | ExternalName | `kubectl create service externalname db --external-name=db.example.com` |
| `kubectl get svc` | List Services | `kubectl get svc -n demo -o wide` |
| `kubectl get svc -A` | Services in all namespaces | `kubectl get svc -A` |
| `kubectl describe svc` | Selector, ports, endpoints, events | `kubectl describe svc web` |
| `kubectl get endpoints` | Backend IP:port list | `kubectl get endpoints web` |
| `kubectl get endpointslices` | Modern endpoints | `kubectl get endpointslices -l kubernetes.io/service-name=web` |
| `kubectl get svc -o yaml` | Full object | `kubectl get svc web -o yaml` |
| `kubectl get pods -l` | Check selector matches Pods | `kubectl get pods -l app=web -o wide` |
| `kubectl get pods --show-labels` | See Pod labels | `kubectl get pods --show-labels` |
| `kubectl port-forward` | Local tunnel | `kubectl port-forward svc/web 8080:80` |
| `kubectl run --rm -it` | Throw-away debug Pod | `kubectl run dbg --rm -it --image=busybox:1.36 -- sh` |
| `kubectl exec` | Run inside Pod | `kubectl exec -it web-xxx -- curl -s localhost:8080` |
| `kubectl get svc -o jsonpath` | Extract NodePort | `kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}'` |
| `kubectl get nodes -o wide` | Find node IPs | `kubectl get nodes -o wide` |
| `kubectl patch svc` | Change type/policy | `kubectl patch svc web -p '{"spec":{"type":"NodePort"}}'` |
| `kubectl delete svc` | Remove Service | `kubectl delete svc web` |
| `nslookup` / `dig` (in Pod) | Test DNS | `nslookup web.demo.svc.cluster.local` |
| `kubectl get cm -n kube-system coredns -o yaml` | View CoreDNS config | see lesson 38 |

`kubectl expose` copies the selector from the target resource's Pod labels, so it will not have a selector mismatch — which is why it is handy for first tests. Manifests are what you commit to Git.

## Examples / Walkthrough

Assume a working cluster (kubeadm/rke2/kind/minikube) and `kubectl` configured.

### 1. Deploy a backend and watch Pod IPs change

```bash
kubectl create namespace demo
kubectl config set-context --current --namespace=demo

kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl get pods -o wide --show-labels
# NAME            READY  STATUS   IP            NODE    LABELS
# web-6f7-aaaa    1/1    Running  10.244.1.5    node-a  app=web,pod-template-hash=6f7
# web-6f7-bbbb    1/1    Running  10.244.2.9    node-b  app=web,...
# web-6f7-cccc    1/1    Running  10.244.1.7    node-a  app=web,...

kubectl delete pod web-6f7-aaaa
kubectl get pods -o wide      # replacement Pod has a NEW IP
```

Point: Pod IPs are disposable.

### 2. ClusterIP Service from YAML

```yaml
# svc-web.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: demo
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
```

```bash
kubectl apply -f svc-web.yaml
kubectl get svc web
# NAME  TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)  AGE
# web   ClusterIP  10.96.34.12    <none>       80/TCP   5s

kubectl get endpoints web
# NAME  ENDPOINTS                                   AGE
# web   10.244.1.7:80,10.244.2.9:80,10.244.3.4:80   10s

kubectl describe svc web
```

Read `describe` output: **Selector**, **IP**, **Port / TargetPort**, **Endpoints**. Teach students to check these four first.

### 3. Call it from another Pod; DNS names

```bash
kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- sh
/ # nslookup web
/ # nslookup web.demo.svc.cluster.local
/ # wget -qO- http://web
/ # wget -qO- http://web.demo
/ # cat /etc/resolv.conf
/ # exit
```

Run a second debug Pod in another namespace to show the short name `web` fails but `web.demo` works:

```bash
kubectl run dbg2 -n default --rm -it --image=busybox:1.36 --restart=Never -- sh -c 'wget -qO- -T 3 http://web || echo FAIL-short; wget -qO- -T 3 http://web.demo | head -3'
```

### 4. Prove load balancing

Make replicas distinguishable:

```bash
for p in $(kubectl get pods -l app=web -o name); do
  kubectl exec $p -- sh -c 'echo "I am $(hostname)" > /usr/share/nginx/html/index.html'
done

kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'for i in $(seq 1 12); do wget -qO- http://web; done'
# I am web-6f7-bbbb
# I am web-6f7-cccc
# ...
```

Output mixes the three hostnames — connection-level random/round-robin balancing. (`wget` opens a new connection each time, so it is balanced; a keep-alive client pins to one Pod per connection.)

### 5. Scale and see endpoints follow

```bash
kubectl scale deploy web --replicas=5
kubectl get endpoints web          # 5 addresses
kubectl scale deploy web --replicas=1
kubectl get endpoints web          # 1 address
kubectl scale deploy web --replicas=3
```

### 6. Service with targetPort different from port

```bash
kubectl create deployment api --image=hashicorp/http-echo:1.0 -- /http-echo -text=hello-api -listen=:5678
kubectl expose deploy api --port=80 --target-port=5678
kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://api
```

Port 80 on the Service → 5678 on the container.

### 7. NodePort

```bash
kubectl expose deploy web --name=web-np --type=NodePort --port=80
kubectl get svc web-np
# web-np  NodePort  10.96.88.3  <none>  80:31872/TCP

kubectl get nodes -o wide                       # INTERNAL-IP / EXTERNAL-IP
NODEPORT=$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')
curl http://<ANY-NODE-IP>:$NODEPORT             # works on every node, even nodes with no web Pod
```

Demonstrate with a node that has no `web` Pod: still answers (kube-proxy forwards across nodes). Then:

```bash
kubectl patch svc web-np -p '{"spec":{"externalTrafficPolicy":"Local"}}'
curl --max-time 3 http://<NODE-WITHOUT-POD>:$NODEPORT      # now times out
curl http://<NODE-WITH-POD>:$NODEPORT                       # works
```

Local nodes: firewall must allow the nodePort (`ufw allow 31872/tcp`, cloud security group), referring back to lesson 11.

### 8. Fixed nodePort via YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-fixed
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Try `nodePort: 80` → API rejects: `provided port is not in the valid range. The range of valid ports is 30000-32767`.

### 9. LoadBalancer on a lab cluster

```bash
kubectl expose deploy web --name=web-lb --type=LoadBalancer --port=80
kubectl get svc web-lb -w
# EXTERNAL-IP  <pending>      ← no cloud controller / no MetalLB
```

On a cloud cluster the column fills with an IP/hostname in 1-3 minutes. On minikube run `minikube tunnel` in another terminal. Explain MetalLB as the bare-metal fix.

### 10. ExternalName

```bash
kubectl create service externalname ext-google --external-name=www.google.com
kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- nslookup ext-google
# ext-google.demo.svc.cluster.local  canonical name = www.google.com
```

### 11. Headless Service

```bash
kubectl apply -f - <<'EOF'
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
EOF

kubectl get svc web-headless
# CLUSTER-IP  None
kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- nslookup web-headless
# returns one A record per Pod, compare with: nslookup web   (returns one ClusterIP)
```

### 12. port-forward

```bash
kubectl port-forward svc/web 8080:80 &
curl -s localhost:8080 | head -3
kill %1
```

Show that it works even for a ClusterIP Service, from the laptop — it tunnels through the API server, not through the VIP.

### 13. Break it on purpose: selector mismatch

```bash
kubectl create deployment shop --image=nginx:1.27
kubectl create service clusterip shop --tcp=80:80     # selector defaults to app=shop — works
kubectl get endpoints shop                             # has IPs

kubectl patch svc shop -p '{"spec":{"selector":{"app":"shpo"}}}'    # typo
kubectl get endpoints shop
# ENDPOINTS  <none>
kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://shop
# wget: download timed out   (Service exists, nothing behind it)

kubectl get pods --show-labels                 # compare labels to Service selector
kubectl patch svc shop -p '{"spec":{"selector":{"app":"shop"}}}'
```

### 14. Break it: wrong targetPort

```bash
kubectl patch svc web -p '{"spec":{"ports":[{"port":80,"targetPort":8081}]}}'
kubectl get endpoints web               # still lists IPs (:8081) — endpoints exist
kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://web
# wget: can't connect to remote host: Connection refused
```

Lesson: endpoints present but connection refused → port wrong, not selector.

### 15. Look at kube-proxy rules (on a node)

```bash
sudo iptables-save -t nat | grep -E 'KUBE-SVC|default/web|demo/web' | head
sudo iptables -t nat -L KUBE-SERVICES -n | grep web
```

Show the `statistic mode random probability 0.33333` lines — that is the load balancing.

### Cleanup

```bash
kubectl delete namespace demo
kubectl config set-context --current --namespace=default
```

## Common Pitfalls

- **`ENDPOINTS <none>`: Service selector does not match Pod labels** — most frequent Service bug. Compare `kubectl get svc X -o yaml | grep -A3 selector` with `kubectl get pods --show-labels`. Remember the label must be on the **Pod template** (`spec.template.metadata.labels`).
- **Selector matches, still no endpoints: Pods not Ready** — endpoints only list Ready Pods. Check READY column and `kubectl describe pod`. Readiness probes: lesson 45.
- **Wrong `targetPort`** — endpoints exist but connections are refused/time out. `targetPort` must be the port the app listens on inside the container, not the Service `port`.
- **App listens on `127.0.0.1` only** — traffic arrives on the Pod IP, not loopback, so it is refused. Bind to `0.0.0.0`.
- **Service in a different namespace than Pods** — selector only looks in the Service's own namespace.
- **Using short name across namespaces** — `web` resolves only in the same namespace. Use `web.demo`.
- **Testing a ClusterIP from the laptop or with `ping`** — VIP exists only in-cluster NAT rules and ICMP is not implemented. Test with `curl` from a Pod, or use `port-forward`.
- **`nodePort` outside 30000-32767** — rejected by API server. Do not fight it; use LB or Ingress for 80/443.
- **NodePort works on some nodes only / hangs** — firewall or security group blocks the port, or `externalTrafficPolicy: Local` and the node has no Pod.
- **`EXTERNAL-IP <pending>` forever** — no cloud controller or MetalLB. Not a bug in the Service.
- **One LoadBalancer per app on cloud** — cost explosion. Consolidate behind Ingress/Gateway.
- **Multi-port Service without `name`** — API rejects: `name` required when more than one port.
- **Port name mismatch** — `targetPort: web` but Pod port is named `http` → no match, connection refused.
- **ExternalName used with an IP** — must be a hostname; and TLS/Host header mismatch with the original name.
- **Expecting `port-forward` to load balance** — it picks one Pod and stays on it.
- **Editing `spec.clusterIP`** — immutable after creation (except None → none). Delete and recreate.
- **Relying on env var discovery** — only set for Services existing at Pod start. Use DNS.
- **Long lookups for external names** — `ndots:5` adds search-domain attempts. Use trailing dot or lower `ndots` via Pod `dnsConfig` (lesson 38).
- **Deleting the Deployment but leaving the Service** — Service stays with zero endpoints; clean up both.

## FAQ

**Q: Why do I need a Service if I can use the Pod IP?**
A: Pod IPs change every time a Pod is recreated, and a Deployment has many Pods. A Service gives one stable IP and DNS name and spreads traffic over all Ready Pods.

**Q: Is a Service a pod or a process?**
A: No. It is an API object. Its behaviour is implemented by kube-proxy rules (iptables/IPVS) on every node. There is no process listening on the ClusterIP.

**Q: Why can't I ping a ClusterIP?**
A: The VIP only implements the TCP/UDP ports declared in the Service through NAT rules. ICMP is not translated, so pings get no answer. Use `curl`/`nc` on the Service port.

**Q: What is the difference between `port`, `targetPort` and `nodePort`?**
A: `port` is where the Service listens, `targetPort` is the container port traffic is forwarded to, `nodePort` is the port opened on every node for NodePort/LoadBalancer Services (30000-32767).

**Q: Which Service type should I use?**
A: `ClusterIP` for internal services (most of them). `NodePort` for labs or when you front the cluster with your own load balancer. `LoadBalancer` for direct exposure on a cloud. For HTTP(S) with many apps, one Ingress/Gateway in front of ClusterIP Services (lessons 38-39).

**Q: What is the DNS name of a Service?**
A: `<service>.<namespace>.svc.cluster.local`. Inside the same namespace you can use just `<service>`; cross-namespace use `<service>.<namespace>`.

**Q: Why does my Service show no endpoints?**
A: Either the selector matches no Pods (label mismatch/typo, wrong namespace) or the matching Pods are not Ready. Run `kubectl get endpoints`, `kubectl get pods --show-labels`, `kubectl describe svc`.

**Q: What is the difference between Endpoints and EndpointSlice?**
A: Same information (backend Pod IP:port). Endpoints is one object per Service and does not scale; EndpointSlices split the list into chunks of up to 100 and are what kube-proxy uses today. `kubectl get endpoints` still works.

**Q: Can I change the NodePort range?**
A: Yes, with the kube-apiserver flag `--service-node-port-range=20000-40000`. On managed clusters you usually cannot. Not recommended; use LoadBalancer/Ingress for standard ports.

**Q: How does a Service load balance?**
A: kube-proxy rules pick a backend per new connection: random in iptables mode, round robin by default in IPVS mode. It is L4 (connection-level), not per HTTP request; keep-alive connections stick to one Pod.

**Q: How do I keep a client on the same Pod?**
A: `sessionAffinity: ClientIP` (timeout default 10800s). For cookie-based stickiness use an Ingress/Gateway feature.

**Q: What does `clusterIP: None` do?**
A: Makes the Service headless: no VIP, no kube-proxy rules. DNS returns the individual Pod IPs. Needed by StatefulSets and for client-side discovery.

**Q: What is ExternalName for?**
A: A CNAME inside the cluster DNS pointing to an external hostname, so apps use a stable internal name for an outside dependency. No proxying, no ports.

**Q: Why is LoadBalancer EXTERNAL-IP stuck at `<pending>`?**
A: There is no controller to create the load balancer. On cloud, the cloud-controller-manager must be configured; on bare metal install MetalLB; on minikube run `minikube tunnel`.

**Q: What is MetalLB?**
A: An add-on that implements `type: LoadBalancer` on bare-metal clusters by assigning IPs from a pool and announcing them via ARP (L2) or BGP.

**Q: Does a NodePort Service also have a ClusterIP?**
A: Yes. Types are layered: NodePort adds node ports on top of ClusterIP; LoadBalancer adds a cloud LB on top of NodePort.

**Q: How do I get the client's real source IP?**
A: Set `externalTrafficPolicy: Local` (and make sure the LB health-checks nodes), or use an L7 proxy that adds `X-Forwarded-For`/PROXY protocol. Default `Cluster` SNATs the source.

**Q: Which port does `kubectl port-forward svc/web 8080:80` use?**
A: Local port 8080 → Service port 80, which is mapped to a single backing Pod's `targetPort`. The tunnel goes through the API server and does not require NodePort or firewall changes.

**Q: Can two Services have the same selector?**
A: Yes. Many Services can target the same Pods (for example a ClusterIP and a headless one for the same StatefulSet).

**Q: Do Services work across namespaces?**
A: A Service only selects Pods in its own namespace, but any Pod can call any Service by its namespaced DNS name (unless restricted by NetworkPolicy — lesson 48).

**Q: How can I test DNS inside the cluster?**
A: `kubectl run dbg --rm -it --image=busybox:1.36 --restart=Never -- nslookup web.demo` (or an image with `dig`/`curl` like `nicolaka/netshoot`).

**Q: What is kube-dns vs CoreDNS?**
A: The DNS Service is still called `kube-dns` for compatibility; the software behind it is CoreDNS (lesson 38 covers it).

**Q: iptables or IPVS mode — which is better?**
A: iptables is the default and fine for most clusters. IPVS scales better with thousands of Services and offers more balancing algorithms. Check with `kubectl -n kube-system get cm kube-proxy -o yaml | grep mode`.

**Q: Does the Service IP change?**
A: Not for the life of the Service. If you delete and recreate the Service you may get a different one; the DNS name stays the same, so always use DNS.

## Practice / Exercise

### Core

1. Create namespace `lab37`; set it as the current namespace. Deploy `nginx:1.27` as Deployment `web` with 3 replicas.
2. Write `svc-web.yaml` (ClusterIP, port 80 → targetPort 80, port named `http`). Apply it. Record ClusterIP and endpoint list; confirm endpoints match `kubectl get pods -o wide`.
3. Run a debug Pod; fetch the Service by: short name, `web.lab37`, and the full FQDN. Read `/etc/resolv.conf` and explain each line.
4. Make each replica return its hostname in `index.html`; call the Service 15 times and count distribution.
5. Scale to 5 then 2 replicas; watch `kubectl get endpoints web -w` in a second terminal.
6. Create a NodePort Service `web-np` (fixed nodePort 30090). Reach it from your laptop via two different node IPs. Open the firewall if needed.
7. Break the selector (typo), observe `<none>` endpoints and the timeout, then repair it. Write the 3-command diagnosis chain you used.
8. Break `targetPort`, observe "Connection refused" while endpoints exist, repair it. Explain why the symptom differs from exercise 7.
9. Create a headless Service for the same Pods; compare `nslookup` of the normal and headless Services.
10. Use `kubectl port-forward svc/web 9000:80` and `curl localhost:9000`. Delete the Pod it connected to and observe what happens to the forward.
11. Clean up with `kubectl delete ns lab37`.

### Stretch

1. Write one manifest file that contains a Deployment `app` and a multi-port Service (`http` 80→8080 and `metrics` 9090→9090) using **named** ports in the Pod template. Change the container port number in the Deployment and show the Service still works without edit.
2. Create an ExternalName Service `search` → `www.google.com`; resolve it from a Pod and explain why `curl http://search` may fail TLS/Host checks.
3. Create a selector-less Service plus a manual Endpoints object pointing at an IP outside the cluster (e.g. a VM running nginx); call it from a Pod.
4. Set `externalTrafficPolicy: Local`; prove that nodes without Pods stop answering and that the client IP is preserved (use an `nginx` access log or `http-echo` variant that prints the remote address).
5. Enable `sessionAffinity: ClientIP` and show a single client always hits the same Pod while a second client (another debug Pod) may hit a different one.
6. On a node, find the iptables chains for your Service (`KUBE-SVC-*`, `KUBE-SEP-*`), map each `KUBE-SEP` to a Pod IP, and explain the probability values. Find out whether your cluster runs iptables or IPVS mode.
7. Install MetalLB on a lab cluster (kind/kubeadm VM), create an `IPAddressPool`, and get a real `EXTERNAL-IP` for a LoadBalancer Service. (Own research; installation guide in MetalLB docs.)
8. Write a short troubleshooting runbook (flowchart or checklist) "Service unreachable": DNS → Service exists → selector → endpoints → Ready → targetPort → app bind address → node firewall.

## Further Reading

- Kubernetes docs – Service: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes docs – EndpointSlices: https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- Kubernetes docs – DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Kubernetes docs – Connecting Applications with Services: https://kubernetes.io/docs/tutorials/services/connect-applications-service/
- Kubernetes docs – Virtual IPs and Service Proxies (kube-proxy modes): https://kubernetes.io/docs/reference/networking/virtual-ips/
- Kubernetes docs – Use Port Forwarding to Access Applications: https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/
- Kubernetes docs – Debug Services: https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/
- MetalLB: https://metallb.io/

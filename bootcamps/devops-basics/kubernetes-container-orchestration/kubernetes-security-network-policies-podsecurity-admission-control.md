# Lesson 48: Kubernetes Security – Network Policies, PodSecurity & Admission Control

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-47 (architecture, kubectl, Pods/Deployments, Services, Ingress, storage, ConfigMaps/Secrets, probes, scheduling, RBAC). Student can write Deployment/Service YAML, use labels and selectors, create namespaces, and run temporary debug Pods with `kubectl run`.

## Learning Objectives

By end of lesson student can:
- Explain why a default Kubernetes cluster is a flat, fully open network and why that is a security risk
- Name which CNI plugins enforce NetworkPolicy (Calico, Cilium, Antrea, Weave) and which do not (plain Flannel)
- Write a NetworkPolicy using `podSelector`, `policyTypes`, `ingress.from`, `egress.to`, `namespaceSelector`, `ipBlock`, `ports`, `endPort`
- Predict the result of combining several policies (additive, allow-only) and avoid the `namespaceSelector` AND vs OR trap
- Apply the default-deny pattern for ingress and egress, then allow DNS and selected traffic
- Build and test a 3-tier policy set (frontend → backend → database) with temporary Pods
- Enforce Pod Security Standards with PodSecurity Admission namespace labels (`enforce`, `warn`, `audit`) and explain `privileged`, `baseline`, `restricted`
- Use server-side dry-run to preview what a label change would block
- Describe the API request path: authentication → authorization → mutating admission → schema validation → validating admission → etcd
- List the built-in admission controllers and configure `--enable-admission-plugins`
- Read and write a `ValidatingWebhookConfiguration` / `MutatingWebhookConfiguration` skeleton and explain `failurePolicy`, `timeoutSeconds`, `namespaceSelector`

## Topics

- Network Policies: podSelector, policyTypes (Ingress/Egress), namespaceSelector, ipBlock
- Default deny pattern: deny-all ingress + egress; allow selectively; test connectivity
- PodSecurity Admission: profiles (privileged/baseline/restricted), modes (enforce/warn/audit)
- Admission controllers: built-in (LimitRanger, ResourceQuota, NodeRestriction, NamespaceLifecycle)
- ValidatingWebhookConfiguration: concept, how webhooks intercept API calls; common examples

## Concepts

### The flat network problem

By default every Pod in a Kubernetes cluster can reach every other Pod, in every namespace, on every port, by IP. There is no firewall between Pods. Services and DNS make discovery easy, but they are not access control: if an attacker compromises one low-value Pod (for example a public web frontend), they can scan and connect to the database Pod, the internal admin API, the monitoring stack, and the kube-system components.

Kubernetes fixes this with **NetworkPolicy**: a namespaced API object that describes which traffic is allowed to or from a set of Pods. Think of it as a distributed, label-driven firewall. Instead of IP addresses (which change every time a Pod restarts) you select Pods by labels.

Three rules to remember before anything else:

1. **A NetworkPolicy does nothing unless the CNI plugin enforces it.** The API server accepts the object happily even when nothing implements it.
2. **Pods are open until a policy selects them.** A Pod that no NetworkPolicy selects accepts all traffic. Once at least one policy selects the Pod for a direction (Ingress or Egress), only traffic allowed by the union of those policies is permitted in that direction.
3. **Policies are allow-only and additive.** There is no "deny" rule and no rule ordering. The final allowed set is the union of every policy that selects the Pod. You deny by selecting a Pod and not allowing something.

### CNI support — who enforces NetworkPolicy

The Container Network Interface (CNI) plugin builds the Pod network. Enforcement of NetworkPolicy is a feature of the plugin, not of Kubernetes itself.

| CNI | Enforces NetworkPolicy | Notes |
|---|---|---|
| Calico | Yes | iptables or eBPF dataplane; also has its own richer `GlobalNetworkPolicy` CRD |
| Cilium | Yes | eBPF; supports L7 (HTTP, DNS/FQDN) via `CiliumNetworkPolicy` |
| Antrea | Yes | Open vSwitch based |
| Weave Net | Yes | Older, less used today |
| Flannel (plain) | **No** | Only provides connectivity. Policies are silently ignored |
| kindnet (kind default) | Recent versions yes | Older kind clusters: no |
| minikube default | Depends | Start with `minikube start --cni=calico` (or `--cni=cilium`) for labs |
| RKE2 | Canal (Flannel + Calico policy) by default | Policy works; Cilium/Calico selectable in `config.yaml` |
| EKS (VPC CNI) | Yes in recent versions, must enable the network policy agent | Check cloud docs |

How to check what you run:

```bash
kubectl get pods -n kube-system -o wide | grep -E 'calico|cilium|flannel|antrea|weave|canal'
kubectl get ds -n kube-system
```

Quick enforcement test (no extra tools): create a default-deny policy in a test namespace and check that a connection that worked before now times out. If it still works, the CNI does not enforce policies.

### Anatomy of a NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: shop
spec:
  podSelector:              # WHICH Pods this policy protects (the "target")
    matchLabels:
      app: backend
  policyTypes:              # WHICH directions this policy governs
    - Ingress
  ingress:                  # list of allow rules (OR between rules)
    - from:                 # list of peers (OR between peers)
        - podSelector:
            matchLabels:
              app: frontend
      ports:                # list of ports (OR between ports)
        - protocol: TCP
          port: 8080
```

Field by field:

- `metadata.namespace` — the policy only applies to Pods **in its own namespace**. Always set it.
- `spec.podSelector` — selects the Pods the policy applies to. `podSelector: {}` (empty) selects **all Pods in the namespace**.
- `spec.policyTypes` — `Ingress`, `Egress` or both. If omitted, Kubernetes infers: `Ingress` is always included; `Egress` is included only if the policy has an `egress:` section. **Always write `policyTypes` explicitly** — the inference is a classic source of mistakes (a policy with only `egress` rules and no `policyTypes` still turns on Ingress isolation).
- `spec.ingress[].from[]` — who may connect **to** the selected Pods.
- `spec.egress[].to[]` — where the selected Pods may connect **to**.
- `ports[]` — `protocol` (TCP default, UDP, SCTP), `port` (number or named port), optional `endPort` for a range.

### The four peer selector types

A peer inside `from`/`to` is one of:

| Selector | Matches |
|---|---|
| `podSelector` | Pods in the **same namespace as the policy** with these labels |
| `namespaceSelector` | **All Pods** in namespaces whose labels match |
| `podSelector` + `namespaceSelector` in the **same list item** | Pods with the label **in** namespaces with the label (AND) |
| `ipBlock` | A CIDR range, with optional `except`. Meant for traffic outside the cluster; Pod IPs are ephemeral so do not use it for in-cluster Pod peers |

#### The AND vs OR trap (most common interview and production bug)

Two YAML snippets that differ by one dash:

```yaml
# AND — one peer: Pods labeled role=client that live in namespaces labeled team=a
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            team: a
        podSelector:
          matchLabels:
            role: client
```

```yaml
# OR — two peers: ALL pods in namespaces team=a  OR  Pods role=client in the policy's own namespace
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            team: a
      - podSelector:
          matchLabels:
            role: client
```

Memory aid: **one dash = one peer = AND; two dashes = two peers = OR.**

Another gotcha: namespaces have no useful labels by default except `kubernetes.io/metadata.name=<name>` (set automatically since Kubernetes 1.21). Use it to select a namespace by name without labeling it by hand:

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: monitoring
```

### Ports and ranges

```yaml
ports:
  - protocol: TCP
    port: 8000
    endPort: 8999      # allow 8000-8999 (endPort requires port, must be >= port)
  - protocol: UDP
    port: 53
  - port: http           # a named port from the target container spec
```

If `ports` is omitted from a rule, **all ports** are allowed for that peer. If `from`/`to` is omitted but `ports` is present, those ports are allowed from **any** source.

### Empty-vs-missing semantics cheat sheet

| Policy snippet | Effect |
|---|---|
| `ingress: []` (or no `ingress` key) with `policyTypes: [Ingress]` | Deny all ingress to selected Pods |
| `ingress: [{}]` (one empty rule) | Allow all ingress to selected Pods |
| `from: [{podSelector: {}}]` | Allow from all Pods in the policy's namespace |
| `from: [{namespaceSelector: {}}]` | Allow from all Pods in all namespaces |
| `podSelector: {}` at the top | Target = every Pod in the namespace |
| `egress: [{}]` | Allow all egress |
| `ipBlock: {cidr: 0.0.0.0/0}` | Allow everything outside as well (including the internet and often the cloud metadata IP) |

### Default-deny pattern

Zero-trust starting point per namespace: deny everything, then allow only what the application needs.

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```yaml
# deny-all-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

```yaml
# deny-all.yaml — both directions in one object
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**After egress deny, DNS breaks.** Pods can no longer reach CoreDNS, so name resolution fails and everything *looks* broken. Always add an allow-DNS policy right after the egress deny:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns          # CoreDNS Pods keep this label
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Note the AND form here (one dash): CoreDNS Pods in `kube-system` only.

### What NetworkPolicy does NOT do

- No L7 filtering (HTTP paths, methods, hostnames). Plain NetworkPolicy is L3/L4 only. Cilium/Calico CRDs and service meshes add L7.
- No logging of denied traffic in the standard API.
- No FQDN rules ("allow to api.github.com"). Use `ipBlock` or CNI-specific CRDs.
- No cluster-wide (non-namespaced) policy in the core API. Apply the default-deny object in every namespace (Helm/Kustomize/ GitOps loops, or CNI CRDs such as `GlobalNetworkPolicy`).
- Traffic from the **node itself** to local Pods (kubelet health checks) is generally permitted by CNIs regardless of policy, so probes keep working. Behaviour is CNI-dependent.
- Return traffic of an allowed connection is allowed automatically (stateful). You do not need an egress rule for replies to an allowed ingress connection.
- Policies apply to Pod traffic, not to `hostNetwork: true` Pods (behaviour undefined / often unfiltered).

### Three-tier example architecture

```
internet → [ingress controller ns: ingress-nginx]
              |
              v
          frontend (app=frontend, :80)
              |
              v
          backend  (app=backend,  :8080)
              |
              v
          database (app=database, :5432)
```

Allowed flows only:

1. ingress-nginx namespace → frontend :80
2. frontend → backend :8080
3. backend → database :5432
4. Everyone → CoreDNS :53 (egress)

Everything else (frontend → database, database → anywhere, backend → internet) is denied.

### Pod Security Standards (PSS) and PodSecurity Admission (PSA)

Containers can request dangerous things: `privileged: true`, host namespaces (`hostNetwork`, `hostPID`, `hostIPC`), host paths, extra Linux capabilities, running as root. A Pod with those can often take over the node. Kubernetes defines three **Pod Security Standards** — policy levels:

| Profile | Meaning | Typical use |
|---|---|---|
| `privileged` | Unrestricted. No checks. | kube-system, CNI, storage drivers, node agents |
| `baseline` | Blocks known privilege escalations; allows default Pod specs with minimal tweaks | Default for most namespaces |
| `restricted` | Current Pod hardening best practice; strict | Security-sensitive workloads, target state |

What each profile forbids (summary — the official docs list every field):

**Baseline forbids:**
- `hostProcess` Windows containers
- `hostNetwork`, `hostPID`, `hostIPC`
- `privileged: true`
- Adding capabilities beyond a safe default list (for example `NET_ADMIN`, `SYS_ADMIN` are blocked; `NET_BIND_SERVICE`, `CHOWN`, `KILL` etc. allowed)
- `hostPath` volumes
- `hostPort` use
- Disabling or weakening AppArmor, custom SELinux options (types other than the allowed ones), unmasked `/proc`
- Seccomp profile `Unconfined`
- Unsafe sysctls (only a short safe list is allowed)

**Restricted = baseline plus:**
- Volume types limited to: `configMap`, `csi`, `downwardAPI`, `emptyDir`, `ephemeral`, `persistentVolumeClaim`, `projected`, `secret`
- `allowPrivilegeEscalation` must be **false**
- Containers must run as non-root: `runAsNonRoot: true` (and no `runAsUser: 0`)
- Seccomp profile must be set to `RuntimeDefault` or `Localhost` (what seccomp is: lesson 49)
- Capabilities: must **drop `ALL`**, may only add `NET_BIND_SERVICE`

A Pod that passes `restricted`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault        # required by restricted; details in lesson 49
  containers:
    - name: app
      image: nginxinc/nginx-unprivileged:1.27
      ports:
        - containerPort: 8080
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

### PodSecurity Admission — enforcing the standards with namespace labels

**PodSecurity Admission** is a built-in admission controller (stable, enabled by default since Kubernetes 1.25; it replaced the removed PodSecurityPolicy). It is configured per namespace with **labels**:

```
pod-security.kubernetes.io/<MODE>=<LEVEL>
pod-security.kubernetes.io/<MODE>-version=<VERSION>
```

Modes:

| Mode | Behaviour on violation |
|---|---|
| `enforce` | Pod is **rejected** |
| `warn` | Pod is accepted; user gets a **warning** in kubectl output |
| `audit` | Pod is accepted; violation is added as an annotation to the audit event (audit logs are covered in lesson 50) |

Levels: `privileged`, `baseline`, `restricted`.
Versions: `latest` (default) or a specific minor like `v1.30`, pins the checks to that Kubernetes release so cluster upgrades do not suddenly tighten rules.

You can set several modes at once with different levels — the standard rollout approach:

```bash
kubectl label namespace shop \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

Meaning: block anything below baseline; tell users when they would fail restricted; log it.

Important behaviours:

- PSA checks **Pods** (and the Pod template inside Deployments, Jobs, etc. in `warn`/`audit` mode). In `enforce` mode only the actual Pod object is rejected — so a Deployment is *created* fine, but its ReplicaSet fails to create Pods (look at `kubectl describe rs` events, not the Deployment).
- Existing running Pods are **not** evicted when you add an enforce label. Only new Pod creations are checked.
- **Dry-run label change** shows which existing Pods would violate:
  ```bash
  kubectl label --dry-run=server --overwrite namespace shop \
    pod-security.kubernetes.io/enforce=restricted
  ```
  The server prints warnings listing offending Pods.
- **Exemptions** are configured in the API server's `AdmissionConfiguration` (not in the namespace): `usernames`, `runtimeClasses`, `namespaces` (for example `kube-system`). Cloud/managed clusters often hide this.
- Cluster-wide defaults also come from `AdmissionConfiguration`:
  ```yaml
  apiVersion: apiserver.config.k8s.io/v1
  kind: AdmissionConfiguration
  plugins:
    - name: PodSecurity
      configuration:
        apiVersion: pod-security.admission.config.k8s.io/v1
        kind: PodSecurityConfiguration
        defaults:
          enforce: baseline
          enforce-version: latest
          warn: restricted
          audit: restricted
        exemptions:
          usernames: []
          runtimeClasses: []
          namespaces: [kube-system]
  ```
- PSA is coarse: three fixed levels, no custom rules. For custom policy ("only registry X", "require labels", "no :latest") use `ValidatingAdmissionPolicy` (CEL, built-in since 1.30) or a policy engine (Kyverno, Gatekeeper — Kyverno intro in lesson 50).

### Admission control — the API request path

Every request that creates, changes, or deletes an object travels through the API server in this order:

```
kubectl / client
   |
   v
1. Authentication      who are you?                (certs, tokens, OIDC)       lesson 47
2. Authorization       may you do this?            (RBAC)                      lesson 47
3. Mutating admission  may CHANGE the object       (defaults, injection)
4. Object schema validation
5. Validating admission may only ACCEPT or REJECT
   |
   v
6. Persist to etcd
```

Authentication and authorization decide *who may call the API*. **Admission controllers** decide *whether the content of the request is acceptable* and can modify it. They only run for write operations (create/update/delete/connect), never for reads (get/list/watch).

Two kinds:

- **Mutating** — can modify the object (set default StorageClass, inject a sidecar, add labels). Run first, in order. Mutating webhooks may be called more than once (re-invocation) if another mutator changed the object.
- **Validating** — can only allow or deny. Run after all mutations so they see the final object. Run in parallel.

A rejection at any step stops the request with an error like:

```
Error from server (Forbidden): pods "bad" is forbidden: violates PodSecurity "restricted:latest": ...
```

### Built-in admission controllers

Compiled into `kube-apiserver`. Enabled/disabled with flags:

```bash
kube-apiserver --enable-admission-plugins=NodeRestriction,PodSecurity,... \
               --disable-admission-plugins=...
```

On kubeadm clusters edit `/etc/kubernetes/manifests/kube-apiserver.yaml` (static Pod; kubelet restarts the API server on change).

Check which are active:

```bash
kubectl -n kube-system get pod -l component=kube-apiserver -o yaml | grep -i admission
# or inside the control-plane node:
kube-apiserver -h | grep -A3 enable-admission-plugins
```

Commonly enabled by default (version dependent):

| Controller | What it does |
|---|---|
| `NamespaceLifecycle` | Rejects creating objects in a namespace that is Terminating or does not exist; protects `default`, `kube-system`, `kube-public` from deletion |
| `LimitRanger` | Applies default requests/limits and enforces min/max from `LimitRange` objects (lesson 45) |
| `ResourceQuota` | Rejects creation that would exceed a namespace `ResourceQuota` (lesson 45) |
| `ServiceAccount` | Auto-assigns the `default` ServiceAccount and mounts its token |
| `DefaultStorageClass` | Fills `storageClassName` on PVCs that did not specify one (lesson 41) |
| `DefaultTolerationSeconds` | Adds default `not-ready`/`unreachable` tolerations (300 s) to Pods |
| `NodeRestriction` | Limits what a kubelet may modify: only its own Node object and Pods bound to that node; stops a compromised node from labeling itself into trusted pools (e.g. `node-restriction.kubernetes.io/` labels) |
| `PodSecurity` | The PSA described above |
| `MutatingAdmissionWebhook` | Calls external mutating webhooks (below) |
| `ValidatingAdmissionWebhook` | Calls external validating webhooks (below) |
| `ValidatingAdmissionPolicy` | In-process CEL-based validation, no webhook server needed (GA in 1.30) |
| `Priority` | Resolves `priorityClassName` into a priority number |
| `RuntimeClass` | Applies the RuntimeClass overhead and scheduling info to Pods |
| `StorageObjectInUseProtection` | Adds finalizers so PVC/PV in use cannot be deleted |
| `CertificateApproval/Signing/SubjectRestriction` | Guard CertificateSigningRequest handling |
| `TaintNodesByCondition` | Taints new nodes as not-ready until initialised |

Not enabled by default and worth knowing: `AlwaysPullImages` (forces `imagePullPolicy: Always`, protects private images in multi-tenant clusters), `EventRateLimit`.

### Dynamic admission: webhooks

When built-in controllers are not enough, you run your own HTTPS service (inside or outside the cluster) and register it. The API server sends it an `AdmissionReview` JSON request for matching operations; the service replies with an `AdmissionReview` response: `allowed: true/false` (plus a `patch` for mutating webhooks).

Real-world users of webhooks:

- **Istio / Linkerd** — mutating: inject sidecar proxy containers
- **cert-manager** — validating: reject invalid Certificate resources (lesson 38)
- **Kyverno / OPA Gatekeeper** — validating + mutating: policy engines (Kyverno: lesson 50)
- **Vault agent injector** — mutating: add secret-fetching sidecar
- **ingress-nginx** — validating: reject Ingress with broken snippets or duplicate paths
- Custom: "all Pods must have `owner` label", "images only from registry.example.com", "add default `imagePullSecrets`"

#### ValidatingWebhookConfiguration skeleton

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: require-owner-label
webhooks:
  - name: owner-label.example.com          # must be a fully-qualified name
    admissionReviewVersions: ["v1"]
    sideEffects: None                      # None | NoneOnDryRun
    failurePolicy: Fail                    # Fail (reject on webhook error) | Ignore
    timeoutSeconds: 5                      # 1-30, default 10
    matchPolicy: Equivalent
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
        scope: "Namespaced"
    namespaceSelector:                     # skip system namespaces!
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["kube-system", "webhook-system"]
    objectSelector:                        # optional: only objects with this label
      matchLabels: {}
    clientConfig:
      service:                             # in-cluster webhook
        name: owner-webhook
        namespace: webhook-system
        path: /validate
        port: 443
      caBundle: LS0tLS1CRUdJTi...          # CA that signed the webhook server cert (base64)
```

A `MutatingWebhookConfiguration` has the same shape plus `reinvocationPolicy: Never | IfNeeded`, and the webhook response carries a JSONPatch.

Out-of-cluster webhook: use `clientConfig.url: https://host:port/path` instead of `service`.

#### Key webhook settings and what they mean

| Field | Meaning | Why it matters |
|---|---|---|
| `failurePolicy: Fail` | If the webhook is unreachable/times out, the request is **rejected** | Safe, but an outage of the webhook blocks Pod creation cluster-wide |
| `failurePolicy: Ignore` | Request is allowed if the webhook fails | Availability over security; policy silently not enforced |
| `timeoutSeconds` | Max wait (1-30 s) | Keep small; slow webhooks slow every API call |
| `namespaceSelector` / `objectSelector` | Narrow the scope | Exclude the webhook's own namespace and `kube-system` |
| `sideEffects` | Declares whether calling it changes external state | Required; dry-run requests are skipped if `Some` |
| `matchConditions` (CEL, 1.28+) | Fine filter before calling the webhook | Reduce webhook traffic |
| `caBundle` | Trust anchor for the TLS cert of the webhook | Wrong value = every call fails with x509 errors |
| `reinvocationPolicy` | Mutating only: call again if later mutators changed the object | Needed when webhooks depend on each other |

**Deadlock danger:** if a `failurePolicy: Fail` webhook runs as a Pod, matches Pods, and is not excluded from its own scope, then when that Pod dies it cannot be recreated (the API server asks the dead webhook for permission to create the webhook Pod). Always exclude the webhook's namespace and `kube-system` with `namespaceSelector`.

### ValidatingAdmissionPolicy — webhook-free validation (brief)

Since Kubernetes 1.30 (GA) you can express validation rules in CEL (Common Expression Language), evaluated inside the API server — no webhook server, no TLS, no outage risk:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: max-replicas
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 5"
      message: "replicas must be 5 or fewer"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: max-replicas-binding
spec:
  policyName: max-replicas
  validationActions: ["Deny"]       # Deny | Warn | Audit
```

Good for simple rules. For mutation, rich policy libraries, image verification, or reports use a policy engine (lesson 50 introduces Kyverno).

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl get networkpolicy -A` | List policies in all namespaces | `kubectl get netpol -A` |
| `kubectl describe networkpolicy NAME -n NS` | Show selectors and rules as the API understood them | `kubectl describe netpol allow-frontend -n shop` |
| `kubectl apply -f policy.yaml` | Create or update a policy | `kubectl apply -f deny-all.yaml` |
| `kubectl delete networkpolicy NAME -n NS` | Remove a policy (traffic reopens immediately) | `kubectl delete netpol default-deny-all -n shop` |
| `kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -n NS --labels=app=frontend -- sh` | Temporary client Pod with chosen labels for testing | see walkthrough |
| `kubectl exec POD -n NS -- wget -qO- --timeout=3 http://svc:port` | HTTP connectivity test | `kubectl exec -n shop deploy/frontend -- wget -qO- -T 3 http://backend:8080` |
| `kubectl exec POD -- nc -zv -w 3 HOST PORT` | TCP port test | `kubectl exec -n shop deploy/backend -- nc -zv -w 3 database 5432` |
| `kubectl label ns NS pod-security.kubernetes.io/enforce=LEVEL` | Set PSA mode on namespace | `kubectl label ns shop pod-security.kubernetes.io/enforce=baseline` |
| `kubectl label --dry-run=server --overwrite ns NS ...` | Preview which Pods would violate a level | `kubectl label --dry-run=server --overwrite ns shop pod-security.kubernetes.io/enforce=restricted` |
| `kubectl get ns --show-labels` | See PSA labels on all namespaces | `kubectl get ns --show-labels` |
| `kubectl get ns NS -o jsonpath='{.metadata.labels}'` | Print one namespace's labels | `kubectl get ns shop -o jsonpath='{.metadata.labels}'` |
| `kubectl describe rs NAME` | See PSA / admission rejection of Pod creation by a Deployment | `kubectl describe rs -n shop -l app=web` |
| `kubectl get events -n NS --sort-by=.lastTimestamp` | Find `FailedCreate` admission errors | `kubectl get events -n shop --sort-by=.lastTimestamp` |
| `kubectl get validatingwebhookconfigurations` | List validating webhooks | `kubectl get validatingwebhookconfigurations` |
| `kubectl get mutatingwebhookconfigurations` | List mutating webhooks | `kubectl get mutatingwebhookconfigurations` |
| `kubectl describe validatingwebhookconfiguration NAME` | Rules, failurePolicy, targets | `kubectl describe validatingwebhookconfiguration ingress-nginx-admission` |
| `kubectl get validatingadmissionpolicy` | List CEL policies | `kubectl get validatingadmissionpolicy` |
| `kubectl delete validatingwebhookconfiguration NAME` | Emergency removal of a broken webhook that blocks the cluster | `kubectl delete validatingwebhookconfiguration require-owner-label` |
| `kubectl -n kube-system get pod -l component=kube-apiserver -o yaml` | Inspect API server flags including admission plugins | add `\| grep admission` |
| `kubectl auth can-i create networkpolicies -n NS` | Check RBAC for policy management | `kubectl auth can-i create netpol -n shop` |

## Examples / Walkthrough

### Lab prerequisites

A cluster with a policy-enforcing CNI. For minikube:

```bash
minikube start --cni=calico --kubernetes-version=stable
# or kind with Calico: create cluster with disableDefaultCNI: true then install Calico manifests
kubectl get pods -n kube-system | grep -E 'calico|cilium'
```

### Part A — Build the 3-tier app (no policies yet)

```bash
kubectl create namespace shop

# database: simple TCP listener standing in for postgres on 5432
kubectl -n shop run database --image=postgres:16 --labels=app=database \
  --env=POSTGRES_PASSWORD=labpassword --port=5432
kubectl -n shop expose pod database --port=5432

# backend: simple HTTP server on 8080
kubectl -n shop create deployment backend --image=nginxinc/nginx-unprivileged:1.27 --port=8080
kubectl -n shop expose deployment backend --port=8080

# frontend
kubectl -n shop create deployment frontend --image=nginxinc/nginx-unprivileged:1.27 --port=8080
kubectl -n shop expose deployment frontend --port=80 --target-port=8080

kubectl -n shop get pods,svc -o wide --show-labels
```

Check labels: `deployment` creates Pods with `app=backend` / `app=frontend`; the standalone Pod got `app=database`.

Baseline connectivity — everything reaches everything (we need a client with `wget` and `nc`):

```bash
kubectl -n shop run probe --image=busybox:1.36 --restart=Never --labels=app=probe -- sleep 3600

kubectl -n shop exec probe -- wget -qO- -T 3 http://backend:8080 | head -3     # works
kubectl -n shop exec probe -- nc -zv -w 3 database 5432                         # open
kubectl -n shop exec probe -- wget -qO- -T 3 http://frontend | head -3          # works
```

The `probe` Pod is not part of the architecture, yet it reaches the database. That is the problem.

### Part B — Default deny

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shop
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF

kubectl -n shop exec probe -- wget -qO- -T 3 http://backend:8080
# wget: bad address 'backend'     <- DNS egress blocked, so even name lookup fails

kubectl -n shop exec probe -- wget -qO- -T 3 http://10.x.x.x:8080   # use backend Pod IP
# wget: download timed out        <- traffic dropped
```

Two different failure messages teach the diagnosis:
- `bad address` / `Temporary failure in name resolution` = DNS is blocked (egress to kube-dns).
- `timed out` with an IP = the connection is dropped by policy (no RST, no "connection refused").
- `Connection refused` = reached the Pod, nothing listening — **not** a policy problem.

### Part C — Allow DNS, then allow each hop

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: shop
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - {protocol: UDP, port: 53}
        - {protocol: TCP, port: 53}
EOF
```

Now the three-tier rules. Remember each hop needs **two** policies — egress from the sender and ingress on the receiver — because the namespace is default-deny in both directions.

```yaml
# 1) ingress-nginx -> frontend (ingress on frontend)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-from-ingress
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: frontend}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - {protocol: TCP, port: 8080}
---
# 2a) frontend egress -> backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress-to-backend
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: frontend}
  policyTypes: [Egress]
  egress:
    - to:
        - podSelector:
            matchLabels: {app: backend}
      ports:
        - {protocol: TCP, port: 8080}
---
# 2b) backend ingress <- frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress-from-frontend
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: backend}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: {app: frontend}
      ports:
        - {protocol: TCP, port: 8080}
---
# 3a) backend egress -> database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-to-database
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: backend}
  policyTypes: [Egress]
  egress:
    - to:
        - podSelector:
            matchLabels: {app: database}
      ports:
        - {protocol: TCP, port: 5432}
---
# 3b) database ingress <- backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress-from-backend
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: database}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: {app: backend}
      ports:
        - {protocol: TCP, port: 5432}
```

Save as `three-tier.yaml` and `kubectl apply -f three-tier.yaml`.

### Part D — Test the matrix

Test from the right source Pods. The `nginxinc/nginx-unprivileged` image has no `nc`; use `wget` for HTTP and bash `/dev/tcp` or a throw-away busybox with the **same labels** as the source:

```bash
# frontend -> backend should WORK  (busybox pretending to be frontend)
kubectl -n shop run t1 --rm -it --restart=Never --image=busybox:1.36 \
  --labels=app=frontend -- wget -qO- -T 3 http://backend:8080 | head -3

# frontend -> database should FAIL (timeout)
kubectl -n shop run t2 --rm -it --restart=Never --image=busybox:1.36 \
  --labels=app=frontend -- nc -zv -w 3 database 5432

# backend -> database should WORK
kubectl -n shop run t3 --rm -it --restart=Never --image=busybox:1.36 \
  --labels=app=backend -- nc -zv -w 3 database 5432

# unlabeled / wrong labels -> backend should FAIL
kubectl -n shop run t4 --rm -it --restart=Never --image=busybox:1.36 \
  -- wget -qO- -T 3 http://backend:8080

# database -> internet should FAIL (no egress allowed)
kubectl -n shop run t5 --rm -it --restart=Never --image=busybox:1.36 \
  --labels=app=database -- wget -qO- -T 3 http://example.com
```

Expected summary table to fill in during class:

| Source | Destination | Port | Expected |
|---|---|---|---|
| ingress-nginx | frontend | 8080 | allow |
| frontend | backend | 8080 | allow |
| frontend | database | 5432 | **deny** |
| backend | database | 5432 | allow |
| probe (no role) | backend | 8080 | **deny** |
| database | internet | 80 | **deny** |
| any Pod | CoreDNS | 53 | allow |

Using `--labels` on a throw-away Pod to impersonate a role is the quickest way to test policies. It also shows the weakness: **anyone who can create Pods in the namespace can pick any label**, so combine NetworkPolicy with RBAC (lesson 47) that limits who may create Pods.

### Part E — Cross-namespace and ipBlock

Allow a monitoring namespace to scrape the backend metrics port:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-prometheus
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: backend}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus   # AND: only Prometheus Pods in monitoring
      ports:
        - {protocol: TCP, port: 9100}
```

Allow the backend to call a specific external API range except one host:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-external
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: backend}
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24
            except:
              - 203.0.113.99/32
      ports:
        - {protocol: TCP, port: 443}
```

Block the cloud metadata endpoint while allowing other internet traffic (a hardening classic):

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0
          except:
            - 169.254.169.254/32       # AWS/GCP/Azure instance metadata
```

### Part F — PodSecurity Admission lab

```bash
kubectl create namespace psa-demo

# 1. Nothing enforced yet: a privileged Pod is accepted
cat <<'EOF' | kubectl -n psa-demo apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: priv
spec:
  containers:
    - name: c
      image: busybox:1.36
      command: ["sleep","3600"]
      securityContext:
        privileged: true
EOF
kubectl -n psa-demo get pod priv

# 2. See what would break if we enforce baseline (dry-run, nothing changes)
kubectl label --dry-run=server --overwrite ns psa-demo \
  pod-security.kubernetes.io/enforce=baseline
# Warning: existing pods in namespace "psa-demo" violate the new PodSecurity enforce level "baseline:latest"
# Warning: priv: privileged
# namespace/psa-demo labeled (server dry run)

# 3. Enforce baseline for real, warn and audit on restricted
kubectl label --overwrite ns psa-demo \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# 4. The running privileged Pod is still alive (enforce is not retroactive)
kubectl -n psa-demo get pod priv

# 5. A NEW privileged Pod is rejected
kubectl -n psa-demo run priv2 --image=busybox:1.36 \
  --overrides='{"spec":{"containers":[{"name":"priv2","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
# Error from server (Forbidden): pods "priv2" is forbidden: violates PodSecurity "baseline:latest": privileged (container "priv2" must not set securityContext.privileged=true)

# 6. A plain Pod passes baseline but gets a WARNING for restricted
kubectl -n psa-demo run plain --image=nginx:1.27
# Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "plain" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (...must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (...), seccompProfile (...)
# pod/plain created

# 7. Now enforce restricted and show a Deployment failing at the ReplicaSet level
kubectl label --overwrite ns psa-demo pod-security.kubernetes.io/enforce=restricted
kubectl -n psa-demo create deployment web --image=nginx:1.27
kubectl -n psa-demo get deploy web              # READY 0/1
kubectl -n psa-demo describe rs -l app=web | tail -8   # FailedCreate ... violates PodSecurity "restricted:latest"
kubectl -n psa-demo get events --sort-by=.lastTimestamp | tail

# 8. Fix: a compliant Deployment
cat <<'EOF' | kubectl -n psa-demo apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-ok
spec:
  replicas: 1
  selector:
    matchLabels: {app: web-ok}
  template:
    metadata:
      labels: {app: web-ok}
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: web
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
EOF
kubectl -n psa-demo get pods
```

Teaching points:
- `kubectl create deployment web` **succeeds** (the Deployment object is fine) but Pods never appear — the rejection is visible in the **ReplicaSet** events. Students commonly look at the Deployment and see nothing.
- The plain `nginx` image fails `restricted` mainly because it runs as root and wants to bind port 80; `nginxinc/nginx-unprivileged` listens on 8080 as UID 101.
- `warn` messages are printed by kubectl itself; GitOps/CI tools (Argo CD, Helm) might hide them.

### Part G — Observe admission controllers

```bash
# Which plugins does this API server run?
kubectl -n kube-system get pod -l component=kube-apiserver -o jsonpath='{.items[0].spec.containers[0].command}' | tr ',' '\n' | grep -i admission

# LimitRanger / ResourceQuota in action (objects covered in lesson 45)
kubectl create ns quota-demo
kubectl -n quota-demo create quota q --hard=pods=1
kubectl -n quota-demo run a --image=nginx:1.27
kubectl -n quota-demo run b --image=nginx:1.27
# Error from server (Forbidden): pods "b" is forbidden: exceeded quota: q, requested: pods=1, used: pods=1, limited: pods=1

# NamespaceLifecycle
kubectl delete ns quota-demo --wait=false
kubectl -n quota-demo run c --image=nginx:1.27
# Error from server (Forbidden): ... namespace quota-demo because it is being terminated

# DefaultStorageClass: PVC without storageClassName gets one added
kubectl get pvc -o jsonpath='{.items[*].spec.storageClassName}'
```

### Part H — Look at a real webhook

```bash
kubectl get validatingwebhookconfigurations
# ingress-nginx-admission, cert-manager-webhook ... (if installed in lessons 38)

kubectl get validatingwebhookconfiguration ingress-nginx-admission -o yaml | less
# rules: ingresses, failurePolicy: Fail, clientConfig.service, caBundle

# Prove the webhook is in the path: create an Ingress that ingress-nginx rejects
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bad
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: "this is not valid nginx config;"
spec:
  ingressClassName: nginx
  rules:
    - host: bad.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: {service: {name: backend, port: {number: 8080}}}
EOF
# Error from server (BadRequest): admission webhook "validate.nginx.ingress.kubernetes.io" denied the request: ...
```

### Part I — Mini webhook outage experiment (instructor demo)

```bash
# Register a validating webhook pointing at a non-existent service with failurePolicy: Fail
cat <<'EOF' | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: broken-demo
webhooks:
  - name: broken.demo.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
    timeoutSeconds: 3
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["configmaps"]
    namespaceSelector:
      matchLabels:
        webhook-demo: "true"
    clientConfig:
      service:
        name: does-not-exist
        namespace: default
        path: /validate
EOF

kubectl create ns wh-demo
kubectl label ns wh-demo webhook-demo=true
kubectl -n wh-demo create configmap x --from-literal=a=b
# Error from server (InternalError): Internal error occurred: failed calling webhook "broken.demo.example.com": ... service "does-not-exist" not found

# Other namespaces unaffected thanks to namespaceSelector
kubectl create configmap y --from-literal=a=b      # works

# Cleanup / emergency fix = delete the configuration
kubectl delete validatingwebhookconfiguration broken-demo
```

This is exactly the outage scenario in production: a broken webhook with `failurePolicy: Fail` blocks a whole class of API calls until someone deletes the configuration.

### Cleanup

```bash
kubectl delete ns shop psa-demo wh-demo
```

## Common Pitfalls

- **CNI does not support NetworkPolicy** — policies are accepted by the API and silently ignored (plain Flannel, old kindnet). Always run a deny test first to prove enforcement.
- **Forgetting DNS after default-deny egress** — every Pod fails with `bad address` or `Temporary failure in name resolution`. Add the allow-DNS policy (UDP and TCP 53 to `k8s-app=kube-dns` in `kube-system`) immediately.
- **Only writing one side of a connection** — with default deny both ways, `frontend → backend` needs an egress rule on frontend **and** an ingress rule on backend. Missing either side times out.
- **AND vs OR in `from`/`to`** — a missing dash makes the rule much broader (all Pods in a namespace) or narrower than intended. Review indentation carefully; use `kubectl describe netpol` to read back how it was parsed.
- **Policy in the wrong namespace** — policies only apply to Pods in their own namespace. A policy in `default` never protects Pods in `shop`.
- **`podSelector` in `from` assumed to be cluster-wide** — without a `namespaceSelector` it only matches Pods in the **policy's own** namespace.
- **No `policyTypes`** — inferred values surprise people. A policy with only `egress:` rules and no `policyTypes: [Egress]` actually also isolates ingress. Be explicit.
- **Relying on `ipBlock` for Pod-to-Pod rules** — Pod IPs change; and when traffic passes through a Service or NAT the source IP seen by policy may differ. Use label selectors in-cluster.
- **Egress to a Service ClusterIP** — policies match the Pod IP after kube-proxy DNAT in most CNIs; write the rule against the backend Pods' labels, not the Service IP.
- **Impersonation by labels** — anyone allowed to create Pods can set `app=backend` and get its access. Restrict Pod creation with RBAC and PodSecurity.
- **Blocking kubelet probes** — usually not blocked (node traffic is allowed), but with some CNIs/hostNetwork setups a strict policy can break probes; check `kubectl describe pod` events if Pods go NotReady after applying policy.
- **PSA `enforce` is not retroactive** — running Pods keep running; they fail only when restarted or rescheduled. Use `--dry-run=server` and `warn`/`audit` first.
- **Looking for PSA errors on the Deployment** — the Deployment shows no error; the ReplicaSet events (`kubectl describe rs`) show `FailedCreate ... violates PodSecurity`.
- **Enabling `enforce: restricted` on `kube-system`** — breaks CNI, kube-proxy, CSI drivers (they need privileged). Keep system namespaces `privileged` or exempt them.
- **Using the PSP mental model** — PodSecurityPolicy was removed in 1.25. PSA has no per-user policy objects; it works through namespace labels and API-server exemptions.
- **`failurePolicy: Fail` webhook that covers its own Pod** — deadlock: the webhook Pod cannot be recreated because the API server needs the dead webhook to approve it. Always exclude the webhook's namespace and `kube-system` with `namespaceSelector`.
- **`failurePolicy: Ignore` assumed safe** — then your security control silently turns off during outages. Choose consciously per webhook.
- **Wrong or expired `caBundle`/serving cert** — all calls fail with `x509: certificate signed by unknown authority`. Use cert-manager's `cainjector` annotation `cert-manager.io/inject-ca-from` to keep the bundle updated.
- **Slow webhooks** — every matching API call waits up to `timeoutSeconds`. Keep logic fast, scope rules tightly, set `timeoutSeconds: 5` or lower.
- **Expecting admission to run on reads** — admission runs on create/update/delete/connect only. It cannot hide `get secrets`; that is RBAC (lesson 47).
- **Expecting admission to fix existing objects** — it only sees new requests. Old non-compliant resources remain until recreated.
- **Editing the API server manifest with a typo** — on kubeadm the static Pod manifest is the API server; a bad `--enable-admission-plugins` value makes it crash-loop and `kubectl` stops working. Back up the file first (`cp` outside `/etc/kubernetes/manifests`).

## FAQ

**Q: Does a NetworkPolicy block traffic by default when I create it?**
A: A NetworkPolicy only restricts the Pods its `podSelector` selects, and only in the directions listed in `policyTypes`. Pods that no policy selects stay fully open. So the first policy you add to a namespace usually restricts only the selected Pods; that is why people apply a `podSelector: {}` default-deny policy first.

**Q: How do I deny a specific source, e.g. "block everyone except X"?**
A: There is no deny rule. Isolate the Pod (select it with a policy for that direction) and list only the sources you allow. Everything not listed is denied. For explicit deny rules use CNI-specific CRDs (Calico `GlobalNetworkPolicy`, Cilium `CiliumNetworkPolicy`).

**Q: If I have two policies selecting the same Pod, which one wins?**
A: Neither "wins"; they are unioned. The Pod accepts traffic allowed by any policy. There is no precedence and no ordering. Adding a policy can only widen what is allowed for an already-isolated Pod.

**Q: Why does my policy have no effect?**
A: In order of likelihood: (1) the CNI does not enforce NetworkPolicy; (2) the policy is in a different namespace than the target Pods; (3) `podSelector` labels do not match the Pods (`kubectl get pods --show-labels`); (4) traffic you test is allowed by another policy; (5) you test from a `hostNetwork` Pod or from a node.

**Q: Why did everything stop working, including DNS, after default-deny egress?**
A: Pods need egress to CoreDNS on port 53 (UDP and TCP). Add the allow-DNS policy shown in this lesson. Test with `nslookup kubernetes.default` from a Pod.

**Q: Do I need to allow reply traffic?**
A: No. NetworkPolicy is stateful: if a connection is allowed in one direction, return packets on that connection are allowed automatically.

**Q: Can I allow traffic from the Ingress controller?**
A: Yes. Allow ingress to the app Pods from the namespace of the ingress controller (`namespaceSelector` with `kubernetes.io/metadata.name: ingress-nginx`), optionally combined with a `podSelector` for the controller Pods (AND). The Ingress controller connects to Pod IPs directly, so the Pod's own policy must allow it. The same applies to Gateway API data planes (lesson 39) — allow from the namespace where the gateway proxy Pods run.

**Q: Can I limit traffic to a domain name like github.com?**
A: Not in core NetworkPolicy. Options: Cilium `toFQDNs`, Calico Enterprise/Cloud domain rules, an egress proxy or egress gateway, or `ipBlock` ranges (fragile).

**Q: What is the difference between a NetworkPolicy and a Service mesh authorization policy?**
A: NetworkPolicy works at L3/L4 by Pod labels and IPs, enforced by the CNI. A mesh (Istio, Linkerd) adds mTLS identity and L7 rules (HTTP method/path, JWT claims). They complement each other; NetworkPolicy is the baseline.

**Q: How do I find out which policies affect a Pod?**
A: There is no single command. List `kubectl get netpol -n NS -o wide`, and compare each `podSelector` against `kubectl get pod POD --show-labels`. Cilium has `cilium policy` tools and Calico has `calicoctl` for effective policy views. Network policy editors/visualizers exist (Cilium editor, `kubectl np-viewer` plugins).

**Q: What does `podSelector: {}` mean?**
A: Empty selector matches every Pod in the namespace of the policy. As `spec.podSelector` it targets all Pods; inside `from`/`to` it allows all Pods of the same namespace.

**Q: What is the difference between `Ingress` in a NetworkPolicy and an Ingress object?**
A: Same word, different things. NetworkPolicy "ingress" means incoming network traffic to a Pod. The Ingress resource (lesson 38) is an HTTP routing object for traffic entering the cluster.

**Q: Which Pod Security profile should I use?**
A: Target `restricted` for application namespaces, accept `baseline` where an image truly needs it, and keep `privileged` only for system namespaces (CNI, CSI, node agents). Roll out with `warn=restricted`, `audit=restricted`, `enforce=baseline`, then raise `enforce` once warnings stop.

**Q: Do the PSA labels block `kubectl exec` or changes to running Pods?**
A: PSA checks Pod creation and updates to Pod spec fields (which are mostly immutable). It does not evict existing Pods and does not restrict `exec`. Use RBAC for `pods/exec`.

**Q: What happened to PodSecurityPolicy?**
A: PSP was deprecated in 1.21 and removed in 1.25. Replacement options: PodSecurity Admission (built-in, namespace-label based), ValidatingAdmissionPolicy (CEL), or Kyverno/Gatekeeper for custom policy. Migrate by mapping each PSP to the closest PSS level.

**Q: How do I let one special Pod run privileged in an otherwise restricted cluster?**
A: Put it in its own namespace labeled `enforce=privileged` (or `baseline`), protect that namespace with RBAC, and exempt specific usernames/runtimeClasses/namespaces in the API server `AdmissionConfiguration`. Never lower the level of a shared application namespace for one workload.

**Q: Why does my Pod get rejected only with a Deployment and I see nothing?**
A: The rejection occurs when the ReplicaSet controller creates the Pod. Run `kubectl describe rs <name>` or `kubectl get events` in the namespace to see the `FailedCreate` message.

**Q: What is the order: RBAC or admission first?**
A: Authentication → authorization (RBAC) → mutating admission → schema validation → validating admission → persist. RBAC decides if you may call the API at all; admission decides if the content is acceptable.

**Q: Mutating or validating — which runs first and why?**
A: Mutating first, so validating controllers check the final, mutated object. Mutating webhooks run serially; validating ones run in parallel.

**Q: Can admission controllers read data (e.g. other Pods) to decide?**
A: Webhooks can call back to the API server or hold caches. CEL `ValidatingAdmissionPolicy` can reference parameter resources (`paramKind`) and the old object on updates, but cannot query arbitrary cluster state. Policy engines like Kyverno can look up cluster resources.

**Q: How do I disable a built-in admission plugin?**
A: Add `--disable-admission-plugins=Name` to `kube-apiserver` (edit `/etc/kubernetes/manifests/kube-apiserver.yaml` on kubeadm). Do this only deliberately; most plugins are security or consistency guards. Managed clusters (EKS/GKE/AKS) do not let you change these flags.

**Q: A webhook is blocking everything and I cannot deploy the fix. What do I do?**
A: Delete or edit the `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration` (`kubectl delete validatingwebhookconfiguration NAME`) — this object is cluster-scoped and not itself intercepted unless a webhook matches webhook configurations. Alternatively set `failurePolicy: Ignore` temporarily. Then fix the webhook Pod or certificate.

**Q: What is `sideEffects` for?**
A: A webhook must declare whether it writes external state. `None` or `NoneOnDryRun` tells the API server it is safe to call during `--dry-run` requests. Webhooks declared with side effects are skipped on dry-run requests, so dry-run results can differ from real ones.

**Q: Do admission webhooks run for `kubectl get`?**
A: No. Admission applies only to create, update, delete, and connect (exec/attach/port-forward) operations.

**Q: ValidatingWebhook vs ValidatingAdmissionPolicy — which should I pick?**
A: For simple field checks use ValidatingAdmissionPolicy (CEL): no extra server, no TLS, no availability risk. For mutation, external lookups, image verification or complex libraries, use a webhook-based engine (Kyverno intro in lesson 50).

## Practice / Exercise

### Core

1. **Prove enforcement.** In a new namespace `np-test` run an `nginx` Deployment and a `busybox` client Pod. Confirm the client reaches nginx. Apply a `podSelector: {}` ingress-deny policy and confirm the connection now times out. If it does not, identify why (CNI?).
2. **Default deny + DNS.** Apply deny-all (both directions) to `shop`, observe the `bad address` DNS failure, then add the allow-DNS policy and confirm names resolve again but connections still time out.
3. **3-tier policy set.** Deploy frontend/backend/database as in the walkthrough and write the five policies (ingress-nginx → frontend, frontend → backend, backend → database, each with both egress and ingress sides). Fill in the test matrix table with real results.
4. **AND vs OR.** Write two policies for the backend: one allowing `namespaceSelector: team=a` AND `podSelector: role=client`, and one with two separate peers (OR). Create test Pods in various namespaces/labels and record which succeed in each case.
5. **PSA ladder.** In namespace `psa-demo` apply `enforce=baseline`, `warn=restricted`. Create a privileged Pod (rejected), an unhardened nginx Pod (accepted with warning), and a hardened Pod (no warning).
6. **Fix a Deployment under restricted.** Set `enforce=restricted`, create `kubectl create deployment web --image=nginx`, find the rejection reason in the ReplicaSet events, and rewrite the manifest so it passes.
7. **Admission plugin hunt.** Find the `--enable-admission-plugins` value of your API server and list five controllers it enables (explicit or default). Trigger `ResourceQuota` and `NamespaceLifecycle` rejections yourself.
8. **Read a webhook.** Run `kubectl get validatingwebhookconfigurations -o yaml` and, for one entry, write down: which resources/operations it intercepts, its `failurePolicy`, `timeoutSeconds`, `namespaceSelector`, and where `clientConfig` points.

### Stretch

1. **Metadata endpoint block.** Add an egress policy that allows the backend to reach the internet on 443 except `169.254.169.254/32`. Test from a Pod (if on cloud) or reason through the rule evaluation and explain why a rule with `0.0.0.0/0` plus `except` is safer than an unrestricted egress.
2. **Port ranges.** Write a policy allowing TCP `8000-8999` using `endPort`, and prove with `nc` that 8500 passes while 9000 times out.
3. **Namespace-wide template.** Write a small shell loop or Kustomize-ready file that applies default-deny + allow-DNS to every namespace labeled `tier=app` (preview of lessons 56-58 — here just a bash `for` loop is enough).
4. **Webhook outage drill.** Reproduce Part I (broken webhook with `failurePolicy: Fail`), then fix it three ways: delete the configuration, switch to `Ignore`, narrow the `namespaceSelector`. Explain the trade-offs.
5. **CEL policy.** Write a `ValidatingAdmissionPolicy` that rejects Deployments whose containers use an image tag `latest`, bind it with `Deny`, then test with `nginx:latest` and `nginx:1.27`. Hint: `object.spec.template.spec.containers.all(c, !c.image.endsWith(':latest'))`.
6. **Exemption design.** On paper, design the PSA configuration for a cluster with namespaces `kube-system` (CNI), `monitoring` (node-exporter needs hostNetwork), `payments` (strict), `dev` (relaxed). Choose enforce/warn/audit levels and say which namespaces need exemptions and why.

## Further Reading

- Kubernetes docs — Network Policies: https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Declare Network Policy (tutorial): https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/
- Network Policy recipes (ahmetb/kubernetes-network-policy-recipes): https://github.com/ahmetb/kubernetes-network-policy-recipes
- Pod Security Standards: https://kubernetes.io/docs/concepts/security/pod-security-standards/
- Pod Security Admission: https://kubernetes.io/docs/concepts/security/pod-security-admission/
- Enforce Pod Security Standards with namespace labels: https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/
- Admission Controllers reference: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
- Dynamic Admission Control (webhooks): https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
- Validating Admission Policy (CEL): https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/
- Calico NetworkPolicy docs: https://docs.tigera.io/calico/latest/network-policy/
- Cilium network policy: https://docs.cilium.io/en/stable/security/policy/

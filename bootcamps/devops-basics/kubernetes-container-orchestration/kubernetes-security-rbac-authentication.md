# Lesson 47: Kubernetes Security – RBAC & Authentication

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-46 (cluster architecture, kubeconfig and contexts, kubectl, Pods/Deployments, Services, Secrets, namespaces). Student can read and write YAML manifests and use `kubectl apply/get/describe/exec/logs`. Basic familiarity with TLS certificates (lesson 13 certbot) helps.

## Learning Objectives

By end of lesson student can:
- Describe the path of an API request: authentication, then authorization, then admission (admission itself is lesson 48)
- Name Kubernetes authentication methods (client certificates, ServiceAccount tokens, OIDC, webhook token, bootstrap tokens) and explain which ones are used by humans vs workloads
- Create a real user end-to-end: private key, CSR, `CertificateSigningRequest` object, approval, kubeconfig entry
- Explain what a ServiceAccount is, what the `default` ServiceAccount does, and how bound (projected) tokens differ from old long-lived Secret tokens
- Create custom ServiceAccounts, mint tokens with `kubectl create token`, and disable token auto-mount with `automountServiceAccountToken: false`
- Write `Role` and `ClusterRole` rules using `apiGroups`, `resources`, `resourceNames`, `verbs`, and subresources such as `pods/log` and `pods/exec`
- Bind permissions with `RoleBinding` and `ClusterRoleBinding`, and explain why a ClusterRole bound by a RoleBinding is namespace-scoped
- Test permissions with `kubectl auth can-i` (`--as`, `--list`) and `kubectl auth whoami`
- Use the built-in roles `cluster-admin`, `admin`, `edit`, `view`
- Explain privilege-escalation prevention (`escalate`, `bind` verbs) and apply least privilege
- Audit RBAC with `kubectl` queries and community tools (`rbac-tool`, `kubectl-who-can`, `rakkess`)
- Diagnose `Forbidden` errors from the error message alone

## Topics

- Authentication methods: kubeconfig certificates, ServiceAccount tokens
- ServiceAccounts: default SA, creating custom SA, automountServiceAccountToken
- Roles and ClusterRoles: rules (apiGroups, resources, verbs); kubectl create role
- RoleBindings and ClusterRoleBindings: subjects (User, Group, ServiceAccount); scope
- RBAC best practices: least privilege; kubectl auth can-i; audit RBAC with tools

## Concepts

### The API request path: authn, authz, admission

Everything in Kubernetes goes through `kube-apiserver` (lesson 34): `kubectl`, the kubelet, the scheduler, controllers, CI pipelines, operators, your own pods. Every single request passes three gates, in this order:

1. **Authentication (authn)** – *Who are you?* The API server inspects the credentials (client certificate, bearer token, OIDC ID token) and produces an identity: a **username**, a list of **groups**, and optional extra info. If no authenticator accepts the credentials, the request is treated as `system:anonymous` (if anonymous auth is enabled) and usually gets **HTTP 401 Unauthorized**.
2. **Authorization (authz)** – *Are you allowed to do this?* The API server asks its authorizers (normally `Node` and `RBAC`) whether this identity may perform this **verb** on this **resource** in this **namespace**. No allow = **HTTP 403 Forbidden**.
3. **Admission control** – *Is this particular object acceptable?* Mutating and validating admission plugins can modify or reject the object (defaults, quotas, PodSecurity, webhooks). Covered in lesson 48.

Only after all three does the object reach etcd.

Key mental model:
- **401** means authn failed – wrong/expired credentials, unknown user. RBAC is not involved yet.
- **403** means authn succeeded, RBAC said no. Fix is a Role/Binding, not a new certificate.

**Kubernetes has no User object.** There is no `kubectl create user`. A "user" is just a string (the `CN` of a client certificate, or a claim in an OIDC token) that the cluster trusts because a trusted CA or identity provider vouched for it. ServiceAccounts, in contrast, ARE real API objects.

### Authentication methods

| Method | Used by | How it works |
|---|---|---|
| **X.509 client certificates** | Admins, kubelets, control-plane components | Cert signed by the cluster CA. `CN` = username, `O` = group(s). Default in kubeadm/rke2 admin kubeconfig |
| **ServiceAccount tokens** | Pods, CI jobs, operators (non-human) | Signed JWT verified by the API server. Username is `system:serviceaccount:<ns>:<name>` |
| **OIDC (OpenID Connect)** | Humans in real companies | API server trusts an external IdP (Keycloak, Dex, Google, Azure AD). `kubectl` gets an ID token via a plugin such as `kubelogin` (concept only here) |
| **Webhook token authentication** | Custom SSO bridges | API server POSTs the token to an external service to validate |
| **Bootstrap tokens** | `kubeadm join` | Short-lived secrets in `kube-system` letting new nodes request certs |
| **Static token file / basic auth** | Legacy / lab only | Removed (basic auth) or discouraged (static token file). Never use in production |

Rule of thumb: **humans → OIDC (or short-lived certs); workloads → ServiceAccounts.** Long-lived client certificates for humans are a security smell because Kubernetes has **no certificate revocation list**: a leaked cert stays valid until it expires (or the CA is rotated).

### Client certificates and kubeconfig (recap + depth)

The admin kubeconfig you got from kubeadm/rke2 contains a client cert whose identity is in group `system:masters`, which is hard-wired to `cluster-admin`. That is why it can do everything and why you should not hand it to anyone.

Kubeconfig anatomy (lesson 34), three lists wired together by a context:

```yaml
apiVersion: v1
kind: Config
clusters:            # WHERE: API server URL + CA to trust
- name: lab
  cluster:
    server: https://10.0.0.10:6443
    certificate-authority-data: <base64 CA>
users:               # WHO: credentials
- name: anna
  user:
    client-certificate-data: <base64 cert>
    client-key-data: <base64 key>
contexts:            # glue: cluster + user + default namespace
- name: anna@lab
  context: {cluster: lab, user: anna, namespace: dev}
current-context: anna@lab
```

How an API server decides who a cert-holder is:
- It verifies the cert chain against `--client-ca-file`.
- `Subject: CN=anna` becomes **username** `anna`.
- Each `O=` (Organization) field becomes a **group**. `O=developers` → group `developers`.

So to give "anna" permissions you bind to **User `anna`** or to **Group `developers`**, but only if the cert has those values.

### ServiceAccounts

A **ServiceAccount (SA)** is an identity for a process running in a Pod. It is a namespaced API object.

- Every namespace automatically gets a `default` ServiceAccount.
- Every Pod that does not specify `serviceAccountName` runs as `default` in its namespace.
- The identity string is `system:serviceaccount:<namespace>:<name>`, and it also belongs to groups `system:serviceaccounts` and `system:serviceaccounts:<namespace>`.
- **A new SA has zero permissions** (apart from a few discovery APIs every authenticated user gets, like `system:discovery`). It is deny-by-default.

#### Tokens: bound (projected) vs legacy

- **Legacy (before 1.24):** creating an SA auto-created a Secret holding a never-expiring JWT. Leaked token = permanent access. Since **1.24 these are no longer auto-created**.
- **Bound / projected tokens (current):** the kubelet requests a short-lived JWT from the API server through the **TokenRequest API** and mounts it into the Pod. Properties:
  - **Audience-bound** and **time-limited** (default 1 hour, kubelet refreshes it automatically at ~80% of lifetime).
  - **Bound to the Pod** – when the Pod is deleted, the token becomes invalid.
  - Mounted at `/var/run/secrets/kubernetes.io/serviceaccount/` with three files: `token`, `ca.crt`, `namespace`.

Why Pods get the token at all: so that in-cluster programs (operators, controllers, CI runners, ingress controllers, Prometheus) can call the API server. A plain web app almost never needs this.

#### automountServiceAccountToken

If your app never talks to the Kubernetes API, **do not give it a token**. Set on the Pod spec (wins) or on the ServiceAccount (default for all its Pods):

```yaml
spec:
  automountServiceAccountToken: false
```

Reason: if an attacker gets code execution in the container, a mounted token is the first thing they use to pivot into the cluster.

### RBAC: Role-Based Access Control

RBAC is the authorizer (`--authorization-mode=Node,RBAC`). It works on **additive allow rules only**: there is no "deny" in RBAC. Permission is the union of every rule from every binding that matches the identity.

Four API objects (group `rbac.authorization.k8s.io/v1`):

| Object | Scope | Defines |
|---|---|---|
| `Role` | One namespace | A set of allowed actions on resources in that namespace |
| `ClusterRole` | Cluster-wide definition | Actions on cluster-scoped resources (nodes, PVs, namespaces), non-resource URLs (`/healthz`), OR reusable rules for any namespace |
| `RoleBinding` | One namespace | Attaches a Role **or ClusterRole** to subjects *inside that namespace* |
| `ClusterRoleBinding` | Whole cluster | Attaches a ClusterRole to subjects *in every namespace* |

Sentence to memorise: **Roles say WHAT. Bindings say WHO and WHERE.**

#### Rule anatomy

```yaml
rules:
- apiGroups: [""]              # "" = core group (pods, services, configmaps, secrets, nodes...)
  resources: ["pods"]          # what kind of object
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]          # deployments, statefulsets, daemonsets, replicasets
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/log"]      # SUBRESOURCE: reading logs is separate from reading pods
  verbs: ["get"]
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]   # only this one object
  verbs: ["get", "update"]
```

- **apiGroups** – find via `kubectl api-resources` (lesson 35): the `APIVERSION` column minus the version. `v1` → `""`, `apps/v1` → `apps`, `networking.k8s.io/v1` → `networking.k8s.io`, `batch/v1` → `batch`.
- **resources** – plural lowercase names. `*` = all.
- **verbs** – map to HTTP: `get` (one), `list` (collection), `watch` (stream), `create`, `update` (PUT), `patch` (PATCH), `delete`, `deletecollection`. Special: `impersonate`, `bind`, `escalate`, `use`, `approve`.
- **resourceNames** – restrict to specific object names. Cannot be used with `create` or with `list`/`watch` of whole collection (except via field selectors).
- **Subresources** – `pods/log`, `pods/exec`, `pods/portforward`, `pods/status`, `deployments/scale`. `pods/exec` + `create` verb = shell in any pod, which is effectively access to everything that pod can reach (including its mounted secrets and token). Treat as highly privileged.
- **nonResourceURLs** (ClusterRole only) – `/healthz`, `/metrics`, `/version`.

A subtle fact: **`get` on `secrets` AND `list` on `secrets` both reveal secret contents** (list returns the full objects). Granting "list secrets" is the same as granting "read all secrets".

#### Subjects

```yaml
subjects:
- kind: User                  # string from cert CN / OIDC claim
  name: anna
  apiGroup: rbac.authorization.k8s.io
- kind: Group                 # cert O= / OIDC groups claim
  name: developers
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount        # ONLY kind that has a namespace field
  name: ci-deployer
  namespace: dev
```

#### Role vs ClusterRole, RoleBinding vs ClusterRoleBinding: the four combinations

| Role object | Binding object | Effective scope |
|---|---|---|
| Role | RoleBinding (same ns) | That one namespace |
| ClusterRole | ClusterRoleBinding | All namespaces + cluster-scoped resources |
| ClusterRole | **RoleBinding** | **Only the namespace of the RoleBinding** (reusable template!) |
| Role | ClusterRoleBinding | **Not allowed** |

The third row is the most useful pattern: define `view`/`edit` once as ClusterRoles, then hand them out per namespace with RoleBindings. This is exactly how the built-in roles are meant to be used.

A binding's `roleRef` is **immutable**. To change which role a binding points to, delete and recreate it.

### Built-in user-facing ClusterRoles

| ClusterRole | Meaning |
|---|---|
| `cluster-admin` | Everything, everywhere. Bound to group `system:masters`. Via ClusterRoleBinding = superuser; via RoleBinding = full control of one namespace (but not ResourceQuota/namespace itself) |
| `admin` | Full control inside a namespace, including creating Roles/RoleBindings there. Intended for RoleBinding |
| `edit` | Read/write most objects in a namespace (incl. Secrets, ServiceAccount use), **no** Roles/RoleBindings |
| `view` | Read-only on most objects. **Cannot read Secrets** and cannot view Roles/RoleBindings |

Also exist `system:*` ClusterRoles (control-plane components): never edit them – they are auto-reconciled at API server restart (`rbac.authorization.kubernetes.io/autoupdate: "true"`).

**Aggregated ClusterRoles** (brief): `admin`, `edit`, `view` are built with `aggregationRule` – any ClusterRole labeled `rbac.authorization.k8s.io/aggregate-to-view: "true"` is merged into `view`. That is how CRDs (e.g. Gateway API, cert-manager) extend `view/edit/admin` to their custom resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["monitoring.example.com"]
  resources: ["probes"]
  verbs: ["get", "list", "watch"]
```

### Privilege-escalation prevention

The API server stops users from granting more than they have:
- To **create or update a Role/ClusterRole**, you must already hold every permission in it – or hold the `escalate` verb on `roles`/`clusterroles`.
- To **create a binding**, you must already hold all the permissions in the referenced role – or hold the `bind` verb on that role.

Dangerous verbs/permissions to treat like root:
- `escalate`, `bind`, `impersonate` (act as any user/group)
- `create` on `pods` (can run a pod mounting any secret in the namespace or running as a privileged pod when no admission blocks it)
- `get/list` on `secrets`
- `create` on `pods/exec`
- `create` on `serviceaccounts/token` (mint tokens for any SA)
- `*` on `*`

### Least privilege: practical rules

1. Start from zero; add only verbs/resources the workload demonstrably uses.
2. Prefer **Role + RoleBinding** over ClusterRole + ClusterRoleBinding.
3. One ServiceAccount per workload; never use `default` for anything that needs permissions.
4. Avoid wildcards (`*`) in `resources`, `verbs`, `apiGroups`.
5. Use `resourceNames` when a workload touches one known object.
6. Do not bind `cluster-admin` to ServiceAccounts, CI systems, or Helm/Tiller-like tools. Bind scoped Roles instead.
7. Prefer groups over individual users in bindings: add a person to the IdP group, no YAML change.
8. Review regularly; remove unused bindings; review after staff changes.
9. Disable token auto-mount for pods that do not call the API.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl auth whoami` | Show identity the API server sees (1.28+) | `kubectl auth whoami` |
| `kubectl auth can-i <verb> <resource>` | Check own permission | `kubectl auth can-i create deployments -n dev` |
| `kubectl auth can-i ... --as <user>` | Impersonate user to test | `kubectl auth can-i delete pods --as anna -n dev` |
| `kubectl auth can-i ... --as system:serviceaccount:<ns>:<sa>` | Test a ServiceAccount | `kubectl auth can-i list secrets --as system:serviceaccount:dev:ci -n dev` |
| `kubectl auth can-i --list` | List everything allowed in namespace | `kubectl auth can-i --list -n dev --as anna` |
| `kubectl auth can-i '*' '*'` | Am I cluster-admin? | `kubectl auth can-i '*' '*' --all-namespaces` |
| `kubectl create serviceaccount <name>` | Create SA | `kubectl create sa ci-deployer -n dev` |
| `kubectl create token <sa>` | Mint short-lived bound token | `kubectl create token ci-deployer -n dev --duration 30m` |
| `kubectl create role <name> --verb --resource` | Generate Role | `kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev` |
| `kubectl create clusterrole ...` | Generate ClusterRole | `kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes` |
| `kubectl create rolebinding <name> --role/--clusterrole --user/--group/--serviceaccount` | Bind in a namespace | `kubectl create rolebinding anna-edit --clusterrole=edit --user=anna -n dev` |
| `kubectl create clusterrolebinding ...` | Bind cluster-wide | `kubectl create clusterrolebinding ops-view --clusterrole=view --group=ops` |
| `kubectl get roles,rolebindings -n <ns>` | List namespaced RBAC | `kubectl get roles,rolebindings -n dev` |
| `kubectl get clusterroles,clusterrolebindings` | List cluster RBAC | `kubectl get clusterrolebindings -o wide` |
| `kubectl describe clusterrole <name>` | Show rules readably | `kubectl describe clusterrole view` |
| `kubectl get csr` | List certificate signing requests | `kubectl get csr` |
| `kubectl certificate approve <csr>` | Approve a CSR | `kubectl certificate approve anna` |
| `kubectl certificate deny <csr>` | Deny a CSR | `kubectl certificate deny anna` |
| `kubectl config set-credentials` | Add user creds to kubeconfig | `kubectl config set-credentials anna --client-certificate=anna.crt --client-key=anna.key --embed-certs=true` |
| `kubectl config set-context` | Add context | `kubectl config set-context anna@lab --cluster=lab --user=anna --namespace=dev` |
| `kubectl config view --minify --raw` | Dump current kubeconfig | `kubectl config view --minify --raw > admin.yaml` |
| `kubectl get sa <name> -o yaml` | Inspect SA | `kubectl get sa default -n dev -o yaml` |
| `openssl genrsa` / `openssl req -new` | Make key and CSR | see walkthrough |

Notes:
- `kubectl create role ... --resource=deployments.apps` – the `.apps` suffix sets the API group. Subresource: `--resource=pods/log`.
- `kubectl create rolebinding --serviceaccount=<namespace>:<name>` (colon form).
- `kubectl auth can-i` prints only `yes`/`no`. Add `-v=6` if you need to see the API call.
- `--as` needs the `impersonate` permission – you (the admin) have it; normal users do not.

## Examples / Walkthrough

Lab assumptions: a working cluster, admin kubeconfig, namespace `dev` and `prod`.

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl auth whoami
```

Expected output of `whoami` for the kubeadm admin: `Username kubernetes-admin`, `Groups [system:masters system:authenticated]`.

### Example 1: Feel the deny-by-default

```bash
kubectl auth can-i get pods --as anna -n dev
# no
kubectl get pods -n dev --as anna
# Error from server (Forbidden): pods is forbidden: User "anna" cannot list resource "pods" in API group "" in the namespace "dev"
```

**Read this error word by word** – it contains the whole fix:
- `User "anna"` → the subject for the binding
- `cannot list` → the verb
- `resource "pods"` → the resource
- `API group ""` → the apiGroups entry
- `in the namespace "dev"` → the namespace for the Role/RoleBinding

### Example 2: Create a user with a client certificate (end to end)

Goal: user `anna`, group `developers`, a kubeconfig that works, permission to read pods in `dev` only.

**Step 1 – private key and CSR.** `CN` = username, `O` = group:

```bash
mkdir -p ~/rbac-lab && cd ~/rbac-lab
openssl genrsa -out anna.key 2048
openssl req -new -key anna.key -out anna.csr -subj "/CN=anna/O=developers"
```

**Step 2 – submit a CertificateSigningRequest object** (the API server and its controller-manager hold the cluster CA key and sign for you; you never touch the CA private key):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: anna
spec:
  request: $(base64 -w0 < anna.csr)      # macOS: base64 < anna.csr | tr -d '\n'
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400               # 1 day. Min 600. Shorter = safer, no revocation exists
  usages:
  - client auth
EOF
kubectl get csr
# NAME   AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
# anna   5s    kubernetes.io/kube-apiserver-client   kubernetes-admin   24h                 Pending
```

**Step 3 – approve and fetch the signed certificate:**

```bash
kubectl certificate approve anna
kubectl get csr anna -o jsonpath='{.status.certificate}' | base64 -d > anna.crt
openssl x509 -in anna.crt -noout -subject -dates
# subject=CN=anna, O=developers
```

**Step 4 – build a kubeconfig for Anna** (separate file so you do not break your admin one):

```bash
export KUBECONFIG=$PWD/anna.kubeconfig
SERVER=$(kubectl --kubeconfig ~/.kube/config config view --minify -o jsonpath='{.clusters[0].cluster.server}')
kubectl --kubeconfig ~/.kube/config config view --minify --raw \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > ca.crt

kubectl config set-cluster lab --server="$SERVER" --certificate-authority=ca.crt --embed-certs=true
kubectl config set-credentials anna --client-certificate=anna.crt --client-key=anna.key --embed-certs=true
kubectl config set-context anna@lab --cluster=lab --user=anna --namespace=dev
kubectl config use-context anna@lab

kubectl auth whoami          # Username anna, Groups [developers system:authenticated]
kubectl get pods             # Forbidden – authenticated, but no RBAC rule yet
unset KUBECONFIG             # back to admin
```

Authentication works (we are `anna`), authorization denies (no binding). That is the exact 401 vs 403 split.

**Step 5 – grant read-only access to pods in `dev`:**

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding anna-pod-reader --role=pod-reader --user=anna -n dev
```

Declarative equivalent (this is what you commit to Git):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: anna-pod-reader
  namespace: dev
subjects:
- kind: User
  name: anna
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Step 6 – verify as the admin with impersonation, no kubeconfig switch needed:**

```bash
kubectl auth can-i list pods -n dev --as anna          # yes
kubectl auth can-i list pods -n prod --as anna         # no  (Role is namespaced)
kubectl auth can-i delete pods -n dev --as anna        # no  (verb not granted)
kubectl auth can-i get pods/log -n dev --as anna       # no  (subresource separate!)
kubectl auth can-i --list -n dev --as anna
```

### Example 3: Subresources – pod-reader cannot read logs

```bash
kubectl run web --image=nginx -n dev
kubectl logs web -n dev --as anna
# Error ... User "anna" cannot get resource "pods/log" in API group "" in the namespace "dev"
```

Fix: extend the Role:

```bash
kubectl patch role pod-reader -n dev --type=json \
  -p '[{"op":"add","path":"/rules/-","value":{"apiGroups":[""],"resources":["pods/log"],"verbs":["get"]}}]'
kubectl logs web -n dev --as anna     # works
```

### Example 4: Group binding with a built-in ClusterRole

Instead of per-user bindings, grant the whole `developers` group edit rights in `dev`:

```bash
kubectl create rolebinding developers-edit --clusterrole=edit --group=developers -n dev
kubectl auth can-i create deployments -n dev --as anna --as-group developers   # yes
kubectl auth can-i create deployments -n prod --as anna --as-group developers  # no
```

Note: with `--as anna` alone, the group is **not** added automatically; add `--as-group developers`. With the real certificate (O=developers) the group comes automatically.

`edit` ClusterRole + `RoleBinding` = permissions only in `dev`. Same ClusterRole bound with a ClusterRoleBinding would be everywhere – check which kind you typed.

### Example 5: ServiceAccount for a deployer (CI/CD) with scoped permissions

```bash
kubectl create sa ci-deployer -n dev

kubectl create role deployer -n dev \
  --verb=get,list,watch,create,update,patch \
  --resource=deployments.apps,services
kubectl create rolebinding ci-deployer-binding -n dev \
  --role=deployer --serviceaccount=dev:ci-deployer

kubectl auth can-i patch deployments.apps -n dev --as system:serviceaccount:dev:ci-deployer   # yes
kubectl auth can-i delete deployments.apps -n dev --as system:serviceaccount:dev:ci-deployer  # no
kubectl auth can-i list secrets -n dev --as system:serviceaccount:dev:ci-deployer             # no

# short-lived token for the pipeline (kubectl 1.24+)
kubectl create token ci-deployer -n dev --duration 1h
```

Use that token in a kubeconfig:

```bash
TOKEN=$(kubectl create token ci-deployer -n dev --duration 1h)
kubectl --server="$SERVER" --certificate-authority=ca.crt --token="$TOKEN" -n dev get deployments
```

Jargon check: that is **bearer token auth** – a plain `Authorization: Bearer <jwt>` header.

### Example 6: Pod that calls the API with its own token (curl)

```bash
kubectl create sa pod-lister -n dev
kubectl create role list-pods -n dev --verb=list --resource=pods
kubectl create rolebinding pod-lister-binding -n dev --role=list-pods --serviceaccount=dev:pod-lister

kubectl run api-client -n dev --image=curlimages/curl --overrides='{"spec":{"serviceAccountName":"pod-lister"}}' \
  --command -- sleep 3600

kubectl exec -n dev api-client -- sh -c '
SA=/var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat $SA/token)
NS=$(cat $SA/namespace)
curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/$NS/pods | head -20
'
```

Result: JSON `PodList`. Now try something not granted:

```bash
kubectl exec -n dev api-client -- sh -c '
SA=/var/run/secrets/kubernetes.io/serviceaccount
curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" \
  https://kubernetes.default.svc/api/v1/namespaces/dev/secrets'
```

Returns a `Status` object: `"reason":"Forbidden"`, `"code":403`, and the message naming `system:serviceaccount:dev:pod-lister`. Nice demonstration that Pods authenticate exactly like users.

Decode the JWT payload to show students the bound claims (audience, expiry, pod binding):

```bash
kubectl exec -n dev api-client -- sh -c 'cut -d. -f2 /var/run/secrets/kubernetes.io/serviceaccount/token' \
  | tr '_-' '/+' | base64 -d 2>/dev/null; echo
```

Look for `exp`, `aud`, `sub: system:serviceaccount:dev:pod-lister`, and `kubernetes.io` → `pod` name.

### Example 7: Turn the token off

```bash
kubectl run no-token -n dev --image=busybox --command --overrides='{"spec":{"automountServiceAccountToken":false}}' -- sleep 3600
kubectl exec -n dev no-token -- ls /var/run/secrets/kubernetes.io/serviceaccount
# ls: /var/run/secrets/kubernetes.io/serviceaccount: No such file or directory
```

Or on the SA (default for every Pod using it):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app
  namespace: dev
automountServiceAccountToken: false
```

Pod-level `automountServiceAccountToken: true` overrides the SA value, so an app that really needs the API can opt back in.

### Example 8: ClusterRole + ClusterRoleBinding for cluster-scoped resources

Nodes and PersistentVolumes have no namespace. A `Role` cannot cover them.

```bash
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create clusterrolebinding anna-node-reader --clusterrole=node-reader --user=anna
kubectl auth can-i list nodes --as anna     # yes
kubectl auth can-i list pods -n dev --as anna  # still yes, via the earlier RoleBinding
```

### Example 9: Same ClusterRole, two scopes

```bash
kubectl create clusterrole pod-reader-cluster --verb=get,list,watch --resource=pods

# Only in prod:
kubectl create rolebinding bob-prod-pods --clusterrole=pod-reader-cluster --user=bob -n prod
# Everywhere:
kubectl create clusterrolebinding carol-all-pods --clusterrole=pod-reader-cluster --user=carol

kubectl auth can-i list pods -n prod --as bob   # yes
kubectl auth can-i list pods -n dev  --as bob   # no
kubectl auth can-i list pods -n dev  --as carol # yes
kubectl auth can-i list pods --all-namespaces --as bob   # no
kubectl auth can-i list pods --all-namespaces --as carol # yes
```

### Example 10: Restrict to a single named object

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: update-one-deployment
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  resourceNames: ["web"]
  verbs: ["get", "patch", "update"]
```

`kubectl auth can-i patch deployment/web -n dev --as <user>` → yes; `deployment/api` → no. Note: `list` of all deployments is still denied because `list` was not granted.

### Example 11: Escalation prevention in action

```bash
# anna can only read pods. Try to give herself more:
kubectl create rolebinding steal --clusterrole=edit --user=anna -n dev --as anna
# Error ... forbidden: user "anna" (groups=["system:authenticated"]) is attempting to grant RBAC permissions not currently held
```

Even if you give a user `create` on `rolebindings`, they can only bind permissions they already own – unless you also give the `bind` verb on the target role.

### Example 12: Audit who has what

Pure `kubectl` (no extra tools):

```bash
# Every binding that points at cluster-admin:
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name + " -> " + ([.subjects[]? | .kind+"/"+.name] | join(","))'

# Who is in system:masters? (Not visible in RBAC: certificates only)
# Every RoleBinding in all namespaces with their subjects:
kubectl get rolebindings,clusterrolebindings -A -o wide

# Roles that allow wildcard verbs:
kubectl get roles,clusterroles -A -o json | \
  jq -r '.items[] | select(.rules[]?.verbs[]? == "*") | .kind+"/"+.metadata.name'

# What can this ServiceAccount do everywhere?
kubectl auth can-i --list --as system:serviceaccount:dev:ci-deployer -n dev
```

Community tools (mention, optional demo):

| Tool | What it answers | Example |
|---|---|---|
| `kubectl-who-can` (Aqua) | Who can do X? | `kubectl who-can delete pods -n dev` |
| `rakkess` | Matrix of resources × verbs for one subject | `kubectl access-matrix -n dev --as anna` |
| `rbac-tool` (Alcide/InsightCloudSec) | Visualize, lookup, generate policies | `kubectl rbac-tool lookup anna`, `kubectl rbac-tool who-can get secrets` |
| `kubectl-tree`, `rback` | Graph subject → binding → role | Generates Graphviz |
| `KubiScan` / `Kubescape` | Flags risky roles | Scan results list overly-permissive bindings |

Install tools via `krew` (kubectl plugin manager): `kubectl krew install who-can access-matrix rbac-tool`.

### Example 13: Clean up

```bash
kubectl delete csr anna
kubectl delete role,rolebinding --all -n dev
kubectl delete sa ci-deployer pod-lister web-app -n dev --ignore-not-found
kubectl delete clusterrole node-reader pod-reader-cluster monitoring-view --ignore-not-found
kubectl delete clusterrolebinding anna-node-reader carol-all-pods --ignore-not-found
kubectl delete pod web api-client no-token -n dev --ignore-not-found
rm -rf ~/rbac-lab
```

## Common Pitfalls

- **Expecting a "create user" command** — Kubernetes has no User object. Users come from certificates or an external IdP. Only ServiceAccounts exist as objects.
- **Confusing 401 and 403** — 401 = identity not recognised (bad/expired cert or token); 403 = identity fine, no RBAC rule. Do not debug RBAC for a 401.
- **Putting the wrong `apiGroups`** — `deployments` need `["apps"]`, not `[""]`. Result: rule silently does nothing. Check with `kubectl api-resources`.
- **Forgetting subresources** — `pods` does not include `pods/log`, `pods/exec`, `pods/portforward`. Each needs its own rule.
- **Binding in the wrong namespace** — a RoleBinding only works in its own namespace. If the Role is in `dev` and the binding in `default`, nothing happens (the binding looks for the Role in its own namespace).
- **Missing `namespace` on a ServiceAccount subject** — `kind: ServiceAccount` requires `namespace`. Without it the binding is rejected or matches nothing.
- **Using `kubectl create rolebinding --serviceaccount=name`** — must be `namespace:name`.
- **Role pointing at a ClusterRoleBinding** — not allowed; `ClusterRoleBinding` only references ClusterRoles.
- **Editing `roleRef` of an existing binding** — immutable. Delete and recreate.
- **Binding `cluster-admin` "just to make it work"** — the most common real-world compromise path. Find the failing verb in the 403 message and grant just that.
- **Granting `list`/`get` on secrets lightly** — both expose secret values. `view` intentionally excludes Secrets.
- **Giving `create pods` to a "limited" user** — they can mount any Secret or ServiceAccount token in the namespace into their own pod. Treat as near-admin unless admission policies restrict it (lesson 48).
- **Giving `pods/exec`** — equals shell inside workloads; whoever owns the pod's data/token is reachable.
- **Long-lived client certificates for people** — no revocation. Use short `expirationSeconds`, or OIDC.
- **Testing as yourself** — `kubectl auth can-i` without `--as` tests the admin. Always add `--as` and, for groups, `--as-group`.
- **`--as anna` without `--as-group`** — group-based bindings are not applied during impersonation unless you pass the group.
- **Legacy token Secrets** — creating `kubernetes.io/service-account-token` Secrets by hand gives a never-expiring token. Prefer `kubectl create token`.
- **Assuming `default` SA has no power** — on its own it has none, but admins often bind it to roles "temporarily" and then every Pod in the namespace inherits them.
- **Deleting a Role and expecting bindings to vanish** — bindings stay and dangle; recreating a Role with the same name re-activates them. Clean up both.
- **Editing `system:*` roles** — they are overwritten on API server restart (auto-reconcile). Create your own roles.
- **Aggregated roles surprises** — a CRD installing a ClusterRole labeled `aggregate-to-edit` silently expands what `edit` users can do.

## FAQ

**Q: What is the difference between authentication and authorization in Kubernetes?**
A: Authentication proves who you are (certificate, token, OIDC). Authorization decides what that identity may do (RBAC). Failing the first gives 401, failing the second gives 403.

**Q: How do I create a user in Kubernetes?**
A: You do not create one as an object. Issue the person a credential the cluster trusts: either a client certificate signed by the cluster CA (via a `CertificateSigningRequest`, with `CN`=username and `O`=group) or an OIDC token from your identity provider. Then give that username or group permissions using a RoleBinding or ClusterRoleBinding.

**Q: What is a ServiceAccount and when do I need one?**
A: It is an identity for a Pod. You need a custom one whenever a Pod calls the Kubernetes API (CI runner, operator, ingress controller, monitoring agent). For ordinary apps that never call the API, you do not need any permissions and should disable token mounting.

**Q: What is the difference between a Role and a ClusterRole?**
A: A Role exists in one namespace and only grants permissions on namespaced resources there. A ClusterRole is cluster-level: it can grant access to cluster-scoped resources (nodes, PVs, namespaces), non-resource URLs, or act as a reusable rule set that you attach per namespace with a RoleBinding.

**Q: If I bind a ClusterRole with a RoleBinding, is it cluster-wide?**
A: No. Scope comes from the binding. A RoleBinding limits the ClusterRole's permissions to the binding's namespace. Only a ClusterRoleBinding makes them cluster-wide.

**Q: Can RBAC deny something?**
A: No. RBAC is allow-only and additive. If any binding allows the action, it is allowed. To "deny", remove the permission from every Role the subject gets (or use admission policy – lesson 48/50).

**Q: Why does `kubectl get pods` work but `kubectl logs` gives Forbidden?**
A: `logs` uses the subresource `pods/log`. Granting `pods` does not include subresources. Add a rule for `pods/log` with verb `get`.

**Q: Why is my Role not working even though the rule looks right?**
A: The most frequent causes: wrong `apiGroups`, missing subresource, binding in a different namespace, wrong subject name or kind, or testing without `--as`. Run `kubectl auth can-i <verb> <resource> --as <subject> -n <ns>`, then `kubectl describe rolebinding` to see subjects and roleRef.

**Q: How do I see everything a user can do?**
A: `kubectl auth can-i --list --as <user> -n <namespace>` (per namespace). Cluster-wide, use `kubectl access-matrix` (rakkess) or `kubectl rbac-tool lookup <user>`.

**Q: How do I find who can read secrets?**
A: `kubectl who-can get secrets -n <ns>` (kubectl-who-can plugin) or `kubectl rbac-tool who-can get secrets`.

**Q: How do I revoke a user's certificate?**
A: You cannot revoke a single cert in Kubernetes (no CRL/OCSP). Remove their RoleBindings (the cert stays valid but powerless), keep expirations short, or rotate the cluster CA (disruptive). That is why OIDC with short-lived tokens is preferred for humans.

**Q: What does `system:masters` do?**
A: Any identity in that group is hard-wired to `cluster-admin`, bypassing RBAC evaluation. Never put people or automation in it; do not create certificates with `O=system:masters`.

**Q: Why did my Pod's token stop being a Secret?**
A: Since Kubernetes 1.24 tokens are issued on demand via the TokenRequest API, bound to the Pod, audience-limited, and expiring. Nothing is stored as a Secret. Use `kubectl create token <sa>` when you need one outside a Pod.

**Q: How long does `kubectl create token` last?**
A: Default 1 hour (the server may cap it). Set `--duration 30m` or longer, up to the API server limit (often 24h+).

**Q: Should my web app use the default ServiceAccount?**
A: Only if it needs no permissions, and even then set `automountServiceAccountToken: false` so no token is available to attackers. Best: dedicated SA per app.

**Q: What are `bind` and `escalate` verbs?**
A: They let a user bypass escalation prevention. `escalate` on roles = may write a Role with permissions you do not hold; `bind` on a role = may create bindings to that role even if you do not hold its permissions. Grant almost never.

**Q: Is `view` safe to hand to everyone?**
A: It is read-only and excludes Secrets and RBAC objects, so it is the safest built-in. It still exposes configuration, environment variables in pod specs (never put secrets there), and cluster topology.

**Q: Where do RBAC objects live?**
A: In etcd, like every API object. Keep them in Git as YAML and apply via CI/GitOps, so changes are reviewed.

**Q: How does a Group get into a request?**
A: From the client certificate `O` fields, or from the OIDC `groups` claim, or from the ServiceAccount identity (`system:serviceaccounts`, `system:serviceaccounts:<ns>`). Everyone authenticated also gets `system:authenticated`.

**Q: What is the anonymous user?**
A: `system:anonymous` in group `system:unauthenticated`, used when no credentials are sent and `--anonymous-auth` is true (default). It has almost no permissions, only a few health/version endpoints. Check with `kubectl get clusterrolebindings -o wide | grep -i anonymous`; any binding to it is a red flag. kube-bench (lesson 49) checks this.

**Q: Can a ServiceAccount from namespace A be used in namespace B?**
A: A Pod can only reference an SA in its own namespace. But a RoleBinding in B can name an SA from A as subject (with `namespace: A`), giving it rights in B.

**Q: How do I give Helm/Argo/Flux limited rights instead of cluster-admin?**
A: Create a ServiceAccount, a namespaced Role covering the object kinds they deploy, and a RoleBinding per target namespace. Cluster-scoped objects (CRDs, namespaces) need a small ClusterRole.

## Practice / Exercise

**Core (all students):**

1. **Identity and deny-by-default.** Run `kubectl auth whoami`. Then run `kubectl auth can-i list pods -n dev --as intern`. Read the result; list which of the five parts of a Forbidden message you can identify when you run `kubectl get pods -n dev --as intern`.
2. **Create a user with a certificate.** Generate a key and CSR for `CN=maria,O=qa`; submit a `CertificateSigningRequest` with `expirationSeconds: 3600`; approve; extract the cert; build `maria.kubeconfig`. Show `kubectl auth whoami` with that kubeconfig.
3. **Role and RoleBinding.** In namespace `dev`, give `maria` permission to `get,list,watch` pods and `get` on `pods/log`. Prove with `can-i` that she can read logs in `dev` but not in `prod`, and cannot delete pods.
4. **Group binding.** Replace the user binding with a RoleBinding to group `qa` using the built-in `view` ClusterRole. Prove it using `--as maria --as-group qa`, and then with the real kubeconfig.
5. **ServiceAccount.** Create SA `reporter` in `dev` with a Role allowing only `get,list` on `deployments.apps` and `services`. Mint a token with `kubectl create token reporter -n dev --duration 15m` and use it with `kubectl --token=...` to list deployments. Confirm that listing secrets is denied.
6. **Pod using its token.** Run a `curlimages/curl` Pod with `serviceAccountName: reporter` and call the API with `curl` to list deployments in `dev`. Then call the same URL for `secrets` and record the HTTP status code.
7. **Disable the token.** Run a Pod with `automountServiceAccountToken: false` and prove the token directory does not exist.

**Stretch:**

1. **Cluster-scoped.** Create a ClusterRole and ClusterRoleBinding that let group `qa` list nodes and namespaces but nothing else. Verify with `can-i --list`.
2. **One ClusterRole, two scopes.** Bind a custom ClusterRole `deployment-manager` to user `dave` in `dev` via RoleBinding and to user `erin` everywhere via ClusterRoleBinding. Show the difference with `can-i ... --all-namespaces`.
3. **resourceNames.** Allow a ServiceAccount to `patch` only the Deployment named `web`. Show it fails on another Deployment.
4. **Escalation attempt.** As `maria` (using her kubeconfig, after giving her `create` on `rolebindings` in `dev`), try binding `edit` to herself. Explain the error. Then add the `bind` verb for `clusterroles` with `resourceNames: ["edit"]` and observe the change. Remove it afterwards and explain why this is dangerous.
5. **Audit.** Using only `kubectl` and `jq`, list all ClusterRoleBindings that reference `cluster-admin` and every Role/ClusterRole containing a `*` verb. Install two krew plugins (`who-can`, `access-matrix`) and answer: who can `get secrets` in `kube-system`? What can `reporter` do?
6. **Aggregation.** Create a ClusterRole labeled `rbac.authorization.k8s.io/aggregate-to-view: "true"` for a made-up CRD resource, then confirm via `kubectl get clusterrole view -o yaml` that the rule was merged.
7. **Least-privilege rewrite.** Take this over-broad role and rewrite it for an app that only reads one ConfigMap and lists its own Pods:
   ```yaml
   rules:
   - apiGroups: ["*"]
     resources: ["*"]
     verbs: ["*"]
   ```

**Checklist to hand in:** YAML for Role, RoleBinding, SA; the `can-i` outputs proving allowed and denied cases; the curl output with a 200 and a 403.

## Further Reading

- Kubernetes docs – Controlling Access to the API: https://kubernetes.io/docs/concepts/security/controlling-access/
- Authenticating (all methods, ServiceAccount tokens, OIDC): https://kubernetes.io/docs/reference/access-authn-authz/authentication/
- Using RBAC Authorization (rules, aggregation, escalation prevention): https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Managing Service Accounts: https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/
- Certificates and CertificateSigningRequests: https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/
- Configure Service Accounts for Pods (projected tokens): https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
- RBAC Good Practices: https://kubernetes.io/docs/concepts/security/rbac-good-practices/
- kubectl-who-can: https://github.com/aquasecurity/kubectl-who-can
- rakkess (access-matrix): https://github.com/corneliusweig/rakkess
- rbac-tool: https://github.com/alcideio/rbac-tool
- krew plugin manager: https://krew.sigs.k8s.io/

# Lesson 43: Secrets Management – External Secrets Operator (ESO)

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-42 (cluster architecture, kubectl, Pods/Deployments, Services, Ingress, PV/PVC, ConfigMaps and native Secrets). Student can create a native Secret, consume it as env var or file, and read `kubectl describe` output. A Vault server is **pre-configured** by the instructor; students do not install or initialise Vault in this lesson.

## Learning Objectives

By end of lesson student can:
- List the limitations of native Kubernetes Secrets (base64 is not encryption, etcd storage, no rotation, no central audit, Git-commit problem)
- Explain the External Secrets Operator architecture: controller, webhook, cert-controller and the CRDs `SecretStore`, `ClusterSecretStore`, `ExternalSecret`
- Explain the data flow: Vault -> ESO controller -> native Secret -> Pod
- Describe Vault KV v2 and the Kubernetes auth method (policy, role, ServiceAccount token login) at concept level
- Write a `SecretStore` for the Vault provider using ServiceAccount (Kubernetes auth) and AppRole auth
- Decide between `SecretStore` (namespaced) and `ClusterSecretStore` (cluster-wide)
- Write an `ExternalSecret` with `data` (key mapping), `dataFrom` (pull all keys), `refreshInterval`, `secretStoreRef` and `target`
- Verify sync with `kubectl get externalsecret`, `describe`, status conditions, and decode the resulting Secret
- Force a re-sync, explain how rotation reaches the Pod, and troubleshoot `SecretSyncedError`, `403 permission denied`, wrong path/property errors

## Topics

- Why ESO: limitations of native K8s Secrets (base64, etcd, no rotation)
- ESO architecture: operator, SecretStore, ClusterSecretStore, ExternalSecret CRDs
- Pre-configured Vault: KV v2 engine, Kubernetes auth method (concept walkthrough)
- SecretStore: Vault provider config, auth (serviceAccount token, approle)
- ExternalSecret: data mapping, refreshInterval, secretStoreRef; verify secret sync

## Concepts

### Recap: what lesson 42 gave us and where it hurts

Lesson 42 showed that a native `Secret` is an API object holding key/value pairs that Pods consume as env vars (`secretKeyRef`, `envFrom`) or files (volume). It works, but production use exposes five weaknesses:

| Weakness | What it means in practice |
|---|---|
| **base64 is encoding, not encryption** | `echo cGFzc3dvcmQ= \| base64 -d` reveals it. Anyone who can `kubectl get secret -o yaml` can read it. |
| **Stored in etcd** | By default etcd holds Secrets as plain data on disk. Encryption at rest is an optional API-server setting (`EncryptionConfiguration`) that many clusters never enable. An etcd backup is a backup of every password. |
| **No rotation** | Nothing changes a Secret value on a schedule. Someone must edit it and restart Pods. Passwords live for years. |
| **Source of truth lives in Git or in someone's shell history** | To create the Secret declaratively you need the YAML. Committing it to Git (even base64) leaks credentials. Creating it with `kubectl create secret` leaves no record and no review. |
| **No central audit / access policy** | Kubernetes RBAC controls *who reads the Secret object*, but there is no per-secret history, no lease, no "who read this at 03:12" as a first-class feature. Each cluster has its own copy, so ten clusters means ten copies to keep in sync. |

The industry answer: keep secrets in a dedicated **secret manager** (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, 1Password, ...) and have the cluster *fetch* them at runtime. The open-source tool that does the fetching is the **External Secrets Operator (ESO)**.

### What ESO is

ESO is a Kubernetes **operator** (a controller plus CRDs, lesson 34 reconciliation loop idea). You declare *what* you want ("the key `db/postgres` from Vault, as a Secret called `pg-creds`"); ESO continuously makes the cluster match.

Key idea: **ESO does not replace Kubernetes Secrets, it manufactures them.** The application stays unchanged and still reads a normal `Secret`. The difference is where the data originates and who owns the Secret object.

```
 ┌───────────────┐   read    ┌──────────────────┐  create/update   ┌───────────────┐  mount/env  ┌─────┐
 │  Vault (KV v2)│ <──────── │  ESO controller  │ ───────────────> │ native Secret │ ──────────> │ Pod │
 │ source of     │  auth     │ (reconcile loop) │  (owned by the   │ (in etcd)     │             │     │
 │ truth         │           │                  │   ExternalSecret)│               │             │     │
 └───────────────┘           └──────────────────┘                  └───────────────┘             └─────┘
        ^                            ^
        │ policy + audit log         │ reads CRDs: SecretStore, ExternalSecret
```

What this buys:
- **Git-safe**: you commit `ExternalSecret` YAML, which contains only *references* (path, key names), never values. GitOps friendly.
- **One source of truth**: change the value in Vault, every cluster syncs.
- **Rotation**: `refreshInterval` re-reads the value periodically.
- **Audit and policy in Vault**: who/what read which path is logged centrally; Vault policies limit which cluster identity can read which path.
- **Provider-agnostic**: same `ExternalSecret` shape for Vault, AWS, Azure, GCP and about 30 more providers; only the `SecretStore` differs.

What it does **not** fix: the resulting native Secret still sits in etcd and is still readable by anyone with RBAC `get secrets` in that namespace. ESO reduces *sprawl and manual handling*, not etcd exposure. Pair it with etcd encryption at rest and tight RBAC (lesson 47).

### ESO components

Installed as three Deployments in one namespace (commonly `external-secrets`):

| Component | Role |
|---|---|
| **controller** (`external-secrets`) | The reconciler. Watches `ExternalSecret`, `SecretStore`, `ClusterSecretStore`; talks to the provider API; writes Secrets. |
| **webhook** (`external-secrets-webhook`) | Validating webhook for the CRDs (rejects malformed objects at apply time) and does conversion between CRD versions. |
| **cert-controller** | Creates and rotates the TLS certificate the webhook needs. |

CRDs (API group `external-secrets.io`):

| CRD | Scope | Purpose |
|---|---|---|
| `SecretStore` | Namespaced | "How to connect and authenticate to a provider." Usable only by `ExternalSecret` objects **in the same namespace**. |
| `ClusterSecretStore` | Cluster | Same thing, but any namespace may reference it. |
| `ExternalSecret` | Namespaced | "Which remote keys to fetch, and what native Secret to produce." |
| `ClusterExternalSecret` | Cluster | Stamps the same `ExternalSecret` into many namespaces by label selector (mention only). |
| `PushSecret` | Namespaced | Reverse direction: push a K8s Secret *into* the provider (mention only). |

Rule of thumb: **SecretStore = the connection, ExternalSecret = the request.** One store, many requests.

#### SecretStore vs ClusterSecretStore

| | SecretStore | ClusterSecretStore |
|---|---|---|
| Scope | One namespace | Whole cluster |
| Who may reference it | `ExternalSecret` in same namespace | `ExternalSecret` in any namespace (unless `conditions` restrict) |
| Auth credentials live in | Same namespace | Namespace named explicitly in `serviceAccountRef.namespace` / `secretRef.namespace` |
| Typical use | Team owns its own Vault role and namespace | Platform team provides one shared store |
| Risk | Lower blast radius | Any namespace can request anything the store's Vault role may read; limit with `conditions.namespaces` and a narrow Vault policy |

Team pattern: platform team creates a `ClusterSecretStore` bound to a Vault role whose policy covers only `secret/data/<cluster>/*`; app teams create `ExternalSecret` objects only.

### Vault in 10 minutes (instructor walkthrough – already configured)

Vault is pre-installed and configured for the class. The lesson shows **what exists** so the ESO fields make sense. Students do not run the Vault admin commands; the instructor shows them as reference.

**Vault** is a secret manager: an HTTP API in front of encrypted storage, with authentication, policies, leases and audit logging.

**Secrets engine = a mounted plug-in that stores/generates secrets.** The one we use: **KV (key-value) version 2**, mounted at path `secret/`. KV v2 differs from v1:
- **Versioned**: every write creates a new version; old versions can be read or destroyed. (This is why `ExternalSecret` can pin a `version`.)
- **API path has an extra `data/` segment**: the logical path `secret/myapp/db` is addressed over the API as `secret/data/myapp/db`. CLI hides this, API and **policies** do not. This is the number-one source of 403 errors.

Instructor reference (not run by students):

```bash
# write a secret (KV v2): CLI inserts the data/ segment automatically
vault kv put secret/myapp/db username=appuser password='S3cr3t-DemoOnly'

# read it back
vault kv get secret/myapp/db
vault kv get -field=password secret/myapp/db
vault kv get -version=1 secret/myapp/db

# metadata and versions
vault kv metadata get secret/myapp/db

# update one key without losing others
vault kv patch secret/myapp/db password='N3w-DemoOnly'
```

Values above are placeholders for the demo. Never put real production credentials in lesson material, chat or Git.

**Policy** = allow-list of paths and capabilities (HCL):

```hcl
# policy: myapp-read
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
path "secret/metadata/myapp/*" {
  capabilities = ["list", "read"]
}
```

Note `secret/data/...` in the policy. Policy written against `secret/myapp/*` matches nothing in KV v2.

**Auth method = how a client proves identity to Vault and gets a token with policies attached.**

#### Kubernetes auth method (concept)

Lets a Pod/controller log in to Vault **using its ServiceAccount token** instead of a password. No static secret to distribute.

```
1. ESO builds a token for ServiceAccount "vault-auth" (TokenRequest API, short-lived JWT)
2. ESO -> Vault:  POST /v1/auth/kubernetes/login   { role: "eso-role", jwt: "<SA token>" }
3. Vault -> Kubernetes API: TokenReview "is this JWT valid, and who is it?"
4. Kubernetes answers: yes, ServiceAccount vault-auth in namespace external-secrets
5. Vault checks role "eso-role": bound SA name + namespace match? -> issues Vault token with policy "myapp-read"
6. ESO uses the Vault token to read secret/data/myapp/db
```

Admin-side setup, instructor reference only:

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443"

vault write auth/kubernetes/role/eso-role \
    bound_service_account_names=vault-auth \
    bound_service_account_namespaces=external-secrets \
    policies=myapp-read \
    ttl=1h
```

Three things must line up or login fails: **role name**, **bound ServiceAccount name**, **bound namespace**. The ServiceAccount itself is an ordinary object (lesson 47 covers ServiceAccounts in depth; here it is only an identity):

```bash
kubectl create namespace external-secrets
kubectl -n external-secrets create serviceaccount vault-auth
```

#### AppRole auth method (concept)

For systems that cannot present a ServiceAccount token (other clusters, CI, VMs). Two values form the credential pair:
- **role_id** – like a username, not very secret
- **secret_id** – like a password, secret, can be single-use/TTL-bound

ESO stores `secret_id` in a native Secret and references it; ESO sends both to `auth/approle/login`. Trade-off: AppRole needs one **bootstrap secret** in the cluster (the `secret_id`), Kubernetes auth needs none. Prefer Kubernetes auth when Vault can reach the cluster API; use AppRole otherwise.

| | Kubernetes auth | AppRole |
|---|---|---|
| Credential | Short-lived SA JWT, auto-minted | role_id + secret_id |
| Bootstrap secret in cluster | None | secret_id Secret |
| Rotation burden | None | Rotate secret_id |
| Needs Vault -> cluster API connectivity | Yes (TokenReview) | No |
| Best for | In-cluster ESO | Cross-cluster, CI, no API reachability |

### Installing ESO

Helm is taught fully in lessons 53-55; here it is used only as an installer.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace \
  --set installCRDs=true

kubectl -n external-secrets get pods
kubectl get crd | grep external-secrets.io
```

Expected: three Pods Running (controller, webhook, cert-controller) and CRDs such as `externalsecrets.external-secrets.io`, `secretstores.external-secrets.io`, `clustersecretstores.external-secrets.io`. In class the instructor usually pre-installs it; students only verify.

### SecretStore: Vault provider

#### Kubernetes auth (ServiceAccount)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: demo
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"   # Vault address reachable from the cluster
      path: "secret"                          # mount path of the KV engine
      version: "v2"                           # KV version
      auth:
        kubernetes:
          mountPath: "kubernetes"             # where auth method is enabled in Vault
          role: "eso-role"                    # Vault role name
          serviceAccountRef:
            name: "vault-auth"                # SA in the SAME namespace as this SecretStore
```

Field walk-through:

| Field | Meaning | Common mistake |
|---|---|---|
| `server` | Vault URL. Use in-cluster Service DNS (lesson 37) or external URL. | `http` vs `https`; wrong port; DNS from another namespace needs full `svc.namespace` |
| `path` | The **mount** name of the KV engine (`secret`), *not* a secret path | Putting `secret/data` here |
| `version` | `v1` or `v2` | Default is `v2`, but be explicit |
| `auth.kubernetes.mountPath` | Name where `vault auth enable kubernetes` mounted it | Custom mount names (`k8s-prod`) not matched |
| `auth.kubernetes.role` | Vault role | Role name typo, or role not bound to this SA/namespace |
| `serviceAccountRef.name` | Identity ESO logs in as | SA does not exist in the store's namespace |
| `caProvider` / `caBundle` | Trust a private CA for `https` Vault | Missing -> `x509: certificate signed by unknown authority` |
| `namespace` (Vault Enterprise) | Vault namespace header | Not used on OSS Vault |

A `SecretStore` is **namespaced**: SA `vault-auth` must exist in namespace `demo`, and Vault role `eso-role` must be bound to `bound_service_account_namespaces=demo`.

#### ClusterSecretStore variant

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-global
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-role"
          serviceAccountRef:
            name: "vault-auth"
            namespace: "external-secrets"   # REQUIRED for cluster-scoped: which namespace the SA lives in
  conditions:
    - namespaces: ["demo", "staging"]       # optional allow-list of consuming namespaces
```

Differences: `kind`, no `metadata.namespace`, `serviceAccountRef.namespace` is mandatory, optional `conditions` restricts who can use it.

#### AppRole auth

```bash
# bootstrap Secret holding the secret_id (value comes from Vault admin; placeholder here)
kubectl -n demo create secret generic vault-approle \
  --from-literal=secret-id='PLACEHOLDER-SECRET-ID'
```

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-approle-backend
  namespace: demo
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        appRole:
          path: "approle"                    # auth mount
          roleId: "11111111-2222-3333-4444-555555555555"   # placeholder, not secret
          secretRef:
            name: "vault-approle"            # Secret holding secret_id
            key: "secret-id"
```

#### Checking a store

```bash
kubectl -n demo get secretstore
# NAME            AGE   STATUS   CAPABILITIES   READY
# vault-backend   10s   Valid    ReadWrite      True
kubectl -n demo describe secretstore vault-backend
```

`STATUS Valid / READY True` means ESO authenticated successfully. `InvalidProviderConfig` or `ReadyFalse` shows the login error in `describe` events. **Always get the store to `Valid` before writing any ExternalSecret.**

### ExternalSecret

An `ExternalSecret` says: *"using store X, fetch these remote values, and keep a native Secret named Y in sync."*

#### Anatomy

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-db
  namespace: demo
spec:
  refreshInterval: 1h                 # how often to re-read Vault
  secretStoreRef:
    name: vault-backend               # which store
    kind: SecretStore                 # or ClusterSecretStore
  target:
    name: myapp-db-secret             # name of the native Secret ESO creates (default = ExternalSecret name)
    creationPolicy: Owner             # Owner (default) | Orphan | Merge | None
  data:                               # explicit key mapping
    - secretKey: DB_USER              # key name in the K8s Secret
      remoteRef:
        key: myapp/db                 # path under the mount, WITHOUT secret/ and WITHOUT data/
        property: username            # field inside the Vault secret
    - secretKey: DB_PASSWORD
      remoteRef:
        key: myapp/db
        property: password
```

Apply and verify:

```bash
kubectl apply -f externalsecret.yaml
kubectl -n demo get externalsecret myapp-db
# NAME      STORE           REFRESH INTERVAL   STATUS         READY
# myapp-db  vault-backend   1h                 SecretSynced   True

kubectl -n demo get secret myapp-db-secret
kubectl -n demo get secret myapp-db-secret -o jsonpath='{.data.DB_USER}' | base64 -d
```

#### Path rules (KV v2)

| You want Vault logical path | `remoteRef.key` | `property` |
|---|---|---|
| `secret/myapp/db` field `password` | `myapp/db` | `password` |
| `secret/myapp/db` (whole secret) | `myapp/db` | omit (returns JSON of all fields) |
| nested JSON field `a.b` | `myapp/db` | `a.b` |

Do **not** write `secret/myapp/db` (mount name is already in the store's `path`) and do **not** write `secret/data/myapp/db` (ESO inserts `data/` itself for `version: v2`). Wrong path is the top "everything looks right but it fails" cause.

Optional on `remoteRef`: `version: "2"` pins a KV v2 version; omit for latest.

#### data vs dataFrom

| | `data` | `dataFrom` |
|---|---|---|
| What | Map specific remote fields to specific Secret keys | Pull **all** fields of a secret (or search) into the Secret |
| Key names | You choose (`secretKey`) | Same as Vault field names (can add `rewrite`) |
| When | You need renaming or only some fields | App can consume all keys, e.g. `envFrom` |

```yaml
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault-backend, kind: SecretStore }
  target: { name: myapp-all }
  dataFrom:
    - extract:
        key: myapp/db            # every field of secret/myapp/db becomes a key
```

Mixing is allowed: `dataFrom` for the bulk, `data` for renamed extras. If a key appears in both, `data` wins.

#### target: creationPolicy and template (brief)

| `creationPolicy` | Behaviour |
|---|---|
| `Owner` (default) | ESO creates the Secret and sets an ownerReference. Delete the ExternalSecret -> the Secret is garbage-collected. |
| `Orphan` | ESO creates it but leaves it when the ExternalSecret is deleted. |
| `Merge` | ESO merges keys into an **existing** Secret (does not create). |
| `None` | ESO only reads; does not write a Secret (used with generators/templating consumers). |

`target.deletionPolicy` (`Retain` default / `Delete` / `Merge`) controls what happens when the **remote** secret disappears.

`target.template` reshapes the output, e.g. build a connection string or a `kubernetes.io/dockerconfigjson` Secret:

```yaml
  target:
    name: myapp-conn
    template:
      type: Opaque
      data:
        DATABASE_URL: "postgres://{{ .DB_USER }}:{{ .DB_PASSWORD }}@postgres.demo.svc:5432/app"
```

Here `{{ .DB_USER }}` refers to the keys fetched by `data`. This is ESO's own template engine, unrelated to Helm templating (lesson 54). Keep it brief in class; it is a common follow-up question.

#### refreshInterval, rotation, force sync

- ESO re-reads the remote value every `refreshInterval`. If it changed, it **updates the native Secret**.
- `refreshInterval: 0` (or `"0s"`) = fetch **once**, never again.
- Pods consuming the Secret:
  - **Volume mount**: kubelet refreshes the mounted file eventually (typically within about a minute; not for `subPath` mounts). The app must re-read the file.
  - **Env var** (`secretKeyRef`/`envFrom`): values are fixed at container start. **Pod must restart** to see a rotated value. Tools like Reloader automate this (mention only).
- Force immediate re-sync without waiting:

```bash
kubectl -n demo annotate externalsecret myapp-db force-sync=$(date +%s) --overwrite
```

Any change to an annotation/spec triggers a reconcile; the `force-sync` annotation with a new value is the convention.

Choosing an interval: 1h is typical. Very short intervals (seconds) hammer Vault and rate limits for no benefit; the real rotation cadence is set in Vault, not in ESO.

#### Status and conditions

```bash
kubectl -n demo describe externalsecret myapp-db
```

Relevant fields:

```
Status:
  Conditions:
    Type:    Ready
    Status:  True
    Reason:  SecretSynced
    Message: Secret was synced
  Refresh Time:          2026-01-10T10:00:00Z
  Synced Resource Version: 1-abc123
Events:
  Normal  Updated  ... Updated Secret
```

| Reason | Meaning |
|---|---|
| `SecretSynced` | Healthy, Secret matches remote |
| `SecretSyncedError` | Last sync failed; `Message` has the cause (permission, path, store) |
| `SecretDeleted` | Remote gone and `deletionPolicy: Delete` applied |
| `Ready` condition `False` | Do not trust the Secret contents; read `Message` |

### Consuming the synced Secret

Nothing special: it is a plain Secret (lesson 42 syntax).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: myapp
          image: nginx:1.27
          envFrom:
            - secretRef:
                name: myapp-db-secret      # created and kept in sync by ESO
```

If the Secret does not exist yet when the Pod starts, the Pod shows `CreateContainerConfigError` until ESO produces it; it recovers on its own once the ExternalSecret syncs.

### Security notes (concept)

- ESO needs **read** on exactly the paths it syncs: write a narrow Vault policy per role.
- The ESO controller needs Kubernetes RBAC to create/update Secrets cluster-wide: it is a powerful component; protect its namespace.
- The resulting Secret in etcd is still readable by users with `get secrets`; lock down RBAC (lesson 47) and enable etcd encryption at rest.
- Do not commit AppRole `secret_id`, Vault tokens or real passwords. `ExternalSecret`/`SecretStore` YAML with only references is safe to commit.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace --set installCRDs=true` | Install ESO | see Installing ESO |
| `kubectl -n external-secrets get pods` | Check controller, webhook, cert-controller | `kubectl -n external-secrets get pods` |
| `kubectl get crd \| grep external-secrets.io` | Confirm CRDs | `kubectl get crd \| grep external-secrets.io` |
| `kubectl -n NS get secretstore` | Store health (STATUS/READY) | `kubectl -n demo get secretstore` |
| `kubectl get clustersecretstore` | Cluster store health | `kubectl get clustersecretstore` |
| `kubectl -n NS describe secretstore NAME` | Login errors in events | `kubectl -n demo describe secretstore vault-backend` |
| `kubectl -n NS get externalsecret` | Sync status per ExternalSecret | `kubectl -n demo get es` |
| `kubectl -n NS describe externalsecret NAME` | Conditions, refresh time, errors | `kubectl -n demo describe es myapp-db` |
| `kubectl -n NS annotate externalsecret NAME force-sync=$(date +%s) --overwrite` | Force immediate re-sync | see refresh section |
| `kubectl -n NS get secret NAME -o jsonpath='{.data.KEY}' \| base64 -d` | Verify decoded value | `kubectl -n demo get secret myapp-db-secret -o jsonpath='{.data.DB_USER}' \| base64 -d` |
| `kubectl -n NS get secret NAME -o jsonpath='{.metadata.ownerReferences[*].kind}'` | Confirm ESO owns the Secret | prints `ExternalSecret` |
| `kubectl -n external-secrets logs deploy/external-secrets` | Controller logs for deep debugging | `kubectl -n external-secrets logs deploy/external-secrets --tail=50` |
| `kubectl api-resources --api-group=external-secrets.io` | List ESO resource kinds and short names | `kubectl api-resources --api-group=external-secrets.io` |
| `kubectl explain externalsecret.spec.target` | Field docs from CRD schema | `kubectl explain externalsecret.spec.data.remoteRef` |

Vault CLI (instructor reference):

| Command | Purpose |
|---|---|
| `vault kv put secret/PATH k=v ...` | Create/overwrite version |
| `vault kv get secret/PATH` | Read latest |
| `vault kv get -field=KEY secret/PATH` | Read one field |
| `vault kv patch secret/PATH k=v` | Update a field |
| `vault kv metadata get secret/PATH` | See versions |
| `vault policy read NAME` | Show policy |
| `vault read auth/kubernetes/role/ROLE` | Check bound SA/namespace |

Short names: `externalsecret` = `es`, `secretstore` = `ss`, `clustersecretstore` = `css`.

## Examples / Walkthrough

Assumptions: namespace `demo`, ESO installed, Vault reachable at `http://vault.vault.svc:8200`, KV v2 at `secret/`, Vault role `eso-role` bound to ServiceAccount `vault-auth` in namespace `demo`, policy allows read of `secret/data/myapp/*`. Instructor has already run `vault kv put secret/myapp/db username=appuser password='S3cr3t-DemoOnly'`.

### Step 1: Verify ESO is running

```bash
kubectl -n external-secrets get pods
kubectl get crd | grep external-secrets.io
```

### Step 2: Namespace and ServiceAccount

```bash
kubectl create namespace demo
kubectl -n demo create serviceaccount vault-auth
```

### Step 3: SecretStore

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: demo
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-role"
          serviceAccountRef:
            name: "vault-auth"
EOF

kubectl -n demo get secretstore vault-backend
```

Expect `STATUS Valid`, `READY True`. If not, `kubectl -n demo describe secretstore vault-backend` and fix before continuing.

### Step 4: ExternalSecret

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-db
  namespace: demo
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-db-secret
  data:
    - secretKey: DB_USER
      remoteRef:
        key: myapp/db
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: myapp/db
        property: password
EOF

kubectl -n demo get externalsecret myapp-db
```

Expect `SecretSynced` / `True` within a few seconds.

### Step 5: Inspect the manufactured Secret

```bash
kubectl -n demo get secret myapp-db-secret
kubectl -n demo get secret myapp-db-secret -o jsonpath='{.data.DB_USER}' | base64 -d; echo
kubectl -n demo get secret myapp-db-secret -o jsonpath='{.metadata.ownerReferences[0].kind}'; echo
```

Discuss: value matches Vault; owner is `ExternalSecret`; the Secret has no `kubectl create` history.

### Step 6: Consume it

```bash
kubectl -n demo create deployment myapp --image=nginx:1.27
kubectl -n demo set env deployment/myapp --from=secret/myapp-db-secret
kubectl -n demo rollout status deployment/myapp
kubectl -n demo exec deploy/myapp -- printenv DB_USER
```

### Step 7: Rotate in Vault and observe

Instructor changes the value:

```bash
vault kv patch secret/myapp/db password='N3w-DemoOnly'
```

Students:

```bash
# wait up to refreshInterval, or force it
kubectl -n demo annotate externalsecret myapp-db force-sync=$(date +%s) --overwrite
kubectl -n demo get secret myapp-db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo

# env var in the running Pod is still the OLD value
kubectl -n demo exec deploy/myapp -- printenv DB_PASSWORD

# restart to pick up
kubectl -n demo rollout restart deployment/myapp
kubectl -n demo rollout status deployment/myapp
kubectl -n demo exec deploy/myapp -- printenv DB_PASSWORD
```

Lesson: Secret updates automatically, **env vars do not**.

### Step 8: Ownership and cleanup behaviour

```bash
kubectl -n demo delete externalsecret myapp-db
kubectl -n demo get secret myapp-db-secret
# Error from server (NotFound)  -> creationPolicy Owner garbage-collected it
```

### Step 9: dataFrom variant

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-all
  namespace: demo
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault-backend, kind: SecretStore }
  target: { name: myapp-all }
  dataFrom:
    - extract:
        key: myapp/db
EOF
kubectl -n demo get secret myapp-all -o jsonpath='{.data}' ; echo
```

Keys `username` and `password` appear with the Vault field names.

### Step 10: Deliberate failure drills

```bash
# a) wrong path
kubectl -n demo patch externalsecret myapp-all --type merge \
  -p '{"spec":{"dataFrom":[{"extract":{"key":"myapp/nope"}}]}}'
kubectl -n demo describe externalsecret myapp-all | sed -n '/Conditions/,/Events/p'
# expect SecretSyncedError ... Secret does not exist

# b) path outside policy
kubectl -n demo patch externalsecret myapp-all --type merge \
  -p '{"spec":{"dataFrom":[{"extract":{"key":"other/team"}}]}}'
# expect 403 permission denied

# restore
kubectl -n demo patch externalsecret myapp-all --type merge \
  -p '{"spec":{"dataFrom":[{"extract":{"key":"myapp/db"}}]}}'
```

### Step 11: Cleanup

```bash
kubectl delete namespace demo
```

## Troubleshooting Decision Table

| Symptom | Likely cause | Check / fix |
|---|---|---|
| SecretStore `READY False`, `InvalidProviderConfig` | Wrong `server` URL, port, TLS | `describe secretstore`; test `curl` from a debug Pod to `/v1/sys/health` |
| SecretStore event `permission denied` on login | Vault role not bound to this SA name/namespace | `vault read auth/kubernetes/role/eso-role`; compare bound names |
| SecretStore event `service account ... not found` | SA missing in store's namespace | `kubectl -n demo get sa vault-auth` |
| ExternalSecret `SecretSyncedError`: `Secret does not exist` | Wrong `key` (e.g. includes `secret/` or `data/`) | Use path relative to mount, no `data/` |
| ExternalSecret error `cannot find secret data for key` / property missing | `property` typo or field absent | `vault kv get secret/myapp/db` and compare |
| ExternalSecret `403 permission denied` on read | Policy covers `secret/myapp/*` instead of `secret/data/myapp/*`, or path outside policy | `vault policy read myapp-read` |
| ExternalSecret `could not get secret store` | `secretStoreRef.name`/`kind` wrong, or SecretStore in other namespace | `kubectl -n demo get ss`; for cluster store use `kind: ClusterSecretStore` |
| Secret exists but values stale | `refreshInterval` long or `0` | Lower interval or force-sync annotation |
| App still has old password after rotation | Env var consumption | Restart Pod, or mount as volume |
| Secret already existed, ESO will not touch it | Existing Secret not owned by ESO | Delete it (ESO recreates), or use `creationPolicy: Merge` |
| `x509: certificate signed by unknown authority` | Vault uses private CA | Add `caProvider`/`caBundle` to store |
| CRD not found on `apply` | Chart installed without CRDs | Reinstall with `installCRDs=true` |
| Webhook error `failed calling webhook` on apply | cert-controller/webhook Pod not ready | `kubectl -n external-secrets get pods`, wait |

Debug order: **Store Valid? -> ExternalSecret conditions -> path/property vs `vault kv get` -> Vault policy -> controller logs.**

## Common Pitfalls

- **Treating base64 as protection** — it is trivially reversible; ESO does not change that for the resulting Secret.
- **Including `secret/` or `data/` in `remoteRef.key`** — the mount name is in the store; ESO adds `data/` for KV v2. Use `myapp/db`.
- **Writing the Vault policy for `secret/myapp/*` (KV v1 style)** — KV v2 needs `secret/data/myapp/*` (and `secret/metadata/...` for list). Symptom: login succeeds, read gives `403`.
- **ServiceAccount in a different namespace than the SecretStore** — for `SecretStore` it must be same-namespace; for `ClusterSecretStore` set `serviceAccountRef.namespace`.
- **Vault role bound to the wrong namespace/SA** — three values must match: role name, SA name, namespace.
- **Expecting running Pods to see rotated values via env** — env is read at container start; restart or use a volume.
- **`refreshInterval: 0` forgotten** — sync happens once, rotation never arrives.
- **Creating the ExternalSecret before the store is `Valid`** — errors cascade; fix the store first.
- **Hand-editing the managed Secret** — ESO overwrites it at the next refresh. Change the source in Vault.
- **Committing real credentials, `secret_id` or Vault tokens** — only reference-only YAML belongs in Git.
- **Using a `ClusterSecretStore` with a broad Vault policy** — every namespace can then request everything that policy allows. Narrow the policy and add `conditions`.
- **Assuming ESO removes etcd exposure** — the Secret still lives in etcd; combine with etcd encryption and RBAC.
- **Pinning the apiVersion wrong** — use `external-secrets.io/v1beta1` (or `v1` on newer releases); check `kubectl api-resources --api-group=external-secrets.io` and `kubectl explain`.

## FAQ

**Q: Why not just use native Secrets with RBAC?**
A: RBAC limits who reads the object, but native Secrets are base64-only, stored in etcd, have no rotation, and the creation path (YAML in Git or manual `kubectl create`) is leaky. ESO keeps the source of truth in a secret manager with policies and audit.

**Q: Does ESO store my secrets itself?**
A: No. It stores nothing of its own. It reads from the provider and writes ordinary Kubernetes Secrets, which live in etcd like any Secret.

**Q: Is the resulting Secret encrypted?**
A: Only if you enable encryption at rest on the API server. ESO does not change that.

**Q: What is the difference between SecretStore and ExternalSecret?**
A: SecretStore = how to connect/authenticate to the provider. ExternalSecret = which values to fetch and what Secret to produce. One store serves many ExternalSecrets.

**Q: When should I use ClusterSecretStore?**
A: When a platform team provides one shared connection for many namespaces. Restrict it with a narrow Vault policy and `conditions.namespaces`.

**Q: What does `refreshInterval` do and what is a good value?**
A: How often ESO re-reads the remote value. 1h is a common default; 1m for demos. `0` means fetch once. Real rotation frequency is decided in Vault.

**Q: I rotated the password in Vault, but my app still uses the old one.**
A: The Secret was likely updated but the Pod reads it from env vars at start. Restart the Deployment, or consume the Secret as a mounted file and make the app re-read it.

**Q: How do I force a sync right now?**
A: `kubectl annotate externalsecret NAME force-sync=$(date +%s) --overwrite`.

**Q: What happens if I delete the ExternalSecret?**
A: With default `creationPolicy: Owner` the managed Secret is garbage-collected. With `Orphan` it stays.

**Q: What happens if the secret disappears from Vault?**
A: Default `deletionPolicy: Retain` keeps the last Secret and marks the ExternalSecret as error. `Delete` removes the Secret.

**Q: Why do I get `403 permission denied` even though Kubernetes login worked?**
A: Login success only proves identity. The Vault policy attached to the role does not allow the path. In KV v2 the policy path must contain `data/`.

**Q: What path do I put in `remoteRef.key`?**
A: The logical path under the mount without the mount name and without `data/`: for `secret/myapp/db` use `myapp/db`.

**Q: Kubernetes auth or AppRole?**
A: Kubernetes auth when Vault can reach the cluster API: no bootstrap secret. AppRole when it cannot (other clusters, CI, VMs); you must protect `secret_id`.

**Q: Can I use ESO with AWS/Azure/GCP instead of Vault?**
A: Yes. Same `ExternalSecret` structure; only `SecretStore.spec.provider` changes.

**Q: Can I commit ExternalSecret YAML to Git?**
A: Yes, that is a main benefit: it holds references only. Never commit values, `secret_id`, or tokens.

**Q: Can ESO create a `kubernetes.io/dockerconfigjson` or TLS Secret?**
A: Yes via `target.template.type` and `template.data`.

**Q: ExternalSecret says `could not get secret store`.**
A: Wrong `secretStoreRef` name or `kind`, or the SecretStore lives in another namespace. A SecretStore is only visible in its own namespace.

**Q: Does ESO need the Vault CLI?**
A: No. It speaks Vault's HTTP API. The CLI is for admins.

**Q: Do I need Helm to install ESO?**
A: Helm is the usual installer. Manifests are also published; Helm is taught properly in lessons 53-55.

## Practice / Exercise

Environment: pre-configured Vault, ESO installed, namespace of your own (`student-NN`).

**Core**

1. Verify ESO: three Pods Running in `external-secrets`; list CRDs.
2. Create namespace `student-NN` and ServiceAccount `vault-auth` in it.
3. Write a `SecretStore` named `vault-backend` for the instructor's Vault (Kubernetes auth). Reach `STATUS Valid`.
4. Write an `ExternalSecret` mapping `username` and `password` of `myapp/db` to keys `DB_USER` / `DB_PASSWORD` in Secret `myapp-db-secret`, `refreshInterval: 1m`.
5. Decode both keys and compare with the value the instructor reads from Vault.
6. Create a Deployment that consumes the Secret through `envFrom`; confirm with `kubectl exec ... printenv`.
7. Instructor rotates the password. Observe: Secret updated, Pod env unchanged. Restart; confirm new value.
8. Produce failures on purpose: wrong `key`, wrong `property`, path outside policy. Record the exact message of each in `describe externalsecret`.

**Stretch**

1. Replace `data` with `dataFrom.extract` and compare the Secret keys.
2. Use `target.template` to build `DATABASE_URL` from `DB_USER` and `DB_PASSWORD`.
3. Create a `ClusterSecretStore` with `conditions.namespaces` limited to your namespace; reference it from an `ExternalSecret` using `kind: ClusterSecretStore`; show that another namespace is refused.
4. Switch the store to AppRole auth (instructor supplies role_id and secret_id); compare the bootstrap burden with Kubernetes auth.
5. Pin `remoteRef.version` to an older KV version and show the old value comes back.
6. Set `creationPolicy: Orphan`, delete the ExternalSecret, show the Secret remains; clean it up.
7. Mount the Secret as a volume instead of env, rotate in Vault, and watch the file change without a Pod restart (allow about a minute).
8. Write one paragraph: which weaknesses of native Secrets did ESO fix, and which remain?

## Further Reading

- External Secrets Operator docs: https://external-secrets.io
- ESO Vault provider: https://external-secrets.io/latest/provider/hashicorp-vault/
- ESO API reference (SecretStore, ExternalSecret): https://external-secrets.io/latest/api/externalsecret/
- Vault KV v2 secrets engine: https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2
- Vault Kubernetes auth method: https://developer.hashicorp.com/vault/docs/auth/kubernetes
- Vault AppRole auth method: https://developer.hashicorp.com/vault/docs/auth/approle
- Kubernetes: Good practices for Secrets: https://kubernetes.io/docs/concepts/security/secrets-good-practices/
- Kubernetes: Encrypting Secret data at rest: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

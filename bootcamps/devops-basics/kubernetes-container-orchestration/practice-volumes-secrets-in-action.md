# Lesson 44: Practice – Volumes & Secrets in Action

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lesson 41 (PV, PVC, StorageClass, StatefulSet), Lesson 42 (ConfigMap, Secret, mounting as env and volume), Lesson 43 (External Secrets Operator: SecretStore, ClusterSecretStore, ExternalSecret, pre-configured Vault). A working cluster with a default StorageClass, External Secrets Operator installed, and a `ClusterSecretStore` named `vault-backend` already pointing at Vault (KV v2, Kubernetes auth) — the instructor prepares this before class.

## Learning Objectives

By end of lesson student can:
- Deploy PostgreSQL as a StatefulSet with a `volumeClaimTemplates` PVC and verify the StorageClass, PV and PVC binding with `kubectl get sc/pv/pvc`
- Create an `ExternalSecret` that pulls database credentials from Vault and confirm the target Kubernetes Secret is created and kept in sync
- Mount application configuration from a ConfigMap as files and consume Secret keys as environment variables in the same Pod
- Prove persistence: write data, delete the Pod, and confirm both the data and the Secret survive
- Explain why the PostgreSQL password does **not** change when the Vault secret is rotated (initialisation happens once)
- Diagnose the three classic failures: Pending PVC, ExternalSecret `SecretSyncedError`, wrong Vault key path / property
- Clean up in the right order and explain why PVCs of a StatefulSet survive `kubectl delete statefulset`

## Topics

- Deploy stateful app (PostgreSQL) with PVC; verify storage class and binding
- Create ExternalSecret pulling DB credentials from pre-configured Vault
- Mount application config via ConfigMap volume
- Verify persistence: delete pod, confirm data and secrets survive
- Troubleshoot common issues: pending PVC, failed ExternalSecret sync, wrong key path

## Concepts

This is a **practice lesson**. Every building block was taught in lessons 41-43. The goal is to wire them together into one working system and to feel where each piece breaks. The instructor demos Steps 0-3, then students continue alone while the instructor walks around. Checkpoints ("✅ Checkpoint") tell the student what they must see before moving on — do not skip them; most "my lab is broken" questions are a missed checkpoint two steps earlier.

### The system we build

```text
                         Vault (KV v2)
                  secret/lab44/postgres
              {username, password, database}
                            │
                            │  (ESO polls every refreshInterval)
                            ▼
   ClusterSecretStore ──► ExternalSecret ──► Secret "postgres-credentials"
      vault-backend         postgres-creds        │            │
                                                  │            │
                    ┌─────────────────────────────┘            └──────────────┐
                    ▼ (env: secretKeyRef)                       (env: secretKeyRef) ▼
        StatefulSet "postgres"                            Deployment "db-client"
        Pod postgres-0                                    Pod db-client-xxxx
          └─ PVC data-postgres-0  ──► PV (dynamic)          └─ ConfigMap "client-config"
          └─ ConfigMap "postgres-init" (init.sql)              mounted at /etc/app
        headless Service "postgres" ◄───────────────────── psql -h postgres-0.postgres
```

Pieces and their jobs:

| Piece | Job in this lab |
|---|---|
| Vault KV v2 secret | Single source of truth for DB credentials. Never appears in Git. |
| `ClusterSecretStore/vault-backend` | Pre-configured connection + auth to Vault (taught in 43; students only *reference* it). |
| `ExternalSecret/postgres-creds` | Declares *which* Vault keys become *which* Secret keys, and how often to re-sync. |
| `Secret/postgres-credentials` | Created and owned by ESO. Consumed by Pods like any normal Secret. |
| `StatefulSet/postgres` + `volumeClaimTemplates` | Stable identity `postgres-0` and its own PVC `data-postgres-0`. |
| `Service/postgres` (headless) | Stable DNS name `postgres-0.postgres.lab44.svc.cluster.local`. |
| `ConfigMap/postgres-init` | Init SQL placed in `/docker-entrypoint-initdb.d` (runs only on first initialisation). |
| `ConfigMap/client-config` | Application settings mounted as files at `/etc/app` in the client Pod. |
| `Deployment/db-client` | Stand-in "application": a Pod with `psql`, the Secret as env, the ConfigMap as files. |

### Why this order of creation

Order matters, and it is itself a teaching point:

1. Namespace
2. **ExternalSecret first** — the Secret must exist before any Pod that references it, otherwise the Pod sticks in `CreateContainerConfigError` ("secret not found").
3. ConfigMaps — same reason (Pod refuses to start without a referenced ConfigMap unless it is marked optional).
4. StatefulSet + headless Service — creates the PVC and the DB.
5. Client Deployment — needs the DB reachable.

### What "survives" means

| Event | Data in PVC | Secret | ConfigMap |
|---|---|---|---|
| `kubectl delete pod postgres-0` | survives (same PVC re-attached to new Pod) | survives | survives |
| `kubectl rollout restart statefulset/postgres` | survives | survives | survives |
| `kubectl delete statefulset postgres` | **survives** (PVC is not deleted by default) | survives | survives |
| `kubectl delete pvc data-postgres-0` | **lost** if reclaimPolicy is `Delete` (typical dynamic StorageClass) | survives | survives |
| `kubectl delete externalsecret postgres-creds` | survives | **deleted** (owned by the ExternalSecret) | survives |
| Vault secret deleted | survives | stays (default `deletionPolicy: Retain`) but stops updating | survives |

### The rotation trap

`POSTGRES_PASSWORD` is read by the official postgres image **only when the data directory is empty** (first start). After that, the password lives inside the database files on the PVC. Rotating the secret in Vault updates the Kubernetes Secret (ESO does its job) but **does not** change the password inside PostgreSQL. In production you rotate with `ALTER USER ... PASSWORD` plus a Secret update, or use Vault's database secrets engine. We will demonstrate the trap deliberately in Step 9.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl get sc` | List StorageClasses, find the default (marked `(default)`) | `kubectl get sc` |
| `kubectl get pv,pvc -n lab44` | See PV/PVC binding status | `kubectl get pv,pvc -n lab44` |
| `kubectl describe pvc <name>` | Binding events, which StorageClass, why Pending | `kubectl describe pvc data-postgres-0 -n lab44` |
| `kubectl get clustersecretstore` | Check the Vault connection is `Valid` / `Ready` | `kubectl get clustersecretstore vault-backend` |
| `kubectl get externalsecret -n lab44` | STATUS `SecretSynced` + READY `True` | `kubectl get es -n lab44` |
| `kubectl describe externalsecret <n>` | Sync errors and events | `kubectl describe es postgres-creds -n lab44` |
| `kubectl annotate es <n> force-sync=$(date +%s) --overwrite` | Make ESO re-sync immediately instead of waiting for `refreshInterval` | `kubectl annotate es postgres-creds force-sync=$(date +%s) --overwrite -n lab44` |
| `kubectl get secret <n> -o jsonpath='{.data.password}' \| base64 -d` | Read a Secret value | `kubectl get secret postgres-credentials -n lab44 -o jsonpath='{.data.password}' \| base64 -d` |
| `kubectl exec -it <pod> -- psql ...` | Run SQL inside a Pod | `kubectl exec -it postgres-0 -n lab44 -- psql -U appuser -d appdb` |
| `kubectl wait --for=condition=Ready pod/<n> --timeout=120s` | Block until Pod is Ready | `kubectl wait --for=condition=Ready pod/postgres-0 -n lab44 --timeout=180s` |
| `kubectl wait --for=condition=Ready externalsecret/<n>` | Block until the first sync works | `kubectl wait --for=condition=Ready es/postgres-creds -n lab44 --timeout=60s` |
| `kubectl get events --sort-by=.lastTimestamp` | Chronological events, best first debugging tool | `kubectl get events -n lab44 --sort-by=.lastTimestamp` |
| `kubectl delete pod <n>` | Prove persistence by killing the Pod | `kubectl delete pod postgres-0 -n lab44` |
| `kubectl logs <pod>` | DB startup / init output | `kubectl logs postgres-0 -n lab44` |

## Examples / Walkthrough

Conventions: all resources live in namespace `lab44`. Students use their own namespace suffix on a shared cluster (`lab44-ali`, `lab44-vali`) and adjust the Vault path accordingly (instructor decides: one shared path per student, e.g. `secret/lab44-ali/postgres`). Commands below assume a single-student cluster.

### Step 0 — Pre-flight checks (5 min)

The instructor already prepared Vault. The Vault secret at `secret/lab44/postgres` contains (for reference — students never type these; they only read via Kubernetes):

```bash
# Done by the instructor, NOT by students:
vault kv put secret/lab44/postgres \
    username=appuser \
    password='S3cr3t-Start-1' \
    database=appdb
```

Students verify the cluster side:

```bash
# 1. A default StorageClass exists?
kubectl get sc
# NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ...
# local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   ...

# 2. ESO is installed? (CRDs + controller Pods)
kubectl get crd | grep external-secrets.io
kubectl get pods -n external-secrets

# 3. The pre-configured store is healthy?
kubectl get clustersecretstore vault-backend
# NAME            AGE   STATUS   CAPABILITIES   READY
# vault-backend   3d    Valid    ReadWrite      True
```

✅ Checkpoint: one StorageClass marked `(default)`, ESO Pods `Running`, `vault-backend` shows `READY True`. If the store is not Ready, stop and call the instructor — nothing later can work.

> Note: if your cluster has **no** default StorageClass, PVCs will stay Pending forever. Either mark one default (`kubectl patch sc local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`) or set `storageClassName` explicitly in the StatefulSet below.

### Step 1 — Namespace

```bash
kubectl create namespace lab44
kubectl config set-context --current --namespace=lab44
```

From now on `-n lab44` is implied. (The reference table keeps `-n lab44` explicit for copy-paste safety.)

### Step 2 — ExternalSecret pulling DB credentials from Vault

`externalsecret.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: postgres-creds
  namespace: lab44
spec:
  refreshInterval: 1m                # re-read Vault every minute (short on purpose for the lab)
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault-backend
  target:
    name: postgres-credentials       # name of the Kubernetes Secret ESO will create
    creationPolicy: Owner            # ESO owns the Secret; deleting the ExternalSecret deletes it
  data:
    - secretKey: POSTGRES_USER       # key inside the Kubernetes Secret
      remoteRef:
        key: lab44/postgres          # path in the KV v2 mount (mount "secret" comes from the store)
        property: username           # field inside the Vault secret
    - secretKey: POSTGRES_PASSWORD
      remoteRef:
        key: lab44/postgres
        property: password
    - secretKey: POSTGRES_DB
      remoteRef:
        key: lab44/postgres
        property: database
```

```bash
kubectl apply -f externalsecret.yaml
kubectl wait --for=condition=Ready externalsecret/postgres-creds --timeout=60s
kubectl get externalsecret
# NAME             STORETYPE            STORE           REFRESH INTERVAL   STATUS         READY
# postgres-creds   ClusterSecretStore   vault-backend   1m                 SecretSynced   True

kubectl get secret postgres-credentials
# NAME                   TYPE     DATA   AGE
# postgres-credentials   Opaque   3      10s

kubectl get secret postgres-credentials -o jsonpath='{.data.POSTGRES_USER}' | base64 -d; echo
# appuser
```

✅ Checkpoint: `STATUS SecretSynced`, `READY True`, Secret has `DATA 3`, decoded user is `appuser`.

Key points to say out loud:
- `key:` is the path **relative to the mount** configured in the store (`secret/` mount → `lab44/postgres`), **not** `secret/data/lab44/postgres`. KV v2 inserts `/data/` itself; typing it is a classic mistake.
- `property:` picks one field. Omit it and you get the whole JSON blob as a single value.
- `secretKey:` names become env var names later (we chose upper-case so `envFrom` works directly).
- Nothing secret is in this YAML — it is safe to commit to Git. That is the entire point of ESO.

### Step 3 — ConfigMaps: init SQL and application config

`configmaps.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
  namespace: lab44
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS visits (
      id         SERIAL PRIMARY KEY,
      who        TEXT NOT NULL,
      created_at TIMESTAMPTZ NOT NULL DEFAULT now()
    );
    INSERT INTO visits (who) VALUES ('init-script');
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: client-config
  namespace: lab44
data:
  app.properties: |
    db.host=postgres-0.postgres.lab44.svc.cluster.local
    db.port=5432
    app.greeting=Hello from a ConfigMap volume
    app.log_level=info
  motd.txt: |
    Practice lab 44 — volumes and secrets in action
```

```bash
kubectl apply -f configmaps.yaml
kubectl get configmap
kubectl describe configmap client-config
```

✅ Checkpoint: both ConfigMaps listed, `client-config` shows two keys (`app.properties`, `motd.txt`).

### Step 4 — PostgreSQL StatefulSet with PVC

`postgres.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres                    # headless: gives each Pod a stable DNS name
  namespace: lab44
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
  namespace: lab44
spec:
  serviceName: postgres             # must match the headless Service above
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
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
              name: pg
          envFrom:
            - secretRef:
                name: postgres-credentials      # POSTGRES_USER / PASSWORD / DB from ESO
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata   # subdirectory avoids the "lost+found" init error
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: init
              mountPath: /docker-entrypoint-initdb.d
              readOnly: true
      volumes:
        - name: init
          configMap:
            name: postgres-init
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        # storageClassName: local-path     # uncomment if the cluster has no default class
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f postgres.yaml

# Watch the PVC get created and bound
kubectl get pvc -w
# NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# data-postgres-0   Pending                                      local-path     2s
# data-postgres-0   Bound     pvc-6d1c...   1Gi     RWO            local-path     8s
# (Ctrl-C when Bound)

kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
kubectl get pods,pvc,pv
```

Explain what happened (bring back lesson 41):
1. StatefulSet controller created Pod `postgres-0` **and** PVC `data-<template>-<pod>` = `data-postgres-0`.
2. With `WaitForFirstConsumer` the PVC stays `Pending` until the Pod is scheduled, then the provisioner creates the PV on that node.
3. The PVC was bound to a dynamically created PV; that PV's `reclaimPolicy` comes from the StorageClass (`Delete` for most dynamic classes).

```bash
# Verify storage class and binding explicitly
kubectl get pvc data-postgres-0 -o jsonpath='{.spec.storageClassName}{"  "}{.status.phase}{"  "}{.spec.volumeName}{"\n"}'
kubectl get pv $(kubectl get pvc data-postgres-0 -o jsonpath='{.spec.volumeName}') \
  -o custom-columns=NAME:.metadata.name,CLASS:.spec.storageClassName,RECLAIM:.spec.persistentVolumeReclaimPolicy,CAP:.spec.capacity.storage,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]
kubectl describe pvc data-postgres-0
```

✅ Checkpoint: `postgres-0` is `1/1 Running`; PVC `Bound`; you can name the StorageClass, the PV and its reclaim policy.

Check the DB initialised with the Secret credentials and ran the init script:

```bash
kubectl logs postgres-0 | grep -E "init process|CREATE TABLE|INSERT|ready to accept"
# /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/init.sql
# CREATE TABLE
# INSERT 0 1
# database system is ready to accept connections

kubectl exec postgres-0 -- psql -U appuser -d appdb -c "SELECT * FROM visits;"
#  id |     who     |          created_at
# ----+-------------+-------------------------------
#   1 | init-script | 2025-...
```

`kubectl exec` runs inside the container, where `POSTGRES_*` env vars exist; local socket auth needs no password so this works.

### Step 5 — Client "application" with ConfigMap volume + Secret env

`client.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-client
  namespace: lab44
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-client
  template:
    metadata:
      labels:
        app: db-client
    spec:
      containers:
        - name: client
          image: postgres:16-alpine          # reuse the image only to get the psql client
          command: ["sh", "-c", "sleep infinity"]
          env:
            - name: PGHOST
              value: postgres-0.postgres.lab44.svc.cluster.local
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_USER
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_PASSWORD
            - name: PGDATABASE
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_DB
          volumeMounts:
            - name: config
              mountPath: /etc/app
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: client-config
```

```bash
kubectl apply -f client.yaml
kubectl rollout status deployment/db-client

# 1. Config arrives as files
kubectl exec deploy/db-client -- ls -l /etc/app
kubectl exec deploy/db-client -- cat /etc/app/app.properties
kubectl exec deploy/db-client -- cat /etc/app/motd.txt

# 2. Secret arrives as env (psql reads PGHOST/PGUSER/PGPASSWORD/PGDATABASE automatically)
kubectl exec deploy/db-client -- env | grep ^PG

# 3. Real network connection over the headless-Service DNS name, with password auth
kubectl exec deploy/db-client -- psql -c "SELECT current_user, inet_server_addr();"
kubectl exec deploy/db-client -- psql -c "INSERT INTO visits (who) VALUES ('db-client'); SELECT count(*) FROM visits;"
```

✅ Checkpoint: files visible under `/etc/app`, env vars present, and psql over the network returns `current_user = appuser`. If this fails with `password authentication failed`, the Secret value and the DB password differ — jump to the troubleshooting table.

Mention for the AI agent / students: ConfigMap volumes are **symlinks** (`/etc/app/app.properties -> ..data/app.properties`). The kubelet updates them atomically (within ~1 minute) when the ConfigMap changes. Env vars, in contrast, are fixed at container start.

Quick live test of that statement:

```bash
kubectl patch configmap client-config --type merge \
  -p '{"data":{"app.greeting":"ignored","app.properties":"db.port=5432\napp.greeting=Updated live\n"}}'
# wait ~60-90s
kubectl exec deploy/db-client -- cat /etc/app/app.properties     # shows "Updated live" — no restart needed
```

### Step 6 — Verify persistence: delete the Pod

Write something recognisable first:

```bash
kubectl exec deploy/db-client -- psql -c "INSERT INTO visits (who) VALUES ('before-delete'); SELECT id, who FROM visits ORDER BY id;"
#  id |      who
# ----+---------------
#   1 | init-script
#   2 | db-client
#   3 | before-delete
```

Kill the database Pod:

```bash
kubectl get pod postgres-0 -o wide          # note node + age
kubectl delete pod postgres-0
kubectl get pods -w                         # StatefulSet recreates postgres-0 with the same name
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
```

Check the data **and** the secret survived:

```bash
kubectl exec deploy/db-client -- psql -c "SELECT id, who FROM visits ORDER BY id;"
# all 3 rows still there — data came back from the same PVC

kubectl logs postgres-0 | head -20
# "Skipping initialization" / "PostgreSQL Database directory appears to contain a database"
# → init.sql did NOT run again (data directory was not empty)

kubectl get secret postgres-credentials -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
# unchanged timestamp: Secret was never recreated

kubectl get pvc data-postgres-0 -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
# unchanged: same PVC, same PV
```

✅ Checkpoint: rows 1-3 present after restart; logs show initialisation skipped; PVC and Secret creation timestamps unchanged.

Variations to run quickly (instructor choice):

```bash
kubectl rollout restart statefulset/postgres            # data survives
kubectl delete statefulset postgres                     # delete the controller, keep PVC
kubectl get pvc                                         # data-postgres-0 is STILL there
kubectl apply -f postgres.yaml                          # recreate: new Pod re-attaches to old PVC
kubectl exec deploy/db-client -- psql -c "SELECT count(*) FROM visits;"   # still 3+
```

### Step 7 — Rotate the Secret in Vault (the sync half of ESO)

The instructor (or a student with Vault access) changes the password in Vault:

```bash
vault kv put secret/lab44/postgres username=appuser password='S3cr3t-Rotated-2' database=appdb
```

Students watch the Kubernetes Secret follow:

```bash
# Option A: wait up to refreshInterval (1m).  Option B: force it:
kubectl annotate externalsecret postgres-creds force-sync=$(date +%s) --overwrite

kubectl get secret postgres-credentials -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d; echo
# S3cr3t-Rotated-2

kubectl describe externalsecret postgres-creds | tail -8
# Events:  Normal  Updated  ...  Updated Secret
```

Now the trap:

```bash
# The running client Pod still has the OLD password in its environment (env is fixed at start)
kubectl exec deploy/db-client -- env | grep PGPASSWORD
# PGPASSWORD=S3cr3t-Start-1

# Restart the client so it picks up the NEW Secret value
kubectl rollout restart deployment/db-client
kubectl rollout status deployment/db-client

# ...and now login FAILS: the DB still holds the original password
kubectl exec deploy/db-client -- psql -c "SELECT 1;"
# psql: error: connection to server at "postgres-0..." failed: FATAL:  password authentication failed for user "appuser"
```

Explain: ESO synced the Kubernetes Secret correctly (its job), but PostgreSQL only reads `POSTGRES_PASSWORD` on first init. The password in the DB must be changed with SQL:

```bash
# Fix: change the password inside the database (connect via local socket; no password needed)
kubectl exec postgres-0 -- psql -U appuser -d appdb -c "ALTER USER appuser WITH PASSWORD 'S3cr3t-Rotated-2';"
kubectl exec deploy/db-client -- psql -c "SELECT 'works again' AS status;"
```

Two lessons:
1. **Env vars never update in a running Pod** — a Secret change requires a Pod restart (or mount the Secret as a volume and have the app re-read the file).
2. **A secret manager rotates the *value*; the *consumer* (the DB) must also be told.** Real rotation = update the store + update the system that uses it.

### Step 8 — Break things on purpose: Pending PVC

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc
  namespace: lab44
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: fast-ssd          # does not exist in this cluster
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc broken-pvc
# NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# broken-pvc   Pending                                      fast-ssd       5s

kubectl describe pvc broken-pvc | tail -6
# Warning  ProvisioningFailed  ...  storageclass.storage.k8s.io "fast-ssd" not found
```

Fix: PVC `storageClassName` is immutable → delete and recreate with an existing class.

```bash
kubectl delete pvc broken-pvc
kubectl get sc                                   # pick a real name
# recreate with storageClassName: local-path (or omit the field to use the default)
```

Also show the *silent* Pending: a PVC with `WaitForFirstConsumer` and **no Pod using it** stays `Pending` with event `waiting for first consumer to be created before binding` — that is normal, not an error.

### Step 9 — Break things on purpose: ExternalSecret failures

**Failure A — wrong Vault path.**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: bad-path
  namespace: lab44
spec:
  refreshInterval: 1m
  secretStoreRef: {kind: ClusterSecretStore, name: vault-backend}
  target: {name: bad-path-secret}
  data:
    - secretKey: PASSWORD
      remoteRef:
        key: lab44/postgress            # typo: extra "s"
        property: password
EOF

kubectl get externalsecret bad-path
# NAME       ...   STATUS              READY
# bad-path   ...   SecretSyncedError   False

kubectl describe externalsecret bad-path | grep -A3 -i "error\|warning"
# Warning  UpdateFailed  ...  error processing spec.data[0] (key: lab44/postgress), err: Secret does not exist

kubectl get secret bad-path-secret
# Error from server (NotFound)
```

**Failure B — right path, wrong property.**

```bash
kubectl patch externalsecret bad-path --type merge -p '{"spec":{"data":[{"secretKey":"PASSWORD","remoteRef":{"key":"lab44/postgres","property":"passwd"}}]}}'
kubectl describe externalsecret bad-path | grep -i "property\|err"
# ...  cannot find secret data for key: "passwd"
```

**Failure C — store not reachable / no permission** (instructor demo only): point `secretStoreRef.name` at a store that does not exist → `ExternalSecret ... could not get SecretStore "nope"`; or use a Vault path the Kubernetes-auth role has no policy for → `permission denied` in the event. Students read the event, not guess.

Fix and clean up:

```bash
kubectl delete externalsecret bad-path
# the Secret ESO created (if any) is garbage-collected with its owner
```

Debug order that always works (write on board):
1. `kubectl get externalsecret` — STATUS / READY
2. `kubectl describe externalsecret <name>` — read Events
3. `kubectl get clustersecretstore` — is the store Ready?
4. `kubectl logs -n external-secrets deploy/external-secrets` — controller-side detail
5. Check the path/property in Vault itself (UI or `vault kv get secret/lab44/postgres`)

### Step 10 — Cleanup (order matters)

```bash
kubectl delete deployment db-client
kubectl delete statefulset postgres
kubectl get pvc                                  # data-postgres-0 still here! StatefulSet never deletes PVCs by default
kubectl delete pvc data-postgres-0               # now the data is gone (reclaimPolicy Delete)
kubectl get pv                                   # PV disappears with it

kubectl delete externalsecret postgres-creds     # Secret postgres-credentials is garbage-collected
kubectl get secret
kubectl delete configmap postgres-init client-config
kubectl delete svc postgres
kubectl delete namespace lab44                   # or just this, to remove everything at once
kubectl config set-context --current --namespace=default
```

✅ Checkpoint: `kubectl get all,pvc,secret,cm -n lab44` empty (or namespace gone) and `kubectl get pv` shows no leftover `lab44` volume.

## Common Pitfalls

- **Pod stuck in `CreateContainerConfigError`: `secret "postgres-credentials" not found`** — the ExternalSecret has not synced yet, or failed. Create and verify the ExternalSecret *before* the workload (`kubectl wait --for=condition=Ready es/...`).
- **PVC `Pending` forever with no error** — either no default StorageClass, or `WaitForFirstConsumer` and no Pod uses the PVC yet. `kubectl describe pvc` event tells which one.
- **PVC `Pending` + `storageclass ... not found`** — typo in `storageClassName`. It is immutable on an existing PVC: delete and recreate.
- **`initdb: directory "/var/lib/postgresql/data" exists but is not empty ... lost+found`** — the volume root contains `lost+found` (ext4). Set `PGDATA` to a subdirectory (`.../data/pgdata`) as in Step 4.
- **ExternalSecret `SecretSyncedError`: `Secret does not exist`** — wrong `remoteRef.key`. Do not include the mount name or `/data/` in the key for KV v2 — the store config already knows the mount and version.
- **ExternalSecret synced but Secret is missing a key** — wrong `property:`; ESO errors instead of creating an empty value. Check the field names in Vault.
- **Secret updated but app still has the old value** — env vars from a Secret are read once at container start. Restart the Pod (`kubectl rollout restart`), or mount as a volume and re-read the file.
- **Vault rotated, DB login fails** — the DB password is stored in the data directory; `POSTGRES_PASSWORD` is only used on first init. Run `ALTER USER` as well.
- **`init.sql` edit has no effect** — scripts in `/docker-entrypoint-initdb.d` run only when the data directory is empty. Delete the PVC (data loss!) or apply SQL manually.
- **`kubectl delete statefulset` and expecting data to be gone** — PVCs from `volumeClaimTemplates` are deliberately kept. Delete them explicitly; recreating the StatefulSet re-attaches the old data.
- **Deleting the ExternalSecret and expecting the Secret to stay** — with `creationPolicy: Owner` the Secret is garbage-collected together with it. Pods already running keep their env values but new Pods will fail.
- **Mounting the ConfigMap over a directory that already has files** — a volume mount *hides* the existing directory content. Use `subPath` to mount a single file when you must not shadow the rest.
- **Committing a base64 Secret manifest "just for the lab"** — base64 is not encryption. This lab's whole point is that Git only holds the ExternalSecret.

## FAQ

**Q: Why use a StatefulSet for PostgreSQL instead of a Deployment with a PVC?**
A: A Deployment with one PVC works for a single replica but gives Pods random names and, when scaled above 1, all replicas fight over the same ReadWriteOnce volume. A StatefulSet gives each replica a stable name (`postgres-0`), a stable DNS name via the headless Service, and its **own** PVC from `volumeClaimTemplates`. For databases that identity is what makes replication and recovery possible. For a single throw-away DB a Deployment is acceptable, but StatefulSet is the standard teaching and production pattern.

**Q: I deleted the Pod and my data is still there. Where is it?**
A: On the PersistentVolume bound to PVC `data-postgres-0`. The Pod is disposable; the PVC/PV outlive it. The StatefulSet creates a replacement `postgres-0` that mounts the same PVC again.

**Q: I deleted the StatefulSet — is my data gone?**
A: No. PVCs created from `volumeClaimTemplates` are not deleted with the StatefulSet (unless `persistentVolumeClaimRetentionPolicy` is configured otherwise). Delete the PVC explicitly when you really want the data removed. With `reclaimPolicy: Delete` the underlying PV and its storage vanish with the PVC.

**Q: What is the difference between the Secret ESO creates and a Secret I write by hand?**
A: None from the Pod's point of view — it is a plain `Opaque` Secret consumed with `secretKeyRef`/`envFrom`/volume. The difference is *lifecycle*: ESO owns it, keeps it in sync with Vault every `refreshInterval`, and will overwrite manual edits. Never edit it directly.

**Q: How fast does a Vault change reach the cluster?**
A: Within `refreshInterval` (1 minute in this lab; typically 1h in production) plus a few seconds. Force an immediate sync by adding/changing an annotation on the ExternalSecret (`kubectl annotate es <name> force-sync=$(date +%s) --overwrite`).

**Q: The Secret updated. Why didn't my running Pod see it?**
A: If it is consumed as an environment variable it never updates in a running container; restart the Pod. If it is mounted as a volume, the files are refreshed by the kubelet (≈1 min) but the application must re-read them. (Exception: `subPath` mounts never update.)

**Q: Where does `lab44/postgres` really live in Vault?**
A: KV v2 stores data under `<mount>/data/<path>`. With a mount called `secret`, the real API path is `secret/data/lab44/postgres`, and the CLI path is `secret/lab44/postgres`. In an ExternalSecret you give only the part **after** the mount: `lab44/postgres`. The mount and `version: v2` are configured once in the SecretStore.

**Q: What do `property` and no `property` do differently?**
A: With `property: password` you get just that field's string. Without it, ESO returns the whole secret as a JSON string (useful with `dataFrom`/templating, but not what you want for a single env var).

**Q: My PVC says `Pending` but there is no error event.**
A: Probably `volumeBindingMode: WaitForFirstConsumer`: the PVC deliberately waits until a Pod that uses it is scheduled, so the PV can be created on the correct node/zone. Create the Pod and it binds. If a Pod exists and the PVC is still Pending, then look at `kubectl describe pvc` for provisioner errors.

**Q: Why mount ConfigMap `postgres-init` read-only into `/docker-entrypoint-initdb.d`?**
A: ConfigMap volumes are read-only anyway; `readOnly: true` makes it explicit. The official postgres image executes every `*.sql`, `*.sh` and `*.sql.gz` file in that directory in alphabetical order, but only on first initialisation of an empty data directory.

**Q: Can I put the DB password in a ConfigMap since it is "just a lab"?**
A: Do not. ConfigMaps are plain text, shown by `kubectl describe`, and often broadly readable. Habits formed in labs end up in production. Use a Secret, preferably ESO-managed.

**Q: Is the ExternalSecret manifest safe to commit to Git?**
A: Yes — it contains only *references* (store name, Vault path, field names), never values. That is the main benefit over committing Secret manifests.

**Q: Why is `kubectl exec postgres-0 -- psql -U appuser` working without a password?**
A: In the official image `pg_hba.conf` trusts local socket connections inside the container. Network connections (the `db-client` Pod) use password authentication, which is why the rotation trap only shows up there.

**Q: What if two students share one Vault path?**
A: They all get the same credentials — fine for reading. Never let students *write* to a shared path; one overwrite breaks everyone's sync. Give each student a prefix (`lab44-<name>/postgres`).

**Q: How would I see everything about one Pod's storage and secrets in one command?**
A: `kubectl describe pod postgres-0` shows `Mounts`, `Volumes` (PVC and ConfigMap names), `Environment Variables from` (Secret name), and Events. Then describe the PVC, PV and ExternalSecret for each link in the chain.

## Practice / Exercise

### Core (all students, in-class)

Complete Steps 0-10 above. Hand-in checklist (screenshot or paste):

1. Output of `kubectl get sc,pv,pvc -n lab44` showing the PVC `Bound` and the StorageClass name.
2. Output of `kubectl get externalsecret,secret -n lab44` showing `SecretSynced` and `postgres-credentials`.
3. Output of `kubectl exec deploy/db-client -- cat /etc/app/app.properties`.
4. Row list from `visits` **after** `kubectl delete pod postgres-0`.
5. One-sentence answer: why did DB login fail after the Vault rotation, and what fixed it?
6. Reproduce a Pending PVC and a `SecretSyncedError`, paste the `describe` event of each, and the fix applied.

### Stretch

1. **Secret as volume.** Change `db-client` to mount `postgres-credentials` at `/etc/db-secret` (read-only, `defaultMode: 0400`) instead of env. Rotate the Vault value again and observe the file contents update in the running Pod **without** restart (wait ~1-2 min after the ESO sync). Why does this approach beat env vars for rotation?
2. **`dataFrom` + template.** Rewrite the ExternalSecret using `dataFrom.extract` (whole secret) and a `target.template` that builds a `DATABASE_URL` key (`postgres://appuser:<password>@postgres-0.postgres:5432/appdb`). Consume `DATABASE_URL` in the client.
3. **Two replicas.** Scale the StatefulSet to 2 (`kubectl scale sts postgres --replicas=2`). Observe `postgres-1` and a second PVC `data-postgres-1`. Why is it a *separate empty database*, not a replica? (Replication setup is out of scope; explain the identity/storage part only.)
4. **Reclaim policy experiment.** Find a StorageClass (or create a PV by hand) with `reclaimPolicy: Retain`. Delete the PVC and check the PV status (`Released`). Explain how to reuse the data (clear `claimRef`) and why Retain is safer for databases.
5. **Resize.** If the StorageClass has `allowVolumeExpansion: true`, edit the PVC `data-postgres-0` from 1Gi to 2Gi and watch `kubectl get pvc` / `df -h` inside the Pod. What happens to the StatefulSet's `volumeClaimTemplates` size (hint: it is immutable; the PVC is edited directly)?
6. **Break-fix swap.** Pair up: one student introduces a hidden bug (wrong `key:`, wrong `storageClassName`, wrong ConfigMap name, wrong `secretKeyRef` key). The other finds it using only `get`, `describe`, `events`, `logs` — timed, 10 minutes.

## Further Reading

- Kubernetes docs — Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Kubernetes docs — StatefulSets (volumeClaimTemplates, PVC retention policy): https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Kubernetes docs — Storage Classes (volumeBindingMode, reclaimPolicy): https://kubernetes.io/docs/concepts/storage/storage-classes/
- Kubernetes docs — ConfigMaps and consuming them as volumes: https://kubernetes.io/docs/concepts/configuration/configmap/
- Kubernetes docs — Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
- External Secrets Operator — ExternalSecret API and Vault provider: https://external-secrets.io/latest/provider/hashicorp-vault/
- External Secrets Operator — Troubleshooting and refresh behaviour: https://external-secrets.io/latest/guides/getting-started/
- Official postgres image — initialisation scripts and `PGDATA`: https://hub.docker.com/_/postgres

# Lesson 42: Configuration Management & Secrets

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-41 (cluster architecture, kubectl, Pods/Deployments/manifests, Services, Ingress, Gateway API, build-push-deploy practice, PV/PVC/StatefulSet). Student can write Pod and Deployment YAML, use `kubectl apply/get/describe/exec/logs`, and knows Docker `-e` / `--env-file` (lesson 30).

## Learning Objectives

By end of lesson student can:
- Explain why configuration must live outside the container image (twelve-factor "config in environment") and which Kubernetes objects carry it
- Create a ConfigMap three ways: `--from-literal`, `--from-file` / `--from-env-file`, and declarative YAML
- Inject ConfigMap data into a Pod as a single env var (`configMapKeyRef`), all keys at once (`envFrom`), or as files in a mounted volume
- Predict which injection methods pick up ConfigMap changes at runtime (volumes: yes, eventually; env vars: no) and force a rollout when needed
- Use `subPath`, `items`, `defaultMode`, and `immutable` correctly and explain their trade-offs
- Create Secrets of the types `Opaque`, `kubernetes.io/dockerconfigjson` (docker-registry), `kubernetes.io/tls`, and `kubernetes.io/basic-auth`
- Explain that base64 is encoding, not encryption, and state what actually protects a Secret (RBAC, encryption at rest, not committing to Git)
- Use `stringData`, `secretKeyRef`, `envFrom`, volume mounts with file permissions, and `imagePullSecrets`
- List best practices: never commit Secret manifests to Git, least-privilege access to Secrets, prefer volume mounts over env vars for sensitive data, rotate credentials
- Debug the common failures: `CreateContainerConfigError`, missing key, wrong namespace, stale env after update

## Topics

- ConfigMap: creation methods (--from-literal, --from-file, YAML); kubectl describe
- Using ConfigMaps: valueFrom.configMapKeyRef (env), envFrom (all keys), volumeMount
- Secret: types (Opaque, docker-registry, tls), base64 encoding; stringData shortcut
- Using Secrets: secretKeyRef (env), envFrom, volume mount as files; permissions
- Best practices: avoid secrets in manifests committed to Git; RBAC for secret access

## Concepts

### Why separate configuration from the image

A container image should be **built once and run everywhere**: dev, staging, production. What differs between environments is configuration: database host, log level, feature flags, API keys, TLS certificates. If that data is baked into the image (lesson 31 `ENV`, `COPY config.yaml`), every environment needs its own image, a password change needs a rebuild, and secrets leak into the registry and image layers (anyone with `docker history` can read them).

Kubernetes gives two API objects for this:

| Object | Holds | Typical content | Stored as |
|---|---|---|---|
| **ConfigMap** | non-sensitive config | log level, URLs, `nginx.conf`, feature flags | plain text in etcd |
| **Secret** | sensitive config | passwords, tokens, TLS keys, registry credentials | base64 in etcd (optionally encrypted at rest) |

Both are namespaced objects. Both are "key → value" maps. A Pod references them by name; the kubelet fetches the data and injects it into the container as environment variables or files. The Pod spec stays identical across environments; only the ConfigMap/Secret contents differ per namespace or cluster.

```text
 ConfigMap app-config  ──┐                     ┌── env var  LOG_LEVEL=debug
 (key/value in etcd)     ├──> kubelet injects ─┤
 Secret db-creds       ──┘                     └── file /etc/app/app.properties
```

Compare with Docker (lesson 30): `docker run -e KEY=val --env-file app.env -v ./conf:/etc/app` maps directly to `env`, `envFrom`, and a volume mount of a ConfigMap. The difference: in Kubernetes the config is a first-class cluster object, versionable, namespaced and access-controlled.

### ConfigMap anatomy

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:                       # UTF-8 text values
  LOG_LEVEL: "debug"        # simple key → simple value
  APP_MODE: "production"
  app.properties: |         # key can be a filename, value a whole file
    server.port=8080
    cache.ttl=300
binaryData:                 # base64-encoded binary values (rare)
  logo.png: iVBORw0KGgoAAAANSUhEUgAA...
immutable: false            # true = cannot be edited, only deleted/recreated
```

Rules:
- There is **no `spec`** — `data` sits at top level (common first-timer mistake).
- All `data` values are **strings**. Write `"8080"`, `"true"` — YAML `true` or `8080` unquoted fails validation for env vars ("cannot unmarshal bool into string").
- Keys must be alphanumerics, `-`, `_`, or `.`; max total size of a ConfigMap is **1 MiB** (etcd limit). Bigger files belong in an image, volume, or object store.
- `data` and `binaryData` keys must not overlap.
- A key may contain newlines (use `|` block scalar) — this is how you store a whole config file.

### Three ways to create a ConfigMap

**1. `--from-literal`** — inline key=value pairs:

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=APP_MODE=production
```

**2. `--from-file`** — filename becomes the key, file content the value:

```bash
kubectl create configmap nginx-conf --from-file=nginx.conf
kubectl create configmap nginx-conf --from-file=site.conf=./nginx.conf   # custom key name
kubectl create configmap app-files --from-file=./conf.d/                  # every file in dir becomes a key
```

**3. `--from-env-file`** — each `KEY=VALUE` line in a dotenv file becomes a separate key:

```bash
cat > app.env <<'EOF'
LOG_LEVEL=debug
APP_MODE=production
EOF
kubectl create configmap app-env --from-env-file=app.env
```

Difference between 2 and 3: `--from-file=app.env` stores the whole file as ONE key named `app.env`; `--from-env-file=app.env` parses it into one key per line. Mixing them up is the number-one reason `envFrom` "does nothing".

**4. Declarative YAML** — write the manifest above and `kubectl apply -f`. This is the only method that fits GitOps/code review.

Generate YAML without creating anything (dry-run from lesson 35/36):

```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=debug \
  --dry-run=client -o yaml > app-config.yaml
```

Inspect:

```bash
kubectl get configmap                      # alias: cm
kubectl describe configmap app-config      # shows Data section, all keys
kubectl get cm app-config -o yaml
kubectl get cm app-config -o jsonpath='{.data.LOG_LEVEL}'
```

### Injecting a ConfigMap — method 1: single env var (`configMapKeyRef`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo level=$LOG_LEVEL mode=$MODE; sleep 3600"]
    env:
    - name: LOG_LEVEL              # name inside the container
      valueFrom:
        configMapKeyRef:
          name: app-config         # ConfigMap name
          key: LOG_LEVEL           # key in that ConfigMap
    - name: MODE                   # rename on the way in
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_MODE
          optional: true           # Pod starts even if key/ConfigMap is missing
```

- Pick exactly which keys become variables, and rename them.
- Without `optional: true`, a missing ConfigMap or key leaves the Pod in `CreateContainerConfigError`.

### Injecting a ConfigMap — method 2: all keys (`envFrom`)

```yaml
    envFrom:
    - configMapRef:
        name: app-config
      prefix: CFG_              # optional: CFG_LOG_LEVEL, CFG_APP_MODE
```

- Every key becomes an env var with the same name.
- Keys that are not valid env var names (contain `-` or `.`, e.g. `app.properties`) are **skipped silently** (an event is logged on the Pod). `kubectl describe pod` → Events shows `Warning InvalidEnvironmentVariableNames`.
- If two sources define the same name, the **later** entry wins; `env:` entries override `envFrom`.

### Injecting a ConfigMap — method 3: volume (files)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-demo
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: conf
      mountPath: /etc/nginx/conf.d     # directory in the container
      readOnly: true
  volumes:
  - name: conf
    configMap:
      name: nginx-conf
```

Result: every key in the ConfigMap becomes a file under `/etc/nginx/conf.d`, content = value.

Select/rename keys with `items`, set permissions with `defaultMode`:

```yaml
  volumes:
  - name: conf
    configMap:
      name: nginx-conf
      defaultMode: 0440            # octal, applies to all files (YAML: 0440 or 288 decimal)
      items:
      - key: site.conf
        path: default.conf         # file name inside the mount
        mode: 0444                 # per-file override
```

With `items`, **only listed keys** are projected; others are ignored.

**Mounting a volume over an existing directory hides everything that was there in the image.** Mount `/etc/nginx` and the stock `mime.types` vanishes → nginx fails. Mount a dedicated directory (`/etc/nginx/conf.d`) or use `subPath` for one file.

### `subPath` — mount one file without hiding the directory

```yaml
    volumeMounts:
    - name: conf
      mountPath: /etc/nginx/nginx.conf   # a FILE path
      subPath: nginx.conf                # key in the ConfigMap
```

Only that one file is placed; siblings stay intact. **Trade-off:** files mounted with `subPath` are **never updated** when the ConfigMap changes (kubelet uses a bind mount of a fixed file, not the symlink swap described below). Use `subPath` only for config that is read once at startup and restarted anyway.

### How volume updates work (and why env vars do not update)

- **Env vars** are resolved once, when the container starts. Editing the ConfigMap afterwards changes nothing in running Pods. A restart is required.
- **Volume-mounted ConfigMaps** are refreshed by the kubelet. The mount contains symlinks: `/etc/conf/key → ..data/key`, and `..data → ..2024_05_01_12_00_00.123456/`. On a change the kubelet writes a new timestamped directory and atomically flips the `..data` symlink. Delay = kubelet sync period (~60 s) + cache TTL, so **up to roughly 1-2 minutes**.
- The application must re-read the file (inotify watch, SIGHUP, periodic reload). nginx needs `nginx -s reload`; many apps never re-read. Updated file ≠ updated behaviour.

```bash
kubectl exec vol-demo -- ls -la /etc/nginx/conf.d     # see ..data symlink and timestamped dir
kubectl edit cm nginx-conf                            # change something
# wait ~1 min
kubectl exec vol-demo -- cat /etc/nginx/conf.d/site.conf
```

### Forcing a rollout after config change

Pods do not restart when a ConfigMap/Secret changes. Options:

1. **Manual:** `kubectl rollout restart deployment/web` (lesson 36) — new Pods read the new values.
2. **Hash in the Pod template:** add an annotation that changes with the config; any change in the Pod template triggers a rolling update:
   ```yaml
   spec:
     template:
       metadata:
         annotations:
           config-version: "2"      # bump on every config change
   ```
   Tools automate this by hashing the config content into an annotation (Helm lesson 54 shows `sha256sum`; Kustomize lessons 56-58 `configMapGenerator` appends a content hash to the ConfigMap name).
3. **Versioned names:** `app-config-v2`, change the Pod reference, apply → rolling update, easy rollback (`kubectl rollout undo` goes back to the old name).

### Immutable ConfigMaps and Secrets

```yaml
immutable: true
```

- Cannot change `data`/`binaryData`; only delete and recreate. Safer (no accidental live edit breaking all Pods) and cheaper for the API server on large clusters (kubelet stops watching the object).
- Pairs naturally with the versioned-name rollout strategy above.

### Secret anatomy

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque                      # default; arbitrary user data
data:                             # values MUST be base64
  username: YXBwdXNlcg==
  password: czNjcjN0UEBzcw==
stringData:                       # values in plain text; API server encodes for you
  api-token: abc123
```

- `data` values are base64-encoded. `stringData` is a **write-only convenience**: the API server merges it into `data` on write; reading the object back shows only `data`. If a key exists in both, `stringData` wins.
- Size limit 1 MiB, like ConfigMap.
- Names/keys follow the same character rules as ConfigMap.

**Base64 is NOT encryption.** Anyone who can `kubectl get secret -o yaml` can decode instantly:

```bash
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

Encode/decode helpers:

```bash
echo -n 's3cr3tP@ss' | base64          # -n is critical: no trailing newline
echo 'czNjcjN0UEBzcw==' | base64 -d
```

Forgetting `-n` puts `\n` inside the password; the app then fails authentication with a mysteriously "correct" password. Always `echo -n` or use `stringData` / `kubectl create secret`.

### Secret types

| Type | Purpose | Required keys |
|---|---|---|
| `Opaque` (default) | arbitrary user-defined data | none |
| `kubernetes.io/dockerconfigjson` | registry credentials for pulling images | `.dockerconfigjson` |
| `kubernetes.io/tls` | TLS certificate + private key | `tls.crt`, `tls.key` |
| `kubernetes.io/basic-auth` | username + password | `username`, `password` |
| `kubernetes.io/ssh-auth` | SSH private key | `ssh-privatekey` |
| `kubernetes.io/service-account-token` | ServiceAccount token (legacy auto-created; now issued as short-lived projected tokens) | managed by control plane |
| `bootstrap.kubernetes.io/token` | node bootstrap tokens (kubeadm) | managed by cluster |

Typed Secrets are **validated**: a `kubernetes.io/tls` Secret without `tls.crt` and `tls.key` is rejected. The type also tells tools (Ingress lesson 38, kubelet image pull) how to consume it.

### Creating Secrets

```bash
# Opaque from literals
kubectl create secret generic db-creds \
  --from-literal=username=appuser \
  --from-literal=password='s3cr3tP@ss'

# From files (file name = key, content = value; no base64 needed on the CLI)
kubectl create secret generic ssh-key --from-file=id_rsa=./id_rsa

# From a dotenv file
kubectl create secret generic app-secrets --from-env-file=secrets.env

# Docker registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=ci-bot \
  --docker-password='REGISTRY_TOKEN' \
  --docker-email=ci@example.com

# TLS (cert + key, PEM files)
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key

# basic-auth
kubectl create secret generic basic --type=kubernetes.io/basic-auth \
  --from-literal=username=admin --from-literal=password=pa55
```

Quote values with special characters (`$`, `!`, `&`) to stop the shell from expanding them.

Inspect (values hidden by `describe`, visible with `-o yaml`):

```bash
kubectl get secrets
kubectl describe secret db-creds          # shows key names and byte sizes, not values
kubectl get secret db-creds -o yaml       # base64 values visible
```

### Using Secrets: single env var (`secretKeyRef`)

```yaml
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
```

The container sees the **decoded** value. Same failure mode as ConfigMaps: missing Secret/key → `CreateContainerConfigError` (unless `optional: true`).

### Using Secrets: `envFrom`

```yaml
    envFrom:
    - secretRef:
        name: app-secrets
```

Every key becomes a variable. Combine with `configMapRef` in the same list.

### Using Secrets: volume mount as files

```yaml
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: creds
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: creds
    secret:
      secretName: db-creds          # NOTE: secretName, not name
      defaultMode: 0400             # owner read-only
      items:
      - key: password
        path: db/password           # /etc/secrets/db/password
```

- Files contain decoded values. Mounted on a **tmpfs** (RAM-backed), never written to node disk.
- Default file mode is `0644`; tighten to `0400`/`0440` for keys. If the container runs as a non-root user, set `securityContext.fsGroup` on the Pod so the group can read (security contexts are explored further in lessons 48-49).
- Updates propagate like ConfigMap volumes (symlink swap, ~1 min); `subPath` mounts never update.
- Prefer volume mounts over env vars for sensitive data. `kubectl describe pod` shows only the `secretKeyRef` reference, not the value, but env vars are inherited by child processes, visible in `/proc/<pid>/environ`, and commonly dumped by crash reporters and debug endpoints (`printenv`, `/debug/vars`). Files are easier to scope and rotate.

### `imagePullSecrets` — private registry access

Docker registry Secret from `kubectl create secret docker-registry regcred ...` is consumed by the kubelet when pulling images:

```yaml
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/team/app:1.4.2
```

Attach it once to the **default ServiceAccount** of a namespace so every Pod in it inherits it:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"regcred"}]}'
```

(ServiceAccounts are explained in lesson 47; today it is just "an object Pods use by default".) Symptom of a missing/wrong pull secret: `ErrImagePull` → `ImagePullBackOff`, Event `pull access denied` / `unauthorized`.

### Projected volumes — combine sources in one directory

A `projected` volume merges ConfigMaps, Secrets (and other sources) into **one** mount point:

```yaml
  volumes:
  - name: all-config
    projected:
      defaultMode: 0440
      sources:
      - configMap:
          name: app-config
      - secret:
          name: db-creds
          items:
          - key: password
            path: db-password
```

Useful when an app expects one config directory but values come from different objects. Key names must be unique across sources.

### Are Secrets actually secure?

Out of the box, **only mildly**:

| Risk | Detail | Mitigation |
|---|---|---|
| base64 only | decode is trivial | treat base64 as obfuscation, nothing more |
| Plain text in etcd | anyone with etcd/disk/backup access reads all Secrets | **encryption at rest** (`EncryptionConfiguration` with `aescbc`/`secretbox` or a KMS provider) on the API server — concept only; configured by cluster admins, verified by reading the Secret straight from etcd |
| Broad API access | `get`/`list` on `secrets` in a namespace = reading every password there; `list` also leaks values | least-privilege RBAC — Roles/RoleBindings are the subject of lesson 47; the rule: grant `get` on specific `resourceNames`, avoid `list`/`watch` on secrets, never grant cluster-wide |
| Anyone who can create a Pod in the namespace can mount any Secret | Pod creation = Secret read | restrict Pod creation, namespace-per-team |
| Git leaks | `kubectl get secret -o yaml > secret.yaml; git add` | **never commit** Secret manifests; see below |
| Env var leakage | logs, crash dumps, child processes | mount as files, avoid logging env |
| No rotation, no audit | native Secrets are static objects | external secret managers (Vault + External Secrets Operator, lesson 43) |

### Best practices

1. **Never commit Secret manifests with real values to Git.** Base64 in a repo is a plaintext password in a repo. Alternatives: create Secrets imperatively out-of-band (`kubectl create secret ...` from CI variables), use sealed/encrypted manifests (concept: Sealed Secrets, SOPS), or sync from an external vault (lesson 43 ESO). Add `*secret*.yaml` to `.gitignore` as a safety net.
2. **Separate ConfigMap and Secret** — never put passwords in ConfigMaps ("it's only a test") because ConfigMaps are readable by far more roles.
3. **Least privilege on Secrets** — pointer: RBAC in lesson 47. Review who can `get secrets`.
4. **Scope per app and namespace** — one Secret per application, not one giant shared Secret.
5. **Prefer volume mounts, `defaultMode: 0400`**, for sensitive material.
6. **`immutable: true`** + versioned names for safe rollouts and rollbacks.
7. **Rotate** — update the Secret, then `kubectl rollout restart` (env) or let the app reload (files). Short-lived credentials beat long-lived ones.
8. **Do not bake config into images**, and do not echo Secrets into logs (`set -x` in entrypoint scripts!).
9. **Enable encryption at rest** on clusters you operate; on managed clusters (EKS/GKE/AKS) turn on the KMS envelope-encryption option.
10. **Keep config small and typed**: strings only, <1 MiB; bigger → volume/object store.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl create configmap NAME --from-literal=K=V` | ConfigMap from inline pairs | `kubectl create configmap app-config --from-literal=LOG_LEVEL=debug` |
| `kubectl create configmap NAME --from-file=PATH` | file or directory → keys | `kubectl create configmap nginx-conf --from-file=nginx.conf` |
| `kubectl create configmap NAME --from-env-file=F` | dotenv lines → separate keys | `kubectl create configmap app-env --from-env-file=app.env` |
| `kubectl get cm` / `describe cm NAME` | list / inspect ConfigMaps | `kubectl describe cm app-config` |
| `kubectl get cm NAME -o jsonpath='{.data.KEY}'` | read one value | `kubectl get cm app-config -o jsonpath='{.data.LOG_LEVEL}'` |
| `kubectl create secret generic NAME --from-literal=K=V` | Opaque Secret | `kubectl create secret generic db-creds --from-literal=password=s3cr3t` |
| `kubectl create secret generic NAME --from-file=K=PATH` | Secret from file | `kubectl create secret generic ssh-key --from-file=id_rsa` |
| `kubectl create secret docker-registry NAME ...` | registry pull credentials | `kubectl create secret docker-registry regcred --docker-server=R --docker-username=U --docker-password=P` |
| `kubectl create secret tls NAME --cert= --key=` | TLS Secret | `kubectl create secret tls web-tls --cert=tls.crt --key=tls.key` |
| `kubectl get secret NAME -o yaml` | show Secret (base64) | `kubectl get secret db-creds -o yaml` |
| `kubectl get secret NAME -o jsonpath='{.data.K}' \| base64 -d` | decode one key | `kubectl get secret db-creds -o jsonpath='{.data.password}' \| base64 -d` |
| `kubectl describe secret NAME` | key names + sizes, no values | `kubectl describe secret db-creds` |
| `--dry-run=client -o yaml` | generate manifest without creating | `kubectl create cm x --from-literal=a=b --dry-run=client -o yaml` |
| `kubectl rollout restart deploy/NAME` | restart Pods to re-read env config | `kubectl rollout restart deploy/web` |
| `kubectl exec POD -- env` | inspect injected env vars | `kubectl exec env-demo -- env \| sort` |
| `kubectl exec POD -- ls -la DIR` | inspect mounted config files | `kubectl exec vol-demo -- ls -la /etc/nginx/conf.d` |
| `kubectl edit cm NAME` | edit live (volume updates propagate) | `kubectl edit cm nginx-conf` |
| `kubectl delete cm,secret NAME` | remove objects | `kubectl delete cm app-config` |
| `echo -n 'x' \| base64` / `base64 -d` | encode / decode | `echo -n 'admin' \| base64` |

### YAML field cheat-sheet

| Where | Field | Meaning |
|---|---|---|
| container | `env[].valueFrom.configMapKeyRef.{name,key,optional}` | one key → one env var |
| container | `env[].valueFrom.secretKeyRef.{name,key,optional}` | same, from Secret |
| container | `envFrom[].configMapRef.name` / `secretRef.name` / `prefix` | all keys → env vars |
| pod | `volumes[].configMap.{name,items,defaultMode,optional}` | ConfigMap as files |
| pod | `volumes[].secret.{secretName,items,defaultMode,optional}` | Secret as files |
| pod | `volumes[].projected.sources[]` | merge several sources into one dir |
| container | `volumeMounts[].{name,mountPath,readOnly,subPath}` | where/how to mount |
| pod | `imagePullSecrets[].name` | registry credentials |
| ConfigMap/Secret | `immutable: true` | freeze data |
| Secret | `type`, `data`, `stringData` | kind, base64 values, plain values |

## Examples / Walkthrough

Working namespace for the whole lesson:

```bash
kubectl create namespace cfg-demo
kubectl config set-context --current --namespace=cfg-demo
```

### Example 1 — ConfigMap three ways, compare results

```bash
kubectl create configmap cm-literal --from-literal=LOG_LEVEL=debug --from-literal=APP_MODE=production

printf 'LOG_LEVEL=debug\nAPP_MODE=production\n' > app.env
kubectl create configmap cm-file    --from-file=app.env          # ONE key "app.env"
kubectl create configmap cm-envfile --from-env-file=app.env      # TWO keys

kubectl get cm cm-literal cm-file cm-envfile
kubectl describe cm cm-file       # note: key is app.env, value is two lines
kubectl describe cm cm-envfile    # note: keys LOG_LEVEL and APP_MODE
```

Teach: `cm-file` fed to `envFrom` yields **no** useful variables (the key `app.env` contains a dot → invalid env name); `cm-envfile` works.

### Example 2 — env vars: single key vs envFrom

```yaml
# env-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | sort | grep -E 'LOG|MODE|CFG'; sleep 3600"]
    env:
    - name: MODE
      valueFrom:
        configMapKeyRef:
          name: cm-literal
          key: APP_MODE
    envFrom:
    - configMapRef:
        name: cm-envfile
      prefix: CFG_
```

```bash
kubectl apply -f env-demo.yaml
kubectl logs env-demo
# CFG_APP_MODE=production
# CFG_LOG_LEVEL=debug
# MODE=production
```

### Example 3 — ConfigMap as files (nginx page)

```bash
cat > index.html <<'EOF'
<h1>Hello from a ConfigMap, version 1</h1>
EOF
kubectl create configmap web-content --from-file=index.html
```

```yaml
# web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: content
      mountPath: /usr/share/nginx/html     # replaces the default dir content
      readOnly: true
  volumes:
  - name: content
    configMap:
      name: web-content
```

```bash
kubectl apply -f web.yaml
kubectl exec web -- ls -la /usr/share/nginx/html
kubectl port-forward pod/web 8080:80 &
curl -s localhost:8080
```

Now edit and watch the volume refresh (no Pod restart):

```bash
kubectl create configmap web-content --from-literal=index.html='<h1>version 2</h1>' \
  --dry-run=client -o yaml | kubectl apply -f -
# wait 60-90 seconds
curl -s localhost:8080                       # version 2
kubectl exec web -- ls -la /usr/share/nginx/html    # ..data symlink, new timestamp dir
```

### Example 4 — env var does NOT refresh

```bash
kubectl create configmap color --from-literal=COLOR=blue
```

```yaml
# color.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: color
spec:
  replicas: 2
  selector:
    matchLabels: {app: color}
  template:
    metadata:
      labels: {app: color}
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "while true; do echo color=$COLOR; sleep 10; done"]
        envFrom:
        - configMapRef:
            name: color
```

```bash
kubectl apply -f color.yaml
kubectl create configmap color --from-literal=COLOR=green --dry-run=client -o yaml | kubectl apply -f -
sleep 30
kubectl logs deploy/color --tail=2          # still blue
kubectl rollout restart deploy/color
kubectl rollout status deploy/color
kubectl logs deploy/color --tail=2          # now green
```

### Example 5 — subPath mounts one file, never updates

```yaml
# subpath.yaml  (mount a single file into /etc without hiding the dir)
apiVersion: v1
kind: Pod
metadata:
  name: subpath-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: cfg
      mountPath: /etc/app.conf
      subPath: app.conf
  volumes:
  - name: cfg
    configMap:
      name: subpath-cm
```

```bash
kubectl create configmap subpath-cm --from-literal=app.conf='mode=a'
kubectl apply -f subpath.yaml
kubectl exec subpath-demo -- cat /etc/app.conf       # mode=a
kubectl create configmap subpath-cm --from-literal=app.conf='mode=b' --dry-run=client -o yaml | kubectl apply -f -
sleep 90
kubectl exec subpath-demo -- cat /etc/app.conf       # STILL mode=a
```

### Example 6 — Secret creation, base64, decode

```bash
kubectl create secret generic db-creds \
  --from-literal=username=appuser \
  --from-literal=password='s3cr3tP@ss'

kubectl get secret db-creds -o yaml        # data: username: YXBwdXNlcg== ...
kubectl describe secret db-creds           # no values shown
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d; echo
```

Show the stringData shortcut:

```yaml
# secret-string.yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-token
type: Opaque
stringData:
  token: "abc-123"
```

```bash
kubectl apply -f secret-string.yaml
kubectl get secret api-token -o yaml       # stringData is gone, data.token holds base64
```

### Example 7 — consume a Secret as env and as files, compare

```yaml
# secret-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username
    volumeMounts:
    - name: creds
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: creds
    secret:
      secretName: db-creds
      defaultMode: 0400
```

```bash
kubectl apply -f secret-demo.yaml
kubectl exec secret-demo -- sh -c 'echo $DB_USER'            # appuser
kubectl exec secret-demo -- ls -l /etc/secrets               # -r-------- username, password
kubectl exec secret-demo -- cat /etc/secrets/password        # s3cr3tP@ss (decoded)
kubectl exec secret-demo -- df /etc/secrets                  # tmpfs: RAM only
kubectl describe pod secret-demo | grep -A2 -i 'DB_USER'     # shows reference, not the value
```

### Example 8 — TLS Secret

```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 30 \
  -keyout tls.key -out tls.crt -subj "/CN=demo.local"
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key
kubectl get secret web-tls -o jsonpath='{.type}'; echo     # kubernetes.io/tls
kubectl get secret web-tls -o jsonpath='{.data}' | head -c 200; echo
```

Tie back to lesson 38: an Ingress `tls.secretName` and a Gateway `certificateRefs` (lesson 39) point at exactly this kind of Secret.

### Example 9 — docker-registry Secret and imagePullSecrets

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=ci-bot --docker-password='TOKEN' --docker-email=ci@example.com
kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | head -c 300; echo
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/team/app:1.0.0
```

Without `imagePullSecrets` on a private image: `kubectl describe pod` → `Failed to pull image ... unauthorized` → `ImagePullBackOff`.

### Example 10 — failure demo: missing key

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    env:
    - name: X
      valueFrom:
        configMapKeyRef:
          name: cm-literal
          key: DOES_NOT_EXIST
```

```bash
kubectl apply -f broken.yaml
kubectl get pod broken          # CreateContainerConfigError
kubectl describe pod broken     # Error: couldn't find key DOES_NOT_EXIST in ConfigMap cfg-demo/cm-literal
```

Fix by correcting the key or adding `optional: true`.

### Example 11 — immutable ConfigMap

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: frozen
data:
  mode: "a"
immutable: true
EOF
kubectl patch cm frozen --type merge -p '{"data":{"mode":"b"}}'
# The ConfigMap "frozen" is invalid: data: Forbidden: field is immutable when `immutable` is set
```

### Cleanup

```bash
kubectl delete namespace cfg-demo
kubectl config set-context --current --namespace=default
```

## Common Pitfalls

- **Unquoted numbers/booleans in `data`** — `data: {PORT: 8080}` is rejected (`cannot unmarshal number into Go struct field ... of type string`). Quote: `PORT: "8080"`.
- **`echo` without `-n` before base64** — newline becomes part of the password; auth fails with the "right" value. Use `echo -n`, `stringData`, or `kubectl create secret`.
- **Believing base64 = secure** — it is encoding; anyone with `get secrets` reads everything. Treat Secret manifests like plaintext passwords.
- **Committing Secret YAML to Git** — permanent leak (history). Rotate immediately and move to out-of-band creation or an external secret manager (lesson 43).
- **Editing a ConfigMap and expecting env vars to change** — env is resolved at container start. `kubectl rollout restart` (or a hashed annotation / versioned name).
- **Expecting `subPath` mounts to update** — they never do. Use whole-directory mounts for reloadable config.
- **Mounting over a directory with image content** — the mount hides existing files (`/etc/nginx` loses `mime.types`). Mount a subdirectory, or use `subPath`/`items` for single files.
- **`--from-file` vs `--from-env-file` confusion** — `--from-file=app.env` yields ONE key; `envFrom` then exports nothing useful.
- **Invalid env var names via `envFrom`** — keys with `.` or `-` are silently skipped; check `kubectl describe pod` events.
- **ConfigMap/Secret in the wrong namespace** — Pods can only reference objects in **their own namespace**. Symptom: `configmap "x" not found`, Pod stuck `ContainerCreating` (volumes) or `CreateContainerConfigError` (env).
- **`secretName` vs `name`** — Secret volumes use `secretName`; ConfigMap volumes use `name`. Wrong field → validation error.
- **File readable by everyone** — default mode 0644. Set `defaultMode: 0400` and match the container user (`fsGroup`/`runAsUser`).
- **Pod created before the ConfigMap/Secret** — volumes block in `ContainerCreating` (`MountVolume.SetUp failed ... not found`) until the object exists; env refs give `CreateContainerConfigError`. The Pod recovers once the object appears (kubelet retries), but env-based Pods may need deletion.
- **Config >1 MiB** — `Request entity too large`. Ship large data differently.
- **Secret values logged by entrypoint scripts** (`set -x`, `env` dumps) — keep secrets out of logs and CI output.
- **Putting passwords in ConfigMaps** — wider read access, no tooling support for masking. Always Secrets.
- **Using Secret `stringData` and expecting to read it back** — it is write-only; read `data`.
- **Private image without `imagePullSecrets`** — `ImagePullBackOff` / `unauthorized`. Secret must be in the same namespace as the Pod.
- **Forgetting that `kubectl apply` of a changed ConfigMap does not trigger a rollout** — Deployment template didn't change, so no new Pods.

## FAQ

**Q: What is the difference between a ConfigMap and a Secret?**
A: Both store key/value data and are consumed identically (env vars or files). ConfigMap is for non-sensitive settings; Secret is for sensitive data, is base64-encoded in the API, is mounted as tmpfs, is hidden from `kubectl describe`, and can be given tighter RBAC and encryption at rest. Secrets are only *marginally* more secure by default — the difference is mostly access control and handling, not cryptography.

**Q: Is a Kubernetes Secret encrypted?**
A: Not by default. It is base64-encoded (reversible by anyone) and stored plaintext in etcd. Protection comes from: RBAC limiting who can read it, encryption at rest configured on the API server (`EncryptionConfiguration`, KMS), restricted etcd access, and not committing it to Git.

**Q: How do I decode a Secret value?**
A: `kubectl get secret NAME -o jsonpath='{.data.KEY}' | base64 -d`. For all keys: `kubectl get secret NAME -o json | jq '.data | map_values(@base64d)'`.

**Q: Why is my env var not updated after I edited the ConfigMap?**
A: Env vars are set once at container start. Restart the Pods: `kubectl rollout restart deployment/NAME`. Volume-mounted files do update (≈1-2 min) but your app must re-read them.

**Q: How long until a mounted ConfigMap/Secret updates inside the Pod?**
A: Roughly the kubelet sync period plus cache TTL — typically up to 60-90 seconds. `subPath` mounts and env vars never update.

**Q: How can I roll out new Pods automatically when config changes?**
A: Kubernetes does not do it natively. Options: change a Pod-template annotation containing a hash of the config; version the ConfigMap name (`app-config-v2`); `kubectl rollout restart`. Helm (lesson 54) and Kustomize (lessons 56-58) automate the hash/name approach.

**Q: What does `envFrom` do with keys like `app.properties`?**
A: Invalid env variable names (containing `.` or `-`) are skipped, with a warning event. Use a volume mount for file-like keys, or rename keys to valid identifiers.

**Q: `data` or `stringData` in a Secret?**
A: `data` requires base64 values; `stringData` accepts plain text and is encoded for you on write. Reading the object always shows `data`. Use `stringData` for hand-written manifests, `kubectl create secret` for everything else. Remember: neither belongs in Git.

**Q: Can a Pod use a ConfigMap or Secret from another namespace?**
A: No. References are namespace-local. Create the object in each namespace (or use tooling that replicates it).

**Q: What happens if the referenced ConfigMap/Secret doesn't exist?**
A: Env references → `CreateContainerConfigError`. Volume references → Pod stuck in `ContainerCreating` with `MountVolume.SetUp failed ... not found`. Set `optional: true` to let the Pod start without it.

**Q: What does `immutable: true` give me?**
A: The data can never be edited — only deleted and recreated. Prevents accidental breaking changes and reduces API server/kubelet watch load. Combine with versioned names for controlled rollouts.

**Q: How big can a ConfigMap or Secret be?**
A: 1 MiB total (etcd object limit). Larger content should live in an image layer, a volume (lesson 41), or object storage.

**Q: Should I use env vars or volume mounts for passwords?**
A: Prefer volume mounts with `defaultMode: 0400`: not inherited by child processes, not in `/proc/*/environ`, not dumped by crash handlers, and updatable without restart. Env vars are acceptable when an app can only read the environment — then restrict who can `exec` into Pods.

**Q: How do I pull images from a private registry?**
A: Create `kubectl create secret docker-registry regcred ...` in the Pod's namespace and reference it in `imagePullSecrets` (or patch the namespace's `default` ServiceAccount so all Pods inherit it).

**Q: How do I store TLS certs for Ingress?**
A: `kubectl create secret tls NAME --cert=tls.crt --key=tls.key`, then reference it in the Ingress `tls[].secretName` (lesson 38) or a Gateway listener `certificateRefs` (lesson 39). cert-manager can create and renew these Secrets automatically.

**Q: Can I put a whole config file in a ConfigMap?**
A: Yes: `--from-file=nginx.conf`, or a YAML key with a `|` block scalar. Mount it as a volume; use `subPath` if you only want that single file in an existing directory (accepting no live updates).

**Q: How do I see what config a running container actually has?**
A: `kubectl exec POD -- env | sort` for variables; `kubectl exec POD -- ls -la /mount/path` and `cat` for files; `kubectl get pod POD -o yaml` for the spec references.

**Q: Who can read my Secrets?**
A: Anyone with `get`/`list` on `secrets` in the namespace (list returns values too), anyone who can create a Pod in that namespace (they can mount it), and anyone with etcd or node access. RBAC to restrict this is covered in lesson 47.

**Q: How do I rotate a password stored in a Secret?**
A: Update the Secret (`kubectl create secret ... --dry-run=client -o yaml | kubectl apply -f -`), then restart workloads using env vars (`rollout restart`) or wait for file refresh and trigger an app reload. Change the credential in the backing system too. External managers (lesson 43) automate this.

**Q: Why are my Secret files owned by root with mode 0644 and my non-root app can't read them / everyone can?**
A: Default mode is 0644, owner root. Set `defaultMode` and `securityContext.fsGroup` so the app's group can read while others cannot.

**Q: Is there a difference between `kubectl create` and `kubectl apply` for ConfigMaps?**
A: `create` fails if the object exists. `apply` updates. Idiomatic update-from-literal: `kubectl create cm X --from-literal=... --dry-run=client -o yaml | kubectl apply -f -`.

**Q: Does `kubectl describe secret` show passwords?**
A: No — only key names and sizes. But `kubectl get secret -o yaml` does (base64). Restrict `get secrets`, not just `describe`.

## Practice / Exercise

### Core

1. **ConfigMap creation drill.** In namespace `lab42`, create three ConfigMaps — `--from-literal` (two keys), `--from-file` of a small properties file, `--from-env-file` of a dotenv file. Use `kubectl describe` and `-o yaml` to explain the key layout difference between the last two.
2. **Env injection.** Write a Pod that reads `LOG_LEVEL` via `configMapKeyRef` and all keys of the env-file ConfigMap via `envFrom` with prefix `APP_`. Verify with `kubectl exec ... -- env | sort`.
3. **Volume injection.** Mount a ConfigMap holding `index.html` into an nginx Deployment (2 replicas) at `/usr/share/nginx/html`. Expose with a Service (lesson 37) or `kubectl port-forward`; `curl` it.
4. **Live update vs env.** Change the ConfigMap content. Measure how long until the file changes in the Pod. Confirm an env-var consumer did NOT change; fix with `kubectl rollout restart`.
5. **Secret basics.** Create `db-creds` (`username`, `password`) with `kubectl create secret generic`. Decode the password from `-o jsonpath`. Re-create it with `stringData` in YAML and observe what `kubectl get -o yaml` shows.
6. **Secret consumption.** Pod that gets `DB_USER` from `secretKeyRef` AND mounts the whole Secret at `/etc/secrets` with `defaultMode: 0400`. Show `ls -l`, `cat`, and `df` (tmpfs) inside the container.
7. **Break and fix.** Reference a non-existent key and a non-existent ConfigMap in two different Pods. Capture the error from `kubectl get pod` and `kubectl describe pod`. Fix with the right key / `optional: true`.
8. **TLS + registry Secrets.** Generate a self-signed cert with `openssl`, create a `kubernetes.io/tls` Secret; create a `docker-registry` Secret (fake credentials) and verify `.dockerconfigjson` decodes to valid JSON.

### Stretch

1. **subPath experiment.** Mount one file with `subPath`, edit the ConfigMap, prove the file does not update. Repeat with a directory mount and prove it does. Explain with the `..data` symlink listing.
2. **Hash-triggered rollout.** Compute `sha256sum` of a config file, write it into a Pod-template annotation, apply; change the config, recompute, re-apply and watch a rolling update occur without `rollout restart`.
3. **Immutable + versioned names.** Create `app-config-v1` (`immutable: true`), a Deployment referencing it, then roll to `app-config-v2`, and `kubectl rollout undo` back. Show the patch-on-immutable error.
4. **Projected volume.** Combine a ConfigMap and a Secret into one `projected` volume at `/etc/app` with `items` renaming; verify the directory listing and modes.
5. **Secret hygiene audit.** Write a `.gitignore` rule for secret manifests; use `kubectl get secrets -A -o json | jq` to list Secrets, their types, and which Pods reference them (via `volumes`/`env` paths). Write 5 sentences on how you would remove the plaintext Secret from this repo's Git history and rotate it.
6. **Encryption-at-rest reading.** In the Kubernetes docs, find "Encrypting Confidential Data at Rest"; write down the three provider types and what `identity` means. (Read-only: do not edit a cluster's API server flags.)

## Further Reading

- ConfigMaps: https://kubernetes.io/docs/concepts/configuration/configmap/
- Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
- Configure a Pod to use a ConfigMap: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/
- Distribute credentials securely using Secrets: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/
- Pull an image from a private registry: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- Encrypting Secret data at rest: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- Good practices for Kubernetes Secrets: https://kubernetes.io/docs/concepts/security/secrets-good-practices/
- Projected volumes: https://kubernetes.io/docs/concepts/storage/projected-volumes/
- The Twelve-Factor App, III. Config: https://12factor.net/config

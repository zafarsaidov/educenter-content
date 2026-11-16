# Lesson 58: Practice – Manage Environments with Kustomize

**Module:** Kustomize – Native K8s Templating
**Duration:** 120-150 min
**Prerequisites:** Lessons 56-57 (Kustomize basics, overlays, images transformer, strategic merge and JSON 6902 patches, configMapGenerator/secretGenerator); lesson 36 (Deployments, rollouts); lesson 37-38 (Services, Ingress); lesson 45 (probes, requests/limits); lesson 53-55 (Helm, for the "Kustomize vs Helm" comparison). A working cluster (kind/minikube/kubeadm/rke2 from lesson 52) and `kubectl` 1.27+.

## Learning Objectives

By end of lesson student can:
- Design a `base/` + `overlays/{dev,staging,prod}` directory layout for a real application and explain what belongs in base and what belongs in an overlay.
- Give each environment its own namespace, replica count, image tag, resource limits, Ingress host and application config without copying any base YAML.
- Use `configMapGenerator` with `behavior: merge` and `behavior: replace` to deliver environment-specific config, and explain why the generated name carries a content hash.
- Add environment-only objects (PodDisruptionBudget in prod) by listing extra `resources:` in one overlay.
- Render, compare and apply environments with `kubectl kustomize`, `kubectl apply -k`, `kubectl diff -k` and `kubectl delete -k`.
- Trigger a rolling update by changing one config literal and observe the hash-suffix rollout mechanism; roll back by `kubectl rollout undo` and by `git revert`.
- Diagnose the common Kustomize failures (bad path, patch that matches nothing, immutable selector, missing namespace, stale tag) from the error text.

## Topics

- Set up base + 3 overlays (dev/staging/prod) for a real application
- Apply different replicas, resource limits, image tags per environment
- Use configMapGenerator for environment-specific app config
- Apply to cluster: `kubectl apply -k overlays/dev`; `kubectl apply -k overlays/prod`
- Verify differences: `kubectl diff -k`; review generated output with `kustomize build`
- Extras (over-fill for live filtering): prod-only PodDisruptionBudget, per-environment Ingress hosts via JSON 6902, rollout on config change, rollback strategies, break-it drills, troubleshooting table

## Concepts

### What we are building

One application, `webshop`, deployed three times into three namespaces of the same cluster:

| Environment | Namespace | Replicas | Image tag | CPU limit | Memory limit | Ingress host | Extra |
|---|---|---|---|---|---|---|---|
| dev | `webshop-dev` | 1 | 6.7.1 (newest) | 100m | 128Mi | `webshop.dev.example.test` | debug logging |
| staging | `webshop-staging` | 2 | 6.7.0 | 250m | 256Mi | `webshop.staging.example.test` | – |
| prod | `webshop-prod` | 3 | 6.5.4 (pinned, proven) | 500m | 512Mi | `webshop.prod.example.test` | PodDisruptionBudget, zero-downtime rollout strategy, own feature flags |

The workload is `podinfo` (`ghcr.io/stefanprodan/podinfo`), a tiny Go HTTP app made for Kubernetes demos. It listens on port 9898, answers `/healthz` and `/readyz`, and returns JSON on `/` that includes the `message`, `color`, `version` and `hostname` fields. It reads `PODINFO_UI_MESSAGE`, `PODINFO_UI_COLOR` and `PODINFO_LEVEL` from environment variables, so a ConfigMap visibly changes what the app says. That makes every Kustomize transformation checkable with `curl`.

> Tag note for the instructor: before class run `docker pull ghcr.io/stefanprodan/podinfo:6.7.1`, `:6.7.0` and `:6.5.4` (or `crane ls`) to confirm the tags still exist in the registry. If one is gone, substitute any three real tags - the lab only needs three different ones.

### Base vs overlay: the dividing line

- **Base** = everything true in *every* environment: the Deployment skeleton (container, port, probes, env wiring), the Service, the Ingress shape, the default config values. A base must be deployable-looking on its own, but we never apply it directly.
- **Overlay** = only the *differences*: namespace, replica count, tag, limits, host name, config overrides, extra objects. If an overlay file starts to look like a copy of a base file, the base is probably missing a default or you should be using a patch instead.

Rule of thumb: when you add a new environment, you should create **one directory with ~5 small files**, never copy `deployment.yaml`.

### Why no copy-paste of YAML

Without Kustomize the three environments would be three copies of `deployment.yaml` that drift (somebody adds a probe to prod only, staging forgets the new env var). With base + overlays there is exactly one Deployment definition; an environment's file only states "make it 3 replicas". `kustomize build` computes the final YAML; the cluster never sees the overlay files, only the merged result.

### The tools used in this lab

- `kubectl kustomize <dir>` renders the final YAML to stdout (same as `kustomize build <dir>`).
- `kubectl apply -k <dir>` renders and applies in one step.
- `kubectl diff -k <dir>` renders and shows what *would* change on the live cluster (a server-side dry run diff). Exit code `0` = no changes, `1` = differences found, `>1` = error.
- `kubectl delete -k <dir>` renders and deletes every object the overlay produces.
- Standalone `kustomize` binary (optional): newer than the copy embedded in kubectl, needed for `kustomize edit`. Check both with `kubectl version --client` and `kustomize version`.

### Names, hashes and why Deployments roll

`configMapGenerator` appends a hash of the content to the ConfigMap name (`webshop-config-7kh2tf8d9m`) and rewrites every reference to it (`envFrom.configMapRef.name`, `volumes[].configMap.name`) in the same build. Change a literal → new hash → new name → the Deployment's pod template changes → a normal rolling update happens by itself. Kubernetes alone does *not* restart pods when a ConfigMap's content changes in place; the hash trick is how Kustomize gets that behaviour. The hash is deterministic: same content always gives the same name, which is what makes `git revert` a clean rollback.

### Merge vs replace in generators

When an overlay declares a generator with the same `name` as the base:

- `behavior: merge` - keep the base keys, override or add the ones listed in the overlay. Use for "same config, a few values differ".
- `behavior: replace` - throw away the base content, use only the overlay's. Use for "totally different file in prod".
- `behavior: create` (default) - make a new ConfigMap; if the name collides with an existing one the build fails with an ID conflict.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl kustomize DIR` | Render overlay to YAML (no cluster needed) | `kubectl kustomize overlays/prod` |
| `kustomize build DIR` | Same, with standalone binary | `kustomize build overlays/prod` |
| `kubectl apply -k DIR` | Render + apply | `kubectl apply -k overlays/dev` |
| `kubectl diff -k DIR` | Show pending changes vs live cluster | `kubectl diff -k overlays/prod; echo $?` |
| `kubectl delete -k DIR` | Delete everything the overlay renders | `kubectl delete -k overlays/dev` |
| `kubectl apply -k DIR --dry-run=server` | Validate against the API server without persisting | `kubectl apply -k overlays/prod --dry-run=server` |
| `kubectl get -k DIR` | Show live state of the rendered objects | `kubectl get -k overlays/staging` |
| `kustomize edit set image` | Change `images:` entry from CLI (CI-friendly) | `cd overlays/prod && kustomize edit set image ghcr.io/stefanprodan/podinfo=ghcr.io/stefanprodan/podinfo:6.7.1` |
| `kustomize edit set namespace` | Change `namespace:` from CLI | `kustomize edit set namespace webshop-qa` |
| `kubectl rollout status deploy/NAME -n NS` | Wait for a rollout to finish | `kubectl rollout status deploy/webshop -n webshop-prod` |
| `kubectl rollout undo deploy/NAME -n NS` | Go back to previous ReplicaSet | `kubectl rollout undo deploy/webshop -n webshop-prod` |
| `diff <(kubectl kustomize A) <(kubectl kustomize B)` | Compare two rendered environments | `diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)` |
| `kubectl port-forward svc/NAME L:P -n NS` | Reach the Service from your laptop | `kubectl -n webshop-dev port-forward svc/webshop 8081:80` |

## Examples / Walkthrough

Work in an empty directory. Every step has a **Checkpoint**; do not move on until it matches. Hash suffixes (`-7kh2tf8d9m`) and pod names will differ on your machine.

### Step 0 - Prerequisites check (5 min)

```bash
kubectl version --client          # client >= 1.27 so the embedded kustomize supports `labels:`
kubectl get nodes                 # cluster reachable, nodes Ready
kubectl kustomize --help | head -3
which kustomize || echo "standalone kustomize not installed (optional)"
which jq || echo "install jq for prettier curl output"
```

**Checkpoint:** nodes are `Ready`; `kubectl kustomize --help` prints usage. If you want Ingress to actually answer, ingress-nginx from lesson 38 must be installed - the lab works without it because Ingress objects are created either way and we test through `port-forward`.

### Step 1 - Scaffold the project and put it under Git (5 min)

```bash
mkdir -p webshop-kustomize && cd webshop-kustomize
git init
mkdir -p base overlays/dev overlays/staging overlays/prod
```

Target tree (we will fill it in step by step):

```text
webshop-kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── features.properties
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── patch-resources.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── patch-resources.yaml
    └── prod/
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── patch-resources.yaml
        ├── patch-strategy.yaml
        ├── pdb.yaml
        └── features.properties
```

**Checkpoint:** `find . -type d -not -path './.git*'` lists `base` and the three overlay directories.

### Step 2 - Write the base (15 min)

`base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webshop
  labels:
    app.kubernetes.io/name: webshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: webshop
  template:
    metadata:
      labels:
        app.kubernetes.io/name: webshop
    spec:
      containers:
        - name: podinfo
          image: ghcr.io/stefanprodan/podinfo:6.7.1
          ports:
            - name: http
              containerPort: 9898
          envFrom:
            - configMapRef:
                name: webshop-config        # Kustomize rewrites this to the hashed name
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
          volumeMounts:
            - name: features
              mountPath: /etc/webshop
              readOnly: true
      volumes:
        - name: features
          configMap:
            name: webshop-features           # also rewritten to hashed name
```

`base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webshop
  labels:
    app.kubernetes.io/name: webshop
spec:
  selector:
    app.kubernetes.io/name: webshop
  ports:
    - name: http
      port: 80
      targetPort: http
```

`base/ingress.yaml` (the host is a placeholder every overlay replaces):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webshop
  labels:
    app.kubernetes.io/name: webshop
spec:
  ingressClassName: nginx
  rules:
    - host: webshop.example.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webshop
                port:
                  number: 80
```

`base/features.properties` (a *file* ConfigMap, mounted at `/etc/webshop/features.properties`):

```properties
checkout.enabled=true
reviews.enabled=false
banner.text=base
```

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml

# Non-selector label: safe to change later (includeSelectors defaults to false)
labels:
  - pairs:
      app.kubernetes.io/part-of: webshop
    includeSelectors: false

configMapGenerator:
  # env-style config injected with envFrom
  - name: webshop-config
    literals:
      - APP_ENV=base
      - PODINFO_UI_MESSAGE=webshop (base)
      - "PODINFO_UI_COLOR=#34577c"
      - PODINFO_LEVEL=info
  # file-style config mounted as a volume
  - name: webshop-features
    files:
      - features.properties
```

Render the base alone (never apply it - it has no namespace and no environment identity):

```bash
kubectl kustomize base | head -60
kubectl kustomize base | grep -E '^kind:|^  name:'
```

Expected (trimmed):

```text
kind: ConfigMap
  name: webshop-config-4g6hk5mc9d
kind: ConfigMap
  name: webshop-features-b4tbk2t8g7
kind: Service
  name: webshop
kind: Deployment
  name: webshop
kind: Ingress
  name: webshop
```

Look at the rendered Deployment: `envFrom.configMapRef.name` and `volumes[0].configMap.name` now hold the **hashed** names.

```bash
git add -A && git commit -m "base: webshop deployment, service, ingress, generated config"
```

**Checkpoint:** `kubectl kustomize base` succeeds with no error and the Deployment references `webshop-config-<hash>`. Then `kubectl apply -k base --dry-run=server -o name` returns object names without creating anything (it will say the namespace is `default`; that is why we do not apply base for real).

### Step 3 - The dev overlay (15 min)

`overlays/dev/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webshop-dev
```

`overlays/dev/patch-resources.yaml` - a **strategic merge patch**: only the fields that differ; the `containers` list is merged by `name`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webshop
spec:
  template:
    spec:
      containers:
        - name: podinfo
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 128Mi
```

`overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - namespace.yaml

namespace: webshop-dev

labels:
  - pairs:
      env: dev
    includeSelectors: false

images:
  - name: ghcr.io/stefanprodan/podinfo
    newTag: 6.7.1

replicas:
  - name: webshop
    count: 1

patches:
  # strategic merge patch from a file
  - path: patch-resources.yaml
  # JSON 6902 patch inline, aimed with a target
  - target:
      kind: Ingress
      name: webshop
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: webshop.dev.example.test

configMapGenerator:
  - name: webshop-config
    behavior: merge
    literals:
      - APP_ENV=dev
      - PODINFO_UI_MESSAGE=Hello from DEV
      - "PODINFO_UI_COLOR=#2e8b57"
      - PODINFO_LEVEL=debug
```

Why each block exists:

- `resources: ../../base` pulls in the base; `namespace.yaml` adds the Namespace object itself so `apply -k` creates it and `delete -k` removes it.
- `namespace:` stamps `webshop-dev` onto every namespaced object in the build.
- `labels:` adds `env: dev` for filtering (`kubectl get all -l env=dev -A`) without touching selectors.
- `images:` swaps the tag. The match is on the image *name*; the tag in the base file is ignored.
- `replicas:` is a shortcut transformer for `spec.replicas` (it also works on StatefulSets etc.). A strategic merge patch on `spec.replicas` gives the same result.
- `patches:` mixes both patch styles in one list.
- `configMapGenerator ... behavior: merge` keeps base keys and overrides these four.

Render and read it:

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/dev | grep -E 'namespace:|replicas:|image:|host:|cpu:|memory:|APP_ENV|PODINFO_'
```

Expected (trimmed):

```text
  APP_ENV: dev
  PODINFO_LEVEL: debug
  PODINFO_UI_COLOR: '#2e8b57'
  PODINFO_UI_MESSAGE: Hello from DEV
  namespace: webshop-dev
  replicas: 1
        image: ghcr.io/stefanprodan/podinfo:6.7.1
            cpu: 100m
            memory: 128Mi
            cpu: 25m
            memory: 32Mi
  - host: webshop.dev.example.test
```

```bash
git add -A && git commit -m "overlay dev"
```

**Checkpoint:** all five dev-specific values (namespace, replicas, tag, limits `100m/128Mi`, host) appear in the rendered output, and the base file is unchanged (`git diff HEAD~1 -- base` is empty).

### Step 4 - Apply dev and verify it works (10 min)

```bash
kubectl apply -k overlays/dev
kubectl -n webshop-dev rollout status deploy/webshop
kubectl -n webshop-dev get deploy,svc,ingress,cm,pods
```

Expected:

```text
namespace/webshop-dev created
configmap/webshop-config-7kh2tf8d9m created
configmap/webshop-features-b4tbk2t8g7 created
service/webshop created
deployment.apps/webshop created
ingress.networking.k8s.io/webshop created
...
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webshop   1/1     1            1           25s
```

Prove that the configuration reached the app:

```bash
kubectl -n webshop-dev port-forward svc/webshop 8081:80 >/dev/null &
sleep 2
curl -s localhost:8081 | jq '{message, color, version, hostname}'
kubectl -n webshop-dev exec deploy/webshop -- env | grep -E 'APP_ENV|PODINFO_'
kubectl -n webshop-dev exec deploy/webshop -- cat /etc/webshop/features.properties
kill %1
```

Expected: `"message": "Hello from DEV"`, `"version": "6.7.1"`, `APP_ENV=dev`, and the base `features.properties` content (dev did not override it).

**Checkpoint:** `curl` returns the DEV message and version 6.7.1. If the pod is not Ready: `kubectl -n webshop-dev describe pod` and see the troubleshooting table at the end.

### Step 5 - The staging overlay (10 min)

Staging is the dev overlay with different numbers. Create the three files.

`overlays/staging/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webshop-staging
```

`overlays/staging/patch-resources.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webshop
spec:
  template:
    spec:
      containers:
        - name: podinfo
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

`overlays/staging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - namespace.yaml

namespace: webshop-staging

labels:
  - pairs:
      env: staging
    includeSelectors: false

images:
  - name: ghcr.io/stefanprodan/podinfo
    newTag: 6.7.0

replicas:
  - name: webshop
    count: 2

patches:
  - path: patch-resources.yaml
  - target:
      kind: Ingress
      name: webshop
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: webshop.staging.example.test

configMapGenerator:
  - name: webshop-config
    behavior: merge
    literals:
      - APP_ENV=staging
      - PODINFO_UI_MESSAGE=Hello from STAGING
      - "PODINFO_UI_COLOR=#d98e04"
```

Apply and check:

```bash
kubectl apply -k overlays/staging
kubectl -n webshop-staging rollout status deploy/webshop
kubectl -n webshop-staging get deploy,pods -o wide
git add -A && git commit -m "overlay staging"
```

**Checkpoint:** `READY 2/2` in `webshop-staging`; `kubectl -n webshop-staging get deploy webshop -o jsonpath='{.spec.template.spec.containers[0].image}'` prints `...:6.7.0`.

### Step 6 - The prod overlay with extras (20 min)

Prod differs in more ways: pinned older proven tag, 3 replicas, bigger limits, a **zero-downtime rollout strategy**, a **PodDisruptionBudget**, and a **different feature-flag file** (`behavior: replace`).

`overlays/prod/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webshop-prod
```

`overlays/prod/patch-resources.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webshop
spec:
  template:
    spec:
      containers:
        - name: podinfo
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

`overlays/prod/patch-strategy.yaml` - a separate patch file because it is a separate concern (small focused patches are easier to review):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webshop
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

`overlays/prod/pdb.yaml` - exists only in prod, so it is listed only in the prod overlay's `resources:`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webshop
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: webshop
```

`overlays/prod/features.properties` - completely different content (replaces the base file):

```properties
checkout.enabled=true
reviews.enabled=true
banner.text=Spring sale
maintenance.mode=false
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - namespace.yaml
  - pdb.yaml                 # prod-only object

namespace: webshop-prod

labels:
  - pairs:
      env: prod
    includeSelectors: false

images:
  - name: ghcr.io/stefanprodan/podinfo
    newTag: 6.5.4

replicas:
  - name: webshop
    count: 3

patches:
  - path: patch-resources.yaml
  - path: patch-strategy.yaml
  - target:
      kind: Ingress
      name: webshop
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: webshop.prod.example.test

configMapGenerator:
  - name: webshop-config
    behavior: merge
    literals:
      - APP_ENV=prod
      - PODINFO_UI_MESSAGE=Welcome to the shop
      - "PODINFO_UI_COLOR=#8b1a1a"
      - PODINFO_LEVEL=warn
  - name: webshop-features
    behavior: replace
    files:
      - features.properties
```

Render the prod-only pieces:

```bash
kubectl kustomize overlays/prod | grep -E '^kind:'
kubectl kustomize overlays/prod | grep -A8 'strategy:'
kubectl kustomize overlays/prod | grep -B2 -A6 'kind: PodDisruptionBudget'
```

Expected: the kind list now includes `PodDisruptionBudget` (dev/staging do not), and `maxUnavailable: 0` is under `strategy`.

```bash
git add -A && git commit -m "overlay prod"
```

**Checkpoint:** both overlays render a Namespace, so `kubectl kustomize overlays/prod | grep -c '^kind:'` must be exactly one higher than the same count for `overlays/dev` (the extra object is the PDB). Run both and compare.

### Step 7 - Compare environments before applying prod (10 min)

```bash
# What differs between dev and prod, as a diff of the rendered YAML
diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)

# Short summary table with grep
for e in dev staging prod; do
  echo "== $e"
  kubectl kustomize overlays/$e | grep -E '^  namespace:|^  replicas:|image:|limits:' -A0
done
```

`diff` output is long but readable: you see exactly the lines Kustomize changed, which is the review artifact you would paste into a pull request.

A reusable check script - save as `verify.sh` in the project root:

```bash
#!/usr/bin/env bash
set -euo pipefail
for env in dev staging prod; do
  ns="webshop-$env"
  echo "== $env ($ns)"
  kubectl -n "$ns" get deploy webshop -o jsonpath='replicas={.spec.replicas} image={.spec.template.spec.containers[0].image} cpu-limit={.spec.template.spec.containers[0].resources.limits.cpu} mem-limit={.spec.template.spec.containers[0].resources.limits.memory}{"\n"}'
  kubectl -n "$ns" get cm -o name
  kubectl -n "$ns" get ingress webshop -o jsonpath='host={.spec.rules[0].host}{"\n"}'
done
```

```bash
chmod +x verify.sh
```

### Step 8 - Preview with `kubectl diff -k`, then apply prod (10 min)

Prod does not exist yet, so the diff shows everything as new:

```bash
kubectl diff -k overlays/prod | head -40; echo "exit=${PIPESTATUS[0]}"
```

Expected: a long `+` listing of Namespace/ConfigMaps/Service/Deployment/Ingress/PDB and `exit=1` (differences found). Now apply:

```bash
kubectl apply -k overlays/prod
kubectl -n webshop-prod rollout status deploy/webshop
kubectl -n webshop-prod get deploy,pdb,ingress,cm,pods
./verify.sh
```

Expected `verify.sh` output:

```text
== dev (webshop-dev)
replicas=1 image=ghcr.io/stefanprodan/podinfo:6.7.1 cpu-limit=100m mem-limit=128Mi
configmap/webshop-config-7kh2tf8d9m
configmap/webshop-features-b4tbk2t8g7
host=webshop.dev.example.test
== staging (webshop-staging)
replicas=2 image=ghcr.io/stefanprodan/podinfo:6.7.0 cpu-limit=250m mem-limit=256Mi
...
== prod (webshop-prod)
replicas=3 image=ghcr.io/stefanprodan/podinfo:6.5.4 cpu-limit=500m mem-limit=512Mi
...
host=webshop.prod.example.test
```

Check prod functionally, and confirm its feature file was **replaced** while the env config was **merged**:

```bash
kubectl -n webshop-prod port-forward svc/webshop 8083:80 >/dev/null &
sleep 2
curl -s localhost:8083 | jq '{message, version}'
kill %1
kubectl -n webshop-prod exec deploy/webshop -- cat /etc/webshop/features.properties
kubectl -n webshop-prod get pdb
```

Expected: message `Welcome to the shop`, version `6.5.4`, features file shows `banner.text=Spring sale` and `maintenance.mode=false`, PDB shows `MIN AVAILABLE 2` and `ALLOWED DISRUPTIONS 1`.

**Checkpoint:** `./verify.sh` prints three different lines for replicas/image/limits/host, and prod's PDB exists.

### Step 9 - Drift detection with `kubectl diff -k` (5 min)

After a successful apply, the diff must be empty:

```bash
kubectl diff -k overlays/prod; echo "exit=$?"     # no output, exit=0
```

Now make a deliberate out-of-band change (a "hotfix at 3 a.m."):

```bash
kubectl -n webshop-prod scale deploy/webshop --replicas=5
kubectl diff -k overlays/prod; echo "exit=$?"
```

Expected:

```text
-  replicas: 5
+  replicas: 3
exit=1
```

The diff shows the live cluster has drifted from Git. Re-applying brings it back:

```bash
kubectl apply -k overlays/prod
kubectl -n webshop-prod get deploy webshop        # 3/3 again
```

Teaching point: Git (the overlay) is the source of truth; emergency manual edits are overwritten by the next `apply -k`. If the hotfix should stay, edit the overlay.

### Step 10 - Change config, watch the hash-suffix rollout (10 min)

Edit `overlays/prod/kustomization.yaml`: change `PODINFO_UI_MESSAGE=Welcome to the shop` to `PODINFO_UI_MESSAGE=Welcome to the shop - v2`.

```bash
kubectl diff -k overlays/prod | grep -E '^[+-]' | grep -v '^[+-]{3}'
```

You will see a **new ConfigMap name** (`webshop-config-<newhash>`) appear in the Deployment's `envFrom` and a new ConfigMap object added - not an in-place edit of the old one.

```bash
kubectl apply -k overlays/prod
kubectl -n webshop-prod rollout status deploy/webshop
kubectl -n webshop-prod get cm
kubectl -n webshop-prod get rs
kubectl -n webshop-prod rollout history deploy/webshop
```

Expected:

```text
NAME                            DATA   AGE
webshop-config-7kh2tf8d9m       4      12m      <- old, still there
webshop-config-9m4t2cdd68       4      20s      <- new
webshop-features-xxxxxxxxxx     4      12m
...
NAME                 DESIRED   CURRENT   READY
webshop-6b8d9c7f5d   3         3         3        <- new ReplicaSet
webshop-7f65c9d86b   0         0         0        <- previous
```

Because `maxUnavailable: 0, maxSurge: 1` is set in prod, pods are replaced one at a time and capacity never drops below 3. Compare with dev (default 25%/25% with 1 replica): there is a short moment with a different shape of rollout.

Confirm the app now says v2:

```bash
kubectl -n webshop-prod port-forward svc/webshop 8083:80 >/dev/null & sleep 2; curl -s localhost:8083 | jq .message; kill %1
git add -A && git commit -m "prod: message v2"
```

**Checkpoint:** two ReplicaSets, two config ConfigMaps in prod, message shows `v2`. Old ConfigMaps are *not* deleted by `apply -k` - that is normal (see pitfalls).

### Step 11 - Roll back two ways (10 min)

**Way 1 - cluster-side: `kubectl rollout undo`.** Fast, in an emergency.

```bash
kubectl -n webshop-prod rollout undo deploy/webshop
kubectl -n webshop-prod rollout status deploy/webshop
kubectl diff -k overlays/prod; echo "exit=$?"
```

The app is back on the old config (the old hashed ConfigMap still exists, which is why undo works), but `diff` now reports a difference: **Git still says v2**. A rollback made only in the cluster is invisible to Git and the next `apply -k` will roll *forward* again.

**Way 2 - Git-side: `git revert` then apply.** The durable way.

```bash
kubectl apply -k overlays/prod                       # first, re-sync to Git (v2)
git revert --no-edit HEAD                             # undo the "message v2" commit
kubectl diff -k overlays/prod                         # shows the move back
kubectl apply -k overlays/prod
kubectl -n webshop-prod get rs                        # the OLD ReplicaSet is scaled up again, no third RS is created
```

Why no new ReplicaSet? The hash is computed from content; the reverted literal gives the old content, so the old name `webshop-config-7kh2tf8d9m`, so the pod template is byte-identical to revision 1, so the Deployment controller reuses the existing ReplicaSet. Deterministic names turn Git history into a rollback mechanism.

**Checkpoint:** after way 2, `kubectl diff -k overlays/prod` is empty (exit 0) and `curl` shows `Welcome to the shop`.

### Step 12 - Break it on purpose (15 min)

Do these in a scratch copy (`cp -r overlays/dev overlays/dev-broken` and edit it) or revert with `git checkout .` after each. Predict the error before running.

**Drill A - wrong path in `patches:`**

Change `path: patch-resources.yaml` to `path: patch-resource.yaml` in an overlay.

```bash
kubectl kustomize overlays/dev
```

Error (wording varies by version): `error: ... evalsymlink failure on '.../patch-resource.yaml' : lstat ...: no such file or directory`. Fix: file names are exact; `ls` the overlay directory.

**Drill B - patch container name typo**

In `patch-resources.yaml` change `- name: podinfo` to `- name: podinf0`.

```bash
kubectl kustomize overlays/dev | grep -B2 -A12 'podinf0'
kubectl apply -k overlays/dev --dry-run=server
```

Kustomize does **not** fail - the list merge is keyed by `name`, so it *adds a second container* with no image. The API server rejects it: `spec.template.spec.containers[1].image: Required value`. Lesson: a patch that does not match silently becomes an addition; always read the rendered output, and run `--dry-run=server`.

**Drill C - immutable selector via `commonLabels`**

Add this to `overlays/dev/kustomization.yaml`:

```yaml
commonLabels:
  env: dev
```

```bash
kubectl kustomize overlays/dev | grep -B1 -A4 'matchLabels'    # selector now contains env: dev
kubectl apply -k overlays/dev
```

Error: `The Deployment "webshop" is invalid: spec.selector: Invalid value: ...: field is immutable`. `commonLabels` (deprecated) rewrites **selectors too**, and a Deployment selector cannot change. Fix: remove `commonLabels`, use `labels:` with `includeSelectors: false` (what we did). If the object already exists with the wrong selector you must delete and recreate it.

**Drill D - base reference broken**

Change `../../base` to `../base` in an overlay.

```bash
kubectl kustomize overlays/dev
```

Error: `accumulating resources ... must be a directory or a file ... '../base' ...no such file`. Relative paths are relative to the `kustomization.yaml`, not to your shell's current directory.

**Drill E - forgetting the `images` override**

Delete the `images:` block from the staging overlay, render, and read the Deployment's `image:`. It silently falls back to the base tag (`6.7.1`) - staging is no longer running what you think. Nothing errors. This is why `verify.sh` prints the image per environment.

Restore everything: `git checkout . && git clean -fd overlays`.

**Checkpoint:** you can explain, for each drill, *which layer* complained (Kustomize build, API server, or nobody).

### Step 13 - Cleanup (5 min)

```bash
kubectl delete -k overlays/dev
kubectl delete -k overlays/staging
kubectl delete -k overlays/prod
kubectl get ns | grep webshop || echo "all webshop namespaces gone"
```

Because each overlay lists `namespace.yaml`, `delete -k` removes the namespace too, which removes anything left inside it (including orphaned old ConfigMaps from step 10). If a PDB blocks nothing here; PDBs only guard evictions, not deletes.

## Common Pitfalls

- **Applying `base/` directly** — it has no environment identity (default namespace, base tag). Only apply overlays. Add a README line or CI guard if the team keeps doing it.
- **`commonLabels` changes selectors** — breaks an existing Deployment with `field is immutable`. Prefer `labels:` with `includeSelectors: false` for descriptive labels; use selectors only for labels that will never change.
- **Patch that matches nothing is not an error for container lists** — a typo in a container `name` creates a second container. For JSON 6902 patches a wrong `target` name fails with `no matches for Id`; a wrong `path` fails with an apply-time error. Always render and read.
- **Old generated ConfigMaps pile up** — `apply -k` never deletes the previous hashed ConfigMap. Clean periodically (`kubectl get cm`), or prune by label, or rely on namespace lifecycle. They are also what lets `rollout undo` work, so do not delete them blindly.
- **`disableNameSuffixHash: true` removes the rolling-update trigger** — pods keep running the old config until restarted manually (`kubectl rollout restart`). Only use it when something outside Kustomize references the ConfigMap by a fixed name.
- **Forgetting `images:` in an overlay** — falls back silently to the base tag. Make your verify step print the running image.
- **`behavior: merge/replace` without a matching base generator** — build error `merging from generator ... id ... does not exist; cannot merge or replace`. The generator `name` must equal the base's name exactly.
- **Missing Namespace object** — `apply -k` fails with `namespaces "webshop-dev" not found` if you stamp `namespace:` but never create the Namespace. Either include `namespace.yaml` or `kubectl create ns` first.
- **Secrets in overlays committed to Git** — `secretGenerator` literals are plain text in the repo. Do not put real passwords in overlays; this lab deliberately has none (see lesson 43 for ESO).
- **Relying on the kubectl-embedded kustomize version** — older kubectl embeds an old Kustomize without `labels:`, `replacements:` or newer behaviours. `kubectl version --client` and `kustomize version` can differ; if a field is rejected as unknown, install the standalone binary and use `kustomize build | kubectl apply -f -`.
- **`kubectl diff` exit code misread** — `1` means "differences", not "failure". In scripts check for `>1` as the real error.
- **Hotfixing with `kubectl scale/edit`** — the change is lost at the next `apply -k`. Change the overlay and commit.

## FAQ

**Q: Why do I need `namespace.yaml` if `namespace:` already sets the namespace?**
A: The `namespace:` field only *stamps* the namespace name onto objects; it does not create the Namespace. Without the Namespace object (or a prior `kubectl create ns`) the apply fails with "namespace not found". Including it also means `kubectl delete -k` removes the namespace during cleanup.

**Q: Why is there no `namePrefix` in this lab?**
A: Each environment has its own namespace, so identical object names (`webshop`) do not collide. Use `namePrefix`/`nameSuffix` when two environments must share one namespace (for example `dev-webshop` and `qa-webshop` side by side). Kustomize also rewrites references (Ingress backend, `envFrom`) when it renames. It is a good stretch exercise.

**Q: Should replicas be set with the `replicas:` transformer or a patch?**
A: Both work. The `replicas:` transformer is shorter and clearer for the single question "how many". A patch is better when you are already patching the same Deployment for other reasons (strategy, resources). Do not do both for the same field in one overlay; it is confusing which wins.

**Q: What is the difference between the `patches:` entry with `path` and the one with `target` and inline `patch`?**
A: `path:` points to a file; Kustomize decides the format from the content - a YAML object means strategic merge, a YAML list of `op`/`path`/`value` means JSON 6902. An inline `patch: |-` is the same content written in place. JSON 6902 patches need a `target` because the patch body has no `metadata.name`; strategic merge patches identify their object by `apiVersion/kind/metadata.name` in the body.

**Q: When do I choose strategic merge vs JSON 6902?**
A: Strategic merge for "set/override these fields" - readable, merges lists by key (containers by `name`). JSON 6902 for exact operations: replacing one element by index (`/spec/rules/0/host`), removing a field, or adding an item to a list that strategic merge would treat unpredictably.

**Q: What does `behavior: merge` do to keys that exist only in the base?**
A: They are kept. Only keys present in the overlay's generator are added or overridden. `replace` discards all base keys and uses only the overlay's content.

**Q: Why did my pods restart when I only changed a ConfigMap literal?**
A: Because the generated ConfigMap name includes a hash of its content. A new literal gives a new name, the Deployment's pod template now references a different ConfigMap, and the Deployment controller starts a rolling update. That is intentional and is the whole point of the hash suffix.

**Q: Can I update a ConfigMap in place and have pods pick it up without restart?**
A: Files mounted as a volume are eventually refreshed by the kubelet, but env vars never are, and applications must re-read the file. Generated, hashed ConfigMaps avoid the ambiguity: new config means new pods, always.

**Q: `kubectl diff -k` printed nothing but I changed the overlay. Why?**
A: Either the change does not alter the rendered output (for example a patch targeting a field already at that value) or you edited a file that no `kustomization.yaml` references. Run `kubectl kustomize overlays/prod` and grep for your change.

**Q: Does `kubectl apply -k` delete objects I removed from the overlay?**
A: No. `apply` never deletes objects that disappeared from the manifests unless you use pruning. If you remove `pdb.yaml` from prod's `resources:` the PDB stays in the cluster. Delete it manually or use `kubectl delete -k` on the old version of the overlay, or prune with a label selector (`kubectl apply -k ... --prune -l ...`; read the docs first, prune is easy to misuse).

**Q: How do I promote a release from dev to staging to prod?**
A: Change only the `images:` `newTag` in the next overlay (by hand, or `kustomize edit set image` in a script), commit, apply. Base never changes for a plain version bump. Automating this step belongs to the CI/CD module later in the bootcamp.

**Q: Can one overlay build on another (for example `qa` based on `staging`)?**
A: Yes. An overlay's `resources:` can point at another overlay directory (`../staging`). Keep chains short (two levels at most) or it becomes hard to see where a value comes from. Use `kubectl kustomize` on each level while debugging.

**Q: How is this different from the Helm chart we built in lesson 55?**
A: Helm templates YAML with variables (`{{ .Values.replicas }}`) and packages/versions a release; Kustomize leaves base YAML valid and patches it, with no template language and nothing to install. Kustomize is lighter for "my own app in several environments"; Helm is better for distributing configurable software to others and for release history. They combine: `helm template ... | kubectl kustomize` style pipelines (lesson 57) or Kustomize overlays over rendered Helm output.

**Q: Can I put real Secrets in `secretGenerator` in these overlays?**
A: Technically yes, but the values would live in plain text in Git. For anything real use an external secret store (lesson 43). For throwaway lab credentials it is fine.

**Q: `kubectl apply -k` says `error: unknown field "labels"` / `unknown field "replicas"`.**
A: Your kubectl embeds an old Kustomize. Upgrade kubectl, or install standalone `kustomize` and run `kustomize build overlays/dev | kubectl apply -f -`.

**Q: How do I see exactly what Kustomize will send without a cluster?**
A: `kubectl kustomize overlays/prod > /tmp/prod.yaml` renders locally. Add `kubectl apply -f /tmp/prod.yaml --dry-run=client` for schema-level validation, or `--dry-run=server` (needs a cluster) for full API validation including admission.

## Troubleshooting Table

| Symptom | Likely cause | How to confirm | Fix |
|---|---|---|---|
| `error: accumulating resources: ... no such file or directory` | Wrong relative path in `resources:` | `ls` from the directory containing the `kustomization.yaml` | Fix path (relative to that file, not to your shell) |
| `evalsymlink failure on '.../patch-x.yaml'` | `patches: path` typo | `ls overlays/<env>` | Correct the file name |
| `namespaces "webshop-dev" not found` | Namespace object not in resources | `kubectl get ns` | Add `namespace.yaml` to overlay `resources:` or create the namespace first |
| `field is immutable` on Deployment `spec.selector` | `commonLabels` changed the selector | `kubectl kustomize ... \| grep -A4 matchLabels` vs live object | Remove `commonLabels`, use `labels:` with `includeSelectors: false`; delete/recreate the Deployment once |
| `containers[1].image: Required value` | Patch container `name` typo created an extra container | `kubectl kustomize ... \| grep -c 'name: podin'` | Match the exact container name from base |
| `merging from generator ... cannot merge or replace` | Overlay generator `name` differs from base, or base has none | Compare the two `name:` values | Use the identical name |
| `may not add resource with an already registered id` | Two generators/resources with same kind+name, `behavior` omitted | Read the id in the error | Add `behavior: merge`/`replace` or rename |
| Pod `ImagePullBackOff` | Tag from overlay does not exist, or registry blocked | `kubectl describe pod` events | Use an existing tag; check `images.newTag` spelling |
| Pod `CrashLoopBackOff` after config change | Bad env value for the app (for example invalid `PODINFO_LEVEL`) | `kubectl logs --previous` | Fix the literal; `git revert` and re-apply |
| Pods Running but not Ready | Probe port/path wrong, or resources far too low | `kubectl describe pod`; `kubectl get events` | Align probes with app; raise requests/limits in the overlay patch |
| Pods `Pending` with `Insufficient cpu` | Requests too high for the lab cluster (3 prod replicas x 250m) | `kubectl describe pod` shows FailedScheduling | Lower prod requests in `patch-resources.yaml` for the lab, or add nodes |
| `OOMKilled` | Memory limit too low | `kubectl describe pod` → `Last State: OOMKilled` | Raise limit in the overlay |
| Env label/replicas not what I expect | Another transformer/patch also touches the field | `kubectl kustomize` and search the field; look for duplicate settings | Keep one mechanism per field per overlay |
| Image tag is the base one in staging | `images:` block missing or `name` does not match the base image string | Render and grep `image:` | Match `name:` exactly to the image repo in base (without tag) |
| `kubectl diff -k` always shows a change | Server-defaulted fields or a controller (HPA, mutating webhook) rewrites the object | Read the diff lines | Exclude that field from management or accept the noise; confirm with `--server-side` |
| `rollout undo` worked but next apply reverted it | Git still contains the bad change | `kubectl diff -k` is non-empty | `git revert` the commit and apply |
| Old ConfigMaps accumulating | Normal: apply never deletes superseded hashed names | `kubectl get cm` | Periodic cleanup / prune by label / rely on namespace deletion |
| Ingress has no address | No controller installed or `ingressClassName` mismatch | `kubectl get ingressclass`, `kubectl -n webshop-dev describe ingress` | Install ingress-nginx (lesson 38); use port-forward for this lab |
| `unknown field "labels"` / `"replicas"` | Embedded Kustomize too old | `kubectl version --client`, `kustomize version` | Upgrade or use standalone `kustomize build \| kubectl apply -f -` |

## Practice / Exercise

### Core (do all, ~45 min)

1. Re-create the whole project from memory in a new directory (`webshop2`) using only `kubectl kustomize` to test; apply only dev. Time yourself. Goal: under 20 minutes.
2. Add a fourth environment `qa` (namespace `webshop-qa`, 2 replicas, tag `6.7.1`, message `Hello from QA`) by creating **one new directory with at most four files** and **zero edits** to `base/` or other overlays. Prove with `git status` that nothing else changed.
3. In staging, add a new key `FEATURE_X=on` to `webshop-config` using `behavior: merge`, apply, and show with `kubectl exec ... env` that the pods restarted and picked it up. Record the old and new ConfigMap names.
4. Use `kubectl diff -k` to preview a change that raises prod replicas to 4 **without applying it**; paste the diff and the exit code into your notes.
5. Produce `diff <(kubectl kustomize overlays/staging) <(kubectl kustomize overlays/prod)` and write one sentence explaining each differing hunk (tag, replicas, limits, strategy, PDB, host, config).
6. Do rollback both ways (`rollout undo` and `git revert`) in staging and write down the difference in outcome of `kubectl diff -k` after each.

### Stretch

1. Make prod pods **Guaranteed** QoS (requests equal to limits, lesson 45) through the overlay patch only, and prove it with `kubectl get pod -o jsonpath='{.items[0].status.qosClass}'`.
2. Replace the `replicas:` transformer in prod with a strategic merge patch on `spec.replicas`. Show the rendered output is identical (`diff` of before/after builds is empty).
3. Add `namePrefix: dev-` to the dev overlay and observe what Kustomize rewrites (Ingress backend, Deployment `envFrom`, Service). Identify one reference it does **not** rewrite automatically if you add a custom CRD field; discuss why.
4. Write a JSON 6902 patch for prod that **adds** an Ingress annotation `nginx.ingress.kubernetes.io/proxy-body-size: "8m"` using `op: add` on `/metadata/annotations/...` (remember `~1` escaping for `/` in the key). Make it fail first with a wrong path, then fix it.
5. Install standalone `kustomize` and promote dev's tag with `kustomize edit set image ...` from a tiny shell script `promote.sh <env> <tag>`; commit the result and show the diff.
6. Add a `secretGenerator` entry to dev with a literal `DB_PASSWORD=devpass` (lab only), mount it via `envFrom`, and explain in two sentences why this is acceptable in a lab but not in the real prod overlay.
7. Take the Helm chart from lesson 55, render it with `helm template shop ./shop > rendered/shop.yaml`, wrap it in a `kustomization.yaml` (`resources: - rendered/shop.yaml`) and patch one value per environment. List two advantages and two disadvantages of this combination.

## Further Reading

- Kustomize docs, reference and examples: https://kubectl.docs.kubernetes.io/references/kustomize/
- Kubernetes docs - Declarative management with Kustomize: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
- `kustomization` file field reference (`images`, `replicas`, `labels`, `patches`, generators): https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/
- Strategic merge vs JSON 6902 patches: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/
- podinfo (demo application used in this lab): https://github.com/stefanprodan/podinfo
- Kubernetes docs - PodDisruptionBudget: https://kubernetes.io/docs/tasks/run-application/configure-pdb/

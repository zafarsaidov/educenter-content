# Lesson 56: Kustomize Basics – Overlays and Resources

**Module:** Kustomize – Native K8s Templating
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-55 (kubectl, Deployments, Services, ConfigMaps, Namespaces, labels/selectors, kubectl diff, Helm charts and values). Students have a working cluster (kubeadm/rke2 from lesson 52, or kind/minikube) and `kubectl` configured. Students just finished Helm (templating with `{{ }}`) — today we meet the opposite philosophy: **no templates at all**.

## Learning Objectives

By end of lesson student can:
- Explain the Kustomize philosophy: template-free, declarative, patch/transform-based customization of plain YAML.
- Contrast Kustomize (transform valid YAML) with Helm (render templates from values) in one paragraph.
- Run Kustomize three ways: `kubectl apply -k`, `kubectl kustomize`, and the standalone `kustomize` CLI; explain why versions can differ.
- Write a `kustomization.yaml` with `apiVersion`, `kind`, `resources`, `namePrefix`, `nameSuffix`, `namespace`, `labels`, `commonLabels`, `commonAnnotations`.
- Build the standard layout: `base/` plus `overlays/dev` and `overlays/prod`, each overlay pulling the base via `resources: - ../../base`.
- Explain what Kustomize rewrites automatically when names change (references to ConfigMaps, Secrets, ServiceAccounts, Services in Ingress) and what it does **not** rewrite (label selectors, hostnames, env var values).
- Use the `images` transformer (`name`, `newName`, `newTag`, `digest`) to set a different image per environment without editing the base.
- Use the `replicas` transformer for per-environment replica counts.
- Render and read output with `kustomize build` / `kubectl kustomize`, preview with `kubectl diff -k`, remove with `kubectl delete -k`.
- Reference remote bases (Git URL with `?ref=`) and know the supply-chain risk.
- Diagnose the common errors: accumulating resources, "security; file is not in or below", already-registered resource ID, immutable selector after changing `commonLabels`, namespace not created.

## Topics

- Kustomize concepts: no templating, patch-based, built into kubectl (kubectl apply -k)
- kustomization.yaml: resources, namePrefix, nameSuffix, commonLabels, namespace
- Bases and overlays: directory structure (base/, overlays/dev/, overlays/prod/)
- Images transformer: overriding image name and tag per environment
- kubectl apply -k, kubectl kustomize (build); kustomize CLI

## Concepts

### Why another tool? The environment problem again

Lesson 53 opened with the pain of copy-pasting YAML for dev/staging/prod. Helm solved it by turning YAML into *templates* with holes (`{{ .Values.replicas }}`). That works but has costs:

- The templates are no longer valid YAML. You cannot `kubectl apply -f` them, editors cannot lint them, and reading `{{- if }}` blocks inside YAML is hard.
- To expose a new knob (say, an extra annotation) the chart author must add a value and a template block. Consumers of a third-party chart can only change what the author anticipated.

Kustomize takes the other road:

1. Start from **plain, valid Kubernetes YAML** (the *base*). It works on its own — you can `kubectl apply -f` it.
2. Describe **transformations** on top of it in a small file called `kustomization.yaml`: "add this prefix, put everything in this namespace, change this image tag, set 5 replicas".
3. Kustomize reads the base, applies the transformations, and prints the final YAML. **No variables, no `{{ }}`, no rendering language.**

One-sentence definition: *Kustomize customizes Kubernetes manifests by layering declarative transformations over plain YAML, without templates.*

| | Helm | Kustomize |
|---|---|---|
| Source files | Templates (not valid YAML) | Plain valid YAML |
| Customization | `values.yaml` fills blanks the author left | Transformers/patches modify any field |
| Packaging/versioning | Charts, repos, releases, rollback | None — it is just a build step; Git is your history |
| Lifecycle | `helm install/upgrade/rollback` | `kubectl apply -k` (state tracked by kubectl, not by Kustomize) |
| Learning curve | Go templating | YAML only |
| Third-party apps | Huge ecosystem of charts | Few upstream "kustomize bases"; usually used for *your own* apps |

The formal trade-off discussion and using the two together is lesson 57. Today: learn the mechanics well.

### Where Kustomize lives: three ways to run it

1. **`kubectl apply -k <dir>`** — built into kubectl since v1.14. Builds the directory and applies the result. `-k` means "treat this argument as a kustomization directory" (compare `-f` for files).
2. **`kubectl kustomize <dir>`** — only *builds*: prints final YAML to stdout, touches no cluster. Think `helm template`.
3. **`kustomize build <dir>`** — the standalone binary. Same job as (2), but newer features, plus helper subcommands (`kustomize create`, `kustomize edit ...`).

**Version difference (important in support questions):** kubectl embeds a *copy* of Kustomize at the moment that kubectl release was cut, so it is often a few minor versions behind the standalone release. `kubectl version` prints the embedded Kustomize version (look for `Kustomize Version: v5.x.y`). If a feature from the Kustomize release notes "doesn't work", compare the embedded version with `kustomize version`. Rule of thumb: use `kubectl kustomize` for quick demos; use the standalone CLI in CI so the version is pinned independently of kubectl.

Install the standalone CLI:

```bash
# macOS
brew install kustomize

# Linux (official script; downloads latest into the current directory)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/

# verify
kustomize version
kubectl version --client        # shows the embedded Kustomize version
```

### The kustomization.yaml file

A **kustomization** is a directory containing a file named `kustomization.yaml` (also accepted: `kustomization.yml`, `Kustomization`). It lists *what to include* and *what to transform*.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

`apiVersion` and `kind` are technically optional (older tools filled them in), but always write them — newer versions warn when absent and editors give schema help.

Key fields covered today:

| Field | What it does |
|---|---|
| `resources` | List of things to include: YAML files, **directories** (another kustomization), or remote URLs |
| `namePrefix` / `nameSuffix` | Add text to the `metadata.name` of every resource (and fix references) |
| `namespace` | Set `metadata.namespace` on every namespaced resource |
| `labels` | Add labels (optionally also into selectors/templates via `includeSelectors`, `includeTemplates`) |
| `commonLabels` | Older field: add labels everywhere **including selectors** (see pitfall) |
| `commonAnnotations` | Add annotations to every resource |
| `images` | Change image name/tag/digest of matching containers |
| `replicas` | Set the replica count of named Deployments/StatefulSets |

Other fields exist (Kustomize can also patch, generate ConfigMaps/Secrets, and more). Patches and generators are lesson 57; ignore them today.

Order of operations: Kustomize loads `resources`, then applies the transformers (prefix/suffix, namespace, labels, annotations, images, replicas), then emits YAML. You don't control the order inside one file, and don't need to.

### Bases and overlays

- **Base** — a kustomization (directory) holding the *common* manifests for the app. Usually valid on its own.
- **Overlay** — a kustomization that includes a base (via `resources`) and adds environment-specific transformations.

Conventional layout:

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── namespace.yaml
    └── prod/
        ├── kustomization.yaml
        └── namespace.yaml
```

The overlay's `kustomization.yaml` points at the base by **relative path**:

```yaml
resources:
  - ../../base
```

Rules to remember:
- The overlay never modifies files in `base/`. It only layers transformations on the in-memory copy.
- An overlay can include **another overlay** too (stacking: `prod-eu` includes `prod`). Keep stacking shallow — readers must mentally merge each layer.
- `base` and `overlay` are *conventions*, not keywords. Kustomize sees only "a directory with a kustomization.yaml that refers to other directories/files".
- There is no inheritance of `namePrefix`/labels *upwards*: a base cannot see the overlay.

### What gets rewritten automatically ("reference rewriting")

Changing a resource's name could break things that point at it. Kustomize knows many Kubernetes reference fields and rewrites them to match, so adding `namePrefix: dev-` does not break the app:

| Reference | Rewritten? |
|---|---|
| ConfigMap/Secret name in `envFrom.configMapRef.name`, `env.valueFrom.configMapKeyRef.name`, `volumes.configMap.name` | **Yes** |
| `serviceAccountName` pointing at a ServiceAccount in the set | **Yes** |
| Ingress `backend.service.name` → Service | **Yes** |
| HPA `scaleTargetRef.name` → Deployment | **Yes** |
| RoleBinding `roleRef` / `subjects` (ServiceAccount name+namespace) | **Yes** |
| PVC name in `volumes.persistentVolumeClaim.claimName` | **Yes** |
| Service `spec.selector` (labels, not a name) | Not a name reference — selectors change only through label transformers |
| Hostname in code/env var, e.g. `DB_HOST=postgres` | **No** — it is just a string; Kustomize cannot know it is a Service name |
| Service DNS names inside ConfigMap *data* | **No** |

The last two rows are the classic "namePrefix broke my app" cause: the app connects to `backend` but the Service is now `dev-backend`. Fix: do not prefix names an app hard-codes, or make the app config derived (lesson 57 generators and patches help).

Special case: `namePrefix`/`nameSuffix` **skip** `Namespace` objects and CRDs — a Namespace called `dev` stays `dev`. (Always confirm with `kustomize build`.)

### Labels: `labels` vs `commonLabels` (selector immutability trap)

A Deployment's `spec.selector` is **immutable** after creation. A Service selects pods by label. So adding a label *into selectors* is a dangerous operation on an existing app.

- `commonLabels` — adds the labels to **every** resource's `metadata.labels` *and* to Deployment/StatefulSet `spec.selector.matchLabels`, `spec.template.metadata.labels`, and Service `spec.selector`. Convenient, dangerous, and deprecated in favour of `labels`.
- `labels` — newer, explicit:

```yaml
labels:
  - pairs:
      env: dev
    includeSelectors: false    # do NOT touch selectors (default false)
    includeTemplates: true     # add to the pod template labels (so pods carry the label)
```

Behaviour:
- `includeSelectors: false` and `includeTemplates: false` → only the object's own `metadata.labels`.
- `includeTemplates: true` → also the pod template labels (good for `kubectl get pods -l env=dev`).
- `includeSelectors: true` → also selectors (equivalent of the old `commonLabels`). Use only for a brand-new application; never change it later on a running Deployment.

**The trap:** first apply works. A week later someone adds `commonLabels: {team: payments}`. `kubectl apply -k` fails with `field is immutable` on the Deployment (selector changed). Fixes: use `labels` with `includeSelectors: false`, or delete the Deployment and recreate it (downtime).

`commonAnnotations` is simpler: adds annotations to every resource (and pod templates). No selector risk.

### Transformers in one table

| Field | Example | Effect |
|---|---|---|
| `namePrefix: dev-` | `web` → `dev-web` | Prefix for all names (not Namespace) |
| `nameSuffix: -v2` | `web` → `web-v2` | Suffix for all names |
| `namespace: dev` | adds `metadata.namespace: dev` | Namespaced resources only; cluster-scoped untouched |
| `commonAnnotations` | `owner: platform-team` | On every resource |
| `images` | `name: nginx` / `newTag: 1.27.2` | Rewrites matching `image:` fields |
| `replicas` | `name: web` / `count: 3` | Sets `spec.replicas` of that workload |

### The images transformer in detail

Why not just edit the image in the base? Because *base stays env-agnostic* and CI usually injects the tag per build. The `images` list rewrites every container (and initContainer) whose image **name** matches:

```yaml
images:
  - name: nginx            # matches image: nginx or nginx:<anything> in the manifests
    newTag: 1.27.2         # change only the tag
```

Fields:

| Field | Meaning | Example result |
|---|---|---|
| `name` | The image name (without tag/digest) to match, exactly as written in the base | `nginx` |
| `newName` | Replace the name/registry (keeps tag unless `newTag` given) | `registry.example.com/mirror/nginx` |
| `newTag` | Replace the tag | `1.27.2` |
| `digest` | Pin by digest (`sha256:...`); **overrides** the tag, results in `name@sha256:...` | `nginx@sha256:abc...` |

Rules:
- `name` must match what the base has. If the base says `image: nginx:1.26.2`, `name: nginx` matches (tag ignored). If it says `docker.io/library/nginx`, you must write `name: docker.io/library/nginx`.
- If `digest` is set, `newTag` is ignored (digest wins). Digest pinning is the production-grade choice: tags are mutable.
- Kustomize edits `image:` in containers, initContainers, and common pod-spec locations. Image strings buried in env vars or ConfigMap data are untouched.

Helper command (standalone CLI) to edit the file for you — handy in CI:

```bash
cd overlays/prod
kustomize edit set image nginx=nginx:1.27.2
kustomize edit set image nginx=registry.example.com/mirror/nginx:1.27.2
```

### The replicas transformer

```yaml
replicas:
  - name: web        # the *original* resource name in the base (before prefix)
    count: 4
```

Works on Deployment, StatefulSet, ReplicaSet, ReplicationController. The name is matched **before** `namePrefix` is applied — a frequent source of "my replicas aren't changing" (people write `dev-web`). Helper: `kustomize edit set replicas web=4`.

Why this exists: changing only the replica count is so common that a patch would be overkill. Anything beyond (resources/limits, probes, env changes) needs patches → lesson 57.

### Reading the output: `kustomize build` ordering

`kustomize build` writes a multi-document YAML stream separated by `---`. By default it sorts into a "legacy" order: Namespaces first, then ConfigMaps/Secrets/ServiceAccounts, RBAC, Services, and workloads (Deployments) later — so `kubectl apply` of the stream works without ordering errors (a Namespace exists before things inside it). Don't rely on the order of resources *within* a kind.

### Remote bases (brief)

`resources` may point to a Git repository or an HTTP URL:

```yaml
resources:
  - github.com/example-org/platform-bases//web?ref=v1.4.0     # repo//subdir?ref=<tag|branch|sha>
  - https://raw.githubusercontent.com/example-org/repo/v1.4.0/manifests/service.yaml
```

- `//` separates the repo from the subdirectory inside it.
- **Always pin `?ref=`** to a tag or commit SHA. Without it you track the default branch: today's build differs from yesterday's, and a compromised repo changes what you deploy (supply-chain risk).
- Needs `git` and network at build time; builds are slower and not hermetic. For production, vendor the base (copy into your repo) or pin hard.

### Namespace transformer pitfalls

- `namespace: dev` sets the namespace on resources, but does **not create** the Namespace. Add a `namespace.yaml` (kind Namespace) to `resources`, or create it separately. (Namespace objects are skipped by the prefix, as noted above.)
- It overwrites any namespace written in the base manifest. Mixed-namespace bases collapse into one — fine for one app per namespace, bad for a base that deliberately spans namespaces.
- Cluster-scoped kinds (ClusterRole, Namespace, PersistentVolume) are not changed. For ClusterRoleBinding subjects that are ServiceAccounts, the `namespace` field inside `subjects` is updated to match.
- Cross-namespace references written as strings (e.g. `http://db.shared.svc.cluster.local`) are not changed.

### Suggested timeline (instructor)

| Minutes | Block |
|---|---|
| 0-15 | Philosophy; Helm vs Kustomize recap; three ways to run |
| 15-35 | Write base (Deployment, Service, ConfigMap); `kubectl kustomize base`, apply it |
| 35-60 | First overlay: namespace, prefix, labels, annotations; read output; reference rewriting demo |
| 60-80 | images + replicas; `kustomize edit`; `kubectl diff -k` |
| 80-100 | prod overlay; side-by-side comparison; delete -k |
| 100-120 | Pitfalls live: immutable selector, security restrictor, duplicate IDs; remote base |
| 120-150 | Exercises, Q&A |

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl kustomize <dir>` | Build a kustomization, print YAML (no cluster contact) | `kubectl kustomize overlays/dev` |
| `kubectl apply -k <dir>` | Build and apply to the cluster | `kubectl apply -k overlays/dev` |
| `kubectl diff -k <dir>` | Show what would change on the cluster | `kubectl diff -k overlays/prod` |
| `kubectl delete -k <dir>` | Delete everything the kustomization produces | `kubectl delete -k overlays/dev` |
| `kubectl get -k <dir>` | Get the live objects matching the kustomization | `kubectl get -k overlays/dev` |
| `kubectl apply -k <dir> --dry-run=client -o yaml` | Validate client side, show result | `kubectl apply -k overlays/dev --dry-run=client -o yaml` |
| `kubectl apply -k <dir> --dry-run=server` | Validate against the API server (admission, schema) | `kubectl apply -k overlays/dev --dry-run=server` |
| `kustomize build <dir>` | Standalone build | `kustomize build overlays/prod` |
| `kustomize build <dir> \| kubectl apply -f -` | Build with standalone CLI, apply with kubectl | `kustomize build overlays/prod \| kubectl apply -f -` |
| `kustomize build --load-restrictor LoadRestrictionsNone <dir>` | Allow files outside the kustomization root | `kustomize build --load-restrictor LoadRestrictionsNone overlays/dev` |
| `kubectl kustomize --load-restrictor LoadRestrictionsNone <dir>` | Same via kubectl build (`apply -k` has no such flag) | `kubectl kustomize --load-restrictor LoadRestrictionsNone overlays/dev` |
| `kustomize create --autodetect` | Generate `kustomization.yaml` listing YAML in the dir | `cd base && kustomize create --autodetect` |
| `kustomize edit add resource <file>` | Append to `resources` | `kustomize edit add resource ingress.yaml` |
| `kustomize edit set image <name>=<name>:<tag>` | Edit `images` entry | `kustomize edit set image nginx=nginx:1.27.2` |
| `kustomize edit set nameprefix <p>` | Set `namePrefix` | `kustomize edit set nameprefix dev-` |
| `kustomize edit set namespace <ns>` | Set `namespace` | `kustomize edit set namespace dev` |
| `kustomize edit set replicas <name>=<n>` | Edit `replicas` | `kustomize edit set replicas web=3` |
| `kustomize version` | Show standalone CLI version | `kustomize version` |
| `kubectl version` | Includes embedded Kustomize version | `kubectl version` |

`kustomization.yaml` skeleton (every field from today):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev                  # target namespace for namespaced resources
namePrefix: dev-                # prefix on every metadata.name
nameSuffix: -v1                 # suffix on every metadata.name

resources:                      # files, directories, remote URLs
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      app.kubernetes.io/env: dev
    includeSelectors: false
    includeTemplates: true

commonAnnotations:
  owner: platform-team

images:
  - name: nginx
    newName: nginx
    newTag: 1.27.2
    # digest: sha256:<digest>   # overrides newTag

replicas:
  - name: web
    count: 2
```

## Examples / Walkthrough

### Lab setup

Work in a scratch directory. Any cluster works (kind/minikube/rke2). The app is a small nginx web server: Deployment + Service + ConfigMap.

```bash
mkdir -p ~/kustomize-demo/{base,overlays/dev,overlays/prod}
cd ~/kustomize-demo
kubectl version
kubectl config current-context        # confirm you are on a lab cluster, not production!
```

### Step 1 — Write the base as plain YAML

`base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app.kubernetes.io/name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: web
  template:
    metadata:
      labels:
        app.kubernetes.io/name: web
    spec:
      containers:
        - name: web
          image: nginx:1.26.2
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: web-config
```

`base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app.kubernetes.io/name: web
spec:
  selector:
    app.kubernetes.io/name: web
  ports:
    - port: 80
      targetPort: 80
```

`base/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_MODE: "base"
  LOG_LEVEL: "info"
```

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

### Step 2 — Build the base (nothing touches the cluster)

```bash
kubectl kustomize base
```

Expected: the three documents separated by `---`, ConfigMap first, then Service, then Deployment (legacy ordering). Ask students: *what did Kustomize change?* Answer: nothing — it just collected files. Prove it:

```bash
kubectl kustomize base | diff - <(cat base/configmap.yaml; echo '---'; cat base/service.yaml; echo '---'; cat base/deployment.yaml) | head
```

(Differences are only key ordering and blank lines — Kustomize re-serializes YAML.)

The base alone is a valid app. Optionally apply it to the default namespace and clean up:

```bash
kubectl apply -k base
kubectl get deploy,svc,cm
kubectl delete -k base
```

### Step 3 — First overlay: dev

`overlays/dev/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

`overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev
namePrefix: dev-

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: dev
    includeSelectors: false
    includeTemplates: true

commonAnnotations:
  owner: platform-team

images:
  - name: nginx
    newTag: 1.27.2

replicas:
  - name: web
    count: 2
```

Build and read it:

```bash
kubectl kustomize overlays/dev
```

Walk through the (abbreviated) output with students:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    owner: platform-team
  labels:
    environment: dev
  name: dev                     # NOT dev-dev: Namespaces skip the prefix
---
apiVersion: v1
data:
  APP_MODE: base
  LOG_LEVEL: info
kind: ConfigMap
metadata:
  annotations:
    owner: platform-team
  labels:
    environment: dev
  name: dev-web-config          # prefixed
  namespace: dev                # namespace set
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: web
    environment: dev
  name: dev-web                 # prefixed
  namespace: dev
spec:
  selector:
    app.kubernetes.io/name: web # unchanged: includeSelectors is false
  ports: [...]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-web
  namespace: dev
spec:
  replicas: 2                   # from replicas transformer
  selector:
    matchLabels:
      app.kubernetes.io/name: web   # unchanged
  template:
    metadata:
      labels:
        app.kubernetes.io/name: web
        environment: dev        # includeTemplates: true
    spec:
      containers:
        - image: nginx:1.27.2   # from images transformer
          envFrom:
            - configMapRef:
                name: dev-web-config   # reference rewritten automatically
```

Point out the five things: prefix, namespace, label in template, image tag, **ConfigMap reference rewritten**.

Apply it:

```bash
kubectl apply -k overlays/dev
kubectl get all,cm -n dev
kubectl get pods -n dev -l environment=dev
kubectl get deploy -n dev dev-web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Expected: namespace `dev`, Deployment `dev-web` with 2 pods, image `nginx:1.27.2`.

### Step 4 — Prod overlay

`overlays/prod/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod
namePrefix: prod-

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: prod
    includeSelectors: false
    includeTemplates: true

commonAnnotations:
  owner: platform-team
  contact: oncall@example.com

images:
  - name: nginx
    newName: nginx
    newTag: 1.27.2

replicas:
  - name: web
    count: 4
```

```bash
kubectl kustomize overlays/prod | head -40
kubectl apply -k overlays/prod
kubectl get deploy -A -l app.kubernetes.io/name=web
```

Both environments run from the same base — verify base files are untouched: `git status` (if the directory is a Git repo) or `ls -l base`.

### Step 5 — Compare environments

```bash
diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)
```

This is the best teaching moment: the *entire* difference between environments is visible in one diff, no template reading needed.

### Step 6 — Preview changes with `kubectl diff -k`

Change prod replicas to 5 in `overlays/prod/kustomization.yaml`, then:

```bash
kubectl diff -k overlays/prod
```

Expected: unified diff showing `replicas: 4` → `5` for `prod-web` (plus possibly `metadata.generation` and `managedFields` noise lines). Exit code is 1 when there are differences, 0 when none — useful in CI gates. Apply it only if desired.

### Step 7 — Image pinning by digest and registry mirror

Find a real digest (any machine with Docker/crane; for the lesson you may use a placeholder and just `build`, not apply):

```bash
docker pull nginx:1.27.2
docker inspect --format='{{index .RepoDigests 0}}' nginx:1.27.2
# nginx@sha256:<64 hex chars>
```

In `overlays/prod/kustomization.yaml`:

```yaml
images:
  - name: nginx
    newName: registry.example.com/mirror/nginx
    digest: sha256:<paste-the-64-hex-digest>
```

```bash
kubectl kustomize overlays/prod | grep image:
# image: registry.example.com/mirror/nginx@sha256:<digest>
```

Note: `newTag` is gone in this form; digest wins if both present. Revert after the demo unless the mirror exists (else pods hit `ImagePullBackOff`).

### Step 8 — The standalone CLI and `kustomize edit`

```bash
cd overlays/dev
kustomize edit set image nginx=nginx:1.27.3
kustomize edit set replicas web=3
cat kustomization.yaml          # the file was edited in place
cd ../..
kustomize build overlays/dev | kubectl diff -f -
```

Show students that `kustomize edit` is just a YAML editor — the file is the source of truth and goes in Git. In CI the typical pattern is: `kustomize edit set image app=$REGISTRY/app:$GIT_SHA` then `kustomize build | kubectl apply -f -` (CI pipelines are lessons 59+; today just know the command exists).

### Step 9 — Break things on purpose (immutable selector)

```bash
cd overlays/dev
# Add to kustomization.yaml:
#   commonLabels:
#     team: payments
kubectl apply -k .
```

Expected error (abbreviated):

```
The Deployment "dev-web" is invalid: spec.selector: Invalid value: ... field is immutable
```

Explain: `commonLabels` pushed `team: payments` into `spec.selector.matchLabels`; the API server refuses selector changes. Fix: remove `commonLabels`, use the `labels` block with `includeSelectors: false`. Rule: **never add `commonLabels` to an already-deployed kustomization.**

### Step 10 — Break things on purpose (duplicate resource)

Create a second file `overlays/dev/extra-service.yaml` with a Service named `web` (same name as in base, same kind) and add it to `resources`:

```bash
kustomize build overlays/dev
```

Expected error contains: `may not add resource with an already registered id: Service.v1.[noGrp]/web.[noNs]`. Explanation: two inputs produce the same (kind, name, namespace) — Kustomize can't tell which wins. Overriding existing resources is the job of *patches* (lesson 57), not duplicate files.

### Step 11 — Break things on purpose (file outside root)

In `overlays/dev/kustomization.yaml` add `resources: - ../../shared/ingress.yaml` where `shared/` is a sibling of `base/` and `overlays/` but contains a plain file (not a kustomization). Then:

```bash
kubectl kustomize overlays/dev
```

Expected error:

```
security; file '/.../shared/ingress.yaml' is not in or below '/.../overlays/dev'
```

Explanation: by default a kustomization may load **files** only from within its own directory tree; **directories** with their own `kustomization.yaml` (like `../../base`) are fine. Fixes, in order of preference:

1. Make `shared/` a proper kustomization (add `shared/kustomization.yaml` listing `ingress.yaml`) and reference the **directory**: `resources: - ../../shared`.
2. Move the file under the overlay or base.
3. Last resort (build only, not available for `kubectl apply -k`): `kubectl kustomize --load-restrictor LoadRestrictionsNone overlays/dev | kubectl apply -f -`.

### Step 12 — Remote base (read-only demo)

```bash
mkdir -p /tmp/remote-demo && cd /tmp/remote-demo
cat > kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/kubernetes-sigs/kustomize//examples/helloWorld?ref=v5.4.2
namePrefix: demo-
EOF
kubectl kustomize . | head -30
```

(Needs `git` and network; if GitHub is blocked use the VPN — recall the class network situation from earlier lessons. The path/tag are an example: if the build fails, check that the `?ref=` tag exists. Do **not** apply remote content to a real cluster without reading it.)

### Step 13 — Clean up

```bash
kubectl delete -k overlays/dev
kubectl delete -k overlays/prod
kubectl get ns dev prod           # both namespaces were in resources, so they are gone (or Terminating)
```

Explain: `delete -k` deletes exactly the objects the build produces — including the Namespace because we put it in `resources`. If the Namespace were *not* in the kustomization, it would be left behind.

## Common Pitfalls

- **Adding `commonLabels` to a deployed kustomization** — it rewrites `spec.selector`, which is immutable → `field is immutable` error. Use `labels` with `includeSelectors: false`; start new apps correctly because changing later requires delete/recreate.
- **`namespace:` does not create the namespace** — pods fail with `namespaces "dev" not found`. Add a `namespace.yaml` to `resources` (or create it first).
- **`namePrefix` breaks hard-coded hostnames** — `DB_HOST=postgres` stays `postgres` while the Service becomes `dev-postgres`. Only real references (ConfigMap names, SA names, Ingress backends, etc.) are rewritten. Avoid prefixing, or compute hostnames outside app config.
- **`replicas:` name written with the prefix** — the transformer matches the *original* base name (`web`), not `dev-web`.
- **`images:` `name` doesn't match the base** — base says `docker.io/library/nginx`, overlay says `nginx` → no change, silently. Compare with `kubectl kustomize | grep image:`.
- **Both `newTag` and `digest`** — digest wins; the tag is ignored. Choose one.
- **Referencing files outside the kustomization root** — `security; file ... is not in or below ...`. Make the target a directory with its own `kustomization.yaml`, or build with `--load-restrictor LoadRestrictionsNone` (not available in `kubectl apply -k`).
- **Same resource included twice** — `may not add resource with an already registered id`. Happens when an overlay lists a file that base already contains, or two bases overlap. Use patches (lesson 57) to modify, not duplicates.
- **Editing the base for one environment** — defeats the pattern. Base = shared; environment differences belong in overlays.
- **Unpinned remote base** — a URL without `?ref=` follows the default branch; builds change under you. Always pin to a tag or SHA.
- **Mismatched Kustomize versions** — `kubectl kustomize` embeds an older Kustomize than the standalone CLI; a feature works locally but fails in CI (or vice versa). Compare `kubectl version` with `kustomize version`; pin the CLI version in CI.
- **Assuming `kubectl apply -k` tracks releases like Helm** — there is no release object, no `rollback`. History is Git; rollback = apply an older commit. Deleted-from-YAML objects are *not* removed by plain `apply -k` (use `kubectl delete -k` for teardown, or `--prune` carefully).
- **Not previewing** — apply straight to prod without `kubectl diff -k` or `kubectl kustomize | less` first. Make diff a habit.
- **Stacking too many overlays** — overlay of overlay of overlay; nobody can predict the output. Keep to base + one overlay per environment; read with `kustomize build`.

## FAQ

**Q: What is Kustomize in one sentence?**
A: A tool that takes plain Kubernetes YAML and applies declarative transformations (prefix, namespace, labels, images, replicas, patches) to produce the final YAML for an environment, with no templating language.

**Q: Is Kustomize part of kubectl or a separate tool?**
A: Both. `kubectl apply -k` and `kubectl kustomize` use a copy of Kustomize embedded in kubectl. The standalone `kustomize` CLI is a separate binary that is usually newer and has helper subcommands like `kustomize edit`.

**Q: What is the difference between `kubectl apply -k` and `kubectl apply -f`?**
A: `-f` applies the YAML files/URLs/stdin you give it as-is. `-k` takes a *directory containing `kustomization.yaml`*, builds it (applies the transformations), then applies the result.

**Q: What is the difference between `kubectl kustomize` and `kustomize build`?**
A: They do the same job — print the final YAML. `kubectl kustomize` uses the Kustomize version built into kubectl; `kustomize build` uses the standalone binary's version. Neither contacts the cluster.

**Q: What is a base and what is an overlay?**
A: A base is a kustomization holding the common manifests for an app. An overlay is a kustomization that includes a base (`resources: - ../../base`) and adds environment-specific transformations such as namespace, prefix, image tag and replicas.

**Q: Do I have to name the folders `base` and `overlays`?**
A: No. They are conventions. Kustomize only cares that a directory has a `kustomization.yaml` and that `resources` point at valid paths. Teams use `base/` + `overlays/<env>` almost universally because tooling and docs assume it.

**Q: Can an overlay change base files?**
A: No. The base on disk is never modified. The overlay applies transformations to an in-memory copy and outputs the result.

**Q: Why use Kustomize instead of Helm?**
A: Use Kustomize when you own the manifests and want per-environment differences without learning a template language, and when you want everything to stay valid YAML. Use Helm when you want to package, version, and distribute an app, or install third-party software. Many teams use both; the details are lesson 57.

**Q: Does Kustomize have rollbacks and release history like Helm?**
A: No. Kustomize only builds YAML. Version history is your Git history; to roll back, check out the older commit and `kubectl apply -k` again (Deployments also keep their own `kubectl rollout undo` history from lesson 36).

**Q: Why did `namePrefix` not rename my Namespace?**
A: Kustomize deliberately skips Namespace objects (and CRDs) when applying `namePrefix`/`nameSuffix`; a Namespace's name is a reference target for the `namespace:` field. Verify with `kustomize build`.

**Q: I set `namePrefix: dev-` and my app can't reach `backend`. Why?**
A: The Service is now `dev-backend` but the app connects to a hard-coded string `backend`. Kustomize rewrites real reference fields (ConfigMap names, ServiceAccount names, Ingress backend service names…), not arbitrary strings inside env vars or config data. Fix the app config or avoid prefixes for names the app hard-codes.

**Q: What's the difference between `labels` and `commonLabels`?**
A: `commonLabels` adds labels everywhere **including selectors** (dangerous on existing Deployments, deprecated). `labels` lets you choose: `includeSelectors` (selectors) and `includeTemplates` (pod template labels). Prefer `labels` with `includeSelectors: false`.

**Q: Why do I get "field is immutable" after adding a label?**
A: The label ended up in `spec.selector` (via `commonLabels` or `includeSelectors: true`). Selectors can't change after creation. Remove the selector change, or delete and recreate the Deployment.

**Q: How do I set a different image tag per environment?**
A: Put an `images:` entry in each overlay: `name: nginx`, `newTag: 1.27.2`. The base keeps its default. In CI use `kustomize edit set image nginx=nginx:$TAG` to edit the file.

**Q: What is the difference between `newName`, `newTag` and `digest`?**
A: `newName` replaces the image name (and registry), `newTag` replaces only the tag, `digest` pins an immutable content hash (`name@sha256:...`) and overrides the tag. Prefer digests in production.

**Q: How do I change replica count per environment?**
A: Use the `replicas:` list in the overlay: `- name: web` / `count: 4`. The name is the **original** resource name from the base, before any prefix.

**Q: How do I change resource limits or env vars per environment?**
A: That requires patching a field inside the container spec — covered by patches in lesson 57. Today's transformers only cover names, namespace, labels, annotations, image and replica count.

**Q: Where does Kustomize get ConfigMaps from if I don't write them?**
A: Kustomize can generate ConfigMaps and Secrets from files/literals (with content hash suffixes). That is lesson 57. Today the ConfigMap is a normal YAML file in `resources`.

**Q: What does `kubectl diff -k` show?**
A: The difference between the live cluster objects and what `kubectl apply -k` would produce. Exit code 1 means differences exist, 0 means none. Use it before every production apply.

**Q: How do I delete everything an overlay created?**
A: `kubectl delete -k overlays/dev`. It builds the same output and deletes those objects. Namespaces are removed only if the Namespace object is part of the kustomization.

**Q: Does `kubectl apply -k` delete resources I removed from the YAML?**
A: Not by default. Plain `apply` never deletes. Removed objects stay in the cluster until deleted explicitly. (`--prune` with labels exists but is easy to misuse; GitOps tools handle this properly.)

**Q: What does "security; file ... is not in or below ..." mean?**
A: Kustomize's load restrictor blocks `resources:` entries that are plain files outside the kustomization directory. Turn the target folder into its own kustomization and reference the directory, or move the file. `--load-restrictor LoadRestrictionsNone` works for `kustomize build` / `kubectl kustomize` only.

**Q: What does "may not add resource with an already registered id" mean?**
A: Two sources produced a resource with the same kind, name and namespace. Remove the duplicate or modify the existing resource with a patch instead of re-declaring it (patches: lesson 57).

**Q: Can I use remote Git repos as bases?**
A: Yes: `resources: - github.com/org/repo//path?ref=v1.2.3`. Always pin `ref` to a tag/commit, and read what you are about to apply. Prefer vendoring for production.

**Q: Which comes first in the build output: Deployment or Service?**
A: Kustomize sorts using a legacy order (Namespaces, ConfigMaps/Secrets, RBAC, Services, then workloads). Don't depend on order inside the same kind.

**Q: Can I mix Kustomize with the lesson 55 Helm chart?**
A: Yes, it is a common pattern, but it's lesson 57 material. Today stay with plain YAML bases.

**Q: Can Kustomize and `kubectl apply -k` handle CRDs and custom resources?**
A: Yes — they're just YAML objects. Namespace/labels/prefix transformers apply to them too (prefix skips CRD *definitions*). Some transformers need extra configuration to know custom reference fields; beyond today.

**Q: How do I verify what an overlay will deploy without a cluster?**
A: `kubectl kustomize overlays/prod` (or `kustomize build`). Pipe it to a linter or `kubectl apply --dry-run=client -f -`. For schema/admission validation you need a cluster: `kubectl apply -k overlays/prod --dry-run=server`.

## Practice / Exercise

### Core (everyone, ~45 min)

1. **Base:** Create `base/` with a Deployment, a Service and a ConfigMap for nginx (or your own app). Run `kubectl kustomize base` and confirm the output matches your files. Apply it and delete it with `kubectl apply -k base` / `kubectl delete -k base`.
2. **Dev overlay:** Create `overlays/dev` with `namespace: dev`, `namePrefix: dev-`, a `labels` block (`environment: dev`, `includeTemplates: true`, `includeSelectors: false`), `commonAnnotations`, a `namespace.yaml`, and `replicas: 2`. Build it and list **five** differences from the base.
3. **Reference rewriting:** In the dev output, find the ConfigMap reference inside the Deployment and show it now says `dev-web-config`. Remove the ConfigMap from `resources` and rebuild — what does `kubectl apply --dry-run=server` say?
4. **Images:** In a prod overlay set `images` → a different tag from dev. Verify with `kubectl kustomize overlays/prod | grep image:`. Then switch to `digest:` (placeholder digest is fine for `build`) and note what happens to `newTag`.
5. **Compare:** `diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)`. Explain every hunk.
6. **Apply and verify:** Apply both overlays, run `kubectl get deploy,svc,cm -A -l app.kubernetes.io/name=web`, then change prod replicas and use `kubectl diff -k overlays/prod` before applying.
7. **Teardown:** `kubectl delete -k` both overlays; confirm namespaces are gone.

### Stretch (fast finishers, ~30 min)

1. **Immutable selector:** Reproduce the `field is immutable` error by adding `commonLabels` to a deployed overlay. Fix it by switching to `labels` + `includeSelectors: false`. Write down the recovery steps if the Deployment is already live in prod.
2. **Duplicate ID and security restrictor:** Reproduce both errors from Steps 10-11 and fix each one properly (no `LoadRestrictionsNone`).
3. **Third overlay (`staging`):** add `overlays/staging` between dev and prod (3 replicas, tag between dev and prod). Then make `prod` include `staging` as its base — evaluate: is stacking better or worse than three independent overlays off the same base? Write 3 sentences.
4. **`kustomize edit` workflow:** Use only `kustomize edit set image/replicas/nameprefix/namespace` and `kustomize edit add resource` to create a new overlay from scratch (start with `kustomize create`).
5. **Prefix pitfall:** Add a second Service+Deployment `api` to the base, make `web` call it with `API_HOST=api` in the ConfigMap. Deploy dev with `namePrefix: dev-`; watch `web` fail to resolve `api`. Fix it without removing the prefix (hint: you have only today's tools — what are your options? Compare with what lesson 57 will offer).
6. **Remote base:** Build the Step 12 remote example, read the resulting YAML fully, and write down what you would check before applying remote content to a cluster.
7. **Server-side validation:** Introduce a typo in the base (e.g. `replicass`), then compare `kubectl kustomize` (succeeds) vs `kubectl apply -k --dry-run=server` (fails). Explain why build alone is not validation.

Check-yourself questions: Which fields get rewritten when a name changes? Why does `replicas` use the original name? Why is `digest` safer than `newTag`? What does the Namespace transformer not do?

## Further Reading

- Kustomize documentation: https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/
- Kubernetes docs — Declarative management with Kustomize: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
- Kustomize `kustomization.yaml` field reference: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/
- `images` transformer reference: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/images/
- `labels` reference (replacement for `commonLabels`): https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/labels/
- Kustomize releases and embedded-version table (compare with your kubectl): https://github.com/kubernetes-sigs/kustomize/releases
- Next: Lesson 57 — patches, generators, and Kustomize vs Helm.

# Lesson 53: Helm Basics – Charts and Repositories

**Module:** Helm – Kubernetes Package Manager
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-52 (kubectl, Deployments, Services, Ingress, ConfigMap/Secret, RBAC, a working cluster from lesson 52 or kind/minikube). Students have already typed `helm install` for ingress-nginx, cert-manager, External Secrets Operator and Kyverno — today we learn what those commands actually did.

## Learning Objectives

By end of lesson student can:
- Explain what Helm solves: packaging, versioning, parameterizing and lifecycle-managing a group of Kubernetes manifests as one unit.
- Define chart, release, revision, repository, values and `Chart.yaml`, and explain how they relate.
- Describe Helm 3 architecture: client-only binary, no Tiller, release state stored as Secrets in the cluster.
- Install the Helm CLI and verify it talks to the right cluster (same kubeconfig/context as `kubectl`).
- Read the anatomy of a chart (`Chart.yaml`, `values.yaml`, `templates/`, `charts/`, `.helmignore`, `values.schema.json`) without needing to author one yet.
- Add, update, list, search and remove chart repositories; search Artifact Hub; pull charts from an OCI registry.
- Inspect a chart *before* installing it with `helm show chart|values|readme|all`.
- Install a release with `--set`, `--set-string`, `--set-file`, `-f` (multiple files, precedence), `--namespace`, `--create-namespace`, `--version`, `--wait`, `--timeout`, `--atomic`, `--generate-name`, `--dry-run`.
- Manage releases: `list`, `status`, `get values|manifest|notes|hooks|all`, `history`, `upgrade` (`--install`, `--reuse-values`, `--reset-values`, `--reset-then-reuse-values`), `rollback`, `uninstall` (`--keep-history`).
- Inspect where Helm stores release data (`sh.helm.release.v1.*` Secrets) and decode it.
- Recognize and fix common failures: `pending-install` stuck release, "cannot re-use a name", immutable field errors, CRD handling, `--reuse-values` surprises.

## Topics

- Helm concepts: package manager for K8s, charts, releases, revisions, Chart.yaml
- Chart structure: Chart.yaml, values.yaml, templates/, charts/ (dependencies)
- Repositories: helm repo add/update/list/remove; search repo and Artifact Hub
- Installing releases: helm install (--set, -f values.yaml, --namespace, --create-namespace)
- Managing releases: helm list, status, get values/manifest, upgrade, rollback, uninstall

## Concepts

### The problem Helm solves

Through lessons 34-52 we deployed apps with plain YAML: a Deployment, a Service, an Ingress, a ConfigMap, a Secret, maybe a PVC. One real application is often 8-20 such objects. That creates recurring pain:

| Pain with plain `kubectl apply -f` | What Helm adds |
|---|---|
| Same app needed in dev, staging, prod with different replicas/image/hostname → copy-paste YAML directories | One chart, many **values** files |
| No concept of "the app" — just a pile of objects; deleting means remembering every object | A **release** groups all objects; `helm uninstall` removes them all |
| Upgrade failed halfway → manual cleanup, no history | **Revisions** with `helm rollback` |
| Installing third-party software (ingress-nginx, Prometheus, Postgres) means copying 2000 lines of upstream YAML | `helm install` from a public **repository** |
| No version for the whole bundle | **Chart version** and **app version** |
| Hard to know which values an app accepts | `values.yaml` is the documented configuration surface |

Helm is called "the package manager for Kubernetes". Analogy students already know from Linux (lesson 4 / apt): `apt install nginx` resolves a package from a repository, installs files, records it in a database, and lets you upgrade/remove it. Helm does the same for Kubernetes objects.

| Linux package manager | Helm |
|---|---|
| `.deb` package | **Chart** (a directory or `.tgz`) |
| apt repository (`sources.list`) | **Chart repository** (`helm repo add`) |
| Installed package (record in dpkg database) | **Release** (record stored in the cluster) |
| Package version | Chart `version` + `appVersion` |
| `/etc/nginx/nginx.conf` config | `values.yaml` |
| `apt install`, `apt upgrade`, `apt remove` | `helm install`, `helm upgrade`, `helm uninstall` |

### Vocabulary (memorize these)

- **Chart** — a package: a directory (or `.tgz`) containing templated Kubernetes manifests plus metadata and default configuration. Think "a recipe with blanks to fill".
- **Values** — the configuration that fills the blanks. Defaults live in the chart's `values.yaml`; you override them per install.
- **Release** — one *installed instance* of a chart in a cluster, with a name and a namespace. You can install the same chart many times: `helm install web1 ...` and `helm install web2 ...` are two independent releases.
- **Revision** — a numbered version of a release. First install = revision 1. Every `upgrade` or `rollback` creates a new revision (rollback does **not** rewind the counter; it creates a new revision whose content equals an old one).
- **Repository** — an HTTP server hosting an `index.yaml` plus chart `.tgz` files. Or an OCI registry.
- **Chart version vs appVersion** — `version` is the version of the *chart packaging* (SemVer, bump when templates change). `appVersion` is the version of the *software inside* (informational, e.g. nginx `1.27.0`). They move independently.
- **Template** — a YAML file in `templates/` containing placeholders. How placeholders work is lesson 54; today we treat them as black boxes and only look at the *rendered* result.

### Helm 3 architecture: client-only

Helm 2 had a server component in the cluster called **Tiller** with broad cluster-admin permissions — a security problem. **Helm 3 removed Tiller.** Today:

```
 your laptop / CI runner                                 Kubernetes cluster
┌────────────────────────┐   kubeconfig credentials   ┌─────────────────────────┐
│ helm CLI (single bin)  │ ─────────────────────────▶ │ kube-apiserver          │
│  - reads chart         │                            │  Deployments, Services… │
│  - renders templates   │                            │  Secret sh.helm.release │
│  - talks to API server │                            │   (release history)     │
└────────────────────────┘                            └─────────────────────────┘
```

Consequences you should be able to explain:
1. **Helm uses your kubeconfig** and therefore your RBAC permissions (lesson 47). If `kubectl` cannot create a Deployment in a namespace, `helm install` cannot either. Same context, same namespace default, same `KUBECONFIG` env var.
2. **Release state is stored in the cluster** as one **Secret per revision** (type `helm.sh/release.v1`) named `sh.helm.release.v1.<release>.v<revision>`, in the **release's namespace**. It contains the chart, values and rendered manifest (gzip + base64). That is how `helm history`, `helm rollback` and `helm get` work from any machine.
3. **No daemon** — nothing runs between commands. Helm computes the diff at command time (three-way merge between the old manifest, the new manifest and the live state).
4. Releases are **namespace-scoped**: the same release name can exist in two namespaces. `helm list` shows only the current namespace; use `-A` for all.

Storage driver: default `secret`. Others (`configmap`, `sql`, `memory`) exist via `HELM_DRIVER` env var — rarely changed.

### Installing the Helm CLI

Helm is one static binary. Pick one:

```bash
# Official install script (needs curl; inspect before piping in prod)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh && ./get_helm.sh

# macOS
brew install helm

# Debian/Ubuntu (official apt repo)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install helm

# Snap
sudo snap install helm --classic

# Verify
helm version            # client version only — there is no server version in Helm 3
helm env                # shows paths, driver, kubeconfig, namespace
```

Paths from `helm env` worth knowing: `HELM_CACHE_HOME` (downloaded chart tarballs, repo index cache), `HELM_CONFIG_HOME` (`repositories.yaml` lives here), `HELM_DATA_HOME` (plugins). Typical Linux: `~/.cache/helm`, `~/.config/helm`, `~/.local/share/helm`.

Shell completion (saves lots of typing live):

```bash
source <(helm completion bash)        # or zsh / fish
echo 'source <(helm completion bash)' >> ~/.bashrc
```

### Chart anatomy

A chart is a **directory** with a fixed layout. The directory name equals the chart name. We will *look at* this today (using `helm pull --untar` on a real chart) and *build one* in lesson 55 after learning templating in lesson 54.

```
podinfo/
├── Chart.yaml            # metadata: name, version, appVersion, dependencies
├── values.yaml           # default configuration values
├── values.schema.json    # (optional) JSON Schema validating values
├── .helmignore           # (optional) files excluded when packaging
├── README.md             # (optional) docs shown by `helm show readme`
├── LICENSE               # (optional)
├── crds/                 # (optional) CustomResourceDefinitions installed first, never upgraded/deleted by Helm
├── templates/            # templated manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl      # named template snippets (leading underscore = not rendered as a manifest)
│   └── NOTES.txt         # message printed after install (also `helm get notes`)
└── charts/               # sub-charts (dependencies), populated by `helm dependency update`
```

#### Chart.yaml

```yaml
apiVersion: v2              # v2 = Helm 3 charts (v1 = Helm 2, avoid)
name: podinfo               # chart name (lowercase, dashes)
description: Tiny Go web app used for Kubernetes demos
type: application           # application (installable) | library (reusable templates only, not installable)
version: 6.7.1              # CHART version, SemVer. Bump on any chart change.
appVersion: "6.7.1"         # version of the packaged software (string, quote it)
kubeVersion: ">=1.23.0-0"   # optional constraint on supported Kubernetes versions
keywords: [podinfo, demo]
home: https://github.com/stefanprodan/podinfo
sources:
  - https://github.com/stefanprodan/podinfo
maintainers:
  - name: stefanprodan
dependencies:               # optional: sub-charts
  - name: redis
    version: "19.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

Field meanings students ask about:
- **`apiVersion: v2`** — chart format version, not Kubernetes API version.
- **`type: library`** — chart that only provides shared helper templates for other charts; `helm install` refuses it.
- **`version`** must be valid SemVer (`1.2.3`, `1.2.3-rc.1`). Repositories index charts by it.
- **`appVersion`** — purely informational by default; many charts use it as the default image tag.
- **`dependencies`** — other charts this chart needs; downloaded into `charts/` by `helm dependency update` (see below).

#### values.yaml

The *documented* defaults. A good chart comments every key. Example fragment from podinfo:

```yaml
replicaCount: 1
image:
  repository: ghcr.io/stefanprodan/podinfo
  tag: 6.7.1
  pullPolicy: IfNotPresent
ui:
  message: ""
  color: "#34577c"
service:
  enabled: true
  type: ClusterIP
  httpPort: 9898
ingress:
  enabled: false
resources:
  limits: {}
  requests:
    cpu: 1m
    memory: 16Mi
```

Overriding rules (important, memorize): values are **merged as nested maps**; you only specify what changes. **Lists are replaced wholesale**, not merged — a classic trap.

#### templates/

YAML files that look like normal Kubernetes manifests but contain placeholders in double braces. Example of what you will *see* when you open one (do not worry about the syntax — lesson 54 explains every symbol):

```yaml
# templates/service.yaml (excerpt, for orientation only)
apiVersion: v1
kind: Service
metadata:
  name: <placeholder for release name>
spec:
  type: <placeholder from values: service.type>
```

What matters today: Helm *renders* every file in `templates/` into plain YAML using the merged values, then sends the result to the API server. `NOTES.txt` is rendered and printed but is not sent as a manifest.

#### charts/ and .helmignore

- `charts/` holds dependency charts (as `.tgz` or directories). With `helm dependency update`, Helm reads `Chart.yaml` `dependencies`, downloads them here, and writes `Chart.lock` (pinned versions).
- `.helmignore` works like `.gitignore` for `helm package` (lesson 55) — keeps `.git`, editor files, test data out of the archive.

#### values.schema.json (brief)

Optional JSON Schema file. Helm validates the merged values against it on `install`, `upgrade`, `lint` and `template`. It turns typos like `replicaCount: "three"` into immediate errors instead of broken manifests. You will rarely write one today; you will see "values don't meet the specifications of the schema" errors when you pass bad values to charts that ship one.

### Repositories

A **chart repository** is a plain HTTP(S) server serving:
- `index.yaml` — catalog of every chart and version (name, version, appVersion, description, URLs, digest)
- the chart archives `name-version.tgz`

Any static file host (GitHub Pages, S3, nginx) can be one. Helm keeps a local copy of each repo's index in `HELM_CACHE_HOME`; `helm repo update` refreshes it — analogous to `apt update`.

Popular sources:
- **Artifact Hub** (https://artifacthub.io) — the web search engine/directory for charts and operators; not a repo itself, it points at repos.
- Vendor repos: `https://kubernetes.github.io/ingress-nginx`, `https://charts.jetstack.io` (cert-manager), `https://charts.external-secrets.io`, `https://kyverno.github.io/kyverno/`, `https://prometheus-community.github.io/helm-charts`, `https://stefanprodan.github.io/podinfo`.
- **OCI registries** (Docker Hub, GHCR, ECR, Harbor from lesson 33) — modern alternative; no `repo add` needed.

**Trust note:** a chart can create ClusterRoles, DaemonSets with host access, etc. Reading `helm show values` and `helm get manifest` *before* installing third-party charts is the equivalent of reading a shell script before `curl | bash`.

### Values precedence

When several sources set the same key, later/higher wins. Lowest to highest priority:

1. Chart's own `values.yaml` (defaults)
2. Parent chart's `values.yaml` (if this is a sub-chart)
3. Files passed with `-f` / `--values`, **in command-line order** (later file overrides earlier)
4. `--set`, `--set-string`, `--set-file` flags (highest; among themselves, later flag wins)

Memory trick: **file < flag**, and **right-most wins**.

```bash
helm install web podinfo/podinfo -f base.yaml -f prod.yaml --set replicaCount=5
#                                  ^ lowest    ^ overrides base  ^ overrides both files
```

### Release lifecycle

```
          install                upgrade               upgrade
(nothing) ───────▶ rev 1 deployed ──────▶ rev 2 deployed ──────▶ rev 3 deployed
                                              ▲                        │
                                              │    rollback to 2       ▼
                                              └─────────── rev 4 deployed (content of rev 2)
```

Release statuses you will see in `helm list`/`helm history`:

| Status | Meaning |
|---|---|
| `deployed` | Current good revision |
| `superseded` | Older revision replaced by a newer one |
| `failed` | Install/upgrade did not complete |
| `pending-install` / `pending-upgrade` / `pending-rollback` | Operation in progress — or was interrupted (stuck, see pitfalls) |
| `uninstalling` | Deletion running |
| `uninstalled` | Deleted but history kept (`--keep-history`) |

## Commands / Syntax Reference

### Repository commands

| Command | Purpose | Example |
|---|---|---|
| `helm repo add NAME URL` | Register a repository | `helm repo add podinfo https://stefanprodan.github.io/podinfo` |
| `helm repo list` | Show registered repos | `helm repo list` |
| `helm repo update` | Refresh all repo indexes (like `apt update`) | `helm repo update` / `helm repo update podinfo` |
| `helm repo remove NAME` | Unregister repo | `helm repo remove podinfo` |
| `helm repo index DIR` | Generate `index.yaml` for a directory of charts (hosting your own repo) | mention only |
| `helm search repo KEYWORD` | Search added repos (local index) | `helm search repo nginx` |
| `helm search repo NAME --versions` | List all versions of a chart | `helm search repo podinfo/podinfo --versions \| head` |
| `helm search repo NAME --devel` | Include pre-release versions | `helm search repo podinfo --devel` |
| `helm search hub KEYWORD` | Search Artifact Hub (needs internet) | `helm search hub prometheus --max-col-width 80` |

### Inspecting charts

| Command | Purpose | Example |
|---|---|---|
| `helm show chart CHART` | Print `Chart.yaml` | `helm show chart podinfo/podinfo` |
| `helm show values CHART` | Print default `values.yaml` | `helm show values podinfo/podinfo > podinfo-values.yaml` |
| `helm show readme CHART` | Print README | `helm show readme podinfo/podinfo` |
| `helm show crds CHART` | Print CRDs shipped in `crds/` | `helm show crds jetstack/cert-manager` |
| `helm show all CHART` | Everything above | `helm show all podinfo/podinfo` |
| `helm pull CHART` | Download chart `.tgz` | `helm pull podinfo/podinfo --version 6.7.1` |
| `helm pull CHART --untar` | Download and unpack to a directory | `helm pull podinfo/podinfo --untar --untardir ./charts-src` |
| `helm pull oci://REG/PATH` | Pull from OCI registry | `helm pull oci://ghcr.io/stefanprodan/charts/podinfo --version 6.7.1` |

### Install

| Command / flag | Purpose | Example |
|---|---|---|
| `helm install RELEASE CHART` | Install chart as a named release | `helm install web podinfo/podinfo` |
| `--namespace NS` / `-n` | Target namespace | `-n demo` |
| `--create-namespace` | Create namespace if missing | `--create-namespace` |
| `--set k=v` | Override a value (nested with dots, lists with `{}`) | `--set replicaCount=3,ui.color=red` |
| `--set-string k=v` | Force a string (avoid `true`→bool, `007`→int) | `--set-string podAnnotations.build=0042` |
| `--set-file k=path` | Value from file contents | `--set-file ui.message=./msg.txt` |
| `--set-json k=JSON` | Value from JSON | `--set-json 'resources={"limits":{"cpu":"100m"}}'` |
| `-f FILE` / `--values FILE` | Values file (repeatable) | `-f prod.yaml` |
| `--version X` | Pin chart version (supports ranges) | `--version 6.7.1` |
| `--wait` | Wait until Pods/Deployments are Ready | `--wait` |
| `--timeout DUR` | Wait limit (default `5m0s`) | `--timeout 10m` |
| `--atomic` | On failure, roll back automatically (implies `--wait`) | `--atomic` |
| `--generate-name` | Auto name `chartname-<timestamp>` | `helm install podinfo/podinfo --generate-name` |
| `--dry-run` | Contact the API server for validation, print rendered result, install nothing | `--dry-run` |
| `--description TEXT` | Message stored on the revision | `--description "initial prod rollout"` |
| `--skip-crds` | Do not install `crds/` | `--skip-crds` |

### Release management

| Command | Purpose | Example |
|---|---|---|
| `helm list` | Releases in current namespace | `helm list` |
| `helm list -A` | All namespaces | `helm list -A` |
| `helm list --failed` / `--pending` / `--uninstalled` / `--all` | Filter by status | `helm list --pending -A` |
| `helm list -o json` / `yaml` | Machine-readable | `helm list -o json \| jq .` |
| `helm status RELEASE` | Status, resources, NOTES | `helm status web -n demo` |
| `helm get values RELEASE` | Only **user-supplied** values | `helm get values web` |
| `helm get values RELEASE --all` | Computed (defaults merged with yours) | `helm get values web --all` |
| `helm get manifest RELEASE` | Rendered YAML that was applied | `helm get manifest web` |
| `helm get notes RELEASE` | Re-print NOTES.txt | `helm get notes web` |
| `helm get hooks RELEASE` | Hook manifests (one-line mention) | `helm get hooks web` |
| `helm get all RELEASE` | Everything | `helm get all web` |
| `helm history RELEASE` | Revision table | `helm history web` |
| `helm get values RELEASE --revision N` | Values of an old revision | `helm get values web --revision 2` |
| `helm upgrade RELEASE CHART` | New revision from chart/values | `helm upgrade web podinfo/podinfo -f prod.yaml` |
| `helm upgrade --install RELEASE CHART` | Install if absent, else upgrade (idempotent) | `helm upgrade --install web podinfo/podinfo` |
| `--reuse-values` | Start from last release's values, apply new flags on top | `helm upgrade web podinfo/podinfo --reuse-values --set replicaCount=4` |
| `--reset-values` | Ignore previous values, use chart defaults + flags given now | `--reset-values` |
| `--reset-then-reuse-values` | Reset to the *new chart's* defaults, then re-apply your previous values, then new flags | `--reset-then-reuse-values` |
| `helm rollback RELEASE [REV]` | Go back (creates a new revision) | `helm rollback web 1` |
| `helm rollback --wait --cleanup-on-fail` | Safer rollback | `helm rollback web 1 --wait` |
| `helm uninstall RELEASE` | Delete release and its objects | `helm uninstall web -n demo` |
| `helm uninstall --keep-history` | Delete objects but keep revision records (status `uninstalled`) | `helm uninstall web --keep-history` |

### Dependencies (brief)

| Command | Purpose | Example |
|---|---|---|
| `helm dependency list CHARTDIR` | Show declared dependencies and status | `helm dependency list ./mychart` |
| `helm dependency update CHARTDIR` | Download deps into `charts/`, write `Chart.lock` | `helm dependency update ./mychart` |
| `helm dependency build CHARTDIR` | Rebuild `charts/` strictly from `Chart.lock` (reproducible) | `helm dependency build ./mychart` |

### Environment and OCI

| Command | Purpose | Example |
|---|---|---|
| `helm env` | Print Helm environment | `helm env` |
| `helm version --short` | Client version | `helm version --short` |
| `helm registry login HOST` | Authenticate to OCI registry | `helm registry login ghcr.io -u USER` |
| `helm install RELEASE oci://REG/PATH` | Install straight from OCI | `helm install web oci://ghcr.io/stefanprodan/charts/podinfo --version 6.7.1` |
| `helm plugin list` | Show plugins (e.g. `helm-diff`) | `helm plugin list` |

### `--set` syntax cheat sheet

| Goal | Syntax |
|---|---|
| Scalar | `--set replicaCount=3` |
| Nested key | `--set image.tag=6.7.0` |
| Multiple | `--set a=1,b=2` (or repeat `--set`) |
| List element by index | `--set ingress.hosts[0].host=example.com` |
| Simple list | `--set 'tolerations={a,b}'` (quote braces for zsh/bash) |
| Literal dot in key | `--set 'podAnnotations.example\.com/owner=team-a'` |
| Literal comma in value | `--set 'ui.message=hello\, world'` |
| Null (delete a default key) | `--set securityContext=null` |
| Force string | `--set-string version=1.10` |
| Bool | `--set ingress.enabled=true` |

Rule of thumb: **more than 2-3 `--set` flags → use a values file.** It can live in Git (lesson 14); a long command line cannot be reviewed.

## Examples / Walkthrough

Lab setup: any cluster from lesson 52, kind or minikube; `kubectl get nodes` shows Ready. We use the **podinfo** chart (a small Go web app maintained for demos): fast to pull, no persistence, no dependencies.

### 1. Verify Helm and the target cluster

```bash
helm version --short                  # e.g. v3.16.x
kubectl config current-context        # Helm will use THIS context
kubectl get nodes
helm list -A                          # releases already present (ingress-nginx, cert-manager, ... from earlier lessons)
```

Point out: Helm has no connection config of its own. Switch context with `kubectl config use-context` or `helm --kube-context NAME ...`.

### 2. Add a repository and explore

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update
helm repo list
helm search repo podinfo
helm search repo podinfo/podinfo --versions | head -5
helm search hub podinfo --max-col-width 60       # Artifact Hub (internet)
```

Expected shape:

```
NAME             	CHART VERSION	APP VERSION	DESCRIPTION
podinfo/podinfo  	6.7.1        	6.7.1      	Podinfo Helm chart for Kubernetes
podinfo/podinfo-... 
```

Discuss: `search repo` searches the *local cached index* — that is why `helm repo update` matters before looking for a new version.

### 3. Inspect before you install

```bash
helm show chart podinfo/podinfo
helm show readme podinfo/podinfo | less
helm show values podinfo/podinfo > default-values.yaml
wc -l default-values.yaml
grep -nE '^(replicaCount|image|service|ingress|resources):' default-values.yaml
```

Ask students: which keys would you change for production? (replicaCount, resources, ingress.enabled, service.type…) Everything configurable is visible here — no need to read templates.

Optional: look inside the chart as a directory.

```bash
mkdir -p ~/helm-lab && cd ~/helm-lab
helm pull podinfo/podinfo --untar
ls -R podinfo | head -30
cat podinfo/Chart.yaml
ls podinfo/templates
cat podinfo/templates/NOTES.txt | head -20        # contains placeholders — decoded in lesson 54
```

### 4. First install, with defaults

```bash
helm install web podinfo/podinfo \
  --namespace demo --create-namespace \
  --wait --timeout 3m
```

Read the output: `NAME`, `LAST DEPLOYED`, `NAMESPACE`, `STATUS: deployed`, `REVISION: 1`, then `NOTES:` (rendered NOTES.txt — instructions for reaching the app).

```bash
helm list -n demo
helm status web -n demo
kubectl get all -n demo
kubectl get deploy,svc -n demo --show-labels      # note the labels Helm-managed objects carry
kubectl get deploy web-podinfo -n demo -o jsonpath='{.metadata.annotations}' ; echo
```

Look at the two metadata annotations Helm adds: `meta.helm.sh/release-name` and `meta.helm.sh/release-namespace`, plus label `app.kubernetes.io/managed-by: Helm`. Helm refuses to take over objects that do not carry them — source of the "exists and cannot be imported into the current release" error (pitfalls).

Test the app:

```bash
kubectl -n demo port-forward svc/web-podinfo 9898:9898 &
curl -s localhost:9898 | head
kill %1
```

### 5. Where Helm stores the release

```bash
kubectl get secrets -n demo
# NAME                           TYPE                 DATA
# sh.helm.release.v1.web.v1      helm.sh/release.v1   1

kubectl get secret sh.helm.release.v1.web.v1 -n demo -o jsonpath='{.metadata.labels}' ; echo
# {"modifiedAt":"...","name":"web","owner":"helm","status":"deployed","version":"1"}

# Decode the payload: base64 (k8s) → base64 (helm) → gzip → JSON
kubectl get secret sh.helm.release.v1.web.v1 -n demo -o jsonpath='{.data.release}' \
  | base64 -d | base64 -d | gunzip | jq 'keys'
# ["chart","config","info","manifest","name","namespace","version", ...]
```

Points: this Secret is how Helm knows "release `web` exists in `demo`". Delete it by hand and Helm forgets the release while the Deployment keeps running (orphans). Also: anyone who can read Secrets in the namespace can read the release's **values** — treat `--set password=...` accordingly (lesson 42/43: use Secrets/ESO, not chart values, for real credentials).

### 6. Customizing the install: `--set`

```bash
helm install web2 podinfo/podinfo -n demo \
  --set replicaCount=2 \
  --set ui.message="Hello from web2" \
  --set ui.color="#2d6a4f" \
  --wait

helm get values web2 -n demo           # only what WE supplied
helm get values web2 -n demo --all     # defaults merged with ours
kubectl get deploy web2-podinfo -n demo       # READY 2/2
```

Two releases of the same chart, side by side: `kubectl get deploy -n demo` shows `web-podinfo` and `web2-podinfo`. The **release name prefixes object names** — that is how a chart avoids collisions.

### 7. Customizing with a values file (preferred)

```bash
cat > prod-values.yaml <<'EOF'
replicaCount: 3
ui:
  message: "Production"
  color: "#9d0208"
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
EOF

helm upgrade web podinfo/podinfo -n demo -f prod-values.yaml --wait
helm history web -n demo
# REVISION  STATUS      CHART           APP VERSION  DESCRIPTION
# 1         superseded  podinfo-6.7.1   6.7.1        Install complete
# 2         deployed    podinfo-6.7.1   6.7.1        Upgrade complete
```

Precedence demo — a base file, an override file and a flag:

```bash
cat > base.yaml <<'EOF'
replicaCount: 1
ui:
  message: "from base"
EOF
cat > override.yaml <<'EOF'
ui:
  message: "from override"
EOF

helm install prec podinfo/podinfo -n demo \
  -f base.yaml -f override.yaml --set replicaCount=4 --dry-run \
  | grep -E 'replicas|PODINFO_UI_MESSAGE' -A1
# replicas: 4          <- --set beat base.yaml
# value: "from override"  <- later -f beat earlier -f
```

(`--dry-run` prints rendered YAML; we only read it, not decode how it was produced.) Swap the order of the `-f` files and re-run to prove right-most wins.

### 8. `--dry-run` as a safety check

```bash
helm install risky podinfo/podinfo -n demo --set replicaCount=10 --dry-run --debug | head -60
helm list -n demo | grep risky     # nothing: no release created
```

`--dry-run` = render + validate against the API server (schema, existing objects) but persist nothing. `--debug` adds computed values to the output. Use before any prod upgrade. (Local-only rendering without a cluster is lesson 55.)

### 9. Upgrade pitfalls live: `--reuse-values` vs `--reset-values`

State now: release `web` at revision 2 with `prod-values.yaml` applied.

```bash
# (a) Upgrade with only one flag, NO -f, NO reuse flag
helm upgrade web podinfo/podinfo -n demo --set ui.color="#000000" --wait
helm get values web -n demo
# Only ui.color is listed — replicaCount=3, message, resources from prod-values.yaml are GONE.
kubectl get deploy web-podinfo -n demo   # back to 1 replica!
```

Explain: by default `helm upgrade` takes values **only from the current command** (plus chart defaults). It does not remember the previous `-f`. This is the #1 Helm surprise.

```bash
# (b) Fix: re-apply the file
helm upgrade web podinfo/podinfo -n demo -f prod-values.yaml --set ui.color="#000000" --wait

# (c) Or inherit previous values
helm upgrade web podinfo/podinfo -n demo --reuse-values --set replicaCount=5 --wait
```

`--reuse-values` caveat: it reuses the **previous release's computed values**, so **new keys added to the chart's `values.yaml` in a newer chart version are not picked up** and templates can break. Safer modern option:

```bash
helm upgrade web podinfo/podinfo -n demo --version 6.7.1 --reset-then-reuse-values --set replicaCount=5
```

`--reset-then-reuse-values` resets to the *new chart's* defaults, re-applies the values you previously supplied, then applies new flags. **Best practice for production: keep a values file in Git and always pass `-f values.yaml` explicitly** — then none of these flags are needed.

### 10. Rollback

```bash
helm history web -n demo
helm rollback web 2 -n demo --wait         # content of revision 2
helm history web -n demo
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade complete
# 3         superseded  Upgrade complete
# 4         superseded  Upgrade complete
# 5         deployed    Rollback to 2
helm get values web -n demo --revision 2   # compare what revision 2 contained
```

Rollback = new revision (5) with revision 2's content. History grows; `helm upgrade --history-max 10` (default 10) trims old revisions (and their Secrets).

Limits of rollback: restores **manifests**; does **not** restore data in PVCs (lesson 41), nor reverse CRD changes, nor reverse database schema migrations run by the app.

### 11. Upgrading to a specific chart version, and `--atomic`

```bash
helm upgrade web podinfo/podinfo -n demo --version 6.6.0 -f prod-values.yaml --atomic --timeout 2m
helm list -n demo           # CHART column shows podinfo-6.6.0
```

Force a failure to see `--atomic`:

```bash
helm upgrade web podinfo/podinfo -n demo -f prod-values.yaml \
  --set image.tag=does-not-exist --atomic --timeout 90s
# Error: UPGRADE FAILED: release web failed, and has been rolled back due to atomic being set: context deadline exceeded
helm history web -n demo     # last entries: failed, then "Rollback to N"
kubectl get pods -n demo
```

Compare with `--wait` alone: Helm reports failure but leaves the broken revision deployed (`failed`). `--atomic` auto-rolls back — recommended for CI/CD (lessons 59+).

### 12. `upgrade --install` (idempotent deploy)

```bash
helm upgrade --install web3 podinfo/podinfo -n demo --create-namespace -f prod-values.yaml --wait
helm upgrade --install web3 podinfo/podinfo -n demo -f prod-values.yaml --wait   # second run = upgrade, no error
```

This is the command to put in scripts and pipelines: works whether the release exists or not.

### 13. Inspecting what is deployed

```bash
helm get manifest web -n demo | head -40       # exact YAML Helm applied
helm get manifest web -n demo | grep '^kind:'
helm get notes web -n demo
helm get all web -n demo | head -80
diff <(helm get manifest web -n demo --revision 2) <(helm get manifest web -n demo --revision 4) && echo same
```

Tie to lesson 36 (`kubectl diff`) — drift: if someone `kubectl edit`s a Helm-managed Deployment, Helm's stored manifest no longer matches live state; next `helm upgrade` three-way-merges and may revert the manual change. Never hand-edit Helm-managed objects.

### 14. Uninstall

```bash
helm uninstall web2 -n demo
helm uninstall prec -n demo 2>/dev/null; true
kubectl get deploy -n demo                      # web2-podinfo gone

helm uninstall web3 -n demo --keep-history
helm list -n demo --uninstalled                 # web3  ... uninstalled
helm rollback web3 1 -n demo                    # brings it back (only possible with kept history)

helm uninstall web -n demo
kubectl get secrets -n demo | grep helm         # release Secrets removed too
```

What `uninstall` does **not** delete: PVCs created by StatefulSet `volumeClaimTemplates` (lesson 41), CRDs from `crds/`, namespaces created via `--create-namespace`. Clean those manually.

### 15. Chart dependencies (look, don't build)

```bash
cd ~/helm-lab
helm pull oci://registry-1.docker.io/bitnamicharts/nginx --untar     # OCI pull, no `repo add`
cat nginx/Chart.yaml | sed -n '1,40p'                                  # note the dependencies: block (common library chart)
ls nginx/charts                                                        # bundled sub-chart(s)
helm dependency list ./nginx
```

Explain: a chart can declare other charts it needs (a web app chart depending on a Postgres chart). `dependency update` fetches them; sub-chart values are configured under a key named after the sub-chart in the parent's values (`redis.auth.enabled`), and `condition: redis.enabled` toggles the whole dependency. Authoring that is lesson 55.

### 16. Installing from OCI

```bash
helm install nginx-oci oci://registry-1.docker.io/bitnamicharts/nginx \
  --version 18.2.0 -n demo --set service.type=ClusterIP --wait
helm list -n demo
helm uninstall nginx-oci -n demo
```

OCI notes: `helm registry login <host>` for private registries (credentials like lesson 33 `docker login`), versions are OCI tags, `helm search repo` does **not** work for OCI (no index) — browse the registry or Artifact Hub instead. Pin `--version`; some tags/registries/limits change over time — verify the exact chart and version before class.

### 17. Re-visit earlier lessons: what those installs really were

```bash
helm list -A
helm get values ingress-nginx -n ingress-nginx          # what we set in lesson 38
helm get values cert-manager -n cert-manager            # e.g. crds.enabled=true / installCRDs=true
helm history ingress-nginx -n ingress-nginx
helm show values ingress-nginx/ingress-nginx | grep -n 'ingressClassResource' -A6
```

(Names/namespaces depend on how each student installed them; adapt live.) Takeaway: every operator in this course is just a chart plus a values file — the same workflow we practised today.

## Common Pitfalls

- **Release stuck in `pending-install` / `pending-upgrade`** — Helm was interrupted (Ctrl-C, CI timeout, lost connection) mid-operation and left the last revision in a pending state; the next command fails with `another operation (install/upgrade/rollback) is in progress`. Check `helm history REL -n NS`. Fix: `helm rollback REL <last-good-rev>` (works for pending-upgrade), or for a stuck first install `helm uninstall REL -n NS` then reinstall; in the worst case delete the pending revision Secret `sh.helm.release.v1.REL.vN` (understand what you remove). Prevent with `--atomic` and realistic `--timeout`.
- **`Error: INSTALLATION FAILED: cannot re-use a name that is still in use`** — a release with that name exists (maybe in `failed` state or `uninstalled` with kept history) in this namespace. Use `helm list -A --all`, pick a new name, or `helm upgrade --install`.
- **Wrong namespace** — `helm list` shows nothing because releases are namespace-scoped. Add `-n NS` or `-A`; set the default with `kubectl config set-context --current --namespace NS` (Helm follows kubeconfig).
- **Upgrade silently drops earlier customizations** — `helm upgrade` does not remember previous `--set`/`-f`. Always pass the full values file again (or `--reset-then-reuse-values`). Keep values in Git.
- **`--reuse-values` with a newer chart** — new default keys do not appear; templates referencing them break or render empty. Prefer `-f values.yaml` or `--reset-then-reuse-values`.
- **`--set` list/quote mistakes** — shell eats `{}` and `,`; zsh needs quotes: `--set 'a={x,y}'`. Literal dots in keys need `\.`. Numbers/booleans parsed as such: use `--set-string` for `"true"` or `"007"`.
- **Lists replace, maps merge** — overriding `tolerations` or `ingress.hosts` replaces the *entire* default list. Copy the full default list from `helm show values` and edit it.
- **`Error: UPGRADE FAILED: cannot patch ... field is immutable`** — you changed something Kubernetes forbids changing in place (Deployment `spec.selector`, Service `clusterIP`, StatefulSet `volumeClaimTemplates`, Job `spec.template`). Fix: `helm uninstall` and reinstall (accepting downtime), or delete just that object (`kubectl delete deploy X --cascade=orphan` carefully) and re-run upgrade; avoid renaming selector labels in chart updates.
- **`Error: rendered manifests contain a resource that already exists. Unable to continue with install: ... exists and cannot be imported into the current release: invalid ownership metadata`** — the object was created earlier by `kubectl apply`. Delete it, or adopt it by adding Helm's labels/annotations (`app.kubernetes.io/managed-by=Helm`, `meta.helm.sh/release-name`, `meta.helm.sh/release-namespace`).
- **CRDs** — CRDs in `crds/` are installed on `install` only; Helm **never upgrades or deletes** them (data-loss safety). Many charts (cert-manager, kyverno, ESO) instead expose a value like `installCRDs`/`crds.enabled`. After a chart upgrade that changes CRDs, apply new CRDs manually (`kubectl apply --server-side -f ...`). Uninstalling a chart leaves CRDs (and all custom resources) behind. A resource of a CRD kind in the same release can fail with "no matches for kind" on first install if the CRD is not yet established.
- **Secrets in values** — `helm get values` and the release Secret expose everything passed with `--set`/`-f`. Do not commit real passwords in values files; use Kubernetes Secrets / ESO (lessons 42-43) or encrypted tooling.
- **Stale repo cache** — `helm search repo` shows old versions: run `helm repo update`. After `repo add` of a URL you typo'd: `Error: looks like "..." is not a valid chart repository` — verify `<URL>/index.yaml` loads in a browser/curl.
- **Unpinned chart versions** — omitting `--version` installs the latest, which differs between dev today and prod next month. Always pin in automation.
- **Editing Helm-managed objects with `kubectl edit`** — causes drift; next upgrade reverts or conflicts. Change values and upgrade instead.
- **`helm uninstall` leftovers** — PVCs, CRDs, namespaces, and any objects created by hooks remain. Clean up explicitly, especially in labs to free disk.
- **Context confusion** — `helm` and `kubectl` share kubeconfig; installing to the prod cluster because the current context was prod is a classic outage. Check `kubectl config current-context` or pass `--kube-context` explicitly in scripts.
- **Assuming rollback undoes data** — rollback only reverts manifests, not PVC data or DB migrations.

## FAQ

**Q: What is the difference between a chart, a release and a revision?**
A: A chart is the package (templates + default values). A release is one installed instance of a chart in a cluster with a name and namespace. A revision is a numbered snapshot of a release; every install, upgrade or rollback adds one.

**Q: Is Helm just a templating tool?**
A: Templating is one part (lesson 54). Helm also versions releases, stores history, rolls back, manages dependencies and distributes packages through repositories. Kustomize (lessons 56-58) is the template-free, patch-based alternative that does not track releases.

**Q: Do I need a Helm server or Tiller in the cluster?**
A: No. Helm 3 is client-only. It uses your kubeconfig and stores release data as Secrets in the cluster. Tiller existed only in Helm 2.

**Q: Where does Helm store information about installed releases?**
A: In Secrets named `sh.helm.release.v1.<release>.v<revision>` of type `helm.sh/release.v1`, in the release's namespace. Each holds the chart, values and rendered manifest, base64+gzip encoded.

**Q: Why does `helm list` show nothing even though I installed something?**
A: Releases are namespace-scoped and `helm list` shows the current namespace only. Use `helm list -A` or `-n <namespace>`. Also check `helm list --all` for failed/uninstalled entries.

**Q: What is the difference between `helm repo add` and Artifact Hub?**
A: `helm repo add` registers an actual chart repository (URL with `index.yaml`) on your machine. Artifact Hub is a website/search index that tells you which repository hosts which chart; `helm search hub` queries it. You still need `helm repo add` (or an `oci://` URL) to install.

**Q: Do I have to run `helm repo update`?**
A: Yes, before searching for or installing a newer chart version: it refreshes the cached `index.yaml`. It does not upgrade any release.

**Q: How do I see all configurable options of a chart?**
A: `helm show values <repo>/<chart>` (add `--version` for a specific version) and `helm show readme`. The chart's page on Artifact Hub documents them too.

**Q: `--set` or `-f values.yaml`, which should I use?**
A: Use `-f` for anything beyond one or two values; the file can be reviewed and versioned in Git. Use `--set` for quick one-offs or for injecting a value from CI (e.g. image tag). `--set` always overrides `-f`.

**Q: In what order are multiple `-f` files applied?**
A: Left to right; the right-most file wins for conflicting keys. Maps are deep-merged, lists are replaced entirely. `--set` flags beat all files.

**Q: Why did my upgrade reset replicas/hostnames to defaults?**
A: `helm upgrade` only uses values given on that command plus chart defaults. Re-pass `-f values.yaml`, or use `--reuse-values` / `--reset-then-reuse-values`. Best practice is always passing the full values file.

**Q: Difference between `--reuse-values`, `--reset-values` and `--reset-then-reuse-values`?**
A: `--reuse-values` starts from the last release's *computed* values (new chart defaults are ignored). `--reset-values` throws previous values away and uses new chart defaults plus current flags. `--reset-then-reuse-values` resets to the new chart's defaults, then re-applies the values you supplied previously, then the new flags. In Git-driven workflows none are needed because `-f` carries everything.

**Q: What does `helm install --dry-run` do, and is it safe?**
A: It renders the chart and validates it against the API server, prints the manifest, and creates nothing in the cluster (no release Secret, no objects). It is safe. For a render that never contacts the cluster, see `helm template` in lesson 55.

**Q: What does `--wait` do, and why use `--atomic`?**
A: `--wait` blocks until Deployments/StatefulSets/Pods/PVCs/Services are ready (or `--timeout`, default 5 minutes). `--atomic` also rolls back automatically on failure and implies `--wait`. Without it a failed upgrade stays in the cluster as a `failed` revision.

**Q: Does `helm rollback` restore data or rewind the revision number?**
A: No to both. It re-applies the manifests of the chosen revision as a *new* revision (e.g. rollback to 2 from 4 creates 5). PVC contents, CRDs and external state are untouched.

**Q: How many revisions does Helm keep?**
A: 10 by default per release. Change with `--history-max N` on upgrade (0 = unlimited). Older revision Secrets are deleted automatically.

**Q: Can I install the same chart twice?**
A: Yes, with different release names (or namespaces). Object names are prefixed with the release name so they don't collide. Cluster-scoped objects (ClusterRole, CRDs, IngressClass) can collide, so some charts cannot be installed twice.

**Q: What's the difference between chart `version` and `appVersion`?**
A: `version` is the chart's own SemVer — bumped when templates/values change. `appVersion` describes the application packaged (e.g. nginx 1.27). A chart can bump `version` without changing the app, and vice versa.

**Q: What is `helm upgrade --install` for?**
A: One idempotent command: installs if the release doesn't exist, upgrades if it does. Use it in scripts and CI so the same command works the first time and every time after.

**Q: Why did `helm uninstall` leave my PVCs / CRDs / namespace?**
A: Helm deliberately avoids deleting data and cluster-wide definitions: PVCs from StatefulSet templates, CRDs from `crds/`, and namespaces created via `--create-namespace` all stay. Delete them with `kubectl` when you really want them gone.

**Q: Can Helm manage objects I already created with `kubectl apply`?**
A: Not directly; Helm errors with "invalid ownership metadata". Either delete and let Helm create them, or add the Helm ownership labels/annotations to adopt them.

**Q: Is it safe to install any chart from the internet?**
A: Treat it like running a script as an administrator. Charts can create ClusterRoles, privileged DaemonSets and webhooks. Inspect with `helm show values`, `helm show all`, `helm install --dry-run`, and `helm get manifest`; prefer official/vendor repositories and pin versions. Image scanning (lesson 50) and admission policy (Kyverno, lesson 50; Pod Security, lesson 48) still apply to what a chart creates.

**Q: What are Helm hooks and tests?**
A: Hooks are annotated manifests (Jobs usually) that run at lifecycle points such as pre-install or post-upgrade (database migrations). `helm test` runs test Pods defined in the chart. Out of scope for this lesson; `helm get hooks RELEASE` shows what a release contains.

**Q: Can I use Helm with GitOps or CI?**
A: Yes. Typical: values files in Git, `helm upgrade --install --atomic -f values-<env>.yaml` in a pipeline (CI/CD module, lessons 59-66), or a GitOps controller that renders charts from Git. The principle is the same: chart + values + pinned version.

**Q: What is `helm diff`?**
A: A community plugin (`helm plugin install https://github.com/databus23/helm-diff`) giving `helm diff upgrade REL CHART -f values.yaml` — a preview of what an upgrade would change. Not part of core Helm; very useful in review workflows.

**Q: Why does `helm search repo` not find a chart on an OCI registry?**
A: OCI registries have no `index.yaml` catalog; Helm cannot search them. Find the chart path on the registry's UI or Artifact Hub and use `helm pull`/`helm install oci://...` with `--version`.

**Q: Helm vs `kubectl apply -f`, when is each better?**
A: `kubectl apply` is fine for a handful of static manifests. Helm wins when you need parameters per environment, versioned releases with rollback, third-party software installs, or dependencies between packages.

## Practice / Exercise

### Core

1. **Setup & orientation.** Install Helm (if needed), run `helm version --short` and `helm env`. Identify the paths for config, cache and data. Confirm `kubectl config current-context` points to your lab cluster. List existing releases with `helm list -A` and write down which operators from earlier lessons are Helm releases.
2. **Repository workflow.** Add the `podinfo` repo, update, list, search. Find all available versions of `podinfo/podinfo` and pick the one before the latest. Remove the repo and re-add it. Run `helm search hub grafana` and note the first three results' repository URLs.
3. **Inspect before install.** Dump `helm show values podinfo/podinfo` to a file. List five values you would change for production and explain why. Use `helm show chart` to find the chart `version`, `appVersion` and `kubeVersion`.
4. **Install with flags.** Install release `core1` into namespace `lesson53` (create it) with `replicaCount=2`, `ui.message="core1"`, `--wait`. Verify with `helm list`, `helm status`, `kubectl get pods`. Use `helm get values core1` and `helm get values core1 --all`; explain the difference.
5. **Values file and precedence.** Create `a.yaml` (`replicaCount: 1`, `ui.message: "A"`) and `b.yaml` (`ui.message: "B"`). Run `helm install prec podinfo/podinfo -n lesson53 -f a.yaml -f b.yaml --set replicaCount=3 --dry-run` and read the rendered replicas and message. Swap the `-f` order and confirm the result changes. State the precedence rule in your own words.
6. **Upgrade and history.** Upgrade `core1` using a new values file (replicas 3, resource requests set). Check `helm history core1`. Then run an upgrade with only `--set ui.color=...` and without `-f`; observe what is lost. Repair using `-f` and then using `--reset-then-reuse-values`.
7. **Rollback.** Roll back `core1` to revision 1. Show the history and explain why the revision number is not 1 again. Compare `helm get values --revision 1` and `--revision 3`.
8. **Failure handling.** Upgrade `core1` with a non-existent image tag and `--atomic --timeout 90s`. Report what Helm printed and what `helm history` shows. Repeat without `--atomic` (use `--wait --timeout 60s`) and compare cluster state; fix by rolling back.
9. **Release storage.** List the Helm Secrets in `lesson53`, read the labels of the latest one, decode its payload with `base64 -d | base64 -d | gunzip | jq 'keys'`, and extract the release values (`.config`). Explain why this is a reason not to put real passwords in values.
10. **Cleanup.** Uninstall `core1` with `--keep-history`, list uninstalled releases, bring it back with `helm rollback`, then uninstall for real. Delete the namespace. List what `uninstall` did not remove (if anything).

### Stretch

1. **Chart dissection.** `helm pull podinfo/podinfo --untar`. Draw the directory tree, annotate each file's purpose, and list every key of `Chart.yaml` that is present. Count how many Kubernetes object kinds the chart can produce (look at the `templates/` filenames) and which are off by default (check `values.yaml` flags such as `ingress.enabled`, `hpa.enabled`).
2. **OCI and pinning.** Install the same app via `helm install oci://ghcr.io/stefanprodan/charts/podinfo --version <v>` and via the classic repo; compare `helm get manifest` output for equality (`diff`). Write a one-paragraph recommendation: classic repo vs OCI for your team.
3. **Idempotent deploy script.** Write `deploy.sh` that, given `ENV=dev|prod`, runs `helm upgrade --install podinfo podinfo/podinfo -n podinfo-$ENV --create-namespace --version <pinned> -f values-$ENV.yaml --atomic --wait --timeout 3m` and prints `helm status`. Use `set -euo pipefail` (lesson 16-17). Run it twice for both envs.
4. **Stuck release drill.** Start an install with a very large `replicaCount` and an impossible resource request, interrupt it with Ctrl-C mid-wait, inspect `helm history` and `helm list --pending`. Recover the release and document the exact commands.
5. **Immutable field drill.** Install a chart, then try to upgrade after changing a selector or Service `clusterIP` through values (if the chart allows) or by manually editing the Deployment's selector label via `kubectl patch` plus an upgrade. Record the error and the recovery path.
6. **Dependency exploration.** Pull a chart that has dependencies (e.g. `bitnami/wordpress` via OCI), run `helm dependency list`, find the keys controlling the sub-charts (`mariadb.*`, `condition:` entries in `Chart.yaml`), and install it with the database sub-chart disabled and an external DB host set (you do not have to complete the install; capture the `--dry-run` output and explain what changed).
7. **Install a plugin.** Install `helm-diff`, run `helm diff upgrade` for a pending change, and write when you would use it in a review process.
8. **Audit a third-party chart.** Pick a chart from Artifact Hub, run `helm show all`, `helm install --dry-run` and list: which ClusterRoles/ClusterRoleBindings it creates, whether it runs as root, whether it ships CRDs and webhooks. Decide whether you would install it in production and why.

## Further Reading

- Helm docs home: https://helm.sh/docs/
- Using Helm (install, upgrade, rollback): https://helm.sh/docs/intro/using_helm/
- Quickstart: https://helm.sh/docs/intro/quickstart/
- Charts (structure, Chart.yaml, dependencies): https://helm.sh/docs/topics/charts/
- Chart repositories: https://helm.sh/docs/topics/chart_repository/
- Registries (OCI support): https://helm.sh/docs/topics/registries/
- CLI command reference: https://helm.sh/docs/helm/
- Helm best practices (values, labels, CRDs): https://helm.sh/docs/chart_best_practices/
- Artifact Hub: https://artifacthub.io/
- Podinfo chart and source: https://github.com/stefanprodan/podinfo
- helm-diff plugin: https://github.com/databus23/helm-diff
- Lesson 54 (Helm templating and customization) and lesson 55 (practice: build and deploy a multi-tier chart) continue this module.

# Lesson 35: kubectl & Core Resource Management

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lesson 34 (Kubernetes architecture, kubeconfig, contexts, declarative model), Lessons 30-33 (containers, images), Lesson 24 (YAML syntax). A working cluster and `kubectl` access — a local `kind`/`minikube`/`k3s`/`rke2` cluster or a shared training cluster. Verify with `kubectl get nodes` before the lesson.

## Learning Objectives

By end of lesson student can:
- Explain what `kubectl` does: it turns commands into REST calls to the kube-apiserver
- Use the core verbs: `get`, `describe`, `apply -f`, `delete`, `edit`, `patch --type merge`
- Distinguish imperative commands, imperative object configuration and declarative `apply`
- Control output with `-o wide`, `-o yaml`, `-o json`, `-o name`, `-o jsonpath`, `-o custom-columns`
- Write working `jsonpath` expressions: field paths, wildcards, filters, `range`, formatting
- Create namespaces, target them with `-n`, list across all with `-A`, and set a default namespace in the current context
- Discover resource types and fields with `kubectl api-resources`, `api-versions`, `explain`
- Label, annotate and select objects with equality-based and set-based selectors
- Preview changes with `--dry-run=client` and `--dry-run=server`
- Read `kubectl describe` output and the Events section to troubleshoot a failing object
- Use shell completion, aliases and `--help` to work fast

## Topics

- Core commands: `get`, `describe`, `apply` (`-f`, `-k`), `delete`, `edit`, `patch` (`--type merge`)
- Output formats: `-o yaml`, `-o json`, `-o wide`, `-o jsonpath`, `-o custom-columns`
- Namespaces: `create`, `-n` flag, `--all-namespaces`; `config set-context --current --namespace`
- Resource discovery: `kubectl explain`, `kubectl api-resources`, `kubectl api-versions`
- Labels / selectors / annotations: `--selector` (`-l`), `kubectl label`, `kubectl annotate`; dry-run

## Concepts

### What kubectl is

`kubectl` is the command-line client for the Kubernetes API. It is **not** magic: every command is translated into an HTTPS request to the **kube-apiserver** (Lesson 34). `kubectl get pods` is a `GET /api/v1/namespaces/default/pods`. `kubectl apply -f x.yaml` is a `PATCH` (or `POST` on first create). `kubectl delete pod web` is a `DELETE`.

Proof — run any command with verbosity 8 and read the HTTP calls:

```bash
kubectl get pods -v=8
# I0101 ... round_trippers.go ... GET https://127.0.0.1:6443/api/v1/namespaces/default/pods?limit=500
# I0101 ... Response Status: 200 OK
```

Which cluster, user and namespace `kubectl` uses comes from the **kubeconfig** file (`~/.kube/config`, or `$KUBECONFIG`), covered in Lesson 34. The *current context* = cluster + user + default namespace. Everything in this lesson assumes the current context points at a working cluster.

### Three ways to manage objects

| Style | Example | When |
|---|---|---|
| **Imperative commands** | `kubectl run web --image=nginx`, `kubectl create namespace dev`, `kubectl label ...` | Quick tests, learning, one-off debugging. Not reproducible. |
| **Imperative object configuration** | `kubectl create -f pod.yaml`, `kubectl replace -f pod.yaml`, `kubectl delete -f pod.yaml` | Rare. `create` fails if the object already exists. |
| **Declarative** | `kubectl apply -f pod.yaml` (or `-f dir/`) | **Production.** Files live in Git; `apply` makes the cluster match the files. Idempotent. |

Rule of thumb: *imperative to explore, declarative to keep*. In real projects nobody types `kubectl run` for production workloads — manifests in Git are applied by hand or (later in this course) by a CI/CD pipeline.

### How `apply` works (three-way merge, briefly)

`kubectl apply` stores the last applied configuration in an annotation on the object (`kubectl.kubernetes.io/last-applied-configuration`), then computes a patch from: (1) your new file, (2) the last applied version, (3) the live object. Fields you removed from the file are removed from the object; fields set by other actors (controllers, `kubectl scale`, autoscalers) that your file never mentioned are left alone. That is why `apply` is safe to re-run.

Newer clusters can use **server-side apply** (`kubectl apply --server-side`) where the API server tracks field ownership (`managedFields`). Mention it; the default client-side apply is enough for this course.

### Resources, kinds, API groups, versions

Every object has:
- `apiVersion` — `group/version`, e.g. `v1` (core group, no name), `apps/v1`, `networking.k8s.io/v1`
- `kind` — the type, e.g. `Pod`, `Namespace`, `Deployment`
- `metadata` — `name`, `namespace`, `labels`, `annotations`, `uid`, `creationTimestamp`, ...
- `spec` — the **desired** state you write
- `status` — the **observed** state, written by the system (read-only for you)

A **resource** is the REST endpoint name (plural, lowercase: `pods`); the **kind** is the object type (`Pod`). `kubectl` accepts plural, singular and short names: `pods` = `pod` = `po`.

### Namespaces

A **namespace** is a virtual partition of one cluster. Names of namespaced objects must be unique *within* a namespace, not across the cluster. Use them to separate teams, apps or environments, and to attach quotas and access rules (later lessons).

Default namespaces in a fresh cluster:

| Namespace | Purpose |
|---|---|
| `default` | Where objects land if you specify nothing |
| `kube-system` | Control-plane add-ons: CoreDNS, kube-proxy, (on some clusters) metrics-server |
| `kube-public` | Readable by everyone, even unauthenticated; rarely used |
| `kube-node-lease` | Node heartbeat Lease objects |

Not all resources are namespaced. **Cluster-scoped** examples: `nodes`, `namespaces`, `persistentvolumes`, `storageclasses`, `clusterroles`. `kubectl api-resources --namespaced=false` lists them.

### Labels, selectors, annotations

- **Labels** — key/value pairs for *identifying and grouping* objects. Meaningful to users and to the system. Used by selectors. Keys like `app`, `tier`, `env`, `version`.
- **Selectors** — queries over labels. Used by `kubectl get -l` and, as you will see in later lessons, by Deployments and Services to find the Pods they manage.
- **Annotations** — key/value pairs for *non-identifying metadata*: build info, URLs, tool settings, change notes. Cannot be used in selectors. Can hold large values.

Label constraints: key = optional DNS-prefix + `/` + name (max 63 chars, alphanumerics, `-`, `_`, `.`); value max 63 chars, alphanumerics, `-`, `_`, `.`, may be empty.

Recommended common labels (`app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`, `app.kubernetes.io/managed-by`) give tools a shared vocabulary.

### Dry-run

- `--dry-run=client` — kubectl builds the object locally and prints it; **nothing is sent** to the API server. Use it to generate YAML.
- `--dry-run=server` — the request goes to the API server and runs through authentication, admission and validation but is **not persisted**. Use it to check "would this really be accepted?".

## Commands / Syntax Reference

### Command shape

```
kubectl [command] [TYPE] [NAME] [flags]
kubectl get pods web -n dev -o yaml
```

### Core commands

| Command | Purpose | Example |
|---|---|---|
| `kubectl get` | List objects | `kubectl get pods -n kube-system` |
| `kubectl describe` | Human-readable detail + Events | `kubectl describe pod web` |
| `kubectl apply -f` | Create/update from file, dir or URL | `kubectl apply -f pod.yaml` |
| `kubectl apply -k` | Apply a kustomization directory | `kubectl apply -k ./overlay` (taught in lessons 56-58) |
| `kubectl create` | Create (fails if exists) | `kubectl create namespace dev` |
| `kubectl delete` | Delete objects | `kubectl delete pod web` / `kubectl delete -f pod.yaml` |
| `kubectl edit` | Open live object in `$EDITOR`, apply on save | `kubectl edit pod web` |
| `kubectl patch` | Change fields without a file | `kubectl patch pod web --type merge -p '{"metadata":{"labels":{"x":"y"}}}'` |
| `kubectl replace` | Replace object with file contents | `kubectl replace -f pod.yaml` |
| `kubectl label` | Add/change/remove labels | `kubectl label pod web tier=frontend` |
| `kubectl annotate` | Add/change/remove annotations | `kubectl annotate pod web owner=team-a` |
| `kubectl explain` | Show API field documentation | `kubectl explain pod.spec.containers` |
| `kubectl api-resources` | List resource types | `kubectl api-resources --namespaced=true` |
| `kubectl api-versions` | List group/version strings | `kubectl api-versions` |
| `kubectl logs` | Container stdout/stderr | `kubectl logs web` |
| `kubectl exec` | Run a command in a container | `kubectl exec -it web -- sh` |
| `kubectl wait` | Block until a condition is true | `kubectl wait --for=condition=Ready pod/web --timeout=60s` |
| `kubectl config` | Manage kubeconfig | `kubectl config get-contexts` |
| `kubectl version` | Client + server versions | `kubectl version` |
| `kubectl cluster-info` | Control-plane endpoints | `kubectl cluster-info` |

### Common `get` flags

| Flag | Meaning |
|---|---|
| `-n NS` / `--namespace NS` | Target namespace |
| `-A` / `--all-namespaces` | Across all namespaces |
| `-o wide` | Extra columns (node, IP, ...) |
| `-o yaml` / `-o json` | Full object |
| `-o name` | Only `type/name` — great for scripting |
| `-o jsonpath='...'` | Extract fields with JSONPath |
| `-o custom-columns=...` | Build your own table |
| `-l key=value` / `--selector` | Filter by label |
| `--field-selector` | Filter by field (`status.phase=Running`, `metadata.name=web`) |
| `-w` / `--watch` | Stream changes |
| `--show-labels` | Add a LABELS column |
| `-L key1,key2` | Add one column per label key |
| `--sort-by='{.metadata.creationTimestamp}'` | Sort by JSONPath |
| `--no-headers` | Drop header row |
| `--chunk-size`, `--limit` | Paginate large lists |

### Delete flags

| Flag | Meaning |
|---|---|
| `--all` | Every object of the type in the namespace |
| `-l key=value` | Only matching |
| `--grace-period=0 --force` | Skip graceful shutdown (last resort) |
| `--wait=false` | Return immediately |
| `--cascade=foreground\|background\|orphan` | What happens to dependents |
| `-f file` / `-f dir/` | Delete what the files define |

### Short names (memorize these)

| Short | Full | Short | Full |
|---|---|---|---|
| `po` | pods | `ns` | namespaces |
| `no` | nodes | `deploy` | deployments |
| `rs` | replicasets | `svc` | services |
| `cm` | configmaps | `pv` / `pvc` | persistentvolume(claim)s |
| `sa` | serviceaccounts | `ing` | ingresses |
| `ep` | endpoints | `ds` | daemonsets |
| `sts` | statefulsets | `hpa` | horizontalpodautoscalers |

(The objects behind most of these names are taught in later lessons — here you only need to recognize the abbreviations.)

## Examples / Walkthrough

### 1. Check access (2 min)

```bash
kubectl version                  # client + server version; skew of ±1 minor is supported
kubectl cluster-info             # API server + CoreDNS URLs
kubectl config current-context   # which cluster am I talking to?
kubectl get nodes -o wide        # nodes, internal IP, OS image, kernel, runtime
kubectl get ns                   # default, kube-system, kube-public, kube-node-lease
```

**Say it out loud:** "Always check the context before a destructive command." Running `kubectl delete` against production because the context was wrong is the classic accident.

### 2. First objects the fast way

```bash
kubectl run web --image=nginx:1.27
kubectl get pods                       # STATUS goes Pending -> ContainerCreating -> Running
kubectl get pods -w                    # watch; Ctrl-C to stop
kubectl get pods -o wide               # which node, which Pod IP
kubectl describe pod web               # scroll to Events at the bottom
kubectl logs web
kubectl exec -it web -- sh -c 'nginx -v; exit'
```

`kubectl run` creates a single **Pod** — the smallest deployable unit, a wrapper around one or more containers (details in lesson 36). `kubectl create deployment api --image=nginx:1.27` creates a Deployment, which manages Pods for you (also lesson 36). Today we only use them as demo objects to practise `kubectl` itself.

```bash
kubectl create deployment api --image=nginx:1.27
kubectl get deploy,pods                # two types in one command, comma-separated, no spaces
```

### 3. Reading `describe`

```bash
kubectl describe pod web
```

Sections, top to bottom: **Name / Namespace / Node / Start Time / Labels / Annotations / Status / IP**, **Containers** (image, state, ports, restart count, env, mounts), **Conditions** (`PodScheduled`, `Initialized`, `ContainersReady`, `Ready`), **Volumes**, **QoS Class**, **Events**. 

The **Events** block is the first place to look when something is stuck:

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  12s   default-scheduler  Successfully assigned default/web to node-1
  Normal  Pulling    11s   kubelet            Pulling image "nginx:1.27"
  Normal  Pulled     6s    kubelet            Successfully pulled image
  Normal  Created    6s    kubelet            Created container web
  Normal  Started    6s    kubelet            Started container web
```

Events live only about an hour. Cluster-wide view:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -A --field-selector type=Warning
```

Reproduce a failure to see it in `describe`:

```bash
kubectl run broken --image=nginx:does-not-exist
kubectl get pod broken            # ErrImagePull -> ImagePullBackOff
kubectl describe pod broken | tail -8   # Failed to pull image ... not found
kubectl delete pod broken
```

### 4. Declarative: `apply -f`

```bash
mkdir -p ~/k8s-lab && cd ~/k8s-lab
cat > pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello
    env: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
EOF

kubectl apply -f pod.yaml      # pod/hello created
kubectl apply -f pod.yaml      # pod/hello unchanged   <- idempotent
kubectl create -f pod.yaml     # Error: pods "hello" already exists   <- create is not
```

Edit the file: change `env: dev` to `env: staging`, then:

```bash
kubectl apply -f pod.yaml      # pod/hello configured
kubectl get pod hello --show-labels
```

Some Pod fields are **immutable**. Change the image in the file to `nginx:1.26` and apply — for a bare Pod the image field *is* updatable, but changing e.g. `spec.containers[0].name` or ports is rejected:

```
The Pod "hello" is invalid: spec: Forbidden: pod updates may not change fields other than ...
```

Fix: `kubectl delete -f pod.yaml && kubectl apply -f pod.yaml`, or `kubectl replace --force -f pod.yaml` (delete + create).

Apply a whole directory or a URL:

```bash
kubectl apply -f ./manifests/            # every .yaml/.yml/.json in the dir
kubectl apply -f ./manifests/ -R         # recurse into subdirectories
kubectl apply -f https://example.com/some.yaml
kubectl apply -f a.yaml -f b.yaml
```

Multi-document file: separate objects with `---` on its own line. `apply -f` handles all of them.

`kubectl apply -k <dir>` applies a *kustomization* (built-in templating by overlay). It is taught in the Kustomize lessons 56-58; for now just recognize the flag.

### 5. `edit` and `patch`

```bash
kubectl edit pod hello
# opens the live YAML in $EDITOR (vi by default). Save+quit = applied. Invalid YAML = re-opened with an error comment.
export KUBE_EDITOR=nano       # pick another editor
```

`edit` changes the **live** object; your file in Git is now out of date. Prefer editing the file and `apply`.

`patch` changes specific fields without opening anything. Three patch types:

| `--type` | Format | Behavior |
|---|---|---|
| `strategic` (default) | JSON/YAML | Kubernetes-aware merge; lists merged by a key where defined (built-in types only) |
| `merge` | JSON Merge Patch (RFC 7386) | Objects merged recursively; **lists replaced entirely**; `null` deletes a key |
| `json` | JSON Patch (RFC 6902) | Explicit operations: `add`, `remove`, `replace`, `test`, `move`, `copy` |

```bash
# merge patch: add a label and an annotation
kubectl patch pod hello --type merge -p '{"metadata":{"labels":{"tier":"frontend"},"annotations":{"owner":"team-a"}}}'

# merge patch: delete a label by setting it to null
kubectl patch pod hello --type merge -p '{"metadata":{"labels":{"tier":null}}}'

# merge patch with YAML from a file
cat > patch.yaml <<'EOF'
metadata:
  annotations:
    note: "patched from file"
EOF
kubectl patch pod hello --type merge --patch-file patch.yaml

# json patch
kubectl patch pod hello --type json -p '[{"op":"add","path":"/metadata/labels/team","value":"platform"}]'

# patch a subresource-free field on a Deployment (demo object from step 2)
kubectl patch deployment api --type merge -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.26"}]}}}}'
```

Quoting tip: JSON contains double quotes, so wrap the patch in **single quotes** in bash. In PowerShell use `--patch-file`.

**Gotcha of merge patch:** because lists are replaced, patching `containers` with only `[{"name":"nginx","image":"..."}]` *replaces the whole container list*. Strategic merge (the default) merges list items by `name` for built-in types. When in doubt, patch scalar fields or use `--type json` with an exact path.

### 6. Output formats

```bash
kubectl get pod hello                    # default table
kubectl get pod hello -o wide            # + IP, NODE, NOMINATED NODE, READINESS GATES
kubectl get pod hello -o yaml            # full object incl. status
kubectl get pod hello -o json | head -30
kubectl get pods -o name                 # pod/hello  pod/web
kubectl get pods -o yaml > backup.yaml   # a List object
```

`-o yaml` of a live object contains noise you must **strip** before re-applying elsewhere: `status`, `metadata.uid`, `resourceVersion`, `creationTimestamp`, `managedFields`, the `last-applied-configuration` annotation, and `spec.nodeName`.

#### JSONPath

`-o jsonpath='<template>'` evaluates a JSONPath template against the JSON form of the result. Rules for kubectl's dialect:

- `{.metadata.name}` — expressions are wrapped in `{ }`; `.` is the root (the object, or the `List` wrapper for multiple objects)
- `{.items[*].metadata.name}` — for a list of objects, kubectl returns a `List` with an `items` array
- `[*]` all elements, `[0]` first, `[-1]` last, `[0:2]` slice
- `[?(@.field=="value")]` filter; `@` is the current element
- `{range .items[*]}...{end}` loop; `{"\n"}` and `{"\t"}` for newline/tab
- `..` recursive descent: `{..image}`
- A key containing dots must be escaped: `{.metadata.labels.app\.kubernetes\.io/name}`

```bash
# single object
kubectl get pod hello -o jsonpath='{.metadata.name}'
kubectl get pod hello -o jsonpath='{.status.podIP}{"\n"}'
kubectl get pod hello -o jsonpath='{.spec.containers[0].image}{"\n"}'
kubectl get pod hello -o jsonpath='{.status.phase}{"\n"}'

# list of objects
kubectl get pods -o jsonpath='{.items[*].metadata.name}'               # space-separated names on one line
kubectl get pods -o jsonpath='{.items[0].metadata.name}'

# one per line with range
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# name + node, tab-separated
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'

# all container images in the namespace
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# all images across the cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# filter: only Running pods' names (jsonpath filter)
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# nodes: internal IPs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'

# nodes: kubelet version and OS
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\t"}{.status.nodeInfo.osImage}{"\n"}{end}'

# label with dots in key
kubectl get pod hello -o jsonpath='{.metadata.labels.app}'

# whole labels map
kubectl get pod hello -o jsonpath='{.metadata.labels}{"\n"}'

# restart counts
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# jsonpath-as-json (kubectl 1.19+)
kubectl get pods -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n'
kubectl get pod hello -o jsonpath-as-json='{.spec.containers[*]}'
```

Debug tip: if a template returns nothing, print the full thing with `-o json | less` and walk the path by eye. The most common error is forgetting `.items[*]` on a list query.

For heavier work pipe JSON to `jq`:

```bash
kubectl get pods -o json | jq -r '.items[] | [.metadata.name, .status.phase] | @tsv'
```

#### custom-columns

`-o custom-columns=HEADER:JSONPATH,HEADER2:JSONPATH2` builds a table. Paths here are **without** `{}` and **without** `.items[*]` — kubectl applies them to each item.

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[*].image,IP:.status.podIP'
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory,KUBELET:.status.nodeInfo.kubeletVersion
kubectl get pods -o custom-columns='NAME:.metadata.name,APP:.metadata.labels.app'

# header-less
kubectl get pods -o custom-columns=NAME:.metadata.name --no-headers

# from a template file
cat > cols.txt <<'EOF'
NAME            NAMESPACE           NODE
metadata.name   metadata.namespace  spec.nodeName
EOF
kubectl get pods -A -o custom-columns-file=cols.txt
```

Sort and filter on the server side before formatting:

```bash
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime
kubectl get pods --field-selector status.phase=Running
kubectl get pods -A --field-selector metadata.namespace!=kube-system
kubectl get pods --field-selector spec.nodeName=node-1
```

`--field-selector` supports only a few fields per resource, with `=`, `==`, `!=`; label selectors (`-l`) support more complex expressions.

### 7. Namespaces

```bash
kubectl get ns
kubectl create namespace dev
kubectl create ns staging
kubectl get ns --show-labels         # every namespace gets kubernetes.io/metadata.name=<name>

# declarative
cat > ns.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: qa
  labels:
    team: platform
EOF
kubectl apply -f ns.yaml
```

Using a namespace:

```bash
kubectl run web --image=nginx:1.27 -n dev       # same name as in default — no clash
kubectl get pods -n dev
kubectl get pods --namespace=staging
kubectl get pods --all-namespaces               # or -A
kubectl get pods -A -o wide
kubectl get pods -n kube-system
kubectl get all -n dev                          # "all" = a curated set of common types, NOT literally everything
```

Set a default namespace for the current context so you stop typing `-n`:

```bash
kubectl config set-context --current --namespace=dev
kubectl config view --minify | grep namespace:
kubectl get pods                                 # now means -n dev
kubectl config set-context --current --namespace=default   # switch back
```

Namespace in a manifest: `metadata.namespace: dev`. If the file has none, `apply -n dev -f pod.yaml` sets it. If both exist and **disagree**, kubectl errors: `the namespace from the provided object does not match the namespace "dev"`.

Delete a namespace = delete **everything inside** it:

```bash
kubectl delete namespace qa          # blocks until contents are gone; may sit in Terminating if a finalizer hangs
```

> **Danger:** `kubectl delete ns <name>` is destructive and irreversible. Check the name twice. Never run `kubectl delete all --all` in a context you have not verified.

### 8. Resource discovery

Which types exist in this cluster (built-ins + any CRDs installed later)?

```bash
kubectl api-resources                              # NAME SHORTNAMES APIVERSION NAMESPACED KIND
kubectl api-resources -o wide                      # + VERBS (get, list, create, delete, ...)
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false           # cluster-scoped only
kubectl api-resources --api-group=apps             # one group
kubectl api-resources --verbs=list -o name         # everything you can list
kubectl api-versions                               # v1, apps/v1, batch/v1, networking.k8s.io/v1, ...
```

Use `api-resources` to find the right `apiVersion` + `kind` to put in a YAML file: look up the resource, read its `APIVERSION` column.

`kubectl explain` is the built-in field reference — **offline-capable**, always matches your cluster version:

```bash
kubectl explain pod                                # description + top-level fields
kubectl explain pod.spec                           # fields under spec
kubectl explain pod.spec.containers                # list of containers + each field's type
kubectl explain pod.spec.containers.ports
kubectl explain pod.metadata.labels
kubectl explain pod.spec --recursive               # full tree (long)
kubectl explain deployment.spec.selector
kubectl explain pods --api-version=v1
kubectl explain pod.spec.restartPolicy             # shows allowed values & default
```

Output format: `FIELD: name <type>`, a `DESCRIPTION`, and `-required-` where mandatory. Habit to build: **never guess a field name, run `explain`.**

### 9. Labels and selectors

```bash
# create demo pods with labels
kubectl run fe1 --image=nginx:1.27 --labels="app=shop,tier=frontend,env=dev"
kubectl run fe2 --image=nginx:1.27 --labels="app=shop,tier=frontend,env=prod"
kubectl run be1 --image=nginx:1.27 --labels="app=shop,tier=backend,env=dev"
kubectl run db1 --image=nginx:1.27 --labels="app=other,tier=backend,env=prod"

kubectl get pods --show-labels
kubectl get pods -L app,tier,env                   # columns per label
```

Equality-based selectors:

```bash
kubectl get pods -l app=shop
kubectl get pods -l app!=shop
kubectl get pods -l app=shop,tier=frontend         # comma = AND
kubectl get pods --selector tier=backend
```

Set-based selectors (quote them — the shell eats parentheses):

```bash
kubectl get pods -l 'env in (dev,staging)'
kubectl get pods -l 'env notin (prod)'
kubectl get pods -l 'tier'                          # label key exists
kubectl get pods -l '!tier'                         # label key does not exist
kubectl get pods -l 'app=shop,env in (dev,prod),tier!=backend'
```

Modify labels:

```bash
kubectl label pod fe1 version=v1                    # add
kubectl label pod fe1 version=v2 --overwrite        # change (needed to overwrite)
kubectl label pod fe1 version-                      # remove (trailing minus)
kubectl label pods -l app=shop owner=team-a         # label many at once
kubectl label pods --all stage=training             # all pods in namespace
kubectl label node node-1 disk=ssd                  # nodes can be labeled too (used for scheduling in lesson 46)
kubectl label ns dev team=platform
```

Selectors work with any verb that takes a list:

```bash
kubectl get pods -l tier=backend -o name
kubectl delete pods -l env=dev                      # delete by label — check with `get -l` first!
kubectl logs -l app=shop --tail=5 --prefix          # logs from multiple pods
kubectl describe pods -l tier=frontend
```

**Habit:** run the same selector with `get` before running it with `delete`.

Annotations:

```bash
kubectl annotate pod fe2 owner="team-a@example.com"
kubectl annotate pod fe2 description="front end, prod" --overwrite
kubectl annotate pod fe2 owner-                    # remove
kubectl get pod fe2 -o jsonpath='{.metadata.annotations}{"\n"}'
kubectl describe pod fe2 | sed -n '/Annotations/,/Status/p'
```

### 10. Dry-run and generating YAML

```bash
# generate a manifest skeleton, send nothing
kubectl run gen --image=nginx:1.27 --port=80 --dry-run=client -o yaml
kubectl run gen --image=nginx:1.27 --dry-run=client -o yaml > gen.yaml

kubectl create namespace demo --dry-run=client -o yaml
kubectl create deployment api2 --image=nginx:1.27 --dry-run=client -o yaml > api2.yaml
```

Edit the skeleton, then `kubectl apply -f`. This is the fastest way to write a correct manifest, and the standard exam-style technique.

```bash
# client dry-run of an apply: validates locally and shows what it would send
kubectl apply -f pod.yaml --dry-run=client

# server dry-run: full API-server validation + admission, nothing persisted
kubectl apply -f pod.yaml --dry-run=server
kubectl apply -f broken.yaml --dry-run=server   # catches errors client mode can't (unknown field, bad value, quota, webhook)
```

Validation flags:

```bash
kubectl apply -f x.yaml --validate=strict        # error on unknown fields (default in modern kubectl)
kubectl apply -f x.yaml --validate=false         # skip schema check (avoid)
```

### 11. `delete` and cleanup

```bash
kubectl delete pod hello                         # graceful: SIGTERM, wait terminationGracePeriodSeconds (30 s), then SIGKILL
kubectl delete pod hello --now                   # grace period 1 s
kubectl delete pod hello --grace-period=0 --force   # remove from API immediately (process may linger on the node)
kubectl delete -f pod.yaml                       # delete what the file defines
kubectl delete pods fe1 fe2 be1 db1              # several by name
kubectl delete pods -l app=shop
kubectl delete pods --all -n dev                 # every pod in namespace dev
kubectl delete deployment api
kubectl delete ns dev staging qa demo
```

Note: deleting a Pod that a Deployment manages makes the Deployment create a replacement. To remove such Pods permanently, delete the Deployment (more in lesson 36).

### 12. Productivity setup

```bash
# bash completion (zsh: replace bash with zsh)
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# alias + completion for the alias
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

# handy aliases
alias kgp='kubectl get pods'
alias kd='kubectl describe'
alias kaf='kubectl apply -f'
alias kns='kubectl config set-context --current --namespace'

# exam / lab shortcuts
export do="--dry-run=client -o yaml"      # then: kubectl run x --image=nginx $do
export now="--grace-period=0 --force"
```

Also helpful: `kubectl <command> --help` (examples included), `kubectl options` (global flags), `kubectl get <TAB>`.

## Common Pitfalls

- **Wrong context/cluster** — you deleted in prod because the context was prod. Always `kubectl config current-context` first; consider a prompt that shows it.
- **Forgetting `-n`** — "my pod disappeared!" It is in another namespace. Use `-A` to search, or set the context's default namespace.
- **`kubectl get all` does not show everything** — it covers a fixed set (Pods, Services, Deployments, ReplicaSets, StatefulSets, Jobs, ...). ConfigMaps, Secrets, Ingresses, PVCs are not included. Use `api-resources` to list real types.
- **`create` vs `apply`** — `create` errors when the object exists and leaves no last-applied annotation; mixing `create` and later `apply` triggers a warning (`missing the kubectl.kubernetes.io/last-applied-configuration annotation`). Pick `apply` for anything you maintain.
- **Editing live objects (`edit`, `patch`) and forgetting the file in Git** — next `apply` silently reverts the change (or the reverse). Keep Git the source of truth.
- **Immutable fields** — some fields (most of a Pod spec after creation, a Deployment's `selector`, Service `clusterIP`) cannot be changed in place. Delete and recreate.
- **Merge patch replaces lists** — `--type merge` with a `containers` list overwrites the whole list. Use strategic merge (default), JSON patch, or edit the file.
- **JSONPath on a list without `.items[*]`** — returns empty. For single objects use `{.metadata.name}`, for lists `{.items[*].metadata.name}`.
- **custom-columns with braces** — custom-columns paths are plain (`.metadata.name`), no `{}` and no `.items`.
- **Unquoted set selectors** — `-l env in (dev,prod)` is a shell syntax error. Always single-quote: `-l 'env in (dev,prod)'`.
- **`label` without `--overwrite`** — `error: 'x' already has a value (y), and --overwrite is false`.
- **Force-deleting everything** — `kubectl delete pods --all --force --grace-period=0` skips graceful shutdown; data loss and orphaned processes are possible.
- **Applying `-o yaml` dumps verbatim** — `status`, `uid`, `resourceVersion`, `managedFields` come along; apply fails or fights with controllers. Generate from `--dry-run=client -o yaml` instead.
- **Terminating namespace stuck** — a finalizer or an unavailable API service blocks deletion; check `kubectl get ns <name> -o yaml` for `status.conditions`.
- **Client/server version skew** — kubectl supports ±1 minor version relative to the API server. Larger gaps cause odd errors.
- **Tabs in YAML** — YAML forbids tab indentation (Lesson 24); use spaces.

## FAQ

**Q: What is the difference between `kubectl apply` and `kubectl create`?**
A: `create` makes a new object and fails if it already exists. `apply` creates it if missing, otherwise computes a diff against the last applied configuration and updates only what changed. `apply` is idempotent and the standard for declarative workflows.

**Q: What is the difference between `edit` and `patch`?**
A: Both modify the live object. `edit` opens the full YAML in an editor interactively; `patch` applies a small change non-interactively, so it is scriptable. Neither updates your manifest file.

**Q: Which patch type should I use?**
A: `--type merge` for simple object-field changes (labels, annotations, scalar values) — easy to reason about. Default `strategic` for Kubernetes built-in types when a list should be merged by key. `--type json` when you need an exact path, an array element index, or a `remove`/`test` operation.

**Q: How do I see the YAML of an object that is already running?**
A: `kubectl get <type> <name> -o yaml`. Strip `status`, `uid`, `resourceVersion`, `creationTimestamp`, `managedFields` before reusing it.

**Q: How do I generate a YAML template instead of writing it from scratch?**
A: `kubectl run NAME --image=IMG --dry-run=client -o yaml` (Pod) or `kubectl create deployment NAME --image=IMG --dry-run=client -o yaml`. Redirect to a file, edit, `kubectl apply -f`.

**Q: What does `--dry-run=client` vs `--dry-run=server` mean?**
A: `client` — built locally, never sent; good for generating YAML. `server` — sent to the API server, validated and run through admission, but not saved; good for answering "will the cluster accept this?".

**Q: How do I list the pods of all namespaces?**
A: `kubectl get pods -A` (same as `--all-namespaces`). Add `-o wide` for node and IP.

**Q: How do I change the default namespace permanently?**
A: `kubectl config set-context --current --namespace=NAME`. It is stored in the kubeconfig's current context; check with `kubectl config view --minify`.

**Q: What is the difference between a label and an annotation?**
A: Labels identify and group objects and are used by selectors (e.g. `-l app=web`, and by controllers/Services in later lessons). Annotations store arbitrary non-identifying metadata (owner, build URL, tool config) and can't be selected on.

**Q: Why does `kubectl get all` not show my ConfigMap/Secret/Ingress?**
A: `all` is a hard-coded category of common workload types. Ask for other types explicitly: `kubectl get cm,secret,ing`. List what exists with `kubectl api-resources`.

**Q: How do I find the right `apiVersion` for a kind?**
A: `kubectl api-resources | grep -i <kind>` shows APIVERSION; `kubectl explain <kind>` confirms and lists fields. `kubectl api-versions` lists every group/version the server offers.

**Q: How do I look up what a field means without Googling?**
A: `kubectl explain <resource>.<field.path>`; add `--recursive` for the full tree. It reflects exactly the version your cluster runs.

**Q: How do I get only names, for scripts?**
A: `kubectl get pods -o name` (gives `pod/web`) or `-o jsonpath='{.items[*].metadata.name}'`, or `--no-headers -o custom-columns=NAME:.metadata.name`.

**Q: How do I filter pods by status?**
A: `--field-selector status.phase=Running` (or `Pending`, `Failed`, `Succeeded`). Pods in `CrashLoopBackOff` still have phase `Running`; check `containerStatuses` or the STATUS column instead.

**Q: A pod stays in `Pending`. Where do I look?**
A: `kubectl describe pod <name>` → Events. Typical messages: `FailedScheduling` (no node fits), image pull problems (`ErrImagePull`), quota exceeded. Later lessons cover the causes in detail.

**Q: Why did my pod come back after I deleted it?**
A: Something manages it (e.g. a Deployment) and recreates it to match the desired state. Delete the owner (`kubectl delete deployment <name>`). Details in lesson 36.

**Q: What is `-v=8` for?**
A: kubectl verbosity. Levels 6-9 print the HTTP requests and responses — useful to see what `apply` actually sent, or to debug auth errors.

**Q: Can I use kubectl with several clusters?**
A: Yes — kubeconfig holds many contexts. `kubectl config get-contexts`, `kubectl config use-context NAME`, or per-command `--context NAME`. Merge files with `KUBECONFIG=a:b kubectl config view --flatten`.

**Q: What is the difference between `-o json` + `jq` and `-o jsonpath`?**
A: `jsonpath` is built into kubectl with a limited syntax; `jq` is a separate tool with a full query language. Use jsonpath for simple extraction in portable scripts, `jq` for transformations.

**Q: What does `kubectl delete --grace-period=0 --force` do and is it safe?**
A: It deletes the object from the API immediately without waiting for the kubelet to confirm the container stopped. Use only for stuck objects; the container process may still be running on the node.

## Practice / Exercise

Clean slate first: `kubectl create namespace lab35` and `kubectl config set-context --current --namespace=lab35`.

### Core

1. **Context check.** Print the current context, the cluster endpoint, server and client versions. Record how many nodes the cluster has and their kubelet versions with one `-o custom-columns` command.
2. **Namespaces.** Create namespaces `dev` and `prod` — one imperatively, one from a YAML file with the label `env: <name>`. Show the labels with `kubectl get ns --show-labels`.
3. **Imperative → declarative.** Run `kubectl run web --image=nginx:1.27 --dry-run=client -o yaml > web.yaml`. Add labels `app: web` and `tier: frontend` in the file. Apply it in namespace `dev`. Apply again — what changes in the output?
4. **create vs apply.** Run `kubectl create -f web.yaml -n dev`. What error appears? Explain why.
5. **Describe and Events.** Run `kubectl run bad --image=nginx:nope -n dev`. Find the exact failure reason in `describe` Events. Delete the pod.
6. **Output formats.** For pod `web`: print its IP with `jsonpath`; its node with `-o wide`; its image with `jsonpath`; a three-column table (name, phase, node) with `custom-columns`.
7. **Labels.** Create four pods in `prod` with `kubectl run ... --labels=` combining `app=shop|blog`, `tier=frontend|backend`. Write one selector for each: all `shop` pods; frontend pods not belonging to `blog`; pods whose `tier` label exists; pods with `app in (shop,blog)`.
8. **Patch.** With `--type merge`, add annotation `owner=you` and label `reviewed=true` to one pod, then remove the label using `null`.
9. **Namespace default.** Switch the current context default to `prod`, list pods without `-n`, switch back to `lab35`.
10. **Discover.** Using only `api-resources` and `explain`, find (a) whether `namespaces` is namespaced, (b) the `apiVersion` of `deployments`, (c) the allowed values of `pod.spec.restartPolicy`.
11. **Clean up.** Delete pods by label in `prod`, then delete the namespaces `dev`, `prod`, `lab35`. Reset the default namespace to `default`.

### Stretch

1. **Pipeline.** One command that prints, for all pods in all namespaces, `namespace/name` and container images, sorted, using `jsonpath` with `range`. Rewrite it with `jq`.
2. **Server dry-run.** Make a manifest with a misspelled field (`containrs:`). Compare the output of `--dry-run=client` and `--dry-run=server` and explain the difference in what each checks.
3. **JSON patch.** Use `--type json` to add a label with a `/` in the key (hint: escape as `~1`). Then use `test` + `replace` ops to change a value only if it matches.
4. **Merge-patch trap.** Create a Deployment (`kubectl create deployment`) with two containers (use `--dry-run=client -o yaml`, edit, apply). Apply a `merge` patch with one container. Show what happened to the second container, then repeat with the default strategic patch.
5. **Selectors at scale.** Label 10 pods with `batch=1..3` using a loop; delete only batch 2 with one command; verify with `-L batch`.
6. **Cluster inventory script.** Write `inventory.sh` that prints node name, role label, capacity CPU/memory and kubelet version as a TSV file using `custom-columns` or `jsonpath`.
7. **Kubeconfig surgery.** Create a second context in a copy of your kubeconfig pointing at the same cluster with namespace `kube-system`; use it for one command with `--kubeconfig` and `--context`.
8. **Observe apply.** Run `kubectl apply -f pod.yaml -v=8` and identify the HTTP method and URL used on create vs update. Read the `last-applied-configuration` annotation from `-o yaml`.

## Further Reading

- kubectl reference: https://kubernetes.io/docs/reference/kubectl/
- kubectl cheat sheet: https://kubernetes.io/docs/reference/kubectl/quick-reference/
- JSONPath support in kubectl: https://kubernetes.io/docs/reference/kubectl/jsonpath/
- Managing objects (imperative vs declarative): https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/
- Labels and selectors: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
- Recommended labels: https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
- Namespaces: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
- Update API objects with kubectl patch: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/
- Server-side apply: https://kubernetes.io/docs/reference/using-api/server-side-apply/

# Lesson 57: Kustomize Advanced – Patches and Config Management

**Module:** Kustomize – Native K8s Templating
**Duration:** 120-150 min
**Prerequisites:** Lessons 35-36 (kubectl, Deployments), 42 (ConfigMap and Secret), 53-55 (Helm), 56 (Kustomize basics: `kustomization.yaml`, base/overlay layout, `namePrefix`, `namespace`, images transformer, `kubectl apply -k`, `kubectl kustomize`)

## Learning Objectives

By end of lesson student can:
- Explain what a patch is in Kustomize and pick the right patch type (strategic merge vs JSON 6902) for a given change
- Write strategic merge patches that add or replace fields, merge lists by key, and delete fields/list items/whole resources with `$patch`
- Write JSON 6902 patches with `add`, `remove`, `replace`, `copy`, `move`, `test` and correct `path` syntax (array index, `-` append, `~1` escaping)
- Use the `patches:` field with `path:` (file) or `patch:` (inline) plus a `target:` selector (kind, name, labelSelector, annotationSelector)
- Add probes, resources, env vars, volumes and annotations to a base Deployment from an overlay without touching the base
- Generate ConfigMaps from literals, files and env files with `configMapGenerator`, and explain the content-hash suffix and the automatic rollout it triggers
- Use `behavior: create|replace|merge` to override a generated ConfigMap in an overlay
- Disable the hash suffix with `generatorOptions` / per-generator `options` and explain when that is a bad idea
- Generate Secrets with `secretGenerator` and explain why that is still not secure for Git
- Compare Kustomize and Helm and describe how to use both together
- Debug Kustomize output with `kustomize build`, `kubectl diff -k` and read the common error messages

## Topics

- Strategic merge patch: overlay YAML merged with base; adding/replacing fields
- JSON 6902 patch: precise operations (add, remove, replace, copy, move) with path
- configMapGenerator: from files and literals; disableNameSuffixHash option
- secretGenerator: generating secrets in Kustomize
- Kustomize vs Helm: trade-offs; using both (helm template | kustomize apply)
- Supporting topics: `patches:` field and `target` selectors, inline patches, `generatorOptions`, `behavior`, debugging and common errors

## Concepts

### Why patches exist (recap and motivation)

In lesson 56 we customized a base with *transformers*: `namePrefix`, `namespace`, `images`, `labels`. Transformers are blunt tools. They change one well-known thing across **all** resources. Real environments need surgical changes:

- prod runs 6 replicas, dev runs 1 → change `spec.replicas` of one Deployment
- prod needs CPU/memory limits, dev does not → add a `resources` block to one container
- prod needs an extra env var, a readiness probe, a `nodeSelector`, an extra volume
- prod Ingress needs a different annotation

A **patch** is a small piece of YAML (or a list of operations) that says "take this resource from the base and change these fields". The base stays untouched and generic; each overlay carries only its differences. That is Kustomize's whole philosophy: **no templates, no placeholders, no `{{ }}` — the base is always a valid, applyable manifest set, and overlays are diffs on top of it.**

Kustomize supports two patch dialects. Both are declared under the same `patches:` field in `kustomization.yaml`; Kustomize detects which dialect you wrote by looking at the content (a YAML list of `op:` objects = JSON 6902; a partial resource = strategic merge).

| | Strategic merge patch (SMP) | JSON 6902 patch |
|---|---|---|
| Looks like | A partial Kubernetes resource | A list of operations (`op`, `path`, `value`) |
| Identifies target by | `apiVersion`/`kind`/`metadata.name` in the patch (or an explicit `target:`) | **Always** needs an explicit `target:` |
| Lists | Merged **by key** (e.g. containers by `name`) | Addressed **by index** (`/containers/0`) |
| Delete | `$patch: delete`, or set field to `null` | `op: remove` |
| Readability | High — looks like the real manifest | Lower — path strings |
| Precision | Medium | High (exact element, exact operation) |
| Works on CRDs | **No merge-key knowledge** → lists are replaced, not merged | Yes, fully |
| Best for | Everyday edits: replicas, resources, env, probes, annotations | Edge cases: remove one arg, append to array, change one annotation with `/` in key, CRDs |

Rule of thumb: **start with strategic merge; reach for JSON 6902 when SMP cannot express it.**

### The base we use for all examples

Everything below patches this small base. Create it once and keep it.

```
kustomize-lab/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

`base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          env:
            - name: LOG_LEVEL
              value: info
            - name: FEATURE_X
              value: "off"
        - name: sidecar
          image: busybox:1.36
          command: ["sh", "-c", "while true; do sleep 3600; done"]
```

`base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

`overlays/prod/kustomization.yaml` starts as:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
```

Check it renders: `kubectl kustomize overlays/prod` (or `kustomize build overlays/prod`).

### The `patches:` field

All patches live under one field:

```yaml
patches:
  - path: replicas-patch.yaml          # patch stored in a file next to kustomization.yaml
  - patch: |-                          # OR inline patch, written right here
      - op: replace
        path: /spec/replicas
        value: 5
    target:
      kind: Deployment
      name: web
```

Each list entry has:

- **`path:`** — file containing the patch **or**
- **`patch:`** — the patch written inline as a multi-line string (`|-` keeps it clean)
- **`target:`** (optional for SMP, **required** for JSON 6902) — a selector choosing which resource(s) the patch applies to
- **`options:`** (rare) — e.g. `allowNameChange: true`, `allowKindChange: true`

> Older tutorials use `patchesStrategicMerge:` and `patchesJson6902:`. Both are **deprecated**. Use `patches:` for everything. If you see `kustomize edit fix` mentioned, it converts old fields to the new one.

#### `target:` selector fields

| Field | Meaning | Example |
|---|---|---|
| `group` | API group (empty for core) | `apps` |
| `version` | API version | `v1` |
| `kind` | Resource kind | `Deployment` |
| `name` | Resource name — **treated as a regular expression** | `web`, `web-.*` |
| `namespace` | Namespace of the resource | `shop` |
| `labelSelector` | Label query on the resource's own labels | `app=web,tier=frontend` |
| `annotationSelector` | Annotation query | `team=payments` |

All given fields must match (AND). A selector that matches **zero** resources is an error; one that matches **many** applies the patch to **every** match — that is how you patch many resources at once.

Important detail: `target.name` is matched against the resource's name **as it is when the patch is applied**. Kustomize applies `namePrefix` after patches in the usual flow, but name-related behavior differs between versions — if a target "is not found" after adding a prefix/suffix, try the original base name. (See Common Pitfalls.)

### Strategic merge patch (SMP) in depth

A strategic merge patch is a partial resource. Kustomize finds the base resource with the same `apiVersion`, `kind` and `metadata.name`, then **merges** your patch into it using Kubernetes' knowledge of the schema.

#### Merge rules

1. **Scalars and maps you mention are replaced/added.** Fields you do not mention are untouched.
2. **Lists with a merge key are merged element by element**, matching on the key. Elements you do not mention stay.
3. **Lists without a merge key are replaced entirely** by your list.
4. **`null` removes a field.**
5. **Directives (`$patch`) give explicit control** — delete, replace, merge.

Merge keys for common lists (defined in the Kubernetes API types):

| List | Merge key |
|---|---|
| `spec.template.spec.containers` | `name` |
| `spec.template.spec.initContainers` | `name` |
| `containers[].env` | `name` |
| `containers[].volumeMounts` | `mountPath` |
| `containers[].ports` | `containerPort` |
| `spec.template.spec.volumes` | `name` |
| `spec.template.spec.tolerations` | *(none — replaced)* |
| `containers[].command` / `args` | *(none — replaced; plain string list)* |
| `Service.spec.ports` | `port` |
| `metadata.ownerReferences` | `uid` |

> `command` and `args` have **no** merge key, so a patch that specifies `args` replaces the whole list. To add one arg, either repeat the full list in the patch or use JSON 6902 `add` with path `.../args/-`.

#### Example 1 – change replicas

`overlays/prod/replicas-patch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web          # identifies the base resource
spec:
  replicas: 5        # only this field changes
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: replicas-patch.yaml
```

`kubectl kustomize overlays/prod | grep replicas` → `replicas: 5`.

Notice you do **not** repeat the whole Deployment. `apiVersion`, `kind` and `metadata.name` are required — they are the identity — everything else is the delta.

#### Example 2 – add resources to one container (list merged by name)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: nginx            # merge key: matches the existing "nginx" container
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

The `sidecar` container is untouched (not mentioned). The `nginx` container keeps its image, ports and env and gains `resources`. This is the main strength of SMP: list merging by `name`.

#### Example 3 – add and override env vars

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: nginx
          env:
            - name: LOG_LEVEL      # exists in base → value replaced
              value: warn
            - name: REGION         # new → appended
              value: eu-central
```

Result env for `nginx`: `LOG_LEVEL=warn`, `FEATURE_X=off` (kept), `REGION=eu-central`.

#### Example 4 – add readiness and liveness probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: nginx
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
```

Typical use: base has no probes (works anywhere, even a laptop), prod overlay adds them. Probes were taught in lesson 45.

#### Example 5 – add a volume and a volumeMount

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      volumes:
        - name: cache
          emptyDir: {}
      containers:
        - name: nginx
          volumeMounts:
            - name: cache
              mountPath: /var/cache/nginx
```

Both `volumes` (key `name`) and `volumeMounts` (key `mountPath`) merge, so existing entries are preserved.

#### Example 6 – scheduling fields

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      nodeSelector:
        node-role: workers
      tolerations:
        - key: dedicated
          operator: Equal
          value: web
          effect: NoSchedule
```

`tolerations` has no merge key, so if the base already had tolerations this list **replaces** them. Repeat existing entries if you need to keep them.

#### Example 7 – annotations and labels on pod template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  annotations:
    team: platform
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9113"
```

Maps merge key by key, so existing annotations survive.

#### The `$patch` directive

Merging only adds or replaces. To remove things, or to replace a whole structure instead of merging it, use `$patch`:

| Directive | Where | Effect |
|---|---|---|
| `$patch: delete` | Inside a list element or a map | Removes that element/field |
| `$patch: replace` | Inside a map or a list element | Replaces instead of merging |
| `$patch: merge` | (default) | Normal merge |

**Delete a container (list element):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: sidecar
          $patch: delete
```

Removes the `sidecar` container; `nginx` stays.

**Delete one env var:**

```yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          env:
            - name: FEATURE_X
              $patch: delete
```

**Delete a whole resource** from the build (patch the identity, add the directive at the top level):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
$patch: delete
```

Useful when an overlay should not ship something the base includes (for example, no Service for a worker-only variant).

**Replace a list instead of merging it:**

```yaml
spec:
  template:
    spec:
      containers:
        - $patch: replace
        - name: only-one
          image: nginx:1.27-alpine
```

After this, `containers` is exactly `[only-one]`. Use sparingly; it throws away the base's container definitions.

**Remove a field with null:**

```yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          env: null           # drops the whole env list
```

Setting a map/field to `null` deletes it. Handy for removing `resources` or `securityContext`.

#### `setElementOrder` (brief)

Merging lists by key does not guarantee order. If order matters (rare — for example env var `$(VAR)` expansion depends on earlier vars), SMP supports a `$setElementOrder/env` directive listing keys in the desired order. You will almost never need it; know it exists when someone asks "why did my env order change?".

#### SMP and CRDs

Strategic merge relies on struct tags compiled into Kubernetes' built-in types. **Custom resources (CRDs) have no merge keys**, so every list in a CR is replaced as a whole. For CRs (Certificate, ExternalSecret, Gateway, HTTPRoute, Prometheus CRs, …) prefer JSON 6902 when you need to change one list element. Kustomize can be taught CRD schemas via `openapi:` in `kustomization.yaml`, but that is advanced and outside this lesson.

#### Inline SMP patch

No file needed for small edits:

```yaml
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: web
      spec:
        replicas: 3
```

Short and keeps the change next to the transformers. Use files when a patch is more than ~10 lines.

#### SMP with `target:` — patching many resources

When you supply `target:`, the patch is applied to every resource the selector matches, regardless of the name written in the patch. The usual idiom is to keep `kind` and `apiVersion` in the patch and put a placeholder name (the real name comes from the matches):

```yaml
patches:
  - target:
      kind: Deployment
      labelSelector: "tier=backend"
    patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: any
      spec:
        template:
          spec:
            terminationGracePeriodSeconds: 60
```

Every Deployment labeled `tier=backend` gets the change. Test with `kustomize build` and confirm the count of changed resources — a too-broad selector is a classic mistake. (Behavior of the placeholder `name` can differ slightly across Kustomize versions; verify on the version you teach with.)

### JSON 6902 patch in depth

JSON 6902 (RFC 6902) describes a patch as an **ordered list of operations**. It is precise and explicit: there is no merging, so every operation says exactly what to do at an exact path.

Each operation:

```yaml
- op: add | remove | replace | move | copy | test
  path: /json/pointer/to/field
  value: <for add/replace/test>
  from: /source/pointer          # for move/copy
```

A JSON 6902 patch in Kustomize **must** have a `target:` because the patch body carries no resource identity.

#### Path syntax (JSON Pointer)

- Segments separated by `/`, beginning with `/`
- Arrays are addressed by **0-based index**: `/spec/template/spec/containers/0`
- `-` as the last segment means "append to the end of the array": `/spec/template/spec/containers/0/args/-`
- Escape special characters inside a key: `~` becomes `~0`, `/` becomes `~1`

So an annotation key `nginx.ingress.kubernetes.io/rewrite-target` is written `nginx.ingress.kubernetes.io~1rewrite-target`:

```yaml
- op: replace
  path: /metadata/annotations/nginx.ingress.kubernetes.io~1rewrite-target
  value: /$2
```

Same for label keys like `app.kubernetes.io/name` → `app.kubernetes.io~1name`.

#### Operations

| op | Needs | Effect | Fails when |
|---|---|---|---|
| `add` | `path`, `value` | Adds a field, inserts into an array at index, or appends with `-`. If the field exists, it is **replaced** | Parent path missing |
| `replace` | `path`, `value` | Replaces an existing value | Path does not exist |
| `remove` | `path` | Deletes a field or array element | Path does not exist |
| `copy` | `from`, `path` | Copies a value from another path | `from` missing |
| `move` | `from`, `path` | Moves a value (copy + remove) | `from` missing |
| `test` | `path`, `value` | Asserts value equals; **aborts the whole patch** if not | Value differs |

Operations run **in order**; each sees the result of the previous.

#### Example 1 – replace replicas

```yaml
patches:
  - target:
      kind: Deployment
      name: web
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

#### Example 2 – append an arg (SMP cannot do this cleanly)

Assume `nginx` had `args: ["-g", "daemon off;"]`:

```yaml
patches:
  - target:
      kind: Deployment
      name: web
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --verbose
```

Appends without repeating the existing list.

#### Example 3 – remove a specific element

```yaml
- op: remove
  path: /spec/template/spec/containers/1       # removes "sidecar" (index 1)
```

Index-based removal is fragile — if someone reorders containers in the base, you delete the wrong one. Prefer SMP `$patch: delete` when the element has a merge key. Use JSON `remove` for keys/fields or when you add a `test` guard first.

#### Example 4 – `test` as a safety guard

```yaml
- op: test
  path: /spec/template/spec/containers/1/name
  value: sidecar
- op: remove
  path: /spec/template/spec/containers/1
```

If the second container is not `sidecar`, the `test` fails and the whole patch aborts instead of silently removing the wrong container.

#### Example 5 – add a whole new map/array

```yaml
- op: add
  path: /spec/template/spec/nodeSelector
  value:
    node-role: workers
```

`add` to a missing parent key works only if the immediate parent exists (`/spec/template/spec` does).

#### Example 6 – add an env var by appending

```yaml
- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: REGION
    value: eu-central
```

#### Example 7 – copy and move

```yaml
- op: copy
  from: /metadata/labels/app
  path: /metadata/annotations/original-app-label
- op: move
  from: /metadata/annotations/old-key
  path: /metadata/annotations/new-key
```

These two are rarely needed; know that they exist.

#### Example 8 – change an Ingress annotation (key containing `/`)

```yaml
patches:
  - target:
      kind: Ingress
      name: web
    patch: |-
      - op: replace
        path: /metadata/annotations/nginx.ingress.kubernetes.io~1ssl-redirect
        value: "true"
```

#### JSON 6902 as a file

```yaml
patches:
  - path: prod-json-patch.yaml
    target:
      kind: Deployment
      name: web
```

`prod-json-patch.yaml`:

```yaml
- op: replace
  path: /spec/replicas
  value: 5
- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: REGION
    value: eu-central
```

#### JSON 6902 on many resources

```yaml
patches:
  - target:
      kind: Deployment
      labelSelector: "tier=backend"
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/imagePullPolicy
        value: Always
```

Indices (`containers/0`) assume every matched Deployment's first container is the one you mean — document that assumption.

### Choosing between SMP and JSON 6902

| Situation | Use |
|---|---|
| Change scalar (replicas, image tag, port) | Either — SMP is more readable |
| Add resources/probes/env/volumes to a named container | SMP (merge by name) |
| Add one arg/command item | JSON 6902 `add .../-` |
| Remove a container/env var that has a name | SMP `$patch: delete` |
| Remove an unnamed list item or a single field | JSON 6902 `remove` (add a `test` guard) |
| Patch a CRD (Certificate, HTTPRoute, ExternalSecret…) | JSON 6902 |
| Annotation/label with `/` in key | Either (SMP: quote the key; JSON: `~1`) |
| Many resources at once | Either, with `target:` selector |
| Need assertions before change | JSON 6902 `test` |

### Patch order and conflicts

Patches in the list are applied **in the order written**. Later patches see earlier patches' results. Two patches changing the same field → last wins. Patches are applied **after** the base resources are loaded and before most name/label transformers run, so a patch's identity should refer to the **base** name.

### configMapGenerator

Hand-writing a ConfigMap YAML per environment is repetitive. `configMapGenerator` creates ConfigMaps from literals, files or env files at build time, and (the clever part) **appends a hash of the content to the name** so that changing config automatically rolls the Pods.

#### From literals

```yaml
configMapGenerator:
  - name: web-config
    literals:
      - LOG_LEVEL=info
      - FEATURE_X=off
      - API_URL=http://api:8080
```

Output (abbreviated):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config-5f2b8g7k9m
data:
  API_URL: http://api:8080
  FEATURE_X: "off"
  LOG_LEVEL: info
```

#### From files

```yaml
configMapGenerator:
  - name: nginx-conf
    files:
      - nginx.conf                      # key = file name
      - app.properties=prod.properties  # key=file → custom key name
```

The key defaults to the file's base name. `key=path` renames. Files must be inside the kustomization directory (Kustomize forbids `../` outside the root by default — a security rule).

#### From env files

```yaml
configMapGenerator:
  - name: web-env
    envs:
      - app.env
```

`app.env`:

```
LOG_LEVEL=info
FEATURE_X=off
# comments are ignored
```

Each `KEY=VALUE` line becomes a data key. This is the closest to `kubectl create configmap --from-env-file` (lesson 42).

#### Combining sources

One generator can mix `literals`, `files` and `envs`. Keys must be unique across sources.

#### The content-hash suffix

Look at the generated name: `web-config-5f2b8g7k9m`. The suffix is a hash of the ConfigMap's content. Kustomize then **rewrites every reference** to `web-config` in the same build (Deployment `envFrom.configMapRef.name`, `configMapKeyRef`, `volumes[].configMap.name`, …) to the hashed name.

Why it matters:

1. Pods mount/consume the ConfigMap by name; Kubernetes does **not** restart Pods when a ConfigMap changes (env vars never update; volumes update lazily — lesson 42).
2. With the hash, a config change → new hash → new ConfigMap name → the Deployment's pod template changes → **rolling update happens automatically**.
3. The old ConfigMap stays until pruned, so a rollback (`kubectl rollout undo`) still finds its ConfigMap — no broken rollbacks.

Reference it in the Deployment with the **unhashed base name**:

```yaml
containers:
  - name: nginx
    envFrom:
      - configMapRef:
          name: web-config      # Kustomize rewrites to web-config-5f2b8g7k9m
```

Kustomize only rewrites references it understands (built-in kinds and fields). References inside CRDs or in plain strings (an Ingress annotation, an arg like `--config=web-config`) are **not** rewritten. See pitfalls.

#### disableNameSuffixHash

```yaml
generatorOptions:
  disableNameSuffixHash: true
configMapGenerator:
  - name: web-config
    literals:
      - LOG_LEVEL=info
```

or per generator:

```yaml
configMapGenerator:
  - name: web-config
    literals:
      - LOG_LEVEL=info
    options:
      disableNameSuffixHash: true
```

Result: ConfigMap is named plain `web-config`. Use it when:

- Something outside Kustomize refers to the name (a CRD field, an operator, another team's manifest, `helm` values)
- You deliberately manage restarts yourself (`kubectl rollout restart`)

Cost: **no automatic rollout** on config change, and ConfigMap updates are applied in place.

#### generatorOptions (global) and per-generator options

| Option | Meaning |
|---|---|
| `disableNameSuffixHash` | Don't append the hash |
| `labels` | Extra labels on generated ConfigMaps/Secrets |
| `annotations` | Extra annotations |
| `immutable` | Sets `immutable: true` on generated objects (Kubernetes then rejects edits; pairs well with hash suffixes) |

```yaml
generatorOptions:
  labels:
    app.kubernetes.io/part-of: shop
  annotations:
    owner: platform-team
  immutable: true
```

Per-generator `options:` overrides `generatorOptions` for that generator.

#### behavior: create, replace, merge

Typical pattern: base defines `web-config` with defaults; overlay changes some values. Generators in an overlay use `behavior`:

| `behavior` | Effect |
|---|---|
| `create` | Default. Create a new ConfigMap; error if one with that name exists |
| `replace` | Replace the base ConfigMap's data entirely |
| `merge` | Merge keys into the base ConfigMap (overlay keys win) |

`base/kustomization.yaml`:

```yaml
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: web-config
    literals:
      - LOG_LEVEL=info
      - FEATURE_X=off
```

`overlays/prod/kustomization.yaml`:

```yaml
resources:
  - ../../base
configMapGenerator:
  - name: web-config            # SAME name as base
    behavior: merge
    literals:
      - LOG_LEVEL=warn          # overrides
      - REPLICA_HINT=5          # added
```

Result: `LOG_LEVEL=warn`, `FEATURE_X=off`, `REPLICA_HINT=5`. With `behavior: replace` the result would contain only `LOG_LEVEL` and `REPLICA_HINT`.

> Using `behavior: create` (or omitting it) in an overlay with the same name as a base generator fails with an "already exists / ID conflict" error. Say `merge` or `replace` explicitly.

#### Generated ConfigMaps and patches

You can patch a generated ConfigMap like any other resource (target by name `web-config`), but prefer `behavior: merge` — it keeps the hash consistent.

### secretGenerator

Same mechanics as `configMapGenerator`, producing `Secret` objects:

```yaml
secretGenerator:
  - name: db-credentials
    literals:
      - username=app
      - password=s3cr3t
  - name: tls-cert
    type: kubernetes.io/tls
    files:
      - tls.crt
      - tls.key
  - name: regcred
    type: kubernetes.io/dockerconfigjson
    files:
      - .dockerconfigjson=config.json
```

- `type:` defaults to `Opaque`. Others: `kubernetes.io/tls` (needs `tls.crt` + `tls.key`), `kubernetes.io/dockerconfigjson` (key `.dockerconfigjson`), `kubernetes.io/basic-auth`, `kubernetes.io/ssh-auth`
- Literal values are **base64-encoded automatically** in the output `data:` — you write plain text in the generator
- Hash suffix and `behavior` work exactly like ConfigMaps; `envs:` works too
- `options.disableNameSuffixHash` works the same

Output:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-7h2kd9t5c8
type: Opaque
data:
  password: czNjcjN0
  username: YXBw
```

#### Security warning — read this twice

`secretGenerator` is a **convenience, not a security feature**.

- A `kustomization.yaml` with `literals: [password=s3cr3t]` puts the secret in **plain text in Git**
- A `files: [tls.key]` source file committed to Git is plain text in Git
- The generated Secret is only base64 — anyone with `kubectl get secret -o yaml` decodes it (lesson 42)

Acceptable patterns:

1. Secret source files are generated in CI from a secret store and **not committed** (`.gitignore` them)
2. Secret values come from the environment at build time (`secretGenerator` with `envs:` pointing to a file created just before `kustomize build`)
3. Use a real secret manager and keep Secrets out of Kustomize entirely — **External Secrets Operator (lesson 43)** pulls from Vault and creates the Secret; Kustomize only applies the `ExternalSecret` object

For this course: use `secretGenerator` for lab/demo credentials only; production uses ESO.

### Kustomize vs Helm

| Aspect | Kustomize | Helm |
|---|---|---|
| Model | Patch/overlay on plain YAML | Go-template engine + values |
| Placeholders in YAML | None — base always valid | `{{ .Values.x }}` everywhere |
| Learning curve | Low (YAML + a few fields) | Medium (Go templates, sprig, helpers) |
| Built into kubectl | Yes (`kubectl apply -k`) | No (separate `helm` binary) |
| Packaging/versioning | Git tags/dirs; no package format | Charts, versions, repos, OCI |
| Release tracking | None (stateless — kubectl apply) | Releases, history, rollback (`helm rollback`) |
| Dependencies | Manual (`resources:` pointing to remote bases) | `dependencies:` in `Chart.yaml` |
| Third-party apps | Hard (patch someone else's YAML) | Easy (install community charts, set values) |
| Own apps with few envs | Excellent | Overkill sometimes |
| Conditionals/loops | None — structure must come from files | Full (`if`, `range`) |
| Hooks/tests | No | Yes |
| Debug | `kustomize build` shows final YAML | `helm template` shows final YAML |
| Secrets | `secretGenerator` (weak) | Values files (also weak) |

Practical guidance:

- **Your own services, a few environments** → Kustomize: no template language, easy diffs
- **Third-party software, many knobs, versioned distribution** → Helm chart
- **Both** → very common in practice (below)

### Using Kustomize and Helm together

Three common patterns.

**1. Render Helm, customize with Kustomize**

```bash
helm template my-release ingress-nginx/ingress-nginx \
  --namespace ingress-nginx -f values.yaml > base/ingress-nginx.yaml
```

`base/kustomization.yaml`:

```yaml
resources:
  - ingress-nginx.yaml
```

Overlays then patch what the chart's values could not (add an annotation, change a probe). Apply: `kubectl apply -k overlays/prod`.

**2. Pipe straight into kubectl**

```bash
helm template my-release ./chart -f values-prod.yaml | kubectl apply -f -
```

and for a Kustomize layer after Helm, use pattern 1 or 3.

**3. Helm post-renderer**

`helm install --post-renderer ./kustomize-post.sh` lets Helm hand the rendered YAML to a script before applying:

`kustomize-post.sh`:

```bash
#!/bin/sh
cat > all.yaml            # Helm pipes rendered manifests on stdin
kustomize build .         # kustomization.yaml in this dir lists all.yaml + patches
```

`kustomization.yaml` (same directory):

```yaml
resources:
  - all.yaml
patches:
  - path: add-annotation.yaml
```

```bash
chmod +x kustomize-post.sh
helm install my-release ./chart -f values.yaml --post-renderer ./kustomize-post.sh
```

You keep Helm's release tracking/rollback **and** Kustomize patches.

**4. `helmCharts:` field (built-in inflator)**

```yaml
helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.11.2
    releaseName: ingress
    namespace: ingress-nginx
    valuesInline:
      controller:
        replicaCount: 2
```

Needs the flag: `kustomize build --enable-helm .` (or `kubectl kustomize --enable-helm`). Kustomize runs `helm template` itself, then applies your patches. Requires the `helm` binary on PATH. Note: `kubectl apply -k` does **not** enable Helm inflation; build explicitly and pipe to `kubectl apply -f -`.

One-line mentions only (not covered today): `components:` (reusable optional feature bundles) and `replacements:` (copy a value from one resource field to another) — both exist when patches are not enough.

### Debugging patches

1. **Always render before applying**: `kubectl kustomize overlays/prod` or `kustomize build overlays/prod`
2. **Diff against the cluster**: `kubectl diff -k overlays/prod` (exit code 1 = differences, 0 = none)
3. **Diff two overlays**: `diff <(kustomize build overlays/dev) <(kustomize build overlays/prod)`
4. **Isolate a patch**: comment out others and rebuild
5. **Filter**: `kustomize build overlays/prod | grep -A5 'kind: Deployment'`
6. **Count matches** for selectors: `kustomize build . | grep -c 'kind: Deployment'`
7. **Check the version**: `kubectl version` shows an embedded Kustomize version, often **older** than the standalone `kustomize version` — features (e.g. `--enable-helm`, newer `patches` behaviors) can differ

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kustomize build DIR` | Render final YAML to stdout | `kustomize build overlays/prod` |
| `kubectl kustomize DIR` | Same, using kubectl's embedded Kustomize | `kubectl kustomize overlays/prod` |
| `kubectl apply -k DIR` | Render and apply | `kubectl apply -k overlays/prod` |
| `kubectl diff -k DIR` | Show diff vs live cluster | `kubectl diff -k overlays/prod` |
| `kubectl delete -k DIR` | Delete everything the kustomization renders | `kubectl delete -k overlays/dev` |
| `kustomize build --enable-helm DIR` | Allow the `helmCharts:` field | `kustomize build --enable-helm .` |
| `kustomize edit add patch --path P --kind K --name N` | Add a `patches:` entry from CLI | `kustomize edit add patch --path replicas.yaml --kind Deployment --name web` |
| `kustomize edit add configmap NAME --from-literal=K=V` | Add a configMapGenerator entry | `kustomize edit add configmap web-config --from-literal=LOG_LEVEL=info` |
| `kustomize edit fix` | Migrate deprecated fields to current syntax | `kustomize edit fix` |
| `kustomize version` | Show standalone version | `kustomize version` |
| `helm template NAME CHART -f F` | Render a chart for Kustomize base | `helm template web ./chart -f values.yaml > base/all.yaml` |
| `helm install --post-renderer SCRIPT` | Post-process Helm output | `helm install web ./chart --post-renderer ./kustomize-post.sh` |

`kustomization.yaml` fields from this lesson:

| Field | Purpose |
|---|---|
| `patches:` | List of `{path|patch, target, options}` entries |
| `configMapGenerator:` | `name`, `literals`, `files`, `envs`, `behavior`, `options` |
| `secretGenerator:` | Same plus `type` |
| `generatorOptions:` | `disableNameSuffixHash`, `labels`, `annotations`, `immutable` |
| `helmCharts:` | Inflate a Helm chart (needs `--enable-helm`) |

JSON 6902 operations: `add`, `remove`, `replace`, `move`, `copy`, `test`. Path escapes: `~0` = `~`, `~1` = `/`, `-` = array end.

## Examples / Walkthrough

### Example A – full prod overlay with both patch types

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: web-prod
resources:
  - ../../base

patches:
  # SMP from a file: resources + probes
  - path: deployment-hardening.yaml

  # SMP inline: replicas
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: web
      spec:
        replicas: 4

  # JSON 6902 inline: append an arg and drop the sidecar safely
  - target:
      kind: Deployment
      name: web
    patch: |-
      - op: test
        path: /spec/template/spec/containers/1/name
        value: sidecar
      - op: remove
        path: /spec/template/spec/containers/1
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: REGION
          value: eu-central
```

`overlays/prod/deployment-hardening.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: nginx
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 256Mi }
          readinessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 10
            periodSeconds: 10
```

Walkthrough:

```bash
cd kustomize-lab
kubectl kustomize overlays/prod | less
# verify: replicas 4, one container (nginx) with resources+probes+REGION env, namespace web-prod
kubectl create namespace web-prod
kubectl apply -k overlays/prod
kubectl -n web-prod get deploy,pods
```

### Example B – generator with automatic rollout

Base uses a generator and consumes it with `envFrom`:

`base/kustomization.yaml`:

```yaml
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: web-config
    literals:
      - LOG_LEVEL=info
      - FEATURE_X=off
```

Add to `base/deployment.yaml` container `nginx`:

```yaml
          envFrom:
            - configMapRef:
                name: web-config
```

```bash
kubectl create namespace lab
kubectl apply -k base -n lab
kubectl -n lab get cm                         # web-config-<hash>
kubectl -n lab get pods

# change config → new hash → rolling update
sed -i.bak 's/LOG_LEVEL=info/LOG_LEVEL=debug/' base/kustomization.yaml
kubectl diff -k base -n lab                   # shows new ConfigMap name AND Deployment envFrom name change
kubectl apply -k base -n lab
kubectl -n lab rollout status deploy/web
kubectl -n lab get cm                         # TWO ConfigMaps now (old kept, new created)
kubectl -n lab rollout undo deploy/web        # rollback still works; old ConfigMap exists
```

### Example C – override with behavior merge

`overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
configMapGenerator:
  - name: web-config
    behavior: merge
    literals:
      - LOG_LEVEL=debug
      - FEATURE_X=on
```

```bash
kustomize build overlays/dev | grep -A8 'kind: ConfigMap'
```

### Example D – ConfigMap from a file mounted as a config file

`base/nginx.conf`:

```
server {
  listen 80;
  location /healthz { return 200 "ok\n"; }
  location / { root /usr/share/nginx/html; }
}
```

`base/kustomization.yaml` adds:

```yaml
configMapGenerator:
  - name: nginx-conf
    files:
      - nginx.conf=default.conf
```

Patch the Deployment (SMP) to mount it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      volumes:
        - name: conf
          configMap:
            name: nginx-conf          # rewritten to the hashed name
      containers:
        - name: nginx
          volumeMounts:
            - name: conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
```

Editing `nginx.conf` changes the hash and triggers a rollout; no manual `rollout restart` needed.

### Example E – secretGenerator for a lab credential

```yaml
secretGenerator:
  - name: db-credentials
    literals:
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=labonly
    options:
      disableNameSuffixHash: false
```

```bash
kustomize build . | grep -B2 -A6 'kind: Secret'
echo cGFzc3dvcmQ= | base64 -d
```

Show students: the plain value sits in `kustomization.yaml` — then discuss the ESO alternative.

### Example F – Helm output + Kustomize patch

```bash
mkdir -p helm-kust/base helm-kust/overlays/prod
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm template podinfo podinfo/podinfo --set replicaCount=1 > helm-kust/base/podinfo.yaml
cat > helm-kust/base/kustomization.yaml <<'EOF'
resources:
  - podinfo.yaml
EOF
cat > helm-kust/overlays/prod/kustomization.yaml <<'EOF'
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: podinfo
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: add
        path: /spec/template/metadata/annotations/team
        value: platform
EOF
kubectl kustomize helm-kust/overlays/prod | grep -E 'replicas|team'
```

### Example G – patch many resources by label

```yaml
patches:
  - target:
      kind: Deployment
      labelSelector: "app in (web,api)"
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/imagePullPolicy
        value: Always
```

Verify with `kustomize build . | grep -c imagePullPolicy` equals the number of matched Deployments.

### Example H – delete a resource in an overlay

`overlays/worker-only/delete-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
$patch: delete
```

```yaml
patches:
  - path: delete-service.yaml
```

`kustomize build overlays/worker-only | grep 'kind: Service'` → no output.

## Common Pitfalls

- **Patch has no effect** — wrong `metadata.name` or `kind`/`apiVersion` in an SMP means it matches nothing. Note: a patch with a nonexistent identity often **errors**, but with `target:` mismatches or merge-key typos (`name: ngnix`) a new list element is *added* instead of merged. Always read the rendered output.
- **Error "no matches for Id …" / "failed to find unique target for patch …"** — the `target` or the patch's identity does not match any resource in the current build. Check kind spelling, group/version, and whether the resource is actually included in `resources:`. If you added `namePrefix`/`nameSuffix`, try the pre-prefix base name; selectors by `name` are regex, so `web` also matches `web-api`.
- **JSON 6902 without `target`** — the patch cannot identify its resource. Always add `target:`.
- **JSON 6902 index drift** — `/containers/1` silently points to a different container after someone reorders the base. Add a `test` operation or switch to SMP with the container `name`.
- **`replace` on a missing path fails** — use `add` when the field may not exist (`add` creates/overwrites).
- **`add` to a missing parent fails** — `add /spec/template/spec/nodeSelector/disk` fails when `nodeSelector` does not exist; add the whole map first.
- **Forgetting `~1`** — annotation/label keys containing `/` need `~1` in JSON paths; without it you get "path not found" errors.
- **SMP replaces lists without merge keys** — `tolerations`, `args`, `command` are replaced wholesale. Repeat existing items or use JSON `add .../-`.
- **SMP on CRDs** — lists in custom resources are replaced, not merged. Use JSON 6902.
- **Using deprecated `patchesStrategicMerge` / `patchesJson6902` / `bases`** — still work in many versions but warn; run `kustomize edit fix`.
- **`behavior` omitted in overlay generator** — same name as the base generator with default `create` fails with an ID conflict. Use `merge` or `replace`.
- **Hash suffix breaks external references** — an Ingress annotation, an operator CR, or `--config=web-config` arg refers to the plain name, which no longer exists (it is `web-config-5f2b8g7k9m`). Kustomize only rewrites references in known kinds/fields. Fix: `disableNameSuffixHash: true` for that generator, or reference via a field Kustomize understands.
- **Old ConfigMaps pile up** — each content change creates a new ConfigMap and the old one stays. `kubectl apply -k` does not prune by default. Use `kubectl apply -k ... --prune -l <label>` with care, or let a GitOps tool prune, or accept the clutter.
- **`immutable: true` plus no hash** — editing the data of an immutable ConfigMap fails ("field is immutable"). Combine `immutable` with the hash suffix so each change creates a new object.
- **"field is immutable" on apply** — patching `spec.selector` of a Deployment (or `commonLabels`/labels with selectors enabled in the overlay) fails on an existing Deployment because selectors cannot change. Delete and recreate, or avoid changing selector labels between environments.
- **Secrets in Git** — `secretGenerator` literals are plaintext in the repo. Treat as demo-only; use ESO (lesson 43) for real secrets.
- **File outside the kustomization root** — `files: [../../shared/app.conf]` is rejected by the load restrictor. Put files inside the directory (or reference a base via `resources:`).
- **`kubectl kustomize` vs `kustomize` version skew** — a feature works with the standalone binary but not with `kubectl apply -k`. Compare `kubectl version` and `kustomize version`; use `kustomize build | kubectl apply -f -` to use the newer binary.
- **`helmCharts:` ignored by `kubectl apply -k`** — Helm inflation needs `--enable-helm`, supported by `kustomize build` (and `kubectl kustomize` in newer versions), not by `kubectl apply -k`.
- **Over-patching** — fifteen patches per overlay is a sign the base is wrong. Move shared decisions into the base; keep overlays small.

## FAQ

**Q: What is the difference between a transformer and a patch?**
A: A transformer (`namePrefix`, `namespace`, `images`, `labels`) applies one well-defined change to all resources. A patch changes arbitrary fields of specific resources. Use transformers for cross-cutting changes, patches for per-resource edits.

**Q: Strategic merge or JSON 6902 — which should I default to?**
A: Strategic merge. It looks like the manifest it modifies and merges lists by key. Use JSON 6902 for appending one list item, removing one unnamed element, patching CRDs, annotation keys with special characters, or when you want a `test` guard.

**Q: Why does my SMP patch wipe my tolerations / args?**
A: Lists without a merge key (tolerations, command, args) are replaced entirely. Either repeat all items in the patch or switch to JSON 6902 `add` with path ending in `/-`.

**Q: How do I delete an env var or container from the base in an overlay?**
A: SMP: add the element by its key with `$patch: delete`. JSON 6902: `op: remove` with the index (add a `test` operation first so you do not remove the wrong element).

**Q: How do I delete a whole resource from an overlay?**
A: Create a patch with the resource's `apiVersion`, `kind`, `metadata.name` and a top-level `$patch: delete`.

**Q: Why does JSON 6902 need a target but strategic merge does not?**
A: An SMP patch is itself a partial resource, so its `apiVersion/kind/name` identify the target. A JSON 6902 patch is just a list of operations with no identity, so Kustomize needs `target:`. (You may also use `target:` with SMP to apply one patch to many resources.)

**Q: What does `~1` mean in a path?**
A: It is the JSON Pointer escape for `/` inside a key. `nginx.ingress.kubernetes.io/rewrite-target` becomes `nginx.ingress.kubernetes.io~1rewrite-target`. `~0` escapes `~`.

**Q: What does `-` at the end of a path do?**
A: With `op: add` it appends to the end of the array: `/spec/template/spec/containers/0/args/-`.

**Q: Is `target.name` an exact match?**
A: It is a regular expression, so `web` also matches `web-api`. Anchor it (`^web$`) when names overlap.

**Q: Can I patch the same resource twice?**
A: Yes. Patches apply in the order listed; later ones see the results of earlier ones.

**Q: What is the hash suffix on generated ConfigMaps and why?**
A: Kustomize appends a hash of the content to the generated name and rewrites references in the Pod template. Changing config changes the name, which changes the Pod template, which triggers a rolling update. Old ConfigMaps remain, so rollbacks work.

**Q: Can I turn the hash off?**
A: Yes — `generatorOptions.disableNameSuffixHash: true` (global) or `options.disableNameSuffixHash: true` per generator. You lose the automatic rollout, so restart manually (`kubectl rollout restart deploy/web`) after config changes.

**Q: When should I disable the hash?**
A: When something Kustomize does not understand references the name (CRDs, operators, annotations), or when you deliberately control restarts yourself.

**Q: How do I override a base's generated ConfigMap in an overlay?**
A: Declare a generator with the **same name** and `behavior: merge` (add/override keys) or `behavior: replace` (replace all data).

**Q: What is the difference between `literals`, `files` and `envs`?**
A: `literals` = inline `KEY=VALUE`. `files` = each file becomes one key (key is the filename unless you write `key=path`). `envs` = a dotenv file where each line becomes a key.

**Q: Is `secretGenerator` secure?**
A: No. Plaintext values live in your Git repo and the Secret is only base64 in the cluster. Use it for demos or with CI-generated, non-committed files. For production use External Secrets Operator (lesson 43) or another secret manager.

**Q: Can I generate a TLS secret or an image pull secret?**
A: Yes: `type: kubernetes.io/tls` with `tls.crt`/`tls.key` files, or `type: kubernetes.io/dockerconfigjson` with a `.dockerconfigjson` file.

**Q: Kustomize or Helm?**
A: Kustomize for your own apps with a handful of environments and a desire for plain YAML; Helm for third-party software, heavy parameterization, and versioned distribution. Many teams use both.

**Q: How do I use Kustomize on top of a Helm chart?**
A: Render with `helm template`, put the output in a base and patch it; or use `helm install --post-renderer` with a script that runs `kustomize build`; or use the `helmCharts:` field with `kustomize build --enable-helm`.

**Q: Does Kustomize track releases or support rollback like Helm?**
A: No. It is stateless — it renders YAML; `kubectl apply` applies it. Rollback is `git revert` + re-apply, or `kubectl rollout undo` for Deployments.

**Q: How do I preview what will change in the cluster?**
A: `kubectl diff -k overlays/prod`. Exit code 1 means there are differences.

**Q: How do I see exactly what Kustomize generated?**
A: `kustomize build DIR` or `kubectl kustomize DIR`. Always run this before `apply`.

**Q: Why does the rendered YAML have sorted keys / different ordering?**
A: Kustomize re-serializes output with its own ordering (resources ordered by kind, keys sorted). Content is equivalent.

**Q: Why does `kubectl apply -k` behave differently from `kustomize build | kubectl apply -f -`?**
A: Different Kustomize versions (embedded in kubectl vs standalone). Check `kubectl version` and `kustomize version`.

**Q: Can a patch change the name of a resource?**
A: Not by default (`allowNameChange: true` in the patch `options:` allows it). Use `namePrefix`/`nameSuffix` for renames across all resources.

## Practice / Exercise

Lab setup: a cluster (kind/minikube/lab cluster) with `kubectl` ≥ 1.27 and optionally standalone `kustomize`. Use the `kustomize-lab/` tree from the Concepts section.

### Core

1. **Replicas patch (SMP).** Create `overlays/prod/replicas-patch.yaml` setting 5 replicas. Build and confirm only `spec.replicas` differs from the base (`diff <(kustomize build base) <(kustomize build overlays/prod)`).
2. **Resources + probes (SMP).** Add resources and readiness/liveness probes to the `nginx` container in prod only. Confirm the `sidecar` container is unchanged.
3. **Env var edits (SMP).** In prod, change `LOG_LEVEL` to `warn`, add `REGION`, and delete `FEATURE_X` with `$patch: delete`. Verify the final env list.
4. **JSON 6902 replicas + env append.** Redo exercises 1 and 3 (replace replicas, append `REGION`) using JSON 6902 with a `target:`. Compare the rendered output with the SMP version — it should be equivalent.
5. **Annotation with `/`.** Add an Ingress (any minimal Ingress YAML) to the base with annotation `nginx.ingress.kubernetes.io/rewrite-target: /`. In prod, change it to `/$2` with JSON 6902 using `~1` escaping.
6. **configMapGenerator.** Add a generator `web-config` with two literals, consume it via `envFrom` in the Deployment. Apply, note the hashed name, change one literal, re-apply, and watch the rolling update. Run `kubectl get cm` and explain why two ConfigMaps exist.
7. **behavior merge.** In the dev overlay, `merge` a new key and override an existing one. Confirm the final ConfigMap contains base-only keys, overridden keys, and new keys.
8. **Disable the hash.** Set `disableNameSuffixHash: true`, re-apply, and check what changed in the ConfigMap name. Edit the literal again and verify that **no** rollout happens; trigger one manually with `kubectl rollout restart`.
9. **Debug.** Introduce a typo in a patch's `metadata.name` and a `target.name` that matches nothing. Record each error message and how you diagnosed it with `kustomize build`.

### Stretch

1. **Safe removal.** Remove the `sidecar` container with JSON 6902 guarded by a `test` operation. Change the base order of containers, rebuild, and observe the `test` failing instead of removing the wrong container.
2. **Patch many.** Add a second Deployment `api` to the base labeled `tier=backend` (and label `web` too). Use one `labelSelector` target to add `imagePullPolicy: Always` to both. Prove it with `grep -c`.
3. **Delete a resource.** Build an overlay `worker-only` that removes the Service via `$patch: delete`.
4. **Secret generator.** Generate an Opaque secret and a `kubernetes.io/tls` secret (create a self-signed pair with `openssl req -x509 -nodes -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=web.local" -days 30`). Decode the Opaque secret's data with `base64 -d`. Write three sentences on why this is not safe for production and what lesson 43 offers instead.
5. **Hash vs external reference.** Add an arg `--config=web-config` to the container. Observe that Kustomize does **not** rewrite it. Fix it two ways: disabling the hash, and mounting the ConfigMap as a volume (which Kustomize does rewrite).
6. **Helm + Kustomize.** Render the `podinfo` chart with `helm template`, patch replicas and add an annotation in an overlay, and apply. Then redo the same with `helm install --post-renderer`.
7. **`helmCharts:`.** Use the `helmCharts:` field with `kustomize build --enable-helm` and explain why `kubectl apply -k` alone does not work for it.
8. **CRD patch.** Apply any CRD-backed object available in your cluster (for example a `Certificate` from cert-manager, lesson 38) in a base and patch one entry of a list with JSON 6902. Explain why strategic merge would have replaced the list.

## Further Reading

- Kustomize documentation (patches, generators): https://kubectl.docs.kubernetes.io/references/kustomize/
- `patches` reference: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/
- `configMapGenerator` and `secretGenerator` reference: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/configmapgenerator/
- Kubernetes docs – Declarative management using Kustomize: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
- Strategic merge patch (Kubernetes): https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/
- JSON Patch RFC 6902: https://datatracker.ietf.org/doc/html/rfc6902
- JSON Pointer RFC 6901 (`~0`, `~1`): https://datatracker.ietf.org/doc/html/rfc6901
- Helm post-renderers: https://helm.sh/docs/topics/advanced/#post-rendering
- Kustomize source and examples: https://github.com/kubernetes-sigs/kustomize

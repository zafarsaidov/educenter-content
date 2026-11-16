# Lesson 55: Practice – Deploy Complex App with Helm

**Module:** Helm – Kubernetes Package Manager
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-52 (Kubernetes workloads, Services, Ingress, PVC/StatefulSet, ConfigMap/Secret); Lesson 53 (Helm basics: charts, releases, repos, install/upgrade/rollback); Lesson 54 (Go templating, `.Values`, `.Release`, `_helpers.tpl`, `include`, `toYaml | nindent`, `if`/`with`/`range`). Student has a working cluster (kind/minikube/kubeadm/rke2), `kubectl`, and `helm` v3.

## Learning Objectives

By end of lesson student can:
- Design a multi-tier Helm chart (frontend + backend + database) from scratch and decide which fields become values
- Write `Chart.yaml`, `values.yaml`, per-environment override files (`values-dev.yaml`, `values-prod.yaml`)
- Build `_helpers.tpl` with named templates for names, labels, and selector labels, and call them with `include`
- Toggle optional resources (Ingress, persistence) with `if`, and render optional blocks with `with` + `toYaml | nindent`
- Use a checksum annotation so Pods roll automatically when a ConfigMap changes
- Validate a chart before it touches the cluster: `helm lint`, `helm template`, `helm install --dry-run --debug`
- Package a chart with `helm package` (`--version`, `--app-version`) and install from the `.tgz`
- Install, upgrade with changed values, inspect history, roll back, and uninstall a release
- Explain values precedence (`values.yaml` < `-f` files left to right < `--set`) and predict the final rendered value
- Diagnose the most common chart failures (YAML indentation, missing `required` value, selector mismatch, immutable field on upgrade)

## Topics

- Write a Helm chart from scratch for a multi-tier app (frontend + backend + db)
- Define values.yaml with environment overrides; use --set and -f for customization
- Use _helpers.tpl for consistent labels, names, selectors
- Package (helm package), install, upgrade with changed values
- Debug: helm template (local render), helm lint, helm install --dry-run

## Concepts

### What we are building

A three-tier "shop" application, delivered as **one chart**, **one release**:

```
                    (optional Ingress  shop.dev.example.local)
                                   |
                                   v
                      +-----------------------+
                      |  frontend Deployment  |   nginx, serves static page,
                      |  Service :80          |   proxies /api -> backend
                      +-----------+-----------+
                                  |  /api
                                  v
                      +-----------------------+
                      |  backend Deployment   |   small HTTP API
                      |  Service :8080        |   reads DB_HOST, DB_USER, DB_PASSWORD
                      +-----------+-----------+
                                  |  tcp 5432
                                  v
                      +-----------------------+
                      |  db StatefulSet       |   PostgreSQL 16
                      |  headless Service     |   PVC via volumeClaimTemplates
                      |  Secret (password)    |
                      +-----------------------+
```

Why one chart for three tiers: they are released, versioned and rolled back together. In real organizations the database is often a separate release or a managed service; we model it inside the chart so students see every pattern in one place, and Stretch exercise 1 turns it into an optional dependency.

**Stand-in images.** The lab must run on any cluster without building anything, so:
- frontend: `nginx:1.27-alpine`
- backend: `traefik/whoami:v1.10` (tiny HTTP server that echoes request info; it ignores the DB variables, but we still wire them correctly). Replace with the image you built in lesson 40 when you want a real app.
- db: `postgres:16-alpine`

### Chart design: decide what is a value

A good chart exposes **what differs between environments or users**, and hardcodes what never changes.

| Make it a value | Keep hardcoded in template |
|---|---|
| image repository/tag, pullPolicy | container port names (`http`, `postgres`) |
| replica counts | label key names (`app.kubernetes.io/*`) |
| resources requests/limits | probe paths/ports that belong to the image |
| Service type and port | `apiVersion`, `kind` |
| Ingress enable/host/class/annotations | internal wiring like `DB_HOST` value derivation |
| storage size, storageClass, persistence on/off | |
| credentials (password) | |

Rule of thumb: if you would otherwise copy the chart to change one line, that line is a value. If a value is never changed by anyone, it is noise.

### Values layering (precedence)

Helm merges values from lowest to highest priority:

1. chart's own `values.yaml` (defaults)
2. parent chart values (if this chart is a subchart; see Stretch 1)
3. each `-f file.yaml` in command-line order (later file wins)
4. `--set`, `--set-string`, `--set-file` (highest, later `--set` wins)

Maps merge key by key; **lists are replaced whole, not merged**. That is why `ingress.hosts` as a list is awkward to override from `--set`; our chart uses scalars (`ingress.host`) on purpose.

```bash
helm install shop ./shop -f values-dev.yaml --set frontend.replicaCount=3
#   values.yaml  <  values-dev.yaml  <  --set
```

### Naming and labels with `_helpers.tpl`

Every object needs a name that is unique per release and labels that are consistent. Hand-typing them in 12 templates guarantees drift. Instead define **named templates** once:

| Helper | Returns | Used for |
|---|---|---|
| `shop.name` | chart name (overridable) | `app.kubernetes.io/name` |
| `shop.fullname` | `<release>-<chart>` trimmed to 63 chars | object `metadata.name` prefix |
| `shop.chart` | `shop-0.1.0` | `helm.sh/chart` label |
| `shop.selectorLabels` | **immutable** identity labels | `spec.selector.matchLabels` + Pod template labels |
| `shop.labels` | selector labels + chart/version/managed-by | `metadata.labels` on every object |

Two critical rules:
- **Selector labels must never change** after first install: `Deployment.spec.selector` is immutable. Never put version or chart-version in selectorLabels.
- 63-character limit: Kubernetes names (DNS labels) max 63 characters, so helpers `trunc 63 | trimSuffix "-"`.

Because we have three components, helpers take a **dict** `(dict "ctx" . "component" "frontend")`, since `include` accepts only one argument. This is the standard multi-component pattern.

### Rolling Pods on config change: checksum annotation

Changing a ConfigMap does **not** restart Pods that consume it as a file or environment variable. Helm's idiom: put a hash of the rendered ConfigMap in the Pod template annotations. Config changes, hash changes, Pod template changes, Deployment rolls.

```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap-frontend.yaml") . | sha256sum }}
```

### Release lifecycle (recap for this lab)

```
helm lint        -> static checks on chart
helm template    -> render locally, no cluster
helm install --dry-run --debug  -> render with release context, server-side validation, no creation
helm package     -> build shop-0.1.0.tgz
helm install     -> revision 1
helm upgrade     -> revision 2, 3, ...
helm history     -> list revisions
helm rollback    -> new revision with old content
helm uninstall   -> remove objects (PVCs created by StatefulSet templates survive!)
```

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `helm create NAME` | scaffold a chart | `helm create shop` |
| `helm lint PATH` | static analysis (syntax, missing fields, best practices) | `helm lint ./shop -f values-dev.yaml` |
| `helm template REL PATH` | render locally, print YAML, no cluster access | `helm template shop ./shop -f values-dev.yaml` |
| `helm template ... -s FILE` | render only one template file | `helm template shop ./shop -s templates/ingress.yaml --set ingress.enabled=true` |
| `helm install --dry-run --debug` | full render + server validation without creating | `helm install shop ./shop --dry-run --debug -f values-dev.yaml` |
| `helm install REL PATH -n NS --create-namespace` | install a release | `helm install shop ./shop -n shop-dev --create-namespace -f values-dev.yaml` |
| `helm upgrade REL PATH` | new revision | `helm upgrade shop ./shop -n shop-dev -f values-dev.yaml --set backend.replicaCount=3` |
| `helm upgrade --install` | install if missing, else upgrade (idempotent) | `helm upgrade --install shop ./shop -n shop-dev -f values-dev.yaml` |
| `helm upgrade --reuse-values` | start from last release values, then apply new `--set` | `helm upgrade shop ./shop --reuse-values --set frontend.image.tag=1.27.1-alpine` |
| `helm get values REL` | user-supplied values of live release | `helm get values shop -n shop-dev` |
| `helm get values REL -a` | all computed values (defaults included) | `helm get values shop -a` |
| `helm get manifest REL` | exact YAML that was applied | `helm get manifest shop -n shop-dev` |
| `helm history REL` | revisions | `helm history shop -n shop-dev` |
| `helm rollback REL REV` | roll back to revision | `helm rollback shop 1 -n shop-dev` |
| `helm status REL` | status + rendered NOTES.txt | `helm status shop -n shop-dev` |
| `helm package PATH` | build `.tgz` | `helm package ./shop --version 0.1.1 --app-version 1.0.0` |
| `helm show values PATH\|TGZ` | print default values of a chart | `helm show values ./shop-0.1.0.tgz` |
| `helm uninstall REL` | delete release | `helm uninstall shop -n shop-dev` |
| `helm list -A` | releases in all namespaces | `helm list -A` |
| `helm dependency update PATH` | download subcharts into `charts/` | `helm dependency update ./shop` |

`--set` syntax cheat sheet:

| Goal | Syntax |
|---|---|
| nested key | `--set db.persistence.size=5Gi` |
| string that looks like number/bool | `--set-string backend.image.tag=1.10` |
| list item | `--set 'ingress.annotations.nginx\.ingress\.kubernetes\.io/ssl-redirect=false'` (escape dots in keys) |
| value with comma | `--set 'a=x\,y'` |
| from file content | `--set-file frontend.indexHtml=./index.html` |

## Examples / Walkthrough

> Work in an empty directory, for example `~/helm-lab`. Commands assume the context points at a disposable cluster. Expected output is abbreviated; exact timestamps/hashes differ.

### Step 0: Prerequisites check (5 min)

```bash
helm version --short
kubectl version --short 2>/dev/null || kubectl version
kubectl get nodes
kubectl get storageclass          # need a default class for the db PVC (local-path on kind/k3s/rke2)
kubectl get ingressclass          # optional; only needed for Step 13
```

Expected: helm `v3.x`, nodes `Ready`, one StorageClass marked `(default)`.

**Checkpoint:** if there is no default StorageClass, the db PVC stays `Pending` (lesson 41). Either install `local-path-provisioner` or set `db.persistence.enabled=false` in `values-dev.yaml` (we make that switchable on purpose).

### Step 1: Scaffold, then clear out the sample (5 min)

```bash
mkdir -p ~/helm-lab && cd ~/helm-lab
helm create shop
ls shop
# Chart.yaml  charts  templates  values.yaml  .helmignore
```

`helm create` generates a generic nginx chart with many features we do not want. We keep the skeleton (`Chart.yaml`, `.helmignore`) and rewrite everything else so every line is understood:

```bash
rm -rf shop/templates/* shop/values.yaml
ls shop/templates     # empty
```

**Why scaffold at all:** the tool creates the correct directory layout and `.helmignore`. Writing from zero is equally valid: `mkdir -p shop/templates` plus the files below.

### Step 2: Chart.yaml (3 min)

`shop/Chart.yaml`:

```yaml
apiVersion: v2
name: shop
description: Three-tier demo shop (frontend + backend + PostgreSQL)
type: application
# Chart version: bump on ANY change to the chart (templates or default values). SemVer.
version: 0.1.0
# Version of the application being deployed (informational; shown in `helm list`).
appVersion: "1.0.0"
kubeVersion: ">=1.27.0-0"
keywords:
  - shop
  - demo
maintainers:
  - name: DevOps Bootcamp
```

Key points:
- `version` = chart version (what `helm package` uses). `appVersion` = the app inside; they move independently.
- `apiVersion: v2` is Helm 3 charts.
- `kubeVersion` makes `helm install` fail early on too-old clusters.

### Step 3: values.yaml, the chart's public API (10 min)

`shop/values.yaml`:

```yaml
# -- Override the chart name used in labels
nameOverride: ""
# -- Override the full release-scoped name prefix
fullnameOverride: ""

imagePullSecrets: []

frontend:
  replicaCount: 1
  image:
    repository: nginx
    tag: "1.27-alpine"
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
  resources: {}
  # Extra text appended inside the nginx `server {}` block (advanced users)
  extraServerConfig: ""

backend:
  replicaCount: 1
  image:
    repository: traefik/whoami
    tag: "v1.10"
    pullPolicy: IfNotPresent
  # whoami listens on 80 by default; --port changes it
  containerPort: 8080
  service:
    type: ClusterIP
    port: 8080
  # Extra environment variables as a map: NAME: value
  env: {}
  resources: {}

db:
  # Set false to point the backend at an external database instead
  enabled: true
  image:
    repository: postgres
    tag: "16-alpine"
    pullPolicy: IfNotPresent
  auth:
    database: shop
    username: shop
    # REQUIRED. No default on purpose: a default password in a chart is a vulnerability.
    # Production should not pass this on the command line; lesson 43 (ESO) replaces it.
    password: ""
  persistence:
    enabled: true
    size: 1Gi
    # empty -> cluster default StorageClass
    storageClass: ""
  resources: {}

ingress:
  enabled: false
  className: nginx
  host: shop.example.local
  annotations: {}
```

Design choices to point out:
- **Per-component sections** (`frontend.*`, `backend.*`, `db.*`) keep `--set` paths predictable.
- `resources: {}` renders as "nothing" unless a user fills it; template uses `with`.
- `password: ""` + `required` in the template forces a conscious choice (Step 11 shows the error).
- `service.port` and `containerPort` are separate so users can change the Service port without rebuilding the image.

### Step 4: `_helpers.tpl`, names and labels (10 min)

`shop/templates/_helpers.tpl`:

```yaml
{{/*
Chart name, overridable.
*/}}
{{- define "shop.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Fully qualified app name: <release>-<chart>, max 63 chars (DNS label limit).
If the release name already contains the chart name, do not repeat it.
*/}}
{{- define "shop.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Chart label value: name-version (+ is not allowed in labels, so replace it).
*/}}
{{- define "shop.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Selector labels: IMMUTABLE identity of a component's Pods.
Call with: (dict "ctx" . "component" "frontend")
*/}}
{{- define "shop.selectorLabels" -}}
app.kubernetes.io/name: {{ include "shop.name" .ctx }}
app.kubernetes.io/instance: {{ .ctx.Release.Name }}
app.kubernetes.io/component: {{ .component }}
{{- end }}

{{/*
Common labels for metadata.labels: selector labels + descriptive extras.
Call with: (dict "ctx" . "component" "frontend")
*/}}
{{- define "shop.labels" -}}
helm.sh/chart: {{ include "shop.chart" .ctx }}
{{ include "shop.selectorLabels" . }}
{{- if .ctx.Chart.AppVersion }}
app.kubernetes.io/version: {{ .ctx.Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .ctx.Release.Service }}
{{- end }}

{{/*
Name of the database Secret, Service (headless), and host used by the backend.
*/}}
{{- define "shop.db.fullname" -}}
{{- printf "%s-db" (include "shop.fullname" .) | trunc 63 | trimSuffix "-" }}
{{- end }}
```

Teaching notes:
- A file starting with `_` is **not rendered as a manifest**; it only holds definitions.
- `include` returns a string (pipeable: `| nindent 4`); `template` does not (lesson 54).
- Inside helpers called with a dict, the root context is `.ctx`, not `.`. Most common bug: writing `.Release.Name` instead of `.ctx.Release.Name`.
- `{{-` and `-}}` trim whitespace so the output has no blank lines.

### Step 5: ConfigMap and Secret (10 min)

`shop/templates/configmap-frontend.yaml`, an nginx config that proxies `/api` to the backend Service:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "shop.fullname" . }}-frontend
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "frontend") | nindent 4 }}
data:
  default.conf: |
    server {
      listen 80;
      server_name _;

      location / {
        default_type text/html;
        return 200 '<h1>Shop frontend</h1><p>release: {{ .Release.Name }}, chart: {{ .Chart.Name }}-{{ .Chart.Version }}</p><p><a href="/api">/api</a></p>\n';
      }

      location /api {
        proxy_pass http://{{ include "shop.fullname" . }}-backend:{{ .Values.backend.service.port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
      {{- with .Values.frontend.extraServerConfig }}
      {{- . | nindent 6 }}
      {{- end }}
    }
```

`shop/templates/secret-db.yaml`:

```yaml
{{- if .Values.db.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "shop.db.fullname" . }}
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "db") | nindent 4 }}
type: Opaque
stringData:
  POSTGRES_DB: {{ .Values.db.auth.database | quote }}
  POSTGRES_USER: {{ .Values.db.auth.username | quote }}
  POSTGRES_PASSWORD: {{ required "db.auth.password is required (use --set db.auth.password=... or a values file)" .Values.db.auth.password | quote }}
{{- end }}
```

Notes:
- `stringData` lets us write plain text; the API server base64-encodes it (lesson 42).
- `required "message" value` aborts rendering with the message when value is empty. Failing at `helm install` time beats a crash-looping database.
- **Security note:** a plain password in values means it is stored in the release Secret (`sh.helm.release.v1.*`) and in Git if the values file is committed. Acceptable for a lab. Lesson 43 (External Secrets Operator) shows the production approach: the chart would reference an existing Secret created by ESO instead of rendering one.

### Step 6: Database, headless Service + StatefulSet (15 min)

`shop/templates/db-service.yaml`:

```yaml
{{- if .Values.db.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "shop.db.fullname" . }}
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "db") | nindent 4 }}
spec:
  clusterIP: None
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
  selector:
    {{- include "shop.selectorLabels" (dict "ctx" . "component" "db") | nindent 4 }}
{{- end }}
```

`shop/templates/db-statefulset.yaml`:

```yaml
{{- if .Values.db.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "shop.db.fullname" . }}
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "db") | nindent 4 }}
spec:
  serviceName: {{ include "shop.db.fullname" . }}
  replicas: 1
  selector:
    matchLabels:
      {{- include "shop.selectorLabels" (dict "ctx" . "component" "db") | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "shop.selectorLabels" (dict "ctx" . "component" "db") | nindent 8 }}
      annotations:
        checksum/secret: {{ include (print $.Template.BasePath "/secret-db.yaml") . | sha256sum }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: postgres
          image: "{{ .Values.db.image.repository }}:{{ .Values.db.image.tag }}"
          imagePullPolicy: {{ .Values.db.image.pullPolicy }}
          ports:
            - name: postgres
              containerPort: 5432
          envFrom:
            - secretRef:
                name: {{ include "shop.db.fullname" . }}
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          {{- with .Values.db.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- if not .Values.db.persistence.enabled }}
      volumes:
        - name: data
          emptyDir: {}
      {{- end }}
  {{- if .Values.db.persistence.enabled }}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        {{- if .Values.db.persistence.storageClass }}
        storageClassName: {{ .Values.db.persistence.storageClass | quote }}
        {{- end }}
        resources:
          requests:
            storage: {{ .Values.db.persistence.size }}
  {{- end }}
{{- end }}
```

Walkthrough of the tricky parts:
- `PGDATA` points to a **subdirectory** because a freshly formatted volume contains `lost+found`, and `initdb` refuses a non-empty directory.
- Either `volumeClaimTemplates` (persistent) or `emptyDir` (throwaway) is rendered, controlled by one value. This is the standard "toggle persistence" pattern.
- Selector labels are identical in Service selector, StatefulSet selector, and Pod labels; helper guarantees it.
- Probes and readiness for the DB are deliberately omitted here; add them as an exercise (lesson 45).

### Step 7: Backend Deployment + Service (10 min)

`shop/templates/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "shop.fullname" . }}-backend
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "backend") | nindent 4 }}
spec:
  replicas: {{ .Values.backend.replicaCount }}
  selector:
    matchLabels:
      {{- include "shop.selectorLabels" (dict "ctx" . "component" "backend") | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "shop.selectorLabels" (dict "ctx" . "component" "backend") | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
          args: ["--port", "{{ .Values.backend.containerPort }}"]
          ports:
            - name: http
              containerPort: {{ .Values.backend.containerPort }}
          env:
            - name: DB_HOST
              value: {{ include "shop.db.fullname" . | quote }}
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: {{ include "shop.db.fullname" . }}
                  key: POSTGRES_DB
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: {{ include "shop.db.fullname" . }}
                  key: POSTGRES_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "shop.db.fullname" . }}
                  key: POSTGRES_PASSWORD
            {{- range $name, $value := .Values.backend.env }}
            - name: {{ $name }}
              value: {{ $value | quote }}
            {{- end }}
          {{- with .Values.backend.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

`shop/templates/backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "shop.fullname" . }}-backend
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "backend") | nindent 4 }}
spec:
  type: {{ .Values.backend.service.type }}
  ports:
    - name: http
      port: {{ .Values.backend.service.port }}
      targetPort: http
  selector:
    {{- include "shop.selectorLabels" (dict "ctx" . "component" "backend") | nindent 4 }}
```

Notes:
- `range $name, $value := .Values.backend.env` iterates a **map** (sorted by key), letting users add arbitrary variables with `--set backend.env.LOG_LEVEL=debug`.
- `.Values.backend.env` is `{}` by default so the loop renders nothing.
- Backend env references the Secret via `secretKeyRef`, so changing the Secret does not leak into the Deployment manifest.
- Limitation to discuss: if `db.enabled=false`, these `secretKeyRef`s point to a Secret that does not exist. Real charts add an `externalDatabase` block; see Stretch 2.

### Step 8: Frontend Deployment + Service with checksum (10 min)

`shop/templates/frontend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "shop.fullname" . }}-frontend
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "frontend") | nindent 4 }}
spec:
  replicas: {{ .Values.frontend.replicaCount }}
  selector:
    matchLabels:
      {{- include "shop.selectorLabels" (dict "ctx" . "component" "frontend") | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "shop.selectorLabels" (dict "ctx" . "component" "frontend") | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap-frontend.yaml") . | sha256sum }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: nginx
          image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
          imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
          volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d
          {{- with .Values.frontend.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      volumes:
        - name: nginx-conf
          configMap:
            name: {{ include "shop.fullname" . }}-frontend
```

`shop/templates/frontend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "shop.fullname" . }}-frontend
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "frontend") | nindent 4 }}
spec:
  type: {{ .Values.frontend.service.type }}
  ports:
    - name: http
      port: {{ .Values.frontend.service.port }}
      targetPort: http
  selector:
    {{- include "shop.selectorLabels" (dict "ctx" . "component" "frontend") | nindent 4 }}
```

The ConfigMap is mounted over `/etc/nginx/conf.d`, replacing the image's `default.conf`. A mounted ConfigMap updates in place after a delay, but nginx does not re-read config on its own; the **checksum annotation** is what guarantees a restart.

### Step 9: Ingress, toggled by value (5 min)

`shop/templates/ingress.yaml`:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "shop.fullname" . }}
  labels:
    {{- include "shop.labels" (dict "ctx" . "component" "frontend") | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- with .Values.ingress.className }}
  ingressClassName: {{ . }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "shop.fullname" . }}-frontend
                port:
                  number: {{ .Values.frontend.service.port }}
{{- end }}
```

With `ingress.enabled: false` (default) the whole file renders to an empty document and Helm skips it.

### Step 10: NOTES.txt, user instructions after install (3 min)

`shop/templates/NOTES.txt`:

```
{{ .Chart.Name }} {{ .Chart.Version }} installed as release "{{ .Release.Name }}" in namespace "{{ .Release.Namespace }}".

{{- if .Values.ingress.enabled }}

Open: http://{{ .Values.ingress.host }}/
{{- else }}

Reach the frontend locally:
  kubectl -n {{ .Release.Namespace }} port-forward svc/{{ include "shop.fullname" . }}-frontend 8080:{{ .Values.frontend.service.port }}
  curl http://localhost:8080/
  curl http://localhost:8080/api
{{- end }}

Components:
  frontend  replicas={{ .Values.frontend.replicaCount }}  image={{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}
  backend   replicas={{ .Values.backend.replicaCount }}  image={{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}
{{- if .Values.db.enabled }}
  db        persistence={{ .Values.db.persistence.enabled }}  size={{ .Values.db.persistence.size }}
{{- end }}
```

### Step 11: Validate before touching the cluster (10 min)

Static check first:

```bash
helm lint ./shop
```

Expected, because the password is empty:

```
==> Linting ./shop
[INFO] Chart.yaml: icon is recommended
[ERROR] templates/: execution error at (shop/templates/secret-db.yaml:15:28): db.auth.password is required (...)

Error: 1 chart(s) linted, 1 chart(s) failed
```

Our `required` works. Now provide environment overrides.

`shop/values-dev.yaml`:

```yaml
# Development: small, cheap, throwaway
frontend:
  replicaCount: 1
backend:
  replicaCount: 1
  env:
    LOG_LEVEL: debug
db:
  auth:
    password: dev-only-password
  persistence:
    enabled: true
    size: 1Gi
ingress:
  enabled: false
```

`shop/values-prod.yaml`:

```yaml
# Production: replicas, resources, ingress. Password is NOT here; pass it at install time
# (or, better, use a pre-created Secret / ESO, lesson 43).
frontend:
  replicaCount: 3
  resources:
    requests: {cpu: 50m, memory: 64Mi}
    limits: {memory: 128Mi}
backend:
  replicaCount: 3
  env:
    LOG_LEVEL: info
  resources:
    requests: {cpu: 100m, memory: 64Mi}
    limits: {memory: 256Mi}
db:
  persistence:
    enabled: true
    size: 10Gi
  resources:
    requests: {cpu: 250m, memory: 512Mi}
    limits: {memory: 1Gi}
ingress:
  enabled: true
  className: nginx
  host: shop.example.com
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

Lint and render:

```bash
helm lint ./shop -f shop/values-dev.yaml
# ==> Linting ./shop
# 1 chart(s) linted, 0 chart(s) failed

helm template shop ./shop -f shop/values-dev.yaml | less
helm template shop ./shop -f shop/values-dev.yaml | grep -E '^(kind|  name):' | paste - -
```

Expected objects: `ConfigMap`, `Secret`, `Service` ×3 (frontend, backend, db), `Deployment` ×2, `StatefulSet`. No `Ingress` in dev.

Render a single file and see how toggles behave:

```bash
helm template shop ./shop -f shop/values-dev.yaml -s templates/ingress.yaml
# Error: could not find template templates/ingress.yaml in chart
# (the file renders to nothing while ingress.enabled=false, so Helm has no output for it)

helm template shop ./shop -f shop/values-dev.yaml --set ingress.enabled=true -s templates/ingress.yaml
```

Server-side dry run (validates against the real API schemas, checks for name collisions, still creates nothing):

```bash
helm install shop ./shop -n shop-dev --create-namespace \
  -f shop/values-dev.yaml --dry-run --debug | head -80
```

`--debug` prints `USER-SUPPLIED VALUES`, `COMPUTED VALUES`, the full manifest, and rendered NOTES. Differences:

| Tool | Needs cluster | Catches |
|---|---|---|
| `helm lint` | no | syntax, missing Chart.yaml fields, template errors, `required` |
| `helm template` | no | YAML you will actually apply; template logic bugs |
| `--dry-run` | yes (talks to API server) | unknown fields, wrong apiVersion, namespace/CRD problems |

**Checkpoint:** `helm template ... | kubectl apply --dry-run=server -f -` works as an alternative validation path.

### Step 12: Install the dev release (10 min)

```bash
helm install shop ./shop -n shop-dev --create-namespace \
  -f shop/values-dev.yaml \
  --set frontend.replicaCount=2
```

Expected:

```
NAME: shop
LAST DEPLOYED: ...
NAMESPACE: shop-dev
STATUS: deployed
REVISION: 1
NOTES:
shop 0.1.0 installed as release "shop" in namespace "shop-dev".
...
```

Watch it come up:

```bash
kubectl -n shop-dev get pods -w
helm -n shop-dev list
helm -n shop-dev get values shop
```

`helm get values shop` shows **only what we supplied**: contents of `values-dev.yaml` plus `frontend.replicaCount: 2` (the `--set` won over the file's `1`). Confirm precedence:

```bash
kubectl -n shop-dev get deploy
# shop-backend    1/1
# shop-frontend   2/2        <- 2, not 1: --set > -f
```

Show object names generated by helpers:

```bash
kubectl -n shop-dev get all,cm,secret,pvc -l app.kubernetes.io/instance=shop
```

Expected names: `shop-frontend`, `shop-backend`, `shop-db`. The chart is called `shop` and the release is also `shop`, so the `contains` rule in `shop.fullname` sees the chart name inside the release name and returns just `shop` instead of `shop-shop`. Later, a release called `other` would give `other-shop-frontend`.

**Checkpoint (all must be true):**
- 4 Pods `Running`: 2 frontend, 1 backend, 1 db (`shop-db-0`)
- PVC `data-shop-db-0` is `Bound`
- Labels exist on every object: `kubectl -n shop-dev get all -l app.kubernetes.io/managed-by=Helm`

### Step 13: Verify end-to-end (10 min)

Without Ingress:

```bash
kubectl -n shop-dev port-forward svc/shop-frontend 8080:80 &
curl -s http://localhost:8080/
# <h1>Shop frontend</h1><p>release: shop, chart: shop-0.1.0</p>...
curl -s http://localhost:8080/api | head
# Hostname: shop-backend-xxxxx
# IP: 10.244.x.x
# ...
kill %1
```

`/api` returning `Hostname:` proves frontend → backend Service routing. Run it several times: the hostname alternates only when `backend.replicaCount > 1`.

Verify the DB wiring independently:

```bash
kubectl -n shop-dev exec -it shop-db-0 -- psql -U shop -d shop -c '\conninfo'
kubectl -n shop-dev exec -it shop-db-0 -- psql -U shop -d shop \
  -c 'CREATE TABLE IF NOT EXISTS orders(id serial primary key, item text);' \
  -c "INSERT INTO orders(item) VALUES ('keyboard');" \
  -c 'SELECT * FROM orders;'
```

Check backend env:

```bash
kubectl -n shop-dev exec deploy/shop-backend -- env | grep -E '^DB_|LOG_LEVEL'
# DB_HOST=shop-db
# DB_PORT=5432
# DB_NAME=shop
# DB_USER=shop
# DB_PASSWORD=dev-only-password
# LOG_LEVEL=debug
```

With an Ingress controller installed (lesson 38) you can also try:

```bash
helm upgrade shop ./shop -n shop-dev -f shop/values-dev.yaml \
  --set ingress.enabled=true --set ingress.host=shop.dev.example.local
curl -H 'Host: shop.dev.example.local' http://<ingress-controller-ip>/
```

### Step 14: Upgrade with changed values (15 min)

Three kinds of upgrade, each proving a concept.

**(a) Change a replica count: only the Deployment changes**

```bash
helm upgrade shop ./shop -n shop-dev -f shop/values-dev.yaml --set backend.replicaCount=3
helm -n shop-dev history shop
# REVISION  STATUS      CHART       APP VERSION  DESCRIPTION
# 1         superseded  shop-0.1.0  1.0.0        Install complete
# 2         deployed    shop-0.1.0  1.0.0        Upgrade complete
```

> Important: `--set frontend.replicaCount=2` from Step 12 is **not** remembered. A plain `helm upgrade` resets to chart defaults + the files/`--set` you pass this time, so frontend returns to 1. Helm 3 does not reuse previous values unless you pass `--reuse-values` (or repeat every flag). This is the most common upgrade surprise.

```bash
kubectl -n shop-dev get deploy
# shop-frontend 1/1   <- reverted
```

Preferred: keep **all** overrides in files and run `helm upgrade --install` with the same `-f` files every time.

**(b) Change an image tag: rolling update**

```bash
helm upgrade shop ./shop -n shop-dev -f shop/values-dev.yaml \
  --set frontend.image.tag=1.27.2-alpine
kubectl -n shop-dev rollout status deploy/shop-frontend
helm -n shop-dev get manifest shop | grep 'image:'
```

**(c) Change config: the checksum annotation triggers a roll**

```bash
helm upgrade shop ./shop -n shop-dev -f shop/values-dev.yaml \
  --set frontend.extraServerConfig='add_header X-Shop-Env dev always;'
kubectl -n shop-dev get pods -l app.kubernetes.io/component=frontend
# new Pod names -> rollout happened
curl -sI http://localhost:8080/ | grep -i x-shop-env     # re-run port-forward first
```

Prove the annotation is what caused it: remove the `checksum/config` line from `frontend-deployment.yaml`, repeat with a different `extraServerConfig` and observe **no** Pod restart (ConfigMap updated but nginx still serving old config). Put the annotation back.

Compare revisions without the cluster:

```bash
helm -n shop-dev get values shop --revision 2
helm -n shop-dev get manifest shop --revision 2 > r2.yaml
helm -n shop-dev get manifest shop > current.yaml
diff r2.yaml current.yaml
```

### Step 15: Rollback (5 min)

```bash
helm -n shop-dev history shop
helm -n shop-dev rollback shop 2
helm -n shop-dev history shop
# REVISION ... 5  superseded ...
# REVISION ... 6  deployed   shop-0.1.0 1.0.0 Rollback to 2
```

Rollback does not rewind history; it creates a **new revision** whose content equals revision 2. The StatefulSet and its PVC are untouched, so the `orders` row still exists:

```bash
kubectl -n shop-dev exec shop-db-0 -- psql -U shop -d shop -c 'SELECT * FROM orders;'
```

Caveat: Helm rolls back Kubernetes manifests, **not data**. A destructive schema migration cannot be undone by `helm rollback`.

### Step 16: Package the chart (10 min)

Bump chart version because we changed templates since 0.1.0 was first written:

```bash
helm package ./shop --version 0.1.1 --app-version 1.0.1
# Successfully packaged chart and saved it to: /home/user/helm-lab/shop-0.1.1.tgz

tar tzf shop-0.1.1.tgz
helm show chart shop-0.1.1.tgz
helm show values shop-0.1.1.tgz | head -20
```

- `--version` overrides `Chart.yaml: version`; `--app-version` overrides `appVersion`.
- Files in `.helmignore` (for example `.git/`, `*.swp`, `values-*.yaml` if you add it) are excluded from the archive. Decide whether environment files ship with the chart; many teams keep them in a separate Git repo and exclude them.
- Pushing the `.tgz` to a repository (ChartMuseum/OCI registry) is lesson 53-style repo work and appears in CI lessons later; here we install from the local file.

Lint the package and compare:

```bash
helm lint shop-0.1.1.tgz -f shop/values-dev.yaml
```

### Step 17: Install the "prod" release from the `.tgz` (10 min)

Same chart, different values file, separate namespace, secret supplied at install time:

```bash
helm upgrade --install shop-prod ./shop-0.1.1.tgz \
  -n shop-prod --create-namespace \
  -f shop/values-prod.yaml \
  --set db.auth.password="$(openssl rand -base64 18)"
```

Check:

```bash
helm list -A
kubectl -n shop-prod get deploy,sts,pvc,ingress
# shop-prod-frontend   3/3
# shop-prod-backend    3/3
```

Notice the naming: the release name `shop-prod` contains the chart name `shop`, so `shop.fullname` returns `shop-prod` and objects are `shop-prod-frontend`, not `shop-prod-shop-frontend`. Predict the names before running the command; if your output differs, re-read the `shop.fullname` helper. Also note that `helm package` with `--version 0.1.1` and the Secret/StatefulSet names follow the same prefix (`shop-prod-db`).

Compare the two releases:

```bash
helm get values shop      -n shop-dev  -a | head -40
helm get values shop-prod -n shop-prod -a | head -40
diff <(helm get manifest shop -n shop-dev) <(helm get manifest shop-prod -n shop-prod)
```

Using `-a` shows computed values including defaults, handy when you wonder "what did the chart actually apply?".

### Step 18: Failure drills (10 min)

Do these deliberately; students must fix each.

1. **Bad indentation**: change `nindent 4` to `nindent 2` in one `labels:` block, run `helm lint` and `helm template`. Fix.
2. **Missing value**: `helm template shop ./shop` with no password, see `required` error.
3. **Nil pointer**: write `{{ .Values.frontend.imag.tag }}` (typo), observe `nil pointer evaluating interface {}.tag`.
4. **Immutable selector**: add `version: x` to `shop.selectorLabels`, run `helm upgrade` on the live dev release:
   `Error: UPGRADE FAILED: cannot patch "shop-backend" ... field is immutable`. Revert the helper. Lesson: selector labels are forever.
5. **Failed upgrade state**: after a failed upgrade `helm history` shows `failed`; fix the chart and run `helm upgrade` again (no need to uninstall). Use `helm rollback` if the previous revision was good.

### Step 19: Uninstall and what survives (5 min)

```bash
helm uninstall shop -n shop-dev
kubectl -n shop-dev get all,cm,secret
kubectl -n shop-dev get pvc
# NAME             STATUS   VOLUME    CAPACITY
# data-shop-db-0   Bound    pvc-...   1Gi          <- still here
```

PVCs created from `volumeClaimTemplates` are **not owned by the StatefulSet or by Helm**, so they survive uninstall (a safety feature: your database data is not auto-deleted). Reinstalling the same release name in the same namespace re-attaches the old data. Delete deliberately:

```bash
kubectl -n shop-dev delete pvc data-shop-db-0
helm uninstall shop-prod -n shop-prod
kubectl delete ns shop-dev shop-prod
```

Option: `helm uninstall --keep-history` keeps revision records so you can `helm rollback` a deleted release.

## Common Pitfalls

- **`include` with the wrong context inside a dict helper** — in `shop.labels` you must write `.ctx.Chart.Name`, not `.Chart.Name`. Symptom: `nil pointer evaluating interface {}.Chart`. Fix: use `.ctx` everywhere inside helpers called with a dict.
- **Passing `.` only to `shop.labels`** — helper expects `.ctx` and `.component`, so call it as `include "shop.labels" (dict "ctx" . "component" "backend")`. Wrapping parentheses are mandatory.
- **`nindent` count wrong** — YAML is whitespace-sensitive. Count the indent of the line that holds the key, then add the nesting. Use `helm template` and eyeball output; `helm lint` often catches it, `helm template | kubectl apply --dry-run=server -f -` always does.
- **Forgetting `-}}` / `{{-`** — leaves blank lines or merges lines; manifests with an empty document are fine, merged lines are not.
- **Version/chart label inside selector labels** — selectors are immutable; the next `helm upgrade` fails with `field is immutable`. Keep `shop.selectorLabels` to name/instance/component only.
- **Plain `helm upgrade` resets values** — Helm 3 does not carry over earlier `--set` flags. Use files or `--reuse-values` (but beware stale defaults: `--reuse-values` ignores new defaults added to the chart; `--reset-then-reuse-values` exists on newer Helm).
- **List values with `--set`** — lists are replaced entirely and syntax is ugly; prefer maps or files.
- **Typing numbers/bools into strings** — `--set backend.image.tag=1.10` becomes `1.1` (float). Use `--set-string` or quote in values file.
- **Quoted vs unquoted** — numbers like `"5432"` in env `value:` must be strings in Kubernetes env; use `| quote`.
- **Secrets in `--set` land in shell history and in the release Secret** — acceptable for lab; use ESO / sealed secrets (lesson 43) or a pre-created Secret reference for real environments.
- **ConfigMap changes do not restart Pods** — add the checksum annotation, or the change is invisible until a manual `kubectl rollout restart`.
- **`PGDATA` directly on mount point** — `initdb: directory "/var/lib/postgresql/data" exists but is not empty` (lost+found). Use a subdirectory.
- **Expecting `helm uninstall` to delete PVCs** — it does not for StatefulSet templates; storage cost and stale data surprise teams. Delete explicitly.
- **Forgetting to bump `version:` in Chart.yaml** — packaging the same version with different contents causes confusing repo/cache behavior. Bump on every change.
- **Testing only the happy path** — always render with at least two values files and with each toggle (`ingress.enabled`, `db.persistence.enabled`, `db.enabled`) flipped.

## Troubleshooting Table

| Symptom | Likely cause | Check / fix |
|---|---|---|
| `Error: INSTALLATION FAILED: ... db.auth.password is required` | `required` triggered | pass `-f values-dev.yaml` or `--set db.auth.password=...` |
| `YAML parse error ... mapping values are not allowed in this context` | wrong indentation from `nindent`/`indent` | `helm template ... > out.yaml`, open at reported line |
| `nil pointer evaluating interface {}.x` | typo in `.Values` path, or `.` vs `.ctx` in helper | compare to `helm show values`; inside dict helpers use `.ctx` |
| `Error: template: ... function "xyz" not defined` | misspelled function | check Sprig docs; `toYaml`, `nindent`, `quote`, `required`, `sha256sum` |
| `UPGRADE FAILED: cannot patch ... field is immutable` | selector labels or StatefulSet `volumeClaimTemplates` changed | restore selector labels; for PVC templates recreate StatefulSet (`kubectl delete sts --cascade=orphan`) |
| `Error: rendered manifests contain a resource that already exists` | an object with the same name created outside Helm, or in another release | `kubectl get <kind> <name> -o yaml` ownership annotations; rename or delete |
| `Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress` | previous run crashed or timed out | `helm history`; if `pending-*`, `helm rollback` to last good revision |
| PVC `Pending` | no default StorageClass or WaitForFirstConsumer with no Pod | `kubectl describe pvc`; set `db.persistence.storageClass` or disable persistence |
| db Pod `CrashLoopBackOff`, log `initdb: directory ... not empty` | missing `PGDATA` subdir | check env in StatefulSet |
| backend Pod `CreateContainerConfigError` | secret key missing or `db.enabled=false` | `kubectl describe pod`; check Secret keys |
| frontend returns 502 on `/api` | backend Service name/port mismatch in nginx ConfigMap | `kubectl get svc`; compare `proxy_pass` host:port |
| config change not visible after upgrade | no checksum annotation, or hash computed from wrong file | confirm path `print $.Template.BasePath "/configmap-frontend.yaml"` matches file name |
| `helm lint` warns icon recommended | harmless INFO | add `icon:` URL or ignore |
| `ImagePullBackOff` | tag typo from `--set` | `helm get values`, `kubectl describe pod` |
| `helm list` shows nothing | wrong namespace | `helm list -A` |

## FAQ

**Q: Should I use `helm create` or write the chart from scratch?**
A: Both end up the same. `helm create` gives a directory layout, `.helmignore`, and a working but very generic nginx chart. For learning, delete the generated templates and write your own; for production, many teams still start from `helm create` and trim. Whatever you start from, delete anything you do not understand.

**Q: What is the difference between `helm template` and `helm install --dry-run`?**
A: `helm template` renders entirely offline with a fake release context (no cluster needed; some built-ins such as `.Capabilities` are defaults). `helm install --dry-run` contacts the API server, validates the manifests against real schemas, and uses real release info, but creates nothing. Use `template` for fast inner-loop iteration, `--dry-run --debug` as a final gate.

**Q: Why does my chart name appear twice in names, like `shop-shop-frontend`?**
A: When release name does not contain the chart name, the helper prints `<release>-<chart>`. With release `shop` and chart `shop`, the `contains` check prevents duplication and the result is `shop-frontend`. If you remove the `contains` branch you get `shop-shop-frontend`. Use `fullnameOverride` to set the prefix explicitly.

**Q: Why use a dict argument for helpers instead of just `.`?**
A: `include` takes exactly one argument. A multi-component chart needs both the root context and a component name, so we pack them in a dict. The alternative is to repeat labels per component by hand, which drifts.

**Q: What is the exact precedence of values?**
A: `values.yaml` < parent chart's values for this subchart < `-f` files in order (last wins) < `--set`/`--set-string`/`--set-file` in order (last wins). Maps are merged key by key; lists and scalars are replaced.

**Q: Why did my `--set` disappear after the next `helm upgrade`?**
A: Helm 3 does not remember user-supplied values across upgrades unless you add `--reuse-values`. Best practice is to keep every override in version-controlled values files and pass the same `-f` flags every time.

**Q: Can I put environment values files inside the chart directory?**
A: You can (as in this lab) and they are packaged unless ignored. Many teams keep chart and per-environment values in separate repositories so the same chart artifact is promoted unchanged between environments. Both work; pick one and be consistent.

**Q: How do I make a Secret not be rendered by the chart at all?**
A: Add a value like `db.auth.existingSecret`, and in templates use `{{ default (include "shop.db.fullname" .) .Values.db.auth.existingSecret }}` for `secretRef`/`secretKeyRef` names, rendering the Secret only `if not .Values.db.auth.existingSecret`. The Secret itself is then created by ESO (lesson 43) or an operator.

**Q: What is the checksum annotation and do I need it for Secrets too?**
A: It stores a hash of a rendered file in Pod template annotations. Changed content, changed hash, new ReplicaSet, rolling restart. Do it for any ConfigMap or Secret consumed as env vars or files if the app does not hot reload. We did both (`checksum/config` on frontend, `checksum/secret` on db).

**Q: Why does uninstall keep my database volume?**
A: PVCs from `volumeClaimTemplates` are not owned by the StatefulSet, and Helm only deletes objects it rendered. This protects data. Delete the PVC explicitly when you really want it gone.

**Q: What does `helm rollback` do to the database?**
A: Nothing to the data. It reapplies old manifests as a new revision. Schema migrations or data changes are not reversed; plan backward-compatible migrations.

**Q: What are Helm hooks and `helm test`?**
A: Hooks (annotation `helm.sh/hook`) run Jobs at lifecycle points like pre-upgrade for migrations; `helm test` runs Pods marked as tests. Both are beyond this lesson; know they exist.

**Q: Why is `Chart.yaml: version` different from `appVersion`?**
A: `version` identifies the packaging of the chart (templates/values) and drives `helm package` and repositories. `appVersion` is the version of the application image. You can release chart 0.1.5 with unchanged app 1.0.0 when only templates changed.

**Q: How do I see the final YAML Helm applied to the cluster?**
A: `helm get manifest <release>`. To see the values used: `helm get values <release>` (user supplied) or `-a` (all). For a given historic revision add `--revision N`.

**Q: How should I pick `nindent` vs `indent`?**
A: `nindent N` adds a leading newline then indents; use it when the block starts on the next line after a key. `indent N` has no leading newline; use it when the content continues on the same line (rare). 95% of the time you want `nindent`.

**Q: Should the database be in the app chart at all?**
A: For production, usually no: use a managed database or a dedicated operator/chart and deploy it as its own release. In this lab it is included for a self-contained full-stack example, and Stretch 1 shows the dependency approach.

## Practice / Exercise

### Core

1. Complete Steps 1-12 and show `helm -n shop-dev list`, `kubectl -n shop-dev get all,pvc` with all Pods `Running` and PVC `Bound`.
2. Run `helm lint`, `helm template`, and `helm install --dry-run --debug`; save the rendered manifest to `rendered.yaml` and list every object kind and name.
3. Prove values precedence: install with `-f values-dev.yaml --set frontend.replicaCount=2` and show `helm get values` and live replica count; then upgrade without the `--set` and explain what happened.
4. Upgrade three times (replica count, image tag, `extraServerConfig`). Show `helm history`, then roll back to revision 1 and show the new revision number.
5. Remove the checksum annotation, change the ConfigMap, observe no rollout; restore it and observe a rollout.
6. Package the chart as `0.1.1` with `--app-version 1.0.1`, inspect it with `tar tzf` and `helm show chart`, and install `shop-prod` into namespace `shop-prod` from the `.tgz` using `values-prod.yaml` and a password via `--set`.
7. Write down the differences between the two releases using `diff` of their manifests.
8. Insert a row into the DB, uninstall the dev release, reinstall, and show whether the row survived. Explain why.
9. Break the chart in three of the ways listed in Step 18, record the error text for each, and fix it.

### Stretch

1. **PostgreSQL as an optional dependency:** replace the hand-written db templates with the Bitnami (or other) `postgresql` subchart declared in `Chart.yaml` `dependencies:` with `condition: postgresql.enabled`; run `helm dependency update ./shop`; override subchart values under the `postgresql:` key; wire backend env to the subchart's Service and Secret names. Compare the two designs.
2. **External database mode:** add `externalDatabase.host/port/existingSecret`; render backend env from either the in-chart DB or the external one with `if`/`else`; test with `db.enabled=false`.
3. **`existingSecret`:** support `db.auth.existingSecret` as in the FAQ; test with a Secret created by `kubectl create secret generic`.
4. **Add `probes` as values:** `frontend.probes.enabled` with `httpGet` on `/`; render with `with`/`toYaml`; wire the DB readiness with `pg_isready` exec probe (lesson 45).
5. **Validate values with a schema:** add `values.schema.json` that requires `db.auth.password` of minimum length 8 and restricts `frontend.service.type` to `ClusterIP|NodePort|LoadBalancer`; run `helm lint` with violating values and read the schema error.
6. **HorizontalPodAutoscaler toggle:** add `backend.autoscaling.enabled|minReplicas|maxReplicas|targetCPU`; when enabled, omit `replicas:` from the Deployment (`if not`), and render an HPA.
7. **Library-style helper:** extract the repeated `imagePullSecrets` and `resources` blocks into named templates in `_helpers.tpl` and call them with `include`. Measure how many lines it saves.
8. **Convert** `frontend.extraServerConfig` to use `tpl` so values can contain template expressions such as `{{ .Release.Name }}` and test it.

## Further Reading

- Helm chart best practices: https://helm.sh/docs/chart_best_practices/
- Helm chart template guide (functions, flow control, named templates): https://helm.sh/docs/chart_template_guide/
- `helm package`, `helm lint`, `helm template` reference: https://helm.sh/docs/helm/
- Values files and precedence: https://helm.sh/docs/chart_template_guide/values_files/
- Chart dependencies: https://helm.sh/docs/helm/helm_dependency/
- Sprig function library (used by Helm): https://masterminds.github.io/sprig/
- Kubernetes recommended labels: https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
- StatefulSet volumeClaimTemplates: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

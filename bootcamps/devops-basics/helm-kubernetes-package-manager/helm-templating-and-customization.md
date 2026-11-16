# Lesson 54: Helm Templating and Customization

**Module:** Helm – Kubernetes Package Manager
**Duration:** 120-150 min
**Prerequisites:** Lesson 53 (Helm concepts, chart structure, repositories, `helm install/upgrade/rollback/uninstall`, `--set` and `-f values.yaml`), plus Lessons 34-52 (Deployments, Services, ConfigMaps, Secrets, Ingress). Student can read a Deployment YAML and a `values.yaml` file.

## Learning Objectives

By end of lesson student can:
- Read and write Go template expressions `{{ ... }}` inside Kubernetes manifests
- Use pipelines (`|`), variables (`$x :=`), comments, and whitespace control (`{{-` and `-}}`) without breaking YAML
- Explain the difference between `.` (current scope) and `$` (root scope) and fix scope errors inside `with` and `range`
- Use the built-in objects `.Values`, `.Release`, `.Chart`, `.Files`, `.Capabilities`, `.Template`
- Apply the most used functions: `quote`, `default`, `required`, `toYaml | nindent`, `include`, `tpl`, `printf`, `trunc`, `trimSuffix`, `sha256sum`, `b64enc`, `ternary`, `kindIs`, `empty`
- Define named templates in `_helpers.tpl` and call them with `include` (and explain why `include` beats `template`)
- Write conditionals (`if / else if / else`, `with`) and loops (`range` over lists and maps) and know Helm truthiness rules
- Debug rendering with `helm template`, `--debug`, `--show-only`, `helm lint`, and `helm install --dry-run --debug`
- Recognise and fix the three classic errors: `nil pointer evaluating interface {}`, YAML indentation errors, whitespace bugs
- Predict which value wins when several sources (`values.yaml`, `-f`, `--set`) set the same key

## Topics

- Go template syntax: `{{ }}`, pipeline `|`, whitespace control `{{- -}}`, comments
- Built-in objects: `.Values`, `.Release` (Name/Namespace/IsInstall/IsUpgrade), `.Chart`, `.Files`
- Functions: `quote`, `default`, `toYaml | nindent`, `include`, `tpl`; type checks
- `_helpers.tpl`: defining named templates with `{{- define }}`, `include` vs `template`
- Conditionals (`if/else/with`) and loops (`range` over list/map) in templates
- Extras for depth: `required`, `lookup`, `NOTES.txt`, values precedence, debugging toolbox, common errors

## Concepts

### Why templating exists

A plain Kubernetes manifest is static. To run the same app in `dev` and `prod` you would copy the YAML and edit image tag, replicas, hostnames, resource limits by hand. Two copies drift apart, and a change must be made twice.

Helm solves this by turning manifests into **templates**: YAML files with holes. At install time Helm fills the holes with **values** (from `values.yaml`, `-f file`, `--set`) and produces plain YAML that is sent to the API server.

```
 templates/*.yaml  +  values (merged)  +  release info   ──►  rendered manifests  ──►  kube-apiserver
   (Go templates)     (.Values)           (.Release)           (plain YAML)
```

Key point: **Helm renders on the client**. The API server never sees `{{ }}`. If rendering fails, nothing is applied. If rendering succeeds but produces invalid Kubernetes YAML, the API server rejects it.

Helm uses Go's `text/template` package plus the **Sprig** function library (about 70 extra functions) plus a few Helm-specific functions (`include`, `tpl`, `toYaml`, `required`, `lookup`).

### Anatomy of a template action

Everything between `{{` and `}}` is an **action**. Everything outside is copied literally.

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config      # action: value substitution
data:
  greeting: {{ .Values.greeting | quote }}   # action with a pipeline
```

With `helm install demo ./webapp --set greeting=hello` the output is:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  greeting: "hello"
```

### Comments

```yaml
{{/* This is a template comment. It is removed from output. */}}
{{- /* Comment that also trims surrounding whitespace */ -}}
# This is a YAML comment. It stays in the rendered output.
```

Use `{{/* */}}` for notes to chart authors. A YAML `#` comment is visible in `helm get manifest`. A common mistake is to comment out a template action with `#`: the action **still runs** (and can still fail). To disable an action, use `{{/* ... */}}` around it.

### The pipeline

A pipeline passes the result of the left side as the **last argument** of the right side, like a Unix pipe.

```yaml
name: {{ .Values.name | upper | quote }}
# same as: quote (upper .Values.name)
```

Read pipelines left to right. The first element is a value, every next element is a function receiving the previous result.

```yaml
image: {{ printf "%s:%s" .Values.image.repository .Values.image.tag | quote }}
tag:   {{ .Values.image.tag | default .Chart.AppVersion | quote }}
```

Careful with `default`: the **fallback is the first argument**, the piped value is the last. `{{ .Values.x | default "a" }}` means `default "a" .Values.x`: use `.Values.x`, or `"a"` if it is empty.

### Whitespace control

Template actions leave their surrounding newlines and spaces in the output. YAML is whitespace-sensitive, so this matters.

- `{{-` removes all whitespace (spaces, tabs, **newlines**) **before** the action
- `-}}` removes all whitespace **after** the action

```yaml
# Without trimming
data:
  {{ if .Values.debug }}
  debug: "true"
  {{ end }}
```

Renders (debug=false) as a blank-line mess:

```yaml
data:
  
  
```

With trimming:

```yaml
data:
  {{- if .Values.debug }}
  debug: "true"
  {{- end }}
```

Renders (debug=true):

```yaml
data:
  debug: "true"
```

Mental model: `{{-` eats the newline *above* the action, pulling the action up to the end of the previous line. Then the content between the actions keeps its own leading newline and indentation.

Rule of thumb for control structures (`if`, `range`, `with`, `end`, `define`): put `{{-` on the left, leave the right side alone.
Rule of thumb for value actions: leave both sides alone unless you must remove a space.

Danger: `-}}` also eats the newline **after**, which can glue the next line to the current one:

```yaml
name: {{ .Values.name -}}
version: 1        # renders as: name: myappversion: 1   (BROKEN)
```

Note the space is required: `{{- ` and ` -}}`. `{{-3}}` means the number minus three, not trim.

### Variables

Assign with `:=`, reassign with `=`. Variables are scoped to the block in which they are defined (`if`, `range`, `with`, or template top level).

```yaml
{{- $name := .Release.Name }}
{{- $port := .Values.service.port | default 80 }}
metadata:
  name: {{ $name }}-svc
spec:
  ports:
    - port: {{ $port }}
```

Reassignment (`=`) lets you update an outer variable from inside a loop:

```yaml
{{- $total := 0 }}
{{- range .Values.volumes }}
{{- $total = add $total .sizeGi }}
{{- end }}
# total: {{ $total }}
```

### Scope: `.` versus `$`

The dot `.` means "the current context". At the top of a template it is the root object (with `.Values`, `.Release`, etc.). Inside `with` and `range` the dot is **rebound** to a different value.

```yaml
{{- with .Values.image }}
image: {{ .repository }}:{{ .tag }}        # here . is .Values.image
name: {{ .Release.Name }}                  # ERROR: .Release does not exist under .Values.image
name: {{ $.Release.Name }}                 # OK: $ is always the root
{{- end }}
```

`$` is set to the root context at the start of every template and never changes. Use `$` to reach `.Release`, `.Chart`, `.Values` from inside loops. This is the single most common source of "nil pointer" errors for beginners.

Inside a **named template** (`define`), `.` is whatever you pass in when you `include`. If you call `include "x" .` you pass the root. If you call `include "x" .Values.foo` the template sees only that sub-tree and `$` inside it is that sub-tree too.

### Built-in objects

Helm injects top-level objects into every template.

| Object | What it contains |
|---|---|
| `.Values` | Merged values: chart `values.yaml` + parent chart + `-f` files + `--set` |
| `.Release.Name` | Release name given to `helm install` |
| `.Release.Namespace` | Namespace the release is installed in |
| `.Release.IsInstall` | `true` when this operation is an install |
| `.Release.IsUpgrade` | `true` when this operation is an upgrade or rollback |
| `.Release.Revision` | Revision number (1 on install, +1 per upgrade) |
| `.Release.Service` | Always `Helm` |
| `.Chart` | Contents of `Chart.yaml` (capitalised keys: `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion`) |
| `.Files` | Access to non-template files inside the chart |
| `.Capabilities` | Cluster info: `.Capabilities.KubeVersion.Version`, `.Capabilities.APIVersions.Has "..."` |
| `.Template` | `.Template.Name` (path of current template), `.Template.BasePath` |

Important details:

- **Chart.yaml fields are capitalised** in templates: `.Chart.Name`, not `.Chart.name`. Values keys keep whatever case you wrote in `values.yaml`.
- `helm template` has no cluster, so `.Release.IsInstall` is `true` and `.Release.Namespace` defaults to `default` (use `--namespace` to change it). `.Capabilities` shows a default Kubernetes version unless you pass `--kube-version` and `--api-versions`.
- Values keys are case-sensitive: `.Values.replicaCount` and `.Values.replicacount` are different.
- Built-in object names start with a capital letter. Local variables you create may start lower-case.

#### `.Values`

`values.yaml` in the chart is the **default**. Anything the user passes overrides it.

```yaml
# values.yaml
replicaCount: 2
image:
  repository: nginx
  tag: ""
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources: {}
```

```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

#### `.Files`

Read static files packaged inside the chart (anything not under `templates/` and not excluded by `.helmignore`).

```yaml
# files/app.conf inside the chart
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-conf
data:
  app.conf: |
{{ .Files.Get "files/app.conf" | indent 4 }}
```

Other `.Files` helpers:

| Call | Result |
|---|---|
| `.Files.Get "path"` | File content as string |
| `.Files.GetBytes "path"` | Content as bytes (for binary / `b64enc`) |
| `.Files.Glob "files/*.conf"` | Map of matching files |
| `.Files.Glob "files/*.conf" \| .AsConfig` | Ready-made `data:` entries for a ConfigMap |
| `.Files.Glob "secrets/*" \| .AsSecrets` | Ready-made base64 `data:` entries for a Secret |
| `.Files.Lines "path"` | File split into lines (use with `range`) |

```yaml
data:
{{ (.Files.Glob "files/*.conf").AsConfig | indent 2 }}
```

Gotcha: files in `templates/` are **not** reachable by `.Files`. Files matching `.helmignore` are also invisible.

#### `.Capabilities`

Lets a chart adapt to cluster version.

```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/Ingress" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
kind: Ingress
```

```yaml
{{- if semverCompare ">=1.25.0-0" .Capabilities.KubeVersion.Version }}
# use policy/v1 PodDisruptionBudget
{{- end }}
```

#### `.Template`

`{{ .Template.Name }}` returns something like `webapp/templates/deployment.yaml`. Rarely used. Handy in error messages: `{{ fail (printf "bad config in %s" .Template.Name) }}`.

### Functions you will use daily

Helm ships Go-template builtins plus Sprig. Learn these first.

#### Strings

| Function | Example | Result |
|---|---|---|
| `quote` | `{{ "hi" \| quote }}` | `"hi"` |
| `squote` | `{{ "hi" \| squote }}` | `'hi'` |
| `upper` / `lower` / `title` | `{{ "web" \| upper }}` | `WEB` |
| `trim` | `{{ "  a " \| trim }}` | `a` |
| `trunc` | `{{ "abcdefgh" \| trunc 5 }}` | `abcde` |
| `trimSuffix` | `{{ "app-" \| trimSuffix "-" }}` | `app` |
| `trimPrefix` | `{{ "v1.2" \| trimPrefix "v" }}` | `1.2` |
| `replace` | `{{ "a.b" \| replace "." "-" }}` | `a-b` |
| `printf` | `{{ printf "%s-%d" "web" 3 }}` | `web-3` |
| `contains` | `{{ contains "ab" "cabd" }}` | `true` |
| `hasPrefix` | `{{ hasPrefix "ab" "abc" }}` | `true` |
| `repeat` | `{{ "x" \| repeat 3 }}` | `xxx` |
| `nospace` | `{{ "a b" \| nospace }}` | `ab` |
| `kebabcase` / `snakecase` / `camelcase` | `{{ "FooBar" \| kebabcase }}` | `foo-bar` |

Why `quote` matters: YAML turns `on`, `yes`, `no`, `1.10`, `0123`, and `null` into booleans, numbers, or null. `{{ .Values.version }}` with `1.10` renders `1.10`, parsed as float `1.1`. `{{ .Values.version | quote }}` renders `"1.10"`, always a string. Rule: **quote every value that must be a string** (env var values, labels, annotations, image tags).

Kubernetes names are limited to 63 characters (DNS label). That is why `trunc 63 | trimSuffix "-"` appears in nearly every `_helpers.tpl`.

#### Defaults and validation

```yaml
replicas: {{ .Values.replicaCount | default 1 }}
tag: {{ .Values.image.tag | default .Chart.AppVersion }}
```

`default` triggers on any "empty" value: `""`, `0`, `false`, `nil`, empty list, empty map. That means `{{ .Values.enabled | default true }}` can never be false. For booleans use `hasKey` or `ternary` (see below).

`required` stops rendering with your message when the value is empty:

```yaml
host: {{ required "A valid .Values.db.host is required (set with --set db.host=...)" .Values.db.host }}
```

Failure output:

```
Error: execution error at (webapp/templates/deployment.yaml:20:20): A valid .Values.db.host is required (set with --set db.host=...)
```

`fail` aborts unconditionally: `{{ fail "do not use the legacy mode" }}`.

#### YAML helpers: `toYaml`, `nindent`, `indent`

Passing a whole block (resources, tolerations, affinity) from `values.yaml` into a manifest is the number-one use of these two.

```yaml
# values.yaml
resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

```yaml
# templates/deployment.yaml
        containers:
          - name: app
            image: nginx
            resources:
              {{- toYaml .Values.resources | nindent 14 }}
```

- `toYaml` converts a value (map/list) into YAML text
- `nindent N` = add a newline, then indent every line by N spaces
- `indent N` = indent every line by N spaces (no leading newline)

`nindent 14` must equal the column where the children should start. Count spaces of the key (`resources:` is at 12 spaces), children go 2 deeper = 14.

Empty value gotcha: with `resources: {}` the output is `resources:` followed by `{}`, which is valid. With `resources:` omitted (nil) `toYaml` yields `null`, rendering `resources: null`, also valid but ugly. Guard with `with`:

```yaml
            {{- with .Values.resources }}
            resources:
              {{- toYaml . | nindent 14 }}
            {{- end }}
```

Related: `toJson`, `toPrettyJson`, `fromYaml`, `fromJson`, `toToml`.

```yaml
data:
  settings.json: {{ .Values.settings | toJson | quote }}
```

#### Encoding and hashing

| Function | Use |
|---|---|
| `b64enc` / `b64dec` | Secret `data:` values: `{{ .Values.password \| b64enc }}` |
| `sha256sum` | Fingerprint of a string |
| `randAlphaNum 16` | Random string (changes on every render! see pitfalls) |

**Config checksum annotation**: Pods do not restart when a ConfigMap changes. Add a hash of the rendered ConfigMap into the Pod template annotations. When the ConfigMap changes, the hash changes, the Pod template changes, and Kubernetes rolls the Deployment.

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

#### `include`

`include "name" context` renders a named template and returns the result as a **string**, so it can be piped.

```yaml
labels:
  {{- include "webapp.labels" . | nindent 4 }}
```

#### `tpl`

`tpl` renders a **string** as a template. This allows users to put template expressions inside `values.yaml`.

```yaml
# values.yaml
host: "{{ .Release.Name }}.example.com"
```

```yaml
# templates/ingress.yaml
- host: {{ tpl .Values.host . | quote }}
```

Without `tpl` the output would be the literal text `{{ .Release.Name }}.example.com`. `tpl` takes two arguments: the string and the context. Use it sparingly; it hurts readability and lets values run arbitrary template logic.

#### Types, emptiness, and `ternary`

```yaml
{{ typeOf .Values.port }}                 # float64, string, bool, []interface {}, map[string]interface {}
{{ kindOf .Values.port }}
{{ kindIs "string" .Values.port }}        # true / false
{{ kindIs "map" .Values.labels }}
{{ empty .Values.x }}                     # true for "", 0, false, nil, [] and {}
{{ hasKey .Values "foo" }}                # key exists, even if its value is false
{{ ternary "yes" "no" .Values.enabled }}  # if enabled then "yes" else "no"
{{ coalesce .Values.a .Values.b "fallback" }}   # first non-empty
```

Numbers from YAML arrive as `float64`. `{{ .Values.port }}` prints `8080`, but very large numbers print in scientific notation (`1e+06`). Quote them or use `int64` / `toString`:

```yaml
value: {{ .Values.bigNumber | int64 | quote }}
```

Type conversions: `int`, `int64`, `float64`, `toString`, `toStrings`, `atoi`.

#### Lists and dicts

```yaml
{{- $ports := list 80 443 8080 }}
{{- $labels := dict "app" "web" "tier" "frontend" }}
{{- $labels = merge $labels (dict "env" "prod") }}
{{ first $ports }} {{ last $ports }} {{ has 443 $ports }}
{{ keys $labels | sortAlpha }}
{{ get $labels "app" }}
{{- $_ := set $labels "extra" "1" }}
```

`set` returns the dict; assign to `$_` to throw the result away (otherwise the dict prints). Math: `add`, `sub`, `mul`, `div`, `mod`, `max`, `min`, `ceil`, `floor`, `round`.

Comparison functions are **prefix** form (function first): `eq .Values.env "prod"`, `ne`, `lt`, `le`, `gt`, `ge`. Logic: `and`, `or`, `not`. Careful: `eq` and `lt` need compatible types; comparing a string to an int is an error: `error calling eq: incompatible types for comparison`.

#### `lookup`

`lookup "v1" "Secret" "my-ns" "my-secret"` queries the live cluster during install/upgrade. It returns an empty map when the object does not exist, and **always** returns empty under `helm template` and `--dry-run` (client-side). Typical use: generate a random password only if the Secret does not exist yet so it stays stable across upgrades.

```yaml
{{- $existing := lookup "v1" "Secret" .Release.Namespace "db-secret" }}
{{- $pw := "" }}
{{- if $existing }}
{{- $pw = index $existing.data "password" | b64dec }}
{{- else }}
{{- $pw = randAlphaNum 24 }}
{{- end }}
```

### Named templates and `_helpers.tpl`

Files under `templates/` whose name starts with `_` are **not rendered** as manifests; they hold reusable pieces. By convention the file is `_helpers.tpl`.

Define a named template:

```yaml
{{/* templates/_helpers.tpl */}}

{{/* Expand the name of the chart. */}}
{{- define "webapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/* Create a fully qualified app name (max 63 chars, DNS safe). */}}
{{- define "webapp.fullname" -}}
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

{{/* Selector labels: must NEVER change after first install. */}}
{{- define "webapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "webapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/* Common labels. */}}
{{- define "webapp.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
{{ include "webapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

Use them:

```yaml
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
```

Naming convention: **prefix every template name with the chart name** (`webapp.fullname`). Named templates are **global**: they are shared across the chart and all its subcharts. If two charts both define `labels`, one silently overwrites the other. Prefixing avoids collisions.

#### `include` versus `template`

| | `template "name" .` | `include "name" .` |
|---|---|---|
| Output | Written directly into the output | Returned as a string |
| Pipe to other functions | **No** | Yes (`\| nindent 4`, `\| quote`, `\| sha256sum`) |
| Name from variable | No (must be a literal) | Yes |

`template` cannot be piped, so you cannot fix its indentation. Because YAML needs exact indentation, **always use `include`** in Helm charts.

#### Passing context

```yaml
{{ include "webapp.labels" . }}                  # whole root context
{{ include "mylib.port" .Values.service }}        # a sub-tree
{{ include "mylib.item" (dict "item" $x "root" $) }}   # several values via dict
```

Inside the template, `.item` and `.root.Release.Name` reach them. This `dict` trick is how helpers receive more than one argument.

#### `define` whitespace

Write `{{- define "x" -}}` (trim on both sides) and `{{- end }}` so the defined text has no stray leading/trailing blank lines.

### Conditionals

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

Full form:

```yaml
{{- if eq .Values.env "prod" }}
replicas: 5
{{- else if eq .Values.env "staging" }}
replicas: 2
{{- else }}
replicas: 1
{{- end }}
```

#### Truthiness

A pipeline is **false** when the value is: boolean `false`, number `0`, empty string `""`, `nil`/null, empty list, empty map. Everything else is true.

Pitfalls: the string `"false"` is **true** (non-empty string). `--set enabled=false` is parsed as boolean false (fine). `--set-string enabled=false` is the string "false" = true. Also, `{{ if .Values.a.b.c }}` fails if `.Values.a.b` is nil, with `nil pointer evaluating interface {}.c`. Guard with `and` or use `with`/`dig`:

```yaml
{{- if and .Values.a .Values.a.b .Values.a.b.c }}
{{- $v := dig "a" "b" "c" "" .Values.AsMap }}
```

`and`/`or` short-circuit in Helm 3 (Go 1.18+), so the guarded `and` form is safe.

#### `with`

`with` is "if not empty, and rebind `.`":

```yaml
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

If `.Values.nodeSelector` is empty or missing, the whole block, including the key `nodeSelector:`, vanishes. This is the standard idiom for optional blocks (`tolerations`, `affinity`, `nodeSelector`, `annotations`).

`with` supports `else`:

```yaml
{{- with .Values.persistence }}
...
{{- else }}
emptyDir: {}
{{- end }}
```

Remember: inside `with` the dot is rebound; use `$` for root: `{{ $.Release.Name }}`.

### Loops with `range`

#### Over a list

```yaml
# values.yaml
extraEnv:
  - name: LOG_LEVEL
    value: debug
  - name: REGION
    value: eu-west
```

```yaml
env:
  {{- range .Values.extraEnv }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}
```

Inside `range` the dot is each element. For a list of plain strings:

```yaml
# values.yaml: hosts: [a.example.com, b.example.com]
{{- range .Values.hosts }}
- host: {{ . | quote }}
{{- end }}
```

#### Over a map

```yaml
# values.yaml
env:
  LOG_LEVEL: debug
  REGION: eu-west
```

```yaml
env:
  {{- range $key, $value := .Values.env }}
  - name: {{ $key }}
    value: {{ $value | quote }}
  {{- end }}
```

Maps iterate in **sorted key order**, so output is deterministic.

#### With index

```yaml
{{- range $i, $host := .Values.hosts }}
- id: {{ $i }}
  host: {{ $host }}
{{- end }}
```

#### Empty-aware loops

`range ... else` runs the `else` branch when the collection is empty:

```yaml
{{- range .Values.hosts }}
- {{ . }}
{{- else }}
- localhost
{{- end }}
```

#### Numeric loop

```yaml
{{- range until 3 }}
- worker-{{ . }}        # 0, 1, 2
{{- end }}
```

Also `untilStep 0 10 2`, `seq`. `break` and `continue` exist since Helm 3.? with Go 1.18 (`{{ break }}`, `{{ continue }}`) but are rarely needed.

#### Scope trap in loops

```yaml
{{- range .Values.hosts }}
- name: {{ $.Release.Name }}-{{ . }}     # $ for root, . for the element
{{- end }}
```

Writing `.Release.Name` here fails: the dot is a string. Save the root first when nesting loops: `{{- $root := . }}`.

### Generating several resources from one file

`---` between documents plus `range`:

```yaml
{{- range .Values.queues }}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ $.Release.Name }}-{{ .name }}
data:
  size: {{ .size | quote }}
{{- end }}
```

### NOTES.txt

`templates/NOTES.txt` is rendered after install/upgrade and printed to the user. It is a template but is not a Kubernetes manifest.

```
{{- if .Values.ingress.enabled }}
App URL: http://{{ (index .Values.ingress.hosts 0).host }}
{{- else }}
Run:
  kubectl port-forward -n {{ .Release.Namespace }} svc/{{ include "webapp.fullname" . }} 8080:{{ .Values.service.port }}
  Then open http://127.0.0.1:8080
{{- end }}
```

Show it again any time with `helm get notes RELEASE`.

### Values precedence

Highest wins:

1. `--set`, `--set-string`, `--set-file`, `--set-json` (last one on the command line wins)
2. `-f` / `--values` files (later `-f` overrides earlier)
3. Parent chart `values.yaml` overrides for a subchart
4. Chart's own `values.yaml`

Maps **merge** key by key. **Lists are replaced** entirely, not merged. Setting `--set image.tag=2.0` leaves `image.repository` from `values.yaml` untouched; setting `--set 'hosts[0]=x'` replaces the whole list with one element at index 0 (other indexes are dropped only if the list was shorter; indexes you do not set stay). To remove a default key use `--set key=null`.

`--set` syntax cheat sheet:

| Goal | Syntax |
|---|---|
| nested key | `--set image.tag=1.2` |
| list item | `--set 'hosts[0]=a.example.com'` |
| escape dot | `--set 'ingress.annotations.nginx\.ingress\.kubernetes\.io/ssl-redirect=true'` |
| comma list | `--set 'tolerations={a,b}'` (braces) |
| force string | `--set-string version=1.10` |
| from file | `--set-file config=./app.conf` |

Inspect the merge result: `helm get values RELEASE -a` (all, including defaults) or `helm template ... --debug`.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `helm template RELEASE CHART` | Render locally, print YAML (no cluster needed) | `helm template demo ./webapp` |
| `helm template ... --debug` | Render and print even if YAML is invalid, show computed values | `helm template demo ./webapp --debug` |
| `helm template ... --show-only FILE` | Render only one template | `helm template demo ./webapp --show-only templates/deployment.yaml` |
| `helm template ... -f F --set k=v` | Render with overrides | `helm template demo ./webapp -f prod.yaml --set replicaCount=4` |
| `helm template ... --namespace NS` | Set `.Release.Namespace` | `helm template demo ./webapp -n shop` |
| `helm template ... --kube-version V` | Fake cluster version for `.Capabilities` | `helm template demo ./webapp --kube-version 1.30.0` |
| `helm lint CHART` | Static checks of chart and templates | `helm lint ./webapp --strict` |
| `helm lint ... -f F` | Lint with a values file | `helm lint ./webapp -f prod.yaml` |
| `helm install --dry-run --debug` | Render through the server-side flow, print, do not create | `helm install demo ./webapp --dry-run --debug` |
| `helm install ... --dry-run=server` | Validate against the live API server (admission, `lookup` works) | `helm install demo ./webapp --dry-run=server` |
| `helm get values RELEASE -a` | Values the release actually used | `helm get values demo -a` |
| `helm get manifest RELEASE` | Rendered manifests stored in the release | `helm get manifest demo` |
| `helm get notes RELEASE` | Show NOTES.txt output again | `helm get notes demo` |
| `helm show values CHART` | Print default values of a chart | `helm show values ./webapp` |
| `helm create NAME` | Scaffold a starter chart | `helm create webapp` |

Template function quick table:

| Function | Meaning |
|---|---|
| `quote` | Wrap in double quotes (string-safe) |
| `default X V` | Use `V`, else `X` when `V` is empty |
| `required "msg" V` | Fail with `msg` if `V` is empty |
| `toYaml V \| nindent N` | Embed a map/list as YAML block, indented N |
| `include "t" CTX` | Render named template, returns string |
| `tpl STR CTX` | Render a string as a template |
| `printf "%s-%s" a b` | Format string |
| `trunc N` / `trimSuffix S` | Shorten / strip suffix (63-char names) |
| `sha256sum` | Hash (config checksum annotations) |
| `b64enc` / `b64dec` | Base64 for Secrets |
| `ternary A B COND` | `A` if cond else `B` |
| `empty` / `kindIs` / `typeOf` | Emptiness and type checks |
| `hasKey M "k"` | Key present in map |
| `dict` / `list` / `merge` / `set` / `get` | Build and combine collections |
| `until N` / `seq` | Number sequences for `range` |

## Examples / Walkthrough

All commands run from a scratch directory. We build one small chart called `webapp` by hand, then render it repeatedly with different values. No cluster is needed for steps 1-9 because `helm template` renders locally; step 10 installs it.

### Step 1: Create the chart skeleton

```bash
mkdir -p webapp/templates webapp/files
cd webapp
cat > Chart.yaml <<'EOF'
apiVersion: v2
name: webapp
description: Demo chart for the templating lesson
type: application
version: 0.1.0
appVersion: "1.27.0"
EOF
```

`appVersion` is a string describing the application (used here as the default image tag). `version` is the chart's own version.

### Step 2: Default values

```bash
cat > values.yaml <<'EOF'
replicaCount: 2

image:
  repository: nginx
  tag: ""            # empty = fall back to .Chart.AppVersion
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80

env:
  LOG_LEVEL: info
  REGION: eu-west

extraEnv: []

resources: {}

nodeSelector: {}

ingress:
  enabled: false
  host: "{{ .Release.Name }}.example.com"
  annotations: {}
EOF
```

### Step 3: Helpers

```bash
cat > templates/_helpers.tpl <<'EOF'
{{- define "webapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "webapp.fullname" -}}
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

{{- define "webapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "webapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "webapp.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
{{ include "webapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
EOF
```

### Step 4: ConfigMap using `.Files`, `range`, and `quote`

```bash
cat > files/app.conf <<'EOF'
server_tokens off;
client_max_body_size 5m;
EOF

cat > templates/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
data:
  app.conf: |
{{ .Files.Get "files/app.conf" | indent 4 }}
  {{- range $key, $value := .Values.env }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
EOF
```

### Step 5: Deployment using `toYaml`, `with`, `range`, checksum, `default`

```bash
cat > templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
          envFrom:
            - configMapRef:
                name: {{ include "webapp.fullname" . }}
          {{- if .Values.extraEnv }}
          env:
            {{- range .Values.extraEnv }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          {{- end }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
EOF
```

### Step 6: Service and conditional Ingress

```bash
cat > templates/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      name: http
  selector:
    {{- include "webapp.selectorLabels" . | nindent 4 }}
EOF

cat > templates/ingress.yaml <<'EOF'
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  rules:
    - host: {{ tpl .Values.ingress.host . | quote }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "webapp.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
EOF
```

### Step 7: NOTES.txt

```bash
cat > templates/NOTES.txt <<'EOF'
Release {{ .Release.Name }} (revision {{ .Release.Revision }}) in namespace {{ .Release.Namespace }}
{{- if .Release.IsInstall }}
Fresh install of {{ .Chart.Name }}-{{ .Chart.Version }}.
{{- else }}
Upgraded to {{ .Chart.Name }}-{{ .Chart.Version }}.
{{- end }}
{{- if .Values.ingress.enabled }}
URL: http://{{ tpl .Values.ingress.host . }}
{{- else }}
kubectl port-forward -n {{ .Release.Namespace }} svc/{{ include "webapp.fullname" . }} 8080:{{ .Values.service.port }}
{{- end }}
EOF
cd ..
```

### Step 8: Lint and render

```bash
helm lint ./webapp
# ==> Linting ./webapp
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed

helm template demo ./webapp
```

Expected Deployment excerpt (note the `1.27.0` tag came from `.Chart.AppVersion` via `default`):

```yaml
spec:
  replicas: 2
  template:
    metadata:
      annotations:
        checksum/config: 4f3c...
    spec:
      containers:
        - name: webapp
          image: "nginx:1.27.0"
```

Render a single file:

```bash
helm template demo ./webapp --show-only templates/configmap.yaml
```

Output:

```yaml
---
# Source: webapp/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-webapp
...
data:
  app.conf: |
    server_tokens off;
    client_max_body_size 5m;
  LOG_LEVEL: "info"
  REGION: "eu-west"
```

### Step 9: Customise with values files and `--set`

```bash
cat > prod.yaml <<'EOF'
replicaCount: 4
image:
  tag: "1.27.2"
env:
  LOG_LEVEL: warn
extraEnv:
  - name: FEATURE_X
    value: "on"
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi
nodeSelector:
  disktype: ssd
ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
EOF

helm template demo ./webapp -f prod.yaml --set replicaCount=6 --set env.REGION=us-east
```

Checks to perform by reading output:

- `replicas: 6`: `--set` beat `prod.yaml` (4), which beat `values.yaml` (2)
- `LOG_LEVEL: "warn"` and `REGION: "us-east"`: maps merge key by key
- `resources:` block appears only now, because `with` skipped the empty default
- `nodeSelector` appears only now for the same reason
- `ingress.yaml` renders only with `enabled: true`, host `demo.example.com` via `tpl`
- the `checksum/config` annotation **changed** because the ConfigMap content changed

Compare the empty-value behaviour: run `helm template demo ./webapp | grep -c resources` (zero lines) and again with `-f prod.yaml`.

### Step 10: Dry run against the cluster, then install

```bash
helm install demo ./webapp --dry-run --debug | head -60     # also prints USER-SUPPLIED and COMPUTED VALUES
helm install demo ./webapp -f prod.yaml --create-namespace -n shop
helm get values demo -n shop -a
helm get manifest demo -n shop | head -30
helm get notes demo -n shop
kubectl -n shop get deploy,svc,ingress,cm
```

Upgrade to see the config-checksum roll:

```bash
helm upgrade demo ./webapp -f prod.yaml -n shop --set env.LOG_LEVEL=debug
kubectl -n shop rollout status deploy/demo-webapp
kubectl -n shop get rs        # a new ReplicaSet appeared because the annotation hash changed
```

Clean up: `helm uninstall demo -n shop`.

### Step 11: Break it on purpose (debugging drill)

1. **Typo in a value path.** Change `.Values.image.repository` to `.Values.image.repo` in `deployment.yaml`.
   `helm template` renders `image: ":1.27.0"`: no error, wrong output. Typos in a *leaf* key render empty. Typos in a *parent* key error out (next case).
2. **Nil pointer.** Change to `.Values.imag.repository`.
   `Error: ... executing "webapp/templates/deployment.yaml:27:27: executing ... nil pointer evaluating interface {}.repository`. The parent `imag` is nil, so asking for a child fails.
3. **Indentation.** Change `nindent 12` to `nindent 10` in the `resources` block and render with `-f prod.yaml`. Output:
   `Error: YAML parse error on webapp/templates/deployment.yaml: error converting YAML to JSON: yaml: line 38: mapping values are not allowed in this context`. Use `helm template --debug` to print the broken YAML and count columns.
4. **Whitespace.** Remove the `-` from `{{- end }}` after `range` in `configmap.yaml`: a blank line appears between data keys. Harmless but ugly; removing `-` from `{{- if` can instead glue keys together.
5. **Scope.** Inside `range .Values.extraEnv` write `{{ .Release.Name }}`. Error: `can't evaluate field Release in type interface {}`. Fix with `$.Release.Name`.

## Common Pitfalls

- **`nil pointer evaluating interface {}.X`** — you asked for a child of a key that does not exist (`.Values.a.b` when `a` is missing). Check spelling, make sure the parent exists in `values.yaml` (give empty defaults such as `ingress: {}`), or guard with `if`, `with`, `dig`, or `default (dict) .Values.a`.
- **Dot rebinding inside `with` / `range`** — `.Release` and `.Values` vanish. Use `$.Release.Name` or save `{{- $root := . }}` before the block.
- **`#` does not disable template code** — `# {{ .Values.x }}` still executes and may error. Use `{{/* ... */}}`.
- **Wrong `nindent` number** — YAML parse error or fields landing under the wrong parent. Count the spaces of the parent key and add 2. Use `helm template --debug` to see the broken output.
- **`indent` instead of `nindent`** — first line glued to the key (`resources: limits:`) or lost newline. After a key with a colon use `nindent`.
- **Missing `quote`** — `1.10` becomes `1.1`, `on` becomes boolean `true`, `0123` becomes octal number. Quote env values, labels, annotations, tags.
- **`default` with booleans or zero** — `default true .Values.flag` always yields true because `false` counts as empty. Use `ternary`, `hasKey`, or `kindIs "bool"`.
- **`-}}` eating needed newline** — glued lines such as `name: fooversion: 1`. Use `{{-` for control actions and avoid `-}}` on value actions.
- **Forgetting `{{- end }}`** — `Error: parse error ... unexpected EOF`. Match every `if`/`with`/`range`/`define` with an `end`.
- **`template` instead of `include`** — can't pipe to `nindent`, output has wrong indentation. Always `include`.
- **Unprefixed named templates** (`{{ define "labels" }}`) — global namespace collisions with subcharts. Prefix with the chart name.
- **Changing selector labels after first install** — `spec.selector` is immutable; upgrade fails with `field is immutable`. Keep `selectorLabels` minimal and never put version or chart version in them.
- **`randAlphaNum` in a Secret** — new password on every upgrade, Pods break. Use `lookup` to reuse the existing value, or take the value from `values`.
- **`lookup` in `helm template`** — always returns empty without a cluster. Test lookup logic with `--dry-run=server`.
- **Pods not restarting after ConfigMap change** — no checksum annotation. Add `checksum/config: ... | sha256sum` to the Pod template.
- **List override surprise** — `-f` and `--set` **replace** lists; they never append. Re-supply the whole list.
- **Comma in `--set` values** — `--set a=b,c` is parsed as two keys. Escape `\,` or use `--set-string` / a values file.
- **`tpl` on untrusted input** — values can run template functions. Only use `tpl` where the chart owner controls the values.
- **Quoting `{{ }}` in values.yaml** — a value `host: {{ .Release.Name }}.x` without quotes is invalid YAML. Quote it and render with `tpl`.
- **Large integers printed as `1e+06`** — numbers parsed as float64. Use `int64` or quote in values.
- **Trusting `helm lint` alone** — lint checks syntax, not whether the API server will accept the objects. Follow with `helm install --dry-run=server`.

## FAQ

**Q: What template language does Helm use?**
A: Go's `text/template` language plus the Sprig function library and a few Helm functions (`include`, `tpl`, `toYaml`, `required`, `lookup`). Rendering happens on the client machine where `helm` runs.

**Q: What is the difference between `{{ }}` and `{{- -}}`?**
A: `{{ }}` keeps surrounding whitespace. `{{-` removes whitespace and newlines **before** the action, `-}}` removes them **after**. Needed because YAML is indentation-sensitive.

**Q: What is the difference between `.` and `$`?**
A: `.` is the current context and changes inside `with`, `range`, and named templates. `$` is always the root context from the start of the template. Use `$.Values` or `$.Release.Name` inside loops.

**Q: Why does `{{ .Release.Name }}` fail inside my `range` loop?**
A: Inside `range` the dot is the current list item, not the root. Write `{{ $.Release.Name }}`, or store the root in a variable before the loop: `{{- $root := . }}` and then `{{ $root.Release.Name }}`.

**Q: What does `toYaml | nindent 4` do?**
A: `toYaml` turns a map or list into YAML text. `nindent 4` adds a newline and indents every line by four spaces. Together they embed blocks like `resources` or `tolerations` from `values.yaml` into the manifest at the right indentation.

**Q: `indent` or `nindent`?**
A: `nindent` = newline + indent, use after a `key:` on its own line. `indent` has no leading newline, use when the text starts on the same line or when the template already has a line break (as with `.Files.Get ... | indent 4` placed on its own line).

**Q: How do I make a value optional?**
A: Wrap it in `with`: `{{- with .Values.x }} key: {{ . }} {{- end }}`. When the value is empty the whole block, key included, disappears.

**Q: How do I make a value mandatory?**
A: `{{ required "message" .Values.some.value }}`. Install fails immediately with your message when it is empty.

**Q: What is `default` for?**
A: `{{ .Values.x | default "fallback" }}` uses the fallback when `x` is empty (`""`, `0`, `false`, nil, empty list/map). Careful with booleans; `false` is "empty".

**Q: What does `quote` do and when do I need it?**
A: It wraps the value in double quotes so YAML reads it as a string. Use for env var values, labels, annotations and image tags. Without it `on`, `no`, `1.10` or `0123` may be parsed as booleans or numbers.

**Q: What is `_helpers.tpl`?**
A: A file in `templates/` whose name begins with `_`; Helm does not render it as a manifest. It stores reusable named templates (`define`) such as names and labels. Any file starting with `_` works, `_helpers.tpl` is just the convention.

**Q: `include` versus `template`?**
A: `template` writes directly to output and cannot be piped. `include` returns a string that can be piped to `nindent`, `quote`, or `sha256sum`. Always use `include`.

**Q: Why prefix named templates with the chart name?**
A: Named templates are global across the chart and all its subcharts. `webapp.labels` cannot clash with `redis.labels`, but a bare `labels` from two charts would overwrite each other.

**Q: Why do helpers truncate names to 63 characters?**
A: Kubernetes object names used in labels and DNS must be 63 characters or fewer. The helper does `trunc 63 | trimSuffix "-"` so the cut never leaves a trailing dash.

**Q: What does `tpl` do?**
A: Renders a string as a template using a context: `{{ tpl .Values.host . }}`. It lets users write `{{ .Release.Name }}` inside `values.yaml` strings. Without `tpl` that text is printed literally.

**Q: How do I check the type of a value?**
A: `{{ typeOf .Values.x }}`, or test with `{{ kindIs "string" .Values.x }}`. Values from YAML numbers are `float64`, lists are `[]interface {}`, maps are `map[string]interface {}`.

**Q: How do I see what Helm will render without installing?**
A: `helm template RELEASE CHART [-f file] [--set k=v]`. Add `--debug` to print even invalid YAML, `--show-only templates/x.yaml` to render one file.

**Q: What is the difference between `helm template` and `helm install --dry-run`?**
A: `helm template` is fully local, with no cluster and no `lookup`. `helm install --dry-run --debug` goes through the install flow and prints computed values and manifests but still does not call the API server for objects. `--dry-run=server` sends the objects to the API server for validation without persisting.

**Q: What does `helm lint` check?**
A: Chart structure, `Chart.yaml` fields, template syntax, and that templates render to valid YAML with default (or supplied) values. It does not validate against a real cluster, and it only warns about missing icons or deprecated keys. `--strict` turns warnings into failures.

**Q: Which value wins, `values.yaml`, `-f`, or `--set`?**
A: `--set` beats `-f` files (later files beat earlier ones), which beat the chart's `values.yaml`. Maps merge; lists are replaced whole.

**Q: Can I remove a default value via `--set`?**
A: Yes, set it to null: `--set someKey=null`. It deletes the key from the merged values.

**Q: Why did my Pods not restart after I changed the ConfigMap?**
A: Kubernetes only rolls a Deployment when its Pod template changes. Add `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}` to the Pod template annotations.

**Q: Why does `{{ if .Values.enabled }}` stay true when I set `enabled="false"`?**
A: The string `"false"` is non-empty, therefore true. `--set enabled=false` gives a real boolean. `--set-string` or a quoted YAML value gives a string. Use real booleans.

**Q: How do I loop over a map?**
A: `{{ range $key, $value := .Values.env }} ... {{ $key }} ... {{ $value }} {{ end }}`. Keys come out in sorted order.

**Q: How do I loop N times?**
A: `{{ range until 3 }}` yields 0, 1, 2. `untilStep START END STEP` for custom steps.

**Q: How can a template accept several arguments?**
A: Pass a `dict`: `{{ include "x" (dict "root" $ "item" . "port" 80) }}` and read `.root`, `.item`, `.port` inside the named template.

**Q: How can I fail on purpose?**
A: `{{ fail "message" }}` aborts rendering; `required` is the conditional version. Use them to reject invalid combinations of values.

**Q: What are `.Capabilities` and when do I use them?**
A: Information about the target cluster: Kubernetes version and available API versions. Use `semverCompare` or `.Capabilities.APIVersions.Has` to pick the right `apiVersion` or skip resources the cluster does not support. Under `helm template` pass `--kube-version` and `--api-versions` to simulate.

**Q: What is the difference between `.Chart.Version` and `.Chart.AppVersion`?**
A: `version` is the chart package version (changes when templates change). `appVersion` is the version of the application inside (often the default image tag). They are independent.

**Q: Where do the files for `.Files.Get` live?**
A: Anywhere inside the chart except `templates/` and except paths excluded by `.helmignore`. The conventional folder is `files/`.

**Q: How do I add notes shown after install?**
A: Create `templates/NOTES.txt`; it is rendered as a template and printed after install and upgrade. Show it again with `helm get notes RELEASE`.

**Q: How do I debug "YAML parse error"?**
A: Run `helm template ... --debug`; Helm prints the broken rendered YAML. Look at the reported line and count spaces. Most causes are a wrong `nindent` number or missing newline.

**Q: Is it safe to put passwords in `values.yaml` and render a Secret?**
A: They end up in the release history stored in the cluster (as a Helm release Secret) and in Git if you commit the file. Prefer external secret managers (Lesson 43) and reference them from the chart.

## Practice / Exercise

Use the `webapp` chart built in the walkthrough.

### Core

1. Run `helm lint ./webapp` and `helm template demo ./webapp`. Identify the source line in `values.yaml` for each of: replica count, image tag, `LOG_LEVEL`.
2. Render with `--show-only templates/deployment.yaml` and confirm that no `resources:`, `nodeSelector:` or `env:` block appears. Explain why (`with`/`if` on empty values).
3. Create `staging.yaml` with `replicaCount: 3`, `env.LOG_LEVEL: debug`, and `service.type: NodePort`. Render with `-f staging.yaml` and verify with `grep`. Then add `--set replicaCount=5` and confirm the result.
4. Add an `extraEnv` list with two entries to `staging.yaml` and verify the `env:` block renders. Convert the `env` map in `values.yaml` to also add a third key and see it appear in the ConfigMap.
5. Break the chart on purpose using each of the five drills in Step 11. For each, write the exact error message and the one-line fix.
6. Add a template `templates/serviceaccount.yaml` that is created only when `.Values.serviceAccount.create` is true. Name it with `include "webapp.fullname" .`. Add the default to `values.yaml`.
7. Install the chart into a namespace with `helm install`, run `helm get values -a`, `helm get manifest`, and `helm get notes`. Upgrade with a changed `env.LOG_LEVEL` and show that a new ReplicaSet is created (checksum annotation).

### Stretch

1. Add a `required` check that fails when `ingress.enabled=true` and `ingress.host` is empty. Test the error message.
2. Write a named template `webapp.envList` that takes a map and prints it as `- name:`/`value:` entries, and use it in the Deployment via `include ... | nindent`.
3. Add a `.Files.Glob "files/*.conf"` + `AsConfig` to the ConfigMap so any file dropped in `files/` is included automatically.
4. Add a `semverCompare`-based branch that renders a `PodDisruptionBudget` with `policy/v1` only when the cluster is 1.21 or newer. Test with `helm template --kube-version 1.20.0` and `1.30.0`.
5. Use `lookup` to keep a generated Secret password stable across upgrades. Prove with `--dry-run=server` and an actual upgrade that the value does not change.
6. Use `tpl` to let `ingress.annotations` values reference `{{ .Release.Name }}`, then explain what breaks without `tpl`.
7. Print `{{ typeOf .Values.replicaCount }}`, `{{ typeOf .Values.service.port }}` and `{{ typeOf .Values.env }}` in a temporary template and explain the results.

Deliverable: the `webapp` chart directory, `staging.yaml`, and a text file with the error messages and fixes from Core task 5.

## Further Reading

- Helm docs, Chart Template Guide: https://helm.sh/docs/chart_template_guide/
- Helm docs, Template Function List: https://helm.sh/docs/chart_template_guide/function_list/
- Helm docs, Built-in Objects: https://helm.sh/docs/chart_template_guide/builtin_objects/
- Helm docs, Named Templates: https://helm.sh/docs/chart_template_guide/named_templates/
- Helm docs, Debugging Templates: https://helm.sh/docs/chart_template_guide/debugging/
- Helm docs, Chart Best Practices (templates, values, labels): https://helm.sh/docs/chart_best_practices/
- Sprig function documentation: https://masterminds.github.io/sprig/
- Go `text/template` package: https://pkg.go.dev/text/template

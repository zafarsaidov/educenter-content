# Lesson 36: Pods, Deployments & Manifest Files

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lesson 34 (cluster architecture, kubeconfig), Lesson 35 (kubectl get/describe/apply/delete, namespaces, labels and selectors, `-o yaml`, `kubectl explain`), Lessons 30-31 (containers, images, Dockerfile)

## Learning Objectives

By end of lesson student can:
- Explain what a Pod is and why Kubernetes schedules Pods instead of bare containers
- Write a Pod manifest with `containers` (name, image, ports, resources), `restartPolicy`, `imagePullPolicy`
- Explain the Deployment → ReplicaSet → Pod ownership chain and what each controller is responsible for
- Write a Deployment manifest with `replicas`, `selector`, `template`, and `strategy`
- Configure RollingUpdate with `maxSurge` and `maxUnavailable`, and explain when to choose Recreate
- Scale a Deployment with `kubectl scale` and describe the HPA idea (concept only)
- Roll out, watch, inspect, pause, resume, and undo a rollout with `kubectl rollout`
- Generate manifests with `--dry-run=client -o yaml`, read the structure line by line, and preview changes with `kubectl diff`
- Debug a Pod stuck in `ImagePullBackOff`, `CrashLoopBackOff`, `Pending`

## Topics

- Pod spec: containers (name, image, ports, resources), restartPolicy, imagePullPolicy
- Deployment: replicas, selector, template, strategy (RollingUpdate, Recreate), maxSurge
- ReplicaSet: relationship to Deployment; `kubectl scale` (`--replicas`); HPA concept
- Rollouts: `kubectl rollout status/history/undo`; pause and resume rollout
- Writing YAML manifests: `--dry-run=client -o yaml`; structure walkthrough; `kubectl diff`

## Concepts

### Why Pods and not containers

Kubernetes does not run containers directly. The smallest deployable unit is a **Pod**: one or more containers that share:

- **Network namespace** — one IP address per Pod; containers in the same Pod talk over `localhost` and must not use the same port.
- **Storage** — volumes declared at Pod level can be mounted into several containers of the Pod (volume details come in lesson 41).
- **Lifecycle** — containers in a Pod are scheduled together onto one node, started together, and die together when the Pod is deleted.
- **IPC and (optionally) PID namespace.**

Most Pods have exactly **one** container. Multi-container Pods exist for tightly-coupled helpers:

| Pattern | Idea | Example |
|---|---|---|
| Sidecar | Helper container extends main app | log shipper next to app |
| Init container | Runs to completion BEFORE app containers start | wait for DB, download config |
| Ambassador / adapter | Proxy or format converter | local proxy to remote service |

Rule of thumb: put two containers in the same Pod only if they must scale together and share localhost/volumes. Otherwise use two Pods.

Analogy with Docker: `docker run` → one container. Pod → one "logical host" that may contain several containers. Pod is **ephemeral**: when it dies it is never "repaired" — a new Pod with a new name and new IP replaces it. Never rely on a Pod IP staying stable (stable addressing is solved by Services in lesson 37).

### Anatomy of every Kubernetes manifest

Every object manifest has four top-level fields:

```yaml
apiVersion: v1          # API group/version the object type belongs to
kind: Pod               # object type
metadata:               # identity: name, namespace, labels, annotations
  name: web
spec:                   # DESIRED STATE — what you want
```

After creation the API server adds a fifth field, `status` — **ACTUAL STATE**, written by controllers and kubelet. You never write `status`. The reconciliation loop (lesson 34) works to make `status` match `spec`.

apiVersion cheat sheet for this lesson:

| kind | apiVersion |
|---|---|
| Pod | `v1` |
| Deployment | `apps/v1` |
| ReplicaSet | `apps/v1` |
| Namespace | `v1` |

Find the right pair with `kubectl api-resources | grep -i deployment` (lesson 35).

### Pod spec fields

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  restartPolicy: Always          # Always (default) | OnFailure | Never
  containers:
    - name: nginx                # unique inside the Pod
      image: nginx:1.27          # always pin a tag
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80      # informational only — does NOT open or publish anything
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 250m
          memory: 128Mi
```

Field by field:

**`containers[].name`** — DNS-label style name; used by `kubectl logs -c <name>` and `kubectl exec -c <name>` in multi-container Pods.

**`containers[].image`** — same image references as Docker (`registry/repo:tag` or `@sha256:digest`). No tag means `:latest`.

**`containers[].ports[].containerPort`** — documentation for humans and tools. Even without it the container still listens on its port; with it Services and tools can refer to the port by `name`. It does NOT work like `-p` in Docker (there is no publishing at Pod level).

**`containers[].resources`** — basic form only today: `requests` = what the scheduler reserves for the container when picking a node; `limits` = hard ceiling enforced at runtime (CPU is throttled, memory overuse → OOMKilled). Units: CPU in cores or millicores (`500m` = half a core), memory in `Mi`/`Gi` (power of 2). How QoS classes, LimitRange, and ResourceQuota build on these is lesson 45; here you only need the syntax and to know that **requests drive scheduling**.

**`restartPolicy`** (Pod level, applies to all containers):

| Value | Behaviour | Use for |
|---|---|---|
| `Always` (default) | Restart container whenever it exits (any code) | long-running services; the only value allowed in Deployments |
| `OnFailure` | Restart only on non-zero exit | batch tasks |
| `Never` | Never restart | one-shot debugging Pods |

Restarts use exponential back-off: 10s, 20s, 40s … capped at 5 minutes. A container that keeps crashing shows `CrashLoopBackOff` — that is the back-off wait state, not a separate failure type. Note: the kubelet restarts the *container* in place on the same node; it does not create a new Pod.

**`imagePullPolicy`**:

| Value | Meaning |
|---|---|
| `IfNotPresent` | Pull only if image is not cached on the node (default when tag is anything but `:latest`) |
| `Always` | Ask registry every time container starts (default when tag is `:latest` or omitted) |
| `Never` | Never pull; fail if image not on node (used with locally loaded images) |

Gotcha: if you re-push the *same* tag (`:v1`) with new content, nodes with `IfNotPresent` keep running the old cached image. Use unique immutable tags (git SHA, semver) — the lesson-33 tagging discipline.

**Other useful Pod fields** (short): `command` and `args` override image ENTRYPOINT and CMD respectively; `env` with plain `name`/`value` pairs sets environment variables; `workingDir`; `terminationGracePeriodSeconds` (default 30); `serviceAccountName` (lesson 47).

```yaml
    - name: demo
      image: busybox:1.36
      command: ["sh", "-c"]          # replaces ENTRYPOINT
      args: ["echo hello; sleep 3600"]  # replaces CMD
      env:
        - name: APP_MODE
          value: "demo"
```

Mapping to Docker: `command` ↔ `ENTRYPOINT`, `args` ↔ `CMD`. Mixing this up is a classic mistake.

### Pod lifecycle and phases

`kubectl get pod` shows STATUS. Pod **phase** values: `Pending` (accepted, not all containers running — scheduling or image pulling), `Running`, `Succeeded` (all containers exited 0 and won't restart), `Failed`, `Unknown` (node lost). Common STATUS strings that are not phases but container states / reasons:

| STATUS | Meaning | First check |
|---|---|---|
| `Pending` | no node chosen or volumes/images not ready | `kubectl describe pod` → Events (`FailedScheduling`: insufficient CPU/memory?) |
| `ContainerCreating` | node chosen, pulling image / setting up | wait, then Events |
| `ImagePullBackOff` / `ErrImagePull` | cannot pull image: wrong name/tag, private registry, no network | typo in image, `describe` Events |
| `CrashLoopBackOff` | container starts then exits repeatedly | `kubectl logs <pod> --previous` |
| `OOMKilled` (in lastState) | memory limit exceeded | raise limit or fix leak |
| `Completed` | exited 0 (normal for Jobs / `Never` Pods) | — |
| `Error` | exited non-zero and not restarting | logs |
| `Terminating` | being deleted, waiting for grace period | `--force --grace-period=0` only as last resort |

Termination sequence on `kubectl delete pod`: Pod marked Terminating → SIGTERM sent to PID 1 of each container → wait up to `terminationGracePeriodSeconds` (30s) → SIGKILL. Apps must handle SIGTERM (same lesson as Docker `stop`, lesson 30).

### Why you should almost never create bare Pods

A bare Pod has no controller. If the node dies, the Pod is gone forever. If you delete it, nobody recreates it. You cannot scale it, roll out a new image, or roll back. Bare Pods are for quick tests and debugging. For anything real, use a **controller**: Deployment (stateless apps, this lesson), StatefulSet (lesson 41), DaemonSet, Job/CronJob.

### ReplicaSet

A **ReplicaSet** keeps a stable number of identical Pods running. It is itself a control loop:

```
desired replicas (spec.replicas)  vs  Pods matching selector  →  create or delete Pods to match
```

It finds "its" Pods through `selector.matchLabels`, and stamps new Pods from `template`. It tracks ownership through `metadata.ownerReferences` on each Pod (check with `kubectl get pod <p> -o yaml | grep -A5 ownerReferences`).

You normally do **not** write ReplicaSets yourself. Reason: a ReplicaSet cannot do rolling updates — change its template and existing Pods are not replaced. That is what a Deployment adds.

### Deployment

A **Deployment** manages ReplicaSets, and ReplicaSets manage Pods:

```
Deployment web
 ├── ReplicaSet web-7d9f8c6b5   (revision 2, replicas 3)  ← current
 │     ├── Pod web-7d9f8c6b5-abcde
 │     ├── Pod web-7d9f8c6b5-fghij
 │     └── Pod web-7d9f8c6b5-klmno
 └── ReplicaSet web-5b8c7d6f4   (revision 1, replicas 0)  ← kept for rollback
```

Naming: Pod name = `<deployment>-<replicaset-hash>-<random5>`. The hash is derived from the Pod template — **change anything in `spec.template` and you get a new ReplicaSet**; change `replicas` and you do not.

What a Deployment gives you on top of ReplicaSet:
1. **Rolling updates** — replace Pods gradually with zero (or controlled) downtime.
2. **Rollback** — scale old ReplicaSet back up.
3. **Revision history** — old ReplicaSets retained (`revisionHistoryLimit`, default 10).
4. **Pause/resume** — batch several edits into one rollout.
5. **Scaling** — manual or via HPA.

#### Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: web              # MUST match template.metadata.labels
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:                 # a Pod template: same as a Pod without apiVersion/kind
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              memory: 128Mi
```

The three pieces that must line up:

| Piece | Role |
|---|---|
| `spec.selector.matchLabels` | which Pods this Deployment owns |
| `spec.template.metadata.labels` | labels stamped on created Pods |
| (they must match) | otherwise API rejects: `selector does not match template labels` |

`spec.selector` is **immutable** after creation (apps/v1). Choose labels carefully. Labels on Pods may contain more keys than the selector, never fewer.

#### Update strategies

**RollingUpdate** (default) — gradually replaces old Pods with new ones. Controlled by two knobs (each an absolute number or a percentage of `replicas`):

| Field | Default | Meaning |
|---|---|---|
| `maxSurge` | 25% (rounded up) | how many Pods ABOVE `replicas` may exist during the update |
| `maxUnavailable` | 25% (rounded down) | how many Pods BELOW `replicas` may be unavailable during the update |

Both can't be 0 at once (no progress possible).

Worked examples with `replicas: 4`:

| maxSurge | maxUnavailable | Behaviour |
|---|---|---|
| 1 | 0 | Start 1 new Pod (5 total); when it is Ready, kill 1 old. Never below 4 serving. Safest, slowest, needs spare capacity for +1 Pod |
| 0 | 1 | Kill 1 old first (3 serving), then start 1 new. No extra capacity needed, brief capacity dip |
| 2 | 1 | Aggressive: up to 6 Pods, at least 3 serving. Fast |
| 100% | 0 | Blue/green-like: start 4 new, switch, delete 4 old. Needs double capacity |
| 25% / 25% | (defaults) | 1 surge, 1 unavailable for 4 replicas |

"Available" depends on Pod **readiness**. Today a Pod counts as ready as soon as its containers run; real readiness gating comes with probes in lesson 45 — until then rolling updates can't tell "container started" from "app can serve traffic". Mention this to students; do not teach probe syntax now.

**Recreate** — kill ALL old Pods, then start all new Pods. Causes downtime. Use when two versions must never run together (exclusive DB schema migration, single-writer file lock, license-limited software).

```yaml
  strategy:
    type: Recreate
```

Other rollout-related spec fields:

| Field | Meaning |
|---|---|
| `minReadySeconds` | Pod must be ready this long before counted available (slows rollout, catches quick crashes) |
| `progressDeadlineSeconds` | default 600; after this with no progress, rollout is marked failed (`ProgressDeadlineExceeded`) — it does NOT auto-rollback |
| `revisionHistoryLimit` | how many old ReplicaSets to keep (default 10); set 0 and `rollout undo` is impossible |
| `paused` | true while rollout is paused |

#### What triggers a rollout

Only changes to `spec.template`: image, env, command, labels *inside* template, resources, etc. Changing `replicas`, `strategy`, labels on the Deployment metadata does **not** trigger one. To force a restart without changing anything: `kubectl rollout restart deployment/web` (it stamps an annotation `kubectl.kubernetes.io/restartedAt` into the template, producing a new ReplicaSet).

### Scaling and HPA concept

Manual scaling changes `spec.replicas`:

```bash
kubectl scale deployment web --replicas=5
```

Better practice in Git-driven workflows: edit `replicas` in the manifest and `kubectl apply`. If both scale-by-command and apply are used, the next `apply` reverts to the file's value (declarative file wins).

**HPA — Horizontal Pod Autoscaler (concept only today).** A controller that watches a metric (typically average CPU utilization of Pods, relative to their CPU **request**) and adjusts `replicas` of a Deployment between a min and max:

```
CPU 90% of request, target 50%  →  desired = ceil(current × 90/50)  →  more replicas
CPU 20% of request              →  fewer replicas (after stabilization window)
```

Requirements: metrics-server installed in the cluster, and CPU **requests** set on the Pods (otherwise there is no percentage to compute). A quick imperative example for instructor demo only if metrics-server exists: `kubectl autoscale deployment web --min=2 --max=6 --cpu-percent=60`. Detailed HPA authoring is outside this lesson; just remember who owns `replicas` when an HPA exists — the HPA, so remove `replicas` from the manifest in that case, else apply fights the autoscaler.

### Rollouts

Every template change creates a new **revision**.

```bash
kubectl set image deployment/web nginx=nginx:1.27        # imperative image change
kubectl rollout status deployment/web                     # blocks until done or failed
kubectl rollout history deployment/web                    # list revisions
kubectl rollout history deployment/web --revision=2       # details of one revision
kubectl rollout undo deployment/web                       # back to previous revision
kubectl rollout undo deployment/web --to-revision=1       # back to specific revision
kubectl rollout pause deployment/web                      # freeze
kubectl rollout resume deployment/web                     # unfreeze → single combined rollout
kubectl rollout restart deployment/web                    # rolling restart, same image
```

**CHANGE-CAUSE column.** `rollout history` shows `<none>` unless the annotation `kubernetes.io/change-cause` is set on the Deployment. Best practice: set it in the manifest or with `kubectl annotate deployment web kubernetes.io/change-cause="upgrade nginx to 1.27"` right after the change. The old `--record` flag is deprecated; don't teach it.

**Pause/resume use case.** Pause, then make several changes (image, env, resources) — nothing happens — then resume and get ONE rollout instead of three. Also useful to canary by hand: pause mid-rollout and inspect the new Pods before continuing.

**Undo semantics.** `undo` does not "go back in time" in the cluster; it re-applies the old revision's template as a **new** revision number (revision 3 can be a copy of revision 1). A failed rollout (bad image) keeps old Pods serving as long as `maxUnavailable` allowed it — this is exactly why `maxUnavailable: 0` is a safety net.

### Writing YAML manifests

Nobody types manifests from memory. Workflow:

1. **Generate a skeleton**:
   ```bash
   kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > web.yaml
   kubectl run tmp --image=busybox:1.36 --dry-run=client -o yaml -- sleep 3600 > pod.yaml
   ```
   `--dry-run=client` builds the object locally and prints it, no API call. `--dry-run=server` sends it to the API server, runs validation and admission, but does not persist — catches more errors.
2. **Edit** — remove noise (`creationTimestamp: null`, empty `status: {}`, `resources: {}`), add resources and labels.
3. **Look up fields** with `kubectl explain deployment.spec.strategy.rollingUpdate` (recursive: `kubectl explain pod.spec --recursive | less`).
4. **Preview** with `kubectl diff -f web.yaml` — shows what `apply` would change against live cluster state (exit code 0 = no diff, 1 = diff found, >1 = error).
5. **Apply** with `kubectl apply -f web.yaml`.

YAML rules that bite:
- Indentation is **spaces only**, 2 per level, never tabs.
- A list item starts with `- `; keys of the same item align after the dash.
- Strings that look like other types must be quoted: `"true"`, `"8080"`, `"1.0"`, `"no"`. Env var values **must** be strings.
- `---` on its own line separates multiple documents in one file; `kubectl apply -f` handles all of them in order.
- Comments with `#`.

#### Structure walkthrough (read it with students)

```yaml
apiVersion: apps/v1            # (1) group/version: apps group, v1
kind: Deployment               # (2) what kind of object
metadata:                      # (3) identity
  name: web                    #     unique name in namespace
  namespace: demo              #     optional; default namespace otherwise
  labels: {app: web}           #     labels on the Deployment object itself
  annotations:
    kubernetes.io/change-cause: "initial release"
spec:                          # (4) desired state of Deployment
  replicas: 3                  #     how many Pods
  selector:                    #     which Pods are mine
    matchLabels: {app: web}
  strategy: {...}              #     how to roll updates
  template:                    # (5) Pod template — blueprint
    metadata:
      labels: {app: web}       #     MUST satisfy selector
    spec:                      # (6) Pod spec — everything from the Pod section above
      containers: [...]
```

Count the nesting: Deployment `spec` → `template` → Pod `spec` → `containers`. Two `spec` keys in one file confuses beginners: the first belongs to the Deployment, the second to the Pod.

#### apply vs create

| Command | Behaviour |
|---|---|
| `kubectl create -f x.yaml` | imperative; fails with `AlreadyExists` if present |
| `kubectl apply -f x.yaml` | declarative; creates or patches; remembers last-applied config in an annotation (client-side apply) for 3-way merge |
| `kubectl replace -f x.yaml` | full replace; fails if missing |

Use `apply` for everything you keep in Git.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl run` | Create a bare Pod quickly | `kubectl run web --image=nginx:1.27 --port=80` |
| `kubectl run ... --rm -it` | Throwaway debug Pod | `kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh` |
| `kubectl create deployment` | Create Deployment imperatively | `kubectl create deployment web --image=nginx:1.27 --replicas=3` |
| `kubectl apply -f` | Create/update from manifest | `kubectl apply -f web.yaml` |
| `kubectl diff -f` | Preview change vs live state | `kubectl diff -f web.yaml` |
| `kubectl get pods -o wide` | Pods with node and Pod IP | `kubectl get pods -l app=web -o wide` |
| `kubectl get rs` | List ReplicaSets | `kubectl get rs -l app=web` |
| `kubectl describe pod` | Events, state, reasons | `kubectl describe pod web-abc12` |
| `kubectl logs` | Container logs | `kubectl logs web-abc12 -c nginx --previous -f` |
| `kubectl exec -it` | Shell inside Pod | `kubectl exec -it web-abc12 -- sh` |
| `kubectl scale` | Change replicas | `kubectl scale deploy web --replicas=5` |
| `kubectl set image` | Change image (triggers rollout) | `kubectl set image deploy/web nginx=nginx:1.27` |
| `kubectl rollout status` | Wait for rollout | `kubectl rollout status deploy/web --timeout=120s` |
| `kubectl rollout history` | List revisions | `kubectl rollout history deploy/web --revision=2` |
| `kubectl rollout undo` | Roll back | `kubectl rollout undo deploy/web --to-revision=1` |
| `kubectl rollout pause/resume` | Batch changes | `kubectl rollout pause deploy/web` |
| `kubectl rollout restart` | Rolling restart | `kubectl rollout restart deploy/web` |
| `kubectl annotate` | Set change-cause | `kubectl annotate deploy/web kubernetes.io/change-cause="bump to 1.27"` |
| `kubectl create ... --dry-run=client -o yaml` | Generate manifest | `kubectl create deploy web --image=nginx --dry-run=client -o yaml` |
| `kubectl explain` | Field documentation | `kubectl explain deploy.spec.strategy` |
| `kubectl delete` | Remove object(s) | `kubectl delete -f web.yaml` / `kubectl delete pod web --grace-period=0 --force` |
| `kubectl get deploy -w` | Watch live changes | `kubectl get pods -w` |
| `kubectl wait` | Block until condition | `kubectl wait --for=condition=Available deploy/web --timeout=90s` |

## Examples / Walkthrough

Use a scratch namespace so cleanup is one command.

### 1. First Pod, imperative and declarative

```bash
kubectl create namespace lesson36
kubectl config set-context --current --namespace=lesson36

# imperative
kubectl run web --image=nginx:1.27 --port=80
kubectl get pod web -o wide            # note IP and NODE
kubectl describe pod web | sed -n '/Events/,$p'
```

Read Events top to bottom: `Scheduled` → `Pulling` → `Pulled` → `Created` → `Started`. This is the lifecycle in action.

Generate the same as YAML:

```bash
kubectl delete pod web
kubectl run web --image=nginx:1.27 --port=80 --labels=app=web --dry-run=client -o yaml > pod.yaml
cat pod.yaml
kubectl apply -f pod.yaml
```

Test it without a Service: curl the Pod IP from another Pod.

```bash
POD_IP=$(kubectl get pod web -o jsonpath='{.status.podIP}')
kubectl run curl --rm -it --image=curlimages/curl:8.8.0 --restart=Never -- curl -s http://$POD_IP | head -5
```

Look inside:

```bash
kubectl exec -it web -- sh -c 'hostname; cat /etc/nginx/conf.d/default.conf | head -5; ls /usr/share/nginx/html'
kubectl logs web
```

### 2. Prove a bare Pod is not self-healing

```bash
kubectl delete pod web
kubectl get pods         # gone, nothing recreates it
```

### 3. Failure modes of a Pod (do live, 10 min)

```bash
# ImagePullBackOff — typo in tag
kubectl run bad --image=nginx:9.99.99
kubectl get pod bad -w                  # ErrImagePull → ImagePullBackOff
kubectl describe pod bad | tail -8      # 'manifest unknown' message
kubectl delete pod bad

# CrashLoopBackOff — container exits right away
kubectl run crash --image=busybox:1.36 -- sh -c 'echo boom; exit 1'
kubectl get pod crash -w                # RESTARTS grows, back-off grows
kubectl logs crash --previous
kubectl delete pod crash

# Pending — impossible request
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hungry
spec:
  containers:
    - name: c
      image: nginx:1.27
      resources:
        requests:
          memory: 1000Gi
EOF
kubectl describe pod hungry | grep -A3 Events   # FailedScheduling: Insufficient memory
kubectl delete pod hungry
```

### 4. restartPolicy in action

```bash
kubectl run once --image=busybox:1.36 --restart=Never -- sh -c 'echo done'
kubectl get pod once                      # Completed
kubectl logs once
kubectl delete pod once
```

### 5. Multi-container Pod (sidecar) — quick look

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date; sleep 5; done"]
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f two.yaml
kubectl get pod two                       # READY 2/2
kubectl logs two -c app --tail=3
kubectl exec -it two -c sidecar -- sh -c 'wget -qO- localhost 2>&1 | head -1'   # shared localhost
kubectl delete pod two
```

### 6. First Deployment

```bash
kubectl create deployment web --image=nginx:1.26 --replicas=3 --dry-run=client -o yaml > web.yaml
```

Edit `web.yaml` to match the full manifest from the Concepts section (strategy `maxSurge: 1`, `maxUnavailable: 0`, resources, change-cause annotation). Then:

```bash
kubectl diff -f web.yaml                   # whole object appears as new (+)
kubectl apply -f web.yaml
kubectl get deploy,rs,pods -l app=web
kubectl rollout status deploy/web
```

Show the chain:

```bash
kubectl get pod -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{"  owner="}{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}{"\n"}{end}'
kubectl get rs -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{"  owner="}{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}{"\n"}{end}'
```

### 7. Self-healing and scaling

```bash
kubectl get pods -l app=web -w &           # watch in background (kill with: kill %1)
kubectl delete pod $(kubectl get pod -l app=web -o name | head -1)
# ReplicaSet creates a replacement within seconds
kubectl scale deploy web --replicas=5
kubectl get pods -l app=web
kubectl scale deploy web --replicas=3
```

Now the declarative trap:

```bash
kubectl scale deploy web --replicas=10
kubectl diff -f web.yaml                   # shows replicas: 10 → 3
kubectl apply -f web.yaml                  # reverts to 3
```

### 8. Rolling update, watch it happen

Terminal A:

```bash
kubectl get pods -l app=web -w
```

Terminal B:

```bash
kubectl set image deploy/web nginx=nginx:1.27
kubectl annotate deploy/web kubernetes.io/change-cause="nginx 1.26 -> 1.27" --overwrite
kubectl rollout status deploy/web
kubectl get rs -l app=web              # old RS at 0, new RS at 3
kubectl rollout history deploy/web
```

Observe with `maxSurge: 1, maxUnavailable: 0`: at most 4 Pods, never fewer than 3 Running.

### 9. Failed rollout and rollback

```bash
kubectl set image deploy/web nginx=nginx:does-not-exist
kubectl rollout status deploy/web --timeout=30s    # times out
kubectl get pods -l app=web                        # 3 old Running + 1 new ImagePullBackOff
kubectl rollout history deploy/web
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
kubectl get pods -l app=web
```

Key teaching point: old Pods kept serving because `maxUnavailable: 0`.

### 10. Pause / resume to batch changes

```bash
kubectl rollout pause deploy/web
kubectl set image deploy/web nginx=nginx:1.27-alpine
kubectl set resources deploy/web -c nginx --limits=memory=192Mi
kubectl get rs -l app=web                  # nothing new happened
kubectl rollout resume deploy/web          # ONE rollout with both changes
kubectl rollout status deploy/web
kubectl rollout history deploy/web
```

### 11. Recreate strategy

```bash
kubectl patch deploy web --type merge -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
kubectl set image deploy/web nginx=nginx:1.26
kubectl get pods -l app=web -w             # all terminate first, then new start
```

`rollingUpdate: null` is required: leaving rollingUpdate params with `type: Recreate` is rejected by validation.

### 12. Clean up

```bash
kubectl delete namespace lesson36
kubectl config set-context --current --namespace=default
```

## Common Pitfalls

- **Selector does not match template labels** — API error `selector does not match template labels`. Make `spec.selector.matchLabels` a subset of `spec.template.metadata.labels`.
- **Changing an immutable selector** — `field is immutable` on apply. Delete and recreate the Deployment (Pods go down), or create new Deployment name and switch traffic.
- **Using `:latest` or no tag** — `imagePullPolicy` becomes `Always`, versions are untraceable, rollbacks are meaningless (previous revision has same tag). Pin versions.
- **Re-pushing same tag with `IfNotPresent`** — nodes keep stale image. Use immutable tags or `rollout restart` plus `Always`.
- **`containerPort` thinking it publishes a port** — it is documentation. Nothing is reachable from outside just by declaring it (Services, lesson 37).
- **Confusing `command`/`args` with Docker `CMD`/`ENTRYPOINT`** — `command` replaces ENTRYPOINT, `args` replaces CMD. Setting only `command` discards the image CMD.
- **Tabs in YAML or wrong indentation** — `error converting YAML to JSON`. Use spaces; run `kubectl apply --dry-run=server -f`.
- **Unquoted env values** — `value: 8080` or `value: true` is parsed as number/bool and rejected (`cannot unmarshal number into Go struct field EnvVar.value of type string`). Quote them.
- **Editing `replicas` in file while an HPA owns it** — they fight. Remove `replicas` from manifest when an autoscaler exists.
- **Scaling with `kubectl scale` and then forgetting the file** — next `apply` silently reverts the scale.
- **No resource requests** — scheduler can pile everything on one node; HPA cannot compute percentages. Always set requests (details lesson 45).
- **Assuming `rollout status` success means the app works** — until readiness probes exist (lesson 45) "ready" only means "container process started".
- **Expecting automatic rollback on failed rollout** — Kubernetes only marks `ProgressDeadlineExceeded`; you must `rollout undo` (or let CI/CD do it).
- **`revisionHistoryLimit: 0`** — saves clutter, kills rollback ability.
- **Deleting a Pod managed by a Deployment to "stop" it** — it comes right back. Scale to 0 or delete the Deployment.
- **`kubectl delete pod --force --grace-period=0` as habit** — skips graceful shutdown, can leave stateful workloads inconsistent; use only for stuck `Terminating`.
- **Forgetting namespace** — `kubectl get pods` shows nothing because objects are in another namespace; use `-n` or set the context namespace.
- **Deployment with `restartPolicy: Never`/`OnFailure`** — rejected; Deployments require `Always`.

## FAQ

**Q: What is a Pod, in one sentence?**
A: The smallest deployable unit in Kubernetes: one or more containers that share one network namespace (one IP) and volumes and are always scheduled onto the same node together.

**Q: Why not just run containers directly like with Docker?**
A: Kubernetes needs a unit that can hold helper containers sharing network and storage, and a stable abstraction for schedulers and controllers. Pod is that unit; the container runtime is only an implementation detail beneath it.

**Q: Should I put my app and its database in the same Pod?**
A: No. They scale and fail independently. Same Pod only for tightly-coupled helpers (log shipper, proxy sidecar) that need localhost or shared volumes.

**Q: What is the difference between a Pod, a ReplicaSet and a Deployment?**
A: Pod = running containers. ReplicaSet = keeps N identical Pods alive. Deployment = manages ReplicaSets to give rolling updates, rollback, and history. You create Deployments; Deployments create ReplicaSets; ReplicaSets create Pods.

**Q: Why does my Deployment have so many ReplicaSets?**
A: Each template change makes a new ReplicaSet; old ones are kept (scaled to 0) so you can roll back. `revisionHistoryLimit` (default 10) caps how many are kept.

**Q: What is the random suffix in Pod names?**
A: `<deployment>-<pod-template-hash>-<random>`. The middle part identifies the ReplicaSet (template version); the last part makes each Pod unique. Names change on every replacement, so never hard-code Pod names.

**Q: What does `maxSurge` mean?**
A: Maximum number of Pods allowed above the desired `replicas` count during a rolling update. `maxSurge: 1` with `replicas: 3` → up to 4 Pods temporarily. It is also expressed as a percentage (rounded up).

**Q: And `maxUnavailable`?**
A: Maximum number of Pods that may be unavailable below the desired count during the update (percentage rounded down). `0` means capacity never drops, which requires `maxSurge > 0`.

**Q: When use Recreate instead of RollingUpdate?**
A: When old and new versions cannot coexist (incompatible DB schema, exclusive lock on a resource, single-instance license). Accept downtime in exchange for guaranteed isolation.

**Q: How do I roll back?**
A: `kubectl rollout undo deployment/<name>` (previous revision) or `--to-revision=N`. Check `kubectl rollout history` first.

**Q: Why does `rollout history` show `<none>` in CHANGE-CAUSE?**
A: The `kubernetes.io/change-cause` annotation is not set. Add it with `kubectl annotate` or in the manifest. `--record` is deprecated.

**Q: What changes trigger a new rollout?**
A: Only changes inside `spec.template` (image, env, command, resources, template labels…). Scaling and strategy edits do not.

**Q: How do I restart all Pods of a Deployment without changing anything?**
A: `kubectl rollout restart deployment/<name>` — does a rolling restart by touching an annotation in the template.

**Q: Why use pause/resume?**
A: To apply several edits and get one rollout instead of one per edit; also to inspect a partial rollout (manual canary) before continuing.

**Q: What does `--dry-run=client` do and how is it different from `server`?**
A: `client` renders the object locally, no API call, so only syntax of the command is checked. `server` sends the request to the API server for full validation and admission, but nothing is stored. Use `client -o yaml` to generate manifests and `server` to validate them.

**Q: What does `kubectl diff` tell me?**
A: What `kubectl apply` would change in the live object. Exit code 0: no differences; 1: differences; >1: error. Great as a CI safety check.

**Q: `create` or `apply`?**
A: `apply` for anything managed from files. `create` is fine for one-offs and for generating YAML with `--dry-run`.

**Q: My Pod is in CrashLoopBackOff. First step?**
A: `kubectl logs <pod> --previous` (logs of the crashed instance), then `kubectl describe pod <pod>` for exit code and reason (`OOMKilled`, `Error`), checking `Last State`.

**Q: ImagePullBackOff — what to check?**
A: Image name/tag typo, tag not existing, private registry without credentials, node cannot reach the registry. `kubectl describe pod` Events show the exact error.

**Q: Pod stays Pending.**
A: Almost always scheduling: `kubectl describe pod` → `FailedScheduling` with reason (insufficient CPU/memory, node taints, unbound volumes). Check requests vs node capacity with `kubectl describe node`.

**Q: Does `containerPort` open the port?**
A: No. It is metadata. The container listens on whatever the app binds. Exposure to other Pods/outside is done with Services (next lesson).

**Q: How do two containers in one Pod talk to each other?**
A: Over `localhost:<port>` — they share one network namespace — so they must use different ports.

**Q: Can I change a running Pod's image?**
A: Yes, `spec.containers[].image` is one of the few mutable Pod fields (container restarts). Most other fields are immutable; recreate the Pod. For Deployments just edit the template.

**Q: How do I see what manifest a running object has?**
A: `kubectl get deploy web -o yaml` (includes `status` and defaults). To review before applying changes use `kubectl diff`.

**Q: Can I autoscale?**
A: Yes, with a HorizontalPodAutoscaler plus metrics-server and CPU requests set. HPA changes `replicas` for you; do not also hard-code `replicas` in the manifest.

**Q: What is a Pod IP, can I rely on it?**
A: Cluster-internal IP assigned from the Pod CIDR. It changes when the Pod is replaced, so do not rely on it. Stable addressing comes from Services (lesson 37).

**Q: Why did `rollout status` hang?**
A: New Pods are not becoming available (bad image, crash, Pending). Press Ctrl-C, check `kubectl get pods`, `describe`, `logs`, then fix or `undo`. Rollout times out after `progressDeadlineSeconds` (600s) and is marked failed but nothing rolls back automatically.

## Practice / Exercise

Work in your own namespace (`kubectl create ns <yourname>`).

### Core

1. Create a Pod `hello` from a manifest file: image `nginx:1.27`, label `app=hello`, containerPort 80, requests `cpu: 50m`, `memory: 32Mi`, limits `memory: 64Mi`. Verify with `kubectl get pod -o wide` and `kubectl describe pod`. Curl its IP from a temporary `curlimages/curl` Pod.
2. Delete `hello`; confirm it does not come back. Explain in one sentence why.
3. Generate a Deployment manifest named `shop` with `kubectl create deployment ... --dry-run=client -o yaml`: image `nginx:1.26`, 4 replicas. Clean it up (remove `creationTimestamp`, `status`), add strategy `RollingUpdate` with `maxSurge: 1`, `maxUnavailable: 0`, and a change-cause annotation. Run `kubectl diff`, then `apply`.
4. Delete one Pod of `shop` and watch the replacement. Then run the owner-chain jsonpath command and draw Deployment → ReplicaSet → Pods on paper.
5. Scale `shop` to 6 with `kubectl scale`, then edit the file back to 4 and `apply`. Explain what happened.
6. Update the image to `nginx:1.27` via `kubectl set image`, annotate the change-cause, watch with `kubectl get pods -w` in a second terminal. Count the maximum number of Pods that existed at once and match to maxSurge.
7. Roll out a broken image (`nginx:nope`). Prove the old Pods still serve, then `rollout undo`. Read `rollout history` and explain why revision numbers keep growing.
8. Pause the rollout, change image AND memory limit, resume; show that only one new ReplicaSet appeared.
9. Break three Pods deliberately (bad tag, exit 1 command, 1000Gi memory request). For each, name the STATUS and the command that revealed the cause.

### Stretch

1. Change `shop` to `Recreate`, update the image, and measure downtime: run `kubectl run curl --rm -it ...` loop hitting one Pod IP, or watch `kubectl get pods -w`. Compare against RollingUpdate with `maxSurge: 0, maxUnavailable: 1`.
2. Try `maxSurge: 100%, maxUnavailable: 0` with 4 replicas and watch the Pod count peak at 8.
3. Set `revisionHistoryLimit: 2`, do 5 image updates, show how many ReplicaSets remain and what `rollout undo --to-revision=1` says.
4. Write a Pod with an init container (`busybox`, `sleep 10`) before the main `nginx` container. Observe `Init:0/1` → `PodInitializing` → `Running` and read `kubectl describe pod`.
5. Use `kubectl explain deployment.spec.strategy --recursive` and `kubectl explain pod.spec.containers.imagePullPolicy` to write down defaults for every field you used.
6. Run `kubectl apply --dry-run=server -f` with an intentionally broken manifest (tab indentation, mismatched selector, string/number env value) and record the exact error messages.
7. Use `kubectl wait --for=condition=Available deployment/shop --timeout=60s` in a script that applies a manifest and fails if it never becomes available (CI-style).

## Further Reading

- Pods: https://kubernetes.io/docs/concepts/workloads/pods/
- Pod lifecycle: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- ReplicaSet: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
- Horizontal Pod Autoscaling: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- kubectl cheat sheet: https://kubernetes.io/docs/reference/kubectl/quick-reference/
- Managing resources / `kubectl diff`: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/
- Kubernetes API reference: https://kubernetes.io/docs/reference/kubernetes-api/

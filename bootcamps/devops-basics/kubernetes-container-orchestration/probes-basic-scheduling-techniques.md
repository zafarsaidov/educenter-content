# Lesson 45: Probes & Basic Scheduling Techniques

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-44 (cluster architecture, kubectl, Pods/Deployments/manifests, Services, Ingress/Gateway API, PV/PVC/StatefulSet, ConfigMap/Secret, ESO). Student can write Deployment + Service YAML, read `kubectl describe` events, and use `kubectl get -o jsonpath`.

## Learning Objectives

By end of lesson student can:
- Explain what kubelet does with a probe result and why a running container is not the same as a healthy container
- Write liveness, readiness and startup probes using `httpGet`, `exec`, `tcpSocket` and `grpc` handlers
- Tune `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `successThreshold`, `failureThreshold` and compute worst-case detection / startup time
- Show that a failing readiness probe removes a Pod from Service endpoints without restarting it
- Diagnose a CrashLoopBackOff caused by an over-aggressive liveness probe
- Set `requests` and `limits` for CPU and memory, with correct units (millicores, `Mi`, `Gi`)
- Explain CPU throttling versus memory `OOMKilled`, and why they behave differently
- Predict a Pod's QoS class (Guaranteed / Burstable / BestEffort) from its spec and verify it with `kubectl`
- Explain how requests drive scheduling and how QoS drives eviction order under node pressure
- Create a `LimitRange` (defaults, min, max) and a `ResourceQuota` (compute and object counts) and observe how they reject or modify Pods

## Topics

- Liveness probe: httpGet, exec, tcpSocket; initialDelaySeconds, periodSeconds, failureThreshold
- Readiness probe: difference from liveness, effect on Service endpoints (traffic gating)
- Startup probe: for slow-starting containers; failureThreshold × periodSeconds formula
- Resource requests and limits: CPU (millicores), memory (Mi/Gi), QoS classes (Guaranteed/Burstable/BestEffort)
- LimitRange: per-container defaults; ResourceQuota: per-namespace caps

## Concepts

### "Running" is not "healthy"

Kubernetes knows a container is **running** when its main process (PID 1) is alive. That says nothing about whether the application inside can serve requests. Typical failures that leave PID 1 alive:

- Deadlock: threads wait on each other forever, port is open, no response is ever sent
- Application still booting: JVM warm-up, DB migration, cache loading — process exists, port not yet listening
- Lost dependency: DB connection pool is dead, every request returns 500, process never exits
- Overload: app is alive but the request queue is full and it needs a break

Without more information Kubernetes would (a) send traffic to a Pod that cannot answer, and (b) never restart a hung Pod. **Probes** are how the application tells kubelet its real state.

A **probe** is a periodic check run by **kubelet** (on the node, not by the control plane) against a container. Three kinds, each answering a different question and each triggering a different action:

| Probe | Question | Action on failure |
|---|---|---|
| **startupProbe** | "Has the app finished starting?" | Kill container (restart per `restartPolicy`). While it runs, liveness and readiness are **disabled** |
| **livenessProbe** | "Is the app still alive (not wedged)?" | Kill container; kubelet restarts it (per `restartPolicy`) |
| **readinessProbe** | "Can the app take traffic right now?" | **Do not kill.** Mark Pod `NotReady`; remove its IP from Service endpoints |

Mnemonic: **liveness = restart me, readiness = stop sending me traffic, startup = give me time to boot.**

All three are optional and independent. Defaults when none is defined: container is considered always alive and always ready as long as the process runs.

### Probe handlers (how the check is performed)

Each probe uses exactly one handler:

| Handler | What kubelet does | Success when | Use for |
|---|---|---|---|
| `httpGet` | HTTP GET to `path` + `port` (optional `httpHeaders`, `scheme`) | Status code `200-399` | Web apps and APIs with a health endpoint |
| `tcpSocket` | Opens a TCP connection to `port` | Connection established | Databases, caches, anything without HTTP |
| `exec` | Runs a command **inside** the container | Exit code `0` | Custom checks: `pg_isready`, `redis-cli ping`, check a file exists |
| `grpc` | gRPC health checking protocol (`grpc.health.v1.Health`) against `port` | Response `SERVING` | gRPC services (stable since K8s 1.27) |

Notes:
- `httpGet` is sent **from the node (kubelet)**, to the Pod IP. The request is not made through the Service.
- `httpGet` `host` defaults to the Pod IP. Do not set it unless you know why.
- `exec` forks a process every period; a slow or heavy command costs CPU inside the container's cgroup.
- A `tcpSocket` check only proves something is listening, not that it works.
- `port` may be a number or a named container port (`name: http`).

### Probe timing parameters

Every probe accepts the same timing fields:

| Field | Default | Meaning |
|---|---|---|
| `initialDelaySeconds` | 0 | Wait this long after container start before the **first** probe |
| `periodSeconds` | 10 | How often to probe (min 1) |
| `timeoutSeconds` | 1 | Probe counts as failed if no answer within this time |
| `successThreshold` | 1 | Consecutive successes to flip from failed to success. **Must be 1 for liveness and startup**; only readiness may be higher |
| `failureThreshold` | 3 | Consecutive failures before action is taken (restart / NotReady) |
| `terminationGracePeriodSeconds` (probe-level) | pod value | Optional per-probe override of the kill grace time after a liveness/startup failure |

**Worst-case time to detect a dead app** (liveness):

```
detection ≈ initialDelaySeconds + failureThreshold × periodSeconds   (roughly, +timeout per attempt)
```

Example: `periodSeconds: 10`, `failureThreshold: 3` → app can be dead for about 30 s before it is killed.

**Time before a bad Pod gets traffic removed** (readiness): `failureThreshold × periodSeconds`. Defaults give about 30 s of failed requests before gating — tune lower (`periodSeconds: 5`, `failureThreshold: 2`) for critical paths.

### Liveness probe

Purpose: recover from states a restart can fix (deadlock, memory leak that wedged the app).

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

Design rules for the liveness endpoint:
- **Check only the app itself**, never external dependencies. If `/healthz` calls the database and the DB is down, every replica fails liveness, all restart at once, and you turn a DB outage into a full-app outage plus a restart storm.
- Keep it cheap and fast (no heavy queries).
- Do not use liveness to "wait for" something. That is the startup probe's job.
- If your app crashes on its own when broken (exits non-zero), you may not need a liveness probe at all.

### Readiness probe

Purpose: control **traffic**. Used when the app is temporarily unable to serve: warming up, waiting for a dependency, overloaded, draining.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
  successThreshold: 1
```

How the effect works (links to lesson 37):
1. kubelet runs the readiness probe and updates the Pod's `Ready` condition.
2. The EndpointSlice controller watches Pods. A Pod whose `Ready` is `False` has its address listed as **not ready**; Services and kube-proxy only route to ready addresses.
3. Ingress controllers / Gateway implementations that use Service endpoints stop sending traffic as well.
4. When the probe passes again (`successThreshold` times), the Pod is added back. **No restart happens.**

Readiness is also what makes **rolling updates safe** (lesson 36): a Deployment waits until a new Pod is `Ready` before it counts as available and before it removes an old Pod. Without a readiness probe a Pod is "ready" the moment its container starts, and a rollout can replace healthy Pods with ones that still boot.

Readiness endpoint design:
- May check dependencies (DB reachable, cache warm). This is the right place for that check.
- Careful when **all** replicas depend on the same backend: if it is down they all go NotReady and the Service has no endpoints (503 everywhere). Sometimes degrading gracefully is better than gating.

### Startup probe

Purpose: protect **slow-starting** containers from being killed by the liveness probe while they boot (legacy Java apps, large model loads, long DB migrations).

Without a startup probe you must set a huge `initialDelaySeconds` on liveness, which also delays failure detection for the entire lifetime of the Pod. The startup probe solves this: it runs **first**, liveness and readiness are held off until it succeeds once, then liveness takes over with tight settings.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 5
  failureThreshold: 60        # 60 × 5 s = 300 s to start
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3         # after startup: dead in 30 s
```

**Formula:**

```
maximum startup time = failureThreshold × periodSeconds
```

Example above: 60 × 5 = **300 s** for the app to come up. If it does not succeed within that window, kubelet kills the container. After the first success the startup probe never runs again for that container instance.

### Probe interaction summary

| State of container | startup | liveness | readiness |
|---|---|---|---|
| Just started, startup probe defined | running | paused | paused (Pod NotReady) |
| Startup succeeded | done | running | running |
| No startup probe | — | starts after `initialDelaySeconds` | starts after `initialDelaySeconds` |
| Liveness failing | — | container restarted | Pod NotReady during restart |
| Readiness failing only | — | fine | Pod removed from endpoints, **not** restarted |

Probe results and restarts are visible via `kubectl describe pod` (Events: `Unhealthy`, `Killing`) and `kubectl get pod` (`RESTARTS` column, `READY` column `0/1`).

### Resource requests and limits

Containers share node CPU and memory. Two numbers per resource per container:

| Field | Meaning | Used by |
|---|---|---|
| `requests` | Amount the container is **guaranteed**; the scheduler reserves it | **Scheduler** (placement) and cgroup weight |
| `limits` | Maximum the container may use | **kubelet / kernel** (enforcement via cgroups) |

```yaml
resources:
  requests:
    cpu: 250m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

#### Units

- **CPU:** measured in cores. `1` = one vCPU/core. `m` = millicores: `500m` = 0.5 core, `100m` = 0.1 core. Cannot go below `1m`. CPU is a **compressible** resource: it can be throttled without killing anything.
- **Memory:** bytes. Use binary suffixes `Ki`, `Mi`, `Gi` (powers of 1024). `128Mi` ≈ 134 MB. Beware `128M` (decimal, powers of 1000) and the lower-case `m` trap: `128m` means 0.128 **bytes**, valid syntax, silent bug. Memory is **incompressible**: it cannot be taken back without killing something.
- Ephemeral storage can also be requested/limited (`ephemeral-storage: 1Gi`); mentioned here for completeness.

#### How requests drive scheduling

The scheduler (lesson 34) filters nodes by **sum of requests of Pods already on the node ≤ node allocatable**. It looks at *requests only*, never at actual usage and never at limits. Consequences:
- A node can look idle (low real CPU) yet be "full" because its requests are all reserved. The next Pod stays `Pending` with event `0/3 nodes are available: 3 Insufficient cpu`.
- If you set no requests, the scheduler treats the Pod as needing nothing; many such Pods pile onto one node.
- Over-requesting wastes money; under-requesting causes noisy-neighbour trouble.

```bash
kubectl describe node <node> | sed -n '/Allocated resources/,/Events/p'
kubectl describe node <node> | grep -A8 "Allocatable"
```

Allocatable = capacity minus system and kubelet reservations and eviction thresholds.

#### What happens at the limit: CPU vs memory

| | CPU limit exceeded | Memory limit exceeded |
|---|---|---|
| Behaviour | **Throttled** by the CFS scheduler (container paused for part of each 100 ms period) | Kernel **OOM killer** kills the container process |
| Visible as | Slow responses, latency, no restart | `OOMKilled`, exit code **137**, restart, possibly `CrashLoopBackOff` |
| Fix | Raise limit / remove limit / optimise | Raise memory limit, fix the leak, tune the runtime heap |

Check an OOM:

```bash
kubectl describe pod web | grep -A6 "Last State"
#   Last State:     Terminated
#     Reason:       OOMKilled
#     Exit Code:    137
kubectl get pod web -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

Exit code 137 = 128 + 9 (SIGKILL). Do not confuse with a node-level eviction (see below).

Practice notes:
- JVM, Node.js, Go and Python runtimes size heaps from the cgroup limit (modern versions) or need flags (`-Xmx`, `--max-old-space-size`, `GOMEMLIMIT`). Keep heap below the container limit with headroom for non-heap memory.
- Many teams set **memory request = memory limit** (predictable, no surprise OOM from overcommit) and **CPU request without a CPU limit** (avoids throttling). That is an opinion, not a rule; know the trade-off.
- A container with a limit but no request gets `request = limit` automatically.

### QoS classes

Every Pod gets a **Quality of Service class**, computed from its containers' requests/limits and written to `status.qosClass`. It decides who is killed first under node pressure.

| Class | Rule | Eviction priority |
|---|---|---|
| **Guaranteed** | Every container has CPU **and** memory requests **and** limits, and `request == limit` for each | Last to be evicted (best protected) |
| **Burstable** | At least one container has a request or limit, but the Guaranteed rule is not met | Middle |
| **BestEffort** | **No** container has any request or limit | First to be evicted |

Quick decision: no resources anywhere → BestEffort. All set and equal → Guaranteed. Anything else → Burstable.

You cannot set QoS directly; you get it by how you write `resources`.

```bash
kubectl get pod web -o jsonpath='{.status.qosClass}{"\n"}'
kubectl get pods -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

#### Eviction (concept)

When a node runs low on memory, disk, or PIDs, **kubelet evicts Pods** to reclaim resources (node-pressure eviction). Order roughly: BestEffort and Burstable Pods **using more than their requests** first, ranked by Pod priority and usage over request; Guaranteed Pods last. Evicted Pods end in `Failed` with reason `Evicted` and are rescheduled by their controller elsewhere.

Distinguish three similar-looking events:

| Event | Who acts | Trigger | Marker |
|---|---|---|---|
| Container `OOMKilled` | Kernel cgroup OOM | Container passed its **own** memory limit | exit 137, `reason: OOMKilled` |
| Pod `Evicted` | kubelet | **Node** under memory/disk pressure | Pod `Failed`, `reason: Evicted` |
| Liveness kill | kubelet | Failed liveness probe | Events: `Liveness probe failed ... will be restarted` |

Node condition names: `MemoryPressure`, `DiskPressure`, `PIDPressure` (see `kubectl describe node`).

### LimitRange: per-container defaults and bounds

Problem: developers forget `resources`, giving BestEffort Pods; or request 64 Gi for a hello-world. **LimitRange** is a namespaced policy object that, for objects created **in that namespace**:

1. Sets **defaults** (`default` = limit, `defaultRequest` = request) when a container does not specify them
2. Enforces **min** and **max** per container (or Pod / PVC)
3. Can enforce a **maxLimitRequestRatio**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:            # applied as limits when missing
      cpu: 500m
      memory: 256Mi
    defaultRequest:     # applied as requests when missing
      cpu: 100m
      memory: 64Mi
    min:
      cpu: 50m
      memory: 32Mi
    max:
      cpu: "1"
      memory: 512Mi
```

Behaviour:
- Defaults are injected **at admission time** (the LimitRanger admission controller); the Pod object stored in etcd already contains them. `kubectl get pod -o yaml` shows them.
- A Pod violating min/max is **rejected** with `Forbidden: maximum cpu usage per Container is 1, but limit is 2`.
- LimitRange does **not** affect Pods that already exist; only new ones.
- Types: `Container`, `Pod` (sum of containers), `PersistentVolumeClaim` (min/max storage).
- Only `default`/`defaultRequest` fill gaps; if only a limit is set the request is set to the limit.

### ResourceQuota: per-namespace caps

LimitRange controls **one container**. **ResourceQuota** caps the **whole namespace**: total requests, total limits, and object counts. This is how you share one cluster between teams.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
    count/deployments.apps: "10"
```

Rules:
- When a quota on `requests.*` / `limits.*` is active, **every new Pod must declare those values** or it is rejected: `must specify limits.cpu for: web`. Pair with a LimitRange so defaults get injected and developers are not blocked.
- Exceeding a hard limit: the **create** is rejected with `exceeded quota: team-a-quota, requested: pods=1, used: pods=20, limited: pods=20`. Existing workloads keep running.
- A Deployment that cannot create Pods due to quota shows no error on `kubectl apply`; look at `kubectl describe rs` → Events (`FailedCreate ... exceeded quota`).
- `kubectl describe quota -n team-a` shows `Used` vs `Hard`.
- Other quota keys: `requests.storage`, `<storageclass>.storageclass.storage.k8s.io/requests.storage`, `count/<resource>.<group>`, `requests.ephemeral-storage`, `configmaps`, `secrets`, `replicationcontrollers`, `resourcequotas`, `services.nodeports`, `services.loadbalancers`.

**Scopes (brief):** `scopes` restrict which Pods a quota counts. `BestEffort` / `NotBestEffort` (by QoS), `Terminating` / `NotTerminating`, and `PriorityClass` (with `scopeSelector`). Example: allow at most 5 BestEffort Pods per namespace by scoping a `pods: "5"` quota to `BestEffort`.

#### LimitRange vs ResourceQuota

| | LimitRange | ResourceQuota |
|---|---|---|
| Scope | One container / Pod / PVC | Whole namespace total |
| Does | Defaults + min/max + ratio | Hard totals + object counts |
| Effect on missing resources | **Injects defaults** | **Rejects** the Pod (if quota covers that resource) |
| Typical pairing | Use together: LimitRange fills in, ResourceQuota caps the sum | |

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl describe pod <p>` | See probe config, `State`, `Last State`, Events (`Unhealthy`, `Killing`) | `kubectl describe pod web-7d9 \| less` |
| `kubectl get pods -w` | Watch READY / RESTARTS change live | `kubectl get pods -l app=web -w` |
| `kubectl get endpoints <svc>` | Show ready addresses behind a Service | `kubectl get endpoints web` |
| `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` | Per-address `ready` condition | `kubectl get endpointslices -l kubernetes.io/service-name=web -o yaml` |
| `kubectl logs <p> --previous` | Logs of the last crashed container | `kubectl logs web-7d9 --previous` |
| `kubectl get pod <p> -o jsonpath='{.status.qosClass}'` | Read QoS class | `kubectl get pod web -o jsonpath='{.status.qosClass}'` |
| `kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].restartCount}'` | Restart counter | same pattern |
| `kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'` | Why last restart happened | `OOMKilled` / `Error` |
| `kubectl set resources deploy/<d> --requests=... --limits=...` | Change resources imperatively | `kubectl set resources deploy/web --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi` |
| `kubectl top pod` / `kubectl top node` | Live usage (needs metrics-server) | `kubectl top pod -n team-a --containers` |
| `kubectl describe node <n>` | Capacity, Allocatable, Allocated resources, pressure conditions | `kubectl describe node worker-1` |
| `kubectl get limitrange,resourcequota -n <ns>` | List policies in namespace | `kubectl get limitrange,resourcequota -n team-a` |
| `kubectl describe quota <q> -n <ns>` | `Used` vs `Hard` | `kubectl describe quota team-a-quota -n team-a` |
| `kubectl describe limitrange <l> -n <ns>` | Effective defaults/min/max | `kubectl describe limitrange container-limits -n team-a` |
| `kubectl exec <p> -- <cmd>` | Test what an `exec` probe would run | `kubectl exec db-0 -- pg_isready -U postgres` |
| `kubectl rollout status deploy/<d>` | Rollout waits on readiness | `kubectl rollout status deploy/web` |

Probe field cheat sheet:

```yaml
<probeKind>:                  # startupProbe | livenessProbe | readinessProbe
  httpGet: {path: /x, port: 8080, scheme: HTTP, httpHeaders: [{name: X-Probe, value: "1"}]}
  # OR tcpSocket: {port: 5432}
  # OR exec: {command: ["sh", "-c", "pg_isready -U postgres"]}
  # OR grpc: {port: 9090, service: ""}
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 3
```

## Examples / Walkthrough

Lab assumption: a working cluster (kind/minikube/k3s) with a default StorageClass is not needed here. Use a fresh namespace.

```bash
kubectl create namespace probes-lab
kubectl config set-context --current --namespace=probes-lab
```

### Example 1: A Deployment with liveness + readiness

```yaml
# web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - {name: http, containerPort: 80}
        readinessProbe:
          httpGet: {path: /, port: http}
          periodSeconds: 3
          failureThreshold: 2
        livenessProbe:
          httpGet: {path: /, port: http}
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: {app: web}
  ports:
  - {port: 80, targetPort: http}
```

```bash
kubectl apply -f web.yaml
kubectl rollout status deploy/web
kubectl get pods -l app=web
kubectl get endpoints web          # 2 addresses
kubectl describe pod -l app=web | grep -E "Liveness|Readiness"
```

### Example 2: Prove readiness gates traffic (no restart)

Break the readiness path only in one Pod by making `/` return 404: delete the default page inside the container.

```bash
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- rm /usr/share/nginx/html/index.html

# watch: within ~6 s the Pod goes 0/1 but RESTARTS stays 0
kubectl get pods -l app=web -w
# in another terminal
kubectl get endpoints web          # only 1 address left
kubectl describe pod $POD | tail -n 8   # Events: Readiness probe failed: HTTP probe failed with statuscode: 403/404
```

Restore it and watch the Pod rejoin:

```bash
kubectl exec $POD -- sh -c 'echo ok > /usr/share/nginx/html/index.html'
kubectl get endpoints web          # 2 addresses again
```

Takeaway: container never restarted, only the traffic flow changed.

### Example 3: Liveness failure triggers restart

Same Pod, but make **liveness** fail. Deploy a simple Pod whose health file disappears after 30 seconds:

```yaml
# liveness-exec.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
spec:
  containers:
  - name: app
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command: [cat, /tmp/healthy]
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
```

```bash
kubectl apply -f liveness-exec.yaml
kubectl get pod liveness-exec -w
# after ~40 s: RESTARTS 1
kubectl describe pod liveness-exec | grep -A12 Events
#  Warning  Unhealthy  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
#  Normal   Killing    Container app failed liveness probe, will be restarted
```

### Example 4: tcpSocket and exec probes for a database

```yaml
# pg-probes.yaml (container snippet)
containers:
- name: postgres
  image: postgres:16
  env:
  - {name: POSTGRES_PASSWORD, value: "lab-only"}
  ports:
  - {name: pg, containerPort: 5432}
  startupProbe:
    tcpSocket: {port: pg}
    periodSeconds: 5
    failureThreshold: 30          # up to 150 s to start
  livenessProbe:
    tcpSocket: {port: pg}
    periodSeconds: 10
  readinessProbe:
    exec:
      command: ["sh", "-c", "pg_isready -U postgres -h 127.0.0.1"]
    periodSeconds: 5
```

Why different handlers: TCP check is enough to know the process is accepting connections (liveness); `pg_isready` also confirms the server is willing to accept queries (readiness).

### Example 5: gRPC probe

```yaml
readinessProbe:
  grpc:
    port: 9090
    service: ""        # empty = overall server health
  periodSeconds: 5
```

The server must implement `grpc.health.v1.Health`. Requires Kubernetes 1.27+ (field is GA).

### Example 6: Startup probe protecting a slow app

Simulate an app that needs 40 s to start, with an aggressive liveness probe.

```yaml
# slow-start.yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-start
spec:
  containers:
  - name: app
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - sleep 40; touch /tmp/started; sleep 3600
    livenessProbe:                # WITHOUT startupProbe this kills the app
      exec: {command: [cat, /tmp/started]}
      periodSeconds: 5
      failureThreshold: 2         # dead at ~10 s -> restart loop
    startupProbe:
      exec: {command: [cat, /tmp/started]}
      periodSeconds: 5
      failureThreshold: 12        # 12 × 5 = 60 s allowed
```

```bash
kubectl apply -f slow-start.yaml
kubectl get pod slow-start -w        # READY 0/1 for ~40 s, then 1/1, RESTARTS 0
```

Now remove the `startupProbe` block and re-apply (delete the Pod first): the Pod enters `CrashLoopBackOff`, because liveness fails before the app is up.

### Example 7: Crash loop from a bad liveness probe

```yaml
# bad-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-liveness
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    livenessProbe:
      httpGet: {path: /does-not-exist, port: 80}
      periodSeconds: 3
      failureThreshold: 1
```

```bash
kubectl apply -f bad-liveness.yaml
kubectl get pod bad-liveness -w      # RESTARTS climbs, status becomes CrashLoopBackOff
kubectl describe pod bad-liveness | grep -E "Liveness|Unhealthy|Killing|Back-off"
kubectl logs bad-liveness --previous # nginx itself looks fine: exit was SIGTERM
```

Diagnosis rule: app logs clean + exit code 0 or 143 + many `Unhealthy` events → suspect the probe, not the app.

### Example 8: Requests, limits, and QoS classes

```yaml
# qos.yaml
apiVersion: v1
kind: Pod
metadata: {name: qos-guaranteed}
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests: {cpu: 200m, memory: 128Mi}
      limits:   {cpu: 200m, memory: 128Mi}
---
apiVersion: v1
kind: Pod
metadata: {name: qos-burstable}
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests: {cpu: 100m, memory: 64Mi}
      limits:   {cpu: 500m, memory: 256Mi}
---
apiVersion: v1
kind: Pod
metadata: {name: qos-besteffort}
spec:
  containers:
  - name: app
    image: nginx:1.27
```

```bash
kubectl apply -f qos.yaml
kubectl get pods -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
# NAME             QOS
# qos-besteffort   BestEffort
# qos-burstable    Burstable
# qos-guaranteed   Guaranteed
```

Trick questions for the class:
- Limits set, requests missing → requests copied from limits → **Guaranteed**.
- Requests equal limits for memory but only a request for CPU → **Burstable**.
- Two containers, one with equal requests/limits and one with nothing → **Burstable** (every container must qualify).

### Example 9: OOMKilled

```yaml
# oom.yaml
apiVersion: v1
kind: Pod
metadata: {name: oom-demo}
spec:
  restartPolicy: Always
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    resources:
      requests: {memory: 50Mi}
      limits:   {memory: 100Mi}
```

```bash
kubectl apply -f oom.yaml
kubectl get pod oom-demo -w          # OOMKilled -> CrashLoopBackOff
kubectl describe pod oom-demo | grep -A5 "Last State"
# Reason: OOMKilled  Exit Code: 137
```

Fix: raise the memory limit (or reduce `--vm-bytes`) and the Pod stabilises.

### Example 10: CPU throttling (no kill)

```yaml
# cpu.yaml
apiVersion: v1
kind: Pod
metadata: {name: cpu-demo}
spec:
  containers:
  - name: burn
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "2"]
    resources:
      requests: {cpu: 100m}
      limits:   {cpu: 200m}
```

```bash
kubectl apply -f cpu.yaml
kubectl top pod cpu-demo             # ≈ 200m despite 2 busy threads (needs metrics-server)
kubectl get pod cpu-demo             # still Running, RESTARTS 0
```

CPU is throttled, never killed. Inside the node: `cat /sys/fs/cgroup/.../cpu.stat` shows `nr_throttled` increasing.

### Example 11: Unschedulable Pod (requests too big)

```bash
kubectl run huge --image=nginx:1.27 --overrides='{
  "spec": {"containers": [{"name":"huge","image":"nginx:1.27",
  "resources": {"requests": {"cpu":"64","memory":"512Gi"}}}]}}'
kubectl get pod huge                 # Pending
kubectl describe pod huge | tail -n 5
# 0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.
kubectl delete pod huge
```

### Example 12: LimitRange injects defaults and rejects violations

```bash
kubectl create namespace team-a
kubectl apply -n team-a -f - <<'EOF'
apiVersion: v1
kind: LimitRange
metadata: {name: container-limits}
spec:
  limits:
  - type: Container
    default:        {cpu: 500m, memory: 256Mi}
    defaultRequest: {cpu: 100m, memory: 64Mi}
    min:            {cpu: 50m,  memory: 32Mi}
    max:            {cpu: "1",  memory: 512Mi}
EOF

kubectl describe limitrange container-limits -n team-a

# 1) Pod without resources -> defaults injected
kubectl run plain --image=nginx:1.27 -n team-a
kubectl get pod plain -n team-a -o jsonpath='{.spec.containers[0].resources}{"\n"}'
# {"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"64Mi"}}
kubectl get pod plain -n team-a -o jsonpath='{.status.qosClass}{"\n"}'   # Burstable (no longer BestEffort)

# 2) Pod above max -> rejected
kubectl run greedy --image=nginx:1.27 -n team-a \
  --overrides='{"spec":{"containers":[{"name":"greedy","image":"nginx:1.27","resources":{"limits":{"cpu":"2"}}}]}}'
# Error from server (Forbidden): pods "greedy" is forbidden: maximum cpu usage per Container is 1, but limit is 2
```

### Example 13: ResourceQuota (compute + object counts)

```bash
kubectl apply -n team-a -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata: {name: team-a-quota}
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: 512Mi
    limits.cpu: "1"
    limits.memory: 1Gi
    pods: "4"
    services: "2"
EOF

kubectl describe quota team-a-quota -n team-a
# Used shows the 'plain' Pod: requests.cpu 100m, pods 1

# Scale a Deployment past the cap
kubectl create deployment burst --image=nginx:1.27 --replicas=6 -n team-a
kubectl get deploy burst -n team-a                      # READY 3/6 (pods quota 4 incl. 'plain')
kubectl describe rs -n team-a -l app=burst | tail -n 6  # FailedCreate ... exceeded quota: team-a-quota
```

Note the Deployment applied successfully; the failure is visible only in ReplicaSet events — a classic support question.

### Example 14: Quota without LimitRange rejects bare Pods

```bash
kubectl delete limitrange container-limits -n team-a
kubectl run nodefaults --image=nginx:1.27 -n team-a
# Error from server (Forbidden): pods "nodefaults" is forbidden: failed quota: team-a-quota:
# must specify limits.cpu for: nodefaults; limits.memory for: nodefaults; requests.cpu ...
```

Restore the LimitRange and the same command succeeds. This is why the two objects are deployed together.

### Cleanup

```bash
kubectl delete namespace team-a probes-lab
kubectl config set-context --current --namespace=default
```

## Common Pitfalls

- **Liveness probe checks a dependency (DB, downstream API)** — one backend outage makes every Pod fail liveness and restart together. Liveness checks only the app itself; put dependency checks in readiness.
- **Liveness probe on a slow-starting app, no startup probe** — container is killed before it finishes booting, `CrashLoopBackOff`. Add a `startupProbe` with `failureThreshold × periodSeconds` ≥ worst-case boot time.
- **Same endpoint and same thresholds for liveness and readiness** — a brief slowdown both restarts the Pod *and* removes it from rotation. Use distinct endpoints/timing; readiness more sensitive, liveness more forgiving.
- **`timeoutSeconds: 1` default with a slow health endpoint** — probe fails under load, Pod restarts under load, making load worse. Raise `timeoutSeconds` and keep the endpoint cheap.
- **Expecting readiness failure to restart the Pod** — it never does. It only removes the Pod from endpoints. Use liveness if a restart is what you want.
- **No readiness probe during rolling updates** — new Pods count as ready instantly; users hit half-started containers, and rollout continues even if the new version is broken.
- **`successThreshold` > 1 on liveness/startup** — API rejects it; only readiness supports values above 1.
- **`httpGet` returns 3xx/auth redirect** — probe follows no auth; `401` is a failure. Expose an unauthenticated `/healthz` or use `httpHeaders`. Redirects to another host are treated as success only for 200-399 codes, but the redirect itself is not followed across hosts in the way a browser would.
- **Memory unit typos** — `128m` (0.128 bytes), `128M` vs `128Mi`, `1g` vs `1Gi`. Always use `Mi`/`Gi` for memory and `m` only for CPU.
- **Requests missing everywhere** — all Pods BestEffort, scheduler overpacks nodes, first memory spike evicts a random mix of workloads. Set requests (or a LimitRange default).
- **Requests set far above real usage** — nodes look "full" at 10% real CPU, cluster autoscaler adds nodes, money burns. Measure with `kubectl top` and right-size.
- **CPU limit too tight** — latency spikes with no errors and no restarts; throttling is invisible in `kubectl get pods`. Consider a CPU request without a limit for latency-sensitive services.
- **Limit below the runtime heap** — JVM or Node starts, then `OOMKilled` (exit 137). Align runtime heap flags with the container limit and keep ~20-25 % headroom.
- **Confusing `OOMKilled` and `Evicted`** — first is the container's own limit, second is node pressure. Fix the first by raising limits; the second by right-sizing requests or adding node capacity.
- **ResourceQuota added after workloads exist** — existing Pods are untouched; new Pods without `resources` fail. Roll out quota together with a LimitRange.
- **Quota rejection silent on `kubectl apply` of a Deployment** — the ReplicaSet fails creating Pods. Check `kubectl describe rs` and `kubectl describe quota`.
- **LimitRange expected to change running Pods** — it applies only at creation time of new Pods.
- **Quota counts `Terminating` Pods during rollouts** — a quota with exactly the replica count blocks surge Pods; leave headroom for `maxSurge`.

## FAQ

**Q: What is the difference between liveness and readiness probes?**
A: Liveness answers "should kubelet restart this container?" — a failure kills and restarts it. Readiness answers "should this Pod receive traffic?" — a failure only removes the Pod's IP from Service endpoints and does not restart anything. Use liveness for unrecoverable stuck states, readiness for temporary unavailability.

**Q: Do I need all three probes?**
A: No, all are optional. Typical minimum for production web apps: readiness (safe rollouts and traffic gating) plus liveness. Add startup only for slow-booting apps. A stateless app that exits on failure may need neither liveness nor startup.

**Q: What does the startup probe do that `initialDelaySeconds` does not?**
A: `initialDelaySeconds` is a fixed wait that delays all failure detection for the Pod's whole life if you make it long. A startup probe polls until the app is actually up (up to `failureThreshold × periodSeconds`) and then hands over to a tight liveness probe. Fast boot gets fast detection; slow boot still survives.

**Q: How do I calculate the maximum startup time?**
A: `failureThreshold × periodSeconds` of the startup probe. For `failureThreshold: 30` and `periodSeconds: 10` the app has 300 seconds. If it has not passed once by then, the container is killed and restarted.

**Q: Which handler should I choose?**
A: `httpGet` for any HTTP service with a health endpoint; `grpc` for gRPC services; `tcpSocket` when you only need to know a port is accepting connections (databases, caches, legacy services); `exec` for custom logic (`pg_isready`, `redis-cli ping`, file checks). Prefer `httpGet`/`tcpSocket`/`grpc` since `exec` spawns a process each period.

**Q: What counts as success for an `httpGet` probe?**
A: Any HTTP status from 200 up to and including 399. 4xx and 5xx fail, as does a timeout or connection error.

**Q: Who sends the probe request — the Service, the Ingress, kubelet?**
A: kubelet on the node running the Pod. It connects directly to the Pod IP (or runs `exec` inside the container). The request does not pass through any Service.

**Q: Why is my Pod `Running` but `0/1` READY?**
A: Readiness probe is failing (or has not yet succeeded) or a startup probe is still running. Check `kubectl describe pod` → Events for `Readiness probe failed`, and test the endpoint with `kubectl exec <pod> -- wget -qO- localhost:<port>/<path>` or `curl`.

**Q: Why is my Pod in `CrashLoopBackOff` even though the app logs look fine?**
A: A failing liveness probe may be killing a healthy-but-slow container. Look for `Liveness probe failed` and `Killing` events and exit code 143/0. Fix by loosening thresholds/timeouts or adding a startup probe.

**Q: A Service returns 503 although the Deployment shows replicas — why?**
A: If all Pods are `NotReady`, the Service has no ready endpoints (`kubectl get endpoints <svc>` shows `<none>`). Common causes: readiness path wrong, wrong port, app still starting, or a shared dependency down.

**Q: What are requests and limits, in one sentence each?**
A: Request: the amount reserved for the container and used by the scheduler to place the Pod. Limit: the ceiling the container may not exceed — CPU is throttled at it, memory triggers an OOM kill.

**Q: Does the scheduler use limits or actual usage?**
A: Neither. It uses the sum of **requests** on each node compared to allocatable. Actual usage and limits do not influence placement.

**Q: What does `500m` CPU mean?**
A: Half a core (500 millicores). `1000m` equals `1` core. Memory uses binary suffixes: `256Mi`, `1Gi`.

**Q: What is the difference between CPU throttling and OOMKilled?**
A: CPU is compressible: when the container hits its CPU limit the kernel pauses it for part of each scheduling period — slow but alive. Memory is not compressible: over the limit the kernel's OOM killer terminates the process (exit 137, `OOMKilled`) and kubelet restarts it.

**Q: Should I set CPU limits?**
A: It depends. CPU limits cause throttling even when the node is idle, which hurts latency-sensitive apps; many teams set CPU requests only. Limits protect against runaway neighbours and are required for Guaranteed QoS. Memory limits are almost always set.

**Q: How do I get the Guaranteed QoS class?**
A: Every container (including init containers) must set CPU and memory requests and limits, and each request must equal its limit.

**Q: How do I check a Pod's QoS class?**
A: `kubectl get pod <p> -o jsonpath='{.status.qosClass}'`, or `kubectl describe pod <p>` (`QoS Class:` line).

**Q: In what order are Pods evicted under node memory pressure?**
A: BestEffort and Burstable Pods exceeding their requests go first (ranked by priority, then usage above request); Guaranteed Pods are evicted last. Evicted Pods are marked `Failed/Evicted` and rescheduled by their controller.

**Q: Why does my Pod stay Pending with `Insufficient cpu` when nodes are idle?**
A: The scheduler counts reserved requests, not usage. Nodes can be fully "requested" and nearly idle. Lower requests, add nodes, or free up reservations. See `kubectl describe node` → Allocated resources.

**Q: What does LimitRange do exactly?**
A: In a namespace, it injects default requests/limits into containers that omit them, and rejects containers outside min/max (and optionally a limit/request ratio). It applies at Pod creation only.

**Q: What does ResourceQuota do exactly?**
A: Caps the total requests/limits and object counts (Pods, Services, PVCs, …) in a namespace. Creates that would exceed the cap are rejected with `exceeded quota`.

**Q: Why do I get `must specify limits.cpu` when creating a Pod?**
A: The namespace has a ResourceQuota on compute resources, so every Pod must declare them. Add resources to the Pod or create a LimitRange with defaults.

**Q: My Deployment applied fine but only some Pods exist. Why?**
A: Probably a quota (or LimitRange max) rejects extra Pods. `kubectl describe rs <name>` shows `FailedCreate` events; `kubectl describe quota` shows Used vs Hard.

**Q: Are init containers probed?**
A: No. Init containers run to completion and do not support liveness/readiness/startup probes. (Their resources still count towards the Pod's effective request.)

**Q: Is `kubectl top` always available?**
A: No. It needs the metrics-server add-on. On kind/minikube install or enable it (`minikube addons enable metrics-server`). Without it `kubectl top` returns `Metrics API not available`.

**Q: Can I change probes or resources without recreating Pods?**
A: Not in place for ordinary Pods; editing the Deployment's Pod template triggers a rolling update that replaces the Pods. (Some recent Kubernetes versions allow in-place resource resize, but treat replacement as the baseline.)

## Practice / Exercise

Work in namespace `practice45`. Use `nginx:1.27` and `busybox:1.36`.

### Core

1. **Readiness gating.** Create a Deployment `shop` with 3 replicas of nginx, a `Service`, a readiness probe on `/` (period 3 s, failureThreshold 2). Delete `index.html` in one Pod; show with `kubectl get endpoints shop` that the Pod leaves the Service and `RESTARTS` stays 0. Restore it and show it returns.
2. **Liveness restart.** Create a busybox Pod whose liveness `exec` probe reads a file that the container deletes after 20 s. Record how long until the first restart and match it against `initialDelaySeconds + failureThreshold × periodSeconds`.
3. **Startup probe.** Simulate an app that needs 45 s to start. First give it only an aggressive liveness probe and observe `CrashLoopBackOff`. Then add a startup probe with enough budget (state your calculation) and show 0 restarts.
4. **QoS table.** Create one Pod each for Guaranteed, Burstable and BestEffort. Print their `qosClass` with `custom-columns`. Then predict, without running it, the class for: (a) limits only, (b) requests only, (c) two containers where only one sets resources. Verify each prediction.
5. **OOMKilled.** Run `polinux/stress` allocating 200M with a 100Mi limit. Capture `reason` and exit code from `kubectl get pod -o jsonpath`. Fix it and confirm stability.
6. **LimitRange.** Create a LimitRange with `default`, `defaultRequest`, `min`, `max`. Create a Pod with no resources and print what was injected. Create a Pod that violates `max` and quote the error.
7. **ResourceQuota.** Add a quota with `pods: "3"` and `requests.cpu: 300m`. Scale a Deployment to 6 and explain, using `kubectl describe rs` and `kubectl describe quota`, why only part of the Pods exist.

### Stretch

1. **Dependency trap.** Build two Pods behind a Service where `/healthz` (liveness) calls a backend that you can switch off. Show how coupling liveness to the backend causes simultaneous restarts, then fix by moving the dependency check to readiness.
2. **gRPC probe.** Deploy a gRPC health-enabled sample server (any image implementing `grpc.health.v1`) and probe it with `grpc:`; flip its status to `NOT_SERVING` and watch readiness change.
3. **Right-sizing.** With metrics-server running, generate load against a Deployment and use `kubectl top pod --containers` to choose requests (≈ typical usage) and limits (≈ peak). Document your numbers and the QoS class you end up with.
4. **Eviction thinking exercise.** Write down, for a node running 2 Guaranteed, 3 Burstable and 2 BestEffort Pods, the order in which kubelet would evict under memory pressure, and what you would change in specs so that your database Pod is evicted last.
5. **Quota scopes.** Create a `ResourceQuota` scoped to `BestEffort` with `pods: "1"` and prove a second BestEffort Pod is rejected while Burstable Pods are not counted.
6. **Rollout safety.** Roll out a Deployment to a version whose readiness endpoint always fails. Show with `kubectl rollout status` that the rollout stalls, old Pods keep serving, and `kubectl rollout undo` recovers.

## Further Reading

- Configure Liveness, Readiness and Startup Probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Pod lifecycle (container states, probes, restart policy): https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- Resource Management for Pods and Containers: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- Pod Quality of Service Classes: https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/
- Node-pressure Eviction: https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/
- Limit Ranges: https://kubernetes.io/docs/concepts/policy/limit-range/
- Resource Quotas: https://kubernetes.io/docs/concepts/policy/resource-quotas/
- Configure Default Memory/CPU Requests and Limits for a Namespace: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/
- Kubernetes metrics-server: https://github.com/kubernetes-sigs/metrics-server

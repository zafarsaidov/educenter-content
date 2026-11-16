# Lesson 51: Practice – Scheduling and Security

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lesson 45 (probes, requests/limits), Lesson 46 (taints, tolerations, nodeAffinity), Lesson 47 (RBAC), Lesson 48 (NetworkPolicy, PodSecurity, admission), Lesson 49 (kube-bench, AppArmor, seccomp), Lesson 50 (Trivy, audit logs, Kyverno). Lab cluster with **at least 2 worker nodes**, a CNI that enforces NetworkPolicy (Calico or Cilium — plain Flannel silently ignores policies), Kyverno installed, `trivy` and `docker` on the student workstation, and shell access to a control-plane node (for kube-bench).

## Learning Objectives

By end of lesson student can:
- Dedicate a node to a workload using a label + taint on the node and `nodeAffinity` + `tolerations` in the Pod template, and explain why **both** halves are required
- Predict and observe the three scheduler outcomes: Running on the right node, `Pending` with `FailedScheduling`, and eviction with `NoExecute`
- Build a default-deny network posture (ingress + egress), then open exactly one path (frontend → backend) plus DNS, and prove the rest stays blocked with `wget`/`nc` tests
- Run kube-bench on a control-plane node, read the report, pick one FAIL, apply the remediation to a static-pod manifest and re-run the single check to prove it flipped to PASS
- Scan a known-vulnerable image with Trivy, interpret severity/fixed-version columns, rebuild on an updated base image and compare the counts
- Write a Kyverno `ClusterPolicy` that denies `:latest` (and untagged) images, test enforcement with allowed and denied Pods, and read the policy report
- Troubleshoot the typical failures of each area (Pending Pod, blocked DNS, failed static pod restart, scanner noise, webhook rejection)

## Topics

- Deploy workload with nodeAffinity + toleration for a tainted node
- Apply Network Policy: deny-all then allow only frontend → backend traffic
- Run kube-bench, identify one failing check, apply remediation
- Scan a known-vulnerable image with Trivy; rebuild with updated base image
- Apply Kyverno ClusterPolicy (deny latest tag), test enforcement by deploying :latest

## Concepts

This is a **practice lesson**. Every tool was taught in lessons 46-50; the goal now is to use them together, see them fail on purpose, and fix them. Five labs (A-E) are independent — if time is short, the instructor demos A and C, students do B, D, E. Every lab has a "✅ Checkpoint": the student must see that output before moving on. Most "my lab is broken" questions are a missed checkpoint one step earlier.

### The five labs at a glance

```text
Lab A  Scheduling   label + taint node ──► Deployment with nodeAffinity + toleration ──► Pods only on that node
Lab B  Network      default-deny ──► allow DNS ──► allow frontend→backend ──► intruder still blocked
Lab C  Hardening    kube-bench ──► pick a FAIL ──► edit static pod manifest ──► re-run check ──► PASS
Lab D  Supply chain trivy image (old tag) ──► many HIGH/CRITICAL ──► rebuild on new base ──► fewer
Lab E  Policy       Kyverno ClusterPolicy ──► deploy :latest ──► denied at admission ──► pinned tag OK
```

Security theme: **defence in depth**. Lab A decides *where* a Pod may run, Lab B decides *who* may talk to it, Lab C hardens the *cluster components*, Lab D checks *what is inside the image*, Lab E blocks *bad Pods before they exist*. None of them replaces the others.

### Why taint + affinity together

A **taint** on a node says "keep out unless you tolerate me" — it only *repels*. A **toleration** on a Pod says "I am allowed in" — it does **not** attract the Pod to that node; the Pod may still land on any other node. **nodeAffinity** *attracts*. To truly dedicate a node you need:

| Piece | Where | Effect |
|---|---|---|
| Taint `dedicated=batch:NoSchedule` | Node | Pods without toleration cannot be scheduled there |
| Label `workload=batch` | Node | Gives affinity something to match |
| Toleration for the taint | Pod template | Pod is allowed on the node |
| `nodeAffinity` required `workload=batch` | Pod template | Pod is only allowed *on* the node |

Taint alone = other teams are kept off, but your Pods wander. Affinity alone = your Pods go there, but other teams' Pods can also fill it. Both = exclusive node.

### Default-deny in one picture

NetworkPolicy is **additive and allow-only**. A Pod with no policy selecting it accepts everything. The moment *any* policy selects it for a direction (Ingress or Egress), only traffic explicitly allowed by some policy is accepted in that direction. So the recipe is: (1) select everything with an empty `podSelector: {}` and list both policy types to deny all; (2) add small allow policies on top. Remember: denying **egress** also denies **DNS** (port 53 to CoreDNS in `kube-system`), so name resolution breaks until you allow it explicitly.

### Scan, then gate

Trivy (lesson 50) finds known CVEs *in an image already built*. Kyverno (lesson 50) refuses Pods *at admission time*. Together they form a pipeline: scan finds the problem, policy prevents the problem from being deployed again. In this lab policy is simple (no `:latest`), but the same pattern extends to "only images from our registry" or "only images signed".

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl label node` | Add a label to a node | `kubectl label node worker-1 workload=batch` |
| `kubectl taint node` | Add / remove a taint (`-` suffix removes) | `kubectl taint node worker-1 dedicated=batch:NoSchedule` |
| `kubectl get pods -o wide` | Show which node each Pod runs on | `kubectl get pods -n lab51 -o wide` |
| `kubectl describe pod` | Read scheduling events (`FailedScheduling`) | `kubectl describe pod batch-xxx -n lab51` |
| `kubectl get events --sort-by` | Chronological events | `kubectl get events -n lab51 --sort-by=.lastTimestamp` |
| `kubectl exec ... -- wget` | Test connectivity from inside a Pod | `kubectl exec -n lab51-net deploy/frontend -- wget -qO- -T 3 backend` |
| `kubectl get netpol` | List NetworkPolicies | `kubectl get netpol -n lab51-net` |
| `kubectl auth can-i` | Verify RBAC for a ServiceAccount | `kubectl auth can-i list pods --as=system:serviceaccount:lab51:default` |
| `kube-bench run --targets` | Run CIS checks for components | `kube-bench run --targets master` |
| `kube-bench run --check` | Run one check only | `kube-bench run --targets master --check 1.3.2` |
| `trivy image` | Scan an image for CVEs | `trivy image --severity HIGH,CRITICAL nginx:1.19` |
| `trivy image --ignore-unfixed` | Hide CVEs with no fix yet | `trivy image --ignore-unfixed python:3.9-slim` |
| `trivy image --exit-code 1` | Fail (non-zero exit) when findings exist | `trivy image --exit-code 1 --severity CRITICAL myapp:1.0` |
| `kubectl apply -f policy.yaml` | Create a Kyverno ClusterPolicy | `kubectl apply -f disallow-latest.yaml` |
| `kubectl get clusterpolicy` | List policies and readiness | `kubectl get cpol` |
| `kubectl get policyreport -A` | Background scan results | `kubectl get polr -A` |

## Examples / Walkthrough

### Step 0 — Pre-flight (5 min, everyone)

```bash
# Nodes: need at least 2 workers
kubectl get nodes -o wide

# CNI that enforces NetworkPolicy? Look for calico / cilium pods
kubectl get pods -A | grep -E 'calico|cilium'

# Kyverno ready?
kubectl get pods -n kyverno
kubectl get crd | grep kyverno

# Tools on the workstation
trivy --version
docker version --format '{{.Client.Version}}'

# Fresh namespaces for the labs
kubectl create namespace lab51
kubectl create namespace lab51-net
kubectl create namespace lab51-pol
```

✅ **Checkpoint:** two or more `Ready` workers, Calico/Cilium pods `Running`, Kyverno pods `Running`, `trivy` prints a version. If the CNI is Flannel, **stop** — Lab B will "pass" without enforcing anything. Use a lab cluster with Calico/Cilium.

---

## Lab A — Dedicated node with taint, label, nodeAffinity and toleration (25 min)

### A1. Pick a node and mark it

```bash
# choose a worker (not the control plane)
NODE=$(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')
echo "$NODE"

kubectl label node "$NODE" workload=batch
kubectl taint node "$NODE" dedicated=batch:NoSchedule

# verify both
kubectl get node "$NODE" --show-labels | tr ',' '\n' | grep workload
kubectl describe node "$NODE" | grep -A2 -i taints
```

Expected:

```text
workload=batch
Taints:             dedicated=batch:NoSchedule
```

### A2. A Pod WITHOUT toleration or affinity — stays off the node

```bash
kubectl create deployment plain --image=nginx:1.27 --replicas=4 -n lab51
kubectl get pods -n lab51 -o wide
```

All four replicas land on the **other** workers; none on `$NODE`, because the taint repels them.

✅ **Checkpoint:** the NODE column never shows `$NODE`.

### A3. A Pod WITH toleration but NO affinity — allowed but not attracted

Save as `tolerant.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tolerant
  namespace: lab51
spec:
  replicas: 4
  selector:
    matchLabels:
      app: tolerant
  template:
    metadata:
      labels:
        app: tolerant
    spec:
      tolerations:
        - key: dedicated
          operator: Equal
          value: batch
          effect: NoSchedule
      containers:
        - name: app
          image: nginx:1.27
```

```bash
kubectl apply -f tolerant.yaml
kubectl get pods -n lab51 -l app=tolerant -o wide
```

Some replicas may land on `$NODE`, some elsewhere — the toleration only **permits**. This is the classic misunderstanding: "I added a toleration, why is my Pod on the wrong node?"

### A4. The correct pattern — toleration + required nodeAffinity

Save as `batch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch
  namespace: lab51
spec:
  replicas: 4
  selector:
    matchLabels:
      app: batch
  template:
    metadata:
      labels:
        app: batch
    spec:
      tolerations:
        - key: dedicated
          operator: Equal
          value: batch
          effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: workload
                    operator: In
                    values: ["batch"]
      containers:
        - name: app
          image: nginx:1.27
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
```

```bash
kubectl apply -f batch.yaml
kubectl get pods -n lab51 -l app=batch -o wide
```

✅ **Checkpoint:** all four `batch-*` Pods on `$NODE`. No exceptions.

### A5. Break it on purpose — Pending and the event that explains it

Remove the toleration but keep the affinity:

```bash
kubectl patch deployment batch -n lab51 --type json \
  -p '[{"op":"remove","path":"/spec/template/spec/tolerations"}]'
kubectl get pods -n lab51 -l app=batch
kubectl describe pod -n lab51 -l app=batch | grep -A6 Events | head -20
```

New Pods stay `Pending`. The event reads like:

```text
Warning  FailedScheduling  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {dedicated: batch}, 2 node(s) didn't match Pod's node affinity/selector.
```

Read it aloud: **one node is tainted and you do not tolerate it; the others do not match your affinity.** Restore:

```bash
kubectl apply -f batch.yaml
kubectl rollout status deployment/batch -n lab51
```

### A6. NoExecute — eviction of running Pods

`NoSchedule` never touches Pods already running. `NoExecute` does.

```bash
# plain Pods currently run on *other* nodes. Move them: put NoExecute on a node that runs plain Pods
VICTIM=$(kubectl get pods -n lab51 -l app=plain -o jsonpath='{.items[0].spec.nodeName}')
echo "$VICTIM"
kubectl taint node "$VICTIM" maintenance=true:NoExecute
kubectl get pods -n lab51 -l app=plain -o wide -w     # Ctrl+C after a few seconds
```

Pods on `$VICTIM` are terminated and the ReplicaSet recreates them on other nodes. Remove the taint (trailing `-`):

```bash
kubectl taint node "$VICTIM" maintenance=true:NoExecute-
```

`tolerationSeconds` lets a Pod stay for a limited time before eviction; the built-in `node.kubernetes.io/not-ready:NoExecute` taint uses exactly this (default 300 s).

### A7. Clean up Lab A

```bash
kubectl delete namespace lab51
kubectl taint node "$NODE" dedicated=batch:NoSchedule-
kubectl label node "$NODE" workload-
```

✅ **Checkpoint:** `kubectl describe node "$NODE" | grep -i taints` prints `<none>`.

---

## Lab B — Default-deny, then allow only frontend → backend (35 min)

### B1. Deploy three Pods and a Service

Three roles: **frontend** (allowed), **backend** (target), **intruder** (must be blocked).

```bash
kubectl -n lab51-net create deployment backend  --image=nginx:1.27
kubectl -n lab51-net expose deployment backend --port=80
kubectl -n lab51-net create deployment frontend --image=busybox:1.36 -- sleep 3600
kubectl -n lab51-net create deployment intruder --image=busybox:1.36 -- sleep 3600
kubectl -n lab51-net rollout status deployment/backend
kubectl -n lab51-net rollout status deployment/frontend
kubectl -n lab51-net rollout status deployment/intruder
```

`create deployment` labels Pods `app=<name>` automatically — the policies below select on that label.

### B2. Baseline — everything can talk to everything

```bash
for who in frontend intruder; do
  echo "== $who → backend"
  kubectl -n lab51-net exec deploy/$who -- wget -qO- -T 3 http://backend | head -3
done
```

Both print the nginx welcome HTML. That is Kubernetes' default: a flat, open network.

✅ **Checkpoint:** both succeed. If one fails, fix DNS/CNI first.

### B3. Default deny — ingress and egress

Save as `00-default-deny.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: lab51-net
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

```bash
kubectl apply -f 00-default-deny.yaml
for who in frontend intruder; do
  echo "== $who → backend"
  kubectl -n lab51-net exec deploy/$who -- wget -qO- -T 3 http://backend || echo "BLOCKED"
done
```

Both now print `wget: bad address 'backend'` or time out → `BLOCKED`. Note the message: **`bad address`** means DNS itself is blocked (egress deny includes port 53). That is the number-one "my policy broke everything" surprise.

### B4. Allow DNS egress for all Pods

Save as `01-allow-dns.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: lab51-net
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl apply -f 01-allow-dns.yaml
kubectl -n lab51-net exec deploy/intruder -- nslookup backend
kubectl -n lab51-net exec deploy/intruder -- wget -qO- -T 3 http://backend || echo "STILL BLOCKED (good)"
```

DNS resolves now (`nslookup` returns the ClusterIP), but the HTTP connection is still blocked, because nothing allows ingress to the backend or egress to it.

**Note on the namespaceSelector/podSelector pair:** both are inside the same `to` list item → they are **AND-ed** (Pod with label `k8s-app=kube-dns` *in* `kube-system`). Writing them as two separate `- ` items would **OR** them and open far more than intended.

### B5. Allow frontend → backend (two policies: egress from frontend, ingress to backend)

Both directions are needed because default-deny covers both.

Save as `02-frontend-egress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend-egress
  namespace: lab51-net
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
```

Save as `03-backend-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-from-frontend-ingress
  namespace: lab51-net
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f 02-frontend-egress.yaml -f 03-backend-ingress.yaml
kubectl -n lab51-net get netpol
```

### B6. Prove it — positive and negative tests

```bash
echo "== frontend → backend (must WORK)"
kubectl -n lab51-net exec deploy/frontend -- wget -qO- -T 3 http://backend | head -3

echo "== intruder → backend (must be BLOCKED)"
kubectl -n lab51-net exec deploy/intruder -- wget -qO- -T 3 http://backend || echo "BLOCKED (good)"

echo "== frontend → backend on a wrong port (must be BLOCKED)"
kubectl -n lab51-net exec deploy/frontend -- nc -zv -w 3 backend 8080 || echo "BLOCKED (good)"

echo "== frontend → internet (must be BLOCKED)"
kubectl -n lab51-net exec deploy/frontend -- wget -qO- -T 3 http://example.com || echo "BLOCKED (good)"

echo "== backend → frontend (must be BLOCKED, backend has no egress allow)"
FRONT_IP=$(kubectl -n lab51-net get pod -l app=frontend -o jsonpath='{.items[0].status.podIP}')
kubectl -n lab51-net exec deploy/backend -- sh -c "curl -s -m 3 $FRONT_IP || echo BLOCKED"
```

✅ **Checkpoint:** exactly one success (frontend → backend:80). Every other test says BLOCKED. If `intruder` can still reach the backend, the CNI does not enforce policies (re-check Step 0) or the labels do not match (`kubectl get pods -n lab51-net --show-labels`).

### B7. Debugging toolkit when a path is unexpectedly blocked/open

```bash
kubectl -n lab51-net describe netpol backend-from-frontend-ingress   # which Pods does it select?
kubectl -n lab51-net get pods --show-labels                           # do labels match?
kubectl -n lab51-net get endpoints backend                            # does the Service have endpoints?
```

Calico: `calicoctl get networkpolicy -n lab51-net -o wide`. Cilium: `cilium monitor --type drop` shows dropped packets in real time.

### B8. Clean up Lab B

```bash
kubectl delete namespace lab51-net
```

---

## Lab C — kube-bench: find one FAIL and fix it (30 min)

Do this **on a lab/throwaway cluster** (never first on production). It edits a static-pod manifest on a control-plane node; a typo makes that component disappear until you fix the file.

### C1. Run kube-bench against the control plane

Option 1 — Job (works for most kubeadm-style clusters):

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml
kubectl wait --for=condition=complete job/kube-bench-master --timeout=120s
kubectl logs job/kube-bench-master > ~/kube-bench-before.txt
less ~/kube-bench-before.txt
```

Option 2 — binary directly on the control-plane node (when the Job's host mounts do not fit your distro):

```bash
# on the control-plane node
sudo kube-bench run --targets master > ~/kube-bench-before.txt
```

Look at the summary at the end:

```bash
grep -E '^\[(PASS|FAIL|WARN)\]' ~/kube-bench-before.txt | awk '{print $1}' | sort | uniq -c
grep '^\[FAIL\]' ~/kube-bench-before.txt | head -20
```

### C2. Pick ONE failing check

Good, safe, low-risk candidates on a typical kubeadm cluster (check IDs shift between benchmark versions — **use the ID printed in your report**, not the numbers below):

| Component | Typical FAIL | Remediation (from the report text) |
|---|---|---|
| kube-controller-manager | `--profiling` argument is not set to false | add `--profiling=false` to the command list |
| kube-scheduler | `--profiling` argument is not set to false | add `--profiling=false` |
| kube-apiserver | `--profiling` argument is not set to false | add `--profiling=false` |
| etcd | data directory ownership not `etcd:etcd` | `chown etcd:etcd /var/lib/etcd` (only if an `etcd` user exists) |
| Files | scheduler.conf / controller-manager.conf permissions | `chmod 600 /etc/kubernetes/scheduler.conf` |

The easiest first fix: **scheduler `--profiling=false`**. If the scheduler restarts badly, existing workloads keep running; only new scheduling pauses.

Read the exact remediation text kube-bench prints under the failing check (it appears in the "Remediation" section of the report):

```bash
grep -B1 -A8 'profiling' ~/kube-bench-before.txt | head -40
```

### C3. Apply the remediation safely

```bash
# on the control-plane node
sudo cp /etc/kubernetes/manifests/kube-scheduler.yaml /root/kube-scheduler.yaml.bak   # backup OUTSIDE the manifests dir!
sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
```

Add one line to `spec.containers[0].command`:

```yaml
    - --leader-elect=true
    - --profiling=false        # ← added
```

Save. The kubelet watches the directory, notices the file changed and **recreates the static Pod** within ~20-60 seconds. Watch it:

```bash
kubectl -n kube-system get pods -l component=kube-scheduler -w
kubectl -n kube-system describe pod -l component=kube-scheduler | grep profiling
```

⚠️ **Never leave `.bak` files inside `/etc/kubernetes/manifests/`** — the kubelet treats every `.yaml` there as a static Pod and you will run two schedulers.

### C4. Re-run only that check

```bash
sudo kube-bench run --targets master --check 1.4.1 | grep -E 'PASS|FAIL'    # use YOUR check ID
```

✅ **Checkpoint:** the check flipped from `[FAIL]` to `[PASS]`. Save a before/after:

```bash
sudo kube-bench run --targets master > ~/kube-bench-after.txt
diff <(grep -E '^\[(PASS|FAIL)\]' ~/kube-bench-before.txt) <(grep -E '^\[(PASS|FAIL)\]' ~/kube-bench-after.txt)
```

### C5. If the scheduler (or any component) does not come back

```bash
sudo crictl ps -a | grep kube-scheduler                 # container exited?
sudo crictl logs $(sudo crictl ps -a --name kube-scheduler -q | head -1)
sudo journalctl -u kubelet -n 50 --no-pager | tail -30  # kubelet complaining about the manifest YAML?
```

Typical causes: tab characters, wrong indentation, a typo in the flag name (`--profilng`), a missing leading `-`. Restore from the backup:

```bash
sudo cp /root/kube-scheduler.yaml.bak /etc/kubernetes/manifests/kube-scheduler.yaml
```

---

## Lab D — Trivy: scan a vulnerable image, rebuild, compare (25 min)

### D1. Scan an old, known-vulnerable image

```bash
trivy image --severity HIGH,CRITICAL nginx:1.19 | tee ~/trivy-nginx119.txt | tail -40
```

Read the table columns: **Library**, **Vulnerability** (CVE id), **Severity**, **Installed Version**, **Fixed Version**, **Title**. A blank Fixed Version means no upstream fix yet.

Count by severity:

```bash
trivy image -q -f json nginx:1.19 \
  | jq '[.Results[].Vulnerabilities[]?.Severity] | group_by(.) | map({(.[0]): length}) | add'
```

Typical output (numbers drift daily as the CVE database updates):

```json
{"CRITICAL": 40, "HIGH": 190, "MEDIUM": 280, "LOW": 120}
```

### D2. Only what you can actually fix

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed nginx:1.19 | tail -20
```

`--ignore-unfixed` removes CVEs with no patched version — your action list shrinks to "things updating would fix".

### D3. Build a tiny app on an old base, then on a new base

Save as `app.py`:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"ok\n")

HTTPServer(("", 8080), H).serve_forever()
```

Save as `Dockerfile.old`:

```dockerfile
FROM python:3.8
WORKDIR /app
COPY app.py .
USER 1000
CMD ["python", "app.py"]
```

Save as `Dockerfile.new`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
USER 1000
CMD ["python", "app.py"]
```

```bash
docker build -f Dockerfile.old -t lab51-app:old .
docker build -f Dockerfile.new -t lab51-app:new .

trivy image -q --severity HIGH,CRITICAL lab51-app:old | tail -5
trivy image -q --severity HIGH,CRITICAL lab51-app:new | tail -5
```

Compare totals side by side:

```bash
for t in old new; do
  printf "%s: " $t
  trivy image -q -f json --severity HIGH,CRITICAL lab51-app:$t \
    | jq '[.Results[]?.Vulnerabilities[]?] | length'
done
```

✅ **Checkpoint:** `new` has dramatically fewer HIGH/CRITICAL findings than `old` (often by an order of magnitude). A smaller base (`-slim`, `alpine`, distroless — lesson 31) = fewer packages = fewer CVEs.

### D4. Gate on severity (what CI would do)

```bash
trivy image --exit-code 1 --severity CRITICAL lab51-app:old ; echo "exit code: $?"
trivy image --exit-code 1 --severity CRITICAL lab51-app:new ; echo "exit code: $?"
```

`old` exits **1** (build would fail), `new` should exit **0**. If `new` still fails on a specific CVE with no fix, suppress it knowingly:

```bash
echo "CVE-2024-XXXXX" > .trivyignore      # always add a comment WHY and a review date in real life
trivy image --ignorefile .trivyignore --exit-code 1 --severity CRITICAL lab51-app:new
```

### D5. Scan what is already running in the cluster (optional)

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u | head -10
trivy image --severity CRITICAL $(kubectl get pods -A -o jsonpath='{.items[0].spec.containers[0].image}')
```

---

## Lab E — Kyverno: deny `:latest` and untagged images (25 min)

### E1. Write the policy in Audit mode first

Never start a new policy in `Enforce` on a shared cluster: Audit shows what *would* break without breaking it. Save as `disallow-latest.yaml`:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds: ["Pod"]
      exclude:
        any:
          - resources:
              namespaces: ["kube-system", "kyverno"]
      validate:
        message: "An explicit image tag is required."
        pattern:
          spec:
            containers:
              - image: "*:*"
    - name: validate-image-tag
      match:
        any:
          - resources:
              kinds: ["Pod"]
      exclude:
        any:
          - resources:
              namespaces: ["kube-system", "kyverno"]
      validate:
        message: "Using ':latest' is not allowed. Pin a version or digest."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

```bash
kubectl apply -f disallow-latest.yaml
kubectl get clusterpolicy
```

✅ **Checkpoint:** `READY` is `True` (older Kyverno prints `true`). If `False`, run `kubectl describe cpol disallow-latest-tag`.

Newer Kyverno releases moved the setting from `spec.validationFailureAction` to `spec.rules[].validate.failureAction`; the old field still works with a deprecation warning. Use whichever your installed version documents.

### E2. In Audit mode violations are allowed but recorded

```bash
kubectl -n lab51-pol run audit-demo --image=nginx:latest
kubectl -n lab51-pol get pod audit-demo                    # Running — Audit does not block

# the finding appears in the namespaced policy report
kubectl -n lab51-pol get policyreport
kubectl -n lab51-pol get policyreport -o yaml | grep -B2 -A6 'result: fail' | head -30
```

### E3. Switch to Enforce

```bash
kubectl patch clusterpolicy disallow-latest-tag --type merge \
  -p '{"spec":{"validationFailureAction":"Enforce"}}'
kubectl get clusterpolicy disallow-latest-tag -o jsonpath='{.spec.validationFailureAction}{"\n"}'
```

### E4. Test enforcement — four Pods, two outcomes

```bash
echo "== 1. :latest (must be DENIED)"
kubectl -n lab51-pol run t1 --image=nginx:latest

echo "== 2. no tag at all (must be DENIED, implies latest)"
kubectl -n lab51-pol run t2 --image=nginx

echo "== 3. pinned tag (must be ALLOWED)"
kubectl -n lab51-pol run t3 --image=nginx:1.27

echo "== 4. digest (must be ALLOWED)"
kubectl -n lab51-pol run t4 --image=nginx@sha256:0000000000000000000000000000000000000000000000000000000000000000 --dry-run=server -o name
```

The denied output looks like:

```text
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/lab51-pol/t1 was blocked due to the following policies

disallow-latest-tag:
  validate-image-tag: 'validation error: Using ':latest' is not allowed. Pin a version or digest. rule validate-image-tag failed at path /spec/containers/0/image/'
```

Read the message: **policy name → rule name → your message → the failing path.** The message you wrote in the policy is what developers see — make it say how to fix it.

✅ **Checkpoint:** t1 and t2 rejected, t3 `Running`.

### E5. Deployments are checked too

```bash
kubectl -n lab51-pol create deployment web --image=nginx:latest
kubectl -n lab51-pol get deployment web
kubectl -n lab51-pol get events --sort-by=.lastTimestamp | tail -5
```

The Deployment object is **created** (it is not a Pod), but its ReplicaSet cannot create Pods — `READY 0/1`, with the denial visible in the ReplicaSet events. Kyverno auto-generates equivalent rules for Pod controllers; verify with:

```bash
kubectl get clusterpolicy disallow-latest-tag -o yaml | grep -c 'autogen'
```

Fix by pinning:

```bash
kubectl -n lab51-pol set image deployment/web nginx=nginx:1.27
kubectl -n lab51-pol rollout status deployment/web
```

### E6. Existing workloads and background scan

Enforce does **not** delete or restart Pods that already run `:latest`. With `background: true`, Kyverno periodically re-scans and reports them:

```bash
kubectl get policyreport -A
kubectl get clusterpolicyreport
```

Use the report to find offenders across the cluster, then fix them via their owning Deployments.

### E7. Clean up Lab E

```bash
kubectl delete clusterpolicy disallow-latest-tag
kubectl delete namespace lab51-pol
```

---

## Final integration (optional, 10 min — instructor demo)

Bring the five labs together in one sentence: *"A Pod that runs on the dedicated node (A), reachable only from the frontend (B), on a cluster whose components passed CIS (C), built from a scanned image (D) and admitted only because its tag is pinned (E)."* Ask the class which control stops which attack:

| Threat | Control that stops it |
|---|---|
| Teammate's noisy Pod lands on the GPU/batch node | Taint + affinity (A) |
| Compromised frontend pivots to the database | NetworkPolicy (B) |
| Anonymous access to a component's profiling endpoint | kube-bench hardening (C) |
| Known CVE in base image exploited | Trivy scan in CI + rebuild (D) |
| Someone deploys `nginx:latest` and a tag change silently ships new code | Kyverno policy (E) |

## Common Pitfalls

- **Taint without label/affinity (or the reverse)** — a taint only repels, a toleration only permits, affinity only attracts. Dedicating a node needs taint + toleration + label + affinity.
- **Toleration added but Pod still `Pending`** — the Pod also has a required affinity or nodeSelector that no node matches. Read the `FailedScheduling` event; it lists each node's reason.
- **Forgetting the trailing `-` to remove a taint/label** — `kubectl taint node n key=val:NoSchedule-`, `kubectl label node n key-`. Without `-` you add a second one instead.
- **Default-deny egress breaks DNS** — symptom `bad address 'backend'`. Add the DNS allow policy to the CoreDNS Pods in `kube-system` on UDP and TCP 53.
- **Only writing one direction of a flow** — traffic needs egress allow on the source **and** ingress allow on the destination once both are default-denied.
- **`namespaceSelector` and `podSelector` as two list items** — that is OR, not AND. Put them in the same list item for AND.
- **Testing NetworkPolicy on a CNI that ignores it** — Flannel accepts the objects silently and enforces nothing. Always run a negative test (intruder must fail).
- **Editing static pod manifests without a backup outside the directory** — a `.bak` inside `/etc/kubernetes/manifests/` is run as a second Pod; a syntax error removes the component.
- **Trusting kube-bench check numbers from a blog post** — IDs change per benchmark version and per distro. Always use the ID and remediation text printed in your own report.
- **Applying every kube-bench "remediation" blindly** — some are inappropriate for your distro (rke2/k3s manage components differently) or break integrations. Understand each flag first.
- **Scanning `latest`** — a Trivy result for `:latest` is meaningless tomorrow. Scan the exact tag/digest you ship.
- **Expecting zero CVEs** — chasing zero wastes time. Gate on `--severity CRITICAL --ignore-unfixed`, review the rest on a schedule.
- **`.trivyignore` with no justification** — future-you will not know why a CVE was muted. Comment the reason and a re-check date.
- **Starting a Kyverno policy in Enforce** — one mistaken pattern can block deployments cluster-wide. Audit → review report → Enforce.
- **Forgetting to exclude system namespaces** — a strict policy can stop `kube-system` Pods from being recreated. Exclude `kube-system` and `kyverno` (Kyverno itself).
- **Assuming Enforce fixes existing Pods** — admission only affects new/changed objects. Use the policy report to find existing offenders.

## FAQ

**Q: I tainted a node and added a toleration to my Pod, but Pods still land on other nodes. Why?**
A: A toleration only *allows* a Pod onto the tainted node; it does not *prefer* or *require* it. Add `nodeAffinity` (or `nodeSelector`) with the node's label to attract the Pod.

**Q: What is the difference between `NoSchedule`, `PreferNoSchedule` and `NoExecute`?**
A: `NoSchedule` blocks new Pods without a toleration but leaves running ones. `PreferNoSchedule` is a soft version — the scheduler avoids the node if possible. `NoExecute` also evicts already-running Pods that do not tolerate it (optionally after `tolerationSeconds`).

**Q: My Pod is `Pending`. How do I find out why in 10 seconds?**
A: `kubectl describe pod <name>` and read the last `Events` line. `FailedScheduling` messages list per-node reasons (untolerated taint, affinity mismatch, insufficient cpu/memory, unbound PVC).

**Q: After applying default-deny nothing works, not even name resolution. Is that expected?**
A: Yes. Egress deny includes DNS. Allow UDP/TCP 53 to the `kube-dns` Pods in `kube-system`. The error `bad address` is the tell-tale sign.

**Q: Do I need a policy for responses (return traffic)?**
A: No. NetworkPolicy is stateful for established connections: reply packets of an allowed connection are permitted automatically. You only describe who may *initiate*.

**Q: Why do I need both an egress policy on the frontend and an ingress policy on the backend?**
A: Once both Pods are selected by a default-deny policy, each direction is evaluated independently: the frontend's egress must allow leaving, and the backend's ingress must allow arriving.

**Q: I applied NetworkPolicies but everything still connects. What is wrong?**
A: Most likely the CNI does not enforce NetworkPolicy (Flannel, some default kind setups). Install Calico or Cilium. The second most common cause: the `podSelector` labels do not match the Pods — check `--show-labels`.

**Q: Can I test a NetworkPolicy without a client Pod having curl/wget?**
A: Use `busybox` (`wget`, `nc`), or `kubectl debug` with an ephemeral container (`kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>`). Or run a throwaway Pod: `kubectl run tmp --rm -it --image=busybox:1.36 -- sh`.

**Q: kube-bench says FAIL but my cluster works fine. Do I have to fix everything?**
A: No. CIS is a benchmark, not a law. Fix what is relevant, document accepted risks, and skip checks that do not apply to your distribution (managed clouds like EKS/GKE have their own benchmarks because you cannot edit the control plane).

**Q: I edited the static pod manifest and the component never came back. What now?**
A: Look at `crictl ps -a` and `crictl logs` for the container, and `journalctl -u kubelet` for manifest parse errors. Typical causes are indentation, a bad flag name, or a missing leading dash. Restore your backup copy (kept *outside* `/etc/kubernetes/manifests/`).

**Q: How does kubelet know the static pod file changed?**
A: The kubelet watches the manifest directory (`staticPodPath`) and recreates the Pod when a file changes. No `kubectl apply` is involved, and the API server only sees a read-only mirror Pod.

**Q: Why does Trivy show different results today than last week for the same image?**
A: The vulnerability database updates continuously and new CVEs are published for old software. The image did not change; knowledge about its flaws did. That is why scans run regularly, not once.

**Q: What does an empty "Fixed Version" column mean?**
A: No patched release exists yet. You can only mitigate (different base image, remove the package, network controls). `--ignore-unfixed` hides these from a gate so you only fail on things you can fix.

**Q: Does a smaller image really have fewer vulnerabilities?**
A: Usually yes — scanners match installed packages, and `-slim`/alpine/distroless ship far fewer. It also shrinks the attack surface beyond CVEs: no shell, no package manager for an attacker to use.

**Q: Will Kyverno's `:latest` policy block my existing running Pods?**
A: No. It acts on admission (create/update). Existing Pods continue to run, but background scan marks them as failed in policy reports. They are rejected when recreated (for example by a Deployment rollout).

**Q: Why was my Deployment accepted but its Pods never appeared?**
A: The Deployment object is not a Pod. Kyverno denies the Pod that the ReplicaSet tries to create, so you see `READY 0/N`. Look at the ReplicaSet events (`kubectl describe rs`) for the denial message.

**Q: Audit vs Enforce — when do I use which?**
A: Audit while you learn the blast radius of a new policy (read the policy report), Enforce when the report is clean or you have communicated the change. Many teams keep high-risk cosmetic rules in Audit permanently.

**Q: Does `nginx` without a tag count as `latest`?**
A: Yes — the container runtime resolves it to `:latest`. That is why the policy has two rules: one requires a tag at all, one rejects `:latest` explicitly.

**Q: Can Kyverno itself be blocked by my policy?**
A: Yes, if the policy matches its own namespace and the Pods are recreated. Exclude `kyverno` and `kube-system`. Also be careful with webhook `failurePolicy: Fail` — if Kyverno is down, Pod creation can be blocked cluster-wide.

**Q: How do these five tools relate to Pod Security Admission (lesson 48)?**
A: PSA is a built-in coarse check on Pod security settings (privileged, hostPath, root) by namespace label. Kyverno is a flexible policy engine for anything you can express on a resource (tags, registries, labels, resource limits). Use PSA as a baseline and Kyverno for organisation-specific rules.

## Practice / Exercise

### Core (everyone, in class)

1. **Dedicated node.** Complete Lab A: label + taint a worker, deploy `batch` with toleration + required nodeAffinity, show all replicas on that node with `-o wide`. Then remove the toleration and paste the `FailedScheduling` event line into your notes.
2. **Network posture.** Complete Lab B: default-deny (ingress + egress), DNS allow, frontend → backend allow. Produce the five-line test table (one PASS, four BLOCKED) with real command output.
3. **kube-bench fix.** Complete Lab C: run the benchmark, save `before.txt`, fix one FAIL, re-run only that check, save `after.txt`, and `diff` the PASS/FAIL lines.
4. **Image scan.** Complete Lab D: scan `nginx:1.19`, build `lab51-app:old` and `:new`, compare HIGH+CRITICAL counts and `--exit-code` results.
5. **Policy.** Complete Lab E: deploy the policy in Audit, create a violating Pod, read the policy report, switch to Enforce, and show t1/t2 denied and t3 admitted.

### Stretch (fast finishers / homework)

1. **Complete the three-tier app.** Add a `db` Deployment (`postgres:16`, plain env password from lesson 41) and policies so that only `backend → db:5432` is allowed, `frontend → db` is blocked, and `db` has **no egress at all** except DNS. Prove each with `nc -zv`.
2. **Namespace-wide ingress.** Allow ingress to `backend` from *any* Pod in a namespace labelled `team=web`, using `namespaceSelector`. Test from a Pod in a second namespace before and after labelling it.
3. **Taint-aware policy audit.** Taint the dedicated node `NoExecute` with `tolerationSeconds: 60` on the `batch` Pods and observe the eviction timing with `kubectl get pods -w`.
4. **Soft vs hard affinity.** Switch `batch` from required to `preferredDuringSchedulingIgnoredDuringExecution` (weight 100). Cordon the dedicated node (`kubectl cordon`), scale to 6, and explain where Pods land and why.
5. **Spread across nodes.** Add `topologySpreadConstraints` (`maxSkew: 1`, `topologyKey: kubernetes.io/hostname`, `whenUnsatisfiable: DoNotSchedule`) to `plain`; scale to 7 on a 3-worker cluster and report the distribution.
6. **kube-bench second fix.** Fix a *file permission* FAIL (for example kubeconfig `600`) and an *etcd* FAIL; document risk of each change in two sentences.
7. **Trivy as a gate script.** Write `scan.sh <image>` that exits non-zero when CRITICAL fixed CVEs exist, prints a one-line summary (counts per severity), and honours `.trivyignore`.
8. **Second Kyverno policy.** Require `resources.requests` and `resources.limits` on every container (validate rule, `Audit` first). Generate a policy report and fix one violation.
9. **Third Kyverno policy.** Restrict images to your own registry prefix (`registry.example.local/*`), with an exclusion for system namespaces. Prove `nginx:1.27` is denied.
10. **Write the 1-page runbook.** For each lab, list: symptom, first command to run, most likely cause, fix. This is your on-call cheat sheet.

### Troubleshooting table (instructor reference)

| Symptom | Likely cause | First command |
|---|---|---|
| Pod `Pending`, event "untolerated taint" | missing toleration | `kubectl describe pod` |
| Pod `Pending`, "didn't match Pod's node affinity" | label missing on node or typo in key/value | `kubectl get nodes --show-labels` |
| Toleration present but Pod on wrong node | no affinity/nodeSelector | check `spec.affinity` |
| `wget: bad address` after default-deny | DNS egress not allowed | `kubectl get netpol -n <ns>` |
| Policies have no effect at all | CNI does not enforce NetworkPolicy | `kubectl get pods -A \| grep -E 'calico\|cilium'` |
| Intruder still reaches backend | labels do not match selector / policy in wrong namespace | `kubectl get pods --show-labels` |
| Component missing after manifest edit | YAML error in static pod | `crictl logs`, `journalctl -u kubelet` |
| kube-bench Job stays `Pending`/`Error` | needs control-plane node, tolerations, host mounts | `kubectl describe job kube-bench-master` |
| Check still FAILs after fix | wrong file/flag, component not restarted | `ps aux \| grep <component>` on the node |
| Trivy download is slow/fails | DB download blocked by proxy | `trivy image --skip-db-update` with cached DB |
| `trivy` exit code is 0 but CVEs listed | severity filter excludes them | check `--severity` flag |
| Policy READY false | invalid YAML/pattern or webhook not ready | `kubectl describe cpol` |
| Pod admitted despite policy | policy in Audit, or namespace excluded | check `validationFailureAction` and `exclude` |
| Every Pod rejected cluster-wide | Enforce policy too broad / webhook down with `Fail` | delete policy; check `kubectl -n kyverno get pods` |

## Further Reading

- Kubernetes: Taints and Tolerations — https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
- Kubernetes: Assigning Pods to Nodes (affinity) — https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes: Network Policies — https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Network Policy Recipes (ready-made patterns) — https://github.com/ahmetb/kubernetes-network-policy-recipes
- kube-bench — https://github.com/aquasecurity/kube-bench
- CIS Kubernetes Benchmark — https://www.cisecurity.org/benchmark/kubernetes
- Trivy documentation — https://trivy.dev/latest/docs/
- Kyverno policies library — https://kyverno.io/policies/
- Kyverno: Validate rules — https://kyverno.io/docs/writing-policies/validate/

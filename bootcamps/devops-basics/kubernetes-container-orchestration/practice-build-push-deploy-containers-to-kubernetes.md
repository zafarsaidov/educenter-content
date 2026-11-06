# Lesson 40: Practice – Build, Push, Deploy Containers to Kubernetes

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 30-33 (Docker, Dockerfile, multi-stage builds, Compose, registries), Lessons 34-39 (K8s architecture, kubectl, Pods/Deployments, Services, Ingress, Gateway API). A working cluster (kind, minikube, k3s, rke2 or a lab cluster) with an ingress controller (or Gateway controller) installed.

## Learning Objectives

By end of lesson student can:
- Take a small application from source code to a running, publicly reachable Kubernetes workload without skipping a step
- Write and optimize a multi-stage Dockerfile and measure the size difference
- Tag an image with both a moving tag (`:latest`) and an immutable tag (`:git-sha`) and push both to a registry
- Write Deployment and Service manifests, apply them, and verify with `kubectl get/describe/logs/rollout status`
- Expose the app through an Ingress (or Gateway API HTTPRoute) and test it end-to-end with `curl`
- Perform a rolling update by changing the image tag, watch it happen, and roll back with `kubectl rollout undo`
- Diagnose the four classic deployment failures: `ImagePullBackOff`, `CrashLoopBackOff`, Service with no endpoints, Ingress 404/503
- Clean up all lab resources

## Topics

- Build a multi-stage Dockerfile for a sample app; optimize the image
- Push image to registry with proper tagging (`:latest`, `:git-sha`)
- Write Deployment + Service manifests; apply to cluster; verify with kubectl
- Configure Ingress/Gateway route; test end-to-end HTTP access
- Simulate a rolling update: change image tag, observe rollout

## Concepts

This is a **practice lesson**. Theory is minimal: you already know every piece. The goal is to connect all pieces into one pipeline and to feel where each one breaks. The instructor demos Step 1-3, students then repeat on their own while the instructor walks around.

### The full delivery chain

```
source code ──docker build──> image ──docker tag/push──> registry ──kubectl apply──> Deployment
                                                                                       │
                                                                          ReplicaSet → Pods
                                                                                       │
                                                       Service (ClusterIP, stable VIP + DNS)
                                                                                       │
                                                     Ingress / HTTPRoute (host + path rules)
                                                                                       │
                                                         ingress controller / gateway pod
                                                                                       │
                                                                       client (curl, browser)
```

Every arrow is a place where something can fail. When debugging, walk the chain **from the left** (is the image in the registry? do pods run? does the Service have endpoints? does the Ingress point at the Service?) or **from the right** (curl → ingress → service → pod). Never guess; test each hop.

### Why two tags: `:latest` and `:git-sha`

- `:latest` is a **moving** tag. It points at whatever was pushed last. Convenient for humans, dangerous for deployments: you cannot tell which code runs, and Kubernetes will not re-pull it unless `imagePullPolicy: Always` or the pod is recreated (for `:latest`, the default `imagePullPolicy` is `Always`; for any other tag it is `IfNotPresent`).
- `:git-sha` (e.g. `:3f9c2a1`) is an **immutable** tag. One tag, one build, forever. Deployments reference this tag so that `kubectl rollout history` and `kubectl rollout undo` mean something, and so a rollback goes to exactly the previous code.
- Rule used in production: build once, tag with the SHA, optionally also move `:latest`; Deployment manifests always use the SHA (or a semantic version such as `:1.4.2`).

### Rolling update recap

A Deployment changes Pods by creating a **new ReplicaSet** for the new pod template and scaling it up while the old ReplicaSet scales down. `strategy.rollingUpdate.maxSurge` (extra pods allowed above `replicas`) and `maxUnavailable` (pods allowed to be missing) bound the speed. **Only a change inside `spec.template` triggers a rollout** — changing the image tag does, changing `replicas` does not.

### Lab topology used in this lesson

| Item | Value (replace to match your environment) |
|---|---|
| App name | `hello-k8s` |
| Registry | Docker Hub, user `<DOCKERHUB_USER>` (any registry works) |
| Image | `<DOCKERHUB_USER>/hello-k8s` |
| Namespace | `practice40` |
| Service | `hello-k8s` (ClusterIP, port 80 → container 8080) |
| Hostname | `hello.local` (or `hello.<ingress-ip>.nip.io`) |

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `docker build -t NAME:TAG .` | Build image from Dockerfile in current dir | `docker build -t hello-k8s:dev .` |
| `docker images NAME` | List local images and sizes | `docker images hello-k8s` |
| `docker history NAME:TAG` | Show layers and their sizes | `docker history hello-k8s:dev` |
| `docker tag SRC DST` | Add another name/tag to the same image ID | `docker tag hello-k8s:dev user/hello-k8s:3f9c2a1` |
| `docker push NAME:TAG` | Upload image to registry | `docker push user/hello-k8s:3f9c2a1` |
| `docker run --rm -p 8080:8080 IMG` | Smoke-test locally before deploying | `docker run --rm -p 8080:8080 hello-k8s:dev` |
| `git rev-parse --short HEAD` | Get current short commit SHA | `SHA=$(git rev-parse --short HEAD)` |
| `kubectl create namespace NS` | Create namespace | `kubectl create namespace practice40` |
| `kubectl apply -f FILE` | Create/update from manifest | `kubectl apply -f k8s/` |
| `kubectl get deploy,rs,pods,svc,ing -n NS` | Overview of all lab objects | `kubectl get all,ing -n practice40` |
| `kubectl rollout status deploy/NAME` | Block until rollout completes (or fails) | `kubectl rollout status deploy/hello-k8s -n practice40` |
| `kubectl rollout history deploy/NAME` | List revisions | `kubectl rollout history deploy/hello-k8s -n practice40` |
| `kubectl rollout undo deploy/NAME` | Roll back to previous revision | `kubectl rollout undo deploy/hello-k8s -n practice40` |
| `kubectl set image deploy/NAME CTR=IMG:TAG` | Imperative image change (quick demos) | `kubectl set image deploy/hello-k8s app=user/hello-k8s:v2 -n practice40` |
| `kubectl annotate deploy/NAME kubernetes.io/change-cause="..."` | Record why a revision exists | `kubectl annotate deploy/hello-k8s kubernetes.io/change-cause="v2 add /version" -n practice40` |
| `kubectl get endpoints NAME` / `kubectl get endpointslices` | Does Service have backends? | `kubectl get endpoints hello-k8s -n practice40` |
| `kubectl logs deploy/NAME` | Logs from one pod of the Deployment | `kubectl logs deploy/hello-k8s -n practice40 --tail=20` |
| `kubectl describe pod POD` | Events: pull errors, scheduling, restarts | `kubectl describe pod -l app=hello-k8s -n practice40` |
| `kubectl port-forward svc/NAME L:R` | Bypass ingress to test the Service | `kubectl port-forward svc/hello-k8s 8081:80 -n practice40` |
| `kubectl diff -f FILE` | Preview what apply would change | `kubectl diff -f k8s/deployment.yaml` |
| `kubectl delete namespace NS` | Remove namespace and everything inside | `kubectl delete namespace practice40` |

## Examples / Walkthrough

Total lab time: about 120 minutes. Suggested pacing is in brackets.

### Step 0 – Prepare workspace and verify prerequisites [10 min]

```bash
mkdir -p ~/lab40/hello-k8s/k8s && cd ~/lab40/hello-k8s
git init                                   # we need a git repo to get a commit SHA
docker version --format '{{.Server.Version}}'   # daemon reachable?
kubectl version --short 2>/dev/null || kubectl version   # client + server versions
kubectl get nodes                          # at least one Ready node
kubectl get ingressclass                   # shows nginx (or your controller)
kubectl get gatewayclass                   # only if you use the Gateway path
docker login                               # authenticate to Docker Hub (lesson 33)
```

**Checkpoint 0:** `kubectl get nodes` shows `Ready`; `docker login` says `Login Succeeded`. If the cluster is kind/minikube, note the extra step in Step 4 for loading images without a registry.

### Step 1 – Write the sample application [10 min]

A deliberately tiny Go HTTP server. It prints its version (injected at build time), its hostname (= Pod name, proves load balancing), and exposes `/` and `/version`.

`main.go`:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

// Overridden at build time: -ldflags "-X main.version=..."
var version = "dev"

func main() {
	host, _ := os.Hostname()

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "hello from pod %s (version %s)\n", host, version)
	})
	http.HandleFunc("/version", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, version)
	})

	log.Printf("hello-k8s %s listening on :8080", version)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

`go.mod`:

```
module hello-k8s

go 1.22
```

(Prefer Node or Python? Any app that listens on one port and prints a version works. Adjust the Dockerfile accordingly; the Kubernetes part is identical.)

Commit it so we have a SHA:

```bash
git add . && git commit -m "hello-k8s v1"
git rev-parse --short HEAD        # e.g. 3f9c2a1
```

### Step 2 – Naive single-stage Dockerfile (baseline) [10 min]

First write the *bad* version on purpose to measure the improvement.

`Dockerfile.naive`:

```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
```

```bash
docker build -f Dockerfile.naive -t hello-k8s:naive .
docker images hello-k8s
```

Expected output (sizes approximate):

```
REPOSITORY   TAG     IMAGE ID       CREATED          SIZE
hello-k8s    naive   a1b2c3d4e5f6   10 seconds ago   850MB
```

850 MB to run a 6 MB binary: the image ships the Go toolchain, the package cache, and the source code. Slow to pull on every node, big attack surface.

### Step 3 – Optimized multi-stage Dockerfile [15 min]

`Dockerfile`:

```dockerfile
# ---------- build stage ----------
FROM golang:1.22-alpine AS build
WORKDIR /src

# copy dependency manifests first -> better layer cache
COPY go.mod ./
RUN go mod download

COPY . .
ARG VERSION=dev
# static binary, no libc dependency, smaller (-s -w strips debug info)
RUN CGO_ENABLED=0 GOOS=linux go build \
      -ldflags="-s -w -X main.version=${VERSION}" \
      -o /out/server .

# ---------- runtime stage ----------
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

`.dockerignore`:

```
.git
k8s
*.md
Dockerfile*
```

Build with the version baked in, using the git SHA:

```bash
SHA=$(git rev-parse --short HEAD)
docker build --build-arg VERSION=$SHA -t hello-k8s:$SHA .
docker images hello-k8s
```

Expected:

```
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
hello-k8s    3f9c2a1   9f8e7d6c5b4a   5 seconds ago    4.1MB
hello-k8s    naive     a1b2c3d4e5f6   3 minutes ago    850MB
```

From 850 MB to about 4 MB. Show layers:

```bash
docker history hello-k8s:$SHA
```

Smoke test **before** pushing (never push an image you have not run once):

```bash
docker run --rm -d -p 8080:8080 --name smoke hello-k8s:$SHA
curl -s localhost:8080/            # hello from pod <container-id> (version 3f9c2a1)
curl -s localhost:8080/version     # 3f9c2a1
docker stop smoke
```

**Checkpoint 3:** `curl /version` returns the SHA you built. Image is under ~20 MB. Container runs as non-root (`docker run --rm hello-k8s:$SHA` works with distroless `nonroot`).

Optimization discussion (instructor, 5 min):
- Why copy `go.mod` before the source? Dependency layer is reused until `go.mod` changes.
- Why distroless/static? No shell, no package manager: smaller attack surface, nothing for an attacker to run. Trade-off: `kubectl exec -- sh` will not work (see Pitfalls; use `kubectl debug` ephemeral container).
- Why `USER nonroot`? Container escape and file-write risks drop; some clusters enforce non-root.

### Step 4 – Tag and push to the registry [10 min]

```bash
export REG_USER=<DOCKERHUB_USER>
SHA=$(git rev-parse --short HEAD)

docker tag hello-k8s:$SHA $REG_USER/hello-k8s:$SHA
docker tag hello-k8s:$SHA $REG_USER/hello-k8s:latest

docker push $REG_USER/hello-k8s:$SHA
docker push $REG_USER/hello-k8s:latest
```

Both tags point to the same image ID — the second push uploads almost nothing:

```
The push refers to repository [docker.io/user/hello-k8s]
5f70bf18a086: Layer already exists
latest: digest: sha256:7c1d... size: 738
```

Verify from a clean point of view (the cluster will see the image like this):

```bash
docker rmi $REG_USER/hello-k8s:$SHA                 # remove local copy
docker pull $REG_USER/hello-k8s:$SHA                # prove registry has it
docker image inspect $REG_USER/hello-k8s:$SHA --format '{{index .RepoDigests 0}}'
```

**Private repository?** Make the Docker Hub repo private and create a pull secret (one-liner, details in lesson 42):

```bash
kubectl create secret docker-registry regcred \
  --docker-username=$REG_USER --docker-password='<TOKEN>' \
  --docker-server=https://index.docker.io/v1/ -n practice40
```

and add `imagePullSecrets: [{name: regcred}]` under the Pod `spec` in Step 5. For this lab, a **public** repo is simpler.

**kind / minikube without a registry:** load the image into the node instead of pushing:

```bash
kind load docker-image hello-k8s:$SHA        # kind
minikube image load hello-k8s:$SHA           # minikube
```

and then set `imagePullPolicy: IfNotPresent` and the image to `hello-k8s:$SHA` (no registry prefix).

**Checkpoint 4:** `docker pull` of the SHA tag from the registry works; the web UI shows both tags.

### Step 5 – Write Deployment and Service manifests [15 min]

Generate a skeleton instead of typing from memory (lesson 36 trick):

```bash
kubectl create namespace practice40 --dry-run=client -o yaml > k8s/00-namespace.yaml
kubectl create deployment hello-k8s -n practice40 \
  --image=$REG_USER/hello-k8s:$SHA --replicas=3 --port=8080 \
  --dry-run=client -o yaml > k8s/10-deployment.yaml
```

Edit `k8s/10-deployment.yaml` to the final form:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-k8s
  namespace: practice40
  labels:
    app: hello-k8s
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: hello-k8s
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: hello-k8s
    spec:
      containers:
        - name: app
          image: <DOCKERHUB_USER>/hello-k8s:3f9c2a1   # immutable SHA tag
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 64Mi
```

Notes for the instructor:
- `selector.matchLabels` must equal `template.metadata.labels`, and is **immutable** after creation.
- `maxUnavailable: 0` + `maxSurge: 1` = zero-downtime: a new pod must be created before an old one is removed. Without probes (lesson 45) "created" means "container started", not "ready to serve" — we will fix this later.
- Container is named `app` — that name is used by `kubectl set image`.

`k8s/20-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-k8s
  namespace: practice40
spec:
  type: ClusterIP
  selector:
    app: hello-k8s          # must match pod labels, NOT the Deployment name
  ports:
    - name: http
      port: 80              # Service port
      targetPort: 8080      # container port
```

Preview and apply:

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl diff -f k8s/                       # shows what will be created
kubectl apply -f k8s/
kubectl rollout status deploy/hello-k8s -n practice40
```

Expected:

```
deployment "hello-k8s" successfully rolled out
```

Verify every layer:

```bash
kubectl get deploy,rs,pods,svc -n practice40 -o wide
kubectl get endpoints hello-k8s -n practice40
kubectl logs deploy/hello-k8s -n practice40
kubectl describe svc hello-k8s -n practice40
```

Expected pods and endpoints:

```
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE
pod/hello-k8s-6d4cf56db6-4xk9q   1/1     Running   0          30s   10.244.1.12   worker-1
pod/hello-k8s-6d4cf56db6-m8tqz   1/1     Running   0          30s   10.244.2.7    worker-2
pod/hello-k8s-6d4cf56db6-z2n5w   1/1     Running   0          30s   10.244.1.13   worker-1

NAME              ENDPOINTS                                        AGE
hello-k8s         10.244.1.12:8080,10.244.1.13:8080,10.244.2.7:8080   30s
```

Three pod IPs listed = Service selector matches. **Empty endpoints = selector/label mismatch** (most common Service bug).

Test the Service **without** ingress, via port-forward:

```bash
kubectl port-forward svc/hello-k8s 8081:80 -n practice40 &
curl -s localhost:8081/           # hello from pod hello-k8s-... (version 3f9c2a1)
kill %1
```

Test in-cluster DNS from a throwaway pod (lesson 37):

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -n practice40 -- \
  wget -qO- http://hello-k8s/version
# 3f9c2a1
```

**Checkpoint 5:** 3/3 pods Running, 3 endpoints, `/version` returns the SHA through the Service.

### Step 6 – Expose via Ingress (path A) [15 min]

`k8s/30-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-k8s
  namespace: practice40
spec:
  ingressClassName: nginx
  rules:
    - host: hello.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-k8s
                port:
                  number: 80
```

```bash
kubectl apply -f k8s/30-ingress.yaml
kubectl get ingress -n practice40
```

Expected:

```
NAME        CLASS   HOSTS         ADDRESS        PORTS   AGE
hello-k8s   nginx   hello.local   192.168.1.50   80      20s
```

Find the ingress entry point and test:

```bash
# bare-metal / lab: NodePort of the controller or LoadBalancer IP
kubectl get svc -n ingress-nginx
IP=192.168.1.50                           # ADDRESS from above

# no DNS record? send the Host header manually
curl -s -H "Host: hello.local" http://$IP/
# or: --resolve hello.local:80:$IP
curl -s --resolve hello.local:80:$IP http://hello.local/version
```

Load-balancing proof (different pod names appear):

```bash
for i in $(seq 1 10); do curl -s -H "Host: hello.local" http://$IP/; done | sort | uniq -c
#   4 hello from pod hello-k8s-6d4cf56db6-4xk9q (version 3f9c2a1)
#   3 hello from pod hello-k8s-6d4cf56db6-m8tqz (version 3f9c2a1)
#   3 hello from pod hello-k8s-6d4cf56db6-z2n5w (version 3f9c2a1)
```

If you have no ingress controller IP, use `hello.<IP>.nip.io` as host (nip.io resolves to the IP embedded in the name) and edit `host:` accordingly.

### Step 6b – Alternative: Gateway API HTTPRoute (path B) [optional, 10 min]

Only if a Gateway controller and a `Gateway` named `public-gw` exist (lesson 39). Delete the Ingress first so you do not publish the same host twice.

```bash
kubectl delete -f k8s/30-ingress.yaml
```

`k8s/30-httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-k8s
  namespace: practice40
spec:
  parentRefs:
    - name: public-gw
      namespace: gateway-system      # namespace where the Gateway lives
  hostnames:
    - hello.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: hello-k8s
          port: 80
```

```bash
kubectl apply -f k8s/30-httproute.yaml
kubectl get httproute -n practice40
kubectl describe httproute hello-k8s -n practice40     # look at Accepted / ResolvedRefs
kubectl get gateway -A                                  # ADDRESS column
curl -s --resolve hello.local:80:<GATEWAY_IP> http://hello.local/version
```

`Accepted=True` and `ResolvedRefs=True` in the route status mean the Gateway took the route and the backend Service was found. If the Gateway lives in another namespace and does not allow routes from `practice40`, edit `allowedRoutes` on the listener (or use the correct namespace policy) — that is the Gateway API role split from lesson 39.

**Checkpoint 6:** `curl` from outside the cluster returns the hello text with the correct version, and repeated calls hit different pod names.

### Step 7 – Rolling update by changing the image tag [20 min]

Make a visible code change, build a new SHA, push, and update the manifest.

```bash
# edit main.go: change the greeting text
sed -i.bak 's/hello from pod/HELLO v2 from pod/' main.go && rm main.go.bak
git commit -am "hello-k8s v2: louder greeting"

SHA2=$(git rev-parse --short HEAD)
docker build --build-arg VERSION=$SHA2 -t $REG_USER/hello-k8s:$SHA2 -t $REG_USER/hello-k8s:latest .
docker push $REG_USER/hello-k8s:$SHA2
docker push $REG_USER/hello-k8s:latest
```

Open **two terminals**.

Terminal A — watch pods change live:

```bash
kubectl get pods -n practice40 -w
```

Terminal B — continuous traffic so students *see* mixed versions during the rollout and absence of errors:

```bash
while true; do curl -s -o /dev/null -w "%{http_code} " -H "Host: hello.local" http://$IP/; sleep 0.3; done
# 200 200 200 200 ...  any 502/503 here means a gap in availability
```

Terminal C — do the update the **declarative** way (preferred; manifest in Git stays the source of truth):

```bash
sed -i.bak "s#hello-k8s:$SHA#hello-k8s:$SHA2#" k8s/10-deployment.yaml && rm k8s/10-deployment.yaml.bak
kubectl diff -f k8s/10-deployment.yaml          # shows only the image line changing
kubectl apply -f k8s/10-deployment.yaml
kubectl annotate deploy/hello-k8s -n practice40 kubernetes.io/change-cause="v2 louder greeting ($SHA2)" --overwrite
kubectl rollout status deploy/hello-k8s -n practice40
```

In Terminal A you will see the pattern (maxSurge=1, maxUnavailable=0):

```
hello-k8s-7b8d9c5f4-abcde   0/1   Pending             0     0s
hello-k8s-7b8d9c5f4-abcde   0/1   ContainerCreating   0     0s
hello-k8s-7b8d9c5f4-abcde   1/1   Running             0     3s
hello-k8s-6d4cf56db6-4xk9q  1/1   Terminating         0     6m
... repeated 3 times ...
```

Inspect what happened:

```bash
kubectl get rs -n practice40
# NAME                   DESIRED   CURRENT   READY   AGE
# hello-k8s-6d4cf56db6   0         0         0       8m     <- old, kept for rollback
# hello-k8s-7b8d9c5f4    3         3         3       1m     <- new

kubectl rollout history deploy/hello-k8s -n practice40
kubectl rollout history deploy/hello-k8s -n practice40 --revision=2   # pod template of revision 2
curl -s -H "Host: hello.local" http://$IP/version                     # now the new SHA
```

The old ReplicaSet with 0 replicas is the rollback target; `revisionHistoryLimit: 5` caps how many are kept.

Quick imperative alternative (fine for demos, bad for Git hygiene because the manifest is now out of sync):

```bash
kubectl set image deploy/hello-k8s app=$REG_USER/hello-k8s:$SHA2 -n practice40
```

### Step 8 – Roll back [10 min]

Simulate a **bad release**: point to a tag that does not exist.

```bash
kubectl set image deploy/hello-k8s app=$REG_USER/hello-k8s:does-not-exist -n practice40
kubectl rollout status deploy/hello-k8s -n practice40 --timeout=60s
# Waiting for deployment "hello-k8s" rollout to finish: 1 out of 3 new replicas have been updated...
# error: timed out waiting for the condition
```

```bash
kubectl get pods -n practice40
# hello-k8s-5f6c7d8b9-qwert   0/1   ImagePullBackOff   0   70s
# the 3 old pods are still Running -> maxUnavailable: 0 protected the service
kubectl describe pod -l app=hello-k8s -n practice40 | grep -A3 -i "failed"
# Failed to pull image ".../hello-k8s:does-not-exist": not found
```

The curl loop in Terminal B still shows only `200` — the old pods keep serving because the Deployment refuses to remove them until new ones are up. That is the value of `maxUnavailable: 0`.

Roll back:

```bash
kubectl rollout undo deploy/hello-k8s -n practice40                    # back to previous revision
kubectl rollout status deploy/hello-k8s -n practice40
kubectl rollout history deploy/hello-k8s -n practice40
kubectl rollout undo deploy/hello-k8s -n practice40 --to-revision=1    # go back to the original v1
```

After `undo`, manifest in Git and cluster differ. **Fix the Git side too**: revert the image line in `10-deployment.yaml` (or `git revert`) so the next `kubectl apply -f` does not redeploy the bad version.

### Step 9 – Scale, observe, clean up [10 min]

```bash
kubectl scale deploy/hello-k8s --replicas=5 -n practice40       # replicas change -> NO new revision
kubectl get pods -n practice40 -o wide                           # spread across nodes
kubectl scale deploy/hello-k8s --replicas=3 -n practice40

# delete a pod: ReplicaSet recreates it (self-healing)
kubectl delete pod -l app=hello-k8s -n practice40 --wait=false
kubectl get pods -n practice40 -w                                 # new pods appear immediately

# cleanup
kubectl delete namespace practice40                               # removes everything inside
kubectl get all -n practice40                                     # "No resources found" / namespace gone
docker rmi hello-k8s:naive
```

If you created a lab-only image repository on Docker Hub, delete it in the web UI after the course.

## Troubleshooting Table (keep on the wall)

| Symptom | Where to look | Likely cause | Fix |
|---|---|---|---|
| Pod `ImagePullBackOff` / `ErrImagePull` | `kubectl describe pod` → Events | Wrong image name/tag, tag not pushed, private repo without pull secret, registry rate limit | Fix tag; `docker pull` same string from your laptop; add `imagePullSecrets` |
| Pod `CrashLoopBackOff` | `kubectl logs POD --previous` | App exits immediately: wrong ENTRYPOINT, missing binary, wrong architecture (`exec format error` on arm64 vs amd64) | Run image locally with `docker run`; build with `--platform linux/amd64` if needed |
| Pod `Pending` | `kubectl describe pod` → Events | No node with enough CPU/memory (requests too high), no nodes Ready | Lower requests or add capacity |
| Pod `Running` but Service has no endpoints | `kubectl get endpoints SVC` | `selector` labels do not match pod labels | Align `spec.selector` with `template.metadata.labels` |
| Endpoints exist, connection refused | `kubectl port-forward` + curl | `targetPort` ≠ port the app listens on; app listens on `127.0.0.1` only | Fix `targetPort`; bind app to `0.0.0.0` |
| Ingress 404 | `kubectl describe ingress`, controller logs | Host header does not match `rules.host`; wrong `ingressClassName`; path mismatch | Use `curl -H "Host: ..."`; check `kubectl get ingressclass` |
| Ingress 503 | `kubectl describe ingress` → Backends | Service name/port wrong, or Service has no endpoints | Verify backend `service.name`/`port.number`; check endpoints |
| Ingress has no ADDRESS | `kubectl get pods -n ingress-nginx` | Controller not installed or LoadBalancer IP pending (bare metal) | Use controller NodePort, or install MetalLB |
| HTTPRoute not `Accepted` | `kubectl describe httproute` → Conditions | `parentRefs` name/namespace wrong, listener `allowedRoutes` blocks namespace, hostname mismatch with listener | Correct parentRefs; allow namespace on Gateway listener |
| Rollout stuck at "1 out of 3 updated" | `kubectl rollout status`, `get rs` | New pods failing (pull/crash) | `kubectl rollout undo`; fix image; re-apply |
| `apply` says `unchanged` after editing image | `kubectl diff` | Edited a different file/namespace, or tag is the same string | Check namespace in manifest and `-n` flag; confirm the diff |
| Same tag pushed again, pods not updated | `kubectl get pod -o jsonpath='{..imageID}'` | `IfNotPresent` + mutable tag: nodes use the cached image | Use a new immutable tag (the point of `:git-sha`) |

## Common Pitfalls

- **Deploying `:latest` and expecting a rollout on every push** — Kubernetes only rolls out when the pod template changes. Re-pushing `:latest` changes nothing in the manifest, so no new pods. Use unique tags per build.
- **Using `:latest` in production manifests** — you cannot tell which code runs, `rollout undo` goes to a tag that now points to the *new* code, and `imagePullPolicy` defaults to `Always` (every restart re-pulls). Pin SHA or semver.
- **Forgetting `-n practice40`** — objects land in `default`, `get` shows nothing. Use `kubectl config set-context --current --namespace=practice40` (lesson 35) or put `namespace:` in every manifest.
- **Service `selector` pointing at the Deployment name** — the selector matches **pod labels** (`app: hello-k8s`), never object names.
- **Changing `spec.selector` of an existing Deployment** — immutable; delete and recreate the Deployment (or keep the label set stable from day one).
- **Pushing an image built on Apple Silicon to an amd64 cluster** — pods crash with `exec format error`. Build with `docker build --platform linux/amd64` or `docker buildx build --platform linux/amd64,linux/arm64`.
- **Distroless image + `kubectl exec -- sh`** — no shell exists. Use `kubectl debug -it POD --image=busybox:1.36 --target=app` (ephemeral container) or test in a throwaway pod.
- **App listens on `localhost`** — container works with `docker run -p` on some setups but not through Service/Ingress. Bind `0.0.0.0` (`:8080` in Go).
- **Editing the cluster with `kubectl set image`/`edit` and never updating Git** — next `apply -f` silently reverts the change. Declarative manifest in Git is the source of truth.
- **Rollback without fixing the manifest** — `rollout undo` restores the cluster, not your repository; a later `apply` re-deploys the bad version.
- **Testing Ingress without the Host header** — nginx routes by host; plain `curl http://IP/` hits the default backend (404). Always `-H "Host: ..."` or `--resolve`.
- **Secrets in the image or in `docker build --build-arg`** — build args are visible in `docker history`. Never bake credentials in; configuration injection is lessons 42-43.
- **Skipping the local smoke test** — a crashing image costs a full push/pull/deploy cycle to discover. `docker run` first.

## FAQ

**Q: Why build the image with the git SHA instead of a version number like 1.0.0?**
A: A SHA is generated automatically, unique per commit, and traces any running pod back to exact source code. Semantic versions are great for human-released artifacts; CI usually pushes both (SHA on every commit, semver on release tags).

**Q: Do I still need `:latest` if I deploy by SHA?**
A: No. Many teams keep it only for convenience (`docker run user/app` quickly). Never reference it in Deployment manifests.

**Q: How does Kubernetes know a new image version exists?**
A: It does not poll registries. A rollout happens only when you change the pod template (new tag in the manifest, `kubectl set image`, or `kubectl rollout restart`). GitOps tools and CI automate this change.

**Q: What is the difference between `kubectl rollout restart` and changing the image?**
A: `rollout restart` adds a timestamp annotation to the pod template, so all pods are replaced with the *same* image spec (re-pulling only if `imagePullPolicy: Always` or the image is not cached). Changing the image tag deploys different code.

**Q: Why did my rollout take so long with `maxUnavailable: 0`?**
A: Each new pod must start before an old one is stopped; with `maxSurge: 1` pods are replaced one at a time. Raise `maxSurge` (for example to `25%` or `3`) to speed it up at the cost of temporary extra resources.

**Q: Does the Deployment wait until the app is actually ready?**
A: Not yet. Without a readiness probe, "ready" = container process started. For slow-starting apps a rollout can send traffic too early. Readiness/liveness probes come in lesson 45.

**Q: Can I use a local image without a registry?**
A: In kind and minikube yes (`kind load docker-image`, `minikube image load`) plus `imagePullPolicy: IfNotPresent`. On a multi-node real cluster every node must be able to pull the image, so a registry is required.

**Q: `ImagePullBackOff` but the image exists on Docker Hub. Why?**
A: Check for: typo in user/repo, private repo without `imagePullSecrets`, Docker Hub anonymous pull rate limit (`toomanyrequests` in the events), wrong architecture manifest, or a node with no outbound internet access.

**Q: How do I see which image each pod runs?**
A: `kubectl get pods -n practice40 -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'` or `kubectl get pods -o wide` (IMAGES column with `-o wide` on deployments: `kubectl get deploy -o wide`).

**Q: What does `kubectl rollout undo` actually do?**
A: It re-applies the pod template of the previous revision (which is the previous ReplicaSet) and scales it up while scaling the current one down. It creates a *new* revision number pointing at the old template.

**Q: How many old revisions are kept?**
A: `spec.revisionHistoryLimit` (default 10). Each is an old ReplicaSet scaled to 0. Setting it to 0 disables rollback.

**Q: Ingress or Gateway API for this lab?**
A: Either works. Ingress is simpler and still the most common in existing clusters; Gateway API is the newer role-oriented standard. The Service and Deployment parts are identical — only the routing object changes.

**Q: Why `port: 80` on the Service when the container uses 8080?**
A: The Service port is what clients inside the cluster use (`http://hello-k8s`), while `targetPort` is the container's real port. Decoupling lets you keep a standard port in front of any app.

**Q: Why is the final image only ~4 MB?**
A: The final stage starts from distroless/static (no OS userland) and copies only the static binary; the Go compiler and sources stay in the discarded build stage.

**Q: Can I put the Deployment, Service, and Ingress into one YAML file?**
A: Yes, separate documents with `---`. Separate numbered files (`10-`, `20-`, `30-`) are easier to review and `kubectl apply -f k8s/` applies the whole directory in name order.

## Practice / Exercise

### Core (everyone, in class)

1. Complete Steps 0-9 with your own `<DOCKERHUB_USER>` and namespace `practice40-<yourname>`.
2. Show the image size of the naive and the multi-stage build side by side (`docker images`) and explain the difference in one sentence.
3. Push two tags (`:<sha>` and `:latest`) and prove with `docker image inspect` or the registry UI that they share one digest.
4. Apply Deployment (3 replicas) + Service + Ingress (or HTTPRoute); show `kubectl get endpoints` with three addresses.
5. From outside the cluster run the `curl` loop and show at least two different pod names in the output.
6. Perform the v1 → v2 rolling update while the curl loop runs; show that no non-200 code appeared.
7. Break the release with a non-existing tag, show `ImagePullBackOff` and that the old pods still serve traffic, then `kubectl rollout undo`.
8. Delete your namespace and verify nothing is left.

### Stretch (fast finishers)

1. Add `kubernetes.io/change-cause` annotations to every release and show a readable `kubectl rollout history`.
2. Change `maxSurge: 100%` and `maxUnavailable: 0`; compare rollout duration against the default of 1.
3. Switch the strategy to `Recreate`, run the update with the curl loop, and measure the downtime window. Explain when that strategy is acceptable.
4. Build a multi-arch image with `docker buildx build --platform linux/amd64,linux/arm64 --push` and inspect it with `docker buildx imagetools inspect`.
5. Pin by digest: set `image: user/hello-k8s@sha256:<digest>` and explain what that guarantees compared to a tag.
6. Use `kubectl debug -it POD --image=busybox:1.36 --target=app` to inspect the running distroless container.
7. Deploy the same image a second time as `hello-k8s-canary` (1 replica) with the same `app` label *and* a `track: canary` label; observe how the Service splits traffic roughly 3:1 with the stable pods, then remove it.
8. Write a one-page `RELEASE.md` runbook (build → tag → push → apply → verify → rollback) for a colleague.

### Homework

Repeat the whole flow with a different language (Node.js or Python) and a different registry (GitLab/GitHub registry or a self-hosted `registry:2` from lesson 33), then compare image sizes and cold-start pull times.

## Further Reading

- Kubernetes docs – Deployments (rolling update, rollback): https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes docs – Images (tags, digests, pull policy, pull secrets): https://kubernetes.io/docs/concepts/containers/images/
- Kubernetes docs – Service: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes docs – Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
- Gateway API – HTTPRoute: https://gateway-api.sigs.k8s.io/api-types/httproute/
- Docker docs – Multi-stage builds: https://docs.docker.com/build/building/multi-stage/
- Docker docs – Multi-platform images: https://docs.docker.com/build/building/multi-platform/
- Distroless images: https://github.com/GoogleContainerTools/distroless

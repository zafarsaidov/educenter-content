# Lesson 39: Gateway API – Modern Traffic Management

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lesson 36 (Deployments), Lesson 37 (Services), Lesson 38 (Ingress, IngressClass, TLS termination, cert-manager, CoreDNS)

## Learning Objectives

By end of lesson student can:
- Explain the limitations of Ingress that motivated Gateway API.
- Describe the role-oriented model: infrastructure provider, cluster operator, application developer.
- Install the Gateway API CRDs and a gateway controller, and verify the `GatewayClass` is `Accepted`.
- Create a `Gateway` with HTTP and HTTPS listeners and restrict which namespaces may attach routes.
- Write `HTTPRoute` objects with host, path, header, query-param and method matches.
- Use `RequestRedirect`, `URLRewrite`, `RequestHeaderModifier` and `RequestMirror` filters.
- Split traffic between two Services with `backendRefs` weights (canary, blue/green).
- Explain `TCPRoute`, `TLSRoute`, `GRPCRoute` and the Experimental vs Standard release channels.
- Use `ReferenceGrant` to allow cross-namespace references and explain why it exists.
- Read `status.conditions` on GatewayClass, Gateway and HTTPRoute to debug routing.
- Know that `ingress2gateway` exists to migrate existing Ingress objects.

## Topics

- Gateway API vs Ingress: Ingress limitations, role-oriented model (Infra / Cluster / App)
- Core resources: GatewayClass (provider), Gateway (listener), HTTPRoute (routing rules)
- HTTPRoute: rules, matches (path, header, method), filters (redirect, rewrite), backendRefs
- Advanced routing: TCPRoute, GRPCRoute (concept); ReferenceGrant (cross-namespace traffic)
- Hands-on: deploy a gateway controller, create HTTPRoute, test traffic routing

## Concepts

### Recap: what Ingress gave us and where it hurts

Lesson 38: an `Ingress` object maps `host` + `path` to a Service, a controller (nginx-ingress) implements it, `IngressClass` picks the controller, `tls:` terminates HTTPS. It works, but it has structural problems:

1. **Too little in the spec.** Ingress only knows host, path (`Prefix`/`Exact`/`ImplementationSpecific`) and TLS. Everything else — rewrites, redirects, header manipulation, timeouts, rate limits, canary — lives in **controller-specific annotations** (`nginx.ingress.kubernetes.io/...`). Annotations are untyped strings: no validation, no portability. Move from nginx to Traefik and you rewrite every Ingress.
2. **HTTP only.** No standard way to expose TCP, UDP, gRPC or TLS-passthrough. Each controller invented its own CRDs or ConfigMap tricks (nginx `tcp-services` ConfigMap).
3. **No role separation.** One object holds both infrastructure concerns (which load balancer, which certificates, which ports) and application concerns (which path goes to which Service). A developer who only wants to publish `/api` must be allowed to edit an object that can also hijack the whole hostname.
4. **No traffic splitting, no header/method/query matching** in the standard spec.
5. **No cross-namespace story.** An Ingress can only reference Services in its own namespace.
6. **One-controller-per-class, flat model.** Hard to express "shared corporate gateway, many teams".

**Gateway API** is the successor, developed by the Kubernetes SIG-Network as a set of **CRDs that are NOT built into Kubernetes core** — you install them separately (`gateway.networking.k8s.io`). Ingress is frozen (feature-complete, no new features) but not deprecated or removed; Gateway API is where new work goes.

### Role-oriented model

Gateway API splits one object into three, each owned by a different persona:

| Persona | Owns | Resource |
|---|---|---|
| **Infrastructure provider** (cloud vendor, platform vendor, controller authors) | The *kind* of load balancer / proxy implementation | `GatewayClass` (cluster-scoped) |
| **Cluster operator** (platform / SRE team) | A concrete entry point: listeners, ports, protocols, TLS certs, which namespaces may attach | `Gateway` (namespaced) |
| **Application developer** | Routing rules for their app: paths, headers, backends | `HTTPRoute`, `GRPCRoute`, `TCPRoute`, ... (namespaced) |

Analogy from lesson 38: `GatewayClass` ≈ `IngressClass`; `Gateway` ≈ "the Ingress controller's Service + listeners + certs"; `HTTPRoute` ≈ "the rules part of Ingress".

Because they are separate objects, RBAC (lesson 47) can give developers `create/update httproutes` in their namespace and nothing on `gateways`. The platform team retains control of ports and certificates; developers cannot steal a hostname unless the Gateway allows it.

### How the pieces connect

```
GatewayClass  (cluster)   "controller: example.net/gateway-controller"
     ^
     | gatewayClassName
Gateway       (ns infra)  listeners: http:80, https:443 (hostname, TLS cert, allowedRoutes)
     ^
     | parentRefs
HTTPRoute     (ns app)    hostnames, rules -> matches, filters, backendRefs
     |
     v
Service  ->  Endpoints/Pods
```

- A **Route points up** to a Gateway through `parentRefs`. A Gateway never lists its routes.
- A **Gateway points up** to a class through `gatewayClassName`.
- A Route points **down** to Services through `backendRefs`.
- The Gateway's `listeners[].allowedRoutes` decides which Routes (by namespace and kind) are *allowed* to attach. This is the "handshake": both sides must agree (Route asks via `parentRefs`, Gateway permits via `allowedRoutes`).

The controller watches all three, programs a data plane (Envoy, nginx, HAProxy, cloud LB, Istio, Cilium ...) and writes the result into `status`.

### Implementations

Gateway API is only a specification. Real traffic is handled by an implementation. Common ones:

| Implementation | Data plane | Notes |
|---|---|---|
| Envoy Gateway | Envoy | CNCF project, reference-style, easy install |
| NGINX Gateway Fabric | nginx | From the nginx team; natural step from nginx-ingress |
| Istio | Envoy | Gateway API is first-class for ingress and mesh (Ambient mode) |
| Cilium | eBPF + Envoy | Built into the CNI |
| Traefik | Traefik | Supports Gateway API provider |
| Kong, HAProxy, Contour, GKE Gateway, AWS LBC ... | various | Cloud-integrated |

Each implementation publishes a **conformance report**: which features of the spec it supports (core / extended / implementation-specific). Always check support before relying on an extended feature.

### Release channels and API versions

- **Standard channel**: only GA/Beta stable resources — `GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`, `ReferenceGrant` (versions `v1`, `v1beta1`). Safe for production.
- **Experimental channel**: superset including alpha resources — `TCPRoute`, `UDPRoute`, `TLSRoute`, `BackendTLSPolicy` (some graduated since), and experimental fields. Can change or disappear.
- The CRD bundle you install decides what you can use: `standard-install.yaml` vs `experimental-install.yaml`.

Check the version you run: `kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.metadata.annotations}'` shows `gateway.networking.k8s.io/bundle-version` and `/channel`.

### GatewayClass

Cluster-scoped. Names a controller. Think "StorageClass for load balancers" (concept only — storage comes in lesson 41).

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

- `controllerName` — a string the controller recognises; it only reconciles classes naming it.
- `parametersRef` (optional) — points to implementation-specific config (replicas, service type of the proxy, resources).
- `status.conditions[type=Accepted]` — `True` when the controller took ownership. If `Unknown`/`False`, nothing downstream will ever work.
- Usually created by the controller's install (Helm chart / manifest) or by the cluster operator.

### Gateway

Namespaced. Represents **a request for a data-plane entry point**. The controller provisions it — typically a Deployment + a `LoadBalancer` Service per Gateway (implementation-dependent) — and publishes the address in `status.addresses`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: wildcard-example-com
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              shared-gateway-access: "true"
```

Field walk-through:

- `gatewayClassName` — which implementation.
- `listeners[]` — each listener is a (port, protocol, optional hostname) tuple. Names must be unique within the Gateway; Routes can target one with `parentRefs[].sectionName`.
- `protocol` — `HTTP`, `HTTPS`, `TLS`, `TCP`, `UDP`.
- `hostname` — optional; restricts the listener to that SNI/Host (wildcard allowed). Routes attaching must have intersecting hostnames.
- `tls.mode` — `Terminate` (decrypt at the gateway; certificate from `certificateRefs`) or `Passthrough` (forward encrypted bytes; route by SNI with `TLSRoute`).
- `tls.certificateRefs` — Secret of type `kubernetes.io/tls` in the Gateway's namespace (or another namespace with a `ReferenceGrant`). How that Secret is produced is covered in lesson 38 (cert-manager) and lesson 42 (Secrets); here we just reference it by name.
- `allowedRoutes.namespaces.from` — `Same` (default: only the Gateway's namespace), `All`, or `Selector` (label selector on namespaces). `allowedRoutes.kinds` can additionally restrict Route kinds.
- `status.addresses` — external IP/hostname once provisioned.
- `status.listeners[].attachedRoutes` — how many Routes bound to each listener.

**cert-manager integration:** cert-manager can watch Gateways: annotate the Gateway with `cert-manager.io/cluster-issuer: letsencrypt-prod` and it creates the Certificate/Secret for each HTTPS listener hostname (requires enabling the Gateway API feature in cert-manager's config). Same idea as the Ingress annotation shim from lesson 38.

### HTTPRoute

Namespaced. Attaches to Gateway(s) and describes how HTTP requests are matched, modified and forwarded.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop
  namespace: shop
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
      sectionName: https
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web
          port: 80
```

Structure:

- `parentRefs[]` — Gateways (or other parents, e.g. Services in mesh use) to attach to: `name`, `namespace` (default: Route's), `sectionName` (listener name), `port`.
- `hostnames[]` — Host header values (wildcards `*.example.com` allowed). Omitted = all hostnames the listener accepts. Must intersect the listener's hostname.
- `rules[]` — evaluated by precedence (below); each rule = `matches` + `filters` + `backendRefs` (+ `timeouts`, `sessionPersistence` in newer versions).
- `matches[]` — OR between entries; inside one entry all conditions are AND.
- `backendRefs[]` — Services (kind default `Service`) with `port` (required for Services) and optional `weight`.

### Matches

Inside a `matches` entry:

| Field | Values | Meaning |
|---|---|---|
| `path.type` | `Exact`, `PathPrefix`, `RegularExpression` (implementation-specific) | `PathPrefix: /api` matches `/api` and `/api/x`, **not** `/apiary` (element-wise prefix, same idea as Ingress `Prefix` in lesson 38) |
| `headers[]` | `name`, `value`, `type: Exact` / `RegularExpression` | Match request header |
| `queryParams[]` | `name`, `value`, `type` | Match query string |
| `method` | `GET`, `POST`, ... | HTTP verb |

Example — only POST to `/orders` carrying a version header:

```yaml
matches:
  - path:
      type: Exact
      value: /orders
    method: POST
    headers:
      - name: x-api-version
        value: "2"
```

**Precedence** (when several rules match one request) — defined by the spec, *not* by order in the file:
1. Hostname: most specific (exact over wildcard).
2. `Exact` path over `PathPrefix`; longest `PathPrefix` first.
3. Method match present beats absent.
4. More header matches beat fewer.
5. More query-param matches beat fewer.
6. Tie: oldest Route by `creationTimestamp`, then alphabetical `namespace/name`.

This is a notable improvement over Ingress, where ordering semantics depend on the controller.

### Filters

Filters modify a request/response or short-circuit it. Set per rule (`rules[].filters`) or per backend (`backendRefs[].filters`).

| Filter type | Purpose | Support level |
|---|---|---|
| `RequestHeaderModifier` | `set`, `add`, `remove` request headers | Core |
| `ResponseHeaderModifier` | same for response | Extended |
| `RequestRedirect` | Return 301/302 to `scheme`/`hostname`/`path`/`port`/`statusCode` | Core |
| `URLRewrite` | Rewrite `hostname` and/or `path` (`ReplaceFullPath`, `ReplacePrefixMatch`) before sending to backend | Extended |
| `RequestMirror` | Copy traffic to a second backend, ignore its response | Extended |
| `CORS` (newer) | Cross-origin policy | Extended / Experimental |
| `ExtensionRef` | Implementation-specific custom filter | Extended |

Examples:

HTTP → HTTPS redirect (the classic; attach this route to the `http` listener):
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: infra
spec:
  parentRefs:
    - name: shared-gw
      sectionName: http
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

Strip prefix (equivalent to nginx `rewrite-target`, but typed):
```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /api/v1
    filters:
      - type: URLRewrite
        urlRewrite:
          path:
            type: ReplacePrefixMatch
            replacePrefixMatch: /
    backendRefs:
      - name: api
        port: 8080
```
`/api/v1/users` reaches the backend as `/users`.

Add a header:
```yaml
filters:
  - type: RequestHeaderModifier
    requestHeaderModifier:
      set:
        - name: x-forwarded-proto
          value: https
      add:
        - name: x-env
          value: prod
      remove:
        - x-debug
```

Mirror to a shadow service (test a new version with real traffic, no user impact):
```yaml
filters:
  - type: RequestMirror
    requestMirror:
      backendRef:
        name: api-v2
        port: 8080
```

### Traffic splitting with weights

`backendRefs[].weight` is the first-class, portable replacement for nginx canary annotations. Share = `weight / sum(weights)`; default weight `1`; weight `0` means no traffic.

```yaml
rules:
  - backendRefs:
      - name: app-v1
        port: 80
        weight: 90
      - name: app-v2
        port: 80
        weight: 10
```

- Canary rollout: 99/1 → 90/10 → 50/50 → 0/100, editing weights with `kubectl apply` or `kubectl patch`.
- Blue/green: 100/0 then flip to 0/100; rollback = flip back.
- Weights are proportions per request (statistical) — with 10 requests you will not see exactly 1 canary hit.
- Pair with `RequestHeaderModifier`/header matches to route testers (`x-canary: 1`) deterministically to v2 before opening a percentage.

### Other Route kinds (concept)

| Kind | Layer | Status | Use |
|---|---|---|---|
| `HTTPRoute` | L7 HTTP/HTTPS | Standard (GA) | Web apps, REST APIs |
| `GRPCRoute` | L7 gRPC | Standard (GA since v1.1) | Match by gRPC `service` and `method`, headers; same backendRefs/filters idea |
| `TLSRoute` | L4 TLS by SNI | Experimental | Passthrough: route encrypted traffic by server name without decrypting |
| `TCPRoute` | L4 TCP | Experimental | Databases, message brokers, any raw TCP (this is what the nginx `stream` module did in lesson 13) |
| `UDPRoute` | L4 UDP | Experimental | DNS, game servers, syslog |

`GRPCRoute` sketch:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: greeter
spec:
  parentRefs:
    - name: shared-gw
      sectionName: grpc
  rules:
    - matches:
        - method:
            service: helloworld.Greeter
            method: SayHello
      backendRefs:
        - name: greeter
          port: 50051
```

`TCPRoute` sketch (Experimental CRDs must be installed; listener `protocol: TCP`):
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: pg
spec:
  parentRefs:
    - name: shared-gw
      sectionName: postgres
  rules:
    - backendRefs:
        - name: postgres
          port: 5432
```
A TCPRoute has no host or path to match on — one listener port maps to its backends. Exposing databases through a public gateway is rarely wise; shown for completeness.

### ReferenceGrant — cross-namespace safety

Rule: **a resource may not reference an object in another namespace unless the target namespace explicitly agrees.** The agreement is a `ReferenceGrant` created **in the target namespace**, by whoever owns it.

Typical case 1 — an HTTPRoute in `app-a` sends traffic to a Service in `shared-services`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-app-a-routes
  namespace: shared-services      # lives with the TARGET
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: app-a
  to:
    - group: ""
      kind: Service
      name: metrics-ui           # optional: omit name = all Services in ns
```

Typical case 2 — a Gateway in `infra` uses a certificate Secret from `certs`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gw-certs
  namespace: certs
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: Gateway
      namespace: infra
  to:
    - group: ""
      kind: Secret
```

Why: without it, any tenant could point a Route at another team's Service (data exfiltration / confused-deputy) or make a Gateway load another team's private key. Note that Route → Gateway attachment across namespaces is **not** a ReferenceGrant; it is controlled by the Gateway's `allowedRoutes`. Missing grant symptom: Route status `ResolvedRefs=False`, reason `RefNotPermitted`.

### Status conditions — the debugging surface

Gateway API puts rich status on every object. Always read it first.

| Object | Condition | Meaning when not True |
|---|---|---|
| GatewayClass | `Accepted` | Controller not installed, wrong `controllerName` |
| Gateway | `Accepted` | Config invalid |
| Gateway | `Programmed` | Data plane not ready / no address yet (LB pending) |
| Gateway listener | `Accepted`, `Programmed`, `ResolvedRefs`, `Conflicted` | Bad TLS Secret ref, port/hostname conflict |
| HTTPRoute (per parent) | `Accepted` | Not allowed by `allowedRoutes`, listener not found, hostname mismatch, wrong kind |
| HTTPRoute (per parent) | `ResolvedRefs` | Backend Service missing, wrong port, `RefNotPermitted` |

`kubectl describe httproute shop -n shop` prints them under `Status: Parents:`. Reasons (`NotAllowedByListeners`, `NoMatchingListenerHostname`, `BackendNotFound`, `RefNotPermitted`, `InvalidKind`) tell you exactly what to fix.

### Migration: ingress2gateway

`ingress2gateway` is a Kubernetes SIG-Network CLI that reads Ingress objects (and provider-specific annotations for nginx, Istio, Kong, GCE ...) and prints equivalent Gateway/HTTPRoute YAML.

```bash
ingress2gateway print --providers ingress-nginx --all-namespaces
ingress2gateway print --providers ingress-nginx -n shop > gateway-shop.yaml
```
Treat output as a **draft**: unmapped annotations are reported as warnings — review them. Migration pattern: run Ingress controller and Gateway side by side on different load balancer IPs, test with `curl --resolve`, switch DNS, then delete the Ingress.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl apply --server-side -f <crds>` | Install Gateway API CRDs (server-side apply avoids annotation size limit) | `kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml` |
| `kubectl get crd \| grep gateway.networking` | Verify CRDs installed | `kubectl get crd \| grep gateway.networking.k8s.io` |
| `kubectl api-resources --api-group=gateway.networking.k8s.io` | List Gateway API kinds, short names | same |
| `kubectl get gatewayclass` | Show classes and `ACCEPTED` column | `kubectl get gatewayclass` |
| `kubectl get gateway -A` | Gateways + ADDRESS + PROGRAMMED | `kubectl get gateway -A` |
| `kubectl get httproute -A` | Routes + hostnames | `kubectl get httproute -A -o wide` |
| `kubectl describe gateway <name> -n <ns>` | Read listener status, attachedRoutes, events | `kubectl describe gateway shared-gw -n infra` |
| `kubectl describe httproute <name> -n <ns>` | Read `Accepted` / `ResolvedRefs` | `kubectl describe httproute shop -n shop` |
| `kubectl get gateway <n> -o jsonpath='{.status.addresses[0].value}'` | Get external address | `kubectl get gateway shared-gw -n infra -o jsonpath='{.status.addresses[0].value}'` |
| `kubectl get referencegrant -A` | List grants (short: `refgrant`) | `kubectl get referencegrant -n shared-services` |
| `kubectl wait --for=condition=Programmed gateway/<n>` | Block until ready (scripts) | `kubectl wait --for=condition=Programmed gateway/shared-gw -n infra --timeout=120s` |
| `kubectl patch httproute <n> --type merge -p ...` | Change canary weights | see walkthrough |
| `helm install <rel> <chart> -n <ns> --create-namespace` | Install a gateway controller chart (Helm itself is lessons 53-55) | `helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.2.0 -n envoy-gateway-system --create-namespace` |
| `curl --resolve host:port:ip` | Test host routing without DNS | `curl --resolve shop.example.com:80:203.0.113.10 http://shop.example.com/` |
| `curl -H 'Host: ...'` | Same, via header | `curl -H 'Host: shop.example.com' http://$IP/` |
| `kubectl port-forward svc/<name> 8080:80` | Reach a gateway proxy Service when no LB is available | `kubectl port-forward -n envoy-gateway-system svc/<proxy-svc> 8080:80` |
| `ingress2gateway print` | Convert Ingress to Gateway API | `ingress2gateway print --providers ingress-nginx -A` |

Short names: `gc` (GatewayClass), `gtw` (Gateway), `httproute`, `grpcroute`, `refgrant`.

## Examples / Walkthrough

Goal: install Envoy Gateway, publish two apps behind one Gateway, redirect HTTP → HTTPS-style routing (HTTP only on a lab cluster), do a canary split, test with curl. Works on kind/minikube/k3s; on a cluster without a LoadBalancer implementation use `kubectl port-forward` or NodePort for the proxy Service (lesson 37).

### Step 1 — Install Gateway API CRDs

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
kubectl get crd | grep gateway.networking.k8s.io
# gatewayclasses, gateways, grpcroutes, httproutes, referencegrants
kubectl get crd gateways.gateway.networking.k8s.io \
  -o jsonpath='{.metadata.annotations.gateway\.networking\.k8s\.io/bundle-version}{"\n"}'
```

Some controller charts install the CRDs themselves; installing twice with different versions causes conflicts — check the chart docs first.

### Step 2 — Install a gateway controller (Envoy Gateway)

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.2.0 -n envoy-gateway-system --create-namespace
kubectl wait --timeout=5m -n envoy-gateway-system \
  deployment/envoy-gateway --for=condition=Available
kubectl get pods -n envoy-gateway-system
```
(Helm is only used as an installer here; chart authoring is lessons 53-55.) Alternatively use the project's plain manifest `install.yaml`.

### Step 3 — GatewayClass and Gateway

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: v1
kind: Namespace
metadata:
  name: infra
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
EOF

kubectl get gatewayclass            # ACCEPTED must be True
kubectl get gateway -n infra        # PROGRAMMED True, ADDRESS filled
kubectl describe gateway shared-gw -n infra | sed -n '/Status:/,$p'
```

If `ADDRESS` stays empty on a bare-metal/kind cluster, the proxy Service is a `LoadBalancer` with no provider (lesson 37). Use port-forward in the next steps (see Step 6) or install MetalLB / use `cloud-provider-kind`.

### Step 4 — Two apps in different namespaces

```bash
kubectl create namespace shop
kubectl create deployment web --image=nginx:1.27 --port=80 -n shop
kubectl create deployment api --image=hashicorp/http-echo:1.0 -n shop -- \
  /http-echo -text="api v1" -listen=:5678
kubectl expose deployment web --port=80 -n shop
kubectl expose deployment api --port=5678 -n shop
kubectl get svc,pods -n shop
```
(`kubectl create deployment ... -- cmd` passes the container command; `http-echo` answers with the fixed text.)

### Step 5 — HTTPRoute with path and host matching

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop
  namespace: shop
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
      sectionName: http
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: api
          port: 5678
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web
          port: 80
EOF

kubectl get httproute -n shop
kubectl describe httproute shop -n shop | sed -n '/Parents:/,$p'
# expect: Accepted True, ResolvedRefs True
```

### Step 6 — Test traffic

```bash
# With a LoadBalancer address:
GW=$(kubectl get gateway shared-gw -n infra -o jsonpath='{.status.addresses[0].value}')
curl -i --resolve shop.example.com:80:$GW http://shop.example.com/
curl -s --resolve shop.example.com:80:$GW http://shop.example.com/api/hello
# -> api v1

# Without LB: port-forward the envoy proxy Service the controller created for this Gateway
SVC=$(kubectl get svc -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=shared-gw -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n envoy-gateway-system svc/$SVC 8888:80 &
curl -i -H 'Host: shop.example.com' http://localhost:8888/
curl -s -H 'Host: shop.example.com' http://localhost:8888/api/hello
curl -i -H 'Host: other.example.com' http://localhost:8888/   # 404 – no route for host
```

### Step 7 — Header match and redirect

```bash
kubectl patch httproute shop -n shop --type json -p '[
 {"op":"add","path":"/spec/rules/0","value":{
   "matches":[{"path":{"type":"PathPrefix","value":"/old"}}],
   "filters":[{"type":"RequestRedirect","requestRedirect":{
       "path":{"type":"ReplaceFullPath","replaceFullPath":"/"},
       "statusCode":302}}]
 }}]'
curl -i -H 'Host: shop.example.com' http://localhost:8888/old   # 302 Location: http://shop.example.com/
```

### Step 8 — Canary split 90/10

```bash
kubectl create deployment api-v2 --image=hashicorp/http-echo:1.0 -n shop -- \
  /http-echo -text="api v2" -listen=:5678
kubectl expose deployment api-v2 --port=5678 -n shop

cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-canary
  namespace: shop
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
      sectionName: http
  hostnames:
    - canary.example.com
  rules:
    - backendRefs:
        - name: api
          port: 5678
          weight: 90
        - name: api-v2
          port: 5678
          weight: 10
EOF

for i in $(seq 1 100); do
  curl -s -H 'Host: canary.example.com' http://localhost:8888/
done | sort | uniq -c
# ~90 "api v1", ~10 "api v2"

# promote: flip to 0/100
kubectl patch httproute api-canary -n shop --type json -p '[
 {"op":"replace","path":"/spec/rules/0/backendRefs/0/weight","value":0},
 {"op":"replace","path":"/spec/rules/0/backendRefs/1/weight","value":100}]'
```

### Step 9 — Cross-namespace backend with ReferenceGrant

```bash
kubectl create namespace tools
kubectl create deployment whoami --image=traefik/whoami -n tools
kubectl expose deployment whoami --port=80 -n tools

cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: tools
  namespace: shop
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
  hostnames: ["tools.example.com"]
  rules:
    - backendRefs:
        - name: whoami
          namespace: tools
          port: 80
EOF
kubectl describe httproute tools -n shop | grep -A4 ResolvedRefs
# ResolvedRefs False, Reason RefNotPermitted

cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-shop-routes
  namespace: tools
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: shop
  to:
    - group: ""
      kind: Service
EOF
kubectl describe httproute tools -n shop | grep -A4 ResolvedRefs   # now True
curl -s -H 'Host: tools.example.com' http://localhost:8888/ | head -3
```

### Step 10 — Restrict who can attach (allowedRoutes)

```bash
kubectl label namespace shop shared-gateway-access=true
kubectl patch gateway shared-gw -n infra --type json -p '[
 {"op":"replace","path":"/spec/listeners/0/allowedRoutes",
  "value":{"namespaces":{"from":"Selector","selector":{"matchLabels":{"shared-gateway-access":"true"}}}}}]'
# Routes from namespaces without the label now show Accepted=False, reason NotAllowedByListeners
kubectl label namespace tools foo=bar --overwrite
```

### Step 11 — HTTPS listener (lab outline)

On a cluster with cert-manager (lesson 38), annotate the Gateway and add a listener:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: shop.example.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: shop-example-com-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              shared-gateway-access: "true"
```
cert-manager issues the Secret `shop-example-com-tls`; `kubectl get certificate -n infra` shows readiness. The HTTP listener keeps only the redirect route (see Concepts).

### Cleanup

```bash
kubectl delete namespace shop tools infra
kubectl delete gatewayclass eg
helm uninstall eg -n envoy-gateway-system
kubectl delete namespace envoy-gateway-system
```
Delete CRDs only if nothing else uses Gateway API: deleting the CRDs deletes **all** Gateways and Routes cluster-wide.

## Common Pitfalls

- **Gateway API CRDs not installed** — `error: resource mapping not found for kind "Gateway"`. CRDs are not part of core Kubernetes; install the standard bundle first. Check `kubectl get crd | grep gateway`.
- **Controller installed but no GatewayClass / not Accepted** — Gateway stays with no status. `kubectl get gatewayclass`; `controllerName` must match the controller exactly.
- **Gateway `ADDRESS` empty** — The proxy Service is `type: LoadBalancer` and the cluster has no LB provider (kind, bare metal, minikube without `tunnel`). Use MetalLB, cloud-provider-kind, `minikube tunnel`, or port-forward the proxy Service.
- **Route attached but ignored** — Check `status.parents[].conditions`. Usual reasons: `NotAllowedByListeners` (Gateway `allowedRoutes` default is `Same` namespace only), `NoMatchingListenerHostname` (route hostname does not intersect listener hostname), wrong `sectionName`.
- **Wrong parentRef namespace** — `parentRefs[].namespace` defaults to the Route's namespace; if the Gateway lives in `infra` you must set it.
- **`RefNotPermitted`** — backend Service or TLS Secret in another namespace without a `ReferenceGrant` created in the *target* namespace. Grants in the wrong namespace do nothing.
- **`BackendNotFound` / `port` omitted** — for Service backends `port` is mandatory and must be the Service port, not the container port.
- **Prefix rewrite surprises** — `ReplacePrefixMatch` requires a `PathPrefix` match on that rule; replacing `/api/v1` with `/` on `/api/v1users` behaves per element boundaries. Test with curl.
- **Expecting rule order to decide precedence** — it is decided by the spec's precedence list (exact > longest prefix > method > headers > query), then oldest Route. Two Routes matching the same host/path conflict silently by age — avoid duplicates.
- **Using Experimental kinds with standard CRDs** — `TCPRoute`/`UDPRoute`/`TLSRoute` need the experimental bundle (or a newer release where they graduated) *and* controller support. `no matches for kind "TCPRoute"` = missing CRD.
- **Mixing two controllers on one GatewayClass name** — each class has one controller. Use separate classes.
- **Mixing Ingress and Gateway for the same hostname** — two data planes fighting over DNS/TLS. Migrate host by host.
- **Weights not adding to 100** — they do not need to; shares are relative. `weight: 0` on all backends = 503.
- **Deleting CRDs to "clean up"** — removes every Gateway and Route in the cluster instantly.
- **Assuming nginx annotations work** — `nginx.ingress.kubernetes.io/*` annotations are ignored by Gateway API. Use filters/policies of the new controller.
- **Testing with real DNS before it exists** — use `curl --resolve` or `-H 'Host:'`.

## FAQ

**Q: Is Gateway API replacing Ingress? Is Ingress deprecated?**
A: Ingress is not deprecated or removed, but it is frozen — no new features. New development (traffic splitting, header matching, gRPC, TCP/UDP, role separation) is only in Gateway API. New projects should prefer Gateway API; existing Ingress setups keep working.

**Q: Is Gateway API part of Kubernetes core?**
A: It is an official Kubernetes SIG-Network project, but delivered as CRDs you install separately. Core Kubernetes does not contain the kinds; a conformant controller does the work.

**Q: What is the difference between GatewayClass, Gateway and HTTPRoute?**
A: GatewayClass = which controller/implementation (cluster-wide, like IngressClass/StorageClass). Gateway = a concrete entry point with ports, protocols, hostnames and TLS (owned by platform team). HTTPRoute = the routing rules of one application attaching to a Gateway (owned by developers).

**Q: Why three objects instead of one Ingress?**
A: Separation of duties. Platform team controls ports, certs and which namespaces may use the gateway; developers control routing for their app only. With RBAC you can grant `httproutes` edit without `gateways` edit.

**Q: How does a Route find its Gateway?**
A: Through `spec.parentRefs` (name, namespace, optional `sectionName` = listener name). The Gateway must also allow it via `listeners[].allowedRoutes`. Both must agree, otherwise `Accepted=False`.

**Q: What does a Gateway create in the cluster?**
A: Implementation-dependent. Envoy Gateway and NGINX Gateway Fabric typically create a proxy Deployment and a Service (usually `LoadBalancer`) for the Gateway; the external address appears in `status.addresses`. Cloud implementations may program an external cloud load balancer instead.

**Q: How do I do a canary release?**
A: Put two `backendRefs` in one rule with weights, e.g. 90 and 10, and adjust with `kubectl patch`/`apply`. For deterministic testing add a rule matching a header that points 100% to the new version.

**Q: How do I rewrite or strip a path prefix?**
A: Use the `URLRewrite` filter with `path.type: ReplacePrefixMatch` and `replacePrefixMatch: /`. For a redirect (client sees new URL) use `RequestRedirect`.

**Q: How do I redirect HTTP to HTTPS?**
A: Create an HTTPRoute attached to the HTTP listener (`sectionName: http`) with a single rule containing a `RequestRedirect` filter (`scheme: https`, `statusCode: 301`) and no backendRefs.

**Q: How do I terminate TLS?**
A: HTTPS listener on the Gateway with `tls.mode: Terminate` and `certificateRefs` pointing at a `kubernetes.io/tls` Secret. cert-manager can issue the Secret automatically if you annotate the Gateway with an issuer.

**Q: What is TLS passthrough?**
A: Listener with `protocol: TLS` and `tls.mode: Passthrough`; the gateway does not decrypt, it routes by SNI using `TLSRoute` to a backend that terminates TLS itself.

**Q: Can Gateway API expose a database or non-HTTP service?**
A: Yes in principle: `TCPRoute`/`UDPRoute`/`TLSRoute` (Experimental channel) attach to TCP/UDP/TLS listeners. Support depends on the controller. Avoid publicly exposing databases anyway.

**Q: What is a ReferenceGrant?**
A: A resource in the *target* namespace that says "objects of kind X from namespace Y may reference my objects of kind Z". Needed for HTTPRoute → Service in another namespace and Gateway → Secret in another namespace. Prevents tenants from reaching into each other's namespaces.

**Q: Does ReferenceGrant also control Route → Gateway attachment?**
A: No. Cross-namespace attachment to a Gateway is governed by the Gateway's `allowedRoutes`. ReferenceGrant covers references *from* a Route/Gateway *to* Services/Secrets.

**Q: Which controller should I use?**
A: Whatever fits the platform: Envoy Gateway or NGINX Gateway Fabric for a standalone ingress; Istio if you run a mesh; Cilium if it is already your CNI; the cloud's native class (GKE Gateway, AWS LBC) for managed load balancers. Compare conformance reports for the features you need.

**Q: Can I run Gateway API and nginx-ingress at the same time?**
A: Yes, they are independent. Put them on different load balancers/IPs and migrate hostnames one by one, switching DNS.

**Q: How do I migrate existing Ingress objects?**
A: `ingress2gateway print --providers ingress-nginx -A` generates Gateway and HTTPRoute YAML. Review warnings for annotations it could not translate, test with `curl --resolve`, then switch DNS.

**Q: How do I debug a route that returns 404 or 503?**
A: 1) `kubectl get gatewayclass` Accepted? 2) `kubectl describe gateway` Programmed, listener attachedRoutes > 0? 3) `kubectl describe httproute` Accepted/ResolvedRefs and their reasons. 4) `kubectl get endpoints <svc>` has addresses? 5) test with the right Host header. 404 = no matching rule/host; 503 = matched but no healthy/resolved backend.

**Q: Is the `weight` a percentage?**
A: No, a relative share: `weight / sum of weights`. 90/10 and 9/1 behave identically. Weight 0 sends nothing.

**Q: Do wildcard hostnames work?**
A: Yes, `*.example.com` on listeners and Routes. More specific hostnames win over wildcards.

**Q: How does Gateway API relate to the service mesh?**
A: The GAMMA initiative lets HTTPRoutes use a Service as `parentRef` to configure east-west (in-cluster) routing in mesh implementations. For this lesson only north-south (ingress) traffic matters.

**Q: Where is the nginx annotation for X (rate limit, timeouts, auth)?**
A: Not in the core spec. Many are being standardised (timeouts, CORS, session persistence); until then use the controller's policy CRDs (e.g. Envoy Gateway `BackendTrafficPolicy`/`SecurityPolicy`, NGINX Gateway Fabric policies) attached with `targetRefs`.

## Practice / Exercise

### Core

1. Install the Gateway API standard CRDs and a controller of your choice (Envoy Gateway recommended). Verify `kubectl get gatewayclass` shows `ACCEPTED True`.
2. Create namespace `infra` with a `Gateway` `lab-gw` (HTTP :80, routes allowed from all namespaces). Confirm `PROGRAMMED True` and find the address (or set up port-forward).
3. Deploy two Deployments + Services in namespace `demo`: `blog` (nginx) and `api` (http-echo, text "api"). Create one HTTPRoute for host `demo.local` that sends `/` to `blog` and `/api` to `api` with a prefix-stripping `URLRewrite`. Prove with curl `--resolve`/`Host:` header that both paths work and that `/api/x` reaches the backend as `/x`.
4. Add a rule: requests with header `x-env: staging` go to a third Service `blog-staging`. Show that the header rule wins over the plain `/` rule (precedence).
5. Add an HTTPRoute on the same Gateway that redirects `/old` → `/new` with status 301; verify with `curl -i`.
6. Break things on purpose and read status: (a) wrong `sectionName`; (b) backend with wrong port; (c) host that does not match listener hostname. For each, record the condition type, status and reason from `kubectl describe httproute`.
7. Canary: deploy `api-v2` and create a route with weights 80/20; send 200 requests and count responses. Then promote to 0/100 using `kubectl patch`.

### Stretch

1. Create namespace `team-b` with a Service. Route to it from an HTTPRoute in `demo` (cross-namespace backendRef). Observe `RefNotPermitted`, then fix with a `ReferenceGrant` — in which namespace must it live and why?
2. Restrict the Gateway listener with `allowedRoutes.namespaces.from: Selector` so only namespaces labeled `gateway-access=true` can attach. Prove a route from an unlabeled namespace is `Accepted=False` with reason `NotAllowedByListeners`.
3. Add an HTTPS listener with a cert-manager issued certificate (staging issuer), keep a `RequestRedirect` route on the HTTP listener, test with `curl -k`/`--resolve`.
4. Use `RequestMirror` to shadow `/api` traffic to `api-v2` and verify (via logs, `kubectl logs deploy/api-v2`) that v2 sees requests while clients only get v1 responses.
5. Install the Experimental CRDs, add a `protocol: TCP` listener and a `TCPRoute` to a test echo service (check your controller supports it). Document what changed in the Gateway status.
6. Install `ingress2gateway`, write a 3-host Ingress using nginx annotations (rewrite-target, ssl-redirect), convert it, and list which annotations were not translated.
7. Write a one-page comparison (table) of the same app published with Ingress (lesson 38) vs Gateway API: which fields/annotations map to which Gateway API features.

## Further Reading

- Gateway API docs: https://gateway-api.sigs.k8s.io/
- API reference: https://gateway-api.sigs.k8s.io/reference/spec/
- Concepts (roles, resource model): https://gateway-api.sigs.k8s.io/concepts/api-overview/
- HTTP routing guides (matching, redirects, rewrites, traffic splitting): https://gateway-api.sigs.k8s.io/guides/
- Implementations and conformance: https://gateway-api.sigs.k8s.io/implementations/
- Release channels and versions: https://gateway-api.sigs.k8s.io/concepts/versioning/
- Kubernetes docs, Gateway API: https://kubernetes.io/docs/concepts/services-networking/gateway/
- Envoy Gateway: https://gateway.envoyproxy.io/
- NGINX Gateway Fabric: https://docs.nginx.com/nginx-gateway-fabric/
- ingress2gateway: https://github.com/kubernetes-sigs/ingress2gateway
- cert-manager Gateway API support: https://cert-manager.io/docs/usage/gateway/

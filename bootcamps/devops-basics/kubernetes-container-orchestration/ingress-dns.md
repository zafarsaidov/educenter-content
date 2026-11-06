# Lesson 38: Ingress & DNS

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-37 (cluster architecture, kubectl, Pods/Deployments, Services ClusterIP/NodePort/LoadBalancer). Basic HTTP/TLS knowledge from Lesson 13 (nginx, certbot, Let's Encrypt).

## Learning Objectives

By end of lesson student can:
- Explain why Ingress exists and what problem it solves compared with one NodePort/LoadBalancer Service per app.
- Distinguish the Ingress *resource* (rules) from the Ingress *controller* (the program that implements them) and from the `IngressClass`.
- Install ingress-nginx in a cluster and find its external entry point (LoadBalancer IP / NodePort).
- Write Ingress manifests with host-based and path-based rules, `pathType` (`Prefix`, `Exact`, `ImplementationSpecific`) and `defaultBackend`.
- Terminate TLS in Ingress using a TLS Secret, and automate certificates with cert-manager (`ClusterIssuer`, `Certificate`, ACME HTTP-01, staging vs production).
- Use common `nginx.ingress.kubernetes.io/*` annotations (`rewrite-target`, `ssl-redirect`, rate limiting, body size, timeouts).
- Explain how CoreDNS resolves names inside the cluster, read the Corefile, use `dnsPolicy`/`dnsConfig`, and add stub domains.
- Troubleshoot Ingress (404, 502/503, 308 loops, certificate not issued, DNS failures) with a systematic checklist.

## Topics

- Ingress controller: nginx-ingress, installation via Helm, IngressClass resource
- Ingress resource: rules (host-based, path-based), pathType (Prefix/Exact), defaultBackend
- TLS termination: tls block in Ingress, cert-manager (ClusterIssuer, Certificate, ACME)
- Annotations: nginx.ingress.kubernetes.io/* (rewrite-target, ssl-redirect, rate-limit)
- CoreDNS: in-cluster DNS resolution, dnsPolicy options, custom stub domains

## Concepts

### The problem: one Service per app does not scale

From Lesson 37 we have three ways to expose an app outside the cluster:

| Service type | Result | Problem at scale |
|---|---|---|
| NodePort | Port 30000-32767 on every node | Ugly ports, firewall holes per app, no hostnames, no TLS |
| LoadBalancer | One cloud load balancer + public IP **per Service** | 20 apps = 20 paid load balancers and 20 IPs |
| ClusterIP | Internal only | Not reachable from outside |

What we really want is what we built by hand in Lesson 13: **one** public entry point (nginx) that looks at the `Host` header and URL path and forwards to the right backend, terminating TLS on the way. Ingress is exactly this idea, made declarative and Kubernetes-native.

```
                        Internet
                           |
                  [ LoadBalancer / NodePort ]      <- ONE entry point
                           |
                [ ingress-nginx controller Pods ]   <- reverse proxy (nginx inside)
                  /        |          \
   app1.example.com   app2.example.com   example.com/api
          |                |                   |
     Service app1     Service app2       Service api   (ClusterIP)
          |                |                   |
        Pods             Pods                Pods
```

### Ingress resource vs Ingress controller vs IngressClass

Three separate things — students mix them up constantly:

1. **Ingress resource** (`kind: Ingress`, API group `networking.k8s.io/v1`) — a YAML object describing *routing rules*: "host X + path Y goes to Service Z port P". On its own it does **nothing**. It is just data in etcd.
2. **Ingress controller** — a running program (Pods) that watches Ingress objects through the API server and configures a real proxy accordingly. Examples: **ingress-nginx** (community, most common — used in this course), Traefik, HAProxy Ingress, Istio, Contour, cloud-native ones (AWS ALB controller, GKE). Kubernetes does **not** ship a controller; you must install one.
3. **IngressClass** (`kind: IngressClass`) — an object that names a controller. An Ingress selects its controller via `spec.ingressClassName: nginx`. Lets several controllers coexist in one cluster (e.g. `nginx` for public, `internal` for private traffic).

Rule of thumb: **no controller installed → Ingress objects are silently ignored**. `kubectl get ingress` shows the object but the ADDRESS column stays empty forever.

> Note: the older annotation `kubernetes.io/ingress.class: nginx` is deprecated. Use `spec.ingressClassName`.

> Note: Ingress API is feature-frozen. Newer, more expressive API is Gateway API — see lesson 39. Ingress remains extremely common in production and in exams.

### How ingress-nginx works internally

- Controller Pods run nginx plus a Go controller process.
- The Go process watches Ingress, Service, EndpointSlice and Secret objects.
- On changes it renders an `nginx.conf` (server blocks per host, location blocks per path) and reloads nginx. Endpoint changes (Pod scale up/down) are applied dynamically through Lua without reload.
- Traffic goes **directly to Pod IPs** (from EndpointSlices), not through the Service ClusterIP. The Service is only used to discover Pods.
- The controller is itself exposed via a Service of type LoadBalancer (cloud) or NodePort (bare metal / local). That Service is the single front door.

You can look at the generated config — handy for debugging:

```bash
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- cat /etc/nginx/nginx.conf | head -100
```

### Installing ingress-nginx with Helm

The plan allows Helm here. **Helm is the Kubernetes package manager** — it installs a bundle of manifests ("chart") in one command. It is fully taught in lessons 53-55; today we only copy the install commands.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx   # register chart repository
helm repo update                                                          # refresh chart index
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace                            # install release "ingress-nginx"
kubectl -n ingress-nginx get pods,svc
```

Check the Service:

```
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
service/ingress-nginx-controller     LoadBalancer   10.96.12.34    203.0.113.10     80:31080/TCP,443:31443/TCP
```

- **EXTERNAL-IP** is where DNS records must point (A record, or wildcard `*.example.com`).
- On a cluster without a cloud load balancer (kubeadm, rke2, minikube, kind) EXTERNAL-IP stays `<pending>`. Options: use the NodePorts (`31080`/`31443` above), install a bare-metal load balancer such as MetalLB, run the controller with `hostNetwork`, or on minikube `minikube addons enable ingress` / `minikube tunnel`. For labs, `curl -H "Host: app.example.com" http://<node-ip>:<nodeport>/` is enough.
- Verify IngressClass was created:

```bash
kubectl get ingressclass
# NAME    CONTROLLER             PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx   <none>       2m
```

An IngressClass may carry the annotation `ingressclass.kubernetes.io/is-default-class: "true"`. Ingress objects without `ingressClassName` then use it. With no default class and no `ingressClassName`, the Ingress is ignored.

### The Ingress resource anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
  namespace: demo
spec:
  ingressClassName: nginx        # which controller handles this
  rules:
  - host: app.example.com        # optional; omitted = match any host
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web            # Service in the SAME namespace as the Ingress
            port:
              number: 80         # or `name: http` (named port)
```

Important details:

- Ingress is **namespaced**. The backend Service must live in the same namespace. (Cross-namespace routing needs Gateway API — lesson 39 — or an ExternalName Service workaround.)
- `backend.service.port` takes `number` or `name`, never both.
- `host` supports a wildcard only as the leftmost label: `*.example.com` matches `a.example.com`, not `a.b.example.com`, not `example.com`.
- Without `host`, the rule matches all hostnames (including raw IP requests).

### Host-based routing (virtual hosting)

One IP, many sites — same idea as nginx `server_name` in Lesson 13:

```yaml
spec:
  ingressClassName: nginx
  rules:
  - host: blog.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: blog, port: {number: 80}}}
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: shop, port: {number: 80}}}
```

### Path-based routing (fan-out)

One hostname, different URL prefixes to different Services:

```yaml
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend: {service: {name: api, port: {number: 8080}}}
      - path: /
        pathType: Prefix
        backend: {service: {name: frontend, port: {number: 80}}}
```

Longest matching path wins regardless of order in YAML, so `/api` beats `/`.

### pathType — the part everyone gets wrong

`pathType` is **required** in `networking.k8s.io/v1`.

| pathType | Behaviour | Example `path: /foo` matches |
|---|---|---|
| `Exact` | Case-sensitive exact match of the whole path | `/foo` only. Not `/foo/`, not `/foo/bar` |
| `Prefix` | Match by **path elements** split on `/`, not by raw string prefix | `/foo`, `/foo/`, `/foo/bar`. **Not** `/foobar` |
| `ImplementationSpecific` | Controller decides (ingress-nginx treats it as regex-capable, depending on annotations) | Controller dependent — avoid unless you need regex |

Extra Prefix rules:
- Trailing slash is ignored: `/foo/` and `/foo` are equivalent Prefix paths.
- `path: /` with `Prefix` matches everything.
- If several paths match, the **longest** wins; with equal length, `Exact` beats `Prefix`.

### defaultBackend

Traffic that matches no rule goes to the controller's built-in default backend (returns `404 default backend - 404`). You can override it per Ingress:

```yaml
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: custom-404
      port:
        number: 80
```

- With **no `rules` at all**, `defaultBackend` receives all traffic for that Ingress (a "catch-all" Ingress).
- With rules present, `defaultBackend` serves unmatched hosts/paths.
- That 404 page you see for an unknown host is nginx's, not your app's — recognise it when debugging.

### TLS termination

**TLS termination** = the proxy decrypts HTTPS, then talks plain HTTP to the Pods. Certificates live at the edge only.

Manual way — create a TLS Secret from an existing cert and key (Secrets are taught in lesson 42; for now one command):

```bash
kubectl -n demo create secret tls app-tls --cert=tls.crt --key=tls.key
```

Reference it in the Ingress:

```yaml
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls          # Secret type kubernetes.io/tls, SAME namespace
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: web, port: {number: 80}}}
```

- The Secret must be in the same namespace as the Ingress and contain keys `tls.crt` and `tls.key`.
- The `hosts` in the `tls` block must match the certificate's SAN names (and the rule hosts) — otherwise nginx serves its own fake default certificate ("Kubernetes Ingress Controller Fake Certificate") and browsers warn.
- ingress-nginx redirects HTTP → HTTPS automatically once a `tls` block is present (HTTP 308). Turn off with `nginx.ingress.kubernetes.io/ssl-redirect: "false"`.

Test without DNS:

```bash
curl -k --resolve app.example.com:443:203.0.113.10 https://app.example.com/
```

### cert-manager: automatic certificates

Doing `certbot` by hand (Lesson 13) does not fit a cluster where apps appear and disappear. **cert-manager** is a controller that obtains and renews certificates automatically and stores them as Secrets.

It adds new API kinds (Custom Resource Definitions, "CRDs"):

| Kind | Scope | Meaning |
|---|---|---|
| `Issuer` | Namespaced | A certificate authority config usable only in its own namespace |
| `ClusterIssuer` | Cluster-wide | Same, usable from any namespace (preferred) |
| `Certificate` | Namespaced | "I want a cert for these DNS names, store it in Secret X" |
| `CertificateRequest` | Namespaced | Internal: one signing request (auto-created) |
| `Order` / `Challenge` | Namespaced | Internal: ACME protocol state (auto-created) — look here when issuance fails |

Install (again via Helm, details in lessons 53-55):

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true             # older chart versions: --set installCRDs=true
kubectl -n cert-manager get pods      # cert-manager, cainjector, webhook should be Running
```

#### ACME and HTTP-01 in one paragraph

**ACME** is the protocol Let's Encrypt speaks. To prove you own `app.example.com`, the CA gives a token; cert-manager must serve it at `http://app.example.com/.well-known/acme-challenge/<token>` on port 80. To do that cert-manager creates a temporary Pod + Service + Ingress routed through ingress-nginx. Requirements:

- DNS for the host already points to the ingress controller's public IP.
- Port 80 reachable from the internet.
- Cluster must be publicly reachable (otherwise use DNS-01 challenge — outside this lesson).
- Wildcard certs (`*.example.com`) need DNS-01, not HTTP-01.

#### ClusterIssuers: staging first, then production

Let's Encrypt **production** has strict rate limits (about 5 duplicate certs per week per exact set of names; 50 certs per registered domain per week). A typo loop will lock you out for days. Always test with **staging** (untrusted test CA, very high limits), then switch.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@example.com                 # expiry notices go here
    privateKeySecretRef:
      name: letsencrypt-staging-account-key  # ACME account key, cert-manager creates it
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

```bash
kubectl apply -f clusterissuers.yaml
kubectl get clusterissuer
# NAME                  READY   AGE
# letsencrypt-staging   True    10s      <- READY=True means ACME account registered
# letsencrypt-prod      True    10s
```

#### Way 1 — annotate the Ingress (most common)

cert-manager's "ingress-shim" watches Ingresses; if it sees the issuer annotation plus a `tls` block, it creates a `Certificate` for you:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging   # <- magic line
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-example-com-tls     # cert-manager CREATES this Secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: web, port: {number: 80}}}
```

#### Way 2 — explicit Certificate object

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-example-com
  namespace: demo
spec:
  secretName: app-example-com-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
  - app.example.com
  - www.example.com
  duration: 2160h        # 90 days (default for ACME)
  renewBefore: 720h      # renew 30 days before expiry
```

Then the Ingress only needs `tls.secretName: app-example-com-tls` (no annotation).

#### Watching issuance

```bash
kubectl -n demo get certificate               # READY False -> True when done
kubectl -n demo describe certificate app-example-com-tls
kubectl -n demo get certificaterequest,order,challenge
kubectl -n demo describe challenge <name>     # shows the exact failure reason
kubectl -n demo get secret app-example-com-tls
```

Flow: Ingress/Certificate → CertificateRequest → Order → Challenge → temp solver Ingress → Let's Encrypt validates → Secret populated → ingress-nginx reloads with the cert. Renewal repeats automatically before `renewBefore`.

#### Switching staging → production

1. Change annotation to `letsencrypt-prod`.
2. Delete the old Secret so a new certificate is requested: `kubectl -n demo delete secret app-example-com-tls` (cert-manager re-creates it) — or `cmctl renew app-example-com-tls`.
3. Browser now trusts it; staging certs show "(STAGING) Artificial Apricot" issuer and are untrusted.

### nginx.ingress.kubernetes.io annotations

Annotations tune the generated nginx config **per Ingress**. Key/values are always strings — quote numbers and booleans (`"true"`, `"10"`).

| Annotation | Effect |
|---|---|
| `nginx.ingress.kubernetes.io/rewrite-target: /$2` | Rewrite URL path before sending to backend (with regex capture groups) |
| `nginx.ingress.kubernetes.io/use-regex: "true"` | Treat paths as regex |
| `nginx.ingress.kubernetes.io/ssl-redirect: "false"` | Disable automatic HTTP→HTTPS redirect |
| `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"` | Redirect even if no TLS on this Ingress (e.g. TLS ends at external LB) |
| `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` | Backend speaks HTTPS (also `GRPC`, `GRPCS`, `FCGI`) |
| `nginx.ingress.kubernetes.io/proxy-body-size: "50m"` | Max request body (default `1m` → 413 errors on uploads) |
| `nginx.ingress.kubernetes.io/proxy-read-timeout: "120"` | Seconds nginx waits for backend response (default 60) |
| `nginx.ingress.kubernetes.io/proxy-send-timeout: "120"` | Timeout for sending request to backend |
| `nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"` | Backend TCP connect timeout |
| `nginx.ingress.kubernetes.io/limit-rps: "10"` | Max requests per second per client IP |
| `nginx.ingress.kubernetes.io/limit-rpm: "300"` | Max requests per minute per client IP |
| `nginx.ingress.kubernetes.io/limit-connections: "5"` | Max concurrent connections per client IP |
| `nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"` | Burst = limit × multiplier (default 5) |
| `nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,203.0.113.0/24"` | Allow only these client CIDRs (others get 403). Newer name: `allowlist-source-range` |
| `nginx.ingress.kubernetes.io/enable-cors: "true"` | Add CORS headers (+ `cors-allow-origin` etc.) |
| `nginx.ingress.kubernetes.io/auth-type: basic` + `auth-secret` + `auth-realm` | HTTP basic auth |
| `nginx.ingress.kubernetes.io/affinity: "cookie"` | Sticky sessions via cookie |
| `nginx.ingress.kubernetes.io/canary: "true"` + `canary-weight: "20"` | Send 20% of traffic to this (canary) Ingress |
| `nginx.ingress.kubernetes.io/app-root: /app` | Redirect `/` to `/app` |
| `nginx.ingress.kubernetes.io/permanent-redirect: https://new.example.com` | 301 redirect |
| `nginx.ingress.kubernetes.io/configuration-snippet` | Raw nginx config (often disabled for security; avoid) |

#### rewrite-target

Problem: users call `example.com/api/users`, but the backend serves `/users`, not `/api/users`. Backend answers 404. Solution — capture group + rewrite:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend: {service: {name: api, port: {number: 8080}}}
```

`/api/users` → capture group 2 = `users` → backend receives `/users`. `/api` → backend receives `/`.

Pitfall: a plain `rewrite-target: /` with `path: /api` + Prefix rewrites **every** matching request to `/` — the backend always sees `/`. Only use the bare form when you really want that.

#### Rate limiting example

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5"
    nginx.ingress.kubernetes.io/limit-connections: "10"
```

Test: `for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code}\n" https://app.example.com/; done` — beyond limit you get `503` (default limit-req-status-code) in ingress-nginx. Remember that behind another proxy/cloud LB every client appears to have the LB's IP unless real client IP is forwarded (`use-forwarded-headers`, `externalTrafficPolicy: Local`) — then the limit applies to everyone at once.

#### Redirect behaviour of ssl-redirect

- TLS block present → `http://` gets `308 Permanent Redirect` to `https://`.
- Set `ssl-redirect: "false"` for health-check endpoints or when ACME HTTP-01 must stay on HTTP (cert-manager's solver Ingress already handles this).
- Redirect loop (`ERR_TOO_MANY_REDIRECTS`) usually means an external load balancer terminates TLS and forwards plain HTTP to ingress-nginx, which redirects again. Fix: configure the controller to trust `X-Forwarded-Proto` or use `ssl-redirect: "false"`.

### CoreDNS: DNS inside the cluster

Every Pod must resolve Service names. That is done by **CoreDNS**, a DNS server running as a Deployment in `kube-system`, exposed through the Service `kube-dns` (name kept for compatibility).

```bash
kubectl -n kube-system get deploy coredns
kubectl -n kube-system get svc kube-dns        # CLUSTER-IP e.g. 10.96.0.10
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

How a Pod uses it: the kubelet writes `/etc/resolv.conf` inside each container:

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
# search demo.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10
# options ndots:5
```

- `nameserver` = ClusterIP of kube-dns.
- `search` = suffixes tried in order → inside namespace `demo`, `web` becomes `web.demo.svc.cluster.local`.
- `ndots:5` = a name with fewer than 5 dots is first tried with each search suffix. So `api.example.com` (2 dots) triggers up to 4 failed lookups before the real one. A known latency/cost source; mitigate with a trailing dot (`api.example.com.`) or `dnsConfig.options ndots: 2`.

#### Record types CoreDNS serves

| Name | Resolves to |
|---|---|
| `<svc>.<ns>.svc.cluster.local` | ClusterIP of the Service (A/AAAA) |
| `<svc>.<ns>.svc.cluster.local` (headless) | Set of Pod IPs |
| `<pod-ip-dashed>.<ns>.pod.cluster.local` | Pod IP (e.g. `10-244-1-5.demo.pod.cluster.local`) |
| `<pod>.<svc>.<ns>.svc.cluster.local` | Individual Pod behind a headless Service with stable hostname (StatefulSets) |
| `_http._tcp.<svc>.<ns>.svc.cluster.local` | SRV record with port |
| ExternalName Service | CNAME to external name |

Short names: from same namespace `web`; cross-namespace `web.demo`; fully-qualified `web.demo.svc.cluster.local`. `cluster.local` is the default cluster domain (set at cluster creation, kubeadm `--service-dns-domain`).

#### The Corefile

CoreDNS is configured by a file called the Corefile, stored in a ConfigMap named `coredns` in `kube-system` (ConfigMaps are taught in lesson 42 — today we only *read and edit* this one).

```bash
kubectl -n kube-system get configmap coredns -o yaml
```

Typical content:

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

Reading it plugin by plugin:

| Plugin | Role |
|---|---|
| `errors` | Log errors to stdout |
| `health` / `ready` | HTTP endpoints for liveness (`:8080/health`) and readiness (`:8181/ready`) |
| `kubernetes cluster.local ...` | Answers Service/Pod names from the Kubernetes API |
| `prometheus :9153` | Metrics endpoint |
| `forward . /etc/resolv.conf` | Anything not `cluster.local` is forwarded to the node's upstream DNS |
| `cache 30` | Cache answers for up to 30 s |
| `loop` | Detect forwarding loops and stop CoreDNS (cause of CrashLoopBackOff when node resolv.conf points to 127.0.0.53) |
| `reload` | Pick up Corefile changes automatically (~2 min) |
| `loadbalance` | Round-robin the order of A records |

#### Custom stub domains and upstream resolvers

Scenario: company runs internal DNS at `10.20.0.53` for `corp.example.com`. Pods must resolve those names. Add an extra **server block** to the Corefile:

```bash
kubectl -n kube-system edit configmap coredns
```

```
.:53 {
    ...existing content...
}
corp.example.com:53 {
    errors
    cache 30
    forward . 10.20.0.53
}
```

Wait for `reload` (or `kubectl -n kube-system rollout restart deploy coredns`), then test:

```bash
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup db.corp.example.com
```

Other common edits:
- Replace `forward . /etc/resolv.conf` with `forward . 8.8.8.8 1.1.1.1` to use fixed upstreams.
- Static record: add the `hosts` plugin block:

```
hosts {
    10.20.0.99 legacy.corp.example.com
    fallthrough
}
```

- Debug logging: add `log` plugin, then `kubectl -n kube-system logs -l k8s-app=kube-dns -f`.

#### dnsPolicy

Set per Pod in `spec.dnsPolicy`:

| Value | Meaning |
|---|---|
| `ClusterFirst` (default) | Use CoreDNS for cluster names, forward the rest upstream |
| `Default` | Inherit node's `/etc/resolv.conf` (confusing name: it is **not** the default policy) — no cluster DNS |
| `ClusterFirstWithHostNet` | Needed when the Pod uses `hostNetwork: true` and still wants cluster DNS |
| `None` | Ignore everything; you must supply `dnsConfig` |

```yaml
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers: ["1.1.1.1", "10.96.0.10"]
    searches: ["demo.svc.cluster.local", "corp.example.com"]
    options:
    - name: ndots
      value: "2"
```

`dnsConfig` can also be used with `ClusterFirst` to **add** extra options (e.g. lower `ndots`) without replacing the policy.

#### Ingress vs in-cluster DNS — do not confuse them

- **Public DNS** (Route 53, Cloudflare, registrar): `app.example.com → ingress external IP`. Outside users.
- **CoreDNS**: `web.demo.svc.cluster.local → ClusterIP`. Pods only.
- A Pod calling `https://app.example.com` resolves via public DNS, exits the cluster and re-enters through the ingress (hairpin). Pods should use the Service name instead when talking to a neighbour.
- Tools like `external-dns` can create the public records automatically from Ingress hosts (concept only).

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `kubectl get ingress` | List Ingress objects (ADDRESS empty = no controller/IP) | `kubectl get ing -A` |
| `kubectl describe ingress NAME` | Rules, backends, resolved endpoints, events | `kubectl describe ing web -n demo` |
| `kubectl create ingress` | Imperative Ingress generator | `kubectl create ingress web --class=nginx --rule="app.example.com/*=web:80" --dry-run=client -o yaml` |
| `kubectl get ingressclass` | Show installed controllers | `kubectl get ingressclass` |
| `kubectl -n ingress-nginx get svc` | Find controller external IP/NodePorts | `kubectl -n ingress-nginx get svc ingress-nginx-controller` |
| `kubectl -n ingress-nginx logs deploy/ingress-nginx-controller` | Access and error logs of the proxy | `... logs deploy/ingress-nginx-controller -f --tail=50` |
| `kubectl get endpoints NAME` / `kubectl get endpointslice` | Confirm Service has ready Pod IPs | `kubectl get endpoints web -n demo` |
| `curl --resolve HOST:PORT:IP URL` | Test Ingress without DNS record | `curl --resolve app.example.com:80:203.0.113.10 http://app.example.com/` |
| `curl -H "Host: ..."` | Same idea, HTTP only | `curl -H "Host: app.example.com" http://203.0.113.10/` |
| `curl -vk https://...` | Inspect served certificate | `curl -vk --resolve app.example.com:443:IP https://app.example.com/ 2>&1 | grep -E "subject|issuer"` |
| `helm repo add / install` | Install ingress-nginx / cert-manager | see walkthrough |
| `kubectl get clusterissuer` | Is the ACME account registered | `kubectl get clusterissuer` |
| `kubectl get certificate,certificaterequest,order,challenge -A` | Follow certificate issuance | `kubectl -n demo describe challenge` |
| `kubectl -n kube-system get cm coredns -o yaml` | Read Corefile | — |
| `kubectl -n kube-system rollout restart deploy coredns` | Force CoreDNS to reload config | — |
| `kubectl exec POD -- cat /etc/resolv.conf` | See Pod DNS settings | — |
| `kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup NAME` | One-off DNS test Pod | `... nslookup kubernetes.default` |
| `nslookup` / `dig` inside a Pod | Query CoreDNS | `dig @10.96.0.10 web.demo.svc.cluster.local` |

## Examples / Walkthrough

Lab assumptions: a working cluster (kubeadm/rke2/minikube/kind), `kubectl` and `helm` installed, a node IP or LoadBalancer IP. Domain names can be fake — we use `curl --resolve`. Replace `INGRESS_IP` with your value.

### Part 1 — Two apps behind one Ingress

```bash
# 1. Namespace and two demo apps (commands known from lessons 35-37)
kubectl create namespace demo
kubectl -n demo create deployment blue  --image=nginxdemos/hello:plain-text --replicas=2 --port=80
kubectl -n demo create deployment green --image=hashicorp/http-echo:1.0 -- /http-echo -text=green -listen=:5678
kubectl -n demo expose deployment blue  --port=80
kubectl -n demo expose deployment green --port=80 --target-port=5678

# 2. Verify Services have endpoints BEFORE writing the Ingress
kubectl -n demo get svc,endpoints
```

```bash
# 3. Install the controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
kubectl -n ingress-nginx get pods -w            # wait for Running 1/1
kubectl -n ingress-nginx get svc ingress-nginx-controller
INGRESS_IP=203.0.113.10                         # EXTERNAL-IP, or a node IP with NodePort
```

```yaml
# 4. host-based.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hosts
  namespace: demo
spec:
  ingressClassName: nginx
  rules:
  - host: blue.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: blue, port: {number: 80}}}
  - host: green.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: green, port: {number: 80}}}
```

```bash
kubectl apply -f host-based.yaml
kubectl -n demo get ingress                       # ADDRESS filled after ~30 s
kubectl -n demo describe ingress hosts            # Backends show Pod IPs

curl --resolve blue.example.com:80:$INGRESS_IP  http://blue.example.com/
curl --resolve green.example.com:80:$INGRESS_IP http://green.example.com/
curl -i http://$INGRESS_IP/                       # unknown host -> 404 default backend
```

If using a NodePort: `curl --resolve blue.example.com:31080:<node-ip> http://blue.example.com:31080/`.

### Part 2 — Path-based routing and rewrite

```yaml
# paths.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: paths
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: apps.example.com
    http:
      paths:
      - path: /green(/|$)(.*)
        pathType: ImplementationSpecific
        backend: {service: {name: green, port: {number: 80}}}
      - path: /blue(/|$)(.*)
        pathType: ImplementationSpecific
        backend: {service: {name: blue, port: {number: 80}}}
```

```bash
kubectl apply -f paths.yaml
curl --resolve apps.example.com:80:$INGRESS_IP http://apps.example.com/green
curl --resolve apps.example.com:80:$INGRESS_IP http://apps.example.com/blue/anything
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=5   # see request lines
```

Demonstrate pathType difference live — apply this and test `/exact`, `/exact/`, `/exact/x`:

```yaml
      - path: /exact
        pathType: Exact
        backend: {service: {name: green, port: {number: 80}}}
```

Only `/exact` hits green; the others fall to 404 (or to a `/` Prefix rule if present).

### Part 3 — Rate limit and body size

```bash
kubectl -n demo annotate ingress hosts \
  nginx.ingress.kubernetes.io/limit-rps="2" \
  nginx.ingress.kubernetes.io/proxy-body-size="10m"

for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code} " --resolve blue.example.com:80:$INGRESS_IP http://blue.example.com/
done; echo
# expect a mix of 200 and 503
```

Remove: `kubectl -n demo annotate ingress hosts nginx.ingress.kubernetes.io/limit-rps-` (trailing dash deletes an annotation).

### Part 4 — Manual TLS

```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 30 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=blue.example.com" -addext "subjectAltName=DNS:blue.example.com"
kubectl -n demo create secret tls blue-tls --cert=tls.crt --key=tls.key

kubectl -n demo patch ingress hosts --type merge -p '
spec:
  tls:
  - hosts: [blue.example.com]
    secretName: blue-tls'

curl -ik --resolve blue.example.com:443:$INGRESS_IP https://blue.example.com/
curl -i  --resolve blue.example.com:80:$INGRESS_IP  http://blue.example.com/   # 308 redirect to https
```

Without `-k`, curl rejects the self-signed cert — expected. Inspect: `curl -vk ... 2>&1 | grep -E "subject:|issuer:"`.

### Part 5 — cert-manager with Let's Encrypt (requires public DNS + reachable cluster)

```bash
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set crds.enabled=true
kubectl -n cert-manager get pods
kubectl apply -f clusterissuers.yaml            # staging + prod from Concepts section
kubectl get clusterissuer
```

Create the DNS A record `app.<your-domain>` → ingress IP, then:

```bash
kubectl -n demo create ingress web --class=nginx \
  --rule="app.<your-domain>/*=blue:80,tls=app-tls" \
  --annotation cert-manager.io/cluster-issuer=letsencrypt-staging
kubectl -n demo get certificate -w              # READY becomes True in ~1 min
kubectl -n demo describe certificate app-tls
curl -vk https://app.<your-domain>/ 2>&1 | grep issuer     # (STAGING) ...
```

Switch to production: edit annotation to `letsencrypt-prod`, `kubectl -n demo delete secret app-tls`, wait for the new certificate, `curl -v` now validates without `-k`.

### Part 6 — CoreDNS exploration

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'

kubectl -n demo run dnstest --rm -it --image=busybox:1.36 --restart=Never -- sh
# inside:
cat /etc/resolv.conf
nslookup blue                              # short name, same namespace
nslookup blue.demo.svc.cluster.local
nslookup kubernetes.default
nslookup example.com                       # forwarded upstream
exit
```

Add a stub domain (see Concepts), `rollout restart` CoreDNS, then query `nslookup host.corp.example.com`. Watch CoreDNS logs after adding the `log` plugin to see each query.

### Cleanup

```bash
kubectl delete namespace demo
helm uninstall ingress-nginx -n ingress-nginx
helm uninstall cert-manager -n cert-manager
```

## Common Pitfalls

- **Installed no controller** — Ingress YAML applies fine but ADDRESS stays empty and every request fails. Install ingress-nginx and make sure `ingressClassName` matches `kubectl get ingressclass`.
- **Missing or wrong `ingressClassName`** — No default IngressClass + no field = Ingress ignored. Using the deprecated annotation may also stop working.
- **Backend Service in another namespace** — Ingress can only reference Services in its own namespace. Create the Ingress in the Service's namespace.
- **Wrong Service port** — `port.number` must be the *Service* port, not the container port. 503/502 results.
- **Service has no endpoints** — selector/label mismatch or Pods not Running. `kubectl get endpoints` empty → ingress returns `503 Service Temporarily Unavailable`. Fix the Service before blaming Ingress.
- **`pathType` confusion** — `Prefix /foo` does not match `/foobar`; `Exact /foo` does not match `/foo/`. A trailing-slash bug is the most common "why 404?" in class.
- **Backend 404 after routing works** — the app does not know the `/api` prefix. Use `rewrite-target` with capture groups instead of a bare `rewrite-target: /`.
- **Annotation values not quoted** — `limit-rps: 5` makes YAML emit an integer and the API rejects it ("cannot unmarshal number into string"). Always quote: `"5"`.
- **TLS Secret in the wrong namespace / wrong name** — nginx serves the fake default certificate. Secret must be in the Ingress's namespace and `tls.hosts` must match the certificate.
- **Testing with real Let's Encrypt first** — hits rate limits after a few failures. Use `letsencrypt-staging` until Certificate is Ready, then switch.
- **HTTP-01 never completes** — DNS not pointing to the ingress IP, port 80 blocked by firewall/security group, or `ssl-redirect`/forced HTTPS in front of the solver. Read `kubectl describe challenge` for the exact HTTP status the CA saw.
- **Wildcard cert with HTTP-01** — impossible. Needs DNS-01 solver.
- **Redirect loop behind external TLS-terminating LB** — set `ssl-redirect: "false"` or configure forwarded-proto trust.
- **Large upload fails with 413** — default body limit is 1 MB. Set `proxy-body-size`.
- **Long requests dropped at 60 s (504)** — raise `proxy-read-timeout`.
- **Editing the Corefile with a syntax error** — CoreDNS pods crash and the *whole cluster* loses DNS. Back up first: `kubectl -n kube-system get cm coredns -o yaml > coredns-backup.yaml`.
- **Expecting `dnsPolicy: Default` to be the default** — default is `ClusterFirst`. `Default` removes cluster DNS.
- **`ndots:5` surprise** — external hostnames cause multiple NXDOMAIN lookups; use FQDN with trailing dot or tune `ndots`.
- **Pod using `hostNetwork: true` cannot resolve Services** — set `dnsPolicy: ClusterFirstWithHostNet`.
- **Using public hostnames for pod-to-pod calls** — traffic hairpins through the load balancer. Use `svc.namespace`.

## Troubleshooting Checklist (walk top to bottom)

1. `kubectl get pods,svc,endpoints -n <ns>` — are Pods Ready and does the Service have endpoints?
2. `kubectl run curl --rm -it --image=curlimages/curl --restart=Never -- curl -s http://<svc>.<ns>` — does the Service work from inside?
3. `kubectl describe ingress` — class, rules, backend resolved (`<none>` or "service not found" in Events)?
4. `kubectl -n ingress-nginx get pods,svc` — controller Running, external IP present?
5. `curl --resolve host:80:IP http://host/ -v` — which status? `404 default backend` = host/path rule did not match; `503` = no endpoints; `502` = backend refused/closed; `504` = backend too slow; `308` = redirect to HTTPS.
6. `kubectl -n ingress-nginx logs deploy/ingress-nginx-controller` — upstream errors, config reload failures, admission rejects.
7. TLS: `kubectl get certificate`, `describe challenge`, `curl -vk` to see the served issuer.
8. DNS: `dig host +short` (public) and `nslookup svc.ns` from a Pod (internal); check CoreDNS Pods and logs.

## FAQ

**Q: What is the difference between Ingress and a LoadBalancer Service?**
A: A LoadBalancer Service gives one external IP to one Service at layer 4 (TCP/UDP). Ingress is layer 7 (HTTP/HTTPS): one entry point routes many hostnames and paths to many Services, and can terminate TLS. Ingress itself is usually exposed through a single LoadBalancer Service.

**Q: Do I need an Ingress controller if I create an Ingress resource?**
A: Yes. The resource is just configuration. Without a controller (like ingress-nginx) nothing reads it. Kubernetes does not include one.

**Q: What is IngressClass for?**
A: It names which controller owns an Ingress, so multiple controllers can run side by side. Reference it with `spec.ingressClassName`. One class can be marked default.

**Q: Why does my Ingress show no ADDRESS?**
A: Either no controller is watching it (wrong/missing class), the controller has no external IP (LoadBalancer `<pending>` on bare metal), or the controller Pods are not running.

**Q: Prefix or Exact — which should I use?**
A: `Prefix` for apps and APIs (matches `/api`, `/api/`, `/api/v1/x`). `Exact` for single endpoints like `/healthz`. `ImplementationSpecific` only when you need ingress-nginx regex.

**Q: Why does `/foo` Prefix not match `/foobar`?**
A: Prefix matching works per path element separated by `/`, not per character. `/foo` matches `/foo` and `/foo/anything` but `/foobar` is a different element.

**Q: Can one Ingress route to a Service in another namespace?**
A: No. Backends must be in the Ingress's namespace. Create the Ingress there, or use Gateway API with ReferenceGrant (lesson 39).

**Q: Can I use the same host in two Ingress objects?**
A: Yes, ingress-nginx merges them if paths do not conflict. Conflicting identical host+path: the oldest Ingress wins. Useful for splitting ownership between teams.

**Q: What does `rewrite-target` do exactly?**
A: It changes the URL path nginx sends to the backend. With capture groups (`/$2`) you strip a public prefix like `/api`. The browser URL does not change.

**Q: How do I redirect HTTP to HTTPS?**
A: Add a `tls` block — ingress-nginx redirects with 308 automatically. Force it without TLS block using `force-ssl-redirect: "true"`. Disable with `ssl-redirect: "false"`.

**Q: How do I get HTTPS certificates automatically?**
A: Install cert-manager, create a ClusterIssuer for Let's Encrypt, annotate the Ingress with `cert-manager.io/cluster-issuer: <name>` and add a `tls` block with `secretName`. cert-manager creates the Certificate, completes the ACME challenge and renews before expiry.

**Q: Staging or production issuer?**
A: Test with staging (untrusted certs, generous limits). When Certificate becomes Ready, switch to production and delete the Secret to request a trusted cert. Production has strict rate limits.

**Q: Why is my certificate stuck at READY=False?**
A: `kubectl describe certificate`, then `get certificaterequest,order,challenge` and `describe challenge`. Usual reasons: DNS record missing/wrong, port 80 blocked, cluster not internet-reachable, wrong ingress class in the solver, rate limit.

**Q: Can I get a wildcard certificate?**
A: Only with the DNS-01 challenge and a DNS provider integration — not with HTTP-01.

**Q: Where does the TLS certificate live?**
A: In a Secret (type `kubernetes.io/tls`, keys `tls.crt`, `tls.key`) in the Ingress namespace. cert-manager creates and updates it; ingress-nginx reads it.

**Q: Is TLS between ingress and the Pod encrypted?**
A: Not by default — termination happens at the ingress. For HTTPS backends use `backend-protocol: "HTTPS"`; for end-to-end encryption between Pods you need a service mesh or app-level TLS.

**Q: How do I limit requests per client?**
A: `limit-rps`, `limit-rpm`, `limit-connections` annotations. They are per client IP, so make sure the real client IP reaches nginx.

**Q: Can I put raw nginx config in an annotation?**
A: `configuration-snippet`/`server-snippet` exist but are disabled or restricted in recent versions for security reasons. Prefer dedicated annotations.

**Q: What does CoreDNS do?**
A: It is the cluster DNS server. It answers Service/Pod names (`<svc>.<ns>.svc.cluster.local`) from the Kubernetes API and forwards everything else to upstream DNS.

**Q: How can a Pod reach a Service in another namespace by name?**
A: Use `<svc>.<namespace>` or the FQDN `<svc>.<namespace>.svc.cluster.local`. A bare `<svc>` resolves only in the Pod's own namespace.

**Q: Where is CoreDNS configured?**
A: In the `coredns` ConfigMap in `kube-system` (file name Corefile). Edit it, then wait for `reload` or restart the CoreDNS Deployment.

**Q: How do I resolve a private corporate domain from Pods?**
A: Add a server block for that domain with `forward . <internal-dns-ip>` in the Corefile (a "stub domain"), or change the global `forward` upstream.

**Q: What are the dnsPolicy values?**
A: `ClusterFirst` (default), `Default` (node DNS, no cluster names), `ClusterFirstWithHostNet` (for hostNetwork Pods), `None` (fully custom via `dnsConfig`).

**Q: Why are external DNS lookups slow from Pods?**
A: `ndots:5` makes names with fewer than 5 dots try every search suffix first. Use a trailing dot or lower `ndots` via `dnsConfig`.

**Q: Why did all my Pods lose DNS after I edited CoreDNS?**
A: Corefile syntax error or forwarding loop makes CoreDNS Pods crash. Restore from backup (`kubectl apply -f coredns-backup.yaml`) and restart the Deployment.

**Q: Is Ingress deprecated?**
A: No, but it is frozen — no new features. Gateway API (lesson 39) is the successor and is covered next; Ingress still works and is everywhere in existing clusters.

## Practice / Exercise

### Core

1. Install ingress-nginx with Helm. Record the controller Service type and external IP/NodePorts. Confirm `kubectl get ingressclass`.
2. Deploy two apps (`kubectl create deployment` + `expose`) in namespace `shop`. Write one Ingress with **host-based** rules `a.example.com` and `b.example.com`. Verify with `curl --resolve`.
3. Add a **path-based** Ingress for `example.com`: `/api` → api Service, `/` → frontend Service. Show that the longest path wins by changing YAML order.
4. Demonstrate `pathType` — create `Exact /status` and `Prefix /docs`; test `/status`, `/status/`, `/docs`, `/docs/a`, `/docsx` and explain each result.
5. Configure `defaultBackend` pointing to a custom "not found" Service; test an unknown host.
6. Create a self-signed certificate, store it with `kubectl create secret tls`, enable TLS on an Ingress, and prove the HTTP → HTTPS 308 redirect.
7. Add `limit-rps` and `proxy-body-size` annotations; trigger and screenshot 503 and 413 responses.
8. From a busybox Pod: print `/etc/resolv.conf`, resolve a Service by short name, namespace-qualified name and FQDN, and explain the `search` and `ndots` lines.
9. Read the CoreDNS Corefile; list what each plugin does in your own words.

### Stretch

1. Install cert-manager; create staging and production `ClusterIssuer`s; obtain a **staging** certificate for a real hostname with the annotation method; inspect Certificate → CertificateRequest → Order → Challenge; switch to production.
2. Reproduce and fix three failures: (a) wrong `ingressClassName`, (b) Service selector typo → 503, (c) wrong Service port → 502. Document the symptom and the command that revealed each.
3. Use `rewrite-target` with capture groups to serve an app that only understands `/` under the public prefix `/app1`. Prove the difference vs. bare `rewrite-target: /`.
4. Add a stub domain `corp.test` forwarding to a dnsmasq Pod you deploy; resolve a name from another Pod; back up the Corefile first.
5. Create a Pod with `dnsPolicy: None` and custom `dnsConfig` (`ndots: 1`, extra search domain); confirm via `/etc/resolv.conf`.
6. Canary: two Ingresses for the same host, second with `canary: "true"`, `canary-weight: "30"`; measure the traffic split with a loop of 100 requests.
7. Put an Ingress with `whitelist-source-range` that only allows your IP; test from allowed and disallowed sources.

## Further Reading

- Kubernetes Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
- Ingress controllers: https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
- ingress-nginx docs and annotation reference: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
- cert-manager docs (ACME HTTP-01, ingress annotations): https://cert-manager.io/docs/
- Let's Encrypt rate limits: https://letsencrypt.org/docs/rate-limits/
- DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Customizing DNS service (CoreDNS): https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/
- CoreDNS plugins: https://coredns.io/plugins/

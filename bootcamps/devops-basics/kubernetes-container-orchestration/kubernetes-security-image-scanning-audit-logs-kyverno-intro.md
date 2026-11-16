# Lesson 50: Kubernetes Security – Image Scanning, Audit Logs & Kyverno Intro

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-49 (kubectl, Deployments, static Pods, RBAC, NetworkPolicy, PodSecurity admission, admission webhooks concept, kube-bench/AppArmor/seccomp). Lesson 32 (Trivy intro on a single Docker image). Student has a lab cluster (kubeadm or rke2) with SSH/root access to a control-plane node, `kubectl` admin access, `helm` binary installed (Helm itself is taught in lessons 53-55; here it is only used to install one chart), and `jq`.

## Learning Objectives

By end of lesson student can:
- Explain the three security questions this lesson answers: *what is inside my images* (scanning), *who did what in my cluster* (audit logs), *what is allowed to enter my cluster* (policy as code)
- Run Trivy against images, filesystems, IaC/Kubernetes manifests and a live cluster, and pick the right scanner for each
- Filter results by severity, hide unfixed CVEs, read fix versions and decide on a remediation (bump base image, bump library, accept risk)
- Use Trivy exit codes to gate a pipeline step, write a `.trivyignore` with expiry dates, and run Trivy offline with a pre-downloaded database
- Write a Kubernetes audit policy using the four levels (`None`, `Metadata`, `Request`, `RequestResponse`) and `omitStages`
- Enable auditing on the kube-apiserver static Pod: flags, volume mounts, log rotation
- Read the JSON audit log with `jq` to answer "who deleted that pod", "who exec'd into a pod", "who read secrets"
- Recognise suspicious API activity patterns in audit logs
- Install Kyverno, write `ClusterPolicy` validate rules that deny privileged Pods, `:latest` tags and missing labels, and switch between `Audit` and `Enforce`
- Read `PolicyReport` results, explain mutate and generate rules, and test policies with the Kyverno CLI before applying them

## Topics

- Trivy: trivy image (scan locally), severity levels (CRITICAL/HIGH/MEDIUM), fix versions
- Trivy in CI: scan before push; fail pipeline on CRITICAL; ignoring CVEs (.trivyignore)
- Kubernetes Audit Logs: enabling audit policy, log levels (None/Metadata/Request/RequestResponse)
- Reading audit logs: identifying suspicious API calls, integration with log shipper
- Kyverno intro: install, ClusterPolicy, validate rule (deny privileged pods) — live demo

## Concepts

### Where this lesson fits: supply chain, detection, prevention

Lessons 47-49 locked down *who can call the API* (RBAC), *how pods talk* (NetworkPolicy), *what a pod may request* (PodSecurity) and *how hard the cluster and kernel are* (kube-bench, AppArmor, seccomp). Three gaps remain:

| Gap | Question | Tool in this lesson | Type of control |
|---|---|---|---|
| Vulnerable software inside images | "Does my image contain known-exploitable packages?" | **Trivy** | Preventive (before deploy) |
| No record of API activity | "Who deleted the Deployment at 03:12? Who read the DB secret?" | **Audit logs** | Detective (after the fact) |
| Custom rules beyond built-in admission | "Block `:latest`, require `team` label, forbid privileged Pods" | **Kyverno** | Preventive (at admission) |

Defense in depth: scanning keeps known-bad code out of the registry, admission policy keeps non-compliant objects out of the cluster, audit logs let you investigate whatever got through.

### Software supply chain in one paragraph

Your container is not only your code. A typical image is: *base OS layer* (Debian/Alpine packages) + *language runtime* + *application dependencies* (npm/pip/go modules) + *your code*. Each layer can contain a **CVE** (Common Vulnerabilities and Exposures — a public ID like `CVE-2024-3094` for a known flaw). A **vulnerability scanner** unpacks the image layers, builds a list of installed packages (the **SBOM** — Software Bill Of Materials), and matches name+version against vulnerability databases. It does **not** find unknown bugs, and it does not prove a CVE is exploitable in your setup — it reports *known* issues in *known* versions.

### Trivy

**Trivy** (by Aqua Security, open source) is a single binary scanner. It scans:

| Target | Command | What it finds |
|---|---|---|
| Container image (registry or local daemon) | `trivy image IMG` | OS and language-package CVEs, secrets, misconfig |
| Local directory / repo | `trivy fs DIR` | Dependency CVEs from lock files (`package-lock.json`, `go.sum`, `requirements.txt`), secrets |
| IaC and Kubernetes manifests | `trivy config DIR` | Misconfigurations (Dockerfile, Terraform, K8s YAML, Helm charts) |
| Running cluster | `trivy k8s cluster` | Vulnerabilities in running workload images + misconfig + RBAC issues |
| Git repository | `trivy repo URL` | Same as fs on a remote repo |
| SBOM file | `trivy sbom FILE` | CVEs from an existing SBOM |

**Scanners** (`--scanners`): `vuln` (CVEs), `secret` (hard-coded keys/tokens), `misconfig` (config issues), `license`. Defaults vary by target (`image` → `vuln,secret`; `config` → `misconfig`).

**Vulnerability database.** Trivy downloads a database (`trivy-db`, built from NVD, Debian/Ubuntu/Alpine/Red Hat advisories, GitHub advisories, etc.) on first run and refreshes it every ~6 hours. Cached in `~/.cache/trivy`. No cluster needed; scanning an image only needs the image layers (pulled from registry or read from the local Docker daemon) and the DB.

#### Reading a Trivy table

```
nginx:1.21 (debian 11.3)
Total: 192 (UNKNOWN: 0, LOW: 85, MEDIUM: 52, HIGH: 41, CRITICAL: 14)

┌──────────┬────────────────┬──────────┬──────────────┬───────────────────┬───────────────┬─────────────────────────────┐
│ Library  │ Vulnerability  │ Severity │    Status    │ Installed Version │ Fixed Version │            Title            │
├──────────┼────────────────┼──────────┼──────────────┼───────────────────┼───────────────┼─────────────────────────────┤
│ libssl1.1│ CVE-2022-0778  │ HIGH     │ fixed        │ 1.1.1n-0+deb11u1  │ 1.1.1n-0+deb11u2 │ openssl: infinite loop ... │
│ zlib1g   │ CVE-2022-37434 │ CRITICAL │ fixed        │ 1:1.2.11.dfsg-2   │ 1:1.2.11.dfsg-2+deb11u2 │ zlib: heap overflow  │
│ libc6    │ CVE-2021-33574 │ CRITICAL │ affected     │ 2.31-13           │               │ glibc: mq_notify use-after-free │
└──────────┴────────────────┴──────────┴──────────────┴───────────────────┴───────────────┴─────────────────────────────┘
```

Column meaning:

- **Library** – the package (or `Target` section header names OS vs `node-pkg`/`python-pkg`/`gobinary`).
- **Vulnerability** – CVE (or GHSA, other advisory) ID. Search it on `nvd.nist.gov` or `avd.aquasec.com` for details.
- **Severity** – `UNKNOWN`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` (CVSS-based; vendor severity wins when available).
- **Status** – `fixed` (a patched version exists), `affected` (no fix yet, vendor may fix later), `will_not_fix` (vendor declined, often because the package is deprecated), `fix_deferred`, `end_of_life` (distribution no longer supported).
- **Installed Version** / **Fixed Version** – decide the remediation: if Fixed Version exists, upgrade; if empty, nothing to upgrade to.

#### Triage workflow

1. **Is there a fix?** Filter with `--ignore-unfixed` to see only actionable items.
2. **Is it in the runtime image?** Build-time tools (compilers, `curl`, `git`) belong in the builder stage only (lesson 31 multi-stage). Moving them out removes their CVEs entirely.
3. **Smaller base = fewer CVEs.** `alpine`, `distroless`, `scratch` have far fewer packages than `debian`/`ubuntu`.
4. **Bump base image tag**, rebuild, rescan. Most OS CVEs vanish with a base image refresh.
5. **Language dependencies** (`library` findings): update the dependency in the lock file.
6. **Cannot fix now**: document the exception in `.trivyignore` with an **expiry date** and a reason, so it resurfaces for re-review.
7. **Re-scan on a schedule**: an image that was clean last month may have new CVEs today because the *database* changed, not the image.

A CVE is a *risk indicator*, not a verdict. A CRITICAL in a library your code never calls is lower risk than a MEDIUM in the HTTP parser you expose. Scanners do not do reachability analysis (mostly), so teams set a **policy** (e.g. "no CRITICAL with a fix, ever; HIGH within 14 days") instead of chasing zero.

#### Exit codes and gating

By default Trivy exits `0` even if it found vulnerabilities (it is a *report* tool). `--exit-code 1` makes it exit `1` when something matching the filters is found. Combined with `--severity CRITICAL` this is the standard "fail the build on critical" switch. Any CI system (GitLab CI, GitHub Actions — lessons 59-66) treats non-zero exit as a failed step, so the gating logic is just this one flag.

#### `.trivyignore`

A plain text file, read from the current directory (or `--ignorefile PATH`):

```
# 2026-11-01 owner: platform — glibc, not reachable, no fix upstream
CVE-2021-33574 exp:2027-01-31

# only for this one package path (yaml format, see below)
CVE-2022-37434
```

- One ID per line, `#` comments.
- `exp:YYYY-MM-DD` (supported in current Trivy versions) makes the ignore **expire**; after that date the CVE shows again. Always set it.
- A YAML variant (`.trivyignore.yaml`, `--ignorefile`) can scope by package path and add `statement:` text.
- Alternative: **VEX** (Vulnerability Exploitability eXchange) documents — mention only; they let vendors state "not affected" in a machine-readable file.

#### Offline / air-gapped scanning

```bash
trivy image --download-db-only            # on a connected machine, populates cache
trivy image --skip-db-update --offline-scan IMG   # on the air-gapped machine, reuse cache
```

Copy `~/.cache/trivy/db` into the offline host, or mirror the DB image (`ghcr.io/aquasecurity/trivy-db`) into a private registry and point to it with `--db-repository registry.local/trivy-db`.

#### Scanning a running cluster (brief)

`trivy k8s` connects with your kubeconfig, enumerates workloads, scans each image and checks manifests against misconfiguration rules, and (with `--compliance`) maps results to CIS / NSA / PSS benchmarks. Output is large; use `--report summary` for an overview. The permanent, in-cluster variant is the **Trivy Operator** (runs scans as Jobs and stores `VulnerabilityReport` CRDs) — mention only: it needs its own install and is the same engine. Good fit for "continuously rescan what is already running".

### Kubernetes audit logging

**Audit logging** records requests that reach the **kube-apiserver**: who (user/ServiceAccount, groups, source IP), what (verb + resource + name + namespace), when, and the outcome (HTTP status). Everything in Kubernetes goes through the API server (`kubectl`, controllers, kubelets, operators), so the audit log is the single source of truth for *who changed the cluster*.

Audit logs are **not** application logs, and **not** container logs (`kubectl logs`). They log API *calls*. Audit is **off by default** — a fresh kubeadm cluster has none. Managed clusters (EKS/GKE/AKS) expose it through their cloud logging service.

#### Request stages

Each API request passes through stages; an audit event can be emitted at each:

| Stage | When |
|---|---|
| `RequestReceived` | As soon as the handler receives the request, before anything else |
| `ResponseStarted` | Response headers sent, body not yet (only for long-running requests like `watch`, `exec`) |
| `ResponseComplete` | Body sent (or no body) |
| `Panic` | Server panicked |

`RequestReceived` doubles the log volume and carries no outcome. Almost every policy sets `omitStages: ["RequestReceived"]`.

#### Audit levels

A policy rule decides how much detail to record for matching requests:

| Level | Records | Typical use |
|---|---|---|
| `None` | Nothing | Noisy, harmless requests (health checks, kube-proxy watching endpoints, system:nodes status updates) |
| `Metadata` | Request metadata only: user, verb, resource, name, timestamp, response code. **No bodies.** | Default for most resources — safe and compact |
| `Request` | Metadata + **request body** (what the client sent) | Write operations on important resources (Deployments, RBAC) |
| `RequestResponse` | Metadata + request body + **response body** | Highest detail, heavy, may include secret data — use sparingly |

> **Security trap.** Never log `Secret` or `ConfigMap` content at `Request`/`RequestResponse`: the audit log would then store secret values in plain text. Use `Metadata` for secrets so you see *who read which secret* but not the value.

#### Policy rules — first match wins

An audit policy (`audit.k8s.io/v1`, `kind: Policy`) is an ordered list. For each event the **first matching rule** sets the level. Include a final catch-all rule, otherwise unmatched requests are **not logged**.

Rule fields: `level`, `users`, `userGroups`, `verbs`, `resources` (each with `group` and `resources`/`resourceNames`), `namespaces`, `nonResourceURLs`, `omitStages`.

#### API server flags

| Flag | Meaning |
|---|---|
| `--audit-policy-file=PATH` | Policy file. **Required** to enable auditing |
| `--audit-log-path=PATH` | Write log lines here (`-` = stdout). Setting this enables the log backend |
| `--audit-log-maxage=N` | Days to keep rotated files |
| `--audit-log-maxbackup=N` | How many rotated files to keep |
| `--audit-log-maxsize=N` | Rotate when file exceeds N MB |
| `--audit-log-format=json` | `json` (default) or `legacy` |
| `--audit-webhook-config-file=PATH` | Send events to an HTTP endpoint (kubeconfig-style file) instead of / in addition to a file |
| `--audit-webhook-mode=batch` | `batch` (default), `blocking`, `blocking-strict` |

Two backends: **log** (file on the control-plane node) and **webhook** (HTTP). Most teams use the file backend and ship it with a node log agent, because the webhook backend adds API server dependencies.

#### Anatomy of an audit event (JSON, one per line)

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "e1a4f6a0-6e0d-4a71-8f6f-0c1d7f0d6f6b",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/prod/pods/web-7d4b9-abcde",
  "verb": "delete",
  "user": {"username": "alice", "groups": ["devs", "system:authenticated"]},
  "sourceIPs": ["10.20.0.15"],
  "userAgent": "kubectl/v1.30.2 (linux/amd64) kubernetes/3e1b2a1",
  "objectRef": {"resource": "pods", "namespace": "prod", "name": "web-7d4b9-abcde", "apiVersion": "v1"},
  "responseStatus": {"metadata": {}, "code": 200},
  "requestReceivedTimestamp": "2026-11-16T03:12:41.123456Z",
  "stageTimestamp": "2026-11-16T03:12:41.140000Z",
  "annotations": {"authorization.k8s.io/decision": "allow", "authorization.k8s.io/reason": "RBAC: allowed by RoleBinding ..."}
}
```

Key fields to filter on: `verb`, `user.username`, `user.groups`, `objectRef.resource/name/namespace/subresource`, `responseStatus.code`, `sourceIPs`, `userAgent`, `annotations["authorization.k8s.io/decision"]`.

`kubectl exec` appears as `verb: create` on `objectRef.resource: pods`, `objectRef.subresource: exec` (HTTP 101 upgrade). Reading a secret is `verb: get|list` on `resources: secrets`.

#### Suspicious patterns to hunt for

| Pattern | Why it matters |
|---|---|
| `create` on `pods/exec` or `pods/attach` by a human or unexpected ServiceAccount in prod | Interactive access to workloads |
| `list`/`get` on `secrets` across namespaces by a non-system user | Credential harvesting |
| `create` on `clusterrolebindings`/`rolebindings` binding `cluster-admin` | Privilege escalation |
| Many `403` responses from one user/IP | Probing / brute forcing permissions |
| `system:anonymous` requests with `200` | Anonymous access enabled |
| `delete` on workloads outside business hours | Accidental or malicious change |
| `create` of Pods with `hostPath`, `privileged` | Container breakout attempt (should be blocked by lesson 48/Kyverno, but audit proves attempts) |
| ServiceAccount token used from an IP that is not a cluster node/pod CIDR | Stolen token |
| Changes to `validatingwebhookconfigurations`/`mutatingwebhookconfigurations` | Disabling the policy engine |

#### Integrating with a log shipper

Audit files are only useful if someone looks at them. Pattern:

1. API server writes `/var/log/kubernetes/audit/audit.log` (rotated).
2. A **log shipper agent** (Fluent Bit / Vector / Promtail / Filebeat) runs on the control-plane node, tails the file, parses JSON.
3. Shipper forwards to central storage (Loki/Elasticsearch/VictoriaLogs/cloud logging).
4. Queries and alerts run in the central system (e.g. "alert on `pods/exec` in prod").

Setting up the shipper, storage and alerting is the topic of the monitoring & logging module (lessons 67-72). Here the goal is to produce a correct, rotated JSON file and know what to search for.

### Kyverno

**Kyverno** (Greek "to govern") is a Kubernetes-native **policy engine**. Policies are plain Kubernetes YAML (CRDs) — no new language like Rego (used by OPA Gatekeeper). Kyverno registers itself as an **admission webhook** (the mechanism from lesson 48): every create/update to matching resources is sent to Kyverno, which can **validate** (allow/deny), **mutate** (modify) or **generate** (create other resources), and also **verify images** (signatures — mention only).

Policy kinds:

| Kind | Scope |
|---|---|
| `ClusterPolicy` | Cluster-wide, matches any namespace |
| `Policy` | One namespace only |

Rule types:

| Rule | What it does |
|---|---|
| `validate` | Check the resource against a pattern or deny conditions; block (Enforce) or only report (Audit) |
| `mutate` | Patch the resource on the way in (e.g. add default `securityContext`, add labels) |
| `generate` | Create/synchronise another resource when a trigger appears (e.g. default `NetworkPolicy` and `ResourceQuota` in every new namespace) |
| `verifyImages` | Require signed images (cosign) — mention only |

#### Policy structure

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce      # Enforce = block, Audit = allow + report
  background: true                      # also scan existing resources
  rules:
    - name: no-privileged-containers
      match:
        any:
          - resources:
              kinds: ["Pod"]
      exclude:
        any:
          - resources:
              namespaces: ["kube-system", "kyverno"]
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

Pieces:

- `validationFailureAction`: **`Audit`** records violations in a PolicyReport but lets the object in. **`Enforce`** rejects the request with your `message`. Always roll out new policies in `Audit` first. (In recent Kyverno versions the per-rule field `validate.failureAction` is preferred and the spec-level field is deprecated but still works. Check the installed version.)
- `background: true` – Kyverno also evaluates existing resources periodically and fills reports.
- `match` / `exclude` – select by `kinds`, `names`, `namespaces`, `selector` (labels), `operations` (`CREATE`, `UPDATE`, `DELETE`), `subjects` (users/groups/SAs). `any` = OR of blocks, `all` = AND.
- `validate.pattern` – an overlay that the resource must match. Wildcards: `*` (any chars), `?` (one char; `?*` = non-empty), `!` negation, `|` OR. **Anchors:** `=(field)` – check only *if* the field exists; `(field)` – conditional anchor; `X(field)` – the field must *not* exist; `+(field)` – add if missing (mutate).
- `validate.deny.conditions` – alternative: deny when a JMESPath expression is true (more flexible, e.g. deny when `request.operation == DELETE`).

Pods created by Deployments/StatefulSets/Jobs are covered automatically: for `kinds: ["Pod"]` rules, Kyverno **auto-generates** matching rules for pod controllers (`Deployment`, `DaemonSet`, `StatefulSet`, `Job`, `CronJob`), so the `Deployment` is rejected up front instead of only its Pods failing later.

#### PolicyReport

Kyverno writes results as `PolicyReport` (namespaced) and `ClusterPolicyReport` (cluster-scoped) objects (CRD from the Kubernetes Policy WG):

```bash
kubectl get policyreport -A
kubectl get policyreport -n demo -o yaml
kubectl get clusterpolicyreport
```

Each result has `policy`, `rule`, `result` (`pass`, `fail`, `warn`, `error`, `skip`), `resources` and `message`. Reports are your compliance dashboard: run policies in `Audit`, read reports, fix workloads, then switch to `Enforce`.

#### Kyverno CLI (brief)

The `kyverno` CLI evaluates policies **offline**, no cluster needed — ideal in a pipeline before `kubectl apply`:

```bash
kyverno apply policy.yaml --resource pod.yaml            # test a resource against a policy
kyverno apply policy.yaml --resource ./manifests/        # a directory
kyverno test ./tests/                                    # run kyverno-test.yaml unit tests for policies
```

#### Kyverno vs. what you already know

| Need | PodSecurity admission (lesson 48) | Kyverno |
|---|---|---|
| Block privileged/hostPath/root | Yes (3 fixed profiles) | Yes (any custom rule) |
| Block `:latest` | No | Yes |
| Require labels / resource limits | No | Yes |
| Mutate (inject defaults) | No | Yes |
| Create resources per namespace | No | Yes (generate) |
| Per-policy exceptions | Namespace labels only | `exclude`, `PolicyException` CRD |

PodSecurity is built in and cheap; Kyverno adds flexibility at the cost of running an extra component whose availability affects the API.

## Commands / Syntax Reference

### Trivy

| Command | Purpose | Example |
|---|---|---|
| `trivy image IMG` | Scan an image (vuln + secret) | `trivy image nginx:1.25` |
| `trivy image --severity LIST` | Show only listed severities | `trivy image --severity HIGH,CRITICAL nginx:1.25` |
| `trivy image --ignore-unfixed` | Hide CVEs with no fix available | `trivy image --ignore-unfixed alpine:3.20` |
| `trivy image --exit-code N` | Exit with N when findings match filters | `trivy image --exit-code 1 --severity CRITICAL myapp:1.0` |
| `trivy image --format F -o FILE` | Output `table`, `json`, `sarif`, `cyclonedx`, `spdx-json`, `template` | `trivy image -f json -o out.json myapp:1.0` |
| `trivy image --scanners LIST` | Pick scanners | `trivy image --scanners vuln,secret,misconfig myapp:1.0` |
| `trivy image --input FILE` | Scan a `docker save` tarball | `trivy image --input myapp.tar` |
| `trivy image --ignorefile PATH` | Use a custom ignore file | `trivy image --ignorefile ./.trivyignore myapp:1.0` |
| `trivy image --skip-db-update` | Don't refresh the DB (offline/cache) | `trivy image --skip-db-update myapp:1.0` |
| `trivy image --download-db-only` | Only fetch DB into the cache | `trivy image --download-db-only` |
| `trivy image --db-repository REPO` | Use a mirrored DB | `trivy image --db-repository registry.local/trivy-db myapp:1.0` |
| `trivy fs DIR` | Scan a project directory | `trivy fs --scanners vuln,secret .` |
| `trivy config DIR` | IaC / manifest misconfiguration scan | `trivy config ./k8s/` |
| `trivy k8s cluster --report summary` | Scan a running cluster | `trivy k8s cluster --report summary` |
| `trivy k8s --namespace NS all` | Scan one namespace | `trivy k8s --namespace demo all` |
| `trivy clean --all` | Clear cache | `trivy clean --all` |
| `trivy --version` | Version + DB info | `trivy --version` |

### Audit logging

| Item | Purpose | Example |
|---|---|---|
| `--audit-policy-file` | Enable auditing with this policy | `--audit-policy-file=/etc/kubernetes/audit/policy.yaml` |
| `--audit-log-path` | Log file location | `--audit-log-path=/var/log/kubernetes/audit/audit.log` |
| `--audit-log-maxage/maxbackup/maxsize` | Rotation | `--audit-log-maxage=7 --audit-log-maxbackup=5 --audit-log-maxsize=100` |
| `jq -c 'select(.verb=="delete")' audit.log` | Filter events | see Walkthrough |
| `crictl ps` / `kubectl -n kube-system get pods -l component=kube-apiserver` | Check API server restarted | `crictl ps \| grep kube-apiserver` |

### Kyverno

| Command | Purpose | Example |
|---|---|---|
| `helm repo add kyverno https://kyverno.github.io/kyverno/` | Add chart repo | — |
| `helm install kyverno kyverno/kyverno -n kyverno --create-namespace` | Install | — |
| `kubectl get pods -n kyverno` | Check controllers | — |
| `kubectl get clusterpolicy` (`cpol`) | List policies, READY, action | `kubectl get cpol` |
| `kubectl get policyreport -A` (`polr`) | Violations per namespace | `kubectl get polr -A` |
| `kubectl get clusterpolicyreport` (`cpolr`) | Cluster-scoped results | — |
| `kubectl describe clusterpolicy NAME` | Rules, status, events | `kubectl describe cpol disallow-latest` |
| `kyverno apply POLICY --resource FILE` | Offline test | `kyverno apply p.yaml --resource pod.yaml` |
| `kyverno test DIR` | Policy unit tests | `kyverno test ./tests` |
| `kubectl delete clusterpolicy NAME` | Remove policy | — |

## Examples / Walkthrough

### Part 1 — Trivy on images

**1.1 Install Trivy** (pick one):

```bash
# Debian/Ubuntu official apt repo
sudo apt-get install -y wget gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy

# macOS
brew install trivy

# No install: run as a container (mount docker socket to scan local images)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image nginx:1.25
```

```bash
trivy --version
```

**1.2 First scan (old image on purpose)**

```bash
trivy image nginx:1.21
```

First run downloads the DB (~tens of MB) then prints sections: one for the OS (`nginx:1.21 (debian 11.x)`) and extra sections for language packages. Note the `Total:` line.

**1.3 Filter to what matters**

```bash
trivy image --severity HIGH,CRITICAL nginx:1.21
trivy image --severity CRITICAL --ignore-unfixed nginx:1.21
trivy image --severity CRITICAL --ignore-unfixed nginx:1.21 | head -30
```

Compare totals: `--ignore-unfixed` drops everything nobody can patch yet.

**1.4 Compare base images — the biggest win**

```bash
trivy image --severity HIGH,CRITICAL --quiet nginx:1.21       | grep Total
trivy image --severity HIGH,CRITICAL --quiet nginx:1.27       | grep Total
trivy image --severity HIGH,CRITICAL --quiet nginx:1.27-alpine | grep Total
trivy image --severity HIGH,CRITICAL --quiet gcr.io/distroless/static-debian12 | grep Total
```

Discussion: same app, order-of-magnitude different CVE counts. This is why lesson 31 insisted on minimal base images and multi-stage builds.

**1.5 Fix versions: reading and acting**

```bash
trivy image -f json -o nginx121.json --severity CRITICAL nginx:1.21
jq -r '.Results[].Vulnerabilities[]? | [.PkgName,.VulnerabilityID,.InstalledVersion,.FixedVersion] | @tsv' nginx121.json | sort -u | head
```

For each line: package, CVE, installed version, fixed version. Fixed version empty → no patch → `affected`. Action for OS packages is "rebuild on a newer base image" (not `apt upgrade` in a running container).

**1.6 Scan *your own* image from lesson 31/40**

```bash
docker build -t myapp:1.0 .
trivy image --severity HIGH,CRITICAL myapp:1.0         # uses local docker daemon
docker save myapp:1.0 -o myapp.tar && trivy image --input myapp.tar
```

Observe findings that belong to the **language dependencies** (`Node.js`, `Python`, `gobinary` sections). Fix in `package.json`/`requirements.txt`/`go.mod`, rebuild, rescan.

**1.7 Secrets and misconfiguration in the same image**

```bash
trivy image --scanners vuln,secret,misconfig myapp:1.0
```

`secret` finds things like AWS keys or private keys baked into layers (image layers are permanent — deleting the file in a later layer does not remove it from the earlier layer). `misconfig` flags Dockerfile problems (running as root, missing `HEALTHCHECK`).

**1.8 Scan files and manifests before they reach the cluster**

```bash
trivy fs --scanners vuln,secret ./app          # lock files and secrets in the repo
trivy config ./k8s                              # K8s manifests: root user, no limits, hostPath ...
trivy config --severity HIGH,CRITICAL ./k8s
```

`trivy config` complements Kyverno: it finds the issue in the repo, Kyverno enforces in the cluster.

**1.9 Output formats**

```bash
trivy image -f json  -o report.json  myapp:1.0
trivy image -f sarif -o report.sarif myapp:1.0     # for GitHub code scanning style tools
trivy image -f cyclonedx -o sbom.cdx.json myapp:1.0  # SBOM only
trivy image -f template --template "@contrib/html.tpl" -o report.html myapp:1.0   # template path depends on install
```

### Part 2 — Trivy as a gate (generic pipeline step)

A pipeline step is "run commands, fail if exit code ≠ 0". The same shell script works in any CI system; lessons 59-66 wrap it into GitLab CI / GitHub Actions syntax.

```bash
#!/usr/bin/env bash
# scan-image.sh IMAGE — fail the build when fixable CRITICAL CVEs exist
set -euo pipefail
IMAGE="${1:?usage: scan-image.sh IMAGE}"

# 1) Report everything (never fails) — kept as a build artifact
trivy image --quiet --format json --output trivy-report.json "$IMAGE"

# 2) Human-readable summary of HIGH+CRITICAL in the job log
trivy image --quiet --severity HIGH,CRITICAL --ignore-unfixed "$IMAGE"

# 3) The gate: exit 1 only for fixable CRITICAL findings
trivy image --quiet --severity CRITICAL --ignore-unfixed --exit-code 1 "$IMAGE"
echo "Image $IMAGE passed the CRITICAL gate"
```

```bash
chmod +x scan-image.sh
./scan-image.sh nginx:1.21 ; echo "exit=$?"     # fails: exit=1
./scan-image.sh nginx:1.27-alpine ; echo "exit=$?"   # passes: exit=0
```

Build order in a real pipeline: `docker build` → **scan** → `docker push` (only if scan passed). Scanning *before* push keeps vulnerable images out of the registry. Also schedule a nightly scan of images already in the registry, because the DB changes daily.

### Part 3 — `.trivyignore` with expiry

```bash
cat > .trivyignore <<'EOF'
# glibc CVE: not reachable (no mq_notify use), vendor has no fix — review 2027-01
CVE-2021-33574 exp:2027-01-31
EOF

trivy image --severity CRITICAL nginx:1.21 | grep -c CVE-2021-33574     # appears
trivy image --severity CRITICAL --ignorefile .trivyignore nginx:1.21 | grep -c CVE-2021-33574 || true
```

Try an expired date (`exp:2020-01-01`) and show the CVE comes back.

Rule: every ignore needs *who, why, until when*. An ignore file without dates becomes a permanent blind spot.

### Part 4 — Trivy on the cluster (brief)

```bash
trivy k8s cluster --report summary
trivy k8s --namespace demo --report all --severity HIGH,CRITICAL all
```

It uses your current kubeconfig context, lists images of running workloads, scans them and reports misconfigurations and RBAC findings. For continuous in-cluster scanning use the Trivy Operator (mention only).

### Part 5 — Enable Kubernetes audit logging

Run on the **control-plane node** as root. API server is a static Pod (lesson 49 recap), so edit its manifest and kubelet restarts it.

**5.1 Directories**

```bash
sudo mkdir -p /etc/kubernetes/audit /var/log/kubernetes/audit
```

**5.2 Audit policy** `/etc/kubernetes/audit/policy.yaml`

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # 1. Do not log noisy read-only system traffic
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: ""
        resources: ["nodes", "nodes/status"]
  - level: None
    users:
      - system:kube-controller-manager
      - system:kube-scheduler
      - system:serviceaccount:kube-system:endpoint-controller
    verbs: ["get", "update"]
    namespaces: ["kube-system"]
    resources:
      - group: ""
        resources: ["endpoints"]
  - level: None
    nonResourceURLs:
      - /healthz*
      - /livez*
      - /readyz*
      - /version
      - /metrics

  # 2. Secrets and ConfigMaps: metadata ONLY (never log values)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps", "serviceaccounts/token"]

  # 3. Interactive access: exec/attach/port-forward — who entered a pod
  - level: Metadata
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # 4. RBAC and webhook changes: full request body
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
      - group: "admissionregistration.k8s.io"
        resources: ["validatingwebhookconfigurations", "mutatingwebhookconfigurations"]

  # 5. Workload changes: request body for writes
  - level: Request
    verbs: ["create", "update", "patch", "delete", "deletecollection"]
    resources:
      - group: ""
        resources: ["pods", "services", "namespaces", "persistentvolumeclaims"]
      - group: "apps"
        resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]

  # 6. Catch-all: everything else, metadata only
  - level: Metadata
```

Order matters: the `None` rules for noise are first; the secrets rule is *before* the catch-all and before any rule that could log `Request` bodies for secrets.

**5.3 Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`**

Back up first (a broken manifest = API server down):

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add to `spec.containers[0].command`:

```yaml
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=7
    - --audit-log-maxbackup=5
    - --audit-log-maxsize=100
```

Add to `volumeMounts`:

```yaml
    - mountPath: /etc/kubernetes/audit
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes/audit
      name: audit-log
```

Add to `volumes`:

```yaml
  - hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes/audit
      type: DirectoryOrCreate
    name: audit-log
```

The API server runs in a container, so both the policy directory (input) and log directory (output) must be mounted from the host. Forgetting the mounts is the #1 reason it crash-loops.

**5.4 Verify the restart**

```bash
# API goes away for ~30-60 s while kubelet recreates the static Pod
watch -n2 'crictl ps | grep kube-apiserver'
kubectl get pods -n kube-system -l component=kube-apiserver
sudo ls -l /var/log/kubernetes/audit/
sudo tail -n 3 /var/log/kubernetes/audit/audit.log | jq .
```

If the API server does not come back:

```bash
sudo crictl ps -a | grep kube-apiserver          # look for Exited
sudo crictl logs $(sudo crictl ps -a -q --name kube-apiserver | head -1) 2>&1 | tail -20
# typical: "open /etc/kubernetes/audit/policy.yaml: no such file" (missing mount) or YAML error in policy
sudo cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml   # roll back
```

### Part 6 — Generate activity and read the log

```bash
kubectl create namespace audit-demo
kubectl -n audit-demo create deployment web --image=nginx:1.27-alpine
kubectl -n audit-demo create secret generic db --from-literal=password=S3cr3t
kubectl -n audit-demo get secret db -o yaml > /dev/null
kubectl -n audit-demo exec deploy/web -- ls /
kubectl -n audit-demo delete pod -l app=web
kubectl -n audit-demo create rolebinding oops --clusterrole=cluster-admin --user=mallory
kubectl get pods -A --as=system:anonymous 2>&1 | tail -2          # a forbidden request
```

On the control-plane node (`LOG=/var/log/kubernetes/audit/audit.log`):

```bash
LOG=/var/log/kubernetes/audit/audit.log

# Latest 5 events, compact
sudo tail -n 5 $LOG | jq -c '{verb, user: .user.username, res: .objectRef.resource, sub: .objectRef.subresource, name: .objectRef.name, ns: .objectRef.namespace, code: .responseStatus.code}'

# WHO DELETED a pod?
sudo jq -c 'select(.verb=="delete" and .objectRef.resource=="pods")
  | {time: .requestReceivedTimestamp, user: .user.username, ip: .sourceIPs[0], pod: .objectRef.name, ns: .objectRef.namespace, code: .responseStatus.code}' $LOG

# WHO EXEC'D into a pod?
sudo jq -c 'select(.objectRef.subresource=="exec")
  | {time: .requestReceivedTimestamp, user: .user.username, ip: .sourceIPs[0], pod: .objectRef.name, ns: .objectRef.namespace, agent: .userAgent}' $LOG

# WHO READ SECRETS?
sudo jq -c 'select(.objectRef.resource=="secrets" and (.verb=="get" or .verb=="list"))
  | {time: .requestReceivedTimestamp, user: .user.username, verb, secret: .objectRef.name, ns: .objectRef.namespace}' $LOG

# Which requests were DENIED (403)?
sudo jq -c 'select(.responseStatus.code==403)
  | {user: .user.username, verb, res: .objectRef.resource, ns: .objectRef.namespace, ip: .sourceIPs[0]}' $LOG

# New cluster-admin bindings (RequestResponse body shows subjects)
sudo jq -c 'select(.objectRef.resource=="rolebindings" and .verb=="create")
  | {user: .user.username, name: .objectRef.name, subjects: .requestObject.subjects, role: .requestObject.roleRef.name}' $LOG

# Top users by number of requests (spot noisy/odd identities)
sudo jq -r '.user.username' $LOG | sort | uniq -c | sort -rn | head

# Anonymous access that SUCCEEDED
sudo jq -c 'select(.user.username=="system:anonymous" and .responseStatus.code<300)' $LOG
```

Check that the secret *value* is not in the log:

```bash
sudo grep -c 'S3cr3t' $LOG        # expect 0 — Metadata level logs no body
# if >0 your policy logs secret bodies: fix rule order NOW and rotate the secret
```

Discussion: reading the audit log tells you *who*, *what*, *from where*, *when* and *allowed or denied*. It cannot tell you what a user typed inside `kubectl exec` — that is outside the API server's view.

### Part 7 — Ship the log (concept)

```
/var/log/kubernetes/audit/audit.log  ──tail──>  node agent (Fluent Bit / Vector / Promtail)
                                                     │ parse JSON, add labels (cluster, node)
                                                     ▼
                                   central log store (Loki / Elasticsearch / VictoriaLogs)
                                                     ▼
                                   queries + alerts: "pods/exec in prod", "403 burst", "new cluster-admin binding"
```

Keep local rotation (`maxage/maxbackup/maxsize`) as a safety net: an attacker with root on the node can delete local logs, and a full disk on a control-plane node can take down etcd. Shipper installation and alert rules: lessons 67-72.

### Part 8 — Kyverno install

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
kubectl get pods -n kyverno
kubectl get crd | grep kyverno
kubectl get validatingwebhookconfigurations | grep -i kyverno
```

> The chart creates several controllers (admission, background, cleanup, reports). `helm install` is used here only to get Kyverno running; chart structure, values and templating are lessons 53-55.

Wait until all pods are `Running`/`Ready` before applying policies — the webhook is registered as soon as the admission controller starts.

### Part 9 — Policy 1: deny privileged Pods (start in Audit)

```yaml
# disallow-privileged.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: no-privileged
      match:
        any:
          - resources:
              kinds: ["Pod"]
      exclude:
        any:
          - resources:
              namespaces: ["kube-system", "kyverno"]
      validate:
        message: "Privileged containers are not allowed (securityContext.privileged must be false)."
        pattern:
          spec:
            =(initContainers):
              - =(securityContext):
                  =(privileged): "false"
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

```bash
kubectl apply -f disallow-privileged.yaml
kubectl get cpol
```

Test: a privileged pod in Audit mode is **allowed** but reported.

```bash
cat > priv-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: priv
  namespace: audit-demo
spec:
  containers:
    - name: c
      image: nginx:1.27-alpine
      securityContext:
        privileged: true
EOF
kubectl apply -f priv-pod.yaml           # created (Audit)
kubectl get polr -n audit-demo           # shows FAIL count
kubectl get polr -n audit-demo -o jsonpath='{range .items[*].results[*]}{.policy}{" "}{.rule}{" "}{.result}{"\n"}{end}'
```

Switch to Enforce and retest:

```bash
kubectl delete pod priv -n audit-demo
kubectl patch clusterpolicy disallow-privileged --type merge -p '{"spec":{"validationFailureAction":"Enforce"}}'
kubectl apply -f priv-pod.yaml
```

Expected:

```
Error from server: error when creating "priv-pod.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/audit-demo/priv was blocked due to the following policies

disallow-privileged:
  no-privileged: 'validation error: Privileged containers are not allowed (securityContext.privileged must be false). rule no-privileged failed at path /spec/containers/0/securityContext/privileged/'
```

Read the error: it names the **policy**, the **rule**, your **message** and the failing **path**. This is the user experience developers get — write messages that tell them how to fix it.

Also test through a Deployment (auto-generated controller rules):

```bash
cat > bad-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad
  namespace: audit-demo
spec:
  replicas: 1
  selector:
    matchLabels: {app: bad}
  template:
    metadata:
      labels: {app: bad}
    spec:
      containers:
        - name: c
          image: nginx:1.27-alpine
          securityContext:
            privileged: true
EOF
kubectl apply -f bad-deploy.yaml      # rejected at the Deployment level
```

The Deployment itself is rejected, not just its Pods later.

### Part 10 — Policy 2: disallow `:latest` and untagged images

```yaml
# disallow-latest-tag.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds: ["Pod"]
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
      validate:
        message: "Using ':latest' is not allowed. Pin a version tag or digest."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

```bash
kubectl apply -f disallow-latest-tag.yaml
kubectl -n audit-demo run t1 --image=nginx            # blocked: no tag
kubectl -n audit-demo run t2 --image=nginx:latest     # blocked: latest
kubectl -n audit-demo run t3 --image=nginx:1.27-alpine   # allowed
```

Why block `latest`: not reproducible (a rollout can pull a different image than yesterday), `imagePullPolicy` defaults to `Always` for it, rollbacks are meaningless, and scan results for `latest` change underneath you.

### Part 11 — Policy 3: require labels

```yaml
# require-labels.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-label
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: check-team-label
      match:
        any:
          - resources:
              kinds: ["Deployment", "StatefulSet", "DaemonSet"]
              namespaces: ["audit-demo", "prod", "staging"]
      validate:
        message: "The label 'team' is required on workloads."
        pattern:
          metadata:
            labels:
              team: "?*"
```

```bash
kubectl apply -f require-labels.yaml
kubectl -n audit-demo create deployment nolabel --image=nginx:1.27-alpine      # allowed (Audit), reported
kubectl get polr -n audit-demo
kubectl -n audit-demo label deployment nolabel team=platform
```

`"?*"` = at least one character. Audit mode gives teams time to add labels before you switch to Enforce.

### Part 12 — Mutate and generate (mention + short demo)

**Mutate — add a default label to every Pod:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-managed-by
spec:
  rules:
    - name: add-label
      match:
        any:
          - resources:
              kinds: ["Pod"]
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              +(managed-by): kyverno
```

`+(managed-by)` = "add if not present". Mutation runs *before* validation, so validate rules see the mutated object.

**Generate — a default-deny NetworkPolicy for every new namespace** (ties to lesson 48):

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: default-deny-netpol
spec:
  rules:
    - name: generate-default-deny
      match:
        any:
          - resources:
              kinds: ["Namespace"]
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny
        namespace: "{{request.object.metadata.name}}"
        synchronize: true
        data:
          spec:
            podSelector: {}
            policyTypes: ["Ingress", "Egress"]
```

`synchronize: true` re-creates the object if someone deletes it. Test: `kubectl create ns gen-test; kubectl get netpol -n gen-test`. (Kyverno needs RBAC permission to create the generated kind; the default chart permits common kinds, other kinds need an extra ClusterRole.)

### Part 13 — Test policies offline with the CLI

```bash
# install: https://kyverno.io/docs/kyverno-cli/  (binary or `brew install kyverno`)
kyverno version
kyverno apply disallow-latest-tag.yaml --resource priv-pod.yaml
kyverno apply disallow-privileged.yaml --resource priv-pod.yaml
```

Output shows `pass: N, fail: M` per policy/rule/resource. Put policies and sample manifests in Git, run `kyverno apply` as a pipeline step: bad manifests fail the build *before* they reach the cluster API.

### Part 14 — Clean up

```bash
kubectl delete clusterpolicy disallow-privileged disallow-latest-tag require-team-label --ignore-not-found
kubectl delete clusterpolicy add-managed-by default-deny-netpol --ignore-not-found
kubectl delete namespace audit-demo gen-test --ignore-not-found
rm -f .trivyignore priv-pod.yaml
# Keep audit logging on if you proceed to lesson 51; otherwise restore the backup manifest
```

## Common Pitfalls

- **Treating Trivy's CVE count as a score** — zero is rarely reachable. Define a policy (e.g. no fixable CRITICAL) and gate on that.
- **Forgetting `--exit-code 1`** — Trivy exits `0` by default, so the "gate" never fails. Check the step actually goes red on a known-bad image.
- **Scanning after push** — the vulnerable image is already in the registry and may already be pulled. Scan in the build, push only on success.
- **Scanning once and never again** — the vulnerability DB changes daily. Rescan images already in the registry and in the cluster on a schedule.
- **`apt upgrade` inside the Dockerfile as a fix** — makes builds non-reproducible and hides the real fix. Update the pinned base image tag instead.
- **`.trivyignore` without dates** — becomes permanent. Always add `exp:` and a comment with reason and owner.
- **Ignoring base image choice** — switching from `debian` to `alpine`/distroless removes most findings with zero code changes.
- **Enabling audit without mounting the policy/log directories** — the API server container cannot see the files and crash-loops. Add both `hostPath` volumes and `volumeMounts`.
- **Editing the API server manifest without a backup** — a typo takes the whole cluster API down. Back up, then edit; roll back from `crictl logs` hints.
- **No catch-all rule in the audit policy** — unmatched requests are silently not logged. End with `level: Metadata`.
- **Logging Secrets at `Request`/`RequestResponse`** — secret values end up in the audit log in plain text. Use `Metadata` for `secrets` and place that rule before broader rules.
- **Leaving `RequestReceived` in** — doubles volume for no gain. Use `omitStages: ["RequestReceived"]`.
- **No rotation on the audit log** — disk fills on the control-plane node and can break etcd. Set `maxage/maxbackup/maxsize`.
- **Audit logs left on the node only** — attacker with root deletes them; nobody ever reads them. Ship to central storage and alert (lessons 67-72).
- **Starting Kyverno policies in `Enforce`** — blocks deployments of existing apps. Start in `Audit`, read PolicyReports, fix workloads, then enforce.
- **Not excluding system namespaces** — policy blocks `kube-system` components or Kyverno itself and the cluster cannot heal. Exclude `kube-system` and `kyverno`.
- **Kyverno down = API calls fail** — with `failurePolicy: Fail` (default for validating webhooks) an unavailable Kyverno blocks creates/updates it matches. Run multiple replicas for production and keep namespace exclusions on the webhook.
- **Pattern anchors misunderstood** — plain `securityContext: privileged: "false"` *requires* the field to exist; `=(securityContext)` means "if present, then check". Use `=()` for optional fields.
- **Policy matches `Pod` only and Deployment still applies, then fails later** — Kyverno autogenerates controller rules by default; if you disabled autogen (annotation), only the Pods are rejected and the ReplicaSet loops with events.
- **Expecting Kyverno to scan images for CVEs** — it does not (unless integrated with signatures/attestations). Trivy and Kyverno solve different problems and are used together.

## FAQ

**Q: What is the difference between Trivy and Kyverno?**
A: Trivy looks *inside* images (and manifests) for known vulnerabilities and misconfigurations. Kyverno decides at admission time whether a Kubernetes *object* is allowed. Use Trivy to find a vulnerable image; use Kyverno to enforce rules like "no `:latest`" or "no privileged Pods". They complement each other.

**Q: Why does Trivy report vulnerabilities in a brand-new image?**
A: The image contains OS and library packages whose known CVEs are already published. Trivy matches installed versions against a database that is updated multiple times per day, so an image built yesterday can have new findings today without any change in the image.

**Q: What does "fixed version" mean and what do I do with it?**
A: It is the package version in which the CVE is patched. If installed < fixed, upgrade. For OS packages that means rebuilding on a newer base image; for language dependencies it means bumping the version in your lock file and rebuilding.

**Q: A CVE has no fixed version. Now what?**
A: Options: (1) accept the risk and add it to `.trivyignore` with an expiry and reason; (2) switch to a base image that does not include the package; (3) remove the package (multi-stage build, distroless); (4) add compensating controls (read-only filesystem, seccomp, NetworkPolicy). Re-review at the expiry date.

**Q: What does `--ignore-unfixed` do and is it safe to use?**
A: It hides findings where no patch exists yet. Good for gates ("fail on what I can fix now"). Keep a periodic full report too so unpatched issues stay visible.

**Q: Which severity should fail the pipeline?**
A: Common policy: fail on `CRITICAL` (fixable) always; track `HIGH` with a deadline (for example 14 days); report-only for `MEDIUM`/`LOW`. Adjust to your risk level and tolerance.

**Q: How do I scan an image that is not in a registry yet?**
A: `trivy image myapp:1.0` uses the local Docker daemon, or export with `docker save myapp:1.0 -o myapp.tar` and run `trivy image --input myapp.tar`.

**Q: Can Trivy run without internet?**
A: Yes. Download the DB on a connected machine (`trivy image --download-db-only`), copy the cache or mirror the DB image to a private registry (`--db-repository`), and run with `--skip-db-update --offline-scan`.

**Q: Why would I scan Kubernetes YAML with `trivy config`?**
A: It catches misconfigurations (running as root, no resource limits, hostPath mounts, missing securityContext) in the repository *before* deploy, giving developers fast feedback. Kyverno or PodSecurity then enforce the same rules at the cluster.

**Q: What is an SBOM and do I need one?**
A: A Software Bill Of Materials: the list of packages inside an image. `trivy image -f cyclonedx` produces one. It helps when a new CVE is announced: you can search stored SBOMs for the affected package without rescanning every image.

**Q: What exactly does the Kubernetes audit log record?**
A: API server requests: who (user, groups, ServiceAccount, source IP, user agent), what (verb, resource, name, namespace), when, and the response code. It does not record container stdout/stderr and it does not record commands typed inside `kubectl exec`.

**Q: Is audit logging enabled by default?**
A: No. You must pass `--audit-policy-file` (and a backend such as `--audit-log-path`) to the kube-apiserver. Managed Kubernetes services expose it through their own logging features.

**Q: What are the four audit levels?**
A: `None` (log nothing), `Metadata` (who/what/when without bodies), `Request` (adds the request body), `RequestResponse` (adds request and response bodies). Higher levels cost more storage and can leak sensitive data.

**Q: Why use Metadata for Secrets?**
A: Because `Request`/`RequestResponse` would store secret values in the log. With `Metadata` you still see who read or changed which secret and when, without the value.

**Q: Why does rule order matter in an audit policy?**
A: The first matching rule wins. Put specific rules (silence noise, protect secrets) before general ones, and end with a catch-all. Unmatched events are not logged.

**Q: What is `omitStages: ["RequestReceived"]` for?**
A: Each request can produce several events. `RequestReceived` fires before the outcome is known and doubles volume. Omitting it keeps only `ResponseStarted`/`ResponseComplete` events that contain the result.

**Q: The API server will not start after I enabled audit. What do I check?**
A: `crictl ps -a` and `crictl logs` on the control-plane node. Typical causes: policy file missing in the container (no `hostPath`/`volumeMounts`), invalid policy YAML or `apiVersion`, wrong flag name, or log directory not writable. Restore the backup manifest and retry.

**Q: How do I find who deleted a pod?**
A: `jq -c 'select(.verb=="delete" and .objectRef.resource=="pods" and .objectRef.name=="NAME")' audit.log` — look at `user.username`, `sourceIPs`, `userAgent`. If a controller recreated it, also look for `delete` events by ReplicaSet/Deployment controllers.

**Q: How do I find who ran `kubectl exec`?**
A: Filter `objectRef.subresource=="exec"`. The event shows the user, target pod and source IP. The commands themselves are not in the audit log (for that you need runtime security tooling, out of scope).

**Q: Does auditing slow the cluster?**
A: Slightly. `Metadata` for most traffic, `None` for health/watch noise, and `omitStages` keep the cost low. `RequestResponse` on high-volume resources is the expensive part.

**Q: How do audit logs get into my logging stack?**
A: Write them to a rotated file on the control-plane node, then tail with a log agent (Fluent Bit, Vector, Promtail, Filebeat) and forward to central storage. The webhook backend sends events directly over HTTP but couples API server availability to the receiver. Details in lessons 67-72.

**Q: What is Kyverno and how is it different from OPA Gatekeeper?**
A: Both are policy engines using admission webhooks. Kyverno policies are plain Kubernetes YAML and can also mutate and generate resources; Gatekeeper uses Rego (a dedicated language) and ConstraintTemplates. Kyverno has a lower learning curve for Kubernetes users.

**Q: What is the difference between `Audit` and `Enforce`?**
A: `Audit` allows the resource and records a violation in a PolicyReport; `Enforce` rejects the request with your message. Always start in `Audit`, fix violations, then switch.

**Q: Where do I see Audit-mode violations?**
A: `kubectl get policyreport -A` and `kubectl get clusterpolicyreport`. Each result lists policy, rule, resource and pass/fail.

**Q: Policy applied, but a Deployment with `:latest` still got in. Why?**
A: Common causes: policy is in `Audit` not `Enforce`; the namespace is in `exclude`; the Deployment existed before the policy and `background` scanning only reports it; or the Kyverno pods were not ready when the Deployment was created and webhook `failurePolicy` is `Ignore`. Check `kubectl describe cpol NAME` and webhook status.

**Q: What do `=(field)` and `?*` mean in a pattern?**
A: `=(field)` means "if the field exists, it must match" (optional field); `?*` means "at least one character" (non-empty string); `*` matches any characters; `!value` negates; `a|b` is OR.

**Q: Can I allow an exception to a policy?**
A: Yes: `exclude` by namespace/label/user in the rule, or a `PolicyException` resource for named workloads, so exceptions are reviewed in Git rather than by weakening the policy.

**Q: What happens if Kyverno is down?**
A: For webhook configurations with `failurePolicy: Fail`, requests matching Kyverno's webhook are rejected until it recovers. Production installs run several admission-controller replicas and exclude system namespaces so the cluster can always repair itself.

**Q: Do I still need PodSecurity admission if I have Kyverno?**
A: PodSecurity is built in, simple and has no extra moving parts: keep it as the baseline. Use Kyverno for rules PodSecurity cannot express (tags, labels, limits, registries, mutation, generation).

**Q: Can I test Kyverno policies without a cluster?**
A: Yes, the `kyverno` CLI: `kyverno apply policy.yaml --resource manifests/`, and `kyverno test` for unit tests. Run it in the pipeline to fail bad manifests early.

## Practice / Exercise

### Core

1. **Scan and compare.** Run `trivy image` on `nginx:1.21`, `nginx:1.27` and `nginx:1.27-alpine` with `--severity HIGH,CRITICAL`. Record the `Total:` line of each and explain the differences.
2. **Read a finding.** For one CRITICAL result, write down: package, CVE ID, installed version, fixed version, and the action you would take.
3. **Gate script.** Create `scan-image.sh` from the lesson. Prove it exits `1` for `nginx:1.21` and `0` for a clean image. Show `echo $?` for each.
4. **Ignore with expiry.** Add one CVE to `.trivyignore` with `exp:` and a reason comment. Show it disappears, then change the date to the past and show it returns.
5. **Enable audit logging** on the control-plane node with the lesson policy. Show `audit.log` receiving events and that the API server is `Running`.
6. **Answer from the log.** Create a namespace, a Deployment and a Secret; delete a Pod; `exec` into a Pod. Use `jq` to produce one line each for: who deleted the pod, who exec'd, who read the secret. Prove `grep` finds no secret value.
7. **Install Kyverno** and apply `disallow-privileged` in `Audit`. Create a privileged Pod, find the violation in the PolicyReport, switch to `Enforce`, and show the denial message.
8. **Disallow `:latest`.** Apply `disallow-latest-tag` in `Enforce`; show `nginx` and `nginx:latest` are rejected and `nginx:1.27-alpine` is accepted.

### Stretch

1. **Trivy on the manifests.** Run `trivy config` on your lesson 40 manifests and fix two HIGH findings (e.g. add `securityContext`, `readOnlyRootFilesystem`). Rescan and show the improvement.
2. **Offline scan.** Download the DB, disconnect (or block outbound with a NetworkPolicy/firewall), and scan with `--skip-db-update --offline-scan`.
3. **Cluster scan.** Run `trivy k8s cluster --report summary` and pick the three most critical workloads; propose fixes.
4. **Detect an attack in the audit log.** Create a RoleBinding granting `cluster-admin` to a user, run `kubectl get secrets -A --as=mallory`, and write one `jq` query per event that would raise an alert. Draft the alert condition in words (alert rules themselves belong to lessons 67-72).
5. **Tighten the audit policy.** Add a rule that records `RequestResponse` for `ClusterRoleBinding` changes and `None` for a noisy user of your choice; compare log growth over 10 minutes with `wc -l`.
6. **Require resource limits** with a Kyverno policy (`resources.limits.memory` and `cpu` required on containers), first in `Audit`, then `Enforce`; read the PolicyReport before enforcing.
7. **Mutate.** Write a mutate rule that adds `securityContext.allowPrivilegeEscalation: false` to Pods that do not set it (`+(allowPrivilegeEscalation)`). Verify with `kubectl get pod -o yaml`.
8. **Generate.** Make a policy that generates a `ResourceQuota` in every new namespace. Create a namespace and verify; delete the quota and watch it return (`synchronize: true`).
9. **CLI in the pipeline.** Write a shell script that runs `kyverno apply` over a directory of manifests and exits non-zero on failure; test with one good and one bad manifest.

## Further Reading

- Trivy documentation: https://trivy.dev/latest/docs/
- Trivy ignoring findings (`.trivyignore`, VEX): https://trivy.dev/latest/docs/configuration/filtering/
- Trivy Kubernetes scanning and Trivy Operator: https://trivy.dev/latest/docs/target/kubernetes/ , https://aquasecurity.github.io/trivy-operator/
- Kubernetes auditing: https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/
- Audit policy reference: https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/
- Kyverno documentation: https://kyverno.io/docs/
- Kyverno policy library (ready policies): https://kyverno.io/policies/
- Kyverno CLI: https://kyverno.io/docs/kyverno-cli/
- NVD (CVE database): https://nvd.nist.gov/ ; Aqua vulnerability DB: https://avd.aquasec.com/
- CIS Kubernetes Benchmark and NSA/CISA Kubernetes Hardening Guide (background reading)

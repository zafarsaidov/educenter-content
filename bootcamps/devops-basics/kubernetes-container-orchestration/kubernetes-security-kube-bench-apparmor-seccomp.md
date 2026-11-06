# Lesson 49: Kubernetes Security – Kube-Bench, AppArmor, Seccomp

**Module:** Kubernetes – Container Orchestration
**Duration:** 120-150 min
**Prerequisites:** Lessons 34-48 (cluster architecture and static pods, kubectl, Pods/Deployments, Services, RBAC, NetworkPolicy, PodSecurity admission). Lesson 11 (Linux hardening basics) and lesson 32 (container security flags) help. Student has SSH access to a lab cluster (kubeadm or rke2 control-plane node plus at least one worker) and can edit files as root.

## Learning Objectives

By end of lesson student can:
- Explain what the CIS Kubernetes Benchmark is and why a cluster audit is run against it
- Run kube-bench as a Job inside the cluster and as a binary on a node, selecting targets and individual checks
- Read PASS / FAIL / WARN / INFO results and locate the remediation text for each failed check
- Apply typical remediations: API server flags, etcd data directory permissions, kubelet configuration, controller-manager flags
- Edit a static Pod manifest safely and verify that the component restarted with the new flags
- Explain what AppArmor is, the three profile modes (enforce / complain / unconfined), and why a profile must be loaded on every node
- Write a small AppArmor profile, load it with `apparmor_parser`, and attach it to a container with `securityContext.appArmorProfile` (K8s 1.30+) or the legacy annotation
- Explain seccomp, the difference between `RuntimeDefault`, `Unconfined` and `Localhost` profiles, and write a custom JSON profile
- Discover which syscalls an application needs and build a profile that blocks the rest
- Verify with `kubectl exec` that AppArmor and seccomp restrictions are really applied

## Topics

- kube-bench: CIS Kubernetes Benchmark, installation, running kube-bench, reading results
- Remediating common kube-bench failures: API server flags, etcd permissions, kubelet config
- AppArmor: Linux security module, profiles (enforce/complain/unconfined), aa-status, aa-genprof
- Applying AppArmor to containers: securityContext.appArmorProfile (K8s 1.30+); pod annotation method
- Seccomp: syscall filtering, runtime/default profile, custom JSON profile, securityContext.seccompProfile

## Concepts

### Where this lesson fits in cluster security

Lessons 47-48 secured the **API layer** (who may call the API: authentication, RBAC) and the **network and admission layer** (NetworkPolicy, PodSecurity, admission controllers). This lesson secures two other layers:

1. **The cluster itself** — are control-plane components, etcd and kubelets configured according to a known-good baseline? Tool: **kube-bench**.
2. **The container's interaction with the Linux kernel** — even a non-root container talks to the kernel through syscalls and touches files. Linux can limit both. Tools: **AppArmor** (files, capabilities, network access by path/profile) and **seccomp** (which syscalls are allowed at all).

Mental model, outermost to innermost:

```
Cluster config (kube-bench)  ->  RBAC / admission  ->  NetworkPolicy
          ->  securityContext (user, capabilities)  ->  AppArmor  ->  seccomp  ->  kernel
```

Every layer assumes the one outside it can fail. A container escape bug that bypasses namespaces still has to get past seccomp and AppArmor.

### CIS Kubernetes Benchmark

The **Center for Internet Security (CIS)** publishes the **CIS Kubernetes Benchmark**: a long, numbered checklist of configuration settings, each with a rationale, an audit command, and remediation steps. It is the de-facto baseline that auditors and compliance frameworks (PCI-DSS, SOC 2, ISO 27001 mappings) ask for.

Sections of the benchmark (numbers appear in kube-bench output):

| Section | Content |
|---|---|
| 1 | Control plane node configuration: 1.1 file permissions of manifests/certs, 1.2 API server flags, 1.3 controller-manager, 1.4 scheduler |
| 2 | etcd configuration (TLS, client cert auth, data dir permissions) |
| 3 | Control plane configuration: authentication, logging/audit policy |
| 4 | Worker node configuration: 4.1 kubelet file permissions, 4.2 kubelet flags/config |
| 5 | Policies: RBAC, ServiceAccounts, Pod security, NetworkPolicy, secrets, general policies |

Checks are of two kinds:
- **Automated** — a command can decide PASS/FAIL (shown as PASS or FAIL).
- **Manual** — a human must decide (shown as WARN). Example: "Ensure that the cluster-admin role is only used where required".

Different Kubernetes versions and distributions have different benchmark versions (`cis-1.8`, `cis-1.9`, `rke2-cis-1.8`, `eks-1.5.0`, `gke-1.6.0`...). Always match the profile to your distro.

### kube-bench

**kube-bench** is an open-source Go tool from Aqua Security that runs the CIS checks against a node. It reads the process command lines, config files and file permissions on the node and compares them with the benchmark.

Key facts:
- It must run **on the node it audits** (needs host PID namespace and host file access). Control-plane checks run on control-plane nodes, worker checks on worker nodes.
- It is **read-only** — it reports, it never changes configuration.
- It auto-detects the Kubernetes version and picks the benchmark, or you force it with `--benchmark`.
- Run it **per node type**: running master targets on a worker simply yields failures/INFO about missing files.

Two ways to run:

| Mode | When |
|---|---|
| Binary / package installed on the node | Ad-hoc audit by an admin with SSH access; scripted with Ansible |
| Kubernetes Job (hostPID, hostPath mounts) | No SSH; rerun from CI; one Job per node group |

### Reading kube-bench results

Output is grouped by section. Each check has a status:

| Status | Meaning |
|---|---|
| `[PASS]` | Check ran and is compliant |
| `[FAIL]` | Check ran and found a non-compliant setting — **must fix** (or accept the risk) |
| `[WARN]` | Manual check, or the tool could not decide — a human must review |
| `[INFO]` | Informational only; no action |

After the checks, kube-bench prints **Remediations** — one block per failed/warned check with the exact action to take — and a **Summary** with counts:

```
== Summary master ==
42 checks PASS
11 checks FAIL
11 checks WARN
0 checks INFO
```

The summary is the number to track over time. A score is not a security guarantee; 100% PASS on a cluster with a public anonymous kubelet port that the benchmark did not test is still unsafe.

### Judging a FAIL: fix, accept or document

Not every FAIL needs fixing. Process used in practice:
1. Read the rationale. Is the risk real in your environment?
2. Does the fix break something the platform needs? (Example: `--read-only-port=0` breaks old metrics scrapers that used port 10255.)
3. Fix it, or write down an exception with owner and date. Auditors accept documented exceptions; they do not accept silence.

### Static Pods: how control-plane components are configured

On kubeadm clusters the API server, controller-manager, scheduler and etcd run as **static Pods**: the kubelet watches `/etc/kubernetes/manifests/` and runs whatever YAML it finds there (lesson 52 builds a cluster this way; students saw the idea in lesson 34). Consequences for remediation:
- You **edit the YAML file on the control-plane node**, not through `kubectl`.
- On save, kubelet notices the change and **recreates** the Pod with the new flags. Brief API outage for the API server on a single-control-plane cluster.
- A typo in the manifest means the component does not come back. Always keep a backup **outside** the manifests directory (a `.bak` file inside that directory would be treated as another manifest only if it ends in `.yaml/.yml/.json`; still, keep backups elsewhere).

rke2 differs: RKE2 ships a hardened default and configures components through `/etc/rancher/rke2/config.yaml` plus static pod manifests under `/var/lib/rancher/rke2/agent/pod-manifests/`. Hardening there is done through the config file (e.g. `profile: cis`), not by editing the generated manifests.

### Linux Security Modules (LSM) and AppArmor

A **Linux Security Module** is a kernel framework that adds **mandatory access control (MAC)** on top of normal Unix permissions (discretionary access control, DAC: owner/group/mode). With DAC a root process can do anything; with MAC even root is bound by policy.

**AppArmor** is the LSM used by default on Ubuntu, Debian and SUSE (RHEL-family distributions use SELinux instead — AppArmor profiles are not available there). AppArmor confines a **program** by **path-based profile**:
- which files it may read / write / execute (`/etc/passwd r,`)
- which Linux capabilities it may use (`capability net_bind_service,`)
- network access by family/type (`network inet tcp,`)
- mount, ptrace, signal rules

Profile modes:

| Mode | Behaviour |
|---|---|
| **enforce** | Violations are **blocked** and logged |
| **complain** | Violations are **allowed** but logged (used for learning and tuning a profile) |
| **unconfined** | No profile applies to the process |

AppArmor on a node is a **host** feature. Kubernetes only tells the container runtime "start this container under profile X"; the profile has to **already be loaded in the kernel of the node where the Pod runs**. If it is not, the container fails to start (`CreateContainerError`/`Blocked`). That is the single most common pitfall, see below.

Useful runtime profile: **`RuntimeDefault`** — the container runtime's built-in profile (`cri-containerd.apparmor.d` for containerd, `docker-default` for Docker). It blocks mounts, writes to `/proc/sys`, access to `/sys/firmware`, ptrace of other profiles and similar. It costs nothing to enable and is the recommended minimum.

### Seccomp

**seccomp** (secure computing mode) lets a process install a **filter on syscalls** (BPF program). A syscall is the interface between user space and the kernel: `open`, `read`, `execve`, `mount`, `ptrace`, `unshare`, `keyctl`, `bpf`... Linux has 300+ syscalls; a normal web app uses perhaps 50-80. Every unused syscall is attack surface for kernel exploits.

For each syscall the filter can return an **action**:

| Action | Effect |
|---|---|
| `SCMP_ACT_ALLOW` | Syscall proceeds |
| `SCMP_ACT_ERRNO` | Syscall fails with an error (EPERM) — app sees "Operation not permitted" |
| `SCMP_ACT_KILL_PROCESS` / `SCMP_ACT_KILL` | Process is killed |
| `SCMP_ACT_LOG` | Allowed but logged to the kernel audit log (profile-learning mode) |
| `SCMP_ACT_TRACE` | Notify a ptrace tracer |

A profile has a **`defaultAction`** (what happens to anything not listed) and a list of **`syscalls`** entries that override it. Two styles:
- **Allow-list** (strict): `defaultAction: SCMP_ACT_ERRNO` + list the allowed syscalls. Safest, hardest to build.
- **Deny-list**: `defaultAction: SCMP_ACT_ALLOW` + list blocked syscalls. Easy but leaky.

Kubernetes `seccompProfile.type` values:

| Type | Meaning |
|---|---|
| `RuntimeDefault` | Use the runtime's default profile (containerd/Docker default blocks ~44 dangerous syscalls such as `mount`, `reboot`, `kexec_load`, `bpf`, `keyctl` while allowing the common ones). **Recommended baseline.** |
| `Unconfined` | No filter — what Kubernetes did **by default** for years (before `--seccomp-default`) |
| `Localhost` | Custom JSON profile stored on the node under the kubelet seccomp directory; `localhostProfile` gives the relative path |

Important: **Kubernetes does not apply seccomp by default**. Without configuration, Pods run `Unconfined` unless the kubelet runs with `--seccomp-default` (feature `SeccompDefault`, stable since v1.27), which turns `RuntimeDefault` on for **all** Pods lacking an explicit profile. The `restricted` PodSecurity profile (lesson 48) **requires** `RuntimeDefault` or `Localhost`, so this ties back to the previous lesson.

### AppArmor vs seccomp vs capabilities

| | Capabilities (lesson 32) | AppArmor | seccomp |
|---|---|---|---|
| Controls | Split-up root privileges | File paths, capabilities, network, mounts per program | Which syscalls may be called |
| Granularity | ~40 capabilities | Path/pattern rules | Syscall name + optionally arguments |
| Where defined | Pod spec | Profile file on node + Pod reference | JSON file on node (or runtime built-in) + Pod reference |
| Distro dependency | none | Needs AppArmor kernel module (Ubuntu/Debian/SUSE) | Any kernel with seccomp (nearly all) |
| Typical failure | Dropped capability needed by app | Profile not loaded on node | Blocked syscall makes app crash with EPERM |

They are **complementary** — use all three for sensitive workloads.

## Commands / Syntax Reference

### kube-bench

| Command | Purpose | Example |
|---|---|---|
| `kube-bench run` | Run all checks for auto-detected node type and version | `sudo kube-bench run` |
| `kube-bench run --targets` | Limit to node types: `master,node,etcd,controlplane,policies` | `sudo kube-bench run --targets master,etcd` |
| `kube-bench run --check` | Run specific checks | `sudo kube-bench run --targets master --check 1.2.1,1.2.2` |
| `kube-bench run --group` | Run a whole group | `sudo kube-bench run --targets node --group 4.2` |
| `kube-bench run --benchmark` | Force a benchmark | `sudo kube-bench run --benchmark cis-1.9` |
| `kube-bench run --version` | Force Kubernetes version | `sudo kube-bench run --version 1.30` |
| `kube-bench run --json` | Machine-readable output | `sudo kube-bench run --json > bench.json` |
| `kube-bench run --junit` / `--outputfile` | JUnit for CI, write to file | `sudo kube-bench run --junit --outputfile bench.xml` |
| `kube-bench run --noremediations` | Hide remediation text | `sudo kube-bench run --noremediations` |
| `kube-bench run --scored` / `--unscored` | Show only scored / only manual checks | `sudo kube-bench run --unscored=false` |
| `kube-bench run --config-dir` | Use custom `cfg/` directory | `sudo kube-bench run --config-dir /opt/kube-bench/cfg` |
| `kubectl apply -f job.yaml` | Run as Job in the cluster | `kubectl apply -f job-master.yaml` |
| `kubectl logs job/kube-bench-master` | Read Job output | `kubectl logs job/kube-bench-master` |

### Node inspection helpers used during remediation

| Command | Purpose | Example |
|---|---|---|
| `ps -ef \| grep kube-apiserver` | See effective flags of the running API server | `ps -ef \| grep [k]ube-apiserver \| tr ' ' '\n' \| grep '^--'` |
| `stat -c %a %U:%G <path>` | Show octal permissions and owner | `stat -c '%a %U:%G' /var/lib/etcd` |
| `chmod` / `chown` | Fix permissions | `sudo chmod 700 /var/lib/etcd` |
| `systemctl cat kubelet` | Show kubelet unit and drop-ins | `systemctl cat kubelet` |
| `sudo crictl ps` | List control-plane containers when the API is down | `sudo crictl ps \| grep apiserver` |
| `journalctl -u kubelet -f` | Watch kubelet recreate static pods | `sudo journalctl -u kubelet -f` |

### AppArmor

| Command | Purpose | Example |
|---|---|---|
| `aa-status` (`apparmor_status`) | List loaded profiles and their mode | `sudo aa-status` |
| `apparmor_parser -r <file>` | Load or reload a profile | `sudo apparmor_parser -r /etc/apparmor.d/k8s-deny-write` |
| `apparmor_parser -R <file>` | Remove a profile from the kernel | `sudo apparmor_parser -R /etc/apparmor.d/k8s-deny-write` |
| `aa-enforce <profile>` | Switch to enforce mode | `sudo aa-enforce /etc/apparmor.d/k8s-deny-write` |
| `aa-complain <profile>` | Switch to complain mode | `sudo aa-complain /etc/apparmor.d/k8s-deny-write` |
| `aa-disable <profile>` | Unload and disable | `sudo aa-disable /etc/apparmor.d/k8s-deny-write` |
| `aa-genprof <program>` | Interactive profile generation (scan logs while you run the program) | `sudo aa-genprof /usr/bin/myapp` |
| `aa-logprof` | Update a profile from logged denials | `sudo aa-logprof` |
| `aa-unconfined` | Show network-facing processes without a profile | `sudo aa-unconfined` |
| `cat /sys/module/apparmor/parameters/enabled` | Is AppArmor enabled in kernel? (`Y`) | `cat /sys/module/apparmor/parameters/enabled` |
| `cat /proc/<pid>/attr/current` | Profile applied to a process | `kubectl exec pod -- cat /proc/1/attr/current` |
| `journalctl -k \| grep apparmor` / `dmesg \| grep DENIED` | See denials | `sudo dmesg \| grep 'apparmor="DENIED"'` |

### Seccomp

| Command / field | Purpose | Example |
|---|---|---|
| `securityContext.seccompProfile.type` | `RuntimeDefault` / `Unconfined` / `Localhost` | `type: RuntimeDefault` |
| `securityContext.seccompProfile.localhostProfile` | Path relative to `/var/lib/kubelet/seccomp/` | `localhostProfile: profiles/audit.json` |
| `grep Seccomp /proc/1/status` | Seccomp mode of a process: 0 disabled, 1 strict, 2 filter | `kubectl exec pod -- grep Seccomp /proc/1/status` |
| `strace -c -f <cmd>` | Count syscalls an app uses | `strace -c -f -o /tmp/trace.txt nginx -t` |
| `strace -ff -o /tmp/t <cmd>` | Trace all syscalls to files | `strace -ff -o /tmp/t curl -s example.com` |
| `grep SECCOMP /var/log/syslog` | Kernel audit lines for `SCMP_ACT_LOG` hits | `sudo grep 'type=1326' /var/log/audit/audit.log` |
| `kubelet --seccomp-default` / `seccompDefault: true` | Make `RuntimeDefault` the default for all Pods | in `KubeletConfiguration` |

### Pod spec snippets

```yaml
# AppArmor, Kubernetes 1.30+ (GA field)
securityContext:                       # pod-level applies to all containers
  appArmorProfile:
    type: Localhost                    # RuntimeDefault | Localhost | Unconfined
    localhostProfile: k8s-deny-write   # only with Localhost; name of profile loaded on node
```

```yaml
# AppArmor legacy annotation (pre-1.30, still honoured on older clusters)
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/<container-name>: localhost/k8s-deny-write
    # other values: runtime/default | unconfined
```

```yaml
# Seccomp
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: profiles/block-chmod.json
```

## Examples / Walkthrough

### Part 1 — kube-bench

#### Example 1: Run kube-bench as a Job on the control-plane node

kube-bench publishes ready Job manifests per platform (`job.yaml`, `job-master.yaml`, `job-node.yaml`, `job-rke2.yaml`, ...). For kubeadm control plane:

```yaml
# job-master.yaml (abridged from the upstream manifest)
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-master
spec:
  template:
    spec:
      hostPID: true                       # needs to see host processes
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      restartPolicy: Never
      containers:
        - name: kube-bench
          image: docker.io/aquasec/kube-bench:v0.9.0   # pin a version
          command: ["kube-bench", "run", "--targets", "master"]
          volumeMounts:
            - { name: var-lib-etcd,     mountPath: /var/lib/etcd,     readOnly: true }
            - { name: var-lib-kubelet,  mountPath: /var/lib/kubelet,  readOnly: true }
            - { name: etc-systemd,      mountPath: /etc/systemd,      readOnly: true }
            - { name: etc-kubernetes,   mountPath: /etc/kubernetes,   readOnly: true }
            - { name: usr-bin,          mountPath: /usr/local/mount-from-host/bin, readOnly: true }
      volumes:
        - { name: var-lib-etcd,    hostPath: { path: /var/lib/etcd } }
        - { name: var-lib-kubelet, hostPath: { path: /var/lib/kubelet } }
        - { name: etc-systemd,     hostPath: { path: /etc/systemd } }
        - { name: etc-kubernetes,  hostPath: { path: /etc/kubernetes } }
        - { name: usr-bin,         hostPath: { path: /usr/bin } }
```

```bash
kubectl apply -f job-master.yaml
kubectl wait --for=condition=complete job/kube-bench-master --timeout=120s
kubectl logs job/kube-bench-master | tee master-before.txt
kubectl delete job kube-bench-master
```

Why `hostPID` and hostPath mounts: the Job is **breaking container isolation on purpose** to inspect the host. This is acceptable for an audit Job in a namespace with relaxed PodSecurity (it will be rejected under `baseline`/`restricted` enforcement from lesson 48 — label the namespace `pod-security.kubernetes.io/enforce=privileged` for the run, then delete the namespace).

#### Example 2: Run kube-bench as a binary on a node

```bash
# Download release and unpack (Ubuntu; check releases page for current version)
KB=0.9.0
curl -L -o kube-bench.tgz \
  https://github.com/aquasecurity/kube-bench/releases/download/v${KB}/kube-bench_${KB}_linux_amd64.tar.gz
sudo mkdir -p /opt/kube-bench
sudo tar -xzf kube-bench.tgz -C /opt/kube-bench       # contains kube-bench binary + cfg/
cd /opt/kube-bench

# Auto-detect version and node type
sudo ./kube-bench run | less -R

# Control plane + etcd only
sudo ./kube-bench run --targets master,etcd

# Worker node
sudo ./kube-bench run --targets node

# Also available via package (.deb/.rpm) or the container image
docker run --rm --pid=host -v /etc:/etc:ro -v /var:/var:ro \
  -v $(which kubectl):/usr/local/mount-from-host/bin/kubectl:ro \
  -t docker.io/aquasec/kube-bench:v0.9.0 run --targets master
```

Typical beginning of output:

```
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
...
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
...
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
[FAIL] 1.2.5 Ensure that the --kubelet-certificate-authority argument is set as appropriate (Automated)
```

#### Example 3: Narrow down, read remediations, export JSON

```bash
# One check, with its remediation
sudo ./kube-bench run --targets master --check 1.2.5

# Whole group
sudo ./kube-bench run --targets node --group 4.2

# JSON, then list only failures with jq
sudo ./kube-bench run --json > bench.json
jq -r '.Controls[].tests[].results[] | select(.status=="FAIL") | "\(.test_number)  \(.test_desc)"' bench.json

# Count summary
jq '.Totals' bench.json
```

The remediation text for a FAIL reads like this (shortened):

```
== Remediations master ==
1.2.5 Follow the Kubernetes documentation and setup the TLS connection between
the apiserver and kubelets. Then, edit the API server pod specification file
/etc/kubernetes/manifests/kube-apiserver.yaml on the control plane node and set the
--kubelet-certificate-authority parameter to the path to the cert file for the
certificate authority.
--kubelet-certificate-authority=<ca-string>
```

How to read it: **which file** to edit (`kube-apiserver.yaml`), **which flag** to set, **to what value**. kube-bench never tells you what the value is for your environment — you supply it.

#### Example 4: Remediate file ownership and permissions (etcd)

Failing check: `1.1.11 Ensure that the etcd data directory permissions are set to 700 or more restrictive` and `1.1.12 ...ownership is set to etcd:etcd`.

```bash
# Where is the data dir? Read it from the etcd static pod manifest
sudo grep data-dir /etc/kubernetes/manifests/etcd.yaml
#   - --data-dir=/var/lib/etcd

stat -c '%a %U:%G' /var/lib/etcd        # e.g. 755 root:root  -> FAIL

# Create a dedicated user if missing (kubeadm does not by default)
sudo useradd -r -c "etcd user" -s /sbin/nologin -M etcd

sudo chown -R etcd:etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
stat -c '%a %U:%G' /var/lib/etcd        # 700 etcd:etcd
```

Notes:
- Do this on the control-plane node holding etcd.
- On kubeadm clusters the etcd container runs as root by default, so changing ownership to a non-root user **without also making etcd run as that user** can prevent etcd from reading its data. kube-bench's own remediation says to create the user; whether etcd then starts depends on the setup. Test on a lab cluster first and keep an etcd snapshot (`etcdctl snapshot save`) before changing ownership in production.
- Permission-only checks (`chmod 700`) are safe; ownership checks are where care is needed.

Similar file checks and their safe fixes:

```bash
# 1.1.1 - 1.1.8: static pod manifests: 600, root:root
sudo chmod 600 /etc/kubernetes/manifests/*.yaml
sudo chown root:root /etc/kubernetes/manifests/*.yaml

# 1.1.13/14: admin.conf and super-admin.conf
sudo chmod 600 /etc/kubernetes/admin.conf /etc/kubernetes/super-admin.conf
sudo chown root:root /etc/kubernetes/admin.conf

# 1.1.19: PKI directory ownership; 1.1.20/21: cert 600 / key 600
sudo chown -R root:root /etc/kubernetes/pki/
sudo find /etc/kubernetes/pki -name '*.key' -exec chmod 600 {} \;
sudo find /etc/kubernetes/pki -name '*.crt' -exec chmod 644 {} \;
```

#### Example 5: Remediate API server flags (edit the static pod)

Goal: fix `1.2.1 --anonymous-auth=false` (anonymous requests), confirm `--authorization-mode` includes `Node,RBAC` and not `AlwaysAllow`, and set `--profiling=false` on the API server.

```bash
# 1. Safe copy outside the manifests directory
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak

# 2. See current flags
sudo grep -E -- '--(anonymous-auth|authorization-mode|profiling|insecure)' \
  /etc/kubernetes/manifests/kube-apiserver.yaml

# 3. Edit
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Within `spec.containers[0].command` make sure these lines exist:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC        # never AlwaysAllow
    - --anonymous-auth=false                # was default true
    - --profiling=false                     # CIS 1.2.15
    - --enable-admission-plugins=NodeRestriction
    # ... existing flags unchanged ...
```

```bash
# 4. Watch kubelet recreate the pod (API server is briefly unavailable)
sudo journalctl -u kubelet -f &                      # or:
watch -n2 'sudo crictl ps | grep kube-apiserver'

# 5. Verify
kubectl get nodes                                    # API answers again
ps -ef | grep [k]ube-apiserver | tr ' ' '\n' | grep -E 'anonymous-auth|profiling'
kubectl get pod -n kube-system -l component=kube-apiserver -o jsonpath='{.items[0].spec.containers[0].command}' | tr ',' '\n' | grep anonymous

# 6. Re-run the specific checks
sudo ./kube-bench run --targets master --check 1.2.1,1.2.15
```

Hazards:
- **`--anonymous-auth=false` and health probes.** The API server liveness/readiness probes in kubeadm hit `/livez` and `/readyz` — kubeadm manifests already allow them via `system:public-info-viewer`/anonymous binding in some versions. If a probe uses anonymous access and you disable it, the Pod can enter a restart loop. Check the manifest's `livenessProbe` after the change and use `crictl logs` if the API does not return.
- If the API server does not come back, **restore the backup**: `sudo cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml`. When `kubectl` is down, use `sudo crictl ps -a | grep apiserver` and `sudo crictl logs <id>` to read the error.
- Do **one** change at a time on a single control plane node.

#### Example 6: Controller-manager and scheduler

```bash
sudo cp /etc/kubernetes/manifests/kube-controller-manager.yaml /root/kcm.bak
sudo vim /etc/kubernetes/manifests/kube-controller-manager.yaml
#   - --profiling=false                      # CIS 1.3.2
#   - --use-service-account-credentials=true # CIS 1.3.3
#   - --terminated-pod-gc-threshold=12500    # CIS 1.3.1 (manual)

sudo vim /etc/kubernetes/manifests/kube-scheduler.yaml
#   - --profiling=false                      # CIS 1.4.1
```

Verify as above. Controller-manager and scheduler restart without any cluster-wide outage (leader election hands over on multi-control-plane clusters).

#### Example 7: Worker node — kubelet configuration

```bash
# On a worker (and control-plane nodes, which also run kubelet)
sudo ./kube-bench run --targets node
```

Frequent failures in section 4.2:

| Check | Setting | Safe value |
|---|---|---|
| 4.2.1 | `authentication.anonymous.enabled` | `false` |
| 4.2.2 | `authorization.mode` | `Webhook` (not `AlwaysAllow`) |
| 4.2.3 | `authentication.x509.clientCAFile` | path to cluster CA, e.g. `/etc/kubernetes/pki/ca.crt` |
| 4.2.4 | `readOnlyPort` | `0` (disable the unauthenticated 10255 port) |
| 4.2.5 | `streamingConnectionIdleTimeout` | not `0` (e.g. `5m`) |
| 4.2.6 | `protectKernelDefaults` | `true` |
| 4.2.9 | `eventRecordQPS` | `5` or higher / `0` |
| 4.2.12 | `rotateCertificates` + `serverTLSBootstrap` | `true` |

Find the kubelet config path:

```bash
systemctl cat kubelet | grep -E 'config|KUBELET'      # --config=/var/lib/kubelet/config.yaml
ps -ef | grep [k]ubelet | tr ' ' '\n' | grep '^--config'
```

Edit `/var/lib/kubelet/config.yaml` (a `KubeletConfiguration` object):

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false                       # 4.2.1
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt   # 4.2.3
authorization:
  mode: Webhook                          # 4.2.2
readOnlyPort: 0                          # 4.2.4
protectKernelDefaults: true              # 4.2.6 (needs sysctls set as kubelet expects!)
streamingConnectionIdleTimeout: 5m       # 4.2.5
seccompDefault: true                     # used later in this lesson
```

```bash
sudo cp /var/lib/kubelet/config.yaml /root/kubelet-config.bak
sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
sudo journalctl -u kubelet --since "2 minutes ago" | tail
kubectl get nodes                       # node returns Ready
sudo ./kube-bench run --targets node --group 4.2
```

Warning: `protectKernelDefaults: true` makes kubelet **refuse to start** if kernel sysctls (`vm.overcommit_memory=1`, `kernel.panic=10`, `kernel.panic_on_oops=1`, `vm.panic_on_oom=0`) differ from the kubelet's expected values. Set the sysctls first (lesson 9: `/etc/sysctl.d/`), then enable the option.

Also file checks for kubelet files:

```bash
# 4.1.1 / 4.1.2 kubelet service file
sudo chmod 600 /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf   # path varies, see: systemctl cat kubelet
sudo chown root:root /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# 4.1.5/4.1.6 kubelet.conf; 4.1.9/4.1.10 kubelet config.yaml
sudo chmod 600 /etc/kubernetes/kubelet.conf /var/lib/kubelet/config.yaml
sudo chown root:root /etc/kubernetes/kubelet.conf /var/lib/kubelet/config.yaml
```

Repeat for **every worker** (Ansible from lessons 24-29 is the right tool; students may write a playbook as a stretch exercise).

#### Example 8: Before/after comparison

```bash
sudo ./kube-bench run --targets master,node,etcd,policies --json > after.json
jq '.Totals' after.json
# compare with before.json
diff <(jq -r '.Controls[].tests[].results[]|"\(.test_number) \(.status)"' before.json) \
     <(jq -r '.Controls[].tests[].results[]|"\(.test_number) \(.status)"' after.json)
```

Keep both files — this is the evidence for an auditor.

### Part 2 — AppArmor

Requires a node OS with AppArmor (Ubuntu/Debian). The lab control-plane and worker nodes run Ubuntu.

#### Example 9: Check AppArmor and look at loaded profiles

```bash
cat /sys/module/apparmor/parameters/enabled       # Y
sudo aa-status
# apparmor module is loaded.
# 38 profiles are loaded.
# 36 profiles are in enforce mode.
#    cri-containerd.apparmor.d     <- containerd RuntimeDefault
#    /usr/sbin/tcpdump ...
# 2 profiles are in complain mode.
# 5 processes have profiles defined.
```

The container runtime also loads its own default profile (`cri-containerd.apparmor.d` for containerd, `docker-default` for Docker). Pods using `type: RuntimeDefault` use that.

#### Example 10: Write and load a profile that denies all writes

Create on **every worker node that may run the Pod**:

```bash
sudo tee /etc/apparmor.d/k8s-deny-write > /dev/null <<'EOF'
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,                      # allow file access in general...
  deny /** w,                # ...but deny writing anywhere
}
EOF

sudo apparmor_parser -r /etc/apparmor.d/k8s-deny-write
sudo aa-status | grep k8s-deny-write
#    k8s-deny-write
```

Why `attach_disconnected`: container filesystems are in a separate mount namespace, so paths look "disconnected" from host root; without the flag many legitimate accesses are denied.

Profile syntax summary:

| Rule | Meaning |
|---|---|
| `file,` | allow all file access (then narrowed by later `deny` rules; deny always wins) |
| `/etc/passwd r,` | read-only access |
| `/data/** rw,` | read/write recursively |
| `deny /proc/sys/** w,` | explicit deny (also silences log noise with `audit deny` vs `deny`) |
| `capability net_bind_service,` | allow one capability |
| `deny capability sys_admin,` | deny a capability |
| `network inet tcp,` | allow TCP over IPv4 |
| `deny network raw,` | deny raw sockets |
| `#include <abstractions/base>` | common allowances (libc, locale, /dev/null) |

Copy the profile to all nodes (AppArmor profiles are **not** distributed by Kubernetes):

```bash
for n in worker1 worker2; do
  scp /etc/apparmor.d/k8s-deny-write $n:/tmp/
  ssh $n 'sudo mv /tmp/k8s-deny-write /etc/apparmor.d/ && sudo apparmor_parser -r /etc/apparmor.d/k8s-deny-write'
done
```

In production, ship profiles with Ansible, an image-baked config, or a DaemonSet-based loader such as the Security Profiles Operator, so new nodes automatically get them.

#### Example 11: Attach profile to a Pod (Kubernetes 1.30+)

```yaml
# pod-apparmor.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-deny-write
  containers:
  - name: hello
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Hello AppArmor!' && sleep 1h"]
```

```bash
kubectl apply -f pod-apparmor.yaml
kubectl get pod hello-apparmor -o wide          # note the node
kubectl exec hello-apparmor -- cat /proc/1/attr/current
# k8s-deny-write (enforce)

kubectl exec hello-apparmor -- touch /tmp/test
# touch: /tmp/test: Permission denied
kubectl exec hello-apparmor -- sh -c 'echo hi > /etc/x'
# sh: can't create /etc/x: Permission denied
kubectl exec hello-apparmor -- cat /etc/hostname     # reads still work
```

On the node, the denial is logged:

```bash
sudo dmesg | grep 'apparmor="DENIED"' | tail -3
# apparmor="DENIED" operation="mknod" profile="k8s-deny-write" name="/tmp/test" ... requested_mask="c" denied_mask="c"
```

Per-container instead of per-Pod: put `securityContext.appArmorProfile` inside the container entry. A container-level setting overrides the pod-level one.

#### Example 12: Legacy annotation method (clusters older than 1.30)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor-legacy
  annotations:
    # key: container.apparmor.security.beta.kubernetes.io/<container name>
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-deny-write
spec:
  containers:
  - name: hello
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1h"]
```

Values: `runtime/default`, `localhost/<profile-name>`, `unconfined`. The annotation key must contain the **exact container name** — a typo silently applies nothing. On 1.30+ both can coexist but the field takes precedence; the annotation is deprecated. Always prefer the field on current clusters.

#### Example 13: What happens when the profile is not loaded

```yaml
  securityContext:
    appArmorProfile: { type: Localhost, localhostProfile: does-not-exist }
```

```bash
kubectl get pod
# NAME   READY   STATUS            RESTARTS
# x      0/1     Blocked           0           (or CreateContainerError on some versions)
kubectl describe pod x | tail
# Warning  Failed   ... cannot find AppArmor profile 'does-not-exist' on node worker1
```

Same symptom appears when the profile exists on worker1 but the scheduler places the Pod on worker2. Quick workaround while learning: `nodeSelector` (lesson 46) to pin to nodes that carry the profile. Real fix: distribute to all nodes.

#### Example 14: Complain mode and generating a profile

Develop a profile for a real program without breaking it:

```bash
sudo apt install -y apparmor-utils apparmor-profiles
sudo aa-genprof /usr/bin/myapp          # run the app in another terminal, then press S (scan) and F (finish)
# or start loose and tighten:
sudo aa-complain /etc/apparmor.d/usr.bin.myapp
# ... exercise the application ...
sudo aa-logprof                          # walks through logged denials: Allow / Deny / Glob
sudo aa-enforce /etc/apparmor.d/usr.bin.myapp
```

Order of work: **complain → exercise all code paths → logprof → enforce → test again**. A profile built from a short test run will block rare code paths (log rotation, startup migration) in production.

### Part 3 — Seccomp

#### Example 15: Check seccomp state in a container

```bash
kubectl run sc-test --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec sc-test -- grep Seccomp /proc/1/status
# Seccomp:        0           <- disabled (Unconfined)  [0 disabled, 1 strict, 2 filter]
# Seccomp_filters: 0
kubectl delete pod sc-test --now
```

Seccomp `0` on a default Pod is the reality on many clusters: containers can call almost every syscall.

#### Example 16: RuntimeDefault

```yaml
# pod-seccomp-runtimedefault.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-default
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: c
    image: busybox:1.36
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-seccomp-runtimedefault.yaml
kubectl exec sc-default -- grep Seccomp /proc/1/status
# Seccomp:        2           <- filter mode
# Seccomp_filters: 1

# A blocked syscall: unshare(CLONE_NEWUSER) is denied by the default profile
kubectl exec sc-default -- unshare -U true
# unshare: unshare(0x10000000): Operation not permitted
```

Enable it **for every Pod** at node level (cluster-wide default):

```yaml
# /var/lib/kubelet/config.yaml
seccompDefault: true
```

```bash
sudo systemctl restart kubelet     # new containers get RuntimeDefault when no profile is set
```

Existing Pods are not changed; recreate them. Occasionally apps break under the default (some profilers, debuggers, apps using `ptrace`, `perf_event_open`): opt them out with explicit `type: Unconfined` after review, and track the exception.

#### Example 17: Custom profile — placing the file

Seccomp profiles live on the node under the kubelet root: `/var/lib/kubelet/seccomp/`. `localhostProfile` is the path **relative to that directory**.

```bash
sudo mkdir -p /var/lib/kubelet/seccomp/profiles
```

**Audit profile** — allows everything, logs every syscall. Use it to learn what an app does:

```bash
sudo tee /var/lib/kubelet/seccomp/profiles/audit.json > /dev/null <<'EOF'
{
  "defaultAction": "SCMP_ACT_LOG"
}
EOF
```

**Deny-list profile** — everything allowed except `chmod`-family syscalls, which fail with EPERM:

```bash
sudo tee /var/lib/kubelet/seccomp/profiles/block-chmod.json > /dev/null <<'EOF'
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    {
      "names": ["chmod", "fchmod", "fchmodat", "fchmodat2"],
      "action": "SCMP_ACT_ERRNO",
      "errnoRet": 1
    }
  ]
}
EOF
```

JSON fields:

| Field | Meaning |
|---|---|
| `defaultAction` | Action for syscalls not matched by any rule |
| `architectures` | CPU architectures the rules apply to (include the ones your nodes run) |
| `syscalls[].names` | List of syscall names |
| `syscalls[].action` | `SCMP_ACT_ALLOW`, `SCMP_ACT_ERRNO`, `SCMP_ACT_KILL_PROCESS`, `SCMP_ACT_LOG`... |
| `syscalls[].errnoRet` | errno number returned for `SCMP_ACT_ERRNO` (1 = EPERM, 13 = EACCES, 38 = ENOSYS) |
| `syscalls[].args` | Optionally match on syscall arguments |

Copy the file to **every node** (same distribution problem as AppArmor), keep the same relative path.

#### Example 18: Use the Localhost profile

```yaml
# pod-seccomp-block-chmod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-chmod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/block-chmod.json
  containers:
  - name: c
    image: busybox:1.36
    command: ["sh", "-c", "touch /tmp/f && sleep 3600"]
```

```bash
kubectl apply -f pod-seccomp-block-chmod.yaml
kubectl exec sc-chmod -- touch /tmp/g                 # works (touch is not blocked)
kubectl exec sc-chmod -- chmod 600 /tmp/g
# chmod: /tmp/g: Operation not permitted
kubectl exec sc-chmod -- grep Seccomp /proc/1/status  # Seccomp: 2
```

If the file is missing on the node: Pod status `CreateContainerError`, event: `cannot load seccomp profile "/var/lib/kubelet/seccomp/profiles/block-chmod.json": no such file or directory`.

#### Example 19: Discover which syscalls an app needs

Method A — **strace** on any Linux host or in a debug container:

```bash
# Summary table of syscalls used
strace -c -f -o /tmp/summary.txt curl -s https://example.com > /dev/null
cat /tmp/summary.txt | head -20
# % time     seconds  usecs/call     calls    errors syscall
#  ... connect, socket, read, write, openat, mmap, futex ...

# Just the names, unique
strace -f -o /tmp/trace.txt curl -s https://example.com > /dev/null
awk -F'(' '{print $1}' /tmp/trace.txt | awk '{print $NF}' | sort -u
```

Method B — **audit profile on the cluster**: run the Pod with `audit.json` (`SCMP_ACT_LOG`), exercise the app, then read kernel audit records on the node:

```bash
sudo grep -i seccomp /var/log/syslog | tail           # Ubuntu with rsyslog
sudo journalctl -k | grep -i 'type=1326' | tail       # audit seccomp lines
# audit: type=1326 audit(...): ... comm="nginx" ... syscall=257 compat=0 ... code=0x7ffc0000

# syscall numbers -> names (x86_64)
ausyscall 257                  # openat   (package: auditd)
```

Method C — tools such as the **Security Profiles Operator** (records profiles via eBPF/log) or **oci-seccomp-bpf-hook**; mention only, not needed for this course.

Turn the list of names into an allow-list profile:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read","write","openat","close","fstat","mmap","munmap","brk",
                "rt_sigaction","rt_sigprocmask","futex","epoll_wait","epoll_ctl",
                "accept4","socket","bind","listen","connect","getpid","exit_group"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

The first try always misses something (the dynamic linker, `getrandom`, `clone3`...). Iterate: switch `defaultAction` to `SCMP_ACT_LOG`, run the full test, add what shows up, then switch back to `SCMP_ACT_ERRNO`. An app killed by `SCMP_ACT_KILL_PROCESS` shows as `exit code 159 (SIGSYS)`.

Docker's published default profile (`moby/profiles/seccomp/default.json`) is a good base: copy it and remove entries you do not need.

#### Example 20: Combine layers in one hardened Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
    appArmorProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:1.27
    ports:
    - containerPort: 8080
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - { name: tmp,   mountPath: /tmp }
    - { name: cache, mountPath: /var/cache/nginx }
  volumes:
  - { name: tmp,   emptyDir: {} }
  - { name: cache, emptyDir: {} }
```

This Pod satisfies the **restricted** PodSecurity profile (lesson 48) and adds AppArmor. Verify:

```bash
kubectl apply -f hardened.yaml
kubectl exec hardened -- id
kubectl exec hardened -- grep Seccomp /proc/1/status
kubectl exec hardened -- cat /proc/1/attr/current        # cri-containerd.apparmor.d (enforce)
kubectl exec hardened -- sh -c 'touch /etc/x'            # read-only filesystem
```

### Part 4 — Tying it back

After hardening, run kube-bench `--targets policies` (section 5) to see which Kubernetes policy checks (default ServiceAccount usage, NetworkPolicy per namespace, `cluster-admin` bindings) still fail. Those map to lessons 47-48 and are fixed in the next lessons' practice (lesson 51).

Image vulnerability scanning, API audit logging and policy-as-code admission (Kyverno) complete the security set in lesson 50.

## Common Pitfalls

- **Running kube-bench from a laptop or from a worker for control-plane targets** — it audits the node it runs on. Control-plane checks on a worker show "file not found" failures that are noise. Run `--targets master,etcd` only on control-plane nodes, `--targets node` on workers.
- **Wrong benchmark profile for the distro** — RKE2, EKS, GKE and AKS have their own benchmark IDs; using generic `cis-1.9` on RKE2 reports false failures because paths differ (`/var/lib/rancher/rke2/...`). Use `--benchmark rke2-cis-1.9` or let auto-detect work.
- **Treating a green score as proof of security** — kube-bench checks configuration, not your Pods, images or RBAC contents. WARN (manual) items are skipped by scoring but often matter.
- **Editing a static pod and losing the API server** — a typo or unsupported flag leaves `kube-apiserver` in CrashLoop and `kubectl` dead. Back up first, make one change at a time, and know `crictl ps -a` / `crictl logs` to debug without the API.
- **Backup file left inside `/etc/kubernetes/manifests/`** — a copy named `kube-apiserver.yaml.bak` is ignored, but `kube-apiserver-old.yaml` is a second API server manifest and starts a duplicate Pod with a port conflict. Keep backups elsewhere (e.g. `/root`).
- **Changing etcd directory ownership to a user etcd does not run as** — etcd cannot read its data and the control plane dies. Snapshot first; align the container `runAsUser` or only apply the permission (mode 700) check.
- **`protectKernelDefaults: true` without matching sysctls** — kubelet exits at start with "invalid kernel flag". Configure sysctls first.
- **Fixing one worker only** — kubelet checks must pass on **every** node; script the change (Ansible).
- **AppArmor profile not loaded on every node** — Pod scheduled to a node without the profile fails with `Blocked`/`CreateContainerError`. Distribute profiles to all nodes, or pin Pods with `nodeSelector`/affinity (lesson 46).
- **AppArmor on a non-AppArmor OS** — RHEL, Rocky, Amazon Linux, Fedora use SELinux; `/sys/module/apparmor/parameters/enabled` is missing. Kubernetes accepts the field but the container fails (for `Localhost`) or runs unconfined. Check the OS before relying on it.
- **Annotation key with the wrong container name** — legacy annotations are keyed by container name; a mismatch silently applies no profile.
- **Using the legacy annotation on 1.30+ and the field at once** — both set; conflicting values are rejected or the field wins. Pick the field.
- **Profile only in `complain` mode left in production** — violations are logged but not blocked. Always check `aa-status` shows `enforce`.
- **Assuming Kubernetes enables seccomp by default** — it does not. Pods run `Unconfined` unless you set `RuntimeDefault` per Pod or `seccompDefault: true` on the kubelet.
- **Custom seccomp allow-list built from a short test** — rare code paths call unlisted syscalls and the app fails only under load or on startup edge cases. Use `SCMP_ACT_LOG` first, test fully, then switch to `ERRNO`.
- **Profile path wrong** — `localhostProfile` is relative to `/var/lib/kubelet/seccomp/`; do not put the leading `/var/lib/kubelet/seccomp/` in it. With rke2/other distros the kubelet root differs (`--root-dir`).
- **Seccomp profile present only on one node** — same distribution problem as AppArmor; Pod fails on nodes without the file.
- **Blocking syscalls the container runtime needs** — a profile applies to the container's processes only, but blocking `clone3`, `futex` or `epoll_*` crashes modern runtimes (Go, JVM, Node). Start from the runtime default profile and remove entries, not from empty.
- **Security tools running as privileged with hostPID in a namespace that enforces PodSecurity** — the kube-bench Job is rejected under `baseline`/`restricted`; use a dedicated namespace labelled `privileged`.

## FAQ

**Q: What is kube-bench and does it change my cluster?**
A: kube-bench is an open-source tool that checks a node against the CIS Kubernetes Benchmark. It is read-only: it reports PASS/FAIL/WARN/INFO and prints remediation steps, but never changes any configuration. You apply the fixes yourself.

**Q: What is the CIS Benchmark?**
A: A consensus checklist from the Center for Internet Security of secure settings for Kubernetes control plane, etcd, worker nodes and policies. Each item has a number (for example 1.2.1), a rationale, an audit and a remediation.

**Q: Where do I run kube-bench?**
A: On the node you want to audit — as a binary over SSH, or as a Job with `hostPID: true` and hostPath mounts scheduled to that node. Use `--targets master,etcd` on control-plane nodes and `--targets node` on workers.

**Q: What is the difference between FAIL and WARN?**
A: FAIL means an automated check ran and found a non-compliant setting. WARN means a manual check that a human must evaluate (or the tool could not decide). INFO is purely informational.

**Q: How do I run only one check?**
A: `kube-bench run --targets master --check 1.2.1` for one check, comma-separated for several, or `--group 4.2` for a group.

**Q: How do I use kube-bench output in CI or dashboards?**
A: `kube-bench run --json > result.json` (or `--junit --outputfile`) and parse with `jq`. For example `jq '.Totals' result.json` gives the counts.

**Q: The kube-bench Job says "Pod violates PodSecurity". What now?**
A: The Job uses hostPID and hostPath, which `baseline` and `restricted` forbid. Run it in a namespace labelled `pod-security.kubernetes.io/enforce=privileged`, or run the binary on the node directly.

**Q: Where are the control-plane flags configured?**
A: On kubeadm clusters in static Pod manifests in `/etc/kubernetes/manifests/` (kube-apiserver.yaml, kube-controller-manager.yaml, kube-scheduler.yaml, etcd.yaml). Edit the file and kubelet recreates the Pod. On RKE2 use `/etc/rancher/rke2/config.yaml` (for example `profile: cis`) instead of editing generated manifests.

**Q: How do I know that my edit was applied?**
A: Check the running command line (`ps -ef | grep kube-apiserver` or `crictl ps` then `crictl inspect`), the Pod spec (`kubectl get pod -n kube-system <pod> -o yaml`), and rerun the same kube-bench check.

**Q: The API server does not come back after my edit. How do I recover?**
A: `kubectl` is down, so use the node: `sudo crictl ps -a | grep apiserver`, `sudo crictl logs <container-id>` to see the flag error, restore the backup with `cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml`, and watch `journalctl -u kubelet`.

**Q: Why does CIS want `--anonymous-auth=false` on the API server and kubelet?**
A: Anonymous requests are treated as user `system:anonymous`. If any RBAC binding or a permissive kubelet authorization mode allows it, anyone who can reach the port can read data or run commands. Disabling anonymous authentication removes that entry point.

**Q: What is the kubelet read-only port?**
A: Port 10255, an unauthenticated HTTP endpoint exposing Pod and node information. CIS requires it disabled (`readOnlyPort: 0`). Old metrics collectors used it; modern ones use the authenticated port 10250.

**Q: What is a Linux Security Module (LSM)?**
A: A kernel framework for mandatory access control. AppArmor, SELinux, Smack and others are LSMs. They restrict what even root processes may do, according to policy.

**Q: AppArmor or SELinux?**
A: They solve the same problem differently. AppArmor (path-based profiles) is the default on Ubuntu/Debian/SUSE; SELinux (label-based) on RHEL/Fedora/Rocky. Kubernetes supports AppArmor natively through `appArmorProfile`; SELinux through `seLinuxOptions`. Use the one your node OS ships.

**Q: What are the AppArmor profile modes?**
A: enforce (block and log), complain (allow but log, used to build a profile), unconfined (no profile).

**Q: How do I see which AppArmor profiles are loaded?**
A: `sudo aa-status`. Also `/sys/kernel/security/apparmor/profiles`.

**Q: How do I load a profile?**
A: `sudo apparmor_parser -r /etc/apparmor.d/<file>` (add or reload). `-R` removes it. Loading survives until reboot unless the file is in `/etc/apparmor.d/` (boot loads those automatically).

**Q: How do I attach AppArmor to a container?**
A: On Kubernetes 1.30+ set `securityContext.appArmorProfile` (`type: RuntimeDefault`, `Localhost` with `localhostProfile: <name>`, or `Unconfined`) at Pod or container level. On older clusters use the annotation `container.apparmor.security.beta.kubernetes.io/<container>: localhost/<profile>`.

**Q: My Pod is stuck with "Blocked" or CreateContainerError after setting an AppArmor profile.**
A: The profile is not loaded on the node where the Pod was scheduled. Run `aa-status` on that node, load the profile there, or pin the Pod to a node that has it. Make profile distribution part of node provisioning.

**Q: Does Kubernetes copy AppArmor or seccomp profiles to nodes?**
A: No. You must place AppArmor profiles into the node kernel and seccomp JSON files under `/var/lib/kubelet/seccomp/` yourself — with Ansible, node images, or the Security Profiles Operator.

**Q: What is seccomp?**
A: A Linux kernel feature that filters syscalls made by a process. A profile says which syscalls are allowed, return an error, kill the process, or are logged. Fewer reachable syscalls means fewer ways to exploit kernel bugs.

**Q: What does `RuntimeDefault` do?**
A: It applies the container runtime's built-in seccomp profile, which blocks dozens of risky syscalls (for example `mount`, `reboot`, `kexec_load`, `bpf`, `keyctl`) while allowing what ordinary applications use. It is the recommended baseline and is required by the `restricted` PodSecurity profile.

**Q: Is seccomp enabled by default in Kubernetes?**
A: No. Pods run `Unconfined` unless you set a profile, or enable `seccompDefault: true` in the kubelet configuration (the `--seccomp-default` flag), which makes `RuntimeDefault` the default for Pods that do not set one.

**Q: Where does a custom seccomp profile go and how do I refer to it?**
A: Save the JSON on every node under `/var/lib/kubelet/seccomp/` (for example `profiles/block-chmod.json`) and set `seccompProfile: {type: Localhost, localhostProfile: profiles/block-chmod.json}`. The path is relative to the seccomp directory.

**Q: How do I know which syscalls my app needs?**
A: Run it under `strace -c -f` for a quick list, or run it under an audit profile (`defaultAction: SCMP_ACT_LOG`) and read the kernel audit records, then build an allow-list from the names. Test every code path; iterate.

**Q: What error do I see when seccomp blocks a syscall?**
A: With `SCMP_ACT_ERRNO` the application sees "Operation not permitted" (EPERM) or whatever errno you chose. With `SCMP_ACT_KILL_PROCESS` the process dies with SIGSYS (exit code 159).

**Q: How can I verify a container is under seccomp?**
A: `kubectl exec <pod> -- grep Seccomp /proc/1/status`. `Seccomp: 2` means a filter is active; `0` means none.

**Q: How can I verify AppArmor on a container?**
A: `kubectl exec <pod> -- cat /proc/1/attr/current` prints the profile name and mode, for example `k8s-deny-write (enforce)`. Denials appear in `dmesg` or `journalctl -k` on the node.

**Q: Do AppArmor and seccomp replace `securityContext` capabilities and non-root?**
A: No. They are additional layers. Use non-root user, dropped capabilities, no privilege escalation, read-only root filesystem, seccomp and AppArmor together.

**Q: Does this work on managed clusters (EKS, GKE, AKS)?**
A: kube-bench has dedicated benchmarks for them (`eks-1.5.0`, `gke-1.6.0`, `aks-1.7`), but control-plane checks do not apply because the provider owns the control plane. Seccomp `RuntimeDefault` works everywhere; AppArmor depends on the node OS image (Bottlerocket and Amazon Linux use SELinux or none).

## Practice / Exercise

### Core

1. **Baseline audit.** On the lab control-plane node download kube-bench and run `sudo ./kube-bench run --targets master,etcd --json > before-master.json`. On a worker run `--targets node` into `before-node.json`. Record the totals (`jq '.Totals'`).
2. **Run as a Job.** Create namespace `bench` labelled `pod-security.kubernetes.io/enforce=privileged`, apply `job-master.yaml` there, read `kubectl logs`, delete the Job and the namespace.
3. **Fix permissions.** Choose three FAIL checks from section 1.1 or 4.1 (file permissions/ownership), fix them, and confirm they flip to PASS with `--check <id>`.
4. **Fix an API server flag.** Back up `kube-apiserver.yaml`, set `--profiling=false` (check 1.2.15), wait for the Pod to restart, verify the flag in `ps` and in the Pod YAML, and re-run the check.
5. **Fix a kubelet setting.** On a worker set `readOnlyPort: 0` and `authentication.anonymous.enabled: false` in `/var/lib/kubelet/config.yaml`, restart kubelet, confirm the node is `Ready` and checks 4.2.1 and 4.2.4 pass.
6. **AppArmor deny-write.** Create the `k8s-deny-write` profile on all workers, load it, run the `hello-apparmor` Pod, and show that `touch /tmp/test` fails and `cat /proc/1/attr/current` reports the profile in enforce mode.
7. **Break it on purpose.** Reference a non-existing AppArmor profile name and capture the Pod status and `kubectl describe` event. Fix by loading the profile.
8. **Seccomp.** Show `Seccomp: 0` on a default Pod and `Seccomp: 2` with `RuntimeDefault`. Create `block-chmod.json` under `/var/lib/kubelet/seccomp/profiles/` on the worker, run the `sc-chmod` Pod, and show that `chmod` fails while `touch` works.

### Stretch

1. Write an **Ansible playbook** (lessons 24-29) that distributes an AppArmor profile and a seccomp JSON file to all workers, loads the profile with `apparmor_parser -r`, and fails if `aa-status` does not list it.
2. Enable `seccompDefault: true` on one worker's kubelet, redeploy a Pod without any seccomp setting scheduled there, and confirm `Seccomp: 2`. Note which of your workloads (if any) break.
3. Build an **allow-list seccomp profile** for a simple nginx (or Go) container: start with `SCMP_ACT_LOG`, collect syscalls from the audit log, create `SCMP_ACT_ERRNO` profile, and run until the app serves requests under load with no EPERM.
4. Use `aa-genprof` or complain mode to create a real profile for a small program on a node (not Kubernetes first), move it to enforce, then attach it to a Pod via `Localhost`.
5. Produce an **audit report**: compare `before.json` and `after.json`, list each check fixed, each accepted exception with reason, and each WARN item reviewed by hand. Keep it as the template for compliance evidence.
6. Run `kube-bench run --targets policies` and list which section 5 checks fail; map each to the lesson (RBAC 47, NetworkPolicy 48, PodSecurity 48) that fixes it.

## Further Reading

- kube-bench repository and docs: https://github.com/aquasecurity/kube-bench
- CIS Kubernetes Benchmark (free PDF after registration): https://www.cisecurity.org/benchmark/kubernetes
- Kubernetes: Restrict a Container's Access to Resources with AppArmor: https://kubernetes.io/docs/tutorials/security/apparmor/
- Kubernetes: Restrict a Container's Syscalls with seccomp: https://kubernetes.io/docs/tutorials/security/seccomp/
- Kubernetes: Configure a Security Context for a Pod or Container: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
- Kubernetes: Pod Security Standards (restricted profile requirements): https://kubernetes.io/docs/concepts/security/pod-security-standards/
- AppArmor documentation (Ubuntu): https://ubuntu.com/server/docs/apparmor
- Docker default seccomp profile: https://github.com/moby/profiles/blob/main/seccomp/default.json
- Security Profiles Operator: https://github.com/kubernetes-sigs/security-profiles-operator
- RKE2 CIS hardening guide: https://docs.rke2.io/security/hardening_guide

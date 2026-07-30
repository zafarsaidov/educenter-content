🐧 Linux Basics
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1: DevOps Concept. Introduction to Linux & Essential Commands
  • What is DevOps: culture, CI/CD lifecycle, toolchain overview
  • Linux distributions overview, why Ubuntu for DevOps
  • Terminal basics: pwd, ls, cd, mkdir, rmdir, touch
  • File operations: cp, mv, rm, cat, echo, less, head, tail
  • Getting help: man, --help, whatis, apropos

2: Navigating the Filesystem – Absolute/Relative Paths, Inodes & Directory Structure
  • Linux directory hierarchy: /, /etc, /var, /home, /usr, /tmp, /proc, /dev
  • Absolute vs relative paths: ., .., ~, cd -, pwd
  • Inodes: concept, stat, ls -i, inode limits, why they matter
  • Hard links vs soft links: ln, ln -s, readlink, differences
  • ls options: -la, -lh, -R; tree; hidden files (.dotfiles)

3: File Permissions, Ownership & Text Editors (vim, nano)
  • Permission model: user/group/other, rwx, ls -l output interpretation
  • chmod: numeric (755, 644) and symbolic (+x, g-w, o=r) modes
  • chown, chgrp: changing ownership; recursive (-R); chown user:group
  • Special permissions: SUID, SGID, sticky bit; umask
  • vim: normal/insert/visual modes, navigation, :wq :q! dd yy p /search
  • nano: open, edit, Ctrl+O save, Ctrl+X exit

4: File Search, Text Filtering & awk
  • find: -name, -type (f/d/l), -size, -mtime, -user, -exec
  • grep: -r, -i, -n, -v, -l, -c, -E; basic regex in grep
  • Piping and redirection: |, > (overwrite), >> (append), 2> (stderr), < (stdin), /dev/null, tee; xargs
  • awk basics: fields ($1, $2, $NF), NR, FS, print, printf
  • awk patterns and conditions: /pattern/ matching, if/else, practical one-liners

5: Managing System Services with systemd
  • systemd concepts: units (.service, .socket, .timer), targets, dependency tree
  • systemctl: start, stop, restart, reload, enable, disable, status, is-active, daemon-reload
  • journalctl: -u, -f, --since, --until, -n, -p (priority levels)
  • Writing a custom .service unit file: [Unit], [Service], [Install] sections
  • systemd timers: .timer unit as cron alternative; systemctl list-timers

6: User and Group Management
  • /etc/passwd, /etc/shadow, /etc/group: file structure and fields
  • useradd (-m, -s, -G, -d), usermod (-aG, -s, -l), userdel (-r)
  • groupadd, groupmod, groupdel; managing secondary groups
  • passwd, chage: password aging, expiry policies, -l (list)
  • sudo: /etc/sudoers, visudo, NOPASSWD, /etc/sudoers.d/ drop-in files
  • newgrp: temporary login to a group without re-login; id, groups commands

7: Linux Package Management
  • APT ecosystem: /etc/apt/sources.list, PPAs (add-apt-repository), GPG keys
  • apt: update, upgrade, full-upgrade, install, remove, purge, autoremove
  • apt: search, show, list --installed, --upgradable; apt-cache
  • dpkg: -i (install .deb), -r, -l, -s, -L (list installed files)
  • snap: install, list, remove, refresh, channels (stable/edge/beta)

8: Linux Process and Job Management
  • Process concepts: PID, PPID, states (R/S/D/Z/T), /proc filesystem overview
  • ps: aux, -ef, -o; pgrep, pstree
  • top/htop: interface, sorting, kill interactively, load average meaning
  • Signals: kill -l, kill/pkill/killall with SIGTERM (15), SIGKILL (9), SIGHUP (1)
  • Job control: &, jobs, fg, bg, nohup; nice and renice (CPU priority)

9: Scheduled Tasks & Kernel Parameters
  • cron syntax: 5 fields (min hour day month weekday), special strings (@reboot, @daily)
  • crontab: -e, -l, -r, -u; /etc/cron.d, /etc/cron.daily/weekly/monthly
  • at, atd: scheduling one-time tasks; atq, atrm
  • Kernel parameters: /proc/sys/, /etc/sysctl.conf, sysctl -w/-p; common params (ip_forward, swappiness, file-max)

10: Linux Networking Essentials – TCP/IP, Ports & Tools
  • Network interfaces: ip addr show, ip link set up/down, ip addr add/del
  • Routing: ip route show, ip route add, default gateway, ip neigh
  • Ports & sockets: TCP/UDP port ranges (well-known 0-1023, registered, ephemeral 49152+); common ports: 22/80/443/3306/5432/6379; ss -tuln, lsof -i
  • Connectivity tools: ping (-c, -i), traceroute/mtr, curl, wget
  • DNS tools: dig (A, MX, NS, CNAME, PTR), nslookup; /etc/hosts, /etc/resolv.conf
  • Ubuntu network config: /etc/netplan/ structure, netplan apply, netplan try

11: iptables, ufw & SSH
  • iptables concepts: tables (filter/nat/mangle), chains (INPUT/OUTPUT/FORWARD), ACCEPT/DROP/REJECT
  • iptables rules: -A, -D, -I, -L --line-numbers, -F; allow/block port; iptables-save/restore
  • ufw: enable/disable, allow/deny (port, service, from IP), delete rules, status numbered
  • SSH basics: ssh, ssh-keygen (ed25519), ssh-copy-id, authorized_keys, known_hosts
  • SSH config file: ~/.ssh/config (Host, HostName, User, IdentityFile, Port); scp
  • SSH port forwarding: local (-L localport:host:remoteport), remote (-R remoteport:host:localport)
  • SSH hardening: /etc/ssh/sshd_config (PermitRootLogin no, PasswordAuthentication no, Port)

12: Disk Partitioning, File Systems & Disk Monitoring
  • Disk concepts: block devices (/dev/sda, /dev/vda), MBR vs GPT partition tables
  • cfdisk: create/delete partition, write; parted overview
  • File systems: mkfs.ext4, mkfs.xfs; mount, umount; mount options
  • /etc/fstab: fields (device, mountpoint, fstype, options, dump, pass), UUID
  • Disk monitoring: df -h, du -sh *, lsblk, blkid; iostat -x, iotop
  • Swap: mkswap, swapon/swapoff, /etc/fstab swap entry, free -h, swapon --show


🌐 Web Servers & Load Balancing
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

13: nginx – Web Server, Reverse Proxy & Load Balancing
  • nginx overview: event-driven architecture, config structure (/etc/nginx/nginx.conf, sites-available/enabled), nginx -t, systemctl reload
  • Server blocks: listen, server_name, root, index, location blocks, try_files, custom error pages
  • Reverse proxy: proxy_pass, proxy_set_header (Host, X-Real-IP, X-Forwarded-For), proxy_read_timeout, proxy_connect_timeout
  • Load balancing: upstream block with multiple backends; methods: round-robin (default), ip_hash, least_conn; weight parameter
  • HTTPS with certbot: install certbot, certbot --nginx, certificate auto-renewal (systemd timer), HTTP→HTTPS redirect
  • Stream module: TCP/UDP proxying (stream block, upstream, server); use cases: MySQL proxy, TCP load balancer


🌀 Git Version Control
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

14: Git Fundamentals and Workflow
  • Git concepts: VCS, distributed model, repository, staging area, working tree
  • Setup: git config (user.name, email, editor, alias); git init, git clone
  • Basic workflow: git add (-A, -p), git commit (-m), git status, git log (--oneline, --graph)
  • Working with remotes: git remote (-v, add, remove), git push, git pull, git fetch
  • .gitignore: patterns, git rm --cached, global gitignore; git diff, git show

15: Advanced Git – Branching, Merging, and Collaboration
  • Branches: git branch (-a, -d, -D), git checkout/switch, git log --all --graph
  • Merging: fast-forward vs 3-way merge; git merge; conflict resolution (git mergetool)
  • Rebasing: git rebase, interactive rebase (-i: squash, reword, fixup); rebase vs merge
  • Stash and cherry-pick: git stash (push, pop, list, drop); git cherry-pick
  • Collaboration workflow: fork, PR/MR concept; git tag (-a annotated, releases)


📝 Scripting (Bash & Python)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

16: Shell Scripting Basics & Syntax
  • Script structure: shebang (#!/bin/bash), chmod +x, ./script vs bash script
  • Variables: declaration, quoting ("" vs ''), $VAR, ${VAR}, readonly, unset
  • User input: read, read -p, read -s; positional params: $1 $2 $@ $# $0
  • Conditionals: if/elif/else/fi, test/[ ]/[[ ]], comparison operators (-eq, -lt, -z, -f, -d)
  • Exit codes: $?, exit N; logical operators: &&, ||, !

17: Shell Scripting – Loops & Functions
  • for loops: list iteration, C-style for ((i=0;i<n;i++)), range with {1..10}/seq
  • while/until loops: condition-based; read line from file; break and continue
  • Functions: declaration, calling, arguments ($1 $2), local variables, return value
  • Arrays: declaration, indexing (${arr[0]}), ${arr[@]}, ${#arr[@]}, append
  • Robust scripting: set -e, set -u, set -o pipefail; trap for cleanup

18: Shell Scripting Practice
  • Script 1: System info report (hostname, uptime, disk usage, memory, top processes)
  • Script 2: Log file analyzer (parse nginx/syslog, count errors, filter by date with grep/awk)
  • Script 3: Automated backup script (tar + gzip, date-based naming, rotation/cleanup)
  • Script 4: Bulk user creation from CSV file

19: Python Basics for DevOps
  • Python vs Bash: when to use which; running scripts, shebang, venv basics
  • Data types: str, int, float, bool, list, dict, tuple; type(), f-strings, string methods
  • Control flow: if/elif/else, for/while loops, break/continue, list comprehensions
  • Functions: def, arguments, *args, **kwargs, return; import and modules
  • DevOps stdlib: os (path, environ, makedirs), sys (argv, exit), subprocess (run, check_output), pathlib


🌍 Terraform – Infrastructure as Code
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

20: Terraform Basics – Providers, Resources, HCL
  • IaC concepts: imperative vs declarative, idempotency, desired state
  • HCL syntax: blocks, arguments, expressions, types (string, number, bool, list, map)
  • Provider configuration: required_providers, version constraints, credentials setup
  • Resource block: type, name, arguments, meta-arguments (depends_on, lifecycle)
  • CLI workflow: terraform init, validate, fmt, plan, apply, destroy; .terraform directory

21: Variables, Outputs & Data Sources
  • Input variables: variable block, types, default, description, validation block
  • Variable files: .tfvars, .auto.tfvars, -var flag, TF_VAR_ environment variables
  • Locals: local values, computed expressions, use cases vs input variables
  • Output values: output block, sensitive = true, terraform output command, using in scripts
  • Data sources: data block, filtering arguments, referencing data.*.attribute in resources

22: State Management and Remote Backend
  • Terraform state: tfstate file structure, what it tracks, why it exists
  • State commands: terraform state list, show, mv, rm; terraform refresh
  • Remote backends: S3 + DynamoDB locking setup; GCS; Terraform Cloud
  • terraform import: importing existing infra into state; limitations
  • Workspaces: terraform workspace new/list/select/delete; isolation use cases vs separate state files

23: Terraform Modules, Functions & Dynamic Blocks
  • Modules: root/child module concepts; creating local module (variables.tf, outputs.tf, main.tf)
  • Calling modules: module block, source (local path, git, registry), version pinning
  • Built-in functions: format, join, split, merge, flatten, concat, lookup
  • For expressions: [for item in list : expr], {for k,v in map : k => v}, filtering with if
  • Conditional expressions: condition ? true_val : false_val; try() for error handling
  • Dynamic blocks: dynamic "ingress" { for_each, content {} }; splat expressions resource.*.attr


⚙️ Ansible – Configuration Management
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

24: Introduction to Ansible & YAML Syntax
  • Ansible concepts: agentless, SSH-based, push model, idempotency
  • YAML syntax: indentation, scalars, lists, maps, multi-line strings (| vs >)
  • Installation; ansible.cfg basics; ad-hoc commands (-m ping, -m shell, -m copy)
  • First playbook: plays, tasks, modules (apt, file, copy, service, user)
  • Running: ansible-playbook, --check (dry run), -v/-vv verbosity flags

25: Inventory Files & Folder Structure 
  • Static inventory: INI format (groups, child groups, ranges, aliases)
  • YAML inventory format; host_vars/ and group_vars/ directories
  • Dynamic inventory: concept, AWS/GCP plugins, custom script approach
  • Ansible project directory structure: playbooks, roles, group_vars, host_vars
  • Connection variables: ansible_host, ansible_user, ansible_ssh_private_key_file

26: Ansible Roles for Reusability
  • Role directory structure: tasks/, handlers/, defaults/, vars/, templates/, files/, meta/
  • Creating roles: ansible-galaxy init, naming conventions, Galaxy metadata
  • Handlers: notify, listen; handler ordering; flush_handlers
  • Role defaults vs vars: variable precedence, when to use each
  • Using roles in playbooks: roles keyword, import_role, include_role; role dependencies

27: Conditionals and Loops in Ansible
  • when: clause — simple conditions, variable checks, combining conditions
  • Loops: loop with list, loop with dict2items; with_items (legacy)
  • register: capturing task output; accessing .stdout, .rc, .stderr, .results
  • set_fact: creating dynamic variables; combining with register output
  • debug module: var, msg; practical use in troubleshooting playbooks

28: Blocks, Error Handling, and Templating
  • block/rescue/always: structured error handling; use cases vs ignore_errors
  • ignore_errors, failed_when, changed_when: customizing task behavior
  • Jinja2 templates: template module, .j2 files, variable substitution in files
  • Jinja2 filters: default(), upper(), lower(), join(), selectattr(), map(), to_json
  • Lookups: lookup('file'), lookup('env'), lookup('template'), lookup('password')

29: Ansible Configuration, Tags, and Optimizations
  • ansible.cfg deep dive: forks, timeout, remote_user, roles_path, callback_plugins
  • Tags: adding to tasks/plays/roles, --tags, --skip-tags, special tags (always, never)
  • Limiting: --limit (hostname, group, pattern, !exclude), --start-at-task
  • Performance: pipelining, fact caching (jsonfile/redis), async + poll, strategy free
  • Ansible Vault: encrypt/decrypt/view/edit/rekey files; using vault in playbooks

🐳 Docker – Containerization
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

30: Container Fundamentals & Docker Essentials
  • Container vs VM: Linux namespaces, cgroups, union filesystem (overlay2)
  • Docker architecture: daemon (dockerd), client (CLI), registry, containerd
  • Images and containers: pull, images, run (-d, -it, --name, --rm), ps -a, exec -it
  • Port mapping (-p), volume mounts (-v), environment variables (-e, --env-file)
  • Container lifecycle: create → start → stop → rm; logs, inspect, stats

31: Dockerfile Best Practices & Multi-stage Builds
  • Dockerfile instructions: FROM, RUN, COPY, ADD, WORKDIR, ENV, ARG, EXPOSE, USER, CMD, ENTRYPOINT
  • Layer caching: order matters, cache invalidation, .dockerignore file
  • Best practices: minimal base images (alpine/distroless), non-root USER, combine RUN layers
  • Multi-stage builds: AS alias, COPY --from, separating build and runtime stages
  • Build options: docker build -t, -f, --build-arg, --target, --no-cache

32: Docker Volumes, Networks & Security
  • Volume types: bind mounts (-v /host:/container), named volumes, tmpfs; when to use each
  • Volume commands: docker volume create/ls/inspect/rm/prune; sharing volumes between containers
  • Docker networking deep dive: bridge, host, none, overlay; custom networks; container DNS
  • Docker security: non-root USER, read-only filesystem (--read-only), capabilities (--cap-drop/add)
  • Resource limits: --memory, --cpus, --pids-limit; docker stats; Trivy image scan intro

33: Docker Compose (essentials) & Container Registries
  • Docker Compose: docker-compose.yml (services, image/build, ports, volumes, environment, depends_on)
  • Compose commands: up -d, down, build, logs, ps, exec, restart; compose networks and volumes
  • Docker Hub: login, push, pull; image tagging convention (user/image:tag, semantic versioning)
  • Private registries: self-hosted registry, Harbor concept; docker login with credentials
  • Practice: multi-container app with Compose (app + db + reverse proxy)


☸️ Kubernetes – Container Orchestration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

34: Introduction to Kubernetes & Architecture
  • Why Kubernetes: orchestration challenges, self-healing, autoscaling, service discovery
  • Control plane: kube-apiserver, etcd, kube-scheduler, kube-controller-manager
  • Worker node components: kubelet, kube-proxy, container runtime (containerd/CRI-O)
  • Kubernetes object model: declarative API, desired vs actual state, reconciliation loop
  • Cluster access: kubeconfig file, contexts, kubectl cluster-info, kubectl version

35: kubectl & Core Resource Management
  • Core commands: get, describe, apply (-f, -k), delete, edit, patch (--type merge)
  • Output formats: -o yaml, -o json, -o wide, -o jsonpath, -o custom-columns
  • Namespaces: create, -n flag, --all-namespaces; config set-context --current --namespace
  • Resource discovery: kubectl explain, kubectl api-resources, kubectl api-versions
  • Labels/selectors/annotations: --selector (-l), kubectl label, kubectl annotate; dry-run

36: Pods, Deployments & Manifest Files
  • Pod spec: containers (name, image, ports, resources), restartPolicy, imagePullPolicy
  • Deployment: replicas, selector, template, strategy (RollingUpdate, Recreate), maxSurge
  • ReplicaSet: relationship to Deployment; kubectl scale (--replicas); HPA concept
  • Rollouts: kubectl rollout status/history/undo; pause and resume rollout
  • Writing YAML manifests: --dry-run=client -o yaml; structure walkthrough; kubectl diff

37: Services & Networking – ClusterIP, NodePort
  • Service concepts: stable VIP, DNS name, selector-based targeting, Endpoints object
  • ClusterIP: internal communication, kube-dns service discovery (svc.namespace.svc.cluster.local)
  • NodePort: external access, port ranges (30000-32767), use cases and limitations
  • LoadBalancer: cloud provider integration, external IP provisioning; ExternalName
  • Headless service: clusterIP: None; kubectl port-forward for local debugging

38: Ingress & DNS
  • Ingress controller: nginx-ingress, installation via Helm, IngressClass resource
  • Ingress resource: rules (host-based, path-based), pathType (Prefix/Exact), defaultBackend
  • TLS termination: tls block in Ingress, cert-manager (ClusterIssuer, Certificate, ACME)
  • Annotations: nginx.ingress.kubernetes.io/* (rewrite-target, ssl-redirect, rate-limit)
  • CoreDNS: in-cluster DNS resolution, dnsPolicy options, custom stub domains

39: Gateway API – Modern Traffic Management
  • Gateway API vs Ingress: Ingress limitations, role-oriented model (Infra/Cluster/App)
  • Core resources: GatewayClass (provider), Gateway (listener), HTTPRoute (routing rules)
  • HTTPRoute: rules, matches (path, header, method), filters (redirect, rewrite), backendRefs
  • Advanced routing: TCPRoute, GRPCRoute (concept); ReferenceGrant (cross-namespace traffic)
  • Hands-on: deploy a gateway controller, create HTTPRoute, test traffic routing

40: Practice – Build, Push, Deploy Containers to Kubernetes
  • Build a multi-stage Dockerfile for a sample app; optimize the image
  • Push image to registry with proper tagging (:latest, :git-sha)
  • Write Deployment + Service manifests; apply to cluster; verify with kubectl
  • Configure Ingress/Gateway route; test end-to-end HTTP access
  • Simulate a rolling update: change image tag, observe rollout

41: Stateful Apps & Persistent Volumes
  • PersistentVolume (PV): static provisioning, accessModes (RWO/ROX/RWX), reclaimPolicy
  • PersistentVolumeClaim (PVC): requesting storage, binding, volumeMounts in pod spec
  • StorageClass: dynamic provisioning, provisioner, reclaimPolicy, volumeBindingMode
  • StatefulSet vs Deployment: stable network identity, ordered pod management, sticky storage
  • Hands-on: deploy PostgreSQL as StatefulSet with PVC; verify data survives pod restart

42: Configuration Management & Secrets
  • ConfigMap: creation methods (--from-literal, --from-file, YAML); kubectl describe
  • Using ConfigMaps: valueFrom.configMapKeyRef (env), envFrom (all keys), volumeMount
  • Secret: types (Opaque, docker-registry, tls), base64 encoding; stringData shortcut
  • Using Secrets: secretKeyRef (env), envFrom, volume mount as files; permissions
  • Best practices: avoid secrets in manifests committed to Git; RBAC for secret access

43: Secrets Management – External Secrets Operator (ESO)
  • Why ESO: limitations of native K8s Secrets (base64, etcd, no rotation)
  • ESO architecture: operator, SecretStore, ClusterSecretStore, ExternalSecret CRDs
  • Pre-configured Vault: KV v2 engine, Kubernetes auth method (concept walkthrough)
  • SecretStore: Vault provider config, auth (serviceAccount token, approle)
  • ExternalSecret: data mapping, refreshInterval, secretStoreRef; verify secret sync

44: Practice – Volumes & Secrets in Action
  • Deploy stateful app (PostgreSQL) with PVC; verify storage class and binding
  • Create ExternalSecret pulling DB credentials from pre-configured Vault
  • Mount application config via ConfigMap volume
  • Verify persistence: delete pod, confirm data and secrets survive
  • Troubleshoot common issues: pending PVC, failed ExternalSecret sync, wrong key path

45: Probes & Basic Scheduling Techniques
  • Liveness probe: httpGet, exec, tcpSocket; initialDelaySeconds, periodSeconds, failureThreshold
  • Readiness probe: difference from liveness, effect on Service endpoints (traffic gating)
  • Startup probe: for slow-starting containers; failureThreshold × periodSeconds formula
  • Resource requests and limits: CPU (millicores), memory (Mi/Gi), QoS classes (Guaranteed/Burstable/BestEffort)
  • LimitRange: per-container defaults; ResourceQuota: per-namespace caps

46: Advanced Scheduling with Taints & Node Affinity
  • nodeSelector: simple label-based scheduling; kubectl label nodes
  • Taints and tolerations: kubectl taint, effects (NoSchedule, PreferNoSchedule, NoExecute)
  • nodeAffinity: requiredDuringScheduling, preferredDuringScheduling; operators (In, NotIn, Exists)
  • podAffinity and podAntiAffinity: co-location and spread rules; topologyKey (concept + example)
  • topologySpreadConstraints: maxSkew, whenUnsatisfiable (ScheduleAnyway/DoNotSchedule) — concept

47: Kubernetes Security – RBAC & Authentication
  • Authentication methods: kubeconfig certificates, ServiceAccount tokens
  • ServiceAccounts: default SA, creating custom SA, automountServiceAccountToken
  • Roles and ClusterRoles: rules (apiGroups, resources, verbs); kubectl create role
  • RoleBindings and ClusterRoleBindings: subjects (User, Group, ServiceAccount); scope
  • RBAC best practices: least privilege; kubectl auth can-i; audit RBAC with tools

48: Kubernetes Security – Network Policies, PodSecurity & Admission Control
  • Network Policies: podSelector, policyTypes (Ingress/Egress), namespaceSelector, ipBlock
  • Default deny pattern: deny-all ingress + egress; allow selectively; test connectivity
  • PodSecurity Admission: profiles (privileged/baseline/restricted), modes (enforce/warn/audit)
  • Admission controllers: built-in (LimitRanger, ResourceQuota, NodeRestriction, NamespaceLifecycle)
  • ValidatingWebhookConfiguration: concept, how webhooks intercept API calls; common examples

49: Kubernetes Security – Kube-Bench, AppArmor, Seccomp
  • kube-bench: CIS Kubernetes Benchmark, installation, running kube-bench, reading results
  • Remediating common kube-bench failures: API server flags, etcd permissions, kubelet config
  • AppArmor: Linux security module, profiles (enforce/complain/unconfined), aa-status, aa-genprof
  • Applying AppArmor to containers: securityContext.appArmorProfile (K8s 1.30+); pod annotation method
  • Seccomp: syscall filtering, runtime/default profile, custom JSON profile, securityContext.seccompProfile

50: Kubernetes Security – Image Scanning, Audit Logs & Kyverno Intro
  • Trivy: trivy image (scan locally), severity levels (CRITICAL/HIGH/MEDIUM), fix versions
  • Trivy in CI: scan before push; fail pipeline on CRITICAL; ignoring CVEs (.trivyignore)
  • Kubernetes Audit Logs: enabling audit policy, log levels (None/Metadata/Request/RequestResponse)
  • Reading audit logs: identifying suspicious API calls, integration with log shipper
  • Kyverno intro: install, ClusterPolicy, validate rule (deny privileged pods) — live demo

51: Practice – Scheduling and Security
  • Deploy workload with nodeAffinity + toleration for a tainted node
  • Apply Network Policy: deny-all then allow only frontend → backend traffic
  • Run kube-bench, identify one failing check, apply remediation
  • Scan a known-vulnerable image with Trivy; rebuild with updated base image
  • Apply Kyverno ClusterPolicy (deny latest tag), test enforcement by deploying :latest

52: Cluster Setup – kubeadm (demo) & Rancher rke2 (hands-on)
  • kubeadm demo: prerequisites, kubeadm init (--pod-network-cidr, --apiserver-advertise-address)
  • kubeadm internals: certificate generation, static pods (/etc/kubernetes/manifests), phases
  • kubeadm join: worker node join with token; CNI plugin installation (Flannel/Calico)
  • rke2 hands-on: install server, /etc/rancher/rke2/config.yaml, start service, get kubeconfig
  • rke2 worker: install agent, join cluster; verify nodes; kubeadm vs rke2 comparison


⛵ Helm – Kubernetes Package Manager
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

53: Helm Basics – Charts and Repositories
  • Helm concepts: package manager for K8s, charts, releases, revisions, chart.yaml
  • Chart structure: Chart.yaml, values.yaml, templates/, charts/ (dependencies)
  • Repositories: helm repo add/update/list/remove; search repo and Artifact Hub
  • Installing releases: helm install (--set, -f values.yaml, --namespace, --create-namespace)
  • Managing releases: helm list, status, get values/manifest, upgrade, rollback, uninstall

54: Helm Templating and Customization
  • Go template syntax: {{ }}, pipeline |, whitespace control {{- -}}, comments
  • Built-in objects: .Values, .Release (Name/Namespace/IsInstall/IsUpgrade), .Chart, .Files
  • Functions: quote, default, toYaml | nindent, include, tpl; type checks
  • _helpers.tpl: defining named templates with {{- define }}, include vs template
  • Conditionals (if/else/with) and loops (range over list/map) in templates

55: Practice – Deploy Complex App with Helm
  • Write a Helm chart from scratch for a multi-tier app (frontend + backend + db)
  • Define values.yaml with environment overrides; use --set and -f for customization
  • Use _helpers.tpl for consistent labels, names, selectors
  • Package (helm package), install, upgrade with changed values
  • Debug: helm template (local render), helm lint, helm install --dry-run


🧩 Kustomize – Native K8s Templating
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

56: Kustomize Basics – Overlays and Resources
  • Kustomize concepts: no templating, patch-based, built into kubectl (kubectl apply -k)
  • kustomization.yaml: resources, namePrefix, nameSuffix, commonLabels, namespace
  • Bases and overlays: directory structure (base/, overlays/dev/, overlays/prod/)
  • Images transformer: overriding image name and tag per environment
  • kubectl apply -k, kubectl kustomize (build); kustomize CLI

57: Kustomize Advanced – Patches and Config Management
  • Strategic merge patch: overlay YAML merged with base; adding/replacing fields
  • JSON 6902 patch: precise operations (add, remove, replace, copy, move) with path
  • configMapGenerator: from files and literals; disableNameSuffixHash option
  • secretGenerator: generating secrets in Kustomize
  • Kustomize vs Helm: trade-offs; using both (helm template | kustomize apply)

58: Practice – Manage Environments with Kustomize
  • Set up base + 3 overlays (dev/staging/prod) for a real application
  • Apply different replicas, resource limits, image tags per environment
  • Use configMapGenerator for environment-specific app config
  • Apply to cluster: kubectl apply -k overlays/dev; kubectl apply -k overlays/prod
  • Verify differences: kubectl diff -k; review generated output with kustomize build


🔁 CI/CD – Automation Pipelines
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

59: CI/CD Introduction – Concepts, Tools & Lifecycle
  • CI/CD concepts: continuous integration, delivery, deployment; shift-left testing
  • Pipeline lifecycle: source → build → test → scan → artifact → deploy → verify
  • Tools landscape: GitLab CI, GitHub Actions, Jenkins, ArgoCD — comparison
  • GitLab CI intro: .gitlab-ci.yml structure, stages, jobs, scripts, before_script
  • Artifacts, caching, environments: overview; pipeline visualization in GitLab UI

60: GitLab Setup & Runners
  • GitLab setup: using GitLab.com; project/group structure; protected branches
  • Runner types: shared, group, project-specific; executors (shell, Docker, Kubernetes)
  • Runner registration: gitlab-runner register, config.toml, tags, concurrent
  • .gitlab-ci.yml deep dive: stages order, job keywords (rules, only/except, when, needs)
  • CI/CD variables: predefined ($CI_COMMIT_SHA, $CI_PROJECT_NAME), custom (masked, protected)

61: GitHub Actions – Workflows and Pipelines
  • GitHub Actions concepts: workflow, events (push, PR, schedule, workflow_dispatch)
  • Workflow YAML: on, jobs (runs-on, needs, if), steps (uses, run, with, env)
  • Actions: using marketplace (actions/checkout, actions/setup-node, docker/build-push-action)
  • Secrets and variables: secrets context, vars context, environment protection rules
  • Matrix builds: strategy.matrix, include/exclude; reusable workflows (workflow_call)

62: GitLab CI/CD for Dockerized Applications
  • Building Docker images in CI: Docker-in-Docker (dind)
  • .gitlab-ci.yml: build job with docker build, tag with $CI_COMMIT_SHA / $CI_COMMIT_TAG
  • GitLab Container Registry: $CI_REGISTRY_* variables, docker login, docker push
  • Multi-stage pipeline: build → scan → push; conditional push on main branch only
  • Image scanning: integrating Trivy as a CI job; fail on CRITICAL/HIGH vulnerabilities

63: GitLab CI/CD for Kubernetes – Artifacts and Caching
  • kubectl in CI: kubeconfig as base64-encoded CI variable; KUBECONFIG env setup
  • Deployment job: kubectl apply or helm upgrade --install in pipeline
  • Environments in GitLab: environment keyword, deployment tracking, rollback from UI
  • Artifacts: build outputs, test reports (JUnit XML for GitLab test visualization), expiry
  • Caching: cache key (branch, file hash), paths, when; cache vs artifacts trade-offs

64: GitLab CI/CD with Terraform – IaC Pipelines
  • Terraform in CI: use official hashicorp/terraform Docker image in jobs
  • Pipeline stages: validate → fmt check → plan → apply (manual trigger for apply)
  • State in CI: remote backend credentials as CI variables; state locking in pipelines
  • Destroy job: manual trigger only; environment cleanup; drift detection concept

65: GitOps with ArgoCD
  • GitOps principles: Git as single source of truth, declarative infra, automated reconciliation
  • ArgoCD architecture: API server, repo server, application controller, Dex (SSO)
  • Installing ArgoCD: Helm install, access UI via port-forward or Ingress
  • Application resource: repoURL, path, targetRevision, destination, syncPolicy
  • Sync strategies: auto-sync, self-heal, prune; App of Apps pattern; ApplicationSet intro

66: CI/CD Practice – End-to-End Pipeline
  • Build & push Docker image to registry in GitLab CI (trigger on merge to main)
  • Run Trivy scan as a CI job; fail pipeline on CRITICAL vulnerabilities
  • Deploy to Kubernetes via helm upgrade --install with image tag from $CI_COMMIT_SHA
  • ArgoCD sync verification: trigger app sync, confirm deployment via ArgoCD API/CLI


📈 Monitoring & Logging
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

67: Monitoring Overview – Metrics & Alerts
  • Why monitor: SLI/SLO/SLA concepts; RED method (Rate/Errors/Duration), USE method
  • Prometheus architecture: scrape model, TSDB, /metrics endpoint format, scrape_config
  • Metric types: counter, gauge, histogram, summary; labels and cardinality trade-offs
  • Alertmanager: receiver → route → inhibit → silence pipeline; routing tree config
  • Grafana overview: data sources, dashboards, panels, variables; import community dashboards

68: Metrics Collection with Exporters
  • node_exporter: installation, systemd service, key metrics (CPU, memory, disk, network)
  • Prometheus scrape config: static_configs, scrape_interval, job_name, relabeling basics
  • blackbox_exporter: HTTP/TCP/ICMP probes, probe_success metric, scrape config
  • PromQL basics: instant vectors, range vectors, label selectors ({job="node"})
  • PromQL functions: rate(), irate(), increase(), sum()/avg()/max() by (label)

69: Secure Monitoring – Auth, SSL, and Uptime Kuma
  • Prometheus basic auth: web.yml config, bcrypt password hash, scrape credentials
  • TLS for Prometheus: self-signed cert, web.yml tls_server_config, secure scraping
  • Grafana security: admin password, HTTPS (reverse proxy via nginx), auth options
  • Uptime Kuma: installation (Docker), monitor types (HTTP/TCP/DNS/keyword/ping)
  • Uptime Kuma notifications: Telegram, Slack, email; status pages

70: Centralized Logging with Grafana Loki
  • Loki architecture: distributor, ingester, querier, compactor, object storage backend
  • Promtail: installation, scrape_configs, pipeline_stages (regex, json, labels, timestamp)
  • Loki data model: streams, label-based indexing (no full-text index), chunk storage
  • LogQL: stream selectors {app="nginx"}, filter expressions (|=, !=, |~), pattern filter
  • Grafana integration: Loki data source, Explore view, log panels, correlate logs + metrics

71: Deploy Monitoring Stack in Kubernetes
  • kube-prometheus-stack: what's included (Prometheus, Alertmanager, Grafana, exporters)
  • Helm install: values.yaml (storage PVC, retention, Grafana credentials, ingress)
  • ServiceMonitor and PodMonitor: how Prometheus discovers K8s workloads; label matching
  • PrometheusRule: custom alert rules resource; expr, for, labels, annotations
  • Loki stack in K8s: loki-stack Helm chart, Promtail DaemonSet, log pipeline to Grafana

72: Monitoring & Alerting – Final Practice & Capstone
  • Deploy a sample app exposing /metrics endpoint; verify scraping in Prometheus UI
  • Configure ServiceMonitor; create Grafana dashboard (request rate, error rate, latency)
  • Write a PrometheusRule alert (e.g. error rate > 5%); route to Alertmanager
  • Configure Alertmanager receiver (Telegram or Slack); trigger alert and verify delivery
  • Course recap: end-to-end flow (code → CI/CD → deploy → monitor → alert)

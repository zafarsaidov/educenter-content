# DevOps Basics

This bootcamp takes engineers with no prior operations experience from a bare Linux terminal to running a full CI/CD pipeline that builds, scans, and deploys containerized applications to Kubernetes with GitOps and monitoring in place. Each module builds on the last: Linux and scripting fundamentals first, then infrastructure-as-code and configuration management, then containers and orchestration, then the automation and observability layers that tie it all together.

## Curriculum

<!-- 12 sections · 72 themes -->

### Linux Basics

Terminal fundamentals, filesystem navigation, permissions, text editors, and core system administration — the foundation every later module assumes.

- [DevOps Concept. Introduction to Linux & Essential Commands](./linux-basics/devops-concept-introduction-to-linux-essential-commands.md)
- [Navigating the Filesystem – Absolute/Relative Paths, Inodes & Directory Structure](./linux-basics/navigating-the-filesystem-absolute-relative-paths-inodes-directory-structure.md)
- [File Permissions, Ownership & Text Editors (vim, nano)](./linux-basics/file-permissions-ownership-text-editors-vim-nano.md)
- [File Search, Text Filtering & awk](./linux-basics/file-search-text-filtering-awk.md)
- [Managing System Services with systemd](./linux-basics/managing-system-services-with-systemd.md)
- [User and Group Management](./linux-basics/user-and-group-management.md)
- [Linux Package Management](./linux-basics/linux-package-management.md)
- [Linux Process and Job Management](./linux-basics/linux-process-and-job-management.md)
- [Scheduled Tasks & Kernel Parameters](./linux-basics/scheduled-tasks-kernel-parameters.md)
- [Linux Networking Essentials – TCP/IP, Ports & Tools](./linux-basics/linux-networking-essentials-tcp-ip-ports-tools.md)
- [iptables, ufw & SSH](./linux-basics/iptables-ufw-ssh.md)
- [Disk Partitioning, File Systems & Disk Monitoring](./linux-basics/disk-partitioning-file-systems-disk-monitoring.md)

### Web Servers & Load Balancing

Configuring nginx as a web server, reverse proxy, and load balancer, with HTTPS termination.

- [nginx – Web Server, Reverse Proxy & Load Balancing](./web-servers-load-balancing/nginx-web-server-reverse-proxy-load-balancing.md)

### Git Version Control

Core Git workflow through branching, merging, rebasing, and collaboration patterns.

- [Git Fundamentals and Workflow](./git-version-control/git-fundamentals-and-workflow.md)
- [Advanced Git – Branching, Merging, and Collaboration](./git-version-control/advanced-git-branching-merging-and-collaboration.md)

### Scripting (Bash & Python)

Automating repetitive operations tasks with shell scripts and Python.

- [Shell Scripting Basics & Syntax](./scripting-bash-python/shell-scripting-basics-syntax.md)
- [Shell Scripting – Loops & Functions](./scripting-bash-python/shell-scripting-loops-functions.md)
- [Shell Scripting Practice](./scripting-bash-python/shell-scripting-practice.md)
- [Python Basics for DevOps](./scripting-bash-python/python-basics-for-devops.md)

### Terraform – Infrastructure as Code

Declarative infrastructure provisioning: providers, state, variables, and reusable modules.

- [Terraform Basics – Providers, Resources, HCL](./terraform-infrastructure-as-code/terraform-basics-providers-resources-hcl.md)
- [Variables, Outputs & Data Sources](./terraform-infrastructure-as-code/variables-outputs-data-sources.md)
- [State Management and Remote Backend](./terraform-infrastructure-as-code/state-management-and-remote-backend.md)
- [Terraform Modules, Functions & Dynamic Blocks](./terraform-infrastructure-as-code/terraform-modules-functions-dynamic-blocks.md)

### Ansible – Configuration Management

Agentless configuration management: playbooks, inventories, roles, and templating with Jinja2.

- [Introduction to Ansible & YAML Syntax](./ansible-configuration-management/introduction-to-ansible-yaml-syntax.md)
- [Inventory Files & Folder Structure](./ansible-configuration-management/inventory-files-folder-structure.md)
- [Ansible Roles for Reusability](./ansible-configuration-management/ansible-roles-for-reusability.md)
- [Conditionals and Loops in Ansible](./ansible-configuration-management/conditionals-and-loops-in-ansible.md)
- [Blocks, Error Handling, and Templating](./ansible-configuration-management/blocks-error-handling-and-templating.md)
- [Ansible Configuration, Tags, and Optimizations](./ansible-configuration-management/ansible-configuration-tags-and-optimizations.md)

### Docker – Containerization

Building, securing, and running containers, from a single Dockerfile to a multi-container Compose stack.

- [Container Fundamentals & Docker Essentials](./docker-containerization/container-fundamentals-docker-essentials.md)
- [Dockerfile Best Practices & Multi-stage Builds](./docker-containerization/dockerfile-best-practices-multi-stage-builds.md)
- [Docker Volumes, Networks & Security](./docker-containerization/docker-volumes-networks-security.md)
- [Docker Compose (essentials) & Container Registries](./docker-containerization/docker-compose-essentials-container-registries.md)

### Kubernetes – Container Orchestration

The largest module: cluster architecture, workloads, networking, storage, scheduling, and security, culminating in a self-managed cluster setup.

- [Introduction to Kubernetes & Architecture](./kubernetes-container-orchestration/introduction-to-kubernetes-architecture.md)
- [kubectl & Core Resource Management](./kubernetes-container-orchestration/kubectl-core-resource-management.md)
- [Pods, Deployments & Manifest Files](./kubernetes-container-orchestration/pods-deployments-manifest-files.md)
- [Services & Networking – ClusterIP, NodePort](./kubernetes-container-orchestration/services-networking-clusterip-nodeport.md)
- [Ingress & DNS](./kubernetes-container-orchestration/ingress-dns.md)
- [Gateway API – Modern Traffic Management](./kubernetes-container-orchestration/gateway-api-modern-traffic-management.md)
- [Practice – Build, Push, Deploy Containers to Kubernetes](./kubernetes-container-orchestration/practice-build-push-deploy-containers-to-kubernetes.md)
- [Stateful Apps & Persistent Volumes](./kubernetes-container-orchestration/stateful-apps-persistent-volumes.md)
- [Configuration Management & Secrets](./kubernetes-container-orchestration/configuration-management-secrets.md)
- [Secrets Management – External Secrets Operator (ESO)](./kubernetes-container-orchestration/secrets-management-external-secrets-operator-eso.md)
- [Practice – Volumes & Secrets in Action](./kubernetes-container-orchestration/practice-volumes-secrets-in-action.md)
- [Probes & Basic Scheduling Techniques](./kubernetes-container-orchestration/probes-basic-scheduling-techniques.md)
- [Advanced Scheduling with Taints & Node Affinity](./kubernetes-container-orchestration/advanced-scheduling-with-taints-node-affinity.md)
- [Kubernetes Security – RBAC & Authentication](./kubernetes-container-orchestration/kubernetes-security-rbac-authentication.md)
- [Kubernetes Security – Network Policies, PodSecurity & Admission Control](./kubernetes-container-orchestration/kubernetes-security-network-policies-podsecurity-admission-control.md)
- [Kubernetes Security – Kube-Bench, AppArmor, Seccomp](./kubernetes-container-orchestration/kubernetes-security-kube-bench-apparmor-seccomp.md)
- [Kubernetes Security – Image Scanning, Audit Logs & Kyverno Intro](./kubernetes-container-orchestration/kubernetes-security-image-scanning-audit-logs-kyverno-intro.md)
- [Practice – Scheduling and Security](./kubernetes-container-orchestration/practice-scheduling-and-security.md)
- [Cluster Setup – kubeadm (demo) & Rancher rke2 (hands-on)](./kubernetes-container-orchestration/cluster-setup-kubeadm-demo-rancher-rke2-hands-on.md)

### Helm – Kubernetes Package Manager

Packaging and templating Kubernetes manifests with Helm charts.

- [Helm Basics – Charts and Repositories](./helm-kubernetes-package-manager/helm-basics-charts-and-repositories.md)
- [Helm Templating and Customization](./helm-kubernetes-package-manager/helm-templating-and-customization.md)
- [Practice – Deploy Complex App with Helm](./helm-kubernetes-package-manager/practice-deploy-complex-app-with-helm.md)

### Kustomize – Native K8s Templating

Patch-based, template-free environment management with kubectl's built-in Kustomize support.

- [Kustomize Basics – Overlays and Resources](./kustomize-native-k8s-templating/kustomize-basics-overlays-and-resources.md)
- [Kustomize Advanced – Patches and Config Management](./kustomize-native-k8s-templating/kustomize-advanced-patches-and-config-management.md)
- [Practice – Manage Environments with Kustomize](./kustomize-native-k8s-templating/practice-manage-environments-with-kustomize.md)

### CI/CD – Automation Pipelines

Building automated pipelines with GitLab CI and GitHub Actions, and closing the loop with GitOps via ArgoCD.

- [CI/CD Introduction – Concepts, Tools & Lifecycle](./ci-cd-automation-pipelines/ci-cd-introduction-concepts-tools-lifecycle.md)
- [GitLab Setup & Runners](./ci-cd-automation-pipelines/gitlab-setup-runners.md)
- [GitHub Actions – Workflows and Pipelines](./ci-cd-automation-pipelines/github-actions-workflows-and-pipelines.md)
- [GitLab CI/CD for Dockerized Applications](./ci-cd-automation-pipelines/gitlab-ci-cd-for-dockerized-applications.md)
- [GitLab CI/CD for Kubernetes – Artifacts and Caching](./ci-cd-automation-pipelines/gitlab-ci-cd-for-kubernetes-artifacts-and-caching.md)
- [GitLab CI/CD with Terraform – IaC Pipelines](./ci-cd-automation-pipelines/gitlab-ci-cd-with-terraform-iac-pipelines.md)
- [GitOps with ArgoCD](./ci-cd-automation-pipelines/gitops-with-argocd.md)
- [CI/CD Practice – End-to-End Pipeline](./ci-cd-automation-pipelines/ci-cd-practice-end-to-end-pipeline.md)

### Monitoring & Logging

Metrics, alerting, and centralized logging with Prometheus, Grafana, Loki, and Uptime Kuma — including a Kubernetes deployment and a capstone exercise.

- [Monitoring Overview – Metrics & Alerts](./monitoring-logging/monitoring-overview-metrics-alerts.md)
- [Metrics Collection with Exporters](./monitoring-logging/metrics-collection-with-exporters.md)
- [Secure Monitoring – Auth, SSL, and Uptime Kuma](./monitoring-logging/secure-monitoring-auth-ssl-and-uptime-kuma.md)
- [Centralized Logging with Grafana Loki](./monitoring-logging/centralized-logging-with-grafana-loki.md)
- [Deploy Monitoring Stack in Kubernetes](./monitoring-logging/deploy-monitoring-stack-in-kubernetes.md)
- [Monitoring & Alerting – Final Practice & Capstone](./monitoring-logging/monitoring-alerting-final-practice-capstone.md)

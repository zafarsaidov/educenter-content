# Kubernetes Advanced

This module is for engineers who are comfortable running workloads on Kubernetes and want to operate clusters at a production level. It goes beyond deploying applications and covers the full range of production concerns: controlling how and where pods are scheduled, scaling workloads and clusters automatically, managing persistent storage, securing the cluster with RBAC and admission policies, packaging applications with Helm, automating delivery with GitOps, and keeping the cluster observable and maintainable over time.

## Curriculum

<!-- 12 sections · 45 themes -->

### Node Scheduling and Affinity

Controlling which nodes a pod can land on, from simple label matching to expressive affinity rules and co-location constraints.

- [Node Selectors — Simple Label-Based Node Targeting](./k8s-adv-scheduling/k8s-node-selectors.md)
- [Node Affinity and Anti-Affinity — Expressive Node Targeting](./k8s-adv-scheduling/k8s-node-affinity.md)
- [Pod Affinity and Anti-Affinity — Co-locating and Spreading Pods](./k8s-adv-scheduling/k8s-pod-affinity.md)

### Taints, Tolerations, and Pod Priority

Controlling which pods are repelled from or permitted onto nodes, distributing pods evenly across failure domains, and managing resource pressure with pod priority and preemption.

- [Taints and Tolerations — Dedicated Node Pools](./k8s-adv-taints-priority/k8s-taints-tolerations.md)
- [Topology Spread Constraints — Even Pod Distribution](./k8s-adv-taints-priority/k8s-topology-spread.md)
- [Priority Classes and Preemption](./k8s-adv-taints-priority/k8s-priority-classes.md)

### Autoscaling

Covers the full Kubernetes autoscaling stack: scaling pod replicas, right-sizing resource requests, adding and removing nodes, event-driven scaling with KEDA, and protecting availability during disruptions.

- [Horizontal Pod Autoscaler — Scaling Replicas on Demand](./k8s-adv-autoscaling/k8s-hpa.md)
- [Vertical Pod Autoscaler — Right-Sizing Container Resources](./k8s-adv-autoscaling/k8s-vpa.md)
- [Cluster Autoscaler — Adding and Removing Nodes](./k8s-adv-autoscaling/k8s-cluster-autoscaler.md)
- [KEDA — Event-Driven Autoscaling](./k8s-adv-autoscaling/k8s-keda.md)
- [Pod Disruption Budgets — Safe Voluntary Disruptions](./k8s-adv-autoscaling/k8s-pdb.md)

### Storage

Covers all Kubernetes storage primitives from in-pod ephemeral volumes to dynamic provisioning, StatefulSet per-pod PVCs, snapshot-based backup, and the CSI driver model.

- [Volume Types — emptyDir, hostPath, and Projected Volumes](./k8s-adv-storage/k8s-volume-types.md)
- [PersistentVolumes and PersistentVolumeClaims — Durable Storage](./k8s-adv-storage/k8s-persistent-volumes.md)
- [StorageClasses and Dynamic Provisioning](./k8s-adv-storage/k8s-storage-classes.md)
- [volumeClaimTemplates in StatefulSets — Per-Pod Stable Storage](./k8s-adv-storage/k8s-statefulset-volumes.md)
- [Volume Snapshots — Point-in-Time PVC Backup and Restore](./k8s-adv-storage/k8s-volume-snapshots.md)
- [CSI Drivers — Plugging External Storage into Kubernetes](./k8s-adv-storage/k8s-csi-drivers.md)

### Security: RBAC and Pod Security

Learn how Kubernetes controls who can do what in the cluster, how pods authenticate, and how to harden container runtime settings and enforce security policies cluster-wide.

- [RBAC — Controlling Who Can Do What in the Cluster](./k8s-adv-security-rbac/k8s-rbac.md)
- [ServiceAccounts — Pod Identity in Kubernetes](./k8s-adv-security-rbac/k8s-service-accounts.md)
- [Pod Security Context — Hardening Container Runtime](./k8s-adv-security-rbac/k8s-pod-security-context.md)
- [Pod Security Standards — Cluster-Wide Policy Enforcement](./k8s-adv-security-rbac/k8s-pod-security-standards.md)

### Security: Admission and Secrets Management

Covers policy enforcement via admission controllers, secrets management beyond native Kubernetes Secrets, container image security, and cluster audit logging.

- [Admission Controllers — Policy Enforcement at the API Gateway](./k8s-adv-security-admission/k8s-admission-controllers.md)
- [Secrets Management — Beyond Native Kubernetes Secrets](./k8s-adv-security-admission/k8s-secrets-management.md)
- [Image Security — Scanning and Signing Container Images](./k8s-adv-security-admission/k8s-image-security.md)
- [Audit Logging — Recording API Server Activity](./k8s-adv-security-admission/k8s-audit-logging.md)

### Helm: Charts and Releases

The standard way to package, version, and distribute Kubernetes applications. Covers Helm architecture, the full release lifecycle, and writing Helm charts from scratch.

- [Helm Concepts — Charts, Releases, and Repositories](./k8s-adv-helm-charts/k8s-helm-concepts.md)
- [Managing Helm Releases — Install, Upgrade, Rollback, and History](./k8s-adv-helm-charts/k8s-helm-releases.md)
- [Writing Helm Charts — Templates, Values, and Helpers](./k8s-adv-helm-charts/k8s-helm-writing-charts.md)

### Helm: Testing and Kustomize

Validating Helm charts, managing multi-chart deployments with Helmfile, and overlay-based configuration management with Kustomize.

- [Chart Testing, Linting, and Dry-Run Rendering](./k8s-adv-helm-kustomize/k8s-helm-testing.md)
- [Helmfile — Declarative Multi-Chart Deployments](./k8s-adv-helm-kustomize/k8s-helmfile.md)
- [Kustomize — Overlay-Based Configuration Without Templating](./k8s-adv-helm-kustomize/k8s-kustomize.md)

### Observability and Debugging

Covers the Kubernetes metadata system, health probes, the full kubectl debugging toolkit, common pod failure modes, and resource usage monitoring.

- [Labels, Annotations, and Selectors — The Kubernetes Metadata System](./k8s-adv-observability/k8s-labels-selectors.md)
- [Probes — Liveness, Readiness, and Startup Health Checks](./k8s-adv-observability/k8s-probes.md)
- [kubectl Debugging — Logs, Exec, Describe, and Port-Forward](./k8s-adv-observability/k8s-kubectl-debugging.md)
- [Common Failure Modes — Diagnosing Kubernetes Problems](./k8s-adv-observability/k8s-failure-modes.md)
- [Metrics Server and kubectl top — Resource Usage Monitoring](./k8s-adv-observability/k8s-metrics-server.md)

### Cluster Administration

Covers bootstrapping a cluster with kubeadm, comparing managed vs self-managed Kubernetes offerings, and performing safe node lifecycle operations.

- [kubeadm — Bootstrapping a Kubernetes Cluster](./k8s-adv-cluster-admin/k8s-kubeadm.md)
- [Managed Kubernetes — EKS, GKE, and AKS vs Self-Managed](./k8s-adv-cluster-admin/k8s-managed-kubernetes.md)
- [Node Lifecycle — Drain, Cordon, and Uncordon](./k8s-adv-cluster-admin/k8s-node-lifecycle.md)

### Cluster Operations and Multi-tenancy

Covers in-place cluster upgrades with kubeadm, etcd snapshot backup and restore for disaster recovery, and namespace-based multi-tenancy patterns for sharing clusters between teams.

- [Cluster Upgrades — Upgrading Kubernetes with kubeadm](./k8s-adv-cluster-ops/k8s-cluster-upgrades.md)
- [etcd Backup and Restore — Cluster Disaster Recovery](./k8s-adv-cluster-ops/k8s-etcd-backup.md)
- [Multi-tenancy — Sharing a Cluster Safely Between Teams](./k8s-adv-cluster-ops/k8s-multi-tenancy.md)

### GitOps and CI/CD

Covers the GitOps operational model, ArgoCD for continuous delivery, and multi-environment promotion patterns using Kustomize overlays and Helm values files.

- [The GitOps Model — Declarative State in Git](./k8s-adv-gitops/k8s-gitops-model.md)
- [ArgoCD — GitOps Continuous Delivery for Kubernetes](./k8s-adv-gitops/k8s-argocd.md)
- [Multi-Environment Deployments — Promoting Through dev, staging, prod](./k8s-adv-gitops/k8s-multi-env-deployments.md)

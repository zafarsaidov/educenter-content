# Kubernetes Basic

This module introduces Kubernetes to engineers who already understand containers (Docker) but have no Kubernetes experience. It builds understanding from the ground up: how the cluster is structured, how to define and run workloads, how to manage configuration and secrets, how networking works, and how to control resource consumption. Every section focuses on practical usage — writing YAML manifests, applying them to a cluster, and understanding what happens at each step.

## Curriculum

<!-- 8 sections · 32 themes -->

### Cluster Architecture and Concepts

Learn how a Kubernetes cluster is structured, what each component does, and how the object model and reconciliation loop underpin everything Kubernetes does.

- [Container Orchestration and Why Kubernetes Exists](./k8s-basic-cluster-architecture/k8s-container-orchestration.md)
- [Kubernetes Cluster Architecture](./k8s-basic-cluster-architecture/k8s-cluster-architecture.md)
- [The Kubernetes Object Model and Reconciliation Loop](./k8s-basic-cluster-architecture/k8s-object-model.md)

### kubectl and the Kubernetes API

Learn to install and configure kubectl, navigate multiple clusters and namespaces, and understand the difference between imperative commands and declarative manifest management.

- [Installing kubectl and the kubeconfig File](./k8s-basic-kubectl/k8s-kubectl-installation.md)
- [Contexts, Clusters, and Switching Between Environments](./k8s-basic-kubectl/k8s-contexts-clusters.md)
- [Namespaces — Isolation and Resource Scoping](./k8s-basic-kubectl/k8s-namespaces.md)
- [Imperative vs Declarative: kubectl Commands and YAML](./k8s-basic-kubectl/k8s-imperative-vs-declarative.md)

### Core Workloads: Pods and Deployments

Learn the fundamental building blocks of running containerised applications in Kubernetes, from individual pods up to production-ready Deployments with rolling updates and rollbacks.

- [Pods — The Smallest Deployable Unit](./k8s-basic-pods-deployments/k8s-pods.md)
- [Multi-Container Pod Patterns](./k8s-basic-pods-deployments/k8s-multi-container-pods.md)
- [Init Containers — Sequenced Startup Tasks](./k8s-basic-pods-deployments/k8s-init-containers.md)
- [ReplicaSets — Maintaining Desired Pod Count](./k8s-basic-pods-deployments/k8s-replicasets.md)
- [Deployments — Rolling Updates, Rollbacks, and Scaling](./k8s-basic-pods-deployments/k8s-deployments.md)

### Workload Controllers

Explore the specialised Kubernetes controllers for workloads that go beyond a standard Deployment — node-level agents, stateful applications, and scheduled or one-off batch tasks.

- [DaemonSets — One Pod Per Node](./k8s-basic-workload-controllers/k8s-daemonsets.md)
- [StatefulSets — Stateful Applications with Stable Identity](./k8s-basic-workload-controllers/k8s-statefulsets.md)
- [Jobs — Run-to-Completion Batch Tasks](./k8s-basic-workload-controllers/k8s-jobs.md)
- [CronJobs — Scheduled Jobs](./k8s-basic-workload-controllers/k8s-cronjobs.md)

### Configuration: ConfigMaps and Secrets

Learn how to externalise application configuration and manage sensitive data using Kubernetes-native objects.

- [ConfigMaps — Externalising Application Configuration](./k8s-basic-config/k8s-configmaps.md)
- [Secrets — Storing Sensitive Data](./k8s-basic-config/k8s-secrets.md)
- [Environment Variables — Injecting Values into Containers](./k8s-basic-config/k8s-environment-variables.md)

### Resource Management

Learn how to control CPU and memory consumption at the container, pod, and namespace level to ensure stable and fair workload scheduling.

- [Resource Requests and Limits — CPU and Memory](./k8s-basic-resource-management/k8s-resource-requests-limits.md)
- [Quality of Service Classes and Eviction](./k8s-basic-resource-management/k8s-quality-of-service.md)
- [LimitRange — Per-Namespace Resource Constraints](./k8s-basic-resource-management/k8s-limitrange.md)
- [ResourceQuota — Namespace-Level Consumption Limits](./k8s-basic-resource-management/k8s-resourcequota.md)

### Networking and Traffic Routing

Covers the Kubernetes networking model, Service types, DNS-based service discovery, and HTTP traffic routing with Ingress.

- [The Kubernetes Networking Model](./k8s-basic-networking/k8s-networking-model.md)
- [Service Types — Exposing Pods Inside and Outside the Cluster](./k8s-basic-networking/k8s-service-types.md)
- [Kubernetes DNS and Service Discovery](./k8s-basic-networking/k8s-dns.md)
- [Ingress — HTTP/HTTPS Routing to Services](./k8s-basic-networking/k8s-ingress-resources.md)
- [Ingress Controllers — Implementing the Ingress API](./k8s-basic-networking/k8s-ingress-controllers.md)

### Advanced Networking

Covers the Gateway API, NetworkPolicies for traffic isolation, CNI plugin internals, and service mesh fundamentals.

- [Gateway API — The Modern Replacement for Ingress](./k8s-basic-advanced-networking/k8s-gateway-api.md)
- [NetworkPolicies — Controlling Pod-to-Pod Traffic](./k8s-basic-advanced-networking/k8s-network-policies.md)
- [CNI Plugins — How Pod Networking Is Implemented](./k8s-basic-advanced-networking/k8s-cni-plugins.md)
- [Service Mesh — mTLS, Observability, and Traffic Management](./k8s-basic-advanced-networking/k8s-service-mesh.md)

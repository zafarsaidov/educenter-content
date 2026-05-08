# Monitoring Advanced

This module builds on the foundational concepts from the Basic module and takes you into production-grade observability on Kubernetes. You will learn how to deploy and manage a full monitoring stack using the victoria-metrics-k8s-stack Helm chart, configure Grafana unified alerting, implement dashboards as code through Grafana provisioning, cover VictoriaMetrics multi-tenancy and vmauth, and walk through two practical log collection stacks: VictoriaLogs with VLagent, and Loki with Promtail.

## Curriculum

<!-- 7 sections · 19 themes -->

### Monitoring Kubernetes with VictoriaMetrics

Covers the Kubernetes monitoring architecture, deploying the victoria-metrics-k8s-stack Helm chart, using VMServiceScrape and VMPodScrape CRDs, and alerting on common Kubernetes failure modes.

- [Kubernetes Monitoring Architecture](./mon-adv-kubernetes/mon-k8s-monitoring-arch.md)
- [victoria-metrics-k8s-stack Deployment](./mon-adv-kubernetes/mon-vm-k8s-stack.md)
- [Kubernetes Dashboards and Alerts](./mon-adv-kubernetes/mon-k8s-dashboards-alerts.md)

### Grafana Alerting

Covers Grafana's built-in unified alerting system as an alternative to vmalert + Alertmanager: creating alert rules, configuring contact points, and managing notification policies.

- [Grafana Unified Alerting — Rules and Contact Points](./mon-adv-grafana-alerting/mon-grafana-unified-alerting.md)
- [Notification Policies and Silences](./mon-adv-grafana-alerting/mon-grafana-notification-policies.md)

### Dashboards as Code and Provisioning

Covers Grafana provisioning for datasources and dashboards via configuration files, the dashboard JSON model, and storing dashboards in Git with Helm-based deployment.

- [Grafana Provisioning — Datasources and Dashboards](./mon-adv-dashboards-code/mon-grafana-provisioning.md)
- [Dashboards in Git and Helm Deployment](./mon-adv-dashboards-code/mon-dashboards-in-git.md)

### Multi-tenancy and vmauth

Covers VictoriaMetrics cluster multi-tenancy using accountID/projectID, sending and querying tenant-scoped metrics, and securing VictoriaMetrics with vmauth.

- [VictoriaMetrics Multi-tenancy](./mon-adv-multitenancy/mon-vm-multitenancy.md)
- [vmauth — Authentication and Routing Proxy](./mon-adv-multitenancy/mon-vmauth.md)

### Log Management Fundamentals

Covers why logs are the second pillar of observability, structured vs unstructured logs, log cardinality, correlating logs with metrics, and an overview of VictoriaLogs vs Loki.

- [Logs as the Second Pillar](./mon-adv-log-fundamentals/mon-log-fundamentals.md)
- [VictoriaLogs vs Loki](./mon-adv-log-fundamentals/mon-victorialogs-vs-loki.md)

### VictoriaLogs and VLagent

Covers VictoriaLogs architecture, installation, ingestion protocols, configuring VLagent for log collection and processing, the LogsQL query language, and connecting VictoriaLogs to Grafana.

- [VictoriaLogs Architecture and Installation](./mon-adv-victorialogs/mon-victorialogs-architecture.md)
- [VLagent — Log Collection and Pipeline](./mon-adv-victorialogs/mon-vlagent-configuration.md)
- [LogsQL — Querying VictoriaLogs](./mon-adv-victorialogs/mon-logsql.md)
- [VictoriaLogs in Grafana](./mon-adv-victorialogs/mon-victorialogs-grafana.md)

### Loki and Promtail

Covers Loki's architecture, installation with Helm, storage backends, configuring Promtail to collect and process logs, the LogQL query language, and querying Loki in Grafana.

- [Loki Architecture and Installation](./mon-adv-loki/mon-loki-architecture.md)
- [Promtail — Log Collection and Pipeline Stages](./mon-adv-loki/mon-promtail-configuration.md)
- [LogQL — Querying Loki](./mon-adv-loki/mon-logql.md)
- [Loki in Grafana](./mon-adv-loki/mon-loki-grafana.md)

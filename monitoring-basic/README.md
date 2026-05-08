# Monitoring Basic

This module walks you through the complete core observability stack from first principles to production-ready alerting. You will start by understanding the three pillars of observability, then install and configure VictoriaMetrics, learn to scrape metrics at scale with vmagent, instrument your own applications, master MetricsQL for querying and aggregating data, build informative Grafana dashboards, and finally wire up a complete alerting pipeline using vmalert and Alertmanager.

## Curriculum

<!-- 9 sections · 26 themes -->

### Observability and Monitoring Fundamentals

Introduces the three pillars of observability, the differences between monitoring and observability, metric types, label cardinality, and an overview of the VictoriaMetrics + Grafana stack.

- [Observability and Its Three Pillars](./mon-basic-observability/mon-observability-pillars.md)
- [Metric Types and Labels](./mon-basic-observability/mon-metric-types-labels.md)
- [The VictoriaMetrics + Grafana Stack Overview](./mon-basic-observability/mon-stack-overview.md)

### VictoriaMetrics Architecture and Installation

Covers VictoriaMetrics single-node and cluster deployments, core components, installation methods, storage internals, and self-monitoring.

- [VictoriaMetrics vs Prometheus](./mon-basic-victoriametrics/mon-vm-vs-prometheus.md)
- [Single-Node and Cluster Deployment](./mon-basic-victoriametrics/mon-vm-architecture-installation.md)
- [Storage, Retention, and Self-Monitoring](./mon-basic-victoriametrics/mon-vm-storage-configuration.md)

### Scraping Metrics with vmagent

Covers vmagent's architecture, configuring scrape jobs, service discovery, relabeling, and reliable remote_write to VictoriaMetrics.

- [vmagent Overview](./mon-basic-vmagent/mon-vmagent-overview.md)
- [Scrape Jobs and Service Discovery](./mon-basic-vmagent/mon-scrape-service-discovery.md)
- [Relabeling and Remote Write](./mon-basic-vmagent/mon-relabeling-remote-write.md)

### Instrumenting Applications

Covers the Prometheus exposition format, using client libraries to expose metrics from your own code, metric naming conventions, and the key community exporters.

- [Exposition Format and Naming Conventions](./mon-basic-instrumentation/mon-exposition-format.md)
- [Client Libraries — Exposing Metrics from Code](./mon-basic-instrumentation/mon-client-libraries.md)
- [Exporters](./mon-basic-instrumentation/mon-exporters.md)

### MetricsQL: Querying VictoriaMetrics

Teaches the MetricsQL query language: the difference from PromQL, filtering by labels, aggregation operators, key functions, and building real-world queries using the RED method.

- [MetricsQL Basics — Selectors and Filtering](./mon-basic-metricsql/mon-metricsql-basics.md)
- [Aggregations and Functions](./mon-basic-metricsql/mon-metricsql-aggregations.md)
- [Practical Queries — RED Method](./mon-basic-metricsql/mon-metricsql-practical.md)

### Grafana: Installation and Datasources

Covers installing Grafana, connecting VictoriaMetrics as a datasource, exploring metrics with the Explore view, and configuring user access.

- [Installing Grafana and Adding a Datasource](./mon-basic-grafana-install/mon-grafana-installation.md)
- [Users, Organisations, and Permissions](./mon-basic-grafana-install/mon-grafana-users-permissions.md)

### Building Dashboards in Grafana

Covers dashboard structure, all major panel types, template variables for interactive filtering, annotations, and importing community dashboards.

- [Dashboard Structure and Panel Types](./mon-basic-grafana-dashboards/mon-dashboard-structure.md)
- [Template Variables and Drill-Down](./mon-basic-grafana-dashboards/mon-dashboard-variables.md)
- [Annotations, Links, and Community Dashboards](./mon-basic-grafana-dashboards/mon-dashboard-annotations.md)

### Alerting with vmalert

Covers vmalert's role in the alerting pipeline, writing alerting and recording rules, understanding alert states, and connecting vmalert to Alertmanager.

- [vmalert Architecture and Setup](./mon-basic-vmalert/mon-vmalert-architecture.md)
- [Writing Alerting Rules](./mon-basic-vmalert/mon-alerting-rules.md)
- [Recording Rules and Alertmanager Integration](./mon-basic-vmalert/mon-recording-rules-alertmanager.md)

### Alertmanager: Routing and Notifications

Covers Alertmanager's architecture, routing alerts to the right receiver, grouping and deduplication, inhibition, and configuring Telegram/Slack/email receivers.

- [Alertmanager Architecture and Installation](./mon-basic-alertmanager/mon-alertmanager-architecture.md)
- [Routing and Receivers](./mon-basic-alertmanager/mon-alertmanager-routing.md)
- [Grouping, Inhibition, and Silences](./mon-basic-alertmanager/mon-alertmanager-grouping.md)

# Custom Perses Dashboards for OpenShift Observability

This repository contains a collection of custom Perses dashboards designed for monitoring and observability of OpenShift clusters managed through Red Hat Advanced Cluster Management (RHACM). These dashboards provide comprehensive insights into cluster health, resource utilization, application performance, and operational metrics.

See [SETUP.md](SETUP.md) for the full deployment procedure (RHACM MCO, the multicluster observability add-on, and the Cluster Observability Operator).

## Overview

The dashboards (`perses-dashboards/`) are deployed as `PersesDashboard` CRDs in the `openshift-cluster-observability-operator` namespace and are automatically discovered by Perses/COO's console plugin.

`original-grafana-dashboards/` holds the pre-migration Grafana dashboard sources (as `ConfigMap`s) that `perses-dashboards/` were migrated from, kept for reference.

## Dashboard Collection

- **Cluster - Operators** (`operator-overview.yaml`)
  - One row per `ClusterOperator`, for the selected cluster
  - Status derived from `cluster_operator_conditions` (Available / Progressing / Degraded, and the combined states)
  - Faithful migration of the original Grafana "Cluster - Operators" dashboard

- **ACM - Cluster Health Overview** (`acm-clusters-overview.yaml`)
  - Pick one cluster from a dropdown, drill into its health in detail
  - Core Platform (API server, etcd, controller manager, scheduler, ClusterOperators), Nodes (Ready + Pressure, one row per node), Workloads (one row per namespace, includes `CrashLoopBackOff` detection), and Critical Alerts
  - Not a migration of an original Grafana dashboard — a new single-cluster incident-triage design built during the MCOA migration

- **ACM - All Clusters Overview** (`acm-fleet-overview.yaml`)
  - One row per managed cluster, fleet-wide (no cluster picker) — for comparing the whole fleet at a glance rather than drilling into one cluster
  - Node counts, control plane health, standard OpenShift namespace health, ClusterOperators, critical alerts, OpenShift version, and (where installed) ArgoCD/Dynatrace/Sumologic/CSI-driver/GitOps health
  - Migrated from the original Grafana "ACM - All Clusters Overview" / "Cluster Overview" dashboard (`original-grafana-dashboards/custom-overview.yaml`)

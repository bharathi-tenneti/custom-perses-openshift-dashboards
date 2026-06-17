# Custom Perses Dashboards for OpenShift Observability

This repository contains a collection of custom Perses dashboards designed for monitoring and observability of OpenShift clusters managed through Red Hat Advanced Cluster Management (RHACM). These dashboards provide comprehensive insights into cluster health, resource utilization, application performance, and operational metrics.

## Overview

The dashboards are deployed as Kubernetes CRDs in the `open-cluster-management-observability` namespace and are automatically discovered by Perses

## Dashboard Collection

# Cluster-Operator-overview
 **Cluster Operators Overview** (`operator-overview.yaml`)
  - OpenShift cluster operator status monitoring
  - Visual representation of operator health (Available, Progressing, Degraded)

- **Custom Cluster Overview** (`custom-overview.yaml`)
  - Comprehensive cluster health monitoring
  - Node status, resource utilization, and component health
  - Master nodes, infra nodes, and worker node status
  - Kubernetes control plane components (ETCD, API server, scheduler)
  - OpenShift-specific components (console, OAuth, DNS)

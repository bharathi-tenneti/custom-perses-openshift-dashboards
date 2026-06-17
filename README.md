# Custom Perses Dashboards for OpenShift Observability

This repository contains a collection of custom Perses dashboards designed for monitoring and observability of OpenShift clusters managed through Red Hat Advanced Cluster Management (RHACM). These dashboards provide comprehensive insights into cluster health, resource utilization, application performance, and operational metrics.

## Overview

The dashboards are deployed as Kubernetes CRDs in the `open-cluster-management-observability` namespace and are automatically discovered by Perses

## Dashboard Collection

# Cluster-Operator-overview
 **Cluster Operators Overview** (`operator-overview.yaml`)
  - OpenShift cluster operator status monitoring
  - Visual representation of operator health (Available, Progressing, Degraded)



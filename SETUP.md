# Tutorial: Setting Up Multi cluster Perses Dashboards on OpenShift via RHACM and Cluster Observability Operator(COO)

This dashboard shows fleet-wide cluster health across `local-cluster` and every managed cluster,
using RHACM's multicluster observability add-on (MCOA) as the metrics backend, rendered in Perses
via the Cluster Observability Operator (COO).

MCOA's Perses integration is Technology Preview. Validated end-to-end against an RHACM 2.17-class
hub with two managed clusters.

---

## Phase 1 — Install RHACM MultiCluster Observability (MCO)

This sets up the federated Thanos backend that collects metrics from all managed clusters.

**1.1  Create the observability namespace**
```bash
oc create namespace open-cluster-management-observability
```

**1.2  Create the S3 object storage bucket**

Save as `obc.yaml` and apply:
```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: acm-thanos-bucket
  namespace: open-cluster-management-observability
spec:
  generateBucketName: acm-thanos
  storageClassName: openshift-storage.noobaa.io
```
```bash
oc apply -f obc.yaml
```

**1.3  Create the `thanos-object-storage` secret** (populates S3 credentials from the OBC):
```bash
oc patch secret thanos-object-storage -n open-cluster-management-observability --type=merge \
  -p "{\"stringData\":{\"thanos.yaml\":\"type: S3\nconfig:\n  bucket: $(oc get configmap acm-thanos-bucket -n open-cluster-management-observability -o jsonpath='{.data.BUCKET_NAME}')\n  endpoint: s3.openshift-storage.svc.cluster.local\n  insecure: true\n  access_key: $(oc get secret acm-thanos-bucket -n open-cluster-management-observability -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)\n  secret_key: $(oc get secret acm-thanos-bucket -n open-cluster-management-observability -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)\n\"}}" \
2>/dev/null || oc create secret generic thanos-object-storage \
  -n open-cluster-management-observability \
  --from-literal=thanos.yaml="type: S3
config:
  bucket: $(oc get configmap acm-thanos-bucket -n open-cluster-management-observability -o jsonpath='{.data.BUCKET_NAME}')
  endpoint: s3.openshift-storage.svc.cluster.local
  insecure: true
  access_key: $(oc get secret acm-thanos-bucket -n open-cluster-management-observability -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
  secret_key: $(oc get secret acm-thanos-bucket -n open-cluster-management-observability -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)"
```

**1.4  Copy the global pull secret** (so MCO can pull Thanos/Grafana images):
```bash
oc get secret pull-secret -n openshift-config -o json \
  | sed 's/"namespace": "openshift-config"/"namespace": "open-cluster-management-observability"/g' \
  | sed 's/"name": "pull-secret"/"name": "multiclusterhub-operator-pull-secret"/g' \
  | oc apply -f -
```

**1.5  Apply the MCO instance** (save as `mco-instance.yaml`):
```yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  availabilityConfig: Basic
  observabilityAddonSpec:
    enableMetrics: true
    interval: 300
  storageConfig:
    metricObjectStorage:
      name: thanos-object-storage
      key: thanos.yaml
    storageClass: ocs-storagecluster-ceph-rbd
```
```bash
oc apply -f mco-instance.yaml
```

**1.6 Force-scale for SNO** (SNO cluster — skip if multi-node):
```bash
oc scale statefulset observability-thanos-receive-default -n open-cluster-management-observability --replicas=1
oc scale statefulset observability-alertmanager -n open-cluster-management-observability --replicas=1
oc scale statefulset observability-thanos-rule -n open-cluster-management-observability --replicas=1
oc scale statefulset observability-thanos-query-frontend-memcached -n open-cluster-management-observability --replicas=1
oc scale statefulset observability-thanos-store-memcached -n open-cluster-management-observability --replicas=1
oc scale deployment observability-rbac-query-proxy -n open-cluster-management-observability --replicas=1
```

---

## Phase 2 — Install the Cluster Observability Operator (COO)

This operator manages Perses and renders dashboards in the OpenShift Console. MCOA's Perses
integration (Phase 4) requires this already installed — it does not install COO itself.

Save as [base/coo-operator-perses.yaml](base/coo-operator-perses.yaml) and apply:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cluster-observability-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-observability-operator-group
  namespace: openshift-cluster-observability-operator
spec:
  targetNamespaces: []   
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
spec:
  channel: stable
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```
```bash
oc apply -f base/coo-operator-perses.yaml
```

Wait for the operator pods to become Ready before proceeding:
```bash
oc wait --for=condition=Ready pod -l name=cluster-observability-operator \
  -n openshift-cluster-observability-operator --timeout=120s
```


## Phase 3 — Enable the Multicluster Observability Add-on (MCOA)

Enabling `capabilities.platform.metrics` on the `MultiClusterObservability` CR from Phase 1 causes
the operator to **stop deploying the classic `metrics-collector`** to every managed cluster and
instead deploy `multicluster-observability-addon-manager`, which rolls out `PrometheusAgent`-based
collectors instead (`prom-agent-platform-metrics-collector-0` /
`prom-agent-user-workload-metrics-collector-0`, in each managed cluster's
`open-cluster-management-agent-addon` namespace — or directly in
`open-cluster-management-observability` for the hub monitoring itself as `local-cluster`).

```bash
oc patch mco observability -n open-cluster-management-observability --type=merge -p \
  '{"spec":{"capabilities":{"platform":{"metrics":{"default":{"enabled":true}}},"userWorkloads":{"metrics":{"default":{"enabled":true}}}}}}'
```

Verify the cutover (allow a minute or two):
```bash
oc get cma multicluster-observability-addon
oc get prometheusagents -n open-cluster-management-observability
oc get pods -n open-cluster-management-agent-addon | grep prom-agent
```

---

## Phase 4 — Enable Perses Dashboards via MCOA

```bash
oc patch mco observability -n open-cluster-management-observability --type=merge -p \
  '{"spec":{"capabilities":{"platform":{"metrics":{"default":{"enabled":true},"ui":{"enabled":true}}}}}}'
```

Verify:
```bash
oc get persesdatasources -n open-cluster-management-observability
oc get uiplugin monitoring -o yaml
oc get persesdashboards -n open-cluster-management-observability
```

---

## Phase 5 — Apply the Fleet Datasource, Custom Metrics, and Dashboards

`base/` covers the rest in one shot:

- [base/dynamic-fleet-thanos-datasource.yaml](base/dynamic-fleet-thanos-datasource.yaml) —
  a `PersesGlobalDatasource` pointed at the same `rbac-query-proxy:8080` endpoint MCOA uses
  internally, kept **cluster-wide** (unlike MCOA's own project-scoped one) so this repo's
  dashboards — which live in `openshift-cluster-observability-operator`, a different namespace —
  can reference it.
- [base/acm-clusters-overview-custom-metrics-scrapeconfig.yaml](base/acm-clusters-overview-custom-metrics-scrapeconfig.yaml) —
  federates the two metrics the Workloads panel needs that aren't in any default `ScrapeConfig`:
  `kube_pod_status_ready` and `kube_pod_container_status_waiting_reason`.
- `perses-dashboards/` — the dashboards themselves (`acm-clusters-overview.yaml`,
  `operator-overview.yaml`), referenced unchanged via `../perses-dashboards/`.

```bash
oc apply -k base
```

**This alone is not enough** — the `ScrapeConfig` above has to also be referenced in the
`multicluster-observability-addon` `ClusterManagementAddOn`'s placement, or nothing federates it.
That's an in-place append to a list of configs otherwise fully managed by the operator, which
isn't safe to express as a static `kustomize` resource (a plain `apply` would fight the operator
over the rest of the list). Append it with a JSON patch instead:

```bash
oc patch cma multicluster-observability-addon --type=json -p \
  '[{"op":"add","path":"/spec/installStrategy/placements/0/configs/-","value":{"group":"monitoring.rhobs","name":"acm-clusters-overview-custom-metrics","namespace":"open-cluster-management-observability","resource":"scrapeconfigs"}}]'
```

Verify the two custom metrics start flowing (allow up to 5 minutes — the default federate interval):
```bash
oc get cma multicluster-observability-addon -o yaml | yq '.spec.installStrategy.placements[0].configs'
# once landed, from a thanos-query pod in open-cluster-management-observability:
#   curl -sG http://localhost:9090/api/v1/query --data-urlencode 'query=count by (cluster) (kube_pod_status_ready)'
```

`kube_pod_container_status_waiting_reason` will only show data while something is actually
`CrashLoopBackOff` (or another waiting-reason state) somewhere in the fleet — an empty result on
its own isn't a failure, unlike `kube_pod_status_ready`, which should always have data as soon as
any pod exists anywhere.

Verify the dashboards are live:
```bash
oc get persesdashboards -n openshift-cluster-observability-operator
```

The dashboards are visible in the OpenShift Console under **Fleet Managementt -> Observe → Dashboards**, project
`openshift-cluster-observability-operator`.

---

## Summary of Resources Created

| Kind | Name | Namespace | Purpose |
|------|------|-----------|---------|
| `MultiClusterObservability` | `observability` | cluster-scoped | RHACM's fleet Thanos backend + MCOA capabilities |
| `Subscription` | `cluster-observability-operator` | `openshift-cluster-observability-operator` | Installs COO from OLM |
| `PersesGlobalDatasource` | `dynamic-fleet-thanos-datasource` | cluster-scoped | Fleet Thanos endpoint for this repo's dashboards, no credential needed |
| `ScrapeConfig` | `acm-clusters-overview-custom-metrics` | `open-cluster-management-observability` | Federates the two custom metrics the Workloads panel needs |
| `PersesDashboard` | `acm-clusters-overview`, `my-new-perses-dashboard` | `openshift-cluster-observability-operator` | The dashboards themselves |

Auto-created by MCOA once Phases 3–4 are enabled (not authored in this repo):

| Kind | Name | Namespace | Purpose |
|------|------|-----------|---------|
| `ClusterManagementAddOn` | `multicluster-observability-addon` | cluster-scoped | Manages the PrometheusAgent-based collectors per managed cluster |
| `PrometheusAgent` | `mcoa-default-platform-metrics-collector-global`, `mcoa-default-user-workload-metrics-collector-global` | `open-cluster-management-observability` | Replaces the classic `metrics-collector` |
| `PersesDatasource` | `rbac-query-proxy-datasource` | `open-cluster-management-observability` | MCOA's own project-scoped fleet datasource |
| `UIPlugin` | `monitoring` | cluster-scoped | Enables Perses + ACM alerting in the Console |

---

## Known Gap

Nothing in MCOA's auto-created RBAC grants read on `managedclusters`
(`cluster.open-cluster-management.io`) or `multiclusterobservabilities`
(`observability.open-cluster-management.io`) to the Perses service account — only `namespaces` and
Perses's own CRDs (`persesdashboards`, `persesdatasources`, `persesglobaldatasources`). This hasn't
caused a failure in testing (the dashboards in this repo don't need it — their cluster picker uses
a `PrometheusLabelValuesVariable`, not the Kubernetes API), but if the console's ACM alerting
perspective misbehaves, this is the first thing to check. There is no official Red Hat
documentation for a fix — confirmed by full-text search of the RHACM 2.17 Observability guide (90
pages, no mention of either resource beyond the `MultiClusterObservability` CR itself).

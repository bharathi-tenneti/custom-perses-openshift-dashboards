# Tutorial: Setting Up `my-new-perses-dashboard` on OpenShift

This dashboard shows `cluster_operator_conditions` across `local-cluster` and `cluster2` using RHACM's Thanos as the data source, rendered in Perses via the Cluster Observability Operator.

---

## Phase 1 — Install RHACM MultiCluster Observability (MCO)

This sets up the federated Thanos backend that collects metrics from all managed clusters.

**1.1 — Create the observability namespace**
```bash
oc create namespace open-cluster-management-observability
```

**1.2 — Create the S3 object storage bucket**

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

**1.3 — Create the `thanos-object-storage` secret** (populates S3 credentials from the OBC):
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

**1.4 — Copy the global pull secret** (so MCO can pull Thanos/Grafana images):
```bash
oc get secret pull-secret -n openshift-config -o json \
  | sed 's/"namespace": "openshift-config"/"namespace": "open-cluster-management-observability"/g' \
  | sed 's/"name": "pull-secret"/"name": "multiclusterhub-operator-pull-secret"/g' \
  | oc apply -f -
```

**1.5 — Apply the MCO instance** (save as `mco-instance.yaml`):
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
  storageClassName: ocs-storagecluster-ceph-rbd
```
```bash
oc apply -f mco-instance.yaml
```

**1.6 — Force-scale for SNO** (single-node cluster — skip if multi-node):
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

This operator manages Perses and the `UIPlugin` that integrates it into the OpenShift Console.

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
  targetNamespaces: []   # AllNamespaces mode
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

---

## Phase 3 — Enable the Perses UI Plugin

This `UIPlugin` tells COO to deploy the Perses instance and wire it into the OpenShift Console's Observe tab. It also enables the ACM multi-cluster alertmanager/Thanos integration in the UI.

Save as [base/perses-ui.yaml](base/perses-ui.yaml) and apply:
```yaml
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  name: monitoring
spec:
  type: Monitoring
  monitoring:
    perses:
      enabled: true
    acm:
      enabled: true
      thanosQuerier:
        url: 'http://observability-thanos-query.open-cluster-management-observability.svc.cluster.local:9090'
      alertmanager:
        url: 'http://observability-alertmanager.open-cluster-management-observability.svc.cluster.local:9093'
```
```bash
oc apply -f base/perses-ui.yaml
```

COO automatically creates:
- The `Perses` CR in `openshift-cluster-observability-operator`
- A default `accelerators-thanos-querier-datasource` PersesDatasource (local cluster Prometheus)
- TLS secrets and the `perses` TLS service

---

## Phase 4 — Set Up the Fleet Datasource (RBAC + Secret + PersesDatasource)

This is the critical piece: it creates a service account with a long-lived token that Perses uses to authenticate against RHACM's Thanos `rbac-query-proxy` to get cross-cluster metrics.

Save as [base/perses-fleet-datasource.yaml](base/perses-fleet-datasource.yaml) and apply:

```yaml
# 1. ServiceAccount that will authenticate against Thanos
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dynamic-fleet-thanos-sa
  namespace: openshift-cluster-observability-operator
---
# 2. Long-lived SA token secret (OpenShift populates the token automatically)
apiVersion: v1
kind: Secret
metadata:
  name: dynamic-fleet-thanos-secret
  namespace: openshift-cluster-observability-operator
  annotations:
    kubernetes.io/service-account.name: dynamic-fleet-thanos-sa
type: kubernetes.io/service-account-token
---
# 3. Give the SA permission to query cluster monitoring (needed to reach Thanos)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dynamic-fleet-thanos-sa-reader
subjects:
  - kind: ServiceAccount
    name: dynamic-fleet-thanos-sa
    namespace: openshift-cluster-observability-operator
roleRef:
  kind: ClusterRole
  name: cluster-monitoring-view
  apiGroup: rbac.authorization.k8s.io
---
# 4. PersesDatasource — the fleet-wide Thanos endpoint (with mTLS via internal CA)
apiVersion: perses.dev/v1alpha2
kind: PersesDatasource
metadata:
  name: dynamic-fleet-thanos-datasource
  namespace: openshift-cluster-observability-operator
spec:
  client:
    tls:
      enable: true
      caCert:
        type: file
        certPath: /ca/service-ca.crt
  config:
    default: false
    display:
      name: "ACM MultiCluster Thanos Engine"
    plugin:
      kind: PrometheusDatasource
      spec:
        proxy:
          kind: HTTPProxy
          spec:
            secret: dynamic-fleet-thanos-secret
            url: 'https://observability-thanos-query-frontend.open-cluster-management-observability.svc.cluster.local:9092'
```
```bash
oc apply -f base/perses-fleet-datasource.yaml
```

> **Note:** The Perses operator reconciles the datasource and may rewrite the proxy secret name and URL internally (it creates a copy named `dynamic-fleet-thanos-datasource-secret` and may route through `rbac-query-proxy:8443`). This is expected.

---

## Phase 5 — Grant Perses Permission to Read the Token Secret

By default, the `perses-sa` ServiceAccount cannot read secrets in the namespace. This Role+RoleBinding unlocks that so Perses can read `dynamic-fleet-thanos-secret` to pass the OAuth bearer token upstream to Thanos.

```bash
oc apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: perses-secret-reader
  namespace: openshift-cluster-observability-operator
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: perses-secret-reader
  namespace: openshift-cluster-observability-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: perses-secret-reader
subjects:
- kind: ServiceAccount
  name: perses-sa
  namespace: openshift-cluster-observability-operator
EOF
```

---

## Phase 6 — Grant Perses Permission to Read Observability CRDs

This ClusterRole+ClusterRoleBinding lets Perses list `PersesDatasources`, `ManagedClusters`, and `MultiClusterObservabilities` — needed for the ACM multi-cluster integration to work in the UI.

```bash
oc apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: perses-custom-observability-reader
rules:
- apiGroups: ["perses.dev"]
  resources: ["persesglobaldatasources", "persesdatasources"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["cluster.open-cluster-management.io"]
  resources: ["managedclusters"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["observability.open-cluster-management.io"]
  resources: ["multiclusterobservabilities", "endpoints"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: perses-custom-observability-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: perses-custom-observability-reader
subjects:
- kind: ServiceAccount
  name: perses
  namespace: openshift-cluster-observability-operator
- kind: ServiceAccount
  name: perses-sa
  namespace: openshift-cluster-observability-operator
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
EOF
```

---

## Phase 7 — Apply the Dashboards

The dashboard defines two `StatChart` panels — `cluster_operator_conditions` from the `dynamic-fleet-thanos-datasource`. It also has a `TextVariable` for the console base URL used in panel deep-links.

```bash
oc apply -f base/kustomize.yaml
```


Verify it is live:
```bash
oc get persesdashboards -n openshift-cluster-observability-operator
```

---

## Summary of Resources Created

| Kind | Name | Namespace | Purpose |
|------|------|-----------|---------|
| `Subscription` | `cluster-observability-operator` | `openshift-cluster-observability-operator` | Installs COO from OLM |
| `UIPlugin` | `monitoring` | cluster-scoped | Enables Perses in Console + ACM integration |
| `Perses` | `perses` | `openshift-cluster-observability-operator` | Auto-created by COO from UIPlugin |
| `ServiceAccount` | `dynamic-fleet-thanos-sa` | `openshift-cluster-observability-operator` | Identity for Thanos auth |
| `Secret` | `dynamic-fleet-thanos-secret` | `openshift-cluster-observability-operator` | Long-lived SA token for Thanos OAuth |
| `ClusterRoleBinding` | `dynamic-fleet-thanos-sa-reader` | cluster | Grants `cluster-monitoring-view` to the SA |
| `PersesDatasource` | `dynamic-fleet-thanos-datasource` | `openshift-cluster-observability-operator` | Fleet Thanos endpoint for dashboards |
| `Role` | `perses-secret-reader` | `openshift-cluster-observability-operator` | Lets Perses SA read secrets |
| `RoleBinding` | `perses-secret-reader` | `openshift-cluster-observability-operator` | Binds secret-reader role to `perses-sa` |
| `ClusterRole` | `perses-custom-observability-reader` | cluster | Lets Perses read MCO/ACM CRDs |
| `ClusterRoleBinding` | `perses-custom-observability-binding` | cluster | Binds above to Perses SAs + authenticated users |
| `PersesDashboard` | `openshift-cluster-observability-operator` | The dashboard itself |

The dashboards are visible in the OpenShift Console under **RHACM -> Observe -> Dashboards** once all resources are in place.

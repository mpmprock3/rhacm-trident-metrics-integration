# Integrating NetApp Trident with RHACM Observability

## Overview

By default, RHACM only collects core OpenShift platform metrics. To view NetApp Trident metrics on the Hub cluster's Grafana, you must:

1. Enable User Workload Monitoring
2. Explicitly allow the Trident metrics to traverse the network
3. Inject a custom dashboard into the RHACM Grafana 

---

## Phase 1: Enable Metrics Collection (Run on the MANAGED Cluster)

Execute these steps on the cluster where Trident is installed.

### 1. Enable User Workload Monitoring

OpenShift must be configured to scrape user namespaces.

```bash
oc edit cm cluster-monitoring-config -n openshift-monitoring
```

Ensure the `config.yaml` block includes `enableUserWorkload: true`:

```yaml
data:
  config.yaml: |
    enableUserWorkload: true
```

### 2. Create the Trident ServiceMonitor

This tells the local OpenShift Prometheus to scrape the Trident controller.

```bash
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: trident-sm
  namespace: trident
  labels:
    release: prometheus-operator
spec:
  jobLabel: trident
  selector:
    matchLabels:
      app: controller.csi.trident.netapp.io
  namespaceSelector:
    matchNames:
      - trident
  endpoints:
    - port: metrics
      interval: 30s
EOF
```

### 3. Create the RHACM Metrics Allowlist

This is the **most critical step**. You must deploy a ConfigMap directly into the `trident` namespace to tell the RHACM User Workload (UWL) Collector that these specific metrics are allowed to be sent to the Hub.

```bash
cat <<EOF | oc apply -f -
kind: ConfigMap
apiVersion: v1
metadata:
  name: observability-metrics-custom-allowlist
  namespace: trident
data:
  uwl_metrics_list.yaml: |
    names:
      - trident_build_info
      - trident_version
      - trident_node_count
      - trident_backend_count
      - trident_storageclass_count
      - trident_volume_count
      - trident_snapshot_count
      - trident_volume_total_bytes
      - trident_volume_allocated_bytes
      - trident_backend_info
      - trident_backend_allocated_bytes
      - trident_backend_provisioned_bytes
      - trident_backend_used_bytes
      - trident_operation_duration_milliseconds_sum
      - trident_operation_duration_milliseconds_count
EOF
```

### 4. Restart the UWL Collector

Force the RHACM agent to immediately download the new allowlist.

```bash
oc delete pod -l app=uwl-metrics-collector -n open-cluster-management-addon-observability
```

> Wait 3 minutes for the pod to restart and begin shipping data to the Hub.

---

## Phase 2: Deploy the Grafana Dashboard (Run on the HUB Cluster)

Execute these steps on your central RHACM Hub cluster.

### 1. Download the Raw Dashboard JSON

Download the official NetApp Trident dashboard for Kubernetes:

```bash
curl -sL "https://raw.githubusercontent.com/YvosOnTheHub/LabNetApp/master/Kubernetes_v6/Trident_Scenarios/Scenario03/2_Grafana/Dashboards/Trident_Dashboard_24.06.json" > Trident_Raw.json
```

### 2. Patch the Datasource UID

Standard Grafana dashboards look for a database named `"prometheus"`. RHACM uses a database named `"Observatorium"`. You must patch the JSON file to prevent "Datasource not found" errors.

```bash
sed 's/"uid": "prometheus"/"uid": "Observatorium"/g' Trident_Raw.json > Trident_Ready.json
```

### 3. Create the Dashboard ConfigMap

Wrap the JSON file into a ConfigMap in the RHACM observability namespace:

```bash
oc create configmap trident-dashboard \
  -n open-cluster-management-observability \
  --from-file=trident-dashboard.json=Trident_Ready.json
```

### 4. Apply the RHACM Dashboard Label

**Important:** Do not use the standard upstream Grafana label. You must use RHACM's proprietary custom dashboard label, otherwise the loader will ignore the file.

```bash
oc label cm trident-dashboard -n open-cluster-management-observability \
  grafana-custom-dashboard="true"
```

---

## Phase 3: Verification

### Verify Metrics Arrival

1. Open the Hub Cluster OpenShift Web Console.
2. Navigate to **Observe -> Metrics**.
3. Query `trident_build_info`. If a graph appears, the data pipeline is fully operational.

### Verify the Dashboard

1. Open the RHACM Grafana UI.
2. Search for **Trident Global Data**.
3. Ensure no panels have red triangles and the charts are populated with data.

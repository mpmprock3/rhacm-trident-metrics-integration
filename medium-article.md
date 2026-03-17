# How to Monitor NetApp Trident Storage Across Clusters Using RHACM Observability

## The visibility gap nobody warns you about

You've deployed NetApp Trident on your managed OpenShift clusters. You've set up Red Hat Advanced Cluster Management (RHACM) Observability to get a single pane of glass for your fleet. You open Grafana on your Hub cluster expecting storage metrics — and find nothing.

This isn't a bug. It's by design.

RHACM Observability only collects core OpenShift platform metrics out of the box. Trident metrics — volume counts, backend capacity, provisioning trends — live in user workload territory, and RHACM deliberately ignores them unless you explicitly opt in.

In this article, I'll walk you through the exact steps to bridge that gap: getting Trident metrics flowing from your managed clusters to the Hub's Grafana, complete with a production-ready dashboard.

---

## Architecture: How the Data Flows

Before we start configuring, it helps to understand the pipeline:

```
Trident Controller (managed cluster)
    ↓  scraped by
OpenShift Prometheus (user workload)
    ↓  forwarded by
RHACM UWL Metrics Collector
    ↓  shipped to
Observatorium (Hub cluster)
    ↓  visualized in
Grafana (Hub cluster)
```

Each arrow represents a handoff that must be explicitly configured. Miss one, and the chain breaks silently.

---

## Prerequisites

- An OpenShift managed cluster with NetApp Trident installed (v24.06+)
- RHACM Observability enabled on the Hub cluster
- `oc` CLI access to both the managed and Hub clusters
- Cluster admin privileges on both clusters

---

## Phase 1: Enable Metrics Collection on the Managed Cluster

All steps in this phase run on the cluster where Trident is installed.

### Step 1: Enable User Workload Monitoring

OpenShift ships with monitoring for its own components, but user namespaces like `trident` are not scraped by default. You need to flip a single flag.

Edit the cluster monitoring config:

```bash
oc edit cm cluster-monitoring-config -n openshift-monitoring
```

Ensure the `config.yaml` data block includes:

```yaml
data:
  config.yaml: |
    enableUserWorkload: true
```

This tells OpenShift to deploy an additional Prometheus instance dedicated to scraping user workloads. If you already have this enabled for other applications, you can skip this step.

### Step 2: Create a ServiceMonitor for Trident

With user workload monitoring active, you need to tell Prometheus *what* to scrape. Trident's CSI controller exposes a `/metrics` endpoint, but Prometheus won't discover it without a `ServiceMonitor`.

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

You can verify this works locally before moving forward:

```bash
oc -n openshift-user-workload-monitoring port-forward svc/prometheus-operated 9090:9090
```

Then open `http://localhost:9090` and query `trident_build_info`. If you get results, Prometheus is scraping Trident successfully.

### Step 3: Configure the RHACM Metrics Allowlist

This is the step that trips up most people.

RHACM's observability addon runs a component called the **User Workload (UWL) Metrics Collector** on each managed cluster. It watches for user workload metrics, but it won't forward anything to the Hub unless you explicitly whitelist the metric names.

The mechanism is a ConfigMap deployed **in the same namespace as the workload** — in this case, `trident`:

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

A few things to note about this ConfigMap:

- The name **must** be `observability-metrics-custom-allowlist`. The UWL collector looks for this exact name.
- It must live in the **workload's namespace** (`trident`), not in `openshift-monitoring` or the RHACM addon namespace.
- The key must be `uwl_metrics_list.yaml` with a `names` list underneath.

If you need additional metrics later (for example, `kubelet_volume_stats_capacity_bytes` for PVC-level capacity data), just add them to this list and restart the collector.

### Step 4: Restart the UWL Collector

The collector doesn't watch the ConfigMap for live changes. You need to bounce it:

```bash
oc delete pod -l app=uwl-metrics-collector \
  -n open-cluster-management-addon-observability
```

The Deployment controller will recreate the pod. Give it about 2–3 minutes to start shipping data to the Hub.

---

## Phase 2: Deploy the Dashboard on the Hub Cluster

All steps in this phase run on your RHACM Hub cluster.

### Step 1: Get the Dashboard JSON

NetApp maintains a community Grafana dashboard for Trident. Download it:

```bash
curl -sL "https://raw.githubusercontent.com/YvosOnTheHub/LabNetApp/refs/heads/master/Kubernetes_v6/Trident_Scenarios/Scenario03/2_Grafana/Dashboards/Trident_Dashboard_24.06.json" \
  > Trident_Raw.json
```

### Step 2: Patch the Datasource UID

Here's a subtle but critical detail: standard Grafana dashboards reference a datasource with `"uid": "prometheus"`. RHACM's Grafana uses a datasource called `Observatorium` (backed by Thanos). If you deploy the dashboard as-is, every panel will show a "Datasource not found" error.

Fix it with a single `sed`:

```bash
sed 's/"uid": "prometheus"/"uid": "Observatorium"/g' \
  Trident_Raw.json > Trident_Ready.json
```

This replaces all 16 datasource references in the dashboard JSON.

### Step 3: Create the Dashboard ConfigMap

RHACM loads custom dashboards from ConfigMaps in the `open-cluster-management-observability` namespace:

```bash
oc create configmap trident-dashboard \
  -n open-cluster-management-observability \
  --from-file=trident-dashboard.json=Trident_Ready.json
```

### Step 4: Label it for Discovery

RHACM's dashboard loader only picks up ConfigMaps with a specific label. This is **not** the standard Grafana `grafana_dashboard: "1"` label — it's RHACM-specific:

```bash
oc label cm trident-dashboard \
  -n open-cluster-management-observability \
  grafana-custom-dashboard="true"
```

Without this label, the ConfigMap will sit there doing nothing.

---

## Phase 3: Verify the Integration

### Check Metrics Arrival

Open the Hub cluster's OpenShift Web Console, navigate to **Observe → Metrics**, and run this query:

```promql
trident_build_info
```

If you see results with labels like `trident_version` and `backend_uuid`, the entire pipeline — scraping, allowlisting, forwarding, and storage in Observatorium — is working.

### Check the Dashboard

Open the RHACM Grafana UI and search for **"Trident Global Data"**. You should see:

- **Stat panels** showing Trident version, node count, backend count, PVC count, and snapshot count
- **Pie charts** breaking down capacity allocation across backend types (ONTAP NAS, ONTAP SAN, etc.)
- **Time series graphs** showing provisioning activity trends
- **A bar chart** ranking your top 10 backends by allocated capacity

If any panel shows "No data," verify that the specific metric it queries is included in your allowlist ConfigMap from Phase 1.

---

## Troubleshooting

### Metrics exist locally but don't appear on the Hub

- Confirm the allowlist ConfigMap name is exactly `observability-metrics-custom-allowlist`
- Confirm it's in the `trident` namespace, not elsewhere
- Check the UWL collector logs:

```bash
oc logs -l app=uwl-metrics-collector \
  -n open-cluster-management-addon-observability --tail=50
```

### Dashboard panels show "Datasource not found"

- Verify the datasource UID was patched. Check the JSON:

```bash
grep '"uid"' Trident_Ready.json | head -5
```

All entries should show `"uid": "Observatorium"`, not `"uid": "prometheus"`.

### Dashboard doesn't appear in Grafana

- Confirm the ConfigMap has the label `grafana-custom-dashboard=true`:

```bash
oc get cm trident-dashboard \
  -n open-cluster-management-observability --show-labels
```

---

## What You Get

Once everything is wired up, you have centralized storage observability across your entire OpenShift fleet. From the Hub cluster's Grafana, you can:

- Track how many PVCs and snapshots exist across all managed clusters
- Monitor backend capacity utilization before you run out of space
- Spot provisioning bottlenecks by looking at operation duration trends
- Compare storage consumption across clusters using Grafana's built-in cluster filter

This turns RHACM from a pure platform monitoring tool into a full-stack observability solution that includes your storage layer.

---

## Summary

| Phase | Where | What |
|-------|-------|------|
| 1.1 | Managed cluster | Enable user workload monitoring |
| 1.2 | Managed cluster | Create ServiceMonitor for Trident |
| 1.3 | Managed cluster | Deploy metrics allowlist ConfigMap |
| 1.4 | Managed cluster | Restart UWL collector |
| 2.1 | Hub cluster | Download dashboard JSON |
| 2.2 | Hub cluster | Patch datasource UID |
| 2.3 | Hub cluster | Create dashboard ConfigMap |
| 2.4 | Hub cluster | Label ConfigMap for discovery |
| 3 | Hub cluster | Verify metrics and dashboard |

The entire setup is declarative and reproducible. Once you've done it for one managed cluster, the same allowlist pattern works for every cluster running Trident in your fleet.

---

*If you found this useful, follow me for more content on OpenShift, RHACM, and storage orchestration.*
